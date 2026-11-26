---
kind: workflow
scope: WF-allocate
status: draft
origin: generated
source_paths: [src/jemalloc.c, src/arena.c, src/tcache.c, src/large.c, src/sz.c]
symbols: [je_malloc, je_calloc, je_mallocx, imalloc_body, arena_slab_reg_alloc, large_malloc, tcache_alloc_small_hard]
upstream: [business/index.md, contracts/api-reference.md]
downstream: [business/domain-models/arena.md, architecture/allocator-core.md]
---

# 分配业务流（WF-allocate）

本页描述 jemalloc 从公开 C API 到 slab 位图／大对象 extent 的完整分配控制流。入口语义见[业务总览](index.md)与[API 契约](../contracts/api-reference.md)，落点结构见[arena 领域模型](domain-models/arena.md)与[分配核心架构](../architecture/allocator-core.md)。所有行号锚点均指向仓库根相对路径。

## 1. 入口职责与分流边界

三个公开入口共享同一 `imalloc_body` 核心，仅在 `static_opts_t`／`dynamic_opts_t` 初值上分化。`je_malloc` 独有 fastpath，`je_calloc` 打开溢出检查与 zero，`je_mallocx` 解析 flags。

| 入口 | 定义锚点 | 关键初值 | 失败出口 |
| --- | --- | --- | --- |
| `je_malloc(size)` | `src/jemalloc.c:2323` | fastpath 直取 tcache，见 `src/jemalloc.c:2330-2339` | 不满足 fastpath 即 tail-call `malloc_default` |
| `je_calloc(num, size)` | `src/jemalloc.c:2474` | `may_overflow=true` (`2484`)、`zero=true` (`2492`) | `<jemalloc>: Error in calloc(): out of memory`（`2487`） |
| `je_mallocx(size, flags)` | `src/jemalloc.c:3098` | `assert_nonempty_alloc=true` (`3108`)、解析对齐／zero／tcache 索引（`3115-3131`） | `<jemalloc>: Error in mallocx(): out of memory`（`3110`） |
| `malloc_default(size)` | `src/jemalloc.c:2271` | `set_errno_on_error=true`、OOM 文案 `2283` | `imalloc` 返回 errno 风格码 |
| `imalloc_body(...)` | `src/jemalloc.c:2028` | 统一核心：尺寸计算→采样→`imalloc_no_sample` | `label_oom`／`label_invalid_alignment` |

## 2. 符号级调用树

```text
je_malloc(size) -> src/jemalloc.c:2323 je_malloc(size)
  ├─ [guard: 未初始化] src/jemalloc.c:2326 malloc_initialized() -> malloc_default
  ├─ [guard: 非 fast]  src/jemalloc.c:2331 !tsd_fast || size > SC_LOOKUP_MAXCLASS -> malloc_default
  ├─ [guard: gc 周期]  src/jemalloc.c:2337 ticker_trytick(&tcache->gc_ticker) -> malloc_default
  ├─ [guard: 采样耗尽] src/jemalloc.c:2355 bytes_until_sample < 0 -> malloc_default
  ├─ [小类快速]        src/jemalloc.c:2341 sz_size2index_lookup(size) -> ind (< SC_NBINS, 2347)
  ├─ [小类 bin]        src/jemalloc.c:2369 tcache_small_bin_get(tcache, ind) -> bin
  └─ [outbound]        src/jemalloc.c:2371 cache_bin_alloc_easy(bin, &tcache_success) -> ptr | 未命中 -> malloc_default

malloc_default(size) -> src/jemalloc.c:2271 malloc_default(size)
  └─ src/jemalloc.c:2246 imalloc(sopts, dopts)
       └─ src/jemalloc.c:2028 imalloc_body(sopts, dopts, tsd)
            ├─ [guard] src/jemalloc.c:2049 compute_size_with_overflow -> label_oom
            ├─ [guard] src/jemalloc.c:2054 对齐非 2 幂/过小 -> label_invalid_alignment
            ├─ [prof]  src/jemalloc.c:2116 prof_alloc_prep -> src/jemalloc.c:2130 imalloc_sample
            └─ [core]  src/jemalloc.c:2149 imalloc_no_sample(sopts, dopts, tsd, size, usize, ind)

tcache 补货链（小对象未命中）:
  cache_bin_alloc_easy miss -> src/tcache.c:89 tcache_alloc_small_hard(tsdn, arena, tcache, tbin, binind)
    -> src/tcache.c:94 arena_tcache_fill_small(...)
    -> src/arena.c:271 arena_slab_reg_alloc(slab, bin_info) -> ptr
    -> src/tcache.c:99 cache_bin_alloc_easy(tbin) -> ptr

大对象链（ind >= SC_NBINS）:
  imalloc_no_sample -> src/large.c:14 large_malloc(tsdn, arena, usize, zero)
    -> src/large.c:17 large_palloc -> src/large.c:30 sz_sa2u(usize, CACHELINE)
    -> src/arena.c:428 arena_extent_alloc_large
    -> extents_alloc(extents_dirty, 438-440) | extents_alloc(extents_muzzy, 441-445) | extent_alloc_wrapper(448)
```

`src/jemalloc.c:1912` 的 `imalloc_no_sample` 在 `TCACHE_IND_AUTOMATIC` 下于 `1921-1925` 取 tcache；显式 arena 索引在 `1941` 调 `arena_get`，自动模式把 `arena=NULL` 延迟到 `1949` 的 `iallocztm` 内决断，这是减少锁与 tcache 抖动的核心优化。

## 3. 尺寸类与阈值判定表

