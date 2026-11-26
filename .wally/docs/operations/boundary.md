---
kind: operations
scope: boundary
status: draft
origin: generated
source_paths: [src/pages.c, src/mutex.c, src/jemalloc.c, bin/jeprof.in, README]
symbols: [pages_map, malloc_mutex_lock, jemalloc_secure_getenv]
upstream: [INDEX.md]
downstream: []
---

# 系统边界、快照与仓外黑盒

## 1. 文档快照标识

| 项 | 值 |
|---|---|
| 仓库 | jemalloc C 库（本仓为源码维护源） |
| 分支 | `chore/wally-init-20261126-c74` |
| HEAD SHA | `6d281cef3cd747666b00c4557c0f42786f66ae89` |
| 文档语言 | zh-CN；路径、YAML 键与 C 符号保留原样 |
| 边界类型 | 库↔OS 内存/线程/libc、库↔进程加载器、库↔Perl 工具链 |

`README:1-3` 定性为通用 `malloc(3)` 实现；`README:5-11` 指向 `COPYING`、`INSTALL`、`ChangeLog` 与外部站点 URL `http://jemalloc.net/`。本页所有锚点均以该 SHA 为准；后续提交需要重新核验行号。

## 2. OS 内存边界（`pages_map`）

`pages_map`（`src/pages.c:179-195`）要求入参 `alignment >= PAGE` 且地址已按 alignment 对齐（`:180-181`），FreeBSD 分支使用 `MAP_EXCL` 类机制（`:183-194`）。实际系统映射下沉到 `os_pages_map`（`src/pages.c:54-79`）：Windows 用 `VirtualAlloc`（`:69-70`），其他平台用 `mmap`（`:78`），并显式说明不使用 `MAP_FIXED` 以免替换已有映射（`:72-75`）。`os_overcommits` 为真时强制 `*commit = true`（`:59-61`）。对齐不足时走 `pages_map_slow`（`src/pages.c:155-176`）：多映射 `size + alignment - os_page`（`:156`），防 `size_t` 回绕（`:158-160`），再 `os_pages_trim` 去前导（`:170`）。

```text
arena/extent -> src/pages.c:179 pages_map(addr,size,alignment,commit)
  ├─ [FreeBSD && MAP_EXCL] mmap 直接带对齐(:183-194)
  ├─ 其他 -> src/pages.c:54 os_pages_map -> [outbound] OS: mmap / VirtualAlloc
  └─ src/pages.c:155 pages_map_slow -> os_pages_map(:164) -> os_pages_trim(:170)
src/pages.c:241 pages_unmap / :287 pages_decommit / :549 init_thp_state -> [outbound] OS VM
```

`pages_unmap`（`src/pages.c:241`）、`pages_commit_impl`（`:249`）、`pages_decommit`（`:287`）、`os_overcommits_proc`（`:469`）与 `init_thp_state`（`:549`）同为 OS 边界出口。`src/pages.c` 之外，OS 相关映射细节见 [extent 与 OS 架构页](../architecture/extent-and-os.md)。

## 3. 线程/锁边界（`malloc_mutex_lock`）

`malloc_mutex_lock` 的**定义不在 `src/mutex.c`**：它是内联函数，字面定义在 `include/jemalloc/internal/mutex.h:217-227`（先 `witness_assert_not_owner`，`isthreaded` 时尝试 `malloc_mutex_trylock_final`、失败进 `malloc_mutex_lock_slow`，最后 `witness_lock`）；`malloc_mutex_unlock` 定义在 `include/jemalloc/internal/mutex.h:230-236`。`src/mutex.c:184-187` 的 `malloc_mutex_prefork` 与 `:189-192` 的 `malloc_mutex_postfork_parent` 是调用点；`:194-205` 的 `malloc_mutex_postfork_child` 在子进程重初始化失败时打印 `<jemalloc>: Error re-initializing mutex in child`（`:201-202`）并按 `opt_abort` 决定是否终止（`:203-204`）。`fork` 钩子注册点 `src/jemalloc.c:1584-1586`，全量锁遍历见 `src/jemalloc.c:3781-3851`。`pthread` 与 witness 实现均为仓外/系统能力，**本仓未包含其实现**。

