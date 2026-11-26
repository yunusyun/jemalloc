---
kind: error-dict
scope: error-codes
status: draft
origin: generated
source_paths: [src/jemalloc.c, src/safety_check.c, src/malloc_io.c, src/tcache.c]
symbols: [malloc_abort_invalid_conf, safety_check_fail, malloc_printf, imalloc_body]
upstream: [contracts/api-reference.md]
downstream: [operations/runbook-and-observability.md]
---

# 错误码与字面错误串字典

## 1. 三类错误出口

本仓无自定义错误码枚举；错误以 `errno` 风格 int、字面串写 stderr、以及 `abort()` 三种形式出现（完整导出面见 [入站 API 契约](api-reference.md)）。字面串全部经 `malloc_printf`（`src/malloc_io.c:659`，内部 `malloc_vcprintf`）或 `malloc_write`（`src/malloc_io.c:83-88`）输出；后者优先调用全局回调 `je_malloc_message`（`src/malloc_io.c:84-85`），为空时回落 `wrtmessage`（`:87`）。`safety_check_fail`（`src/safety_check.c:10-23`）先用 `malloc_vsnprintf` 格式化到 `MALLOC_PRINTF_BUFSIZE` 缓冲（`:15`），未安装自定义 `safety_check_abort`（`:4`、`:6-8`）时 `malloc_write(buf)` 后 `abort()`（`:18-20`），否则把消息交给回调（`:22`）。

| 出口 | 函数/锚点 | 终端行为 |
|---|---|---|
| errno 返回 | `imalloc_body`（`src/jemalloc.c:2028`）`label_oom` → `ENOMEM`（`:2190`、`:2197`）；`label_invalid_alignment` → `EINVAL`（`:2211`、`:2224`） | 由公共 API 决定是否在 `result` 写 NULL、是否设置 `errno` |
| 字面串 + abort | `malloc_abort_invalid_conf`（`src/jemalloc.c:898-903`） | `opt_abort_conf` 且存在配置错误时 `abort()`（`:1432-1434`） |
| 安全校验 | `safety_check_fail`（`src/safety_check.c:10`） | 默认 `abort()`；可经 `safety_check_set_abort` 替换 |

## 2. malloc 系列 OOM 与非法对齐字面串

下表只列源码中字面存在的字符串，不推断未出现的错误码。

| 触发 API | 字面串（原文） | 锚点 | 附加行为 |
|---|---|---|---|
| `malloc` | `<jemalloc>: Error in malloc(): out of memory\n` | `src/jemalloc.c:2283` | `set_errno_on_error=true` + `null_out_result_on_error=true`（`:2281-2282`） |
| `calloc` | `<jemalloc>: Error in calloc(): out of memory\n` | `src/jemalloc.c:2487` | `may_overflow=true`，乘法溢出走 `label_oom`（`:2049-2052`） |
| `posix_memalign` | `<jemalloc>: Error allocating aligned memory: out of memory\n` | `src/jemalloc.c:2407` | `min_alignment=sizeof(void*)`（`:2405`） |
| `posix_memalign` | `<jemalloc>: Error allocating aligned memory: invalid alignment\n` | `src/jemalloc.c:2409` | 2 的幂与非零检查在 `src/jemalloc.c:2054-2057` |
| `aligned_alloc` | 同上 OOM 串 | `src/jemalloc.c:2450` | `min_alignment=1`（`:2448`） |
| `aligned_alloc` | 同上 invalid alignment 串 | `src/jemalloc.c:2452` | `bump_empty_aligned_alloc=true`（`:2445`） |
| `memalign` | 同上 OOM 串 | `src/jemalloc.c:2898` | 条件导出 `JEMALLOC_OVERRIDE_MEMALIGN` |
| `memalign` | 同上 invalid alignment 串 | `src/jemalloc.c:2900` | 走 `imalloc`（`:2908`） |
| `valloc` | 同上 OOM 串 | `src/jemalloc.c:2938` | `min_alignment=PAGE`（`:2936`） |
| `valloc` | 同上 invalid alignment 串 | `src/jemalloc.c:2940` | 条件导出 `JEMALLOC_OVERRIDE_VALLOC` |
| `realloc` | `<jemalloc>: Error in realloc(): out of memory\n` | `src/jemalloc.c:2730`、`:2748-2749` | 后者是 `xmalloc` 打开时的 `abort()` 分支；本仓**无** realloc 专用 invalid-alignment 字面串 |
| `mallocx` | `<jemalloc>: Error in mallocx(): out of memory\n` | `src/jemalloc.c:3056`、`:3110` | `:3056` 属实验性 `smallocx`，`:3110` 属公共 `mallocx` |
| `rallocx` | `<jemalloc>: Error in rallocx(): out of memory\n` | `src/jemalloc.c:3301` | `label_oom` 在 `:3299` |

## 3. 初始化、配置与线程相关字面串

