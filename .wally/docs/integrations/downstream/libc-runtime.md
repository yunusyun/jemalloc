---
kind: downstream-contract
scope: libc-runtime
status: draft
origin: generated
source_paths: [src/mutex.c, src/tsd.c, src/jemalloc.c, src/safety_check.c, src/malloc_io.c]
symbols: [malloc_mutex_lock, tsd_fetch_slow, jemalloc_secure_getenv, jemalloc_prefork, jemalloc_postfork_child, safety_check_fail, malloc_write]
upstream: [integrations/index.md, integrations/topology-and-mapping.md]
downstream: []
---

# libc 运行时下游契约

_scope: libc-runtime_。边界为 pthread mutex、atfork、线程本地存储（TLS/TSD）、`getenv` 与 `write`/`abort`；libc/pthread 与内核实现为 `[外部黑盒：本仓未包含服务端实现]`。上游分配路径见 [入站到出站参数映射](../topology-and-mapping.md)，总览见 [integrations/index](../index.md)。

## 1. 下游边界与符号级调用树

```text
[mutex 初始化]
  malloc_mutex_init(mutex,name,rank,lock_order) -> src/mutex.c:135
    ├─ pthread_mutexattr_init(&attr) -> src/mutex.c:162
    ├─ pthread_mutexattr_settype(&attr, MALLOC_MUTEX_TYPE) -> src/mutex.c:165（PTHREAD_MUTEX_DEFAULT, include/jemalloc/internal/mutex.h:124）
    ├─ [outbound] pthread_mutex_init(&mutex->lock,&attr) -> src/mutex.c:166
    └─ pthread_mutexattr_destroy(&attr) -> src/mutex.c:170
[mutex 快/慢路径]
  malloc_mutex_lock(tsdn,mutex) -> include/jemalloc/internal/mutex.h:217
    ├─ malloc_mutex_trylock_final -> include/jemalloc/internal/mutex.h:160 -> MALLOC_MUTEX_TRYLOCK -> pthread_mutex_trylock（include/jemalloc/internal/mutex.h:93）
    └─ [失败] malloc_mutex_lock_slow(mutex) -> src/mutex.c:47 -> spin_cpu_spinwait -> malloc_mutex_lock_final -> include/jemalloc/internal/mutex.h:154 -> pthread_mutex_lock（include/jemalloc/internal/mutex.h:91）
  malloc_mutex_unlock(tsdn,mutex) -> include/jemalloc/internal/mutex.h:230 -> MALLOC_MUTEX_UNLOCK -> pthread_mutex_unlock（include/jemalloc/internal/mutex.h:92）
[atfork]
  malloc_init_hard -> pthread_atfork(jemalloc_prefork,jemalloc_postfork_parent,jemalloc_postfork_child) -> src/jemalloc.c:1584
    ├─ jemalloc_prefork -> src/jemalloc.c:3781 -> malloc_mutex_prefork -> src/mutex.c:185 -> malloc_mutex_lock
    ├─ jemalloc_postfork_parent -> src/jemalloc.c:3855 -> malloc_mutex_postfork_parent -> src/mutex.c:190 -> malloc_mutex_unlock
    └─ jemalloc_postfork_child -> src/jemalloc.c:3894 -> malloc_mutex_postfork_child -> src/mutex.c:195
        └─ malloc_mutex_init(...) 重建锁；失败 -> malloc_printf("<jemalloc>: Error re-initializing mutex in child\n") + abort（src/mutex.c:199-205）
[TLS/TSD]
  tsd_fetch() -> tsd_fetch_impl -> tsd_get -> pthread_getspecific(tsd_tsd) -> include/jemalloc/internal/tsd_generic.h:63
    └─ [state != nominal] tsd_fetch_slow(tsd,minimal) -> src/tsd.c:270 -> tsd_set -> pthread_setspecific（include/jemalloc/internal/tsd_generic.h:55）
  pthread_key_create(&tsd_tsd, tsd_cleanup_wrapper) -> include/jemalloc/internal/tsd_generic.h:94
  _malloc_thread_cleanup -> src/tsd.c:333 -> cleanups[i]() -> tsd_do_data_cleanup -> src/tsd.c:363
[env / 输出 / 终止]
  obtain_malloc_conf(source=3) -> jemalloc_secure_getenv(name) -> src/jemalloc.c:712 -> secure_getenv/issetugid/getenv
  safety_check_fail(format,...) -> src/safety_check.c:10 -> malloc_vsnprintf -> malloc_write -> abort() | safety_check_abort(buf)
  malloc_write(s) -> src/malloc_io.c:83 -> je_malloc_message(NULL,s) | wrtmessage(NULL,s) -> malloc_write_fd(STDERR_FILENO,s,strlen(s)) -> write(2)
```

