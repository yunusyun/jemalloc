---
kind: workflow
scope: WF-deallocate
status: draft
origin: generated
source_paths: [src/jemalloc.c, src/tcache.c, src/arena.c, src/extent.c]
symbols: [je_free, free_fastpath, ifree, tcache_bin_flush_small, arena_slab_reg_dalloc, extent_deregister]
upstream: [business/index.md]
downstream: [business/domain-models/tcache.md, architecture/tcache-and-tsd.md]
---

# 释放业务流（WF-deallocate）

本页把 `free()` 从用户指针到 slab 位图回收、extent 注销的控制流压缩为可直达源码的判定链。入口背景见[业务总览](index.md)，缓存结构见[tcache 领域模型](domain-models/tcache.md)与[tcache 与 TSD 架构](../architecture/tcache-and-tsd.md)。

## 1. 入口与快路径判定

`src/jemalloc.c:2863` 的 `je_free` 只做一次分派：`free_fastpath` 成功即返回，否则落到 `src/jemalloc.c:2771` 的 `free_default`。快路径的收益是绕过 `tsd_fetch_min` 之外的完整 TSD 构造与 hook 调用。

| 阶段 | 定义锚点 | 通过条件 | 失败去向 |
| --- | --- | --- | --- |
| 公开入口 | `src/jemalloc.c:2863` `je_free(ptr)` | 总入口 | `2866-2868` 分流 |
| 快路径 | `src/jemalloc.c:2806` `free_fastpath(ptr, size, size_hint)` | `tsd` 存在且 `tsd_fast`（`2807-2810`） | `false` → `free_default` |
| slab 判定 | `src/jemalloc.c:2828` | `rtree_szind_slab_read_fast` 成功且 `alloc_ctx.slab` 为真 | 采样／大对象返回 `false` |
| 尺寸提示分支 | `src/jemalloc.c:2838` | `size <= SC_LOOKUP_MAXCLASS` 且非页对齐（排除采样对象） | 返回 `false` 走 rtree |
| 入栈 | `src/jemalloc.c:2850` `cache_bin_dalloc_easy(bin, bin_info, ptr)` | `ncached < ncached_max` | 返回 `false` → 慢路径 |
| 慢路径 | `src/jemalloc.c:2771` `free_default(ptr)` | `ptr != NULL`（`2773`） | 空指针直接返回 |
| 核心释放 | `src/jemalloc.c:2556` `ifree(tsd, ptr, tcache, slow_path)` | rtree 读出 `szind/slab`（`2570-2571`） | `idalloctm`（`2586-2590`） |

`config_prof && opt_prof` 时在 `src/jemalloc.c:2575-2577` 调 `prof_free`；`config_stats` 时在 `2581-2583` 累加 `tsd_thread_deallocatedp`。

## 2. 符号级调用树

```text
je_free(ptr) -> src/jemalloc.c:2863 je_free(ptr)
  ├─ [fast] src/jemalloc.c:2806 free_fastpath(ptr, 0, false) -> true -> return
  │    ├─ [guard] src/jemalloc.c:2807 tsd_get(false) / tsd_fast -> false 时退出
  │    ├─ [rtree] src/jemalloc.c:2823 rtree_szind_slab_read_fast -> szind, slab
  │    ├─ [guard] src/jemalloc.c:2838 size > SC_LOOKUP_MAXCLASS 或页对齐 -> 退出
  │    ├─ [guard] src/jemalloc.c:2844 ticker_trytick(&tcache->gc_ticker) -> 退出
  │    └─ [outbound] src/jemalloc.c:2850 cache_bin_dalloc_easy -> true -> return
  └─ [slow] src/jemalloc.c:2771 free_default(ptr)
       ├─ [fast-tsd] src/jemalloc.c:2786 tsd_fast -> ifree(..., false) (2790)
       └─ [slow-tsd] src/jemalloc.c:2793 tcache_get / hook_invoke_dalloc (2798) -> ifree(..., true) (2799)

ifree -> src/jemalloc.c:2556 ifree(tsd, ptr, tcache, slow_path)
  ├─ src/jemalloc.c:2570 rtree_szind_slab_read(&extents_rtree, ...) -> alloc_ctx.szind / slab
  ├─ [prof] src/jemalloc.c:2577 prof_free(...)
  └─ src/jemalloc.c:2586 idalloctm(tsd, ptr, tcache, &alloc_ctx, false, slow_path)

tcache 冲刷链（GC ticker 触发）:
  ticker_trytick -> src/tcache.c:55 tcache_bin_flush_small(tsd, tcache, tbin, binind, rem)
    -> src/tcache.c:134 tcache_bin_flush_small（定义）
    -> src/tcache.c:152 iealloc(...) -> item_extent[]
    -> src/tcache.c:174 malloc_mutex_lock(&bin->lock)
    -> src/tcache.c:189 arena_dalloc_bin_junked_locked(...)
    -> src/arena.c:359 arena_slab_reg_dalloc(slab, slab_data, ptr)

显式冲刷链:
  mallctl tcache.flush -> src/tcache.c:512 tcache_flush_cache
    -> src/tcache.c:517 tcache_bin_flush_small(..., rem=0)  // 小类
    -> src/tcache.c:525 tcache_bin_flush_large(..., 0)      // 大类

extent 注销链:
  extents_evict（retained 状态）-> src/extent.c:619 extent_deregister(tsdn, extent)
    -> src/extent.c:878 extent_deregister -> src/extent.c:879 extent_deregister_impl(..., true)
```

