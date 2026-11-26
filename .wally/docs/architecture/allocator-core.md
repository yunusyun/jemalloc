---
kind: component
scope: allocator-core
status: draft
origin: generated
source_paths: [src/jemalloc.c, src/arena.c, src/bin.c, src/large.c, src/sz.c, src/sc.c]
symbols: [je_malloc, imalloc_body, imalloc_no_sample, arena_new, arena_slab_reg_alloc, large_malloc, sz_boot, sc_data_init]
upstream: [contracts/api-reference.md, overview/callchain-panorama.md]
downstream: [architecture/extent-and-os.md, architecture/tcache-and-tsd.md, business/wf-allocate.md]
---

# 分配器核心（allocator-core）

## 1. 角色与边界

本组件承载 jemalloc 的 `malloc(3)` 主分配路径：导出 C API、尺寸类查表、tcache 快路径、arena/bin/slab 小对象分配、large 大对象分配。对外的导出符号契约见 [contracts/api-reference.md](../contracts/api-reference.md)，本组件在全链路中的位置见 [overview/callchain-panorama.md](../overview/callchain-panorama.md)。内存来自 tiered extent（[architecture/extent-and-os.md](extent-and-os.md)），线程本地缓存状态见 [architecture/tcache-and-tsd.md](tcache-and-tsd.md)，分配业务流见 [business/wf-allocate.md](../business/wf-allocate.md)。

## 2. 符号级调用树（快路径优先）

```text
je_malloc(size) -> src/jemalloc.c:2323
  ├─ [guard] tsd 不可分配 / !malloc_initialized() -> malloc_default src/jemalloc.c:2271
  ├─ [guard] !tsd || !tsd_fast || size > SC_LOOKUP_MAXCLASS -> malloc_default
  ├─ [guard] ticker_trytick(gc_ticker) 命中 -> malloc_default 触发 tcache GC
  ├─ [size] sz_size2index_lookup(size) -> ind（快路径仅覆盖 SC_NBINS）src/jemalloc.c:2341
  ├─ [sample] bytes_until_sample 为负 -> malloc_default（切入 prof 采样）src/jemalloc.c:2355
  ├─ [fast] cache_bin_alloc_easy(bin, &success) -> 命中即返回 src/jemalloc.c:2371
  └─ [outbound] malloc_default -> imalloc src/jemalloc.c:2246 -> imalloc_body src/jemalloc.c:2028
       ├─ [guard] compute_size_with_overflow -> label_oom(ENOMEM)
       ├─ [guard] 对齐非 2 次幂 -> label_invalid_alignment(EINVAL)
       ├─ [reentrancy] slow 且 level>0 -> tcache=NONE, arena=0
       ├─ imalloc_no_sample src/jemalloc.c:1912 -> ipalloct(对齐) / iallocztm(非对齐)
       └─ [outbound] small -> arena/bin/slab；large -> large_malloc src/large.c:14
```

快路径直接访问 `cache_bin_t`，其定义式 `cache_bin_alloc_easy` 内联于 `include/jemalloc/internal/cache_bin.h:88`（本页 source_paths 不含该头文件，仅作调用边说明）。

## 3. 源码锚点与职责契约表

| 文件:行号 | 符号 | 职责 | 入 → 出 |
|---|---|---|---|
| `src/jemalloc.c:2323` | `je_malloc` | 导出入口，fastpath | `size_t` → `void *` / 回落 `malloc_default` |
| `src/jemalloc.c:2028` | `imalloc_body` | 统一尺寸/对齐/重入判定 | `static_opts_t*`, `dynamic_opts_t*` → 指针或 `errno` |
| `src/jemalloc.c:1912` | `imalloc_no_sample` | 填充 tcache/arena 并分流 | `ind`,`usize` → `ipalloct` / `iallocztm` |
| `src/arena.c:1942` | `arena_new` |  arena 构造，分配 `bin_t` 数组、`base_new` | `ind`,`extent_hooks` → `arena_t*` |
| `src/arena.c:271` | `arena_slab_reg_alloc` | slab 内 bitmap 取空闲 reg | `extent_t*`, `bin_info_t*` → `void*` |
| `src/bin.c:12` | `bin_infos_init` | 由 `sc_data_t` 计算 reg/slab/nregs/shards | `sc_data_t*` → `bin_infos[SC_NBINS]` |
| `src/large.c:14` | `large_malloc` | large 入口，转 `large_palloc` | `usize`,`zero` → `void*` |
| `src/sz.c:60` | `sz_boot` | 构建三张尺寸查表 | `sc_data_t*` → 查表就绪 |
| `src/sc.c:259` | `sc_data_init` | 生成 `size_classes` 数据 | `sc_data_t*` → `initialized=true` |

