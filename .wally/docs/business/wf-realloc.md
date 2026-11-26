---
kind: workflow
scope: WF-realloc
status: draft
origin: generated
source_paths: [src/jemalloc.c, src/large.c, src/rtree.c]
symbols: [je_realloc, je_rallocx, je_xallocx, large_ralloc, rtree_leaf_elm_lookup_hard]
upstream: [business/index.md, contracts/api-reference.md]
downstream: [business/domain-models/extent.md]
---

# 重分配业务流（WF-realloc）

本页覆盖 `realloc`／`rallocx`／`xallocx` 三条入口如何借助 rtree 反查旧尺寸，并在「原位收缩／扩展」与「搬移复制」之间决断。入口约定见[业务总览](index.md)与[API 契约](../contracts/api-reference.md)，落点内存结构见[extent 领域模型](domain-models/extent.md)。

## 1. 入口语义矩阵

| 入口 | 定义锚点 | 入参 | 返回值语义 | 特殊分支 |
| --- | --- | --- | --- | --- |
| `je_realloc(ptr, size)` | `src/jemalloc.c:2653` | `size == 0` 视作释放（`2662-2680`）；`ptr == NULL` 视作 `malloc`（`2720-2743`） | 新指针或 `NULL` | `realloc(ptr,0)` 在 `2682` 把 `size` 置 1 仅用于 NULL 路径 |
| `je_rallocx(ptr, size, flags)` | `src/jemalloc.c:3216` | flags 解出对齐／zero／arena（`3221-3224`） | 新指针或 `NULL` | `MALLOCX_ARENA_MASK` 命中时 `3238` `arena_get` |
| `je_xallocx(ptr, size, extra, flags)` | `src/jemalloc.c:3390` | `size + extra` 允许溢出豁免，但需 clamp（`3414-3428`） | 实际 `usize`（`3456`），失败返回旧 `usize`（`3437-3439`） | 永不移动指针 |

三者共同的前置动作是 `src/jemalloc.c:2697-2700`／`3408-3412` 的 `rtree_szind_slab_read`，从指针反推 `alloc_ctx.szind` 并换算 `old_usize = sz_index2size(...)`。

## 2. 符号级调用树

```text
je_realloc(ptr, size) -> src/jemalloc.c:2653 je_realloc(ptr, arg_size)
  ├─ [size==0 && ptr!=NULL] src/jemalloc.c:2677 ifree(...) -> NULL       // 等价于 free
  ├─ [ptr==NULL] src/jemalloc.c:2736 imalloc(&sopts, &dopts) -> ptr      // 等价于 malloc
  └─ [ptr!=NULL] src/jemalloc.c:2697 rtree_szind_slab_read -> old_usize
       ├─ [prof] src/jemalloc.c:2708 irealloc_prof(tsd, ptr, old_usize, usize, ...)
       └─ [core] src/jemalloc.c:2715 iralloc(tsd, ptr, old_usize, size, 0, false, &hook_args)
            -> src/jemalloc.c:287 large_ralloc(...) [大对象]

je_rallocx(ptr, size, flags) -> src/jemalloc.c:3216 je_rallocx(...)
  ├─ [guard] src/jemalloc.c:3236 flags & MALLOCX_ARENA_MASK -> arena_get(3238) | arena=NULL
  ├─ [rtree] src/jemalloc.c:2697/3408 同族反查 -> old_usize
  └─ [outbound] src/jemalloc.c:2715 iralloc(...) -> 新指针或 NULL

je_xallocx(ptr, size, extra, flags) -> src/jemalloc.c:3390 je_xallocx(...)
  ├─ [guard] src/jemalloc.c:3422 size > SC_LARGE_MAXCLASS -> label_not_resized
  ├─ [clamp] src/jemalloc.c:3426 extra = SC_LARGE_MAXCLASS - size
  ├─ [prof] src/jemalloc.c:3431 ixallocx_prof(...)
  ├─ [core] src/jemalloc.c:3434 ixallocx_helper(...) -> newsize
  │    └─ src/jemalloc.c:3312 ixallocx_helper -> src/jemalloc.c:3316 ixalloc -> arena_ralloc_no_move
  └─ [hook] src/jemalloc.c:3448 hook_invoke_expand(hook_expand_xallocx, ...)

大对象决断:
  src/large.c:274 large_ralloc(tsdn, arena, ptr, usize, alignment, zero, tcache, hook_args)
    ├─ src/large.c:277 iealloc(tsdn, ptr) -> extent; 279 oldusize = extent_usize_get
    ├─ [原位] src/large.c:287 large_ralloc_no_move -> 成功则返回同址 (291)
    └─ [搬移] src/large.c:299 large_ralloc_move_helper -> ret
         ├─ src/large.c:311 copysize = min(usize, oldusize)
         ├─ src/large.c:312 memcpy(ret, old, copysize)
         └─ src/large.c:313 isdalloct(tsdn, old, oldusize, tcache, NULL, true)
```

## 3. rtree 反查：旧尺寸的唯一来源