## 2. pthread mutex 请求表与返回解包

| libc/pthread 调用 | 实参（锚点） | 返回值处理 | 平台替换 |
|---|---|---|---|
| `pthread_mutexattr_init` | `&attr`，`src/mutex.c:162` | `!=0`→`return true`（`:163-164`） | — |
| `pthread_mutexattr_settype` | `&attr`, `MALLOC_MUTEX_TYPE`，`src/mutex.c:165` | 忽略返回值 | `PTHREAD_MUTEX_DEFAULT`，`include/jemalloc/internal/mutex.h:124` |
| `pthread_mutex_init` | `&mutex->lock`, `&attr`，`src/mutex.c:166` | `!=0`→`pthread_mutexattr_destroy(&attr)`+`return true`（`:167-169`） | Win `InitializeSRWLock`/`InitializeCriticalSectionAndSpinCount`（`:140-145`）；`JEMALLOC_OS_UNFAIR_LOCK` 直赋 `OS_UNFAIR_LOCK_INIT`（`:147-148`） |
| `pthread_mutexattr_destroy` | `&attr`，`src/mutex.c:170` | 忽略返回值 | — |
| `pthread_mutex_lock` | `&mutex->lock`，宏 `MALLOC_MUTEX_LOCK(mutex)`，`include/jemalloc/internal/mutex.h:91`，调用点 `include/jemalloc/internal/mutex.h:155` | `void` 语义；成功后 `atomic_store_b(&mutex->locked,true,ATOMIC_RELAXED)`（`include/jemalloc/internal/mutex.h:156`） | Win `AcquireSRWLockExclusive`/`EnterCriticalSection`（`include/jemalloc/internal/mutex.h:78,:82`）；Darwin `os_unfair_lock_lock`（`:87`） |
| `pthread_mutex_trylock` | `&mutex->lock`，宏 `MALLOC_MUTEX_TRYLOCK(mutex)`=`pthread_mutex_trylock(...) != 0`，`include/jemalloc/internal/mutex.h:93`，调用点 `include/jemalloc/internal/mutex.h:161,:181,:220` | 返回 `bool`：`true` 表示未取到锁；`include/jemalloc/internal/mutex.h:220-223` 失败转 `malloc_mutex_lock_slow` | Win `!TryAcquireSRWLockExclusive` / `!TryEnterCriticalSection`（`include/jemalloc/internal/mutex.h:80,:84`） |
| `pthread_mutex_unlock` | `&mutex->lock`，宏 `MALLOC_MUTEX_UNLOCK(mutex)`，`include/jemalloc/internal/mutex.h:92`，调用点 `include/jemalloc/internal/mutex.h:234` | 先 `atomic_store_b(&mutex->locked,false,...)`+`witness_unlock`（`include/jemalloc/internal/mutex.h:231-232`），再按需 `isthreaded` 才真正解锁（`include/jemalloc/internal/mutex.h:233-235`） | Win `ReleaseSRWLockExclusive`/`LeaveCriticalSection`（`include/jemalloc/internal/mutex.h:79,:83`） |
| `spin_cpu_spinwait` | 无参；`ncpus==1` 时跳过（`src/mutex.c:51-53`），最多 `MALLOC_MUTEX_MAX_SPIN` 次（`:55-63`） | 自旋后仍失败走 `malloc_mutex_lock_final`（`:67,:85`） | 非 pthread 调用，属 CPU 自旋原语 |

**锚点更正**：上下文清单中 `src/mutex.c:186`/`:191` 所指并非 `malloc_mutex_lock`/`malloc_mutex_unlock` 本体——`:186` 为 `malloc_mutex_prefork` 内的调用、`:191` 为 `malloc_mutex_postfork_parent` 内的调用；本体为 `include/jemalloc/internal/mutex.h:217` 与 `:230`（`JEMALLOC_ALWAYS_INLINE`）。`src/mutex.c` 本身仍以 `malloc_mutex_lock_slow:47`、`malloc_mutex_init:135`、`malloc_mutex_prefork:185`、`malloc_mutex_postfork_child:195`、`malloc_mutex_boot:211` 承担 pthread 装配。

