---
kind: api-contract
scope: mallctl-namespace
status: draft
origin: generated
source_paths: [src/ctl.c, src/jemalloc.c]
symbols: [ctl_byname, ctl_nametomib, ctl_bymib, root_node, ctl_init, epoch_ctl]
upstream: [contracts/api-reference.md]
downstream: [operations/runbook-and-observability.md, contracts/configuration.md]
---

# mallctl 命名空间契约：键值树、MIB 与读写语义

## 1. 三层入口与初始化门槛

公共入口 `je_mallctl`（`src/jemalloc.c:3645`）、`je_mallctlnametomib`（`src/jemalloc.c:3667`）、`je_mallctlbymib`（`src/jemalloc.c:3687`）分别转调 `ctl_byname`（`src/jemalloc.c:3659` → `src/ctl.c:1292`）、`ctl_nametomib`（`src/jemalloc.c:3679` → `src/ctl.c:1324`）、`ctl_bymib`（`src/jemalloc.c:3701` → `src/ctl.c:1338`）。三者共同前置 `malloc_init()`：失败返回 `EAGAIN`，分别记录在 `src/jemalloc.c:3654`、`:3674`、`:3695`；`ctl_*` 层另有 `!ctl_initialized && ctl_init(tsd)` 失败时返回 `EAGAIN`（`src/ctl.c:1300-1303`、`:1327-1330`、`:1344-1347`）。`ctl_init`（`src/ctl.c:1118`）在 `ctl_mtx` 保护下用 `base_alloc` 预分配 `ctl_arenas`（`:1131-1137`）。

```text
mallctl(name, oldp, oldlenp, newp, newlen)
  -> src/jemalloc.c:3645 je_mallctl -> src/ctl.c:1292 ctl_byname
       ├─ [未初始化] src/ctl.c:1118 ctl_init -> EAGAIN
       ├─ src/ctl.c:1194 ctl_lookup(name) -> ENOENT 或 mib[]
       └─ [叶子节点] node->ctl(...) -> 0 / EINVAL / EFAULT / EPERM
mallctlnametomib -> src/ctl.c:1324 ctl_nametomib -> ctl_lookup(mibp=NULL 位置)
mallctlbymib     -> src/ctl.c:1338 ctl_bymib -> 按 mib 逐层下钻 -> ENOENT
```

## 2. `root_node` 全部 13 个根结点

`root_node` 定义于 `src/ctl.c:622-636`，`super_root_node` 用空名 `""` 包一层（`src/ctl.c:637-639`）；`ctl_lookup` 以 `.` 切分名称、逐层匹配 named/indexed 子节点（`src/ctl.c:1203-1229`、`:1235-1257`）。

