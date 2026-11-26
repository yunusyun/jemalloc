---
kind: mapping
scope: inbound-to-outbound-mapping
status: draft
origin: generated
source_paths: [src/jemalloc.c, src/arena.c, src/extent.c, src/pages.c]
symbols: [imalloc_body, arena_extent_alloc_large, extents_alloc, pages_map]
upstream: [contracts/api-reference.md, architecture/index.md]
downstream: [integrations/downstream/os-memory.md, integrations/downstream/libc-runtime.md]
---

# 入站 C API 到出站 OS-VM 的参数映射

_scope: inbound-to-outbound-mapping_。jemalloc 无 JSON/HTTP/业务字典；「入站」为导出的 C API 与 `flags` 位域，「出站」为 `mmap`/`munmap`/`madvise`/`sbrk` 实参。本文追踪 `size`/`flags` 如何被拆成 `arena`/`tcache`/`extent`/OS-VM 参数值；完整 syscall 参数表见 [OS 内存下游契约](downstream/os-memory.md)，线程/锁边界见 [libc 运行时下游契约](downstream/libc-runtime.md)。

## 1. 入站面：`MALLOCX_*` 位域与 `dynamic_opts_t` 装配

`flags` 为单 `int`，布局见 `include/jemalloc/internal/jemalloc_internal_types.h:24-50`（`aaaaaaaa aaaatttt tttttttt 0znnnnnn`）；宏定义见 `include/jemalloc/jemalloc_macros.h.in:15-33`。下表为字面定义与提取规则。

| flags 宏 | 字面定义（path:line） | 位段 | 提取宏 / 结果 |
|---|---|---|---|
| `MALLOCX_LG_ALIGN(la)` | `((int)(la))`，`include/jemalloc/jemalloc_macros.h.in:15` | `[0..5]`，`MALLOCX_LG_ALIGN_BITS=6`（`include/jemalloc/internal/jemalloc_internal_types.h:28`） | `MALLOCX_ALIGN_GET_SPECIFIED(flags)` = `ZU(1) << (flags & 0x3f)`（`:40-41`）；`MALLOCX_ALIGN_GET` 再与 `(SIZE_T_MAX-1)` 相与（`:42-43`） |
| `MALLOCX_ALIGN(a)` | `ffs(a)-1`（32 位）/ `ffs(a>>32)+31`（64 位），`include/jemalloc/jemalloc_macros.h.in:17-21` | 同上 | 入站把对齐字节数编码为 `lg(alignment)` |
| `MALLOCX_ZERO` | `((int)0x40)`，`include/jemalloc/jemalloc_macros.h.in:23` | bit6 | `MALLOCX_ZERO_GET(flags)` = `(bool)(flags & MALLOCX_ZERO)`，`include/jemalloc/internal/jemalloc_internal_types.h:44-45` |
| `MALLOCX_TCACHE(tc)` | `((tc)+2) << 8`，`include/jemalloc/jemalloc_macros.h.in:28` | `[8..19]`，`MALLOCX_TCACHE_BITS=12`、`SHIFT=8`（`:27,:30`） | `MALLOCX_TCACHE_GET(flags)` = `((flags & MASK) >> 8) - 2`，`:47-48`；`MALLOCX_TCACHE_NONE` = `MALLOCX_TCACHE(-1)` = `1<<8`，`:29` |
| `MALLOCX_ARENA(a)` | `(((int)(a))+1) << 20`，`include/jemalloc/jemalloc_macros.h.in:33` | `[20..31]`，`MALLOCX_ARENA_BITS=12`、`SHIFT=20`（`:26,:29`） | `MALLOCX_ARENA_GET(flags)` = `((unsigned)flags >> 20) - 1`，`:49-50`；`MALLOCX_ARENA_LIMIT=4095`，`:34` |

`src/jemalloc.c:3098-3146` `je_mallocx(size, flags)` 是唯一把全部四类位域装配进 `dynamic_opts_t` 的入口（`src/jemalloc.c:1882-1908` 给出哨兵值 `TCACHE_IND_NONE=-1`、`TCACHE_IND_AUTOMATIC=-2`、`ARENA_IND_AUTOMATIC=-1` 及 `dynamic_opts_init` 默认值）。