## 3. atfork：锁序与失败表

| 阶段（锚点） | 下游动作 / 实参 | 顺序与失败处理 |
|---|---|---|
| 注册 `src/jemalloc.c:1584-1591` | `pthread_atfork(jemalloc_prefork, jemalloc_postfork_parent, jemalloc_postfork_child)`；条件排除 `JEMALLOC_ZONE`/`_WIN32`/`__native_client__`（`:1581-1582`） | `!=0`→`malloc_write("<jemalloc>: Error in pthread_atfork()\n")`（`:1586`）+ `if(opt_abort) abort()`（`:1587-1589`）+`return true`（`:1590`） |
| prefork `src/jemalloc.c:3781-3851` | `tsd_fetch()`（`:3798`）；`witness_prefork`（`:3802`）；`ctl_prefork`（`:3804`）；`tcache_prefork`（`:3805`）；`malloc_mutex_prefork(&arenas_lock)`（`:3806`） | 8 阶段 arena 锁（外层 `i=0..7`，内层 `j=0..narenas`，`:3815-3848`），按 `arena_prefork0..7` 固定锁序；`background_thread_prefork0/1`（`:3808,:3812`）、`prof_prefork0/1`（`:3810,:3849`）、`tsd_prefork`（`:3850`） |
| `malloc_mutex_prefork` `src/mutex.c:185-187` | `malloc_mutex_lock(tsdn, mutex)`（`:186`） | 直接复用第 2 节快/慢路径，无额外返回码 |
| postfork parent `src/jemalloc.c:3855-3891` | `tsd_postfork_parent`（`:3873`）；`witness_postfork_parent`（`:3875`）；逐 arena `arena_postfork_parent`（`:3881`）；`prof_postfork_parent`（`:3884`）；`background_thread_postfork_parent`（`:3886`）；`malloc_mutex_postfork_parent(&arenas_lock)`（`:3888`）；`tcache_postfork_parent`（`:3889`）；`ctl_postfork_parent`（`:3890`） | 与 prefork 逆序释放；`malloc_mutex_postfork_parent` 调 `malloc_mutex_unlock`（`src/mutex.c:190-192`） |
| postfork child `src/jemalloc.c:3894-3920` | `tsd_postfork_child`（`:3902`）；`witness_postfork_child`（`:3904`）；逐 arena `arena_postfork_child`（`:3910`）；`prof_postfork_child`（`:3913`）；后台线程（`:3915`）；`malloc_mutex_postfork_child(&arenas_lock)`（`:3917`）；`tcache_postfork_child`（`:3918`）；`ctl_postfork_child`（`:3919`） | 子进程不继承其他线程持锁状态：`malloc_mutex_postfork_child` 调 `malloc_mutex_init(mutex, mutex->witness.name, mutex->witness.rank, mutex->lock_order)`（`src/mutex.c:199-200`）；失败写 `"<jemalloc>: Error re-initializing mutex in child\n"`（`:201-202`）并 `if(opt_abort) abort()`（`:203-205`）；`JEMALLOC_MUTEX_INIT_CB` 分支仅 `malloc_mutex_unlock`（`:196-198`） |
| atexit 注册 `src/jemalloc.c:1511-1512` | `atexit(stats_print_atexit)` | `!=0`→`malloc_write("<jemalloc>: Error in atexit()\n")` |

## 4. TLS/TSD：pthread_key 与状态机