## 4. libc/加载器与环境变量边界

`jemalloc_secure_getenv`（`src/jemalloc.c:712-723`）是对 libc 的封装：有 `JEMALLOC_HAVE_SECURE_GETENV` 时调 `secure_getenv`（`:713-714`）；否则若有 `JEMALLOC_HAVE_ISSETUGID` 且 `issetugid() != 0` 返回 NULL（`:716-719`）；最后回落 `getenv`（`:721`）。唯一调用点在 `obtain_malloc_conf` 的 `MALLOC_CONF` 分支（`src/jemalloc.c:1003`）。其他外部调用还包括 `readlink`/`readlinkat`（`src/jemalloc.c:980`、`:982`）、`atexit`（`:1511`）、`pthread_atfork`（`:1584`）、`malloc_zone_*`（Darwin，`src/zone.c:413-429`）——均为 `[外部黑盒：本仓未包含实现]`。

动态边界（进程启动时才确定）：

| 边界 | 字面证据 | 说明 |
|---|---|---|
| `LD_PRELOAD` 接管 | `bin/jemalloc.sh.in:7-9` | 运行脚本时设置 `@LD_PRELOAD_VAR@` 指向 `libjemalloc.@SOREV@`，仅影响该进程及其子进程 |
| 环境变量配置 | `src/jemalloc.c:994-1012` | `MALLOC_CONF` 在进程启动时读取，优先级最高 |
| mallctl 名称解析 | `src/ctl.c:1194-1289` | 名称→MIB 在运行期解析，拼写错误返回 `ENOENT` |
| 导出符号改写 | `include/jemalloc/jemalloc_protos.h.in:2-5` | `@je_@` 前缀在生成头文件时确定 |

## 5. 工具链与仓外依赖

`bin/jeprof.in:1` 为 `#! /usr/bin/env perl`：Perl 解释器、堆转储文件的读取与外部符号化能力均不在本仓，`[外部黑盒：本仓未包含实现]`。`autogen.sh:3-13` 调用 `autoconf` 与 `./configure`；`configure.ac:1-4` 声明 `AC_PREREQ(2.68)`，autotools 亦为仓外工具链。`README:11` 给出的 `http://jemalloc.net/` 是外部信息源，本仓不镜像其内容。

仓外边界不可被本仓单测闭合：验证 `pages_map` 需真实 OS VM 行为，验证 `malloc_mutex_lock` 需真实 pthread 调度，验证 `jeprof` 需 Perl 环境。相关验证入口与诚实声明见 [开发与测试](dev-and-test.md)、[缺陷规避与 ADR](pitfalls-and-adr.md) 与 [排障页](runbook-and-observability.md)。

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改 OS 映射路径：联动 `src/pages.c:54 / 155 / 179 / 241 / 249 / 287 / 469 / 549` 与 [extent 与 OS 架构页](../architecture/extent-and-os.md)，并注明未覆盖的平台。
2. 改锁实现：联动 `include/jemalloc/internal/mutex.h:217-236` 的定义、`src/mutex.c:184-205` 的 fork 调用点与 `src/jemalloc.c:3781-3919` 的钩子；注意 `src/mutex.c:186/191` 只是调用点而非定义。
3. 改环境变量读取：联动 `jemalloc_secure_getenv`（`src/jemalloc.c:712-723`）、`obtain_malloc_conf`（`:938`）与 [配置字典](../contracts/configuration.md)。
4. 改加载/工具链边界：联动 `bin/jemalloc.sh.in:7-9`、`bin/jeprof.in:1`、`autogen.sh:3-13`、`configure.ac:1-4`，并在 [部署页](deployment-and-runtime.md)同步安装前置条件。
5. 快照更新：分支或 SHA 变动时重核本页全部 `path:line`，禁止沿用旧行号。
