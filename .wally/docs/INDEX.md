# jemalloc · Agent 知识库总路由表（INDEX）

> **Agent 读取规程（必读）**：
> 1. **禁止一次性全量读取 `.wally/docs/`**。先查下方「任务意图路由表」或「源码路径反向索引」，只打开命中的 1–3 篇目标文档（先 `Read(limit=15)` 看顶部 YAML，确认 `source_paths`、`symbols` 与 `status`）。
> 2. **草稿核验纪律**：所有页面均为 `status: draft`、`origin: generated`，是从源码提取的草稿；从文档拿到 `路径:行`（`符号名`）后，动手改代码前必须 `Read(path, offset, limit)` 回读源码字面实现，禁止把近邻词线索当结论。
> 3. **人工精修保护**：后续若出现 `origin: human` 或 `status: verified` 页面，重跑知识库严禁覆盖。基准快照：`chore/wally-init-20261126-c74` @ `6d281cef3cd747666b00c4557c0f42786f66ae89`。
> 4. **外部权威页**：`apis/`、`sqls/`、`unittest/`、`integration_test/`、`devops/` 为 SOP 权威文档，仅在此链接，不复制正文、不按 codewiki 模板改写、不计入 `.wiki-meta.json` 的 `pageOrder`。

jemalloc 是进程内嵌的通用 `malloc(3)` 实现，强调低碎片与可扩展并发；无 HTTP/RPC/DB 服务，入向契约是 20 个公共 C 符号、mallctl 命名空间与 3 个 CLI 工具，出向边界是 OS 内存管理与 pthread/libc 运行时。系统分层见 [系统技术架构](overview/system-architecture.md)，业务闭环见 [业务架构全景](overview/business-architecture.md)。

---

## 1. 任务意图最短路由表（Agent Intent Router）

| Agent 当前待办任务类型 | 按序必读文档（1 → 2 → 3） |
|---|---|
| **新增/修改分配、对齐或 flags 逻辑** | [入向接口契约](contracts/api-reference.md) → [分配器核心](architecture/allocator-core.md) → [分配流程](business/wf-allocate.md) |
| **修改释放 / tcache 回填 / 线程缓存阈值** | [释放流程](business/wf-deallocate.md) → [tcache 与 tsd](architecture/tcache-and-tsd.md) → [tcache 实体](business/domain-models/tcache.md) |
| **处理 realloc / 缩放 / 原地扩张异常** | [缩放流程](business/wf-realloc.md) → [extent 实体](business/domain-models/extent.md) → [错误码字典](contracts/error-codes.md) |
| **调整 decay / muzzy / 后台回收参数** | [decay 回收流程](business/wf-decay.md) → [配置手册](contracts/configuration.md) → [profiler 与后台线程](architecture/profiler-and-background.md) |
| **对接或修改 OS 内存参数（mmap/brk/THP）** | [OS 内存下游契约](integrations/downstream/os-memory.md) → [参数组装映射](integrations/topology-and-mapping.md) → [extent 与 OS 底座](architecture/extent-and-os.md) |
| **定位报错 / abort / errno / safety check** | [错误码字典](contracts/error-codes.md) → [主干调用链全景](overview/callchain-panorama.md) → [排障 Runbook](operations/runbook-and-observability.md) |
| **新增/修改 mallctl 节点或统计输出** | [mallctl 命名空间](contracts/api/mallctl.md) → [控制面与可观测](architecture/control-and-observability.md) → [采样与统计流程](business/wf-profile.md) |
| **跨模块重构 / 评估联动与跑单测** | [模块依赖矩阵](architecture/module-dep-matrix.md) → [高危雷区与历史陷阱](operations/pitfalls-and-adr.md) → [构建与单测映射](operations/dev-and-test.md) |
| **理解术语 / 把需求话术翻译成符号** | [领域术语映射](overview/glossary.md) → [领域模型总览](business/domain-models/index.md) → 对应 WF 专页 |

---

## 2. 源码路径 → 文档反向索引（Source-to-Doc Reverse Index）

> 直接指定修改某源码目录或文件时，由此表 O(1) 反查知识文档：

