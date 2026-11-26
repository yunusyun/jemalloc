---
kind: operations
scope: pitfalls-and-adr
status: draft
origin: generated
source_paths: [src/zone.c, src/safety_check.c, src/jemalloc.c, src/extent.c, scripts/gen_run_tests.py, run_tests.sh]
symbols: [zone_promote, malloc_default_purgeable_zone, safety_check_fail, malloc_conf_init_helper, extent_split_interior]
upstream: [INDEX.md, architecture/index.md]
downstream: [operations/boundary.md]
---

# 陷阱清单与架构决策记录（ADR）

## 1. 死代码 / 未挂载代码警告

`src/zone.c:6-8` 头部是硬编译断言：

```text
#ifndef JEMALLOC_ZONE
#  error "This source file is for zones on Darwin (OS X)."
#endif
```

即该文件**仅在 Darwin 且启用 zone allocator 时参与编译**；在 Linux/BSD 构建中它不是活代码，改动它不会影响非 Darwin 行为，也不会被常规 CI 覆盖。`INSTALL.md:165-168` 的 `--disable-zone-allocator` 可显式关闭接管。因此：`zone.c` 的任何修复必须同时给出 Darwin 实机验证路径，否则只能记为未验证变更。

zone 的提升流程是 constructor 驱动的：`zone_register`（`src/zone.c:436-469`）先用 `zone_default_get`（`:369-397`）取默认 zone，若 `zone_name` 不是 `DefaultMallocZone` 则直接返回（`:443-447`）；随后在仍为 `scalable_zone` 时提前调用 `malloc_default_purgeable_zone`（`src/zone.c:460-461`），避免默认 zone 被替换后 libc 误判；再 `zone_init`（`:332-367`）注册（`:464-465`），最后 `zone_promote`（`:401-434`）反复 `malloc_zone_unregister`/`malloc_zone_register` 直到 `zone_default_get()` 返回 `&jemalloc_zone`（`:432-433`）。源码注释记录了 OSX 10.6 前后注销行为差异（`:405-425`），这是典型平台脆弱点。

## 2. 四源配置优先级陷阱

`MALLOC_CONF_NSOURCES 4`（`src/jemalloc.c:935`）与 `obtain_malloc_conf`（`src/jemalloc.c:938`）定义四源；`malloc_conf_init_helper`（`src/jemalloc.c:1021`）以 `i=0..3` 顺序解析（`:1035-1040`）。三个常被误判的点：

| 陷阱 | 字面证据 | 后果 |
|---|---|---|
| 后解析覆盖同名键 | 循环顺序 `src/jemalloc.c:1035`；`opts_explain` `:1024-1030` | `MALLOC_CONF` 环境变量最终胜出，构建期 `--with-malloc-conf` 最弱 |
| `/etc/malloc.conf` 读的是**链接名**而非文件内容 | `readlink`（`src/jemalloc.c:980`）/`readlinkat`（`:982`），`buf[linklen]='\0'`（`:991`） | 把配置写进目标文件无效；必须是 symlink 名字本身 |
| 安全环境下降级为 `getenv` | `jemalloc_secure_getenv`（`src/jemalloc.c:712-723`）：有 secure_getenv 用它（`:713-714`），否则 `issetugid()!=0` 返回 NULL（`:716-719`），再回落 `getenv`（`:721`） | setuid 程序与普通程序读到的配置可能不同 |

`confirm_conf:true` 可打印每源取值（`src/jemalloc.c:1041-1044`），是排查「配置未生效」的第一手段。完整键表见 [配置字典](../contracts/configuration.md)，做法见 [部署页](deployment-and-runtime.md)。

## 3. 校验失败与分裂失败的行为陷阱

`safety_check_fail`（`src/safety_check.c:10-23`）默认 `malloc_write(buf)` + `abort()`（`:18-20`），但 `safety_check_set_abort`（`:6-8`）可注入自定义回调（`:21-23`）。陷阱：`src/tcache.c:125-129` 在 `safety_check_fail` 后**又**写了一次 `abort()`（`:129`），所以即使注入了不终止的回调，tcache size mismatch 仍会硬终止。此外 `config_opt_safety_checks` 由 `JEMALLOC_OPT_SAFETY_CHECKS` 或 `JEMALLOC_DEBUG` 打开（`include/jemalloc/internal/jemalloc_preamble.h.in:168-181`），发布构建可能完全不做这些检查。

