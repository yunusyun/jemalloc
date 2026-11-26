---
kind: api-contract
scope: inbound-api
status: draft
origin: generated
source_paths: [include/jemalloc/jemalloc_protos.h.in, src/jemalloc.c, include/jemalloc/jemalloc_macros.h.in, bin/jemalloc-config.in]
symbols: [je_malloc, je_calloc, je_free, je_realloc, je_mallocx, je_mallctl, MALLOCX_LG_ALIGN_MASK, usage]
upstream: [INDEX.md]
downstream: [contracts/api/mallctl.md, contracts/error-codes.md, architecture/allocator-core.md]
---

# 入站 API 契约：导出 C 接口、全局变量与命令行工具

## 1. 边界定性

本仓产出 `libjemalloc` 库，不是网络服务；无 HTTP/RPC/MQ/DB 入站端口。「入站 API」= 公共 C 符号 + mallctl 命名空间 + 3 个命令行工具。导出面以 `include/jemalloc/jemalloc_protos.h.in:6-66` 为唯一声明源，`@je_@` 前缀由 `JEMALLOC_NO_DEMANGLE` / `jemalloc_mangle@install_suffix@.h` 决定（`include/jemalloc/jemalloc_protos.h.in:2-5`），本文符号统一写 `je_` 前缀。实现集中于 `src/jemalloc.c`；所有以 `.in` 结尾的 CLI 均为安装时替换 `@prefix@` 等变量的模板，未替换前不可直接运行。

## 2. 全部 20 个公共 C API

