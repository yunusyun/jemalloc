---
kind: operations
scope: runbook-and-observability
status: draft
origin: generated
source_paths: [src/stats.c, src/log.c, src/malloc_io.c, src/ctl.c, Makefile.in]
symbols: [stats_print, log_var_matches_segment, malloc_printf, ctl_byname, check]
upstream: [contracts/error-codes.md]
downstream: [operations/boundary.md]
---

# 排障与可观测操作手册

## 1. 三个真实信号源

| 信号源 | 锚点 | 产出 |
|---|---|---|
| stderr 字面串 | `malloc_printf`（`src/malloc_io.c:659-665`）→ `malloc_vcprintf`；`malloc_write`（`src/malloc_io.c:83-88`）优先 `je_malloc_message`（`:84-85`） | OOM、配置错误、初始化失败，见 [错误码字典](../contracts/error-codes.md) |
| mallctl 快照 | `ctl_byname`（`src/ctl.c:1292-1321`）与 `ctl_refresh`（`src/ctl.c:1044-1115`） | `stats.*` 一致快照，`epoch` 为版本戳 |
| 统计渲染 | `stats_print`（`src/stats.c:1396-1457`） | 表/JSON 输出，包含 `___ Begin jemalloc statistics ___`（`:1443`）与 `--- End jemalloc statistics ---`（`:1455`）边界 |

`stats_print` 先以 `je_mallctl("epoch", ...)` 强制刷新（`src/stats.c:1414-1415`）：`EAGAIN` 时输出 `<jemalloc>: Memory allocation failure in mallctl("epoch", ...)` 并返回（`:1417-1420`），其他错误输出 `<jemalloc>: Failure in mallctl("epoch", ...)` 后 `abort()`（`:1422-1424`）。因此「统计打印直接失败」几乎等价于「ctl 初始化/分配失败」。

## 2. 症状 → 根因决策树

```text
S1 stderr 出现 "<jemalloc>: Error in malloc(): out of memory"
  -> 字面串 src/jemalloc.c:2283；errno 来源 src/jemalloc.c:2190 ENOMEM
  -> 分支 A: stats.allocated/active 接近 resident -> 真实内存压力或泄漏
  -> 分支 B: retained 高而 active 低 -> 页被保留未归还 OS，查 decay 配置
S2 stderr 出现 "<jemalloc>: Abort (abort_conf:true) on invalid conf"
  -> malloc_abort_invalid_conf src/jemalloc.c:898-903
  -> 查四源优先级 src/jemalloc.c:951-1012，确认环境变量是否覆盖构建期值
S3 mallctl("...") 返回 ENOENT
  -> ctl_lookup src/ctl.c:1206/1232/1242/1277/1316 -> 名称拼错或路径只到中间节点
  -> 对照 root_node src/ctl.c:622-636 的 13 个根键
S4 mallctl 返回 EAGAIN
  -> src/jemalloc.c:3654/3674/3695 或 src/ctl.c:1301/1328/1345
  -> 分配器/ctl 初始化失败，通常伴随 OOM 字面串
S5 tcache size mismatch 并 abort
  -> src/tcache.c:126-128 safety_check_fail -> src/safety_check.c:10-23
  -> 应用 sized deallocation 参数错误；检查 dallocx/sdallocx 的 size 实参
S6 统计输出缺 bins/ extents / mutex 段落
  -> src/stats.c:1427-1435 的 opts 逐字符开关；对照 STATS_PRINT_OPTIONS
  -> include/jemalloc/internal/stats.h:5-14 的 J/g/m/d/a/b/l/x/e
S7 日志类别未按预期打印/抑制
  -> src/log.c:22 log_var_matches_segment；'.' 段匹配一切(:30-32)
  -> 前缀匹配规则 :33-37；日志名缓冲 src/log.c:6
```

`log_var_matches_segment`（`src/log.c:22-40`）规则为：单字符 `.` 段匹配任意日志名（`:29-32`）；等长时 `strncmp` 全等（`:33-34`）；段短于日志名时要求前缀相等且日志名下一字符为 `.`（`:35-37`）；段更长则不匹配（`:38-39`）。段边界由 `log_var_extract_segment`（`src/log.c:14-19`）以 `|` 或 `\0` 切分。配置键 `log` 把值写入 `log_var_names`（`src/jemalloc.c:1383-1391`）；运行期分类名可在日志语句中直接读到，如 `LOG("core.malloc.entry", ...)`（`src/jemalloc.c:2324`）。

