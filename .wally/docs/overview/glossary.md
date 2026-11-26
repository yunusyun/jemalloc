---
kind: panorama
scope: glossary
status: draft
origin: generated
source_paths: [include/jemalloc/internal/jemalloc_internal_types.h, include/jemalloc/internal/sc.h, include/jemalloc/internal/extent_structs.h, include/jemalloc/internal/arena_types.h, include/jemalloc/jemalloc_macros.h.in]
symbols: [szind_t, SC_NBINS, SC_SMALL_MAXCLASS, extent_state_t, arena_t, tcache_t, MALLOCX_LG_ALIGN_MASK]
upstream: [INDEX.md]
downstream: [business/domain-models/index.md, architecture/allocator-core.md]
---

# 领域术语与代码符号精确映射表

消除「分配一块内存」这类需求话术与 jemalloc 内部符号之间的命名鸿沟。全部定义取自仓库内头文件字面，术语落地可继续查 [领域模型总览](../business/domain-models/index.md) 与 [分配器核心](../architecture/allocator-core.md)。

## 1. 自然语言术语 ↔ 源码符号映射字典

| 需求/业务常见叫法 | 精确技术与边界定义 | 源码对应符号 | 权威定义位置 |
|---|---|---|---|
| 对象「大小档位 / 规格」 | 请求字节数经查表归并后的整数索引，不是原始字节数 | `szind_t` | `include/jemalloc/internal/jemalloc_internal_types.h:10` |
| 「小对象」 | `ind < SC_NBINS` 且 `usize <= SC_SMALL_MAXCLASS`，从 bin/slab 的 run 中取 reg | `SC_NBINS`, `SC_SMALL_MAXCLASS` | `include/jemalloc/internal/sc.h:229`, `include/jemalloc/internal/sc.h:253` |
| 「大对象 / 大块」 | 超过 small 上限、独占一个或多个 extent 的分配，不再走 slab reg | `SC_LARGE_MAXCLASS` | `include/jemalloc/internal/sc.h:265` |
| 快路径上限 | 查表索引覆盖的最大请求尺寸；超过即尾调用 `malloc_default` | `SC_LOOKUP_MAXCLASS` | `include/jemalloc/internal/sc.h:246` |
| 「页 / extent」 | 一段连续页；内存管理的基本供给单位，带状态机 | `extent_t` / `struct extent_s` | `include/jemalloc/internal/extent_structs.h:20` |
| 「脏页 / muzzy 页 / 保留页」 | extent 的三种非 active 状态，分别对应 decay 的不同回收阶段 | `extent_state_active/dirty/muzzy/retained` | `include/jemalloc/internal/extent_structs.h:12-17` |
| 「arena / 分区」 | 持有 bin 数组、extent 集合、decay 状态与锁的并发隔离域 | `arena_t` | `include/jemalloc/internal/arena_types.h:18` |
| 「线程缓存」 | 每线程无锁缓存，包含 small/large bin 与 gc ticker | `tcache_t` | `include/jemalloc/internal/tcache_types.h:6` |
| 「对齐请求」 | flags 低 6 位编码 `lg(alignment)`；`0` 表示无对齐约束 | `MALLOCX_LG_ALIGN_MASK` | `include/jemalloc/internal/jemalloc_internal_types.h:28`, `:38` |
| 「全部 arena」 | mallctl 索引通配值，用于批量读取所有 arena 统计 | `MALLCTL_ARENAS_ALL` | `include/jemalloc/jemalloc_macros.h.in:47` |
| 「关闭线程缓存」 | `MALLOCX_TCACHE_NONE` 使单次分配绕过 tcache | `MALLOCX_TCACHE_NONE` | `include/jemalloc/jemalloc_macros.h.in:28-29` |

## 2. 易混概念严格辨析