| # | API（声明） | 声明锚点 | 实现锚点 | 语义与可观测差值 |
|---|---|---|---|---|
| 1 | `malloc(size_t)` | `include/jemalloc/jemalloc_protos.h.in:10-12` | `src/jemalloc.c:2323` | 快路径 `tsd_fast && size<=SC_LOOKUP_MAXCLASS` 命中 `cache_bin_alloc_easy`；未命中尾调用 `malloc_default`（`src/jemalloc.c:2271`）；OOM 串 `src/jemalloc.c:2283` |
| 2 | `calloc(size_t, size_t)` | `include/jemalloc/jemalloc_protos.h.in:13-15` | `src/jemalloc.c:2474` | `may_overflow=true`、`zero=true`（`:2484-2492`）；OOM 串 `:2487` |
| 3 | `posix_memalign(void **, size_t, size_t)` | `include/jemalloc/jemalloc_protos.h.in:16-17` | `src/jemalloc.c:2393` | 返回 `errno` 风格 int；`min_alignment=sizeof(void*)`（`:2405`）；OOM `:2407`、非对齐 `:2409` |
| 4 | `aligned_alloc(size_t, size_t)` | `include/jemalloc/jemalloc_protos.h.in:18-21` | `src/jemalloc.c:2433` | `min_alignment=1`、`bump_empty_aligned_alloc=true`（`:2445-2448`）；OOM `:2450`、非对齐 `:2452` |
| 5 | `realloc(void*, size_t)` | `include/jemalloc/jemalloc_protos.h.in:22-24` | `src/jemalloc.c:2653` | `size==0` 等价 `free`（`:2662-2683`）；`ptr==NULL` 等价 `malloc`（`:2720-2743`）；OOM `:2730` 与 `:2748` |
| 6 | `free(void*)` | `include/jemalloc/jemalloc_protos.h.in:25-26` | `src/jemalloc.c:2863` | `free_fastpath`（`:2806`）失败转 `free_default`（`:2771`） |
| 7 | `mallocx(size_t, int flags)` | `include/jemalloc/jemalloc_protos.h.in:28-30` | `src/jemalloc.c:3098` | flags 解码 `:3115-3135`；OOM 串 `:3110` |
| 8 | `rallocx(void*, size_t, int flags)` | `include/jemalloc/jemalloc_protos.h.in:31-33` | `src/jemalloc.c:3216` | 走 `iralloct`；`label_oom` 在 `:3299`，xmalloc 串 `:3301` |
| 9 | `xallocx(void*, size_t, size_t extra, int flags)` | `include/jemalloc/jemalloc_protos.h.in:34-35` | `src/jemalloc.c:3390` | 原地扩张，`extra` 被 `SC_LARGE_MAXCLASS` 截断（`:3426-3428`），返回实际 `usize`（`:3456`） |
| 10 | `sallocx(const void*, int flags)` | `include/jemalloc/jemalloc_protos.h.in:36-37` | `src/jemalloc.c:3461` | `ivsalloc`（debug/force_ivsalloc）或 `isalloc`（`:3473-3478`），`JEMALLOC_ATTR(pure)` |
| 11 | `dallocx(void*, int flags)` | `include/jemalloc/jemalloc_protos.h.in:38` | `src/jemalloc.c:3487` | tcache 选择 `:3498-3517` → `ifree`（`:2556`） |
| 12 | `sdallocx(void*, size_t, int flags)` | `include/jemalloc/jemalloc_protos.h.in:39-40` | `src/jemalloc.c:3594` | 带 `size_hint=true` 的 `free_fastpath`（`:3598`），失败转 `sdallocx_default`（`:3548`） |
| 13 | `nallocx(size_t, int flags)` | `include/jemalloc/jemalloc_protos.h.in:41-42` | `src/jemalloc.c:3619` | `inallocx`（`:3533`）计算 `usize`，超 `SC_LARGE_MAXCLASS` 返回 0（`:3634-3637`） |
| 14 | `mallctl(name, oldp, oldlenp, newp, newlen)` | `include/jemalloc/jemalloc_protos.h.in:44-45` | `src/jemalloc.c:3645` | `ctl_byname`（`:3659`）；初始化失败返回 `EAGAIN`（`:3654`） |
| 15 | `mallctlnametomib(name, mibp, miblenp)` | `include/jemalloc/jemalloc_protos.h.in:46-47` | `src/jemalloc.c:3667` | `ctl_nametomib`（`:3679`）；`EAGAIN` 见 `:3674` |
| 16 | `mallctlbymib(mib, miblen, ...)` | `include/jemalloc/jemalloc_protos.h.in:48-49` | `src/jemalloc.c:3687` | `ctl_bymib`（`:3701`）；`EAGAIN` 见 `:3695` |
| 17 | `malloc_stats_print(write_cb, cbopaque, opts)` | `include/jemalloc/jemalloc_protos.h.in:50-52` | `src/jemalloc.c:3708` | 转 `stats_print`（`:3716`） |
| 18 | `malloc_usable_size(void *ptr)` | `include/jemalloc/jemalloc_protos.h.in:53-54` | `src/jemalloc.c:3722` | `NULL` 返回 0（`:3733-3734`），否则 `isalloc`/`ivsalloc` |
| 19 | `memalign(size_t, size_t)` | `include/jemalloc/jemalloc_protos.h.in:56-60` | `src/jemalloc.c:2885` | 条件导出 `JEMALLOC_OVERRIDE_MEMALIGN`；OOM `:2898`、非对齐 `:2900` |
| 20 | `valloc(size_t)` | `include/jemalloc/jemalloc_protos.h.in:62-66` | `src/jemalloc.c:2924` | 条件导出 `JEMALLOC_OVERRIDE_VALLOC`；`min_alignment=PAGE`（`:2936`） |

## 3. 导出全局变量与 fork 钩子

| 类别 | 符号 | 锚点 | 作用 |
|---|---|---|---|
| 全局变量 | `malloc_conf` | `include/jemalloc/jemalloc_protos.h.in:6` | 编译期嵌入配置串，4 源优先级中的第 2 源 |
| 全局变量 | `malloc_message` | `include/jemalloc/jemalloc_protos.h.in:7-8` | 输出回调；`malloc_write` 在 `src/malloc_io.c:84-85` 先判空再调用 |
| 构造函数 | `jemalloc_constructor` | `src/jemalloc.c:3774` | `JEMALLOC_ATTR(constructor)`，提前 `malloc_init()` 规避 fork/初始化竞态（注释 `:3758-3770`） |
| fork 前 | `jemalloc_prefork` | `src/jemalloc.c:3781` | 按 8 阶段遍历所有 arena 持锁（`:3815-3848`） |
| fork 后（父） | `jemalloc_postfork_parent` | `src/jemalloc.c:3855` | 释放锁（`:3877-3890`） |
| fork 后（子） | `jemalloc_postfork_child` | `src/jemalloc.c:3894` | 子进程重初始化锁（`:3906-3919`） |
| 注册点 | `pthread_atfork` | `src/jemalloc.c:1584-1586` | 注册上述三钩子；失败写 `<jemalloc>: Error in pthread_atfork()` |

## 4. flags 位解码与符号级调用树