| 目标字段（`dynamic_opts_t`） | 装配表达式 | 锚点 `src/jemalloc.c` | 缺省值（`dynamic_opts_init`） |
|---|---|---|---|
| `result` | `&ret` | `:3112` | `NULL`（`:1900`） |
| `num_items` | `1` | `:3113` | `0`（`:1902`） |
| `item_size` | `size` | `:3114` | `0`（`:1903`） |
| `alignment` | 仅当 `(flags & MALLOCX_LG_ALIGN_MASK) != 0`：`MALLOCX_ALIGN_GET_SPECIFIED(flags)` | `:3115-3118` | `0`（`:1904`） |
| `zero` | `MALLOCX_ZERO_GET(flags)`（恒计算，含 `flags==0`） | `:3120` | `false`（`:1905`） |
| `tcache_ind` | `TCACHE_IND_NONE`（等于 `MALLOCX_TCACHE_NONE`）／`MALLOCX_TCACHE_GET(flags)`／`TCACHE_IND_AUTOMATIC` | `:3122-3131` | `TCACHE_IND_AUTOMATIC`（`:1906`） |
| `arena_ind` | 仅当 `(flags & MALLOCX_ARENA_MASK) != 0`：`MALLOCX_ARENA_GET(flags)` | `:3133-3134` | `ARENA_IND_AUTOMATIC`（`:1907`） |

`flags==0` 时跳过整个解码块，保留上表缺省值（`src/jemalloc.c:3115`）。

## 2. 符号级调用树：`size`/`flags` → OS-VM 实参

```text
je_mallocx(size, flags) -> src/jemalloc.c:3098
  ├─ static_opts_init / dynamic_opts_init -> src/jemalloc.c:1864 / :1899
  ├─ [flags!=0 解码] MALLOCX_* -> dopts{alignment,zero,tcache_ind,arena_ind} -> src/jemalloc.c:3115-3135
  └─ imalloc(&sopts,&dopts) -> src/jemalloc.c:3137 -> :2246
      ├─ [init] imalloc_init_check -> src/jemalloc.c:2228 -> malloc_init()
      ├─ tsd_fetch() -> src/jemalloc.c:2252
      └─ imalloc_body -> src/jemalloc.c:2028 / :2258|:2265
          ├─ compute_size_with_overflow(may_overflow,dopts,&size) -> src/jemalloc.c:1990 -> size
          ├─ [guard] alignment<min_alignment 或非 2 幂 -> label_invalid_alignment -> src/jemalloc.c:2054-2057/:2204 -> EINVAL
          ├─ [alignment==0] sz_size2index(size)->ind; sz_index2size(ind)->usize -> src/jemalloc.c:2062-2071
          ├─ [alignment>0] usize=sz_sa2u(size,alignment) -> src/jemalloc.c:2078；>SC_LARGE_MAXCLASS -> label_oom
          ├─ [guard: slow 且 reentrancy>0] tcache_ind=NONE, arena_ind=0 -> src/jemalloc.c:2096-2108
          └─ imalloc_no_sample -> src/jemalloc.c:1912 / :2149
              ├─ [tcache_ind] tsd_tcachep_get | NULL | tcaches_get -> src/jemalloc.c:1918-1930
              ├─ [arena_ind] NULL（延迟）| arena_get(tsdn,ind,true) -> src/jemalloc.c:1933-1942
              ├─ [alignment!=0] ipalloct(usize,alignment,zero,tcache,arena) -> src/jemalloc.c:1945
              └─ [alignment==0] iallocztm(size,ind,zero,tcache,false,arena,slow) -> src/jemalloc.c:1949
                  -> arena_malloc / arena_palloc -> large_malloc -> src/large.c:14/:47
                  -> arena_extent_alloc_large(usize,alignment,&zero) -> src/arena.c:428
                      ├─ szind=sz_size2index(usize); commit=true; pad=sz_large_pad -> src/arena.c:435-440
                      ├─ extents_alloc(dirty) -> src/arena.c:438 -> src/extent.c:537 -> extent_recycle -> src/extent.c:545
                      ├─ [arena_may_have_muzzy] extents_alloc(muzzy) -> src/arena.c:441-445
                      └─ extent_alloc_wrapper -> src/arena.c:448 -> src/extent.c:1531
                          ├─ extent_alloc_retained -> src/extent.c:1539
                          └─ [!(opt_retain && new_addr)] extent_alloc_wrapper_hard -> src/extent.c:1551/:1492
                              ├─ esize=size+pad; palignment=ALIGNMENT_CEILING(alignment,PAGE) -> src/extent.c:1495/:1501
                              ├─ extent_alloc() metadata -> src/extent.c:1496 -> :180（ avail 为空则 base_alloc_extent -> :185）
                              └─ extent_alloc_default_impl -> src/extent.c:1239 -> extent_alloc_core -> :1209
                                  ├─ [dss primary] extent_alloc_dss -> src/extent.c:1217-1221
                                  ├─ [mmap] extent_alloc_mmap -> src/extent.c:1223 -> src/extent_mmap.c:22
                                  │   -> pages_map(new_addr,esize,palignment,commit) -> src/pages.c:179
                                  │      -> os_pages_map -> src/pages.c:54 -> mmap(addr,size,prot,mmap_flags,-1,0) -> src/pages.c:79
                                  └─ [dss secondary] extent_alloc_dss -> src/extent.c:1228-1232
                                      -> extent_dss_sbrk(incr) -> src/extent_dss.c:42/:181
```

