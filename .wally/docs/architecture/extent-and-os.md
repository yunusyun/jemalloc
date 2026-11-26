---
kind: component
scope: extent-and-os-adapter
status: draft
origin: generated
source_paths: [src/extent.c, src/extent_mmap.c, src/extent_dss.c, src/pages.c, src/base.c]
symbols: [extents_alloc, extents_evict, extent_register, extent_deregister, extent_alloc_mmap, extent_alloc_dss, pages_map, base_new]
upstream: [architecture/allocator-core.md]
downstream: [integrations/downstream/os-memory.md, business/domain-models/extent.md]
---

# Extent 与 OS 适配层（extent-and-os-adapter）

## 1. 角色与边界

本组件是分配器核心与操作系统内存边界之间的适配层：管理 extent 的生命周期（获取、登记、拆分、驱逐、注销），并把映射请求落到 `mmap`/`sbrk` 与页级 `commit/decommit`。上游消费方见 [architecture/allocator-core.md](allocator-core.md)，OS 侧外部黑盒边界见 [integrations/downstream/os-memory.md](../integrations/downstream/os-memory.md)，extent 数据模型见 [business/domain-models/extent.md](../business/domain-models/extent.md)。本仓不含 OS VM 实现，`mmap`/`sbrk`/`madvise` 均视为 `[外部黑盒：本仓未包含服务端实现]`。

## 2. 符号级调用树

```text
extents_alloc src/extent.c:537
  └─ extent_recycle -> extent_recycle_extract src/extent.c:892
       ├─ [hit] extents 复用 -> extent_split_interior src/extent.c:980
       └─ [miss] extent_alloc src/extent.c:180 -> base_alloc_extent src/base.c:469
            └─ [outbound] extent_hooks.alloc -> extent_alloc_mmap src/extent_mmap.c:22
                 └─ pages_map src/pages.c:179 -> os_pages_map / pages_map_slow src/pages.c:155
            └─ [alternate] extent_alloc_dss src/extent_dss.c:110 -> extent_dss_sbrk

extents_evict src/extent.c:567
  ├─ malloc_mutex_lock(&extents->mtx) -> 取 LRU 并合并
  ├─ extent_deactivate src/extent.c:682
  └─ [outbound] extent_hooks.purge -> extent_purge_forced_impl src/extent.c:648

生命周期登记：
  extent_register src/extent.c:815 -> rtree 写入 -> prof_gdump 记账
  extent_deregister src/extent.c:878 -> rtree 移除 -> prof_gdump 减账
```

`extents_alloc` 前置 `witness_assert_depth_to_rank(..., WITNESS_RANK_CORE, 0)`（`src/extent.c:542-543`），锁序错误会触发 witness 断言。

## 3. 源码锚点与职责契约表

| 文件:行号 | 符号 | 职责 | 入 → 出 |
|---|---|---|---|
| `src/extent.c:537` | `extents_alloc` | 从 tiered extents 取可用 extent | `size`,`alignment`,`slab` → `extent_t*` |
| `src/extent.c:567` | `extents_evict` | 按 `npages_min` 驱逐/净化 LRU | `extents_t*`,`size_t` → `extent_t*` |
| `src/extent.c:815` | `extent_register` | 登记（含 gdump 增账） | `extent_t*` → `bool` 错误 |
| `src/extent.c:878` | `extent_deregister` | 注销（含 gdump 减账） | `extent_t*` → `void` |
| `src/extent.c:180` | `extent_alloc` | extent 元数据对象池获取 | `arena_t*` → `extent_t*` |
| `src/extent_mmap.c:22` | `extent_alloc_mmap` | mmap 后端默认 hook | `size`,`alignment` → `void*` |
| `src/extent_dss.c:110` | `extent_alloc_dss` | sbrk 后端，含 gap 回收 | `size`,`alignment` → `void*` |
| `src/pages.c:179` | `pages_map` | 页级映射，处理对齐与 commit | `addr`,`size`,`alignment` → `void*` |
| `src/base.c:350` | `base_new` | 构造 arena 元数据 base 分配器 | `ind`,`extent_hooks` → `base_t*` |