## 3. tcache 冲刷规则表

`src/tcache.c:55-79` 的增量 GC 按 `next_gc_bin` 轮转，每次只处理一个 bin；`rem` 是保留数量，冲刷数量为 `ncached - rem`。

| 条件 | 字面锚点 | 动作 | 副作用 |
| --- | --- | --- | --- |
| `low_water > 0` 且小类 | `src/tcache.c:55-57` | `tcache_bin_flush_small(..., ncached - low_water + (low_water >> 2))` | `lg_fill_div++`（`63-65`），条件 `ncached_max >> (div+1) >= 1` |
| `low_water > 0` 且大类 | `src/tcache.c:68-69` | `tcache_bin_flush_large(...)` | 不改变 `lg_fill_div` |
| `low_water < 0` | `src/tcache.c:71-78` | `lg_fill_div--` | 仅小类且 `lg_fill_div > 1` |
| 冲刷完成后 | `src/tcache.c:80-85` | `low_water = ncached`，`next_gc_bin` 递增到 `nhbins` 回零 | 保证轮询覆盖全部 bin |
| 显式 flush | `src/tcache.c:517`／`525` | `rem=0`，清空全部小类与大类 | `prof_accumbytes` 触发 `prof_idump`（`532-536`） |

`src/tcache.c:134` 的实现在 `143` 行算 `nflush = tbin->ncached - rem`，在 `156-206` 行按 extent 所属 arena/bin shard 分组加锁冲刷；跨 shard 对象被暂存到 `ndeferred`（`198-201`）留待下一轮。收尾 `221-223` 行用 `memmove` 把剩余 `rem` 项移到栈顶并写回 `ncached`。

## 4. slab 归还与 extent 注销

`src/arena.c:359` 的 `arena_slab_reg_dalloc` 是回收原子：`362` 行 `arena_slab_regind` 由指针反推 regind，`366` 行断言该位原为置位，`368` 行 `bitmap_unset` 清位，`369` 行 `extent_nfree_inc(slab)` 归还计数。注释 `365` 明确：重复释放未分配指针会触发断言失败。

`src/extent.c:878` 的 `extent_deregister` 仅一层包装，转 `src/extent.c:879` 的 `extent_deregister_impl(tsdn, extent, true)`，第三参数控制是否参与 gdump 统计。它被 `src/extent.c:619` 在 `extents_evict` 中对 retained 状态调用；dirty/muzzy 状态则只做 `extent_state_set(extent, extent_state_active)`（`616`）而不注销，以便复用。

| extent 状态 | `src/extent.c:611-623` 分支 | 处理 |
| --- | --- | --- |
| `extent_state_active` | `612-613` | `not_reached()` |
| `extent_state_dirty` / `extent_state_muzzy` | `614-617` | 标记为 `active`，保留 rtree 注册 |
| `extent_state_retained` | `618-619` | `extent_deregister` 摘除 rtree 映射 |

## 5. 安全校验与隐藏代价

| 检查 | 字面位置 | 后果 |
| --- | --- | --- |
| sized deallocation 错配 | `src/tcache.c:106` `tbin_extents_lookup_size_check` | `126-129` 调 `safety_check_fail` 后 `abort()`，仅 `--enable-extra-size-check` 生效 |
| 采样对象误入快路径 | `src/jemalloc.c:2828` | `slab == false` 时退出快路径，避免把采样对象压入 tcache |
| 页面对齐启发式 | `src/jemalloc.c:2838` | 页对齐视作采样对象，强制走 rtree |
| 重入释放 | `src/jemalloc.c:2561-2563` | 要求 `slow_path`，否则断言 |
| 空指针 | `src/jemalloc.c:2773` | `ptr == NULL` 时 `free_default` 直接返回 |

## 6. 共变耦合与修改清单（Co-Change Playbook）

| 改动点 | 联动文件／符号 | 隐式陷阱 | 建议验证 |
| --- | --- | --- | --- |
| 改快路径判定 | `src/jemalloc.c:2806` → `2828`／`2838`／`2850` | `config_cache_oblivious` 下地址随机化，页对齐启发式失效，必须走 rtree | 分别编译两种 config 对照 |
| 改 GC 折半策略 | `src/tcache.c:55-79` → `src/arena.c:1394` 的补货量 | `lg_fill_div` 增减不对称会造成 bin 长期饥饿 | 长跑观察 `nflushes` 计数 |
| 改 slab 回收位图 | `src/arena.c:359` → `src/tcache.c:189` → `src/jemalloc.c:2850` | 位图与 `nfree` 必须同锁更新，否则计数漂移 | 重复释放与并发释放用例 |
| 改 extent 注销时机 | `src/extent.c:619` → `878` → `src/extent.c:567` `extents_evict` | retained 注销后地址可被新映射复用，`rtree` 陈旧项会导致悬空查找 | 压力测试 extent 分裂／合并 |

释放路径的安全检查是最后一道防线，调整快路径时必须同步复核[tcache 与 TSD 架构](../architecture/tcache-and-tsd.md)中的状态机约束。