## 3. 统计维度与采集矩阵

`STATS_PRINT_OPTIONS`（`include/jemalloc/internal/stats.h:5-14`）逐字符开关如下，渲染分派见 `src/stats.c:1427-1435`、`:1446-1452`。

| 选项字符 | 变量 | 默认 | 关闭时的可见后果 |
|---|---|---|---|
| `J` | `json` | false | 使用表格式 emitter（`src/stats.c:1439-1441`） |
| `g` | `general` | true | 跳过 `stats_general_print`（`src/stats.c:1446-1448`、`:985`） |
| `m` | `merged` | `config_stats` | 跳过合并视图 |
| `d` | `destroyed` | `config_stats` | 跳过已销毁 arena 视图 |
| `a` | `unmerged` | `config_stats` | 跳过未合并视图 |
| `b` | `bins` | true | 跳过 `stats_arena_bins_print`（`src/stats.c:270`） |
| `l` | `large` | true | 跳过 large 分配统计 |
| `x` | `mutex` | true | 跳过 `stats_arena_mutexes_print`（`src/stats.c:636`） |
| `e` | `extents` | true | 跳过 `stats_arena_extents_print`（`src/stats.c:548`） |

采集入口有三条：`malloc_stats_print` → `stats_print`（`src/jemalloc.c:3708-3716`）；`MALLOC_CONF=stats_print:true`（`src/jemalloc.c:1252`，进程退出时由 `atexit(stats_print_atexit)` 注册，`:1509-1516`）；`mallctl("stats.arenas...")` 直接读 `stats_node`（`src/ctl.c:581-593`）。`stats_print_helper`（`src/stats.c:1233`）承担分派。

## 4. 回归验证命令

`Makefile.in:553` 的 `check` 聚合 `check_unit`、`check_integration`、`check_integration_decay`、`check_integration_prof`。具体命令：`make check_unit`（`Makefile.in:539-540`）、`make check_integration`（`:549-550`）、`make check`（`:553`）；decay 与 prof 变体分别注入 `dirty_decay_ms:-1,muzzy_decay_ms:-1` / `:0,:0`（`:546-548`）与 `prof:true` / `prof:true,prof_active:false`（`:541-545`）。测试脚本 `test/test.sh.in:26-33` 定义 pass=0、skip=1、fail=2，汇总行见 `:74`，失败时 `exit 1`（`:76-80`）。

| 排障目标 | 命令 | 判据 |
|---|---|---|
| 验证配置键可解析 | `make check_unit` | `test/unit/mallctl.c:6-41` 的 `ENOENT/EPERM/EINVAL` 断言通过 |
| 验证 decay 组合 | `make check_integration_decay` | 两组 `MALLOC_CONF` 下均无 fail（`Makefile.in:546-548`） |
| 验证 prof 组合 | `make check_integration_prof` | `enable_prof=1` 时两组配置通过（`:541-545`） |
| 一次性全量 | `make check` | 汇总 `fail: 0`（`test/test.sh.in:74`） |

注意 `run_tests.sh:1` 畸形、`scripts/gen_run_tests.py:19` 使用 Python 2 `xrange`，两者不能作为通过证据；详见 [开发与测试](dev-and-test.md)。运行时无法跨越的边界见 [边界页](boundary.md)。

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 改统计输出：联动 `stats_print`（`src/stats.c:1396`）、`STATS_PRINT_OPTIONS`（`include/jemalloc/internal/stats.h:5-14`）、三个 `stats_arena_*_print`（`src/stats.c:270 / 548 / 636`）与本页选项矩阵。
2. 改 epoch/刷新：联动 `epoch_ctl`（`src/ctl.c:1616-1632`）、`ctl_refresh`（`:1044`）、`src/stats.c:1414-1425` 的错误分支与决策树 S1/S4。
3. 改日志匹配：联动 `src/log.c:14-40` 的 `log_var_extract_segment`/`log_var_matches_segment`、`src/jemalloc.c:1383-1391` 的 `log` 键写入点及本页 S7。
4. 改错误串：联动 [错误码字典](../contracts/error-codes.md)、`src/malloc_io.c:83-88 / 659` 输出通道与本页 S1/S2/S5。
5. 改验证矩阵：联动 `Makefile.in:539-553`、`test/test.sh.in:26-80` 的结果码语义与 [配置字典](../contracts/configuration.md)的键名有效性。