| # | 根键 | 类型 | 定义锚点 | 子树要点 |
|---|---|---|---|---|
| 1 | `version` | 叶子（只读） | `src/ctl.c:623` | `CTL_RO_NL_GEN(version, JEMALLOC_VERSION, const char *)` 见 `src/ctl.c:1613` |
| 2 | `epoch` | 叶子（读写） | `src/ctl.c:624` | `epoch_ctl`（`src/ctl.c:1616`）写入时触发 `ctl_refresh`（`:1624`） |
| 3 | `background_thread` | 叶子（读写） | `src/ctl.c:625` | `background_thread_ctl`（`src/ctl.c:1635`）；无支持返回 `ENOENT`（`:1641-1643`） |
| 4 | `max_background_threads` | 叶子 | `src/ctl.c:626` | 配置上限，与 `MALLOC_CONF` 同名键同源 |
| 5 | `thread` | named | `src/ctl.c:627` | `thread_node`（`src/ctl.c:283-291`）：`arena/allocated/allocatedp/deallocated/deallocatedp/tcache/prof` |
| 6 | `config` | named（只读） | `src/ctl.c:628` | `config_node`（`src/ctl.c:293-306`）：`cache_oblivious/debug/fill/lazy_lock/malloc_conf/opt_safety_checks/prof/prof_libgcc/prof_libunwind/stats/utrace/xmalloc` |
| 7 | `opt` | named | `src/ctl.c:629` | `opt_node`（`src/ctl.c:308-342`）：40 个运行期选项镜像，见 [配置字典](../configuration.md) |
| 8 | `tcache` | named | `src/ctl.c:630` | `tcache_node`（`src/ctl.c:344-348`）：`create/flush/destroy` |
| 9 | `arena` | indexed | `src/ctl.c:631` | `arena_node`（`src/ctl.c:366-368`）→ `arena_i_node`（`src/ctl.c:350-361`）：`initialized/decay/purge/reset/destroy/dss/dirty_decay_ms/muzzy_decay_ms/extent_hooks/retain_grow_limit` |
| 10 | `arenas` | named | `src/ctl.c:632` | `arenas_node`（`src/ctl.c:395-409`）：`narenas/dirty_decay_ms/muzzy_decay_ms/quantum/page/tcache_max/nbins/nhbins/bin/nlextents/lextent/create/lookup` |
| 11 | `prof` | named | `src/ctl.c:633` | `prof_node`（`src/ctl.c:411-421`）：`thread_active_init/active/dump/gdump/reset/interval/lg_sample/log_start/log_stop` |
| 12 | `stats` | named | `src/ctl.c:634` | `stats_node`（`src/ctl.c:581-593`）：`allocated/active/metadata/metadata_thp/resident/mapped/retained/background_thread/mutexes/arenas` |
| 13 | `experimental` | named | `src/ctl.c:635` | `experimental_node`（`src/ctl.c:616-620`）：`hooks/utilization/arenas`；hooks 子键 `install/remove`（`:595-598`） |

## 3. 关键子结点展开

| 命名空间 | 子键（字面） | 锚点 | 读写备注 |
|---|---|---|---|
| `thread.` | `arena` `allocated` `allocatedp` `deallocated` `deallocatedp` `tcache` `prof` | `src/ctl.c:283-291` | `tcache`/`prof` 为 further named 子结点；`thread_prof.active` 见 `src/ctl.c:280` |
| `config.` | `malloc_conf` `debug` `fill` `stats` `prof` `xmalloc` `utrace` 等 12 项 | `src/ctl.c:293-306` | 与 `include/jemalloc/internal/jemalloc_preamble.h.in:54-210` 的编译期常量一一对应 |
| `opt.` | `abort` `abort_conf` `confirm_conf` `metadata_thp` `retain` `dss` `narenas` `percpu_arena` `oversize_threshold` `background_thread` `max_background_threads` `dirty_decay_ms` `muzzy_decay_ms` `stats_print` `stats_print_opts` `junk` `zero` `utrace` `xmalloc` `tcache` `thp` `lg_extent_max_active_fit` `lg_tcache_max` `prof` `prof_prefix` `prof_active` `prof_thread_active_init` `lg_prof_sample` `lg_prof_interval` `prof_gdump` `prof_final` `prof_leak` `prof_accum` | `src/ctl.c:308-342` | 与 `src/jemalloc.c:1170-1414` 的 `CONF_MATCH` 分支同名；合并刷新在 `src/ctl.c:1044` |
| `arena.<i>.` | `initialized` `decay` `purge` `reset` `destroy` `dss` `dirty_decay_ms` `muzzy_decay_ms` `extent_hooks` `retain_grow_limit` | `src/ctl.c:350-361` | 索引分配见 `ctl_arena_init`（`src/ctl.c:1004`），复用 destroyed 队列（`:1008-1014`） |
| `arenas.bin.<i>.` | `size` `nregs` `slab_size` `nshards` | `src/ctl.c:370-375` | indexed 节点 `arenas_bin_node`（`:380-382`） |
| `arenas.lextent.<i>.` | `size` | `src/ctl.c:384-386` | indexed 节点 `arenas_lextent_node`（`:391-393`） |
| `stats.arenas.<i>.small` | `allocated` `nmalloc` `ndalloc` `nrequests` `nfills` `nflushes` | `src/ctl.c:422-429` | large 对应键见 `src/ctl.c:431-438` |
| `stats.mutexes.` | `num_ops` `num_wait` 等由 `MUTEX_PROF_DATA_NODE` 宏展开 | `src/ctl.c:440-444` | 全局互斥量清单见 `src/ctl.c:575-577`，`reset` 在 `:577` |
| `experimental.arenas.<i>.` | `pactivep` | `src/ctl.c:605-607` | 只读指针导出 |