`[outbound]` 边界止于 `src/pages.c:79`（`mmap`）、`src/pages.c:135`（`munmap`）、`src/extent_dss.c:44`（`sbrk`）；内核实现 `[外部黑盒：本仓未包含服务端实现]`。

## 3. 参数值逐跳变换表

| 跳 | 入参值 | 出参值 / 决策 | 锚点 |
|---|---|---|---|
| flags→opts | `alignment` 位段 `n` | `dopts.alignment = 1<<n`；未指定则 `0` | `src/jemalloc.c:3116-3118` |
| flags→opts | `MALLOCX_TCACHE_NONE` | `tcache_ind=TCACHE_IND_NONE`，继而 `tcache=NULL` | `src/jemalloc.c:3123-3125`→`:1926-1927` |
| flags→opts | `MALLOCX_ARENA(a)` | `arena_ind=a`，继而 `arena_get(tsdn,a,true)` | `src/jemalloc.c:3133-3134`→`:1941` |
| size→ind/usize | `alignment==0`, `size` | `ind=sz_size2index(size)`；`usize=sz_index2size(ind)` | `src/jemalloc.c:2062-2071` |
| size→usize | `alignment>0`, `(size,alignment)` | `usize=sz_sa2u(size,alignment)`，`0`/`>SC_LARGE_MAXCLASS` → OOM | `src/jemalloc.c:2078-2083` |
| opts→tcache | `tcache_ind=AUTOMATIC/NONE/其他` | `tcache=tsd_tcachep_get(tsd)/NULL/tcaches_get(tsd,ind)` | `src/jemalloc.c:1918-1930` |
| opts→arena | `arena_ind=AUTOMATIC/其他` | `arena=NULL`（tcache 命中前不计算）/`arena_get(tsdn,ind,true)` | `src/jemalloc.c:1933-1942` |
| small/large 分流 | `ind` 与 `sz_size2index(usize)` | `ind<SC_SMALL_MAXCLASS` 走 `iallocztm`；大对象经 `arena_palloc`→`large_malloc` | `src/jemalloc.c:1949-1950`；`src/large.c:14-17` |
| arena→extent | `usize`, `alignment` | `szind=sz_size2index(usize)`, `pad=sz_large_pad`, `commit=true` 先试 dirty→muzzy→新建 | `src/arena.c:435-450` |
| extent→映射 | `esize=usize+pad`, `palignment=ALIGNMENT_CEILING(alignment,PAGE)` | `extent_alloc_mmap(new_addr,esize,palignment,&zero,&commit)` | `src/extent.c:1495-1505`→`:1223`；`src/arena.c:446` |
| 映射→syscall | `prot = *commit ? (PROT_READ\|PROT_WRITE) : PROT_NONE`；`mmap_flags = MAP_PRIVATE\|MAP_ANON`（`MAP_NORESERVE` 视 overcommit） | `mmap(addr,esize,prot,mmap_flags,-1,0)` | `src/pages.c:25-27,:77-79,:610,:617-620` |

## 4. 守卫与失败出口