| 调用（锚点） | 实参 | 返回解包 / 失败 |
|---|---|---|
| `pthread_key_create` | `&tsd_tsd`, `tsd_cleanup_wrapper`，`include/jemalloc/internal/tsd_generic.h:94` | `!=0`→`return true`（`:95`）；成功 `tsd_wrapper_set(&tsd_boot_wrapper)`（`:97`）并 `tsd_booted=true`（`:98`） |
| `pthread_setspecific` | `tsd_tsd`, `(void *)wrapper`，`include/jemalloc/internal/tsd_generic.h:40` 与 `:55` | `!=0`→`malloc_write("<jemalloc>: Error setting TSD\n")`：cleanup 路径再 `pthread_setspecific` 触发下一轮清理（`:40-47`）；`tsd_wrapper_set` 路径直接 `abort()`（`:56-58`） |
| `pthread_getspecific` | `tsd_tsd`，`include/jemalloc/internal/tsd_generic.h:63` | 返回 `tsd_wrapper_t *`；`NULL` 且 `init==true`→经 `tsd_init_check_recursion` 防递归后 `malloc_tsd_malloc(sizeof(tsd_wrapper_t))`（`:65-73`），分配失败写 `"<jemalloc>: Error allocating TSD\n"`+`abort()`（`:75-78`），随后 `tsd_wrapper_set`+`tsd_init_finish`（`:86-87`） |
| `tsd_fetch_slow(tsd,minimal)` | `tsd`, `minimal`，`src/tsd.c:270-316` | 状态机：`nominal_slow` 空操作（`:273-279`）；`nominal_recompute`→`tsd_slow_update`（`:280-281`）；`uninitialized` 且 `tsd_booted`→`tsd_state_set(nominal)`+`tsd_slow_update`+`tsd_set`+`tsd_data_init`（`:282-290`），`minimal`→`minimal_initialized`+`tsd_data_init_nocleanup`（`:291-295`）；`minimal_initialized` 且非 minimal→状态回 `nominal`、reentrancy 减 1、重算（`:296-303`）；`purgatory`→`reincarnated`+`nocleanup`（`:307-310`）；返回 `tsd`（`:315`） |
| `_malloc_thread_cleanup` | 无参；`JEMALLOC_MALLOC_THREAD_CLEANUP` 或 `_WIN32` 下导出，`src/tsd.c:328-353` | 置全部 `pending[i]=true`（`:337-339`），循环调用 `cleanups[i]()` 直到无 `pending`（`:341-351`）；注册入口 `malloc_tsd_cleanup_register` `:356-360` |
| `tsd_do_data_cleanup` | `tsd`，`src/tsd.c:363-370` | 依次 `prof_tdata_cleanup`、`iarena_cleanup`、`arena_cleanup`、`arenas_tdata_cleanup`、`tcache_cleanup`、`witnesses_cleanup`（`:364-369`）；`tsd_cleanup` 在 `:373` |
| `malloc_tsd_malloc` / `malloc_tsd_dalloc` | `size`（内部 `CACHELINE_CEILING(size)`）；`wrapper` | 分别 `a0malloc`（`src/tsd.c:319-321`）与 `a0dalloc`（`:324-326`），规避 boot 期重入 |

TLS 存储位置依配置切换：`JEMALLOC_MALLOC_THREAD_CLEANUP` 用 `tsd_tls` 直接线程变量（`src/tsd.c:19-22`），`JEMALLOC_TLS` 用 `tsd_tls` + `pthread_key_t tsd_tsd`（`:23-26`），Win 用 `DWORD tsd_tsd`（`:27-30`），通用实现用 `pthread_key_t tsd_tsd` + `tsd_init_head` 互斥锁（`:43-47`）。fastpath 判定为 `tsd_state_get(tsd)==tsd_state_nominal`（`include/jemalloc/internal/tsd.h:334-342`），`tsd_fetch_impl` 在非 nominal 时转 `tsd_fetch_slow`（`include/jemalloc/internal/tsd.h:344-360`）。

## 5. 环境变量、输出与终止边界

| 下游调用（锚点） | 实参 | 返回 / 失败处理 |
|---|---|---|
| `secure_getenv(name)` | `name`，`src/jemalloc.c:713-714` | 有 `JEMALLOC_HAVE_SECURE_GETENV` 直接返回 libc 结果 |
| `issetugid()` | 无参，`src/jemalloc.c:717-719` | `!=0`→`return NULL`（丢弃 setuid 进程的 `MALLOC_CONF`） |
| `getenv(name)` | `name`，`src/jemalloc.c:721` | 返回串或 `NULL`；`obtain_malloc_conf` 的 `case 3` 据此返回配置串（`:995-1011`） |
| `je_malloc_message(NULL, s)` | `cbopaque=NULL`, `s`，`src/malloc_io.c:84-85` | 用户钩子存在时优先调用；返回 `void` |
| `wrtmessage(NULL, s)` | `NULL`, `s`，`src/malloc_io.c:72-74` | 转 `malloc_write_fd(STDERR_FILENO, s, strlen(s))` |
| `write(fd, buf, count)` / `syscall(SYS_write, fd, buf, count)` | `fd=STDERR_FILENO`, `buf=s`, `count=strlen(s)`，`include/jemalloc/internal/malloc_io.h:67-85`（`syscall` 分支 `:77`，`write` 分支 `:79`） | `ssize_t result` 被丢弃（`:85`）；`malloc_write` 无返回（`src/malloc_io.c:83-89`） |
| `strerror_r(err, buf, buflen)` | `err`, `buf`, `buflen`，`src/malloc_io.c:96-110` | GNU 变体返回 `char *` 且必要时 `strncpy`（`:102-107`）；否则返回 `int`（`:109`）；供 `buferror` 把 `errno` 转成可读串 |
| `malloc_vsnprintf(buf, MALLOC_PRINTF_BUFSIZE, format, ap)` | `buf`, 容量, `format`, `va_list`，`src/safety_check.c:11-16`；本体 `src/malloc_io.c:327` | 格式化后交 `malloc_write` |
| `safety_check_abort(buf)` | 自定义钩子（默认 `NULL`），`src/safety_check.c:4,:6-8,:22` | 非空时替代默认终止；由 `safety_check_set_abort` 安装 |
| `abort()` | 无参，`src/safety_check.c:20`；`src/mutex.c:204`；`src/jemalloc.c:1588` | 无返回；`opt_xmalloc` 路径在 `src/jemalloc.c:2180/:2207` 同样以 `malloc_write`+`abort` 终结 |