## 4. 刷新、错误与快照语义

`epoch_ctl`（`src/ctl.c:1616-1632`）在 `newp != NULL` 时调用 `ctl_refresh`（`src/ctl.c:1044-1115`）：先清空汇总 `ctl_arena_clear`（`:1053`），逐 arena 合并（`ctl_arena_refresh`，`:1065-1067`），再写 `allocated/active/metadata/resident/mapped/retained`（`:1071-1086`），最后 `ctl_arenas->epoch++`（`:1114`）。`stats_print` 也借 `je_mallctl("epoch", ...)` 触发刷新，见 `src/stats.c:1414-1415`。

| 返回码 | 触发点 | 含义 |
|---|---|---|
| `EAGAIN` | `src/jemalloc.c:3654`、`:3674`、`:3695`；`src/ctl.c:1301`、`:1328`、`:1345` | 分配器或 ctl 初始化失败 |
| `ENOENT` | `src/ctl.c:1206`、`:1232`、`:1242`、`:1277`、`:1316` | 名字不存在 / 路径只到中间节点 / 索引越界（`src/ctl.c:1357`） |
| `EINVAL` | `src/ctl.c:1653` | `background_thread` 写入长度不等于 `sizeof(bool)` |
| `EFAULT` | `src/ctl.c:1668` | 启用后台线程失败 |
| `EPERM` | 单测 `test/unit/mallctl.c:9-12` 对 `version` 写入的断言 | 只读节点被写 |

`MALLCTL_ARENAS_ALL=4096` 与 `MALLCTL_ARENAS_DESTROYED=4097`（`include/jemalloc/jemalloc_macros.h.in:47`、`:52`）是 `arena.<i>.*` / `stats.arenas.<i>.*` 的特殊索引，`ctl_refresh` 用前者取汇总槽（`src/ctl.c:1046`）。运行期排障路径见 [排障与可观测](../../operations/runbook-and-observability.md)。

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 新增根键：必须同时改 `root_node`（`src/ctl.c:622-636`）、对应 `*_node[]` 数组、`ctl_init`（`:1118`）的分配尺寸假设，以及 `test/unit/mallctl.c` 的 `ENOENT/EPERM/EINVAL` 断言集。
2. 改动 `epoch` 刷新：联动 `epoch_ctl`（`src/ctl.c:1616`）、`ctl_refresh`（`:1044`）、`src/stats.c:1414-1425` 的 `EAGAIN/abort` 分支与 [排障页](../../operations/runbook-and-observability.md)的决策树。
3. 配置键重命名：改 `src/jemalloc.c:1170-1414` 的 `CONF_MATCH` 字面量、`opt_node`（`src/ctl.c:308-342`）与 [配置字典](../configuration.md)三处，保持 `config.` / `opt.` 镜像一致。
4. MIB 稳定性：`ctl_lookup`（`src/ctl.c:1194`）以子节点数组下标作 mib 分量，重排 `*_node[]` 会使已缓存的 MIB 静默改指向；变更须同步 `ctl_bymib`（`:1338`）验证与 `mallctlnametomib` 调用方。
5. 后台线程节点变更联动 `src/ctl.c:1635-1669` 的 `background_thread_ctl_init` / `background_threads_enable` 与 `src/jemalloc.c:1788-1799` 的启动路径。