`MALLOCX_LG_ALIGN_MASK` **不在** `include/jemalloc/jemalloc_macros.h.in` 中——其字面定义在 `include/jemalloc/internal/jemalloc_internal_types.h:38`（`((1 << MALLOCX_LG_ALIGN_BITS) - 1)`，`MALLOCX_LG_ALIGN_BITS=6` 见 `:28`）；公共宏文件给出的是 `MALLOCX_LG_ALIGN`（`include/jemalloc/jemalloc_macros.h.in:15`）、`MALLOCX_ZERO`（`:23`）、`MALLOCX_TCACHE`/`MALLOCX_TCACHE_NONE`（`:28-29`）、`MALLOCX_ARENA`（`:33`）、`MALLCTL_ARENAS_ALL`（`:47`）与 `MALLCTL_ARENAS_DESTROYED`（`:52`）。掩码在 `src/jemalloc.c:3063`、`:3116`、`:3538` 被读取。

```text
malloc(size) -> src/jemalloc.c:2323 je_malloc
  ├─ [tsd_fast && size<=SC_LOOKUP_MAXCLASS] cache_bin_alloc_easy(:2371) -> 返回
  ├─ [ticker 触发] -> malloc_default(:2271)
  └─ [采样触发] -> malloc_default(:2271)
       -> imalloc(:2246) -> imalloc_body(:2028)
            ├─ compute_size_with_overflow(:1990) [guard: 乘法溢出] -> label_oom
            ├─ [guard: 对齐非法] -> label_invalid_alignment(:2204) -> EINVAL
            └─ imalloc_no_sample(:1912) -> [outbound] tcache/arena
mallctl(name,...) -> src/jemalloc.c:3645 -> ctl_byname(:3659) -> src/ctl.c:1292
mallocx/rallocx/nallocx -> flags -> MALLOCX_LG_ALIGN_MASK(:38) 解码
```

## 5. 三个命令行工具

| 工具 | 模板锚点 | 真实行为 |
|---|---|---|
| `jemalloc-config` | `bin/jemalloc-config.in:3` (`usage`) | `case "$1"` 分派 `:30-83`，输出 `--version/--revision/--config/--prefix/--bindir/--datadir/--includedir/--libdir/--mandir/--cc/--cflags/--cppflags/--cxxflags/--ldflags/--libs`，未知参数打印 usage 并 `exit 1`（`:80-82`） |
| `jemalloc.sh` | `bin/jemalloc.sh.in:7-9` | 设置 `@LD_PRELOAD_VAR@=${libdir}/libjemalloc.@SOREV@` 后 `exec "$@"`，用于临时 LD_PRELOAD 注入 |
| `jeprof` | `bin/jeprof.in:1` | `#! /usr/bin/env perl`，Perl 脚本；依赖安装后的 `libjemalloc` 产出做堆转储分析，本仓不含服务端实现 |

下游接口细节见 [mallctl 命名空间](api/mallctl.md)，错误值对应见 [错误码字典](error-codes.md)，分配器实现见 [分配核心](../architecture/allocator-core.md)。

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 增删公共 API：同步 `include/jemalloc/jemalloc_protos.h.in:6-66` 声明、`src/jemalloc.c` 实现、`include/jemalloc/internal/private_namespace.h` 名称改写规则、[mallctl 页](api/mallctl.md)的可观测入口与单测 `test/integration/malloc.c:1-14`。
2. 修改 `flags` 语义：联动 `include/jemalloc/jemalloc_macros.h.in:15-33`、内部掩码 `include/jemalloc/internal/jemalloc_internal_types.h:28-49`、`src/jemalloc.c:3115-3135 / 3533-3545`，并复核 `MALLOCX_ARENA_LIMIT` 对 `narenas` 的夹紧（`src/jemalloc.c:1689-1693`）。
3. 改动 OOM/对齐字符串：必须回头更新 [错误码字典](error-codes.md)的字面表，并核对 `src/jemalloc.c:2283 / 2407 / 2450 / 2487 / 2730 / 2748 / 2898 / 2938 / 3056 / 3110 / 3301`。
4. fork 钩子变更：联动 `src/jemalloc.c:1584` 注册点与 `src/mutex.c:184-204` 的 `malloc_mutex_prefork`/`malloc_mutex_postfork_*` 全锁遍历。
5. CLI 变更：联动 `bin/jemalloc-config.in:3-83` 与 `Makefile.in:587-589` 的 `distclean` 产物清单，避免残留过期模板。