`safety_check_fail` 的完整契约（`src/safety_check.c:10-24`）：先 `malloc_vsnprintf` 到 `MALLOC_PRINTF_BUFSIZE` 栈缓冲（`:11-16`），若 `safety_check_abort==NULL` 则 `malloc_write(buf)`+`abort()`（`:18-20`），否则调用钩子（`:21-23`）。`malloc_printf` 位于 `src/malloc_io.c:659`，复用同一 `malloc_write` 出口。

## 6. 共变耦合与修改清单（Co-Change Playbook）

- 改 `MALLOC_MUTEX_*` 平台宏（`include/jemalloc/internal/mutex.h:78-93`）→ 同步 `malloc_mutex_init` 的类型/初始化分支（`src/mutex.c:138-171`）、`malloc_mutex_lock/unlock` 内联体（`include/jemalloc/internal/mutex.h:217-236`）、`malloc_mutex_postfork_child` 的重建逻辑（`src/mutex.c:195-208`）；漏改会在 fork 子进程留下不可解锁的 pthread 锁。
- 改 atfork 锁序（`src/jemalloc.c:3815-3848` 的 8 阶段）→ 同步 `witness_prefork/postfork`（`:3802,:3904`）、`src/mutex.c:185-208` 的 prefork/postfork 三函数，以及 `tcache_prefork`/`ctl_prefork` 等子系统的配对调用；prefork 与 postfork parent 必须严格逆序。
- 改 TSD 实现选择（`src/tsd.c:19-54` 的四种分支）→ 联动 `include/jemalloc/internal/tsd_generic.h:94-128`（`pthread_key_create`/`setspecific`/`getspecific`）、`tsd_fetch_slow` 状态机（`src/tsd.c:270-316`）、`_malloc_thread_cleanup`（`:333-352`）与 `tsd_do_data_cleanup`（`:363-370`）；新增 cleanup 项须同步 `MALLOC_TSD_CLEANUPS_MAX`。
- 改输出/终止路径（`src/malloc_io.c:72-89` 与 `src/safety_check.c:10-24`）→ 联动 `include/jemalloc/internal/malloc_io.h:67-85` 的 `write`/`syscall` 选择、所有 `malloc_printf` 调用点、以及 `src/jemalloc.c` 的 `opt_xmalloc` OOM/EINVAL 出口（`:2177-2224`）；`je_malloc_message` 钩子签名（`src/malloc_io.c:76`）不得变更。
- 改 `jemalloc_secure_getenv`（`src/jemalloc.c:712-723`）→ 同步 `obtain_malloc_conf` 的 `case 3`（`:994-1012`）、`MALLOC_CONF_NSOURCES=4`（`:935`）与构建开关 `JEMALLOC_HAVE_SECURE_GETENV`/`JEMALLOC_HAVE_ISSETUGID`。
- 验证：`make tests && make check`（`Makefile.in:529,:553`）；重点覆盖 pthread_atfork 注册失败、fork 子进程锁重建、`MALLOC_CONF` 在 setuid 上下文被丢弃、`safety_check_set_abort` 自定义钩子；本页为静态追踪，未执行构建，pthread 与内核侧行为为 `[外部黑盒：本仓未包含服务端实现]`。