| 守卫 | 条件（字面） | 失败出口 |
|---|---|---|
| 对齐合法性 | `dopts->alignment < sopts->min_alignment \|\| (alignment & (alignment-1)) != 0`，`src/jemalloc.c:2054-2055` | `label_invalid_alignment` `:2204`：`opt_xmalloc` 时写 `invalid_alignment_string`+`abort()`，否则 `set_errno(EINVAL)`、返回 `EINVAL` `:2205-2224` |
| 尺寸溢出 | `compute_size_with_overflow` 为真，`src/jemalloc.c:2049-2052` | `label_oom` `:2177`：`set_errno(ENOMEM)`、返回 `ENOMEM` `:2189-2197` |
| 尺寸类越界 | `ind >= SC_NSIZES` 或 `usize==0 \|\| usize>SC_LARGE_MAXCLASS`，`src/jemalloc.c:2063,:2080-2082` | `label_oom` |
| 重入 | `sopts->slow && reentrancy_level>0`，`src/jemalloc.c:2097-2108` | 强制 `tcache_ind=NONE`、`arena_ind=0`（arena 0 已初始化） |
| DSS 符号位 | `(intptr_t)size < 0`，`src/extent_dss.c:122-124` | `extent_alloc_dss` 返回 `NULL`，转试 mmap/secondary |
| mmap 对齐失配 | `ret != addr` 且 `addr==NULL`，`src/pages.c:226-234` | `os_pages_unmap` 后 `pages_map_slow`  oversized map + trim，`:155-176` |
| DSS 溢出/竞争 | `dss_prev == (void *)-1` 或回绕，`src/extent_dss.c:167-170,:217-222` | 置 `dss_exhausted=true`，`label_oom` 返回 `NULL` |

`label_oom` 的 `oom_string` 由各入站入口注入，`je_mallocx` 为 `"<jemalloc>: Error in mallocx(): out of memory\n"`（`src/jemalloc.c:3110`）；错误码映射为 `ENOMEM`/`EINVAL`，与 [libc 运行时下游契约](downstream/libc-runtime.md) 的 `malloc_write`/`abort` 出口共用同一路径。

## 5. 共变耦合与修改清单（Co-Change Playbook）

- 改 `MALLOCX_*` 位宽/偏置（`include/jemalloc/internal/jemalloc_internal_types.h:26-50` 与 `include/jemalloc/jemalloc_macros.h.in:15-33`）→ 同步 `je_mallocx` 解码 `src/jemalloc.c:3115-3135`、`extent_structs.h` 的 `EXTENT_BITS_ARENA_WIDTH`、mallctl `arena.<i>` 索引上限；漏改会使 `arena_ind` 静默偏移。
- 改 `dynamic_opts_t` 字段（`src/jemalloc.c:1886-1908`）→ 同步全部分派点 `malloc_default :2271`、`je_calloc :2474`、`je_posix_memalign :2393`、`je_mallocx :3098`、`je_rallocx :3216`、`imalloc_no_sample :1912`、`imalloc_sample :1954`。
- 改大对象 pad/对齐（`sz_large_pad`、`src/arena.c:446`）→ 联动 `src/extent.c:1495-1501` 的 `esize`/`palignment`、`src/extent_mmap.c:22-25` 的 `assert(alignment == ALIGNMENT_CEILING(alignment, PAGE))`、`src/pages.c:180-181` 的对齐断言；再联动 [OS VM 下游契约](downstream/os-memory.md) 的 `mmap` 参数表。
- 改 `extent_alloc_core` 策略顺序（`src/extent.c:1217-1232`）→ 联动 `dss` 配置解析 `src/jemalloc.c:1195-1205`、`extent_dss_prec_set` `src/extent_dss.c:63-69`、`retain` 解析 `:1194` 与 `extent_alloc_wrapper` 的 `opt_retain && new_addr` 守卫 `src/extent.c:1542-1550`。
- 验证：`make tests && make check`（`Makefile.in:529,:553`）；重点单测覆盖 `mallocx` flags 解码、`posix_memalign`/`aligned_alloc` 对齐与 OOM/EINVAL 分支；本页为静态追踪，未执行构建。
- 隐式陷阱：`arena_ind` 与 `tcache_ind` 均用负数哨兵且带偏置，新增 arena 索引不得达到 `MALLOCX_ARENA_LIMIT=4095`；内核对 `mmap_flags`/THP 的实际语义为 `[外部黑盒：本仓未包含服务端实现]`。
