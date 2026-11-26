---
kind: component
scope: architecture-index
status: draft
origin: generated
source_paths: [src/jemalloc.c, src/arena.c, src/extent.c, src/tcache.c, src/ctl.c, src/prof.c, src/base.c, src/pages.c, src/mutex.c, src/zone.c]
symbols: [je_malloc, arena_new, extents_alloc, tcache_create_explicit, ctl_init, prof_dump, base_new, pages_map]
upstream: [INDEX.md, overview/system-architecture.md]
downstream: [architecture/module-dep-matrix.md]
---

# 组件技术架构总表

按运行时框架模型汇总全仓技术模块，并把不单独成篇的 Tier 3 工具文件完整归档，确保源码目录 100% 纳入路由索引。分层拓扑见 [系统技术架构](../overview/system-architecture.md)，依赖边与爆炸半径见 [模块依赖矩阵](module-dep-matrix.md)。

## 1. 全量核心组件矩阵（Tier 1 & Tier 2）

| 组件标识 | 框架模型 | 源码路径 | 核心入口符号（`路径:行号`） | 技术能力与边界 | 直接上游 | 直接下游 | 详情文档 |
|---|---|---|---|---|---|---|---|
| `allocator-core` | `Core Engine` | `src/jemalloc.c`, `src/arena.c`, `src/bin.c`, `src/large.c`, `src/sz.c`, `src/sc.c` | `je_malloc` (`src/jemalloc.c:2323`), `arena_new` (`src/arena.c:1942`) | C API、尺寸类、tcache 快路径、small/large 分配 | 宿主程序 | `extent-and-os`, `tcache-and-tsd` | [allocator-core.md](allocator-core.md) |
| `extent-and-os` | `Data & Adapter` | `src/extent.c`, `src/extent_mmap.c`, `src/extent_dss.c`, `src/pages.c`, `src/base.c` | `extents_alloc` (`src/extent.c:537`), `pages_map` (`src/pages.c:179`), `base_new` (`src/base.c:350`) | extent 状态机、mmap/sbrk 对接、元数据 bump 分配 | `allocator-core` | OS VM（黑盒） | [extent-and-os.md](extent-and-os.md) |
| `tcache-and-tsd` | `Core Engine` | `src/tcache.c`, `src/tsd.c`, `src/rtree.c` | `tcache_create_explicit` (`src/tcache.c:486`), `tsd_fetch_slow` (`src/tsd.c:270`) | 线程本地无锁缓存、rtree 指针反查、TLS 生命周期 | `allocator-core` | `allocator-core` | [tcache-and-tsd.md](tcache-and-tsd.md) |
| `control-and-obs` | `API Service`（mallctl 树） | `src/ctl.c`, `src/stats.c`, `src/log.c`, `src/malloc_io.c` | `ctl_init` (`src/ctl.c:1118`), `stats_print` (`src/stats.c:1396`) | mallctl 动态分发、统计输出、日志与格式化 | 诊断工具 / 宿主程序 | arena/tcache 统计聚合 | [control-and-observability.md](control-and-observability.md) |
| `profiler-and-bg` | `Async/Cron Worker` | `src/prof.c`, `src/background_thread.c`, `src/hook.c` | `prof_dump` (`src/prof.c:1906`), `background_work` (`src/background_thread.c:483`), `hook_install` (`src/hook.c:48`) | 分配采样、堆转储、后台 decay 线程、hook 安装 | `allocator-core` | extent / 磁盘堆转储 | [profiler-and-background.md](profiler-and-background.md) |
| `sync-and-utils` | `Core Engine`（底座） | `src/mutex.c`, `src/witness.c`, `src/bitmap.c`, `src/ckh.c`, `src/nstime.c`, `src/safety_check.c`, `src/test_hooks.c` | `malloc_mutex_init` (`src/mutex.c:135`), `safety_check_fail` (`src/safety_check.c:10`) | 互斥锁、锁序校验、位图、哈希、时间、安全告警 | 全仓各模块 | 无 | [sync-and-utilities.md](sync-and-utilities.md) |
| `cli-tools` | `UI & CLI` | `bin/jemalloc-config.in`, `bin/jemalloc.sh.in`, `bin/jeprof.in`, `include/jemalloc/jemalloc.sh` | `usage` (`bin/jemalloc-config.in:3`) | 构建信息查询、LD_PRELOAD 包装、堆转储分析 | 用户终端 / CI | mallctl / 堆转储文件 | [cli-tools.md](cli-tools.md) |

## 2. 轻量辅助模块归档表（Tier 3 Utilities — 零遗漏兜底）

