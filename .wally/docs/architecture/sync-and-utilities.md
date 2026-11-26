---
kind: component
scope: sync-and-utilities
status: draft
origin: generated
source_paths: [src/mutex.c, src/witness.c, src/bitmap.c, src/ckh.c, src/nstime.c, src/safety_check.c, src/test_hooks.c]
symbols: [malloc_mutex_lock, malloc_mutex_unlock, witness_init, bitmap_info_init, ckh_new, nstime_update, safety_check_fail]
upstream: [architecture/allocator-core.md]
downstream: [operations/pitfalls-and-adr.md]
---

# 同步与基础设施（sync-and-utilities）

## 1. 角色与边界

本组件提供全仓共用的锁原语、锁序校验（witness）、位图、链式哈希表、单调时间源、不可恢复错误出口与测试钩子。它被 [architecture/allocator-core.md](allocator-core.md) 及 [architecture/tcache-and-tsd.md](tcache-and-tsd.md) 等所有组件复用；已知陷阱与架构决策记录见 [operations/pitfalls-and-adr.md](../operations/pitfalls-and-adr.md)。

## 2. 符号级调用树（加锁与锁序校验）

```text
malloc_mutex_lock include/jemalloc/internal/mutex.h:217
  ├─ witness_assert_not_owner(tsdn_witness_tsdp_get(tsdn), &mutex->witness)
  ├─ [isthreaded] malloc_mutex_trylock_final -> malloc_mutex_lock_slow
  ├─ mutex_owner_stats_update
  └─ witness_lock(...)

malloc_mutex_unlock include/jemalloc/internal/mutex.h:230
  ├─ atomic_store_b(&mutex->locked, false)
  ├─ witness_unlock(...)
  └─ [isthreaded] MALLOC_MUTEX_UNLOCK

witness_init src/witness.c:9 -> 写入 name/rank/comp/opaque
  ├─ malloc_mutex_init src/mutex.c:135 调用（src/mutex.c:175 与 src/mutex.c:178）
  └─ 锁序反转 -> witness_lock_error_impl src/witness.c:18 -> malloc_printf

安全出口：
  safety_check_fail src/safety_check.c:10 -> malloc_vsnprintf -> malloc_write -> abort()
                                           └─ [自定义] safety_check_abort(buf)
```

`malloc_mutex_lock`/`malloc_mutex_unlock` 在所列 `src/*.c` 中 `[仓库内未找到字面实现]`：二者逐字定义于 `include/jemalloc/internal/mutex.h:217`/`:230`（`static inline`）；`src/mutex.c:186`/`:191` 只是 `malloc_mutex_prefork`/`malloc_mutex_postfork_parent` 中的调用点，已据实改引。

## 3. 源码锚点与职责契约表

| 文件:行号 | 符号 | 职责 | 入 → 出 |
|---|---|---|---|
| `src/mutex.c:135` | `malloc_mutex_init` | 初始化锁与内嵌 witness | `mutex_t*`,`name`,`rank` → `bool` 错误 |
| `src/witness.c:9` | `witness_init` | 绑定锁名/秩/比较器 | `witness_t*` → `void` |
| `src/bitmap.c:12` | `bitmap_info_init` | 树模式下计算分组偏移 | `bitmap_info_t*`,`nbits` → `void` |
| `src/ckh.c:360` | `ckh_new` | 按最小条目数建哈希表 | `minitems`,`hash`,`keycomp` → `bool` 错误 |
| `src/nstime.c:170` | `nstime_update` | 可替换的时间刷新函数指针 | `nstime_t*` → `bool` 回退 |
| `src/safety_check.c:10` | `safety_check_fail` | 格式化消息后中止或回调 | `format`,`...` → 不返回 |
| `src/test_hooks.c:9` | `test_hooks_arena_new_hook` | arena 新建测试钩子变量 | `void (*)()` 函数指针 |

`src/test_hooks.c` 仅 12 行，导出 `test_hooks_arena_new_hook`（`:9`）与 `test_hooks_libc_hook`（`:12`）两个 `JEMALLOC_EXPORT` 函数指针，初值均为 `NULL`。

## 4. 时间源、位图与哈希表

`nstime_update` 是 `nstime_update_t *JET_MUTABLE` 函数指针（`src/nstime.c:170`），默认指向 `nstime_update_impl`（`src/nstime.c:156`）。后者先 `nstime_copy` 旧值再 `nstime_get`，若时钟回退则保留旧值并返回 `true`（`src/nstime.c:159-166`）；单测通过替换该指针注入假时钟，见 [operations/pitfalls-and-adr.md](../operations/pitfalls-and-adr.md)。

`bitmap_info_init` 在 `#ifdef BITMAP_USE_TREE` 分支（`src/bitmap.c:9-12`）计算 `levels[i].group_offset`，非树分支另有实现。`ckh_new` 断言 `minitems > 0`、`hash != NULL`、`keycomp != NULL`（`src/ckh.c:366-368`），再按 `lg_mincells` 分配并初始化 cell 数组；prof 用它维护 `bt2tctx` 映射。

## 5. 不可恢复错误与测试钩子

`safety_check_fail`（`src/safety_check.c:10`）用 `malloc_vsnprintf` 格式化到 `MALLOC_PRINTF_BUFSIZE` 栈缓冲，若未安装 `safety_check_abort` 则 `malloc_write(buf)` 后 `abort()`（`src/safety_check.c:18-20`），否则把消息交给自定义回调（`src/safety_check.c:22`）。安装入口 `safety_check_set_abort` 位于 `src/safety_check.c:6`。tcache 尺寸不一致即通过此路径上报：`tbin_extents_lookup_size_check` 在 `src/tcache.c:125-129` 检测 `sz_sum != 0` 后调用 `safety_check_fail` 并 `abort()`。

`test_hooks_arena_new_hook`（`src/test_hooks.c:9`）由测试代码注入，在 `arena_new` 路径触发；生产构建中保持 `NULL`，不影响锁序。

## 6. 共变耦合与修改清单（Co-Change Playbook）

- 改动 `malloc_mutex_t` 布局 → 同步 `include/jemalloc/internal/mutex.h:217`/`:230`、`src/mutex.c:135` 与所有 `malloc_mutex_init` 调用点（如 `src/arena.c:1977`）。
- 改动 witness 秩枚举 → 同步 `src/witness.c:9`、`src/witness.c:18` 的错误输出、`src/base.c:366` 的 `WITNESS_RANK_BASE` 及 [operations/pitfalls-and-adr.md](../operations/pitfalls-and-adr.md) 的锁序表。
- 改动位图层级上限 → 同步 `src/bitmap.c:12` 与 `bin_infos_init`（`src/bin.c:23`）的 `BITMAP_INFO_INITIALIZER`。
- 改动 `nstime_update` 指针语义 → 同步 `src/nstime.c:170`、decay 节拍（[business/wf-decay.md](../business/wf-decay.md)）与 `test/unit/decay.c` 的 mock 替换点。
- 验证命令：`make tests && make check`；重点单测 `test/unit/mutex.c`、`test/unit/witness.c`、`test/unit/bitmap.c`、`test/unit/ckh.c`、`test/unit/nstime.c`。
- 隐式陷阱：`malloc_mutex_lock` 在持锁前后各做一次 witness 操作（`include/jemalloc/internal/mutex.h:218`/`226`）；新增直接操作 `mutex->locked` 的代码会绕过锁序校验，必须避免。
