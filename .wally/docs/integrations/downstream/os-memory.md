---
kind: downstream-contract
scope: os-memory
status: draft
origin: generated
source_paths: [src/pages.c, src/extent_mmap.c, src/extent_dss.c, src/jemalloc.c]
symbols: [pages_map, pages_unmap, pages_commit_impl, pages_decommit, pages_huge, extent_alloc_mmap, extent_alloc_dss, obtain_malloc_conf]
upstream: [integrations/index.md, integrations/topology-and-mapping.md]
downstream: []
---

# OS 虚拟机内存下游契约

_scope: os-memory_。本仓是 jemalloc 客户端实现，所有 OS 内核内存管理为 `[外部黑盒：本仓未包含服务端实现]`。本文只记录本仓**实际发出的请求实参、返回解包与失败处理**。上游参数来源见 [入站到出站参数映射](../topology-and-mapping.md)，集成总览见 [integrations/index](../index.md)。

## 1. 下游边界与符号级调用树

```text
[入站] extent_alloc_mmap / extent_alloc_dss -> src/extent.c:1223/:1217
  ├─ extent_alloc_mmap(new_addr,size,alignment,&zero,&commit) -> src/extent_mmap.c:22
  │   └─ pages_map(new_addr,size,alignment,commit) -> src/pages.c:179
  │       ├─ [FreeBSD] mmap(addr,size,prot,mmap_flags|MAP_FIXED|MAP_EXCL,MAP_ALIGNED(...),-1,0) -> src/pages.c:183-204
  │       ├─ os_pages_map(addr,size,os_page,commit) -> src/pages.c:54
  │       │   └─ [outbound] mmap(addr,size,prot,mmap_flags,-1,0) -> src/pages.c:79
  │       └─ [对齐失配] os_pages_unmap + pages_map_slow -> src/pages.c:231-234/:155
  ├─ extent_alloc_dss(tsdn,arena,new_addr,size,alignment,&zero,&commit) -> src/extent_dss.c:110
  │   ├─ extent_dss_max_update -> extent_dss_sbrk(0) -> src/extent_dss.c:92/:97 → sbrk(0) :44
  │   └─ [outbound] sbrk(incr) -> src/extent_dss.c:44（调用点 :181，incr=gap_size_subpage+size :175）
  └─ [元数据/THP] extent_alloc() -> src/extent.c:180（avail 空则 base_alloc_extent -> :185）
      ├─ pages_set_thp_state -> src/pages.c:530 -> pages_huge_unaligned/nohuge_unaligned -> :540/:544
      └─ pages_purge_forced(addr,size) -> madvise(MADV_DONTNEED) -> src/pages.c:337
[解映射/回收]
  pages_unmap(addr,size) -> src/pages.c:241 -> os_pages_unmap -> [outbound] munmap(addr,size) -> src/pages.c:135
  extent_dalloc_mmap(addr,size) -> src/extent_mmap.c:37 -> pages_unmap（仅 !opt_retain）-> :39
[配置探测（启动期）]
  pages_boot -> src/pages.c:599
    ├─ os_page_detect -> sysconf(_SC_PAGESIZE) -> src/pages.c:429
    ├─ os_overcommits_proc -> open/syscall "/proc/sys/vm/overcommit_memory" -> src/pages.c:475-525
    ├─ init_thp_state -> open/syscall "/sys/kernel/mm/transparent_hugepage/enabled" -> src/pages.c:564-592
    └─ obtain_malloc_conf(which_source=2) -> readlink/readlinkat("/etc/malloc.conf") -> src/jemalloc.c:980/:982
```

## 2. 映射与解映射：完整请求实参表

