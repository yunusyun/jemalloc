---
kind: panorama
scope: callchain-panorama
status: draft
origin: generated
source_paths: [src/jemalloc.c, src/arena.c, src/tcache.c, src/extent.c, src/background_thread.c, src/ctl.c]
symbols: [je_malloc, je_free, je_realloc, arena_maybe_decay, je_mallctl, background_work]
upstream: [INDEX.md]
downstream: [integrations/topology-and-mapping.md, contracts/error-codes.md]
---

# 主干调用链全景

提炼贯穿全仓的 4 条主干链路：同步分配、同步释放、后台 decay、控制面读取。静态搜索在线程、fork 回调与 mallctl 动态分发处会断链，末尾给出显式桥接表。字段透传细节见 [参数组装映射](../integrations/topology-and-mapping.md)，错误短路见 [错误码字典](../contracts/error-codes.md)。

## 1. 主干链路索引

| 链路 ID | 链路名称 | 模式 | 起点符号 | 终点落点 | 对应业务页 |
|---|---|---|---|---|---|
| `CC-01` | 分配主链 | 同步（含快路径） | `je_malloc` (`src/jemalloc.c:2323`) | active extent / slab reg | [wf-allocate](../business/wf-allocate.md) |
| `CC-02` | 释放主链 | 同步（含快路径） | `je_free` (`src/jemalloc.c:2863`) | tcache bin / dirty extent | [wf-deallocate](../business/wf-deallocate.md) |
| `CC-03` | decay 回收链 | 后台线程 + 惰性触发 | `arena_maybe_decay` (`src/arena.c:714`) | retained extent / OS 归还 | [wf-decay](../business/wf-decay.md) |
| `CC-04` | 控制面读链 | 同步 | `je_mallctl` (`src/jemalloc.c:3645`) | arena/stats 聚合值 | [mallctl 命名空间](../contracts/api/mallctl.md) |

## 2. 符号级条件调用树

### `CC-01`：分配主链

```text
je_malloc(size) -> src/jemalloc.c:2323
  ├─ [guard: 未初始化 / !tsd_fast / size > SC_LOOKUP_MAXCLASS] malloc_default (:2271)
  ├─ [guard: gc_ticker 命中] malloc_default (:2337)
  ├─ [guard: bytes_until_sample < 0] malloc_default (:2355)
  ├─ [fast] cache_bin_alloc_easy (:2371) -> 命中即返回
  └─ [slow] imalloc (:2246) -> imalloc_body (:2028)
       ├─ [guard: 乘法溢出] label_oom -> ENOMEM (:2049, :2190)
       ├─ [guard: 对齐非 2 次幂] label_invalid_alignment -> EINVAL (:2054, :2211)
       ├─ [guard: 重入] tcache=NONE, arena=0 (:2097-2108)
       ├─ imalloc_no_sample (:1912) -> [outbound] iallocztm (:1949)
       │    ├─ small -> arena_slab_reg_alloc (src/arena.c:271)
       │    └─ large -> large_malloc (src/large.c:14)
       └─ [outbound] extents_alloc (src/extent.c:537) -> pages_map (src/pages.c:179)
```

### `CC-02`：释放主链

```text
je_free(ptr) -> src/jemalloc.c:2863
  ├─ [fast] free_fastpath (:2806) -> 命中小对象直接归还 tcache
  └─ [slow] free_default (:2771) -> ifree (:2556)
       ├─ [rtree] rtree_szind_slab_read (:2570) -> alloc_ctx{szind, slab}
       ├─ [prof] prof_free (:2577) 采样回溯
       ├─ [stats] thread_deallocated += usize (:2582)
       └─ idalloctm (:2586) -> tcache_bin_flush_small (src/tcache.c:134)
            -> arena_slab_reg_dalloc (src/arena.c:1695)
            -> extent_deregister (src/extent.c:878)
```

### `CC-03`：decay 回收链

