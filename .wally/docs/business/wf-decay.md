---
kind: workflow
scope: WF-decay
status: draft
origin: generated
source_paths: [src/arena.c, src/background_thread.c, src/extent.c]
symbols: [arena_maybe_decay, arena_decay_to_limit, arena_decay_ms_set, background_work, extents_evict, extent_purge_forced_impl]
upstream: [business/index.md, contracts/configuration.md]
downstream: [business/domain-models/arena.md, architecture/profiler-and-background.md]
---

# 衰减业务流（WF-decay）

本页描述 dirty／muzzy 页如何按时间阈值被限流清除，以及事件驱动与后台线程两条触发通道。配置项语义见[配置契约](../contracts/configuration.md)，状态载体见[arena 领域模型](domain-models/arena.md)与[分析器与后台架构](../architecture/profiler-and-background.md)。

## 1. 触发通道

三条触发路径最终汇入 `arena_decay`：`src/arena.c:997` 先处理 `arena_decay_dirty`（`998`），成功后才处理 `arena_decay_muzzy`（`1001`），保证状态转换严格 dirty → muzzy → retained。

| 通道 | 字面锚点 | 触发方式 | 锁策略 |
| --- | --- | --- | --- |
| 分配／释放事件 | `src/arena.c:714` `arena_maybe_decay` | decay ticker 到期，`arena_decay_impl`（`948`）调 `964` | `trylock`（`959`）失败即返回 `true` 让调用方稍后重试 |
| 后台线程 | `src/background_thread.c:483` `background_work(tsd, ind)` | `ind==0` 走 `489-490` 的 `background_thread0_work`；其余走 `492-499` 循环 | 持 `info->mtx`（`486`），`background_work_sleep_once`（`497`） |
| 配置变更 | `src/arena.c:787` `arena_decay_ms_set` | mallctl 写入 `decay_ms` | 先 `malloc_mutex_lock`（`793`）再 `arena_decay_reinit`（`802`） |

`src/background_thread.c:506` 的 `background_thread_entry` 在 `510` 设置线程名 `jemalloc_bg_thd`，`514-516` 在 `percpu_arena` 模式下绑定 affinity，`522` 用 `tsd_internal_fetch()` 避免后台工作触发新 arena 创建。

## 2. 符号级调用树

```text
[事件] arena_decay(tsdn, arena, is_bg, all) -> src/arena.c:997 arena_decay
  ├─ src/arena.c:998 arena_decay_dirty(...) -> src/arena.c:948 arena_decay_impl
  │    └─ src/arena.c:959 malloc_mutex_trylock(&decay->mtx)
  │         └─ src/arena.c:714 arena_maybe_decay(tsdn, arena, decay, extents, is_bg)
  └─ src/arena.c:1001 arena_decay_muzzy(...) -> 同构流程

arena_maybe_decay -> src/arena.c:714
  ├─ src/arena.c:719 arena_decay_ms_read -> decay_ms
  ├─ [decay_ms == 0] src/arena.c:722 arena_decay_to_limit(..., 0, extents_npages_get, ...)
  ├─ [时钟回退] src/arena.c:732-744 重置 epoch 并重建 deadline
  ├─ [deadline 到达] src/arena.c:757-760 arena_decay_epoch_advance -> src/arena.c:646
  └─ [后台提前清除] src/arena.c:761-765 arena_decay_try_purge(..., backlog limit)

arena_decay_to_limit -> src/arena.c:916 arena_decay_to_limit(...)
  ├─ [guard] src/arena.c:923-927 decay->purging 时返回，否则置位并解锁
  ├─ src/arena.c:934 arena_stash_decayed(...) -> npurge
  ├─ [npurge != 0] src/arena.c:937 arena_decay_stashed(...) -> npurged
  └─ src/arena.c:943-944 重新加锁并清 purging

后台线程链:
  background_thread0_work (src/background_thread.c:434)
    ├─ src/background_thread.c:448 check_background_thread_creation
    └─ src/background_thread.c:452 background_work_sleep_once
  background_work (src/background_thread.c:483)
    └─ [ind != 0] src/background_thread.c:497 background_work_sleep_once -> 唤醒后执行 arena_decay

extent 层链:
  arena_decay_stashed -> ... -> src/extent.c:567 extents_evict
    ├─ src/extent.c:581 extent_list_first(&extents->lru) -> LRU 候选
    ├─ src/extent.c:586-591 npages <= npages_min -> 停止
    ├─ src/extent.c:597 extent_try_delayed_coalesce
    └─ [retained] src/extent.c:619 extent_deregister
  arena_decay_stashed -> ... -> src/extent.c:2035 extent_purge_forced_impl
    ├─ src/extent.c:2043 purge_forced hook == NULL -> 返回 true
    └─ src/extent.c:2049-2051 调用 hook->purge_forced(base, size, offset, length, arena_ind)
```

## 3. decay_ms 规则表