| 调用（锚点） | 实参表（按形参顺序） | 前置断言 / 派生值 | 返回解包 |
|---|---|---|---|
| `mmap`（POSIX，`src/pages.c:79`） | `addr`=`os_pages_map` 入参；`size`=已 `ALIGNMENT_CEILING(size, os_page)`；`prot`=`*commit ? PAGES_PROT_COMMIT : PAGES_PROT_DECOMMIT`；`flags`=`mmap_flags`；`fd`=`-1`；`offset`=`0` | `ALIGNMENT_ADDR2BASE(addr, os_page)==addr`、`ALIGNMENT_CEILING(size, os_page)==size`、`size!=0`（`:55-57`）；`os_overcommits` 时强制 `*commit=true`（`:59-61`）；`PAGES_PROT_COMMIT=(PROT_READ\|PROT_WRITE)`、`PAGES_PROT_DECOMMIT=PROT_NONE`（`:25-26`） | `MAP_FAILED`→`NULL`（`:83-84`）；`addr!=NULL && ret!=addr`→`os_pages_unmap(ret,size)`+`NULL`（`:85-91`） |
| `mmap`（FreeBSD，`src/pages.c:204`） | `addr`,`size`,`prot`,`flags = mmap_flags \| (addr!=NULL ? MAP_FIXED\|MAP_EXCL : MAP_ALIGNED(ffs_zu(alignment)-1))`,`-1`,`0` | `os_overcommits`→`*commit=true`（`:189-191`）；不用裸 `MAP_FIXED` 以免替换既有映射（注释 `:72-75`） | `MAP_FAILED`→`NULL`（`:205-207`） |
| `mmap`（commit/decommit，`src/pages.c:263-264`） | `addr`,`size`,`prot = commit ? PAGES_PROT_COMMIT : PAGES_PROT_DECOMMIT`,`mmap_flags\|MAP_FIXED`,`-1`,`0` | `PAGE_ADDR2BASE(addr)==addr`、`PAGE_CEILING(size)==size`（`:250-251`）；`os_overcommits` 直接返回 `true`（`:253-255`） | `MAP_FAILED`→`true`（`:265-267`）；`result!=addr`→`os_pages_unmap(result,size)`+`true`（`:268-275`）；`result==addr`→`false`（`:276`） |
| `munmap`（`src/pages.c:135`，封装 `pages_unmap:241`） | `addr`,`size` | `PAGE_ADDR2BASE(addr)==addr`、`PAGE_CEILING(size)==size`（`:242-243`）；Windows 为 `VirtualFree(addr,0,MEM_RELEASE)`（`:133`） | `==-1`→`buferror(get_errno(),buf,...)`+`malloc_printf("<jemalloc>: Error in munmap(): %s\n",buf)`+`if(opt_abort) abort()`（`:137-151`）；成功无返回 |
| `mmap`（lazy purge 探测，`src/pages.c:636`） | `NULL`,`PAGE`,`PAGE`,`&committed=false` | 仅当 `pages_can_purge_lazy`（`:634`） | `NULL`→`pages_boot` 返回 `true`（`:637-639`）；随后 `pages_purge_lazy` 失败置 `pages_can_purge_lazy_runtime=false`（`:641-643`），并 `os_pages_unmap`（`:644`） |
| `pages_map_slow`（`src/pages.c:155-176`） | `alloc_size=size+alignment-os_page`；`os_pages_map(NULL, alloc_size, alignment, commit)`（`:164`）；`leadsize=ALIGNMENT_CEILING(pages,alignment)-pages`（`:168-169`）；`os_pages_trim(pages,alloc_size,leadsize,size,commit)`（`:170`） | `alloc_size<size` 防回绕→`NULL`（`:158-160`）；循环至 `ret!=NULL`（`:163-171`） | 返回 `PAGE_ADDR2BASE(ret)==ret` 的对齐地址（`:174-175`） |
| `os_pages_trim`（`src/pages.c:99-125`） | 前导 `os_pages_unmap(addr, leadsize)`（`:118`）、尾部 `os_pages_unmap((char*)ret+size, trailsize)`（`:121`），`trailsize=alloc_size-leadsize-size`（`:115`） | `alloc_size>=leadsize+size`（`:103`） | 返回 `ret=addr+leadsize`（`:101`） |

