---
kind: component
scope: ctl-stats-logging
status: draft
origin: generated
source_paths: [src/ctl.c, src/stats.c, src/log.c, src/malloc_io.c]
symbols: [ctl_init, ctl_byname, ctl_lookup, stats_print, log_var_matches_segment, malloc_vsnprintf]
upstream: [contracts/api-reference.md, contracts/api/mallctl.md]
downstream: [operations/runbook-and-observability.md]
---

# 控制面与可观测（ctl-stats-logging）

## 1. 角色与边界

本组件实现 `mallctl` 命名空间解析、运行时配置读写、统计快照输出、日志分类过滤与不可依赖 libc 的格式化输出。导出 API 见 [contracts/api-reference.md](../contracts/api-reference.md)，`mallctl` 逐项契约见 [contracts/api/mallctl.md](../contracts/api/mallctl.md)，运维观测口径见 [operations/runbook-and-observability.md](../operations/runbook-and-observability.md)。分配入口如何进入控制面见 [architecture/allocator-core.md](allocator-core.md)。

## 2. 符号级调用树

```text
je_mallctl(name,...) src/jemalloc.c:3645
  ├─ [guard] malloc_init() 失败 -> EAGAIN src/jemalloc.c:3652
  └─ ctl_byname src/ctl.c:1292
       ├─ [guard] !ctl_initialized -> ctl_init src/ctl.c:1118 -> EAGAIN
       ├─ ctl_lookup src/ctl.c:1194 -> 点分名拆为 nodes[]/mib[]/depth
       │    └─ 首段未命中 -> ENOENT src/ctl.c:1206
       └─ [dispatch] named/indexed 节点的 ctl->get / ctl->set

malloc_stats_print -> stats_print src/stats.c:1396
  ├─ ctl_refresh 刷新统计缓存（malloc_stats_prepare）
  ├─ STATS_PRINT_OPTIONS 解析 opts src/stats.c:1401-1403
  └─ write_cb 逐段回调输出 -> stats_arena_bins_print src/stats.c:270
                            -> stats_arena_extents_print src/stats.c:548

LOG 宏 -> log_var_matches_segment src/log.c:22 -> malloc_vsnprintf src/malloc_io.c:327 -> malloc_write
```

## 3. 源码锚点与职责契约表

| 文件:行号 | 符号 | 职责 | 入 → 出 |
|---|---|---|---|
| `src/ctl.c:1118` | `ctl_init` | 懒初始化 ctl 锁与 arena 视图 | `tsd_t*` → `bool` 错误 |
| `src/ctl.c:1292` | `ctl_byname` | 点分名读写主入口 | `name`,`oldp`,`newp` → `errno` |
| `src/ctl.c:1194` | `ctl_lookup` | 名字到 mib/节点数组的解析 | `name` → `nodes[]`,`mib[]`,`depth` |
| `src/stats.c:1396` | `stats_print` | 统一统计输出与选项解析 | `write_cb`,`opts` → `void` |
| `src/log.c:22` | `log_var_matches_segment` | 日志类别段匹配（`.` 通配） | 两段字符串 → `bool` |
| `src/malloc_io.c:327` | `malloc_vsnprintf` | 自实现 vsnprintf，避免重入 libc | `format`,`va_list` → 写入字节数 |

`je_mallctl` 在 `src/jemalloc.c:3645` 做 `malloc_init()` 前置检查并把失败映射为 `EAGAIN`（`src/jemalloc.c:3652-3655`），随后转调 `ctl_byname`（`src/jemalloc.c:3659`）。

## 4. mallctl 命名空间与初始化语义

`root_node`（`src/ctl.c:622-636`）挂载 13 个一级节点：`version`、`epoch`、`background_thread`、`max_background_threads`、`thread`、`config`、`opt`、`tcache`、`arena`、`arenas`、`prof`、`stats`、`experimental`；`super_root_node`（`src/ctl.c:637-639`）提供空前缀入口。`ctl_lookup` 以 `.` 分段（`src/ctl.c:1203-1204`），逐段沿 `named`/`indexed` 节点下降，空段返回 `ENOENT`（`src/ctl.c:1205-1207`）。`ctl_byname` 在 `!ctl_initialized` 时先 `ctl_init`，失败返回 `EAGAIN`（`src/ctl.c:1300-1303`），与 `je_mallctl` 的初始化失败语义一致。

## 5. 日志过滤与安全输出

`log_var_matches_segment`（`src/log.c:22`）先比较段长，单字符 `.` 段为通配（`src/log.c:29-31`），其余用 `strncmp` 精确匹配（`src/log.c:33-34`）；`log_var_extract_segment`（`src/log.c:14`）以 `|` 为分隔符切段。`malloc_vsnprintf` 使用局部 `APPEND_C`/`APPEND_S` 宏（`src/malloc_io.c:331-342`），即使截断也继续累计长度，保证返回值与标准 `vsnprintf` 一致。该实现避免分配路径中调用 libc 格式化函数造成重入，因此 `safety_check_fail`（`src/safety_check.c:10`）也复用它构造消息后直接 `abort()`。

## 6. 共变耦合与修改清单（Co-Change Playbook）

- 新增 mallctl 节点 → 同步 `root_node`（`src/ctl.c:622-636`）、`contracts/api/mallctl.md` 的逐项表与 [operations/runbook-and-observability.md](../operations/runbook-and-observability.md) 的采集示例。
- 改动 `ctl_lookup` 分段规则 → 同步 `ctl_nametomib`（`src/ctl.c:1324`）与 `ctl_bymib`（`src/ctl.c:1338`）及 [contracts/api-reference.md](../contracts/api-reference.md) 的名称格式说明。
- 改动 `STATS_PRINT_OPTIONS` → 同步 `stats_print`（`src/stats.c:1401-1403`）与统计子打印函数 `stats_arena_bins_print`（`src/stats.c:270`）、`stats_arena_extents_print`（`src/stats.c:548`）。
- 改动日志类别命名 → 同步 `log_var_matches_segment`（`src/log.c:22`）的 `.` 通配语义与 [operations/runbook-and-observability.md](../operations/runbook-and-observability.md) 的 `log` 配置串。
- 验证命令：`make tests && make check`；重点单测 `test/unit/ctl.c`、`test/unit/stats.c`。
- 隐式陷阱：所有 `mallctl` 写路径必须在 `ctl_mtx` 保护下更新缓存（`src/ctl.c:1122`），否则 `stats_print` 的 `ctl_refresh` 会读到撕裂视图。