| 概念 A | 概念 B | 关键差异（粒度 / 时机 / 混用后果） | 对照源码锚点 |
|---|---|---|---|
| `szind_t`（档位索引） | `usize`（该档位的实际字节数） | 前者是 `sc_data_t` 生成的离散下标，后者是 `sz_index2size(ind)` 的结果；把二者混用会让对齐与统计全部错位 | `include/jemalloc/internal/jemalloc_internal_types.h:10` vs `src/jemalloc.c:2067` |
| `SC_LOOKUP_MAXCLASS` | `SC_SMALL_MAXCLASS` | 前者是 tcache 查表快路径边界（`src/jemalloc.c:2331`），后者是 bin/slab 适用的尺寸上限；快路径边界严格小于 small 上限 | `include/jemalloc/internal/sc.h:246` vs `include/jemalloc/internal/sc.h:253` |
| `extent_state_dirty` | `extent_state_muzzy` | dirty 仍需 purge（内容可能含用户数据）；muzzydecay 已 purge 但未 decommit；误判会跳过回收或重复释放 | `include/jemalloc/internal/extent_structs.h:14-15` |
| `arena_t`（外部 arena） | `arena 0` / `iarena` | arena 0 是 bootstrap 期必然就绪的已知 arena，重入路径强制退回它（`src/jemalloc.c:2105-2107`）；普通 arena 可能尚未创建 | `include/jemalloc/internal/arena_types.h:18` vs `src/jemalloc.c:2105` |
| `base_alloc`（元数据分配器） | `imalloc`（用户对象分配器） | 前者用 bump allocation 分配 arena/extent 元数据，不经过 tcache；后者服务用户请求；改错对象会引发初始化递归 | `src/base.c:464` vs `src/jemalloc.c:2246` |
| `MALLOCX_ARENA(ind)` | `MALLCTL_ARENAS_ALL` | 前者锁定单个 arena 索引，后者是 mallctl 批量通配；把通配当索引会越界 | `include/jemalloc/internal/jemalloc_internal_types.h:26` vs `include/jemalloc/jemalloc_macros.h.in:47` |

## 3. 代码命名与枚举惯例

- **命名空间宏**：所有公共符号声明写 `@je_@` 前缀，安装时由 `jemalloc_mangle@install_suffix@.h` 决定真实前缀（`include/jemalloc/jemalloc_protos.h.in:2-5`）；本文统一保留 `je_` 写法。
- **状态字面量**：`extent_state_t` 仅 4 个枚举值且 `active == 0`（`include/jemalloc/internal/extent_structs.h:12-17`），任何新增状态必须同步更新 rtree 写入与 decay 判定。
- **档位边界常量**：全部集中在 `include/jemalloc/internal/sc.h:198-265`；`SC_NSIZES` 由 tiny/pseudo/regular 相加得到（`:198`），`SC_NBINS` 由宏展开计算（`:229`），禁止手改派生值。
- **flags 位宽**：arena 12 位、tcache 12 位、align 6 位（`include/jemalloc/internal/jemalloc_internal_types.h:26-28`），`MALLOCX_ARENA_LIMIT = 4095`（`:34`）、`MALLOCX_TCACHE_MAX = 4093`（`:37`）。

## 4. 共变耦合与修改清单（Co-Change Playbook）

1. 新增档位或改边界常量：改 `include/jemalloc/internal/sc.h:198-265` → 重新跑 `src/sc.c:312` `sc_data_init` 与 `src/sz.c:60` `sz_boot` → 同步 `src/bin.c:12` `bin_infos_init` → 更新 [领域模型总览](./business/domain-models/index.md) 与 [分配器核心](./architecture/allocator-core.md) 的档位表 → 跑 `make check`。
2. 新增 `extent_state_t` 枚举值：改 `include/jemalloc/internal/extent_structs.h:12-17` → 检查 `src/extent.c:721` 的 rtree 状态写入与 `src/arena.c:714` 的 decay 判定 → 更新 [extent 实体页](./business/domain-models/extent.md)。
3. 改 flags 位宽：改 `include/jemalloc/internal/jemalloc_internal_types.h:26-38` → 同步 `include/jemalloc/jemalloc_macros.h.in` 的公共宏 → 核对 `src/jemalloc.c:3115-3135` 的解码分支 → 跑 `make tests && make check`。