`mmap_flags` 在 `pages_boot` 中初始化为 `MAP_PRIVATE | MAP_ANON`（`src/pages.c:610`）；仅当 overcommit 检测为真且定义 `MAP_NORESERVE` 时追加 `MAP_NORESERVE`（`:617-620`）。

## 3. commit / decommit / purge / THP 实参表

| 出站调用（锚点） | 实参 | 结果语义（本仓解包） | 条件门 |
|---|---|---|---|
| `mmap(..., mmap_flags\|MAP_FIXED, ...)`（`src/pages.c:263`） | `addr`,`size`,`prot`（依 `commit`），`-1`,`0` | `false` 表示成功设置保护位；`true` 表示失败或地址漂移 | `pages_commit`（`:282`）→`commit=true`；`pages_decommit`（`:287`）→`commit=false` |
| `madvise(addr,size,MADV_FREE\|JEMALLOC_MADV_FREE)`（`src/pages.c:311-317`） | `addr`,`size`,`advice` | `!=0`→`true`（失败）；`0`→`false` | `JEMALLOC_PURGE_MADVISE_FREE`；`pages_can_purge_lazy` 与 `_runtime` 均为真（`:296-305`） |
| `madvise(addr,size,MADV_DONTNEED)`（lazy 变体，`src/pages.c:320`；forced，`src/pages.c:337`） | `addr`,`size`,`MADV_DONTNEED` | `!=0`→`true` | lazy 需 `JEMALLOC_PURGE_MADVISE_DONTNEED && !..._ZEROS`；forced 需 `_DONTNEED && _ZEROS`，否则 `JEMALLOC_MAPS_COALESCE` 走 `pages_commit`（`:335-340`） |
| `VirtualAlloc(addr,size,MEM_RESET,PAGE_READWRITE)`（Win，`src/pages.c:308`） | `addr`,`size`,`MEM_RESET`,`PAGE_READWRITE` | 恒 `false` | `pages_purge_lazy` 的 `_WIN32` 分支 |
| `madvise(addr,size,MADV_HUGEPAGE)`（`src/pages.c:353`） | `addr`,`size`,`MADV_HUGEPAGE` | `!=0`→`true`（失败）；无 `JEMALLOC_HAVE_MADVISE_HUGE` 时恒 `true`（`:355`） | `pages_huge`（`:360`，`aligned=true`）与 `pages_huge_unaligned`（`:365`，`aligned=false`）共用 `pages_huge_impl`（`:347`） |
| `madvise(addr,size,MADV_NOHUGEPAGE)`（`src/pages.c:377`） | `addr`,`size`,`MADV_NOHUGEPAGE` | `!=0`→`true`；无 `MADV_HUGEPAGE` 支持时恒 `false`（`:379`） | `pages_nohuge`（`:384`）/ `pages_nohuge_unaligned`（`:389`） |
| `madvise(addr,size,MADV_DONTDUMP\|MADV_DODUMP)`（`src/pages.c:398/:409`） | `addr`,`size`,`advice` | `!=0`→`true`；宏缺失恒 `false`（`:400/:411`） | `pages_dontdump:394` / `pages_dodump:405`；`PAGE_ADDR2BASE`/`PAGE_CEILING` 断言 |
| `pages_set_thp_state`（`src/pages.c:530-546`） | `ptr`,`size` | `opt_thp==default \|\| opt_thp==init_system_thp_mode`→直接返回；`always` 且系统非 `never`→`pages_huge_unaligned`（`:540`）；`never`→`pages_nohuge_unaligned`（`:544`） | 调用点 `src/extent.c:1244-1245`，门为 `have_madvise_huge && ret!=NULL` |