`src/arena.c:719` 读取 `decay_ms` 后，`720-727` 行用符号值区分三种模式；`src/arena.c:536` 在 `decay_ms > 0` 时为 deadline 叠加 `[0, interval)` 均匀 jitter，避免多 arena 同步清除。

| `decay_ms` | 字面锚点 | 行为 | 返回值 |
| --- | --- | --- | --- |
| `< 0` | `src/arena.c:720` | 禁用；仅在 `== 0` 分支才动作 | `false` |
| `== 0` | `src/arena.c:721-724` | `arena_decay_to_limit(..., all=false, npages_limit=0, npages_decay_max=当前全部)` 立即清空 | `false` |
| `> 0` | `src/arena.c:729-766` | 取单调时间、必要时推进 epoch、按 backlog 限流清除 | `advance_epoch` |
| 时钟回退 | `src/arena.c:732-744` | 把 epoch 拉回 `time` 并 `arena_decay_deadline_init` | 继续流程 |
| 后台非 epoch 触发 | `src/arena.c:761-765` | `arena_decay_try_purge(..., arena_decay_backlog_npages_limit)` | `false` |

`src/arena.c:787` 的 `arena_decay_ms_set` 在 `789-791` 行先校验合法性，再于 `802` 行 `arena_decay_reinit` 重置 backlog，`803` 行立刻调用 `arena_maybe_decay`。源码注释 `794-801` 说明：不做旧 backlog 映射，因此改值可能立即清除大量 dirty 页，设计前提是这类变更很罕见。

## 4. 限流清除与 OS 交界

`src/arena.c:916` 的 `arena_decay_to_limit` 用 `purging` 标志（`923-927`）保证同一 decay 结构同时只有一个清除者；解锁后调用 `arena_stash_decayed`（`934`）挑选 extent，再由 `arena_decay_stashed`（`937`）执行实际 purge。

| 步骤 | 锚点 | 关键规则 |
| --- | --- | --- |
| 选候选 | `src/extent.c:567` `extents_evict` | 从 `extents->lru` 取最早 extent（`581`） |
| 下限保护 | `src/extent.c:586-591` | `extents_npages <= npages_min` 时停止，不清到低于限流目标 |
| 延迟合并 | `src/extent.c:593-600` | `delay_coalesce` 为真时先 `extent_try_delayed_coalesce`，成功即 break |
| 状态切换 | `src/extent.c:611-623` | dirty/muzzy 置为 `active`；retained 走 `src/extent.c:619` 注销 |
| 强制清除 | `src/extent.c:2035` `extent_purge_forced_impl` | hook 为空直接返回 `true`（`2043-2045`）；否则 `2049-2051` 下发到 OS 层 |

`src/extent.c:2038-2039` 的 witness 深度依据 `growing_retained` 在 0 与 1 之间切换；自定义 hook 非默认时，`2046-2054` 包一层 `extent_hook_pre_reentrancy`／`extent_hook_post_reentrancy`。

## 5. 后台线程生命周期

`src/background_thread.c:483` 的 `background_work` 把索引 0 当作控制线程：`489-490` 转 `background_thread0_work`；其他索引在 `state != background_thread_stopped` 时循环 `background_thread_pause_check`（`493`）与 `background_work_sleep_once`（`497`）。`500-502` 在退出时把唤醒时间先设为 `BACKGROUND_THREAD_INDEFINITE_SLEEP`（`487-488`）再复位为 0，避免竞态唤醒。

`src/background_thread.c:434` 的 `background_thread0_work` 额外负责在 `448` 行 `check_background_thread_creation` 按需拉起线程，并在 `461-477` 行统一收尾；`478-479` 行断言最终 `n_background_threads == 1`。

## 6. 共变耦合与修改清单（Co-Change Playbook）

| 改动点 | 联动文件／符号 | 隐式陷阱 | 建议验证 |
| --- | --- | --- | --- |
| 改 decay_ms 语义 | `src/arena.c:719` → `722`／`757` → `787` | 三态 `{<0,=0,>0}` 必须一致，否则配置禁用仍全清 | 三档 mallctl 切换对照 |
| 改 jitter 算法 | `src/arena.c:534-541` → `546` 的 deadline 比较 | 去掉 jitter 会让多 arena 同频清除，造成延迟尖刺 | 多 arena 时间序列采样 |
| 改限流阈值 | `src/arena.c:934` → `src/extent.c:586` `npages_min` | 下限判断用 `<=`，改错会多清一页 | 精确页数断言 |
| 改后台唤醒 | `src/background_thread.c:497` → `src/arena.c:761-765` | 后台提前清除依赖 backlog 限制，唤醒过频会退化为逐次全清 | 后台开关 A/B 测延迟 |
| 改强制清除 hook | `src/extent.c:2043-2051` → 上层 `arena_decay_stashed` | hook 返回 `true` 表示失败，需上层决定是否重试 | 自定义 hook 注入失败 |

衰减流与[配置契约](../contracts/configuration.md)中的 `dirty_decay_ms`／`muzzy_decay_ms`／`background_thread` 三项直接耦合，修改任一处都应重新核对状态机。