`extent_register`/`extent_deregister` 在 `src/extent.c` 内为 `static`，分别包装 `extent_register_impl`、`extent_deregister_impl`；另有 `_no_gdump_add`/`_no_gdump_sub` 变体（`src/extent.c:820`/`883`）用于避免 prof 记账重入。

## 4. mmap / DSS / base 三路后端

| 后端 | 锚点 | 触发条件 | 释放语义 |
|---|---|---|---|
| mmap | `src/extent_mmap.c:22` | 默认 `extent_hooks.alloc` | `extent_dalloc_mmap`（`src/extent_mmap.c:37`）在 `opt_retain=false` 时 `pages_unmap` |
| DSS | `src/extent_dss.c:110` | `dss` 优先级且 `have_dss` | `extent_dalloc`（`src/extent_dss.c:191`），`!commit` 时 `pages_decommit` |
| base | `src/base.c:350` | arena 元数据/内部对象 | bump 分配，随 base block 整体回收 |

`extent_alloc_mmap` 在 `commit=true` 时强制 `*zero=true`（`src/extent_mmap.c:30-32`），因为新建映射保证清零。`extent_alloc_dss` 以 `intptr_t` 检查 `size` 符号位（`src/extent_dss.c:122`），防止大请求被解释为负增量；其对齐 gap 通过 `extent_dalloc_gap` 回收（`src/extent_dss.c:189`）。`pages_map` 在对齐不满足时走 `pages_map_slow`（`src/pages.c:155`），后者多映射 `alignment - os_page` 再 `os_pages_trim`。`base_new` 用 `base_block_alloc` 取首个 block，并用 `malloc_mutex_init(..., WITNESS_RANK_BASE)` 初始化锁（`src/base.c:353`/`366`）。

## 5. 索引一致性与驱逐状态机

每次登记把 extent 的虚拟地址区间写入全局 `rtree`，`extent_deregister` 反向移除；`extent_interior_deregister`（`src/extent.c:838`）专用于 slab 内部指针的清理。`extents_evict` 在 `src/extent.c:572` 持 `extents->mtx`，循环直到取到完全合并的 LRU extent，随后 `extent_deactivate` 转入 muzzy 状态，必要时触发 `extent_purge_forced_impl`。状态转换与 decay 节拍耦合，详见 [architecture/profiler-and-background.md](profiler-and-background.md) 的后台工作线程部分。

## 6. 共变耦合与修改清单（Co-Change Playbook）

- 改动 `extent_hooks` 签名 → 同步 `src/extent_mmap.c:22`、`src/extent_dss.c:110`、`src/base.c:353` 及 [business/domain-models/extent.md](../business/domain-models/extent.md) 的 hook 契约。
- 改动 `rtree` key 编码 → 同步 `extent_register`/`extent_deregister`（`src/extent.c:815`/`878`）、`extent_interior_deregister`（`src/extent.c:838`）与 `src/rtree.c` 的查表逻辑。
- 改动页大小/commit 语义 → 同步 `src/pages.c:179` 与 `src/pages.c:155` 的 trim 路径，并核对 `os_overcommits` 分支（`src/pages.c:189`）。
- 改动 `opt_retain` 默认值 → 同步 `src/extent_mmap.c:11-17` 与 [integrations/downstream/os-memory.md](../integrations/downstream/os-memory.md) 的解除映射描述。
- 验证命令：`make tests && make check`；重点单测 `test/unit/extent.c`、`test/unit/decay.c`。
- 隐式陷阱：`extents_alloc` 的 witness 深度断言要求调用方处于 `WITNESS_RANK_CORE` 深度 0（`src/extent.c:542`）；新增调用路径必须先核对锁序。