THP 模式名数组 `thp_mode_names[] = {"default","always","never","not supported"}`（`src/pages.c:31-36`），默认 `opt_thp = THP_MODE_DEFAULT`（`:37`）。内核对 `MADV_*` 的实际页合并行为为 `[外部黑盒：本仓未包含服务端实现]`。

## 4. `extent_alloc_mmap` / `extent_alloc_dss` 实参装配与返回解包

| 函数（锚点） | 关键实参 / 计算 | 返回解包 | 失败 / errno |
|---|---|---|---|
| `extent_alloc_mmap(new_addr,size,alignment,&zero,&commit)`（`src/extent_mmap.c:22-34`） | `assert(alignment == ALIGNMENT_CEILING(alignment, PAGE))`（`:24`）；`ret = pages_map(new_addr, size, alignment, commit)`（`:25`） | `NULL`→`NULL`（`:26-28`）；成功且 `*commit`→`*zero=true`（`:30-32`），表示映射页已归零 | 错误码由 `pages_map`→`mmap` 透传，本函数不读 `errno` |
| `extent_dalloc_mmap(addr,size)`（`src/extent_mmap.c:37-42`） | `!opt_retain` 时 `pages_unmap(addr,size)`（`:38-40`） | 返回 `opt_retain`（`:41`），告知调用方内存是否仍被保留 | `opt_retain` 默认由 `JEMALLOC_RETAIN` 决定（`:11-17`） |
| `extent_alloc_dss(tsdn,arena,new_addr,size,alignment,&zero,&commit)`（`src/extent_dss.c:110-229`） | `cassert(have_dss)`（`:114`）；`(intptr_t)size<0`→`NULL`（`:122-124`）；`extent_alloc` 取元数据（`:126`）；`extent_dss_max_update(new_addr)` 内 `sbrk(0)`（`:97,:139`）；`gap_addr_page=PAGE_CEILING(max_cur)`（`:149-150`），`ret=ALIGNMENT_CEILING(gap_addr_page,alignment)`（`:151-152`），`dss_next=ret+size`（`:166`），`incr=gap_size_subpage+size`（`:175`） | `dss_prev==max_cur`→成功（`:182`）：更新 `dss_max`（`:184`），`!*commit` 时 `*commit=pages_decommit(ret,size)`（`:193-195`），`*zero&&*commit` 时 `extent_purge_forced_wrapper` 失败则 `memset(ret,0,size)`（`:196-210`），返回 `ret`（`:211`） | 回绕 `ret<max_cur \|\| dss_next<max_cur`→`label_oom`（`:167-170`）；`dss_prev==(void*)-1`→置 `dss_exhausted=true`（`:217-221`）；`label_oom` 释放 gap 并 `NULL`（`:225-228`） |
| `extent_dss_sbrk(increment)`（`src/extent_dss.c:42-49`） | 有 `JEMALLOC_DSS` 时 `sbrk(increment)`（`:44`），否则 `not_implemented()`+`NULL`（`:46-47`） | 返回值即 `sbrk` 返回值；`extent_dss_boot` 以 `sbrk(0)` 设 `dss_base`/`dss_max`（`:265-268`） | `(void*)-1` 表示失败（`extent_dss_max_update:98-100`） |
| `extent_dss_prec_set(dss_prec)`（`src/extent_dss.c:63-69`） | `!have_dss`→返回 `dss_prec != dss_prec_disabled`；否则 `atomic_store_u(&dss_prec_default, prec, ATOMIC_RELEASE)` | 返回 `false` 表示设置成功 | `dss_prec_names[]={"disabled","primary","secondary","N/A"}`（`:14-19`） |

`extent_alloc_core` 的策略顺序为：DSS primary → mmap → DSS secondary（`src/extent.c:1217-1232`），`dss_prec` 来自 `arena->dss_prec`（`src/extent.c:1241-1243`）。DSS 元数据失败的 `errno` 不在本函数内重置。

## 5. 启动探测与配置来源