| 仓库源码路径（前缀或关键文件） | 架构模型 / 角色 | 首选关联文档 | 关联上下游/业务文档 |
|---|---|---|---|
| `src/jemalloc.c` | `Core Engine` / API 入口 | [allocator-core](architecture/allocator-core.md) | [api-reference](contracts/api-reference.md) · [wf-allocate](business/wf-allocate.md) |
| `src/arena.c`, `src/bin.c`, `src/large.c` | `Core Engine` | [allocator-core](architecture/allocator-core.md) | [arena 实体](business/domain-models/arena.md) |
| `src/sc.c`, `src/sz.c`, `include/jemalloc/internal/sc.h` | 尺寸类查表 | [allocator-core](architecture/allocator-core.md) | [glossary](overview/glossary.md) |
| `src/extent.c`, `src/extent_mmap.c`, `src/extent_dss.c`, `src/pages.c` | `Data & Adapter` | [extent-and-os](architecture/extent-and-os.md) | [OS 内存契约](integrations/downstream/os-memory.md) |
| `src/base.c` | 元数据 bump 分配 | [extent-and-os](architecture/extent-and-os.md) | [base 元数据归属](business/domain-models/index.md) |
| `src/tcache.c`, `src/tsd.c`, `src/rtree.c` | `Core Engine` / TLS | [tcache-and-tsd](architecture/tcache-and-tsd.md) | [wf-deallocate](business/wf-deallocate.md) |
| `src/ctl.c`, `src/stats.c`, `src/log.c`, `src/malloc_io.c` | `API Service` / 可观测 | [control-and-observability](architecture/control-and-observability.md) | [mallctl](contracts/api/mallctl.md) |
| `src/prof.c`, `src/background_thread.c`, `src/hook.c` | `Async/Cron Worker` | [profiler-and-background](architecture/profiler-and-background.md) | [wf-profile](business/wf-profile.md) · [wf-decay](business/wf-decay.md) |
| `src/mutex.c`, `src/witness.c`, `src/bitmap.c`, `src/ckh.c`, `src/nstime.c`, `src/safety_check.c` | 底座工具 | [sync-and-utilities](architecture/sync-and-utilities.md) | [pitfalls-and-adr](operations/pitfalls-and-adr.md) |
| `src/zone.c` | Darwin 历史适配（非 Linux 主链） | [sync-and-utilities](architecture/sync-and-utilities.md) | [pitfalls-and-adr](operations/pitfalls-and-adr.md) |
| `include/jemalloc/jemalloc_protos.h.in`, `jemalloc_macros.h.in` | 公共契约 | [api-reference](contracts/api-reference.md) | [configuration](contracts/configuration.md) |
| `include/jemalloc/internal/*_structs.h`, `*_types.h` | 领域模型 | [domain-models/index](business/domain-models/index.md) | [arena](business/domain-models/arena.md) · [extent](business/domain-models/extent.md) · [tcache](business/domain-models/tcache.md) |
| `bin/jemalloc-config.in`, `bin/jemalloc.sh.in`, `bin/jeprof.in` | `UI & CLI` | [cli-tools](architecture/cli-tools.md) | [deployment-and-runtime](operations/deployment-and-runtime.md) |
| `Makefile.in`, `autogen.sh`, `test/*`, `run_tests.sh` | 构建与验证 | [dev-and-test](operations/dev-and-test.md) | [boundary](operations/boundary.md) |
| `.wally/docs/apis/`, `.wally/docs/sqls/`, `.wally/docs/unittest/`, `.wally/docs/integration_test/`, `.wally/docs/devops/` | SOP 外部权威页（不作为代码入口） | [apis/index](apis/index.md) · [unittest](unittest/unittest.md) · [integration_test](integration_test/integration_test.md) · [devops/env](devops/env.md) · [sqls](sqls/README.md) | 仅链接，不改写 |

---

## 3. 模块技术能力清单（Component Registry）