| 源码文件路径 | 核心导出符号 | 调用方 | 职责说明 |
|---|---|---|---|
| `src/arena.c` | `arena_stats_merge` (`:83`), `arena_tcache_fill_small` (`:1381`) | 控制面、tcache | arena 统计聚合与 small bin 填充 |
| `src/bin.c` | `bin_infos_init` (`:12`) | `sc_data_init` 之后启动期 | 由 `sc_data_t` 生成 bin 元数据 |
| `src/bitmap.c` | `bitmap_info_ngroups` (`:40`) | `bin.c`, `arena.c` | slab reg 空闲位图 |
| `src/ckh.c` | `ckh_try_insert` (`:205`), `ckh_grow` (`:257`) | `prof.c` | 采样上下文哈希容器 |
| `src/hash.c` | `hash` 系列（头文件内联） | `ckh.c`, `rtree.c` | 通用哈希函数 |
| `src/div.c` | 除法查表辅助 | `sc.c`, `sz.c` | 尺寸类计算的整数除法优化 |
| `src/nstime.c` | `nstime_update` (`:156`), `nstime_copy` | `arena.c`, `background_thread.c` | 单调时间运算，decay 截止期基础 |
| `src/prng.c` | PRNG 状态推进 | `prof.c` | 采样间隔随机化 |
| `src/ticker.c` | `ticker_trytick` | `tcache.c`, `src/jemalloc.c:2337` | tcache GC 计数回绕 |
| `src/mutex_pool.c` | mutex 池 | `prof.c` | 采样上下文分片锁 |
| `src/jemalloc_cpp.cpp` | `handleOOM` (`:44`), `newImpl` (`:77`) | C++ 应用 | `operator new` 兼容层与 OOM 策略 |
| `src/test_hooks.c` | 测试钩子桩 | 单测 | 可注入的 libc 调用替身 |
| `src/zone.c` | `zone_init` (`:332`), `zone_promote` (`:401`) | Darwin 平台 | 历史 malloc zone 适配，非 Linux 主链，见 [高危雷区](../operations/pitfalls-and-adr.md) |
| `src/log.c` | `log_var_matches_segment` (`:22`) | 全仓 `LOG()` | 日志变量过滤 |
| `src/malloc_io.c` | `malloc_vsnprintf` (`:327`), `je_malloc_message` (`:85`) | 全仓诊断输出 | 无依赖格式化与输出回调 |
| `src/safety_check.c` | `safety_check_fail` (`:10`) | `src/tcache.c:125` | 内部一致性硬失败 |
| `src/rtree.c` | `rtree_leaf_elm_lookup_hard` (`:223`) | `jemalloc.c`, `extent.c` | radix tree 指针→extent 元数据 |
| `src/extent_dss.c` / `src/extent_mmap.c` | `extent_alloc_dss` (`:110`) / `extent_alloc_mmap` (`:22`) | `extent.c` | 两种 OS 内存来源 |
| `include/jemalloc/internal/` | 全部 `*_types.h` / `*_structs.h` / `*_inlines*.h` | 全仓 | 类型、结构与内联快路径定义 |
| `include/msvc_compat/` | C99 兼容头 | MSVC 构建 | Windows 编译兼容 |
| `doc/` | `jemalloc.xml.in`, `*.xsl.in` | 构建安装 | man / HTML 文档源 |
| `scripts/` | `gen_run_tests.py`, `gen_travis.py` | CI | 测试矩阵生成（见 [构建与单测](../operations/dev-and-test.md)） |
| `msvc/`, `m4/`, `build-aux/` | 工程文件与 autotools 辅助 | 构建 | 平台工程与 configure 生成 |

## 3. 源码目录覆盖对账（未映射目录 = 0）

| 顶层/二级目录 | 归档去向 | 映射状态 |
|---|---|---|
| `src/`（35 个 `.c/.cpp`） | 上表 7 个组件 + Tier 3 表 | 已覆盖 |
| `include/jemalloc/` | [api-reference](../contracts/api-reference.md) + [glossary](../overview/glossary.md) | 已覆盖 |
| `include/jemalloc/internal/` | [领域模型](../business/domain-models/index.md) + Tier 3 表 | 已覆盖 |
| `include/msvc_compat/` | Tier 3 表 + [boundary](../operations/boundary.md) | 已覆盖 |
| `test/unit/`, `test/integration/`, `test/src/`, `test/stress/` | [dev-and-test](../operations/dev-and-test.md) | 已覆盖 |
| `bin/` | [cli-tools](cli-tools.md) + [deployment](../operations/deployment-and-runtime.md) | 已覆盖 |
| `doc/`, `scripts/`, `msvc/`, `m4/`, `build-aux/` | Tier 3 表 + [dev-and-test](../operations/dev-and-test.md) | 已覆盖 |

## 4. 共变耦合与修改清单（Co-Change Playbook）

1. 新增 `src/*.c`：先判定归属组件；若无可归属模块，登记进 Tier 3 表并把目录加入上表，禁止出现未映射源码。
2. 改组件边界（如把某函数从 arena 移到 extent）：同步本页组件矩阵、[模块依赖矩阵](module-dep-matrix.md)、对应 component 页的 `upstream/downstream` 与 [INDEX](../INDEX.md) 路由表。
3. 改 `bin/` 或 `include/jemalloc/jemalloc_macros.h.in`：同步 [cli-tools](cli-tools.md) 与 [api-reference](../contracts/api-reference.md)，设置 `DESTDIR` 指向临时目录后跑 `make install` 验证安装产物。