| 探测 / 配置（锚点） | 请求实参 | 解析结果 | 失败处理 |
|---|---|---|---|
| `os_page_detect`（`src/pages.c:417-435`） | `sysconf(_SC_PAGESIZE)`（`:429`）；Win `GetSystemInfo(&si).dwPageSize`（`:420-421`）；FreeBSD `getpagesize()`（`:427`） | `os_page` | `==-1`→`LG_PAGE`（`:430-432`）；`os_page > PAGE`→`malloc_write("<jemalloc>: Unsupported system page size\n")`+`abort`（`:601-606`） |
| `os_overcommits_proc`（`src/pages.c:469-526`） | `syscall(SYS_open,"/proc/sys/vm/overcommit_memory",O_RDONLY\|O_CLOEXEC)`（`:475-476`）或 `open(...)`（`:496`）；`malloc_read_fd(fd,&buf,1)`（`:509`）；`syscall(SYS_close,fd)`/`close`（`:511-514`） | `buf[0]=='0' \|\| buf[0]=='1'`→overcommit 真（`:525`） | `fd==-1`/`nread<1`→`false`（`:505-506,:516-518`） |
| `os_overcommits_sysctl`（`src/pages.c:439-459`） | `sysctl(mib={CTL_VM,VM_OVERCOMMIT},...,&vm_overcommit,...)`（`:447-449`）或 `sysctlbyname("vm.overcommit",...)`（`:453`） | `(vm_overcommit & 0x3)==0`（`:458`） | `!=0`→`false`（`:450,:454`） |
| `init_thp_state`（`src/pages.c:549-596`） | `open/syscall(SYS_open,"/sys/kernel/mm/transparent_hugepage/enabled",O_RDONLY)`（`:564-567`）；`malloc_read_fd`（`:573`） | 与 `"always [madvise] never\n"` / `"[always] madvise never\n"` / `"always madvise [never]\n"`（`:558-560`）比对，写入 `init_system_thp_mode`（`:584-592`） | `!have_madvise_huge` 且 `metadata_thp_enabled()&&opt_abort`→`malloc_write("<jemalloc>: no MADV_HUGEPAGE support\n")`+`abort`（`:550-555`）；其余 `label_error`→`opt_thp=init_system_thp_mode=thp_mode_not_supported`（`:595`） |
| `obtain_malloc_conf`（`src/jemalloc.c:938-1018`） | `which_source=0..3`；`case 2`：`readlink(linkname,buf,PATH_MAX)`（`:980`）或 `readlinkat(AT_FDCWD,linkname,buf,PATH_MAX)`（`:982`），`linkname="/etc/malloc.conf"`（或 `"/etc/"JEMALLOC_PREFIX"malloc.conf"`，`:967-973`）；`case 3`：`jemalloc_secure_getenv("MALLOC_CONF")`（`:995-1003`） | `buf[linklen]='\0'` 后返回 `buf`（`:991-992`）；四个来源按 `config_malloc_conf`→`je_malloc_conf`→文件符号链接名→环境变量顺序解析，重复键以最后者为准 | `linklen==-1`→`linklen=0` 且 `set_errno(saved_errno)`（`:984-989`）；env 缺失→`NULL`（`:1008-1011`） |
| `jemalloc_secure_getenv(name)`（`src/jemalloc.c:712-723`） | 有 `JEMALLOC_HAVE_SECURE_GETENV`→`secure_getenv(name)`（`:714`）；否则 `JEMALLOC_HAVE_ISSETUGID` 时先 `issetugid()!=0` 返回 `NULL`（`:717-719`），再 `getenv(name)`（`:721`） | `char *` 配置串或 `NULL` | setuid 上下文直接丢弃环境变量 |