```text
[惰性] arena_maybe_decay (src/arena.c:714) 或 [后台] background_work (src/background_thread.c:483)
  └─ arena_decay_to_limit (src/arena.c:916)
       ├─ arena_decay_epoch_advance (src/arena.c:646) -> 计算 purge 截止期
       ├─ extents_evict (src/extent.c:567) -> 按 first-fit 选取 extent
       ├─ extent_purge_forced_impl (src/extent.c:2035) -> pages_decommit (src/pages.c:287)
       └─ extent_deregister (src/extent.c:878) -> pages_unmap (src/pages.c:241)
```

### `CC-04`：控制面读链

```text
je_mallctl(name, oldp, ...) -> src/jemalloc.c:3645
  ├─ [guard] malloc_init() 失败 -> EAGAIN (:3654)
  └─ ctl_byname (src/ctl.c:1292) -> ctl_lookup (:1194)
       ├─ [epoch] ctl_refresh (:1044) -> ctl_arena_refresh (:1065)
       ├─ [arena] arena_stats_merge (src/arena.c:83)
       └─ [stats] stats_print (src/stats.c:1396) <- je_malloc_stats_print (src/jemalloc.c:3708)
```

## 3. 跨线程 / 队列 / 子进程 / 协议桥接表

| 投递方符号 | 桥接机制 | 实际接收执行入口 | 载荷结构 |
|---|---|---|---|
| `jemalloc_constructor` (`src/jemalloc.c:3774`) | ELF `JEMALLOC_ATTR(constructor)` | `malloc_init` (`src/jemalloc.c:222`) | 无参，提前完成初始化 |
| `pthread_atfork` (`src/jemalloc.c:1584-1586`) | libc fork 回调注册 | `jemalloc_prefork` (`:3781`) / `jemalloc_postfork_parent` (`:3855`) / `jemalloc_postfork_child` (`:3894`) | 无参，逐 arena 锁状态 |
| `background_thread_create` (`src/background_thread.c:593`) | `pthread_create` | `background_thread_entry` (`:506`) → `background_work` (`:483`) | `background_thread_info_t *` |
| `tcache_bin_flush_small` (`src/tcache.c:134`) | 缓存满阈值触发 | `arena_slab_reg_dalloc` (`src/arena.c:1695`) | `extent_t *`, `szind_t`, reg 数组 |
| `je_mallctl` (`src/jemalloc.c:3645`) | 字符串名 → `ctl_named_node_t` 动态分发 (`src/ctl.c:622-636`) | `ctl_byname` (`src/ctl.c:1292`) 或 `ctl_bymib` (`:1338`) | `name` / `mib[]` + `oldp/newp` |
| `ticker_trytick` (`src/jemalloc.c:2337`) | tcache gc ticker 计数回绕 | `malloc_default` (`src/jemalloc.c:2271`) → `tcache_flush_cache` (`src/tcache.c:512`) | `tcache_t *` |

## 4. 共变耦合与修改清单（Co-Change Playbook）

1. 改任一主干的 guard 条件：按链路 ID 定位对应 `src/jemalloc.c` / `src/arena.c` 行 → 同步对应 WF 专页的「精确规则与阈值」表 → 跑 `make check`。
2. 改 tcache→arena 回收触发点：改 `src/tcache.c:134` → 核对 `src/arena.c:1695` 的 reg 归还与 [tcache 实体页](../business/domain-models/tcache.md) → 跑 `test/unit/binshard.c`。
3. 改后台线程调度：改 `src/background_thread.c:483/593` → 同步 [wf-decay](../business/wf-decay.md) 与 `dirty_decay_ms` 配置说明 → 跑 `test/unit/background_thread.c`。
4. 新增 mallctl 节点：改 `src/ctl.c:622-636` 节点表 → 更新 [mallctl 命名空间](../contracts/api/mallctl.md) → 跑 `test/unit/mallctl.c`。