| 组件标识 | 框架模型 | 核心源码路径 | 核心入口符号 | 上游调用方 | 下游依赖 | 文档 |
|---|---|---|---|---|---|---|
| `allocator-core` | `Core Engine` | `src/jemalloc.c`, `src/arena.c`, `src/bin.c`, `src/large.c`, `src/sz.c`, `src/sc.c` | `je_malloc` (`src/jemalloc.c:2323`), `arena_new` (`src/arena.c:1942`) | 宿主程序 | extent, tcache | [查看](architecture/allocator-core.md) |
| `extent-and-os` | `Data & Adapter` | `src/extent.c`, `src/pages.c`, `src/base.c` | `extents_alloc` (`src/extent.c:537`), `pages_map` (`src/pages.c:179`) | allocator-core | OS VM | [查看](architecture/extent-and-os.md) |
| `tcache-and-tsd` | `Core Engine` | `src/tcache.c`, `src/tsd.c`, `src/rtree.c` | `tcache_create_explicit` (`src/tcache.c:486`) | allocator-core | arena / rtree | [查看](architecture/tcache-and-tsd.md) |
| `control-and-obs` | `API Service` | `src/ctl.c`, `src/stats.c`, `src/log.c`, `src/malloc_io.c` | `ctl_init` (`src/ctl.c:1118`) | mallctl 调用方 | arena 统计 | [查看](architecture/control-and-observability.md) |
| `profiler-and-bg` | `Async/Cron Worker` | `src/prof.c`, `src/background_thread.c`, `src/hook.c` | `background_work` (`src/background_thread.c:483`) | 采样/定时器 | extent / 磁盘 | [查看](architecture/profiler-and-background.md) |
| `sync-and-utils` | `Core Engine`（底座） | `src/mutex.c`, `src/bitmap.c`, `src/ckh.c`, `src/nstime.c` | `malloc_mutex_init` (`src/mutex.c:135`) | 全仓 | 无 | [查看](architecture/sync-and-utilities.md) |
| `cli-tools` | `UI & CLI` | `bin/*.in` | `usage` (`bin/jemalloc-config.in:3`) | 用户终端 / CI | mallctl / 转储文件 | [查看](architecture/cli-tools.md) |

---

## 4. 业务流程、契约与运维速查

- **全景认知**：[系统技术架构](overview/system-architecture.md) · [业务架构全景](overview/business-architecture.md) · [主干调用链全景](overview/callchain-panorama.md) · [领域术语映射](overview/glossary.md)
- **技术架构**：[组件总表](architecture/index.md) · [模块依赖矩阵](architecture/module-dep-matrix.md) · [同步与工具](architecture/sync-and-utilities.md) · [CLI 工具](architecture/cli-tools.md)
- **上下游与契约**：[集成总览](integrations/index.md) · [参数组装映射](integrations/topology-and-mapping.md) · [OS 内存契约](integrations/downstream/os-memory.md) · [libc 运行时契约](integrations/downstream/libc-runtime.md) · [入向接口](contracts/api-reference.md) · [mallctl](contracts/api/mallctl.md) · [错误码](contracts/error-codes.md) · [配置手册](contracts/configuration.md)
- **业务与数据**：[业务流程总表](business/index.md) · [分配](business/wf-allocate.md) · [释放](business/wf-deallocate.md) · [缩放](business/wf-realloc.md) · [decay](business/wf-decay.md) · [采样统计](business/wf-profile.md) · [领域模型总览](business/domain-models/index.md) · [arena](business/domain-models/arena.md) · [extent](business/domain-models/extent.md) · [tcache](business/domain-models/tcache.md)
- **验证、雷区与边界**：[构建与单测](operations/dev-and-test.md) · [部署与运行时](operations/deployment-and-runtime.md) · [排障 Runbook](operations/runbook-and-observability.md) · [高危雷区与历史陷阱](operations/pitfalls-and-adr.md) · [认知边界声明](operations/boundary.md)
- **SOP 外部权威页**：[API 索引](apis/index.md) · [SQL](sqls/README.md) · [单测规范](unittest/unittest.md) · [集成测试](integration_test/integration_test.md) · [开发环境](devops/env.md) · [测试环境部署](devops/test-env-deploy.md)