`sc_data_init` 由 `sc_boot`（`src/sc.c:311-312`）调用；`sz_boot` 依赖其结果，两者顺序不可颠倒。

## 4. 尺寸类—bin—slab 的初始化与分配闭环

`sc_data_init`（`src/sc.c:259`）调用 `size_classes` 写入 `sc_data`，随后 `sz_boot`（`src/sz.c:60`）构建 `sz_size2index_tab`、`sz_index2size_tab`、`sz_boot_pind2sz_tab`。`bin_infos_init`（`src/bin.c:12`）按 `sc->lg_base/ndelta/pgs` 推导 `reg_size`、`slab_size`、`nregs` 与 `bitmap_info`。`arena_new`（`src/arena.c:1942`）按 `bin_infos[i].n_shards` 累加 `nbins_total`，并向 `base_alloc` 申请 `sizeof(arena_t)+sizeof(bin_t)*nbins_total` 的连续块。`arena_slab_reg_alloc`（`src/arena.c:271`）用 `bitmap_sfu` 取得 `regind`，以 `extent_addr + reg_size*regind` 计算地址并递减 `nfree`。大于 `SC_SMALL_MAXCLASS` 的请求由 `large_malloc`（`src/large.c:14`）走独立 extent 路径，分配尾部的 OS 对接见 [architecture/extent-and-os.md](extent-and-os.md)。

## 5. 失败、errno 与可观测边界

| 触发点 | 位置 | 结果 |
|---|---|---|
| 尺寸乘法溢出 | `src/jemalloc.c:2049` | `label_oom` → `ENOMEM` |
| `ind >= SC_NSIZES` | `src/jemalloc.c:2063` | `label_oom` → `ENOMEM` |
| 对齐非 2 次幂 | `src/jemalloc.c:2054` | `label_invalid_alignment` → `EINVAL` |
| `large_palloc` 越界 | `src/large.c:31` | 返回 `NULL` |
| `arena_new` base 失败 | `src/arena.c:1951` | 返回 `NULL` |

快路径 `LOG("core.malloc.entry", ...)` 位于 `src/jemalloc.c:2324`，日志过滤由 `src/log.c` 提供，见 [architecture/control-and-observability.md](control-and-observability.md)。

## 6. 共变耦合与修改清单（Co-Change Playbook）

- 改动 `sc_data_t` 布局或 `SC_NBINS` → 同步 `src/sc.c:259`、`src/sz.c:60`、`src/bin.c:12`、`src/arena.c:1956-1961` 的派生计算，并更新 [contracts/api-reference.md](../contracts/api-reference.md) 的尺寸上限描述。
- 改动 `imalloc_body` 入参结构 → 同步 `static_opts_init`/`dynamic_opts_init`（`src/jemalloc.c:1864`/`1899`）、所有 `mallocx` 族导出入口与 [business/wf-allocate.md](../business/wf-allocate.md) 的参数表。
- 改动 slab reg 布局 → 同步 `arena_slab_reg_alloc` 与 `arena_slab_reg_dalloc`（`src/arena.c:359`）、bin shard 统计。
- 改动 `large_malloc` 对齐语义 → 同步 `large_palloc`（`src/large.c:21`）与 `sz_sa2u` 约束。
- 验证命令：`make tests && make check`；单测入口 `test/unit/arena.c`、`test/unit/large.c`。
- 隐式陷阱：快路径假设 `SC_LOOKUP_MAXCLASS` 小于 `SC_SMALL_MAXCLASS`（`src/jemalloc.c:2346` 断言），扩大查找表上界必须先放宽该断言。