相关运行期键（解析锚点 `src/jemalloc.c`）：`confirm_conf`（`:1170`）、`abort`（`:1175`）、`abort_conf`（`:1176`；`opt_abort_conf && had_conf_error`→`malloc_abort_invalid_conf()` `:1432-1434`）、`metadata_thp`（`:1177-1192`，取值来自 `metadata_thp_mode_names`）、`retain`（`:1194`）、`dss`（`:1195-1205`，调用 `extent_dss_prec_set` `:1201`，错误 `"Error setting dss"`）、`thp`（`:1393-1410`，无 `have_madvise_huge` 报 `"No THP support"` `:1398-1401`）。构建期开关（字面见 `configure.ac`）：`JEMALLOC_RETAIN`、`JEMALLOC_DSS`、`JEMALLOC_PROC_SYS_VM_OVERCOMMIT_MEMORY`（`configure.ac:649,:663`）、`JEMALLOC_SYSCTL_VM_OVERCOMMIT`、`JEMALLOC_USE_SYSCALL`、`JEMALLOC_READLINKAT`（`configure.ac:1433`）、`JEMALLOC_HAVE_MADVISE_HUGE`（`configure.ac:2055`）、`JEMALLOC_PURGE_MADVISE_FREE`（`configure.ac:2010,:2014`）、`JEMALLOC_MAPS_COALESCE`（`configure.ac:1303`）、`JEMALLOC_PREFIX`。

## 6. 共变耦合与修改清单（Co-Change Playbook）

- 改 `mmap_flags` 初始化或 `MAP_NORESERVE` 追加（`src/pages.c:610,:617-620`）→ 同步 `os_pages_map:79`、`pages_commit_impl:263`、`pages_map` FreeBSD 分支 `:194-204` 三处 `mmap` 实参；并复核 overcommit 探测 `os_overcommits_proc:469`。
- 改 `PAGES_PROT_COMMIT`/`PAGES_PROT_DECOMMIT`（`src/pages.c:25-26`）→ 联动 `:77`、`:193`、`:262` 三处 `prot` 派生；与 `pages_decommit` 的 URI 语义（ committing/decommitting）一起回归。
- 改 `pages_map` 对齐策略（`src/pages.c:179-238`）→ 联动 `extent_alloc_mmap` 的 `assert(alignment == ALIGNMENT_CEILING(alignment,PAGE))`（`src/extent_mmap.c:24`）、`extent_alloc_default` 的 `ALIGNMENT_CEILING(..., PAGE)`（`src/extent.c:1265`）、`large_palloc` 的 `sz_sa2u`（`src/large.c:30`）。
- 改 `extent_alloc_dss` 的 `incr`/gap 计算（`src/extent_dss.c:149-181`）→ 同步 `extent_dss_max_update` 的固定地址约束（`:102-105`）与 `dss_next` 回绕守卫（`:167-170`）；并复核 `dss` 配置解析 `src/jemalloc.c:1195-1205`。
- 改 `obtain_malloc_conf` 来源顺序或 `readlink` 缓冲（`src/jemalloc.c:938-1018`）→ 同步 `MALLOC_CONF_NSOURCES=4`（`:935`）、`malloc_conf_init_helper`（`:1021`）与 `jemalloc_secure_getenv`（`:712`）；`PATH_MAX` 缓冲溢出需保持 `buf[linklen]='\0'`（`:991`）。
- 改 THP 判定（`init_thp_state:549`、`pages_set_thp_state:530`）→ 联动 `thp` 键解析 `src/jemalloc.c:1393-1410`、`metadata_thp` `:1177-1192`、以及 `src/extent.c:1244-1245` 的 `have_madvise_huge` 门。
- 验证：`make tests && make check`（`Makefile.in:529,:553`）；重点覆盖 `dss=disabled/primary/secondary`、`retain=true/false`、`thp=always|never`、非法 `MALLOC_CONF` 与 `/etc/malloc.conf` 缺失路径；本页为静态追踪，未执行构建，内核侧行为为 `[外部黑盒：本仓未包含服务端实现]`。