`extent_split_interior`（`src/extent.c:980-1013`）把失败分为 `extent_split_interior_cant_alloc`（空间不足，`:992-994`）与 `extent_split_interior_error`（内部拆分失败，`:1008-1012`），并通过 `to_leak`/`to_salvage` 出参交回清理责任（`:999-1000`）。调用方若忽略这两个出参会泄漏 extent；改动拆分逻辑必须同步审计所有调用点。

## 4. 测试脚本陷阱

`run_tests.sh` 只有一行 `$(dirname "$)")/scripts/gen_run_tests.py | bash`（`run_tests.sh:1`）：`"$)"` 破坏 `dirname` 参数，脚本畸形。`scripts/gen_run_tests.py:1` 的 shebang 为 `#!/usr/bin/env python`，第 19 行用 `xrange`（`scripts/gen_run_tests.py:19`），是 Python 2 语法，Python 3 下直接失败。两文件按原样记录，**不可**作为测试通过依据；可信入口是 `Makefile.in:539` / `:549` / `:553` 的 `check_unit`/`check_integration`/`check` 与 `test/test.sh.in:26-33` 的结果码。详见 [开发与测试](dev-and-test.md)。

## 5. ADR（基于源码证据的已落地决策）

| ADR | 决策 | 证据 | 取舍 |
|---|---|---|---|
| ADR-001 | Darwin 以 malloc zone 接管进程默认分配器，而非改写 libc 符号 | `src/zone.c:6-8`、`:332-367`、`:436-469`；`INSTALL.md:165-168` | 获得系统级接管能力；代价是平台相关死代码与 OSX 版本脆弱性 |
| ADR-002 | 配置采用四源顺序解析、后者覆盖前者 | `src/jemalloc.c:935`、`:938`、`:1021`、`:1035-1040` | 运维可用环境变量即时覆盖；代价是构建期默认值易被静默改写 |
| ADR-003 | 安全校验默认 abort，保留回调注入点 | `src/safety_check.c:4-23`、`src/tcache.c:125-129` | 快速失败保护元数据；代价是回调语义被 `abort()` 二次约束 |
| ADR-004 | extent 内部分裂错误显式区分「不可分配」与「内部错误」，并交回泄漏/挽救对象 | `src/extent.c:976-1013` | 便于调用方分场景清理；代价是出参协议复杂 |
| ADR-005 | 测试门禁以 Makefile 目标与 `test/test.sh` 为准，不依赖矩阵生成脚本 | `Makefile.in:539-553`、`test/test.sh.in:26-80`、`run_tests.sh:1`、`scripts/gen_run_tests.py:19` | 门禁可重复；代价是失去自动组合矩阵 |

架构上下文见 [架构索引](../architecture/index.md)；无法在仓内闭合的外部依赖见 [边界页](boundary.md)。

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改 Darwin zone：联动 `src/zone.c:6-8` 的编译守卫、`:332-367` 的 `zone_init`、`:401-434` 的 `zone_promote`、`:460-461` 的 purgeable zone 调用与 `INSTALL.md:165-168`；必须标注 Darwin 实机未验证项。
2. 改配置优先级：联动 `src/jemalloc.c:935 / 938 / 1021 / 1035-1040 / 712-723`、[配置字典](../contracts/configuration.md)与 [部署页](deployment-and-runtime.md)。
3. 改安全校验：联动 `src/safety_check.c:4-23`、`src/tcache.c:125-129`、`include/jemalloc/internal/jemalloc_preamble.h.in:168-181` 与 [错误码字典](../contracts/error-codes.md)。
4. 改 extent 拆分：联动 `src/extent.c:976-1013` 的枚举与全部调用点，复核 `to_leak`/`to_salvage` 的清理责任，避免 extent 泄漏。
5. 修复测试脚本：联动 `run_tests.sh:1`、`scripts/gen_run_tests.py:19` 与 [开发与测试](dev-and-test.md)；修复前不得把其退出码写入验收结论。