| 字面串（原文） | 锚点 | 条件 |
|---|---|---|
| `<jemalloc>: Abort (abort_conf:true) on invalid conf value (see above).\n` | `src/jemalloc.c:900-901` | `malloc_abort_invalid_conf`；`assert(opt_abort_conf)`（`:899`） |
| `<jemalloc>: Conf string ends with key\n` | `src/jemalloc.c:855-856` | `malloc_conf_next` 解析到 `\0` 且仍有键 |
| `<jemalloc>: Malformed conf string\n` | `src/jemalloc.c:860` | 键/值含非法字符 |
| `<jemalloc>: Conf string ends with comma\n` | `src/jemalloc.c:877-878` | 值后紧跟结尾逗号 |
| `<jemalloc>: %s: %.*s:%.*s\n` | `src/jemalloc.c:908` | `malloc_conf_error` 通用格式；`experimental_` 前缀被豁免（`:911-915`） |
| `<jemalloc>: malloc_conf #%u (%s): "%s"\n` | `src/jemalloc.c:1042-1044` | `opt_confirm_conf` 且非首次解析 |
| `<jemalloc>: -- Set conf value: %.*s:%.*s\n` | `src/jemalloc.c:1062-1065` | `CONF_CONTINUE` 宏 |
| `<jemalloc>: Error in atexit()\n` | `src/jemalloc.c:1512` | `atexit(stats_print_atexit)` 失败；`opt_abort` 时 abort（`:1513-1515`） |
| `<jemalloc>: Error in pthread_atfork()\n` | `src/jemalloc.c:1586` | 钩子注册失败，返回 true 终止初始化（`:1590`） |
| `<jemalloc>: error in background thread creation for arena %u. Abort.\n` | `src/jemalloc.c:356-357` | `arena_new_create_background_thread` 在 `ind != 0` 且非 huge arena 时创建失败 |
| `<jemalloc>: perCPU arena getcpu() not available. Setting narenas to %u.\n` | `src/jemalloc.c:1634-1636` | `!have_percpu_arena || malloc_getcpu() < 0` |
| `<jemalloc>: Reducing narenas to limit (%d)\n` | `src/jemalloc.c:1691` | `narenas_auto >= MALLOCX_ARENA_LIMIT` |
| `<jemalloc>: size mismatch in thread cache detected, likely caused by sized deallocation bugs by application. Abort.\n` | `src/tcache.c:126-128` | `tbin_extents_lookup_size_check` 中 `sz_sum != 0`（`:125`）；随后 `abort()`（`:129`） |

## 4. errno 映射与符号级调用树

```text
公共 API -> src/jemalloc.c:2246 imalloc -> src/jemalloc.c:2028 imalloc_body
  ├─ [乘法溢出 :2049] -> label_oom(:2177)
  │    ├─ [config_xmalloc && opt_xmalloc :2178] malloc_write(oom_string) -> abort()
  │    └─ set_errno(ENOMEM)(:2190) -> return ENOMEM(:2197)
  ├─ [对齐 < min_alignment 或非 2 的幂 :2054] -> label_invalid_alignment(:2204)
  │    ├─ [config_xmalloc && opt_xmalloc :2205] malloc_write(invalid_alignment_string) -> abort()
  │    └─ set_errno(EINVAL)(:2211) -> return EINVAL(:2224)
  └─ 成功 -> *dopts->result = allocation(:2174) -> return 0
mallctl/mallctlnametomib/mallctlbymib -> malloc_init() 失败 -> EAGAIN(:3654/:3674/:3695)
安全校验失败 -> src/safety_check.c:10 safety_check_fail -> malloc_write(:19) -> abort(:20)
```

注意 `src/tcache.c:126` 的 size mismatch 分支在 `safety_check_fail` 之后又写了一次 `abort()`（`src/tcache.c:129`）：若通过 `safety_check_set_abort` 安装了不终止的回调，仍会硬终止；这是字面可见的冗余保护。

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 修改任一 OOM/invalid 字面串：同步 `src/jemalloc.c:2283 / 2407 / 2409 / 2450 / 2452 / 2487 / 2730 / 2748 / 2898 / 2900 / 2938 / 2940 / 3056 / 3110 / 3301`、本页表格与 [排障页](../operations/runbook-and-observability.md)的日志特征列。
2. 改 `errno` 语义：联动 `imalloc_body` 的 `label_oom`/`label_invalid_alignment`（`src/jemalloc.c:2177-2224`）、`imalloc_init_check`（`:2227-2242`）以及 API 层 `sopts.set_errno_on_error` / `null_out_result_on_error` 组合。
3. 改输出通道：联动 `malloc_printf`（`src/malloc_io.c:659`）、`malloc_write`（`:83-88`）与导出全局 `malloc_message`（`include/jemalloc/jemalloc_protos.h.in:7-8`）；自定义回调若非线程安全会影响全部错误串。
4. 改安全校验：联动 `src/safety_check.c:4-23` 的 `safety_check_abort`、`src/tcache.c:126-129` 的调用点，以及 `include/jemalloc/internal/jemalloc_preamble.h.in:164-181` 的 `config_opt_safety_checks` 编译开关。
5. 初始化期字符串变更联动 `src/jemalloc.c:356 / 900 / 1512 / 1586 / 1634 / 1691`，并复核 `opt_abort` 决定是否真的终止进程。
