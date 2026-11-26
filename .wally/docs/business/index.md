---
kind: workflow
scope: business-index
status: draft
origin: generated
source_paths: [src/jemalloc.c, src/arena.c, src/prof.c, src/background_thread.c]
symbols: [je_malloc, je_free, je_realloc, arena_maybe_decay, prof_dump]
upstream: [INDEX.md, overview/business-architecture.md]
downstream: [business/domain-models/index.md, contracts/api-reference.md]
---

# 业务能力与核心流程总清单

全量归档 jemalloc 的内存生命周期闭环（`WF-01` 至 `WF-05`），建立「场景 ↔ 触发入口 ↔ 下游扇出 ↔ 数据模型」索引。业务域划分见 [业务架构全景](../overview/business-architecture.md)，实体与字段字典见 [领域模型总览](domain-models/index.md)。

## 1. 全量业务流程路由总表（Complete Workflow Inventory）

| 流程 ID | 业务域 | 流程名称 | 触发入口（`路径:行号`） | 串联模块与外部下游链 | 变更的核心实体/状态 | 专页路径 |
|---|---|---|---|---|---|---|
| `WF-01` | `DOM-01` | 分配与对齐 | `je_malloc` (`src/jemalloc.c:2323`), `je_calloc` (`:2474`), `je_mallocx` (`:3098`) | `tcache → sz/sc → arena/bin/slab / large → extent → OS VM` | `extent_state_active` | [wf-allocate.md](wf-allocate.md) |
| `WF-02` | `DOM-02` | 释放与缓存 | `je_free` (`src/jemalloc.c:2863`), `je_dallocx` (`:3487`), `je_sdallocx` (`:3594`) | `rtree → tcache → arena_slab_reg_dalloc → extent_deregister` | `active → dirty/muzzy` | [wf-deallocate.md](wf-deallocate.md) |
| `WF-03` | `DOM-03` | 缩放与原地调整 | `je_realloc` (`src/jemalloc.c:2653`), `je_rallocx` (`:3216`), `je_xallocx` (`:3390`) | `rtree → isalloc → iralloct / large_ralloc → extent` | `active → active`（可能搬迁） | [wf-realloc.md](wf-realloc.md) |
| `WF-04` | `DOM-04` | decay 回收与归还 | `arena_maybe_decay` (`src/arena.c:714`), `background_work` (`src/background_thread.c:483`) | `arena decay → extents_evict → pages_decommit → pages_unmap` | `dirty → muzzy → retained → 释放` | [wf-decay.md](wf-decay.md) |
| `WF-05` | `DOM-05` | 分配采样与统计输出 | `prof_sample_threshold_update` (`src/prof.c:1140`), `je_malloc_stats_print` (`src/jemalloc.c:3708`) | `prof → 堆转储文件` / `ctl → stats_print` | 采样计数回绕 → 转储 | [wf-profile.md](wf-profile.md) |

## 2. 业务域 ↔ 接口组 ↔ 错误码 ↔ 配置键交叉速查

| 业务域 | 入向接口组 | 调用的外部下游封装 | 专属错误/降级标识 | 核心控制配置键 |
|---|---|---|---|---|
| `DOM-01` 分配 | `malloc/calloc/aligned_alloc/posix_memalign/mallocx`（[api-reference](../contracts/api-reference.md)） | OS 内存（[os-memory](../integrations/downstream/os-memory.md)） | `ENOMEM`, `EINVAL`, `xmalloc` abort（[error-codes](../contracts/error-codes.md)） | `narenas`, `tcache`, `lg_tcache_max`, `percpu_arena`（[configuration](../contracts/configuration.md)） |
| `DOM-02` 释放 | `free/dallocx/sdallocx` | pthread TLS（[libc-runtime](../integrations/downstream/libc-runtime.md)） | `safety_check_fail` size mismatch | `junk`, `retain`, `dss` |
| `DOM-03` 缩放 | `realloc/rallocx/xallocx/sallocx` | OS 内存 | `ENOMEM`，realloc(ptr,0) 等价 free | `oversize_threshold` |
| `DOM-04` decay | mallctl `arena.<i>.decay_ms` / `background_thread`（[mallctl](../contracts/api/mallctl.md)） | OS 内存 | `[外部黑盒]` madvise 失败冒泡 | `dirty_decay_ms`, `muzzy_decay_ms`, `background_thread`, `max_background_threads` |
| `DOM-05` 诊断 | `mallctl`, `malloc_stats_print`, `jeprof` | Perl 分析器 | `<jemalloc>` 诊断串（[error-codes](../contracts/error-codes.md)） | `prof`, `lg_prof_sample`, `lg_prof_interval`, `prof_prefix`, `stats_print`, `log` |

## 3. 流程执行形态与同步边界

| 流程 ID | 执行形态 | 同步/异步边界 | 可重入风险与规避 | 主要落点 |
|---|---|---|---|---|
| `WF-01` | 同步，含无锁快路径 | tcache 命中即返回，未命中进 arena 锁 | slow path 重入强制 `tcache=NONE, arena=0`（`src/jemalloc.c:2097-2108`） | active extent / slab reg |
| `WF-02` | 同步，含无锁快路径 | `free_fastpath` 命中直接归还 tcache | rtree 反查失败即 safety check（`src/jemalloc.c:2570-2572`） | tcache bin / dirty extent |
| `WF-03` | 同步 | 原地扩张失败才搬迁，失败保留原块 | 采样路径在 `src/jemalloc.c:2529-2552` 处理回滚 | active extent |
| `WF-04` | 后台线程 + 惰性触发 | `background_work`（`src/background_thread.c:483`）与分配路径互斥靠 arena 锁 | `dirty_decay_ms == -1` 时后台不推进该域 | retained extent / OS |
| `WF-05` | 同步采样 + 批量转储 | 采样在 fastpath 计数，转储走独立文件 IO | 转储失败由 `prof_dump_open_impl`（`src/prof.c:1228`）写诊断串 | 堆转储文件 / 统计输出 |

## 4. 共变耦合与修改清单（Co-Change Playbook）

1. 新增流程：在本表登记 `WF-*` 并建专页 → 更新 [业务架构全景](../overview/business-architecture.md) 的域矩阵与不变量 → 更新 [调用链全景](../overview/callchain-panorama.md)。
2. 改任一流程规则：按 `WF-*` 进入专页的「精确规则与阈值」表 → 反查 `src/jemalloc.c` / `src/arena.c` 对应行 → 跑 `make check`。
3. 改接口签名：同步 [api-reference](../contracts/api-reference.md) 的 20 项清单与 [mallctl](../contracts/api/mallctl.md) 节点表 → 跑 `test/unit/mallctl.c` 与 `test/integration/malloc.c`。
4. 改状态实体：同步 [领域模型总览](domain-models/index.md) 及 arena/extent/tcache 三页 → 跑 `test/unit/extent_util.c`。