`src/rtree.c:223` 的 `rtree_leaf_elm_lookup_hard(tsdn, rtree, rtree_ctx, key, dependent, init_missing)` 在 radix tree 未命中缓存时承担硬查找：`227-231` 行按 `RTREE_HEIGHT` 决定从 `rtree->root` 取 node 还是 leaf；`243-262` 行的 `RTREE_GET_CHILD` 宏按层级逐段用 `rtree_subkey(key, level)` 下降，`init_missing` 决定走 `_read`（分配）还是 `_tryread`（不分配）。

| 调用者 | 锚点 | 用途 |
| --- | --- | --- |
| `je_realloc` | `src/jemalloc.c:2697` | 取 `alloc_ctx.szind` → `old_usize`（`2700`）并断言等于 `isalloc`（`2701`） |
| `je_xallocx` | `src/jemalloc.c:3408` | 同上，供 clamp 与 `label_not_resized` 比较（`3437`） |
| `ifree` 复用 | `src/jemalloc.c:2570` | 释放路径共用同一 rtree 语义 |
| `free_fastpath` 复用 | `src/jemalloc.c:2823` | `rtree_szind_slab_read_fast` 避免硬查找 |

rtree 缓存上下文由 `tsd_rtree_ctx(tsd)` 提供（`src/jemalloc.c:2696`／`3407`），硬查找只在 L1/L2 缓存未命中时发生；这也是 `xallocx` 必须在入口就取 `old_usize` 而不能依赖用户传值的原因。

## 4. 原位与搬移决策表

`src/large.c:274` 的 `large_ralloc` 在 `282-284` 行断言新旧尺寸均不小于 `SC_LARGE_MINCLASS`，因此小对象走另一条 inline 路径。

| 决策 | 字面锚点 | 条件 | 结果 |
| --- | --- | --- | --- |
| 原位优先 | `src/large.c:287` | `large_ralloc_no_move` 返回 `false` | 同址返回并触发 `hook_invoke_expand`（`288-290`） |
| 收缩 | `src/large.c:96` `large_ralloc_no_move_shrink` | `extent_hooks->split != NULL`（`104`）且 `diff != 0`（`109`） | `extent_split_wrapper`（`110`）拆分，尾部交 `arena_extents_dirty_dalloc`（`122`） |
| 收缩失败 | `src/large.c:104-105` | `split == NULL` | 返回 `true`，转入搬移 |
| 扩展 | `src/large.c:131` `large_ralloc_no_move_expand` | `extent_hooks->merge != NULL`（`138`） | `extents_alloc` 取尾部 extent（`155`）并合并 |
| 扩展失败 | `src/large.c:138-139` | `merge == NULL` | 返回 `true`，转入搬移 |
| 搬移 | `src/large.c:299-313` | 上述原位尝试失败 | 新分配 + `memcpy` + `isdalloct` 释放旧块 |
| xallocx 不 resize | `src/jemalloc.c:3437-3439` | `usize == old_usize` | `label_not_resized`，返回旧尺寸且仍触发 expand hook（`3446-3450`） |

`src/jemalloc.c:3110` 的 `mallocx` OOM 文案、`src/jemalloc.c:3301` 的 `rallocx` OOM 文案与 `2729-2730` 的 `realloc` 文案分别硬编码，失败时 `2746-2753` 统一 `set_errno(ENOMEM)`，并在 `config_xmalloc && opt_xmalloc` 下 `abort()`。

## 5. 不变式与统计记账

`src/jemalloc.c:2754-2761` 在成功时把新 `usize` 计入 `tsd_thread_allocatedp`、旧 `old_usize` 计入 `tsd_thread_deallocatedp`，并断言 `usize == isalloc(tsdn, ret)`。`src/jemalloc.c:3441-3444` 对 `xallocx` 做同样的双向记账。

`src/jemalloc.c:3401` 断言 `SIZE_T_MAX - size >= extra`，但 API 明确声明不保护 `size + extra` 溢出（`3413-3417`），代码通过夹到 `SC_LARGE_MAXCLASS` 规避越界。这一区别是 `xallocx` 与 `rallocx` 最关键的行为差异。

## 6. 共变耦合与修改清单（Co-Change Playbook）

| 改动点 | 联动文件／符号 | 隐式陷阱 | 建议验证 |
| --- | --- | --- | --- |
| 改 extent hook 可用性 | `src/large.c:104`／`138` → `287` → `299` | 自定义 hook 缺 `split`/`merge` 会静默退化为搬移，改变指针稳定性 | 用 hook 单测覆盖原位失败分支 |
| 改 rtree 反查 | `src/rtree.c:223` → `src/jemalloc.c:2697`／`3408` | `init_missing=true` 会分配元数据，重入路径禁止 | 开启 prof 与 reentrancy 用例 |
| 改 clamp 上限 | `src/jemalloc.c:3422-3428` → `src/large.c:281` | 两处 `SC_LARGE_MAXCLASS` 判定不同步会导致 `xallocx` 返回未 clamp 的尺寸 | 边界尺寸表驱动测试 |
| 改搬移复制长度 | `src/large.c:311-313` → 统计记账 `src/jemalloc.c:2754` | `copysize` 取 min，若记账仍用新 `usize` 会多计 | 对比 `stats.allocated` 与实际占用 |

重分配的所有指针稳定性保证都以本页的 hook 与 rtree 锚点为边界，细节参照[extent 领域模型](domain-models/extent.md)。
