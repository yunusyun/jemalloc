---
kind: config-dict
scope: configuration
status: draft
origin: generated
source_paths: [src/jemalloc.c, src/arena.c, include/jemalloc/internal/jemalloc_preamble.h.in]
symbols: [obtain_malloc_conf, malloc_conf_init_helper, malloc_slow_flag_init, arena_decay_ms_set]
upstream: [INDEX.md]
downstream: [operations/deployment-and-runtime.md, business/wf-decay.md]
---

# MALLOC_CONF 配置字典与四源优先级

## 1. 四源优先级链（唯一真相）

`MALLOC_CONF_NSOURCES 4` 定义于 `src/jemalloc.c:935`；`obtain_malloc_conf`（`src/jemalloc.c:938`）按 `which_source` 依次取值，顺序即优先级（**后解析者覆盖同名键**）：

| 序 | 来源 | 字面证据锚点 | `opts_explain` 文案锚点 |
|---|---|---|---|
| 0 | 编译期 `--with-malloc-conf` → `config_malloc_conf` | `src/jemalloc.c:951-953`；常量定义 `include/jemalloc/internal/jemalloc_preamble.h.in:89`（`JEMALLOC_CONFIG_MALLOC_CONF`） | `src/jemalloc.c:1025` |
| 1 | 全局 `malloc_conf`（`je_malloc_conf`） | `src/jemalloc.c:954-962`；声明 `include/jemalloc/jemalloc_protos.h.in:6` | `src/jemalloc.c:1026` |
| 2 | `/etc/malloc.conf` 符号链接的**名字** | `src/jemalloc.c:963-993`：`readlink`（`:980`）/`readlinkat`（`:982`），失败恢复 errno（`:984-989`） | `src/jemalloc.c:1027-1028` |
| 3 | 环境变量 `MALLOC_CONF` | `src/jemalloc.c:994-1012`：`jemalloc_secure_getenv(envname)`（`:1003`），其实现 `src/jemalloc.c:712-723` | `src/jemalloc.c:1029` |

`malloc_conf_init_helper`（`src/jemalloc.c:1021`）以 `for (i = 0; i < MALLOC_CONF_NSOURCES; i++)`（`:1035`）顺序遍历，`opts_cache[i]` 缓存首次结果（`:1038-1040`），键逐段 `malloc_conf_next` 解析（`:1050-1051`）。`malloc_conf_init`（`src/jemalloc.c:1440`）分两遍：第一遍只取 `confirm_conf` 并缓存四源（`:1445`），第二遍解析全部键（`:1446-1447`）。`confirm_conf:true` 时逐源打印 `<jemalloc>: malloc_conf #%u (%s): "%s"`（`:1042-1044`）。

```text
--with-malloc-conf -> config_malloc_conf(:952)
  -> 全局 je_malloc_conf(:955) -> /etc/malloc.conf readlink(:980) -> MALLOC_CONF getenv(:1003)
       -> malloc_conf_init_helper(:1021) 逐源解析 -> 同名键后者覆盖
            -> malloc_slow_flag_init(:920) 汇总 fastpath 慢标记
```

## 2. 全部 MALLOC_CONF 键（字面 + 锚点）

| 键 | 取值/约束（源码字面） | 解析锚点 |
|---|---|---|
| `confirm_conf` | bool | `src/jemalloc.c:1170` |
| `abort` | bool | `src/jemalloc.c:1175` |
| `abort_conf` | bool | `src/jemalloc.c:1176` |
| `metadata_thp` | `metadata_thp_mode_names[i]` 逐项匹配 | `src/jemalloc.c:1177-1193` |
| `retain` | bool | `src/jemalloc.c:1194` |
| `dss` | `dss_prec_names[i]`；经 `extent_dss_prec_set` 下发 | `src/jemalloc.c:1195-1218` |
| `narenas` | unsigned，min 1 | `src/jemalloc.c:1219-1221` |
| `bin_shards` | 多段 `<start>-<end>:<nshards>\|...`，经 `bin_update_shard_size` | `src/jemalloc.c:1222-1243` |
| `dirty_decay_ms` | ssize_t，`>=-1` | `src/jemalloc.c:1244-1247` |
| `muzzy_decay_ms` | ssize_t，`>=-1` | `src/jemalloc.c:1248-1251` |
| `stats_print` | bool | `src/jemalloc.c:1252` |
| `stats_print_opts` | 逐字符取 `STATS_PRINT_OPTIONS` | `src/jemalloc.c:1253-1256`；`init_opt_stats_print_opts` `:753-776` |
| `junk` | `true\|false\|alloc\|free` | `src/jemalloc.c:1257-1281` |
| `zero` | bool（`config_fill` 下） | `src/jemalloc.c:1282` |
| `utrace` | bool（`config_utrace` 下） | `src/jemalloc.c:1284-1286` |
| `xmalloc` | bool（`config_xmalloc` 下） | `src/jemalloc.c:1287-1289` |
| `tcache` | bool | `src/jemalloc.c:1290` |
| `lg_tcache_max` | ssize_t，`-1 .. (sizeof(size_t)<<3)-1` | `src/jemalloc.c:1291-1292` |
| `oversize_threshold` | size_t，`<= SC_LARGE_MAXCLASS` | `src/jemalloc.c:1302-1304` |
| `lg_extent_max_active_fit` | size_t，`<= sizeof(size_t)<<3` | `src/jemalloc.c:1305-1308` |
| `percpu_arena` | `percpu_arena_mode_names[i]`；无 getcpu 支持报 `No getcpu support` | `src/jemalloc.c:1310-1331` |
| `background_thread` | bool | `src/jemalloc.c:1332-1333` |
| `max_background_threads` | size_t，`1..当前值`，clip 模式 | `src/jemalloc.c:1334-1338` |
| `slab_sizes` | 多段 `<start>-<end>:<pages>\|...`，经 `sc_data_update_slab_size` | `src/jemalloc.c:1339-1362` |
| `prof` | bool（`config_prof` 下） | `src/jemalloc.c:1363-1364` |
| `prof_prefix` | 字符串，默认 `jeprof` | `src/jemalloc.c:1365-1366` |
| `prof_active` | bool | `src/jemalloc.c:1367` |
| `prof_thread_active_init` | bool | `src/jemalloc.c:1368-1369` |
| `lg_prof_sample` | size_t，`<= (sizeof(uint64_t)<<3)-1`，clip | `src/jemalloc.c:1370-1373` |
| `prof_accum` | bool | `src/jemalloc.c:1374` |
| `lg_prof_interval` | ssize_t，`>=-1` | `src/jemalloc.c:1375-1377` |
| `prof_gdump` | bool | `src/jemalloc.c:1378` |
| `prof_final` | bool | `src/jemalloc.c:1379` |
| `prof_leak` | bool | `src/jemalloc.c:1380` |
| `prof_log` | bool | `src/jemalloc.c:1381` |
| `log` | 字符串写入 `log_var_names` | `src/jemalloc.c:1383-1392` |
| `thp` | `thp_mode_names[i]`；无 madvise huge 报 `No THP support` | `src/jemalloc.c:1393-1413` |