| 判定点 | 字面锚点 | 阈值／规则 | 失败出口 |
| --- | --- | --- | --- |
| 非对齐请求尺寸类 | `src/jemalloc.c:2062` `sz_size2index(size)` | `ind >= SC_NSIZES` 视为不可分配（`2063`） | `label_oom` |
| 对齐请求尺寸归一化 | `src/jemalloc.c:2078` `sz_sa2u(size, alignment)` | `usize == 0` 或 `usize > SC_LARGE_MAXCLASS`（`2080`） | `label_oom` |
| calloc 乘积 | `src/jemalloc.c:2049` `compute_size_with_overflow` | `may_overflow=true` 时检查 `num*size` 溢出 | `label_oom` |
| 对齐合法性 | `src/jemalloc.c:2054` | `alignment < min_alignment` 或 `(alignment & (alignment-1)) != 0` | `label_invalid_alignment` → `EINVAL`（`2211`） |
| fastpath 上界 | `src/jemalloc.c:2331` | `size > SC_LOOKUP_MAXCLASS` 退出 fastpath | `malloc_default` |
| fastpath 小类断言 | `src/jemalloc.c:2347-2348` | `ind < SC_NBINS` 且 `size <= SC_SMALL_MAXCLASS` | `assert` |
| 大对象归一化 | `src/large.c:30-31` `sz_sa2u(usize, CACHELINE)` | `ausize == 0 \|\| ausize > SC_LARGE_MAXCLASS` | 返回 `NULL` |
| slab 剩余容量 | `src/arena.c:276-277` | `extent_nfree_get(slab) > 0` 且 bitmap 非 full | `assert` |
| 尺寸表启动 | `src/sz.c:60` `sz_boot(sc_data)` | 依次构造 `sz_boot_pind2sz_tab`／`sz_boot_index2size_tab`／`sz_boot_size2index_tab`（`61-63`） | 启动期完成，无运行期分支 |

`je_mallocx` 的 flags 解码位于 `src/jemalloc.c:3115-3131`：`MALLOCX_LG_ALIGN_MASK` 决定 `dopts.alignment`，`MALLOCX_ZERO` 决定 `dopts.zero`，`MALLOCX_TCACHE_NONE` 映射为 `TCACHE_IND_NONE`，缺省为 `TCACHE_IND_AUTOMATIC`。

## 4. 小对象 slab 位图分配与 tcache 补货

`src/arena.c:271` 的 `arena_slab_reg_alloc` 是最小分配原子：`279` 行 `bitmap_sfu` 取首个空闲 regind，`280-281` 行以 `extent_addr_get(slab) + reg_size * regind` 得到地址，`282` 行 `extent_nfree_dec(slab)` 扣减计数。批量版本 `arena_slab_reg_alloc_batch` 位于 `src/arena.c:287`，在支持 popcount 时用组位图一次扫多个 regind（`302-310`）。

补货触发链固定在 `src/tcache.c:89`：`tcache_alloc_small_hard` 先调 `src/tcache.c:94` 的 `arena_tcache_fill_small`，清空 `prof_accumbytes` 后用 `src/tcache.c:99` 的 `cache_bin_alloc_easy` 重取。`src/arena.c:1394-1395` 的补货数量是 `tcache_bin_info[binind].ncached_max >> tcache->lg_fill_div[binind]`，即填充量可被 GC 动态折半。

## 5. 大对象路径与错误语义

`src/large.c:14` 的 `large_malloc` 直接转 `src/large.c:17` 的 `large_palloc`，以 `CACHELINE` 对齐；`src/arena.c:428` 的 `arena_extent_alloc_large` 按 dirty → muzzy → 新映射三级降级取 extent，并在 `config_stats` 下累计 `mapped_add`。

| 错误 | 字面位置 | 触发条件 | 结果 |
| --- | --- | --- | --- |
| 普通 OOM | `src/jemalloc.c:2177-2197` | `allocation == NULL` | `set_errno(ENOMEM)`（`2190`）、结果置 `NULL` |
| 对齐非法 | `src/jemalloc.c:2204-2224` | 对齐校验失败 | `set_errno(EINVAL)`（`2211`） |
| xmalloc 中止 | `src/jemalloc.c:2178-2181` | `config_xmalloc && opt_xmalloc` | `malloc_write(oom_string)` + `abort()` |
| 重入降级 | `src/jemalloc.c:2097-2108` | `reentrancy_level > 0` 且 slow | 强制 `tcache_ind=NONE`、`arena_ind=0` |

## 6. 共变耦合与修改清单（Co-Change Playbook）

| 改动点 | 联动文件／符号 | 隐式陷阱 | 建议验证 |
| --- | --- | --- | --- |
| 改尺寸类映射 | `src/sz.c:60` `sz_boot` → `src/jemalloc.c:2062` `sz_size2index` → `src/arena.c:271` `arena_slab_reg_alloc` | fastpath 断言 `ind < SC_NBINS` 与新表不一致会直接 assert | 单测覆盖 `SC_LOOKUP_MAXCLASS` 边界 |
| 改 tcache 填充量 | `src/tcache.c:89` → `src/arena.c:1394` → `src/arena.c:271` | `lg_fill_div` 下溢为 0 时退化为满填充 | 压测 GC 折半路径 |
| 改大对象上限 | `src/large.c:31` → `src/jemalloc.c:2081` → `src/jemalloc.c:3390` `je_xallocx` | clamp 需同步 `SC_LARGE_MAXCLASS`，否则 xallocx 静默不 resize | 大对象 OOM 与对齐用例 |
| 改 OOM 文案／errno | `src/jemalloc.c:2283`／`2487`／`3110` → `2177-2197` | 文案按入口分别硬编码，漏改导致日志串味 | 开启 `abort_conf`/`xmalloc` 对照 |

分配路径的所有错误码与 errno 映射均以本节锚点为准；任何改动都应同时复核[API 契约](../contracts/api-reference.md)中的返回值约定。
