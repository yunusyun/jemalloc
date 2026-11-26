---
kind: component
scope: profiler-and-background
status: draft
origin: generated
source_paths: [src/prof.c, src/background_thread.c, src/hook.c]
symbols: [prof_dump, prof_gdump, prof_sample_threshold_update, background_thread_create, background_work, hook_install]
upstream: [architecture/allocator-core.md]
downstream: [business/wf-profile.md, business/wf-decay.md]
---

# 性能剖析与后台线程（profiler-and-background）

## 1. 角色与边界

本组件承担堆剖析采样、采样阈值重算、堆转储（dump）、增长触发转储（gdump）、后台线程调度与分配/释放 hook 注入。上游分配快路径的采样判定见 [architecture/allocator-core.md](allocator-core.md)，剖析业务流见 [business/wf-profile.md](../business/wf-profile.md)，decay 业务流见 [business/wf-decay.md](../business/wf-decay.md)。

## 2. 符号级调用树

```text
采样判定（分配路径）：
  je_malloc fastpath -> bytes_until_sample -= usize src/jemalloc.c:2351-2355
    ├─ 为负 -> malloc_default -> prof_alloc_prep（内联）-> prof_malloc（内联）
    └─ prof_sample_threshold_update src/prof.c:1140
         └─ lg_prof_sample == 0 -> bytes_until_sample=0；否则几何分布抽样 src/prof.c:1146-1158

堆转储：
  prof_gdump src/prof.c:2106
    ├─ [guard] !prof_booted / tsdn_null / !prof_active -> 直接返回 src/prof.c:2112
    ├─ [guard] reentrancy_level > 0 -> 返回 src/prof.c:2116
    ├─ [guard] tdata->enq -> 置 enq_gdump 延迟 src/prof.c:2124-2126
    └─ prof_dump src/prof.c:1906 -> 锁 prof_dump_mtx src/prof.c:1917 -> 合并 gctx/tdata

后台线程：
  background_thread_create src/background_thread.c:593 -> background_thread_create_locked
  background_work src/background_thread.c:483
    ├─ ind==0 -> background_thread0_work src/background_thread.c:434
    └─ ind>0  -> background_work_sleep_once src/background_thread.c:497

hook：
  hook_install src/hook.c:48 -> hook_install_locked src/hook.c:28 -> tsd_global_slow_inc
  hook_invoke_alloc src/hook.c:154 -> 逐 slot 调 alloc_hook
```

`prof_alloc_prep` 与 `prof_malloc` 在 `src/prof.c` 中 `[仓库内未找到字面实现]`：二者逐字定义于 `include/jemalloc/internal/prof_inlines_b.h:146` 与 `:166`（`JEMALLOC_ALWAYS_INLINE`）。

## 3. 源码锚点与职责契约表

| 文件:行号 | 符号 | 职责 | 入 → 出 |
|---|---|---|---|
| `src/prof.c:1140` | `prof_sample_threshold_update` | 用几何分布重算下次采样间隔 | `prof_tdata_t*` → `void` |
| `src/prof.c:2106` | `prof_gdump` | 增长阈值触发堆转储 | `tsdn_t*` → `void` |
| `src/prof.c:1906` | `prof_dump` | 合并上下文并写堆转储 | `tsd_t*`,`filename`,`leakcheck` → `bool` |
| `src/background_thread.c:593` | `background_thread_create` | 按需创建后台线程 | `arena_ind` → `bool` |
| `src/background_thread.c:483` | `background_work` | 后台周期任务分发 | `tsd_t*`,`ind` → `void` |
| `src/hook.c:48` | `hook_install` | 安装分配/释放 hook 槽位 | `hooks_t*` → `void*` 句柄 |

上下文 surveying 给出的 `src/prof.c:327`、`src/prof.c:581` 分别是 `prof_sample_threshold_update` 与 `prof_gdump` 的调用点；本页已据实改引定义行 `src/prof.c:1140`、`src/prof.c:2106`。

## 4. 后台调度与 decay 联动

`background_thread_create`（`src/background_thread.c:593`）持 `background_thread_lock` 调用 `background_thread_create_locked`；`background_threads_enable`（`src/background_thread.c:605`）要求当前线程数从 0 起步。`background_work`（`src/background_thread.c:483`）先锁 `info->mtx` 并把唤醒时间置为 `BACKGROUND_THREAD_INDEFINITE_SLEEP`（`src/background_thread.c:486-488`），然后 ind==0 走 `background_thread0_work`（额外负责启停其他线程），ind>0 走 `background_work_sleep_once`（`src/background_thread.c:497`）。该周期任务是 decay 的执行载体，与 [business/wf-decay.md](../business/wf-decay.md) 的 `dirty_decay_ms`/`muzzy_decay_ms` 节拍一致。

## 5. 转储、重入与 hook 边界

`prof_dump`（`src/prof.c:1906`）断言 `tsd_reentrancy_level_get(tsd) == 0`（`src/prof.c:1909`），先 `pre_reentrancy` 再锁 `prof_dump_mtx`（`src/prof.c:1916-1917`），通过 `prof_tdata_merge_iter` 与 `prof_gctx_merge_iter` 汇总后写出。`prof_gdump` 在 `opt_prof_prefix[0] != '\0'` 时构造文件名（`src/prof.c:2129-2130`）。`prof_active_get`（`src/prof.c:2400`）以 `prof_active_mtx` 保护布尔读取。

`hook_install`（`src/hook.c:48`）在成功占用 slot 后递增 `tsd_global_slow_inc`，使所有线程从快路径转入慢路径；这是 hook 生效的必要代价。`hook_invoke_alloc`（`src/hook.c:154`）通过 `FOR_EACH_HOOK_BEGIN` 遍历已安装 slot，hook 内部再分配必须避免重入死循环。

## 6. 共变耦合与修改清单（Co-Change Playbook）

- 改动采样分桶或 `lg_prof_sample` 语义 → 同步 `prof_sample_threshold_update`（`src/prof.c:1140`）、快路径扣减（`src/jemalloc.c:2351-2355`）与 [business/wf-profile.md](../business/wf-profile.md) 的采样率说明。
- 改动 dump 文件名格式 → 同步 `prof_gdump`（`src/prof.c:2129`）与 `prof_dump`（`src/prof.c:1906`）及 [business/wf-profile.md](../business/wf-profile.md) 的 `prof_prefix` 契约。
- 改动后台线程职责划分 → 同步 `background_work`（`src/background_thread.c:483`）、`background_thread0_work`（`src/background_thread.c:434`）与 [business/wf-decay.md](../business/wf-decay.md) 的节拍表。
- 改动 hook slot 数或 `HOOK_MAX` → 同步 `hook_install_locked`（`src/hook.c:28`）、`hook_invoke_alloc`（`src/hook.c:154`）与 `hook_invoke_dalloc`（`src/hook.c:170`）。
- 验证命令：`make tests && make check`；重点单测 `test/unit/prof.c`、`test/unit/background_thread.c`。
- 隐式陷阱：`prof_dump` 要求重入级别为 0（`src/prof.c:1909`）；从 hook 回调内直接触发 dump 会触发断言，必须先退出 hook 上下文。
