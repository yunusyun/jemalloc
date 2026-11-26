---
kind: component
scope: tcache-and-tsd
status: draft
origin: generated
source_paths: [src/tcache.c, src/tsd.c, src/rtree.c, include/jemalloc/internal/cache_bin.h]
symbols: [tcache_alloc_small_hard, tcache_bin_flush_small, tcache_create_explicit, tsd_fetch_slow, rtree_leaf_elm_lookup_hard, cache_bin_alloc_easy]
upstream: [architecture/allocator-core.md]
downstream: [business/domain-models/tcache.md, business/wf-deallocate.md]
---

# 线程缓存与 TSD（tcache-and-tsd）

## 1. 角色与边界

本组件提供每线程缓存（tcache）、线程状态描述符（TSD）与全局地址索引（rtree）：小对象先走 `cache_bin` 无锁弹栈，miss 时再进入 arena；空闲回补与刷新由 GC 低水位驱动。上游分配入口见 [architecture/allocator-core.md](allocator-core.md)，数据模型见 [business/domain-models/tcache.md](../business/domain-models/tcache.md)，释放业务流见 [business/wf-deallocate.md](../business/wf-deallocate.md)。

## 2. 符号级调用树

```text
分配（tcache 命中优先）：
  je_malloc fastpath -> cache_bin_alloc_easy include/jemalloc/internal/cache_bin.h:88
    ├─ ncached > low_water -> 返回栈顶指针，success=true
    └─ ncached 不大于 low_water -> success=false -> malloc_default -> imalloc_body
         -> tcache_alloc_small_hard src/tcache.c:89
              ├─ arena_tcache_fill_small src/arena.c:1381
              └─ cache_bin_alloc_easy 二次尝试

释放与 GC：
  je_free -> cache_bin 压栈 -> low_water 为负时触发 tcache_bin_gc
    -> tcache_bin_flush_small src/tcache.c:134（binind 小于 SC_NBINS）
       ├─ arena_dalloc_bin_junked_locked -> 归还 slab reg
       └─ [stats] tcache_stats_merge src/tcache.c:601
    -> tcache_bin_flush_large src/tcache.c:68（binind >= SC_NBINS）

索引查询：
  rtree_leaf_elm_lookup_hard src/rtree.c:223
    └─ [cache miss] 逐级 RTREE_GET_CHILD -> rtree_node_alloc_impl src/rtree.c:31
```

`cache_bin_alloc_easy` 在所列 `src/*.c` 中 `[仓库内未找到字面实现]`：它是 `JEMALLOC_ALWAYS_INLINE` 内联函数，逐字定义于 `include/jemalloc/internal/cache_bin.h:88`；`src/jemalloc.c:2371` 与 `src/tcache.c:99` 均为其调用点。

## 3. 源码锚点与职责契约表

| 文件:行号 | 符号 | 职责 | 入 → 出 |
|---|---|---|---|
| `src/tcache.c:89` | `tcache_alloc_small_hard` | bin 空时批量填充再取 | `arena_t*`,`cache_bin_t*` → `void*` |
| `src/tcache.c:134` | `tcache_bin_flush_small` | 刷新小 bin 的 `rem` 个对象 | `tcache_t*`,`binind`,`rem` → `void` |
| `src/tcache.c:486` | `tcache_create_explicit` | mallctl `tcache.create` 创建显式缓存 | `tsd_t*` → `tcache_t*` |
| `src/tsd.c:270` | `tsd_fetch_slow` | TSD 慢路径状态迁移 | `tsd_t*`,`minimal` → `tsd_t*` |
| `src/rtree.c:223` | `rtree_leaf_elm_lookup_hard` | rtree 硬查找（可建节点） | `key`,`init_missing` → `rtree_leaf_elm_t*` |
| `include/jemalloc/internal/cache_bin.h:88` | `cache_bin_alloc_easy` | 单分支弹栈 + 低水位更新 | `cache_bin_t*` → `void*` + `success` |

`tcache_bin_flush_small` 定义位于 `src/tcache.c:134`（上下文 surveying 中的 `:55` 是 `tcache_bin_gc` 内的调用点，已据实修正为定义行）。

## 4. TSD 状态机与 rtree 索引

`tsd_fetch_slow`（`src/tsd.c:270`）按 `tsd_state_*` 分支：`nominal_slow` 直接返回；`nominal_recompute` 调 `tsd_slow_update`；`uninitialized` 且 `!minimal` 时置 `nominal` 并 `tsd_data_init`；`minimal_initialized` 升级为 fully initialized。快路径 `tsd_fast(tsd)` 与 `tsd_get(false)` 在 `src/jemalloc.c:2330-2331` 直接消费该状态。

`rtree_leaf_elm_lookup_hard`（`src/rtree.c:223`）在 `RTREE_HEIGHT > 1` 时从 `rtree->root` 逐层下降，`init_missing` 为真时经 `rtree_node_alloc_impl`（`src/rtree.c:31`）补节点；查找前 `config_debug` 分支校验 L1/L2 cache 不含同 leafkey（`src/rtree.c:233-240`）。rtree 写入侧由 extent 登记驱动，见 [architecture/extent-and-os.md](extent-and-os.md)。

## 5. 刷新、销毁与显式缓存

`tcache_bin_flush_small` 在 `src/tcache.c:138-141` 断言 `binind` 小于 `SC_NBINS` 且 `rem` 不大于 `ncached`，再按 `tcache->arena` 归还。`tcache_bin_gc` 用低水位自适应：`low_water` 为正时刷新 `ncached - low_water + (low_water >> 2)` 个对象并使 `lg_fill_div` 加倍（`src/tcache.c:50-66`）；`low_water` 为负时减半 fill count（`src/tcache.c:71-78`）。`tcache_create_explicit` 从 arena 0 用 `ipallocztm` 申请对齐块（`src/tcache.c:498-499`），再 `tcache_arena_associate` 绑定 arena（`src/tcache.c:506`）。

## 6. 共变耦合与修改清单（Co-Change Playbook）

- 改动 `cache_bin_t` 字段布局 → 同步 `include/jemalloc/internal/cache_bin.h:88`、`src/tcache.c:89`/`134`、`src/jemalloc.c:2371` 及 [business/domain-models/tcache.md](../business/domain-models/tcache.md) 的字段表。
- 改动 `lg_fill_div` 自适应策略 → 同步 `src/tcache.c:59-77` 与 `tcache_stats_merge`（`src/tcache.c:601`；`:382` 为调用点）。
- 改动 TSD 状态枚举 → 同步 `src/tsd.c:270` 全部分支、`src/jemalloc.c:2330-2331` 快路径判定与 [business/wf-deallocate.md](../business/wf-deallocate.md) 的清理时序。
- 改动 rtree 高度或缓存层数 → 同步 `src/rtree.c:223` 的 `RTREE_GET_CHILD` 宏与 `src/rtree.c:233-240` 断言。
- 验证命令：`make tests && make check`；重点单测 `test/unit/tcache.c`、`test/unit/rtree.c`、`test/unit/tsd.c`。
- 隐式陷阱：`tcache_create_explicit` 依赖 arena 0 已完成初始化（`src/tcache.c:499`），在 `malloc_init` 之前调用会触发重入断言。