未知键落到 `CONF_ERROR("Invalid conf pair", ...)`（`src/jemalloc.c:1414`），最终 `opt_abort_conf && had_conf_error` 触发 `malloc_abort_invalid_conf`（`:1432-1434`）。完整错误字面见 [错误码字典](error-codes.md)。

## 3. 编译期开关决定哪些键生效

`include/jemalloc/internal/jemalloc_preamble.h.in` 把 `configure` 结果固化为 `static const bool`：`config_debug`（`:54-60`）、`have_dss`（`:61-67`）、`have_madvise_huge`（`:68-74`）、`config_fill`（`:75-81`）、`config_lazy_lock`（`:82-88`）、`config_malloc_conf`（`:89`）、`config_prof`（`:90-96`）、`config_stats`（`:118-124`）、`config_utrace`（`:132-138`）、`config_xmalloc`（`:139-145`）、`config_log`（`:157-163`）、`config_opt_safety_checks`（`:168-181`）、`have_percpu_arena`（`:183-193`）、`have_background_thread`（`:205-211`）。因此 `junk/zero`、`utrace`、`xmalloc`、`prof*`、`log`、`percpu_arena`、`thp` 键在关闭相应编译特性时不会被解析；运行时以 `config.` mallctl 子树暴露这些常量（`src/ctl.c:293-306`）。安装期如何设定这些开关见 [部署与运行时](../operations/deployment-and-runtime.md)。

## 4. decay 配置的落点

`dirty_decay_ms` / `muzzy_decay_ms` 经 `opt_node`（`src/ctl.c:320-321`）可读，运行时由 `arena_dirty_decay_ms_set`（`src/arena.c:810-814`）与 `arena_muzzy_decay_ms_set`（`src/arena.c:817-819`）进入公共函数 `arena_decay_ms_set`（`src/arena.c:787`）：先 `arena_decay_ms_valid` 校验（`:789-791`），持 `decay->mtx` 后 `arena_decay_reinit`（`:802`）并立即 `arena_maybe_decay(..., false)`（`:803`）。源码注释明确：重设会丢弃旧 backlog，可能立即清除大量脏页（`:794-801`）。这是改动 decay 值的主要行为风险；业务侧衰减策略见 [wf 衰减](../business/wf-decay.md)。

`malloc_slow_flag_init`（`src/jemalloc.c:920-932`）把 `opt_junk_alloc/opt_junk_free/opt_zero/opt_utrace/opt_xmalloc` 折成 `malloc_slow_flags`，并令 `malloc_slow = (malloc_slow_flags != 0)`（`:931`）；`je_malloc` 快路径据此决定是否尾调用 `malloc_default`（`src/jemalloc.c:2331-2333`）。

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 新增/改名配置键：同步 `src/jemalloc.c:1170-1414` 的 `CONF_MATCH` 字面量、`opt_node`（`src/ctl.c:308-342`）、`test/unit/mallctl.c` 的读写断言与本页键表；三处必须逐字一致。
2. 调整优先级：改 `obtain_malloc_conf`（`src/jemalloc.c:938`）的 `switch` 分支与 `opts_explain`（`:1024-1030`）；注意 `/etc/malloc.conf` 分支只在非 `_WIN32` 生效（`:965`）。
3. 改 `confirm_conf` 输出：联动 `src/jemalloc.c:1042-1065` 的两条 `malloc_printf` 与 [错误码字典](error-codes.md)。
4. 改 decay 语义：联动 `src/jemalloc.c:1244-1251`、`src/arena.c:787-819`、`src/ctl.c:357-358` 与 [wf 衰减页](../business/wf-decay.md)的策略描述。
5. 改编译期开关：联动 `include/jemalloc/internal/jemalloc_preamble.h.in:54-211`、`configure.ac` 的 AC_DEFINE 与 [部署页](../operations/deployment-and-runtime.md)的安装命令，并复核 `config.` mallctl 子树（`src/ctl.c:293-306`）。
