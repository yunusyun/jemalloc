---
kind: workflow
scope: WF-profile
status: draft
origin: generated
source_paths: [src/prof.c, src/stats.c, src/jemalloc.c]
symbols: [prof_dump, prof_gdump, prof_sample_threshold_update, stats_print, je_malloc_stats_print]
upstream: [business/index.md]
downstream: [architecture/control-and-observability.md]
---

# 性能分析业务流（WF-profile）

本页覆盖采样阈值生成、堆转储触发与统计输出三条相互独立的观测链路。入口背景见[业务总览](index.md)，控制面全景见[控制与可观测性架构](../architecture/control-and-observability.md)，分配侧采样接入点见[分配业务流](wf-allocate.md)。

## 1. 三条观测链路

| 链路 | 核心符号 | 定义锚点 | 产出 |
| --- | --- | --- | --- |
| 采样阈值 | `prof_sample_threshold_update` | `src/prof.c:1140` | 写入 `tsd_bytes_until_sample`，决定下一次采样点 |
| 堆转储 | `prof_dump` | `src/prof.c:1906` | 生成堆快照文件（`leakcheck` 开关） |
| 增量转储 | `prof_gdump` | `src/prof.c:2106` | 增长触发的自动 dump |
| 统计输出 | `stats_print` | `src/stats.c:1396` | 表格／JSON 文本流 |
| 公开出口 | `je_malloc_stats_print` | `src/jemalloc.c:3708` | 回调输出的 C API |

`src/jemalloc.c:3708` 的 `je_malloc_stats_print(write_cb, cbopaque, opts)` 在 `3714` 行 `tsdn_fetch`，`3716` 行直转 `stats_print`，前后各一次 `check_entry_exit_locking`。`src/jemalloc.c:675` 另有一处 `stats_print` 族调用：以 `opt_stats_print_opts` 在 atexit 阶段输出。

## 2. 符号级调用树

```text
[分配快路径] je_malloc -> src/jemalloc.c:2350 bytes_until_sample -= usize
  ├─ src/jemalloc.c:2355 bytes_until_sample < 0
  │    ├─ [prof 未激活] src/jemalloc.c:2362-2364 tsd_bytes_until_sample_set(SSIZE_MAX) -> malloc_default
  │    └─ [prof 激活]   src/jemalloc.c:2365 malloc_default -> slow path 重采样
  └─ src/jemalloc.c:2116 imalloc_body 中 prof_alloc_prep -> 采样分支

阈值更新链:
  src/prof.c:327 prof_sample_threshold_update(tdata) [调用点]
    -> src/prof.c:1140 prof_sample_threshold_update(prof_tdata_t *tdata)
       ├─ [lg_prof_sample == 0] src/prof.c:1146-1149 置 0 并 return
       ├─ [几何分布] src/prof.c:1152-1159 mean = 2^lg_prof_sample
       └─ src/prof.c:1169 prng_lg_range_u64(&tdata->prng_state, 53)

增量转储链:
  src/prof.c:581 prof_gdump(tsdn) [调用点]
    -> src/prof.c:2106 prof_gdump(tsdn)
       ├─ [guard] src/prof.c:2112 !prof_booted || tsdn_null || !prof_active_get_unlocked -> return
       ├─ [guard] src/prof.c:2116 reentrancy_level > 0 -> return
       ├─ src/prof.c:2120 prof_tdata_get(tsd, false)
       └─ [入队中] src/prof.c:2124-2126 tdata->enq_gdump = true

显式转储链:
  prof_dump -> src/prof.c:1906 prof_dump(tsd, propagate_err, filename, leakcheck)
    ├─ src/prof.c:1911 prof_tdata_get(tsd, true)
    ├─ src/prof.c:1916 pre_reentrancy(tsd, NULL)
    ├─ src/prof.c:1917 malloc_mutex_lock(&prof_dump_mtx)
    ├─ src/prof.c:1923 prof_dump_prep(...)
    ├─ src/prof.c:1925 prof_dump_file(...)
    ├─ src/prof.c:1928 prof_gctx_finish(...)
    └─ src/prof.c:1930-1931 unlock + post_reentrancy

统计输出链:
  je_malloc_stats_print -> src/jemalloc.c:3708 -> src/stats.c:1396 stats_print(write_cb, cbopaque, opts)
    ├─ src/stats.c:1401-1403 STATS_PRINT_OPTIONS 展开为默认值
    ├─ src/stats.c:1414 je_mallctl("epoch", ...) -> 刷新缓存；1417 EAGAIN 特殊处理
    ├─ src/stats.c:1427-1435 按 opts 字符覆盖默认值
    ├─ src/stats.c:1438-1444 emitter_init + emitter_begin
    └─ src/stats.c:1446-1452 general / bins / large / mutex / extents 分节输出
```

## 3. 采样阈值规则表

`src/prof.c:1140` 的函数在 `1141-1144` 行处理 `!config_prof`，`1146-1149` 行处理 `lg_prof_sample == 0`。源码注释 `1152-1167` 给出几何分布公式：均值为 `2^lg_prof_sample`，用 `log(u)/log(1-p)` 反变换生成，`p = 1 / 2^lg_prof_sample`。

| 条件 | 字面锚点 | 阈值行为 |
| --- | --- | --- |
| `!config_prof` | `src/prof.c:1142-1144` | 直接返回，不触碰 tsd |
| `lg_prof_sample == 0` | `src/prof.c:1146-1149` | `tsd_bytes_until_sample_set(..., 0)`，每次分配都采样 |
| `lg_prof_sample > 0` | `src/prof.c:1169` | `prng_lg_range_u64(&tdata->prng_state, 53)` 抽 53 位随机数 |
| 消费端扣减 | `src/jemalloc.c:2352` | `bytes_until_sample -= usize`，小分配逐步逼近阈值 |
| 阈值耗尽且关闭 prof | `src/jemalloc.c:2362-2364` | 设为 `SSIZE_MAX`，避免每次都进慢路径 |
| 采样触发 | `src/jemalloc.c:2116-2131` | `prof_alloc_prep` → `imalloc_sample`，小对象被提升到 `SC_LARGE_MINCLASS` |

`src/jemalloc.c:1965-1977` 描述小对象采样提升：`bumped_usize = sz_s2u(SC_LARGE_MINCLASS)`，分配后调 `arena_prof_promote` 恢复原 `usize` 语义。这是采样路径与大对象路径共享 extent 的关键桥接。

## 4. 堆转储的并发保护

`src/prof.c:1906` 的 `prof_dump` 用三层保护：`1911` 行确保 `prof_tdata` 可用（失败返回 `true`），`1916` 行 `pre_reentrancy` 屏蔽重入，`1917` 行独占 `prof_dump_mtx` 串行化多个 dump 请求。`1923-1928` 行依次 `prof_dump_prep` → `prof_dump_file` → `prof_gctx_finish`，确保合并上下文被正确释放。

`src/prof.c:2106` 的 `prof_gdump` 更保守：`2112` 行同时检查 `prof_booted`、`tsdn_null`、`prof_active_get_unlocked()`，`2116` 行拒绝重入，`2124-2126` 行在 `tdata->enq` 已为真时只置 `enq_gdump` 标志，避免重复入队。`prof_active_get` 的带锁实现位于 `src/prof.c:2400-2407`。

## 5. 统计输出的选项矩阵

`src/stats.c:1396` 的 `stats_print` 先以 `je_mallctl("epoch", ...)`（`1414`）刷新 ctl 缓存；`1417-1425` 行区分 `EAGAIN`（打印内存分配失败信息后返回）与其他失败（打印后 `abort()`）。选项默认值由 `STATS_PRINT_OPTIONS` 宏在 `1401-1403` 展开，`1427-1435` 行按传入字符逐个覆盖。

| 选项字符 | 变量 | 默认值 | 对应输出 |
| --- | --- | --- | --- |
| `J` | `json` | `false` | 切换 `emitter_output_json`（`1440`） |
| `g` | `general` | `true` | `stats_general_print`（`1447`） |
| `m`／`d`／`a` | `merged`／`destroyed`／`unmerged` | `config_stats` | 由 `stats_print_helper(&emitter, merged, destroyed, unmerged, ...)`（`1450`）处理 |
| `b` | `bins` | `true` | bin 明细 |
| `l` | `large` | `true` | 大对象明细 |
| `x` | `mutex` | `true` | 互斥锁统计 |
| `e` | `extents` | `true` | extent 统计 |

`src/stats.c:1443` 输出 `___ Begin jemalloc statistics ___`，`1454-1456` 行闭合 JSON 对象并以 `--- End jemalloc statistics ---` 收尾。选项宏的字面定义在 `include/jemalloc/internal/stats.h:5-14`，与 `src/stats.c:1401-1403` 的展开一致。

## 6. 共变耦合与修改清单（Co-Change Playbook）

| 改动点 | 联动文件／符号 | 隐式陷阱 | 建议验证 |
| --- | --- | --- | --- |
| 改采样分布 | `src/prof.c:1140` → `src/jemalloc.c:2352`／`2116` | 阈值期望变化会改变 `imalloc_sample` 命中率，进而改变小对象布局 | 固定种子跑采样计数对照 |
| 改 dump 锁范围 | `src/prof.c:1917` → `1928` → `1930` | 缩小锁范围会让 `prof_gctx_finish` 与并发 dump 竞争 | 多线程并发 dump 压测 |
| 改 gdump 入队 | `src/prof.c:2124-2126` → `581` 调用点 | 漏置 `enq_gdump` 会丢失增长触发的快照 | 增长阈值边界测试 |
| 改统计选项 | `src/stats.c:1401-1403`／`1427-1435` → `include/jemalloc/internal/stats.h:5-14` | 两处宏／解析不同步会导致选项字符被静默忽略 | 逐字符选项矩阵测试 |
| 改 epoch 刷新 | `src/stats.c:1414-1425` | `EAGAIN` 与其他错误必须区分，否则 OOM 时误 `abort()` | 注入分配失败场景 |

分析流的三项输出共用 prof 线程的 tdata 生命周期，任何调整都应复核[控制与可观测性架构](../architecture/control-and-observability.md)中的 mallctl 命名空间约束。
