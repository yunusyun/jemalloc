---
kind: operations
scope: dev-and-test
status: draft
origin: generated
source_paths: [Makefile.in, autogen.sh, test/test.sh.in, test/unit/mallctl.c, test/integration/malloc.c, run_tests.sh, scripts/gen_run_tests.py]
symbols: [tests, check, check_unit, check_integration]
upstream: [INDEX.md]
downstream: [operations/boundary.md]
---

# 开发与测试操作手册

## 1. 真实引导与构建命令

本仓**没有**提交的 `configure`（只有 `configure.ac` 与 `Makefile.in`），开发者必须先生成配置脚本。`autogen.sh` 的内容是 `for i in autoconf; do ... $i; done`（`autogen.sh:3-10`）随后 `./configure --enable-autogen $@`（`autogen.sh:13`），任一步失败即 `exit 1`（`:7`、`:16`）。`Makefile.in:297` 的 `all: build_lib` 是默认构建目标。

| 场景 | 命令 | 依据 |
|---|---|---|
| 开发者源码树 | `./autogen.sh && make` | `autogen.sh:3-13`、`Makefile.in:297` |
| 打包发行版 | `./configure && make && make install` | `INSTALL.md:2-8` |
| 仅编译测试二进制 | `make tests` | `Makefile.in:529` |
| 跑全量检查 | `make check` | `Makefile.in:553` |
| 仅跑单元测试 | `make check_unit` | `Makefile.in:539-540` |
| 仅跑集成测试 | `make check_integration` | `Makefile.in:549-550` |
| 清理 | `make clean` / `make distclean` / `make relclean` | `Makefile.in:555`、`:586`、`:596` |

`distclean` 会删除生成的 `bin/jemalloc-config`、`bin/jemalloc.sh`、`bin/jeprof`（`Makefile.in:587-589`），`relclean` 进一步删除 `configure` 与 `VERSION`（`:597-598`）。配置开关语义见 [配置字典](../contracts/configuration.md)。

## 2. 测试目标拓扑与执行链

`Makefile.in:526-528` 定义 `tests_unit`/`tests_integration`/`tests_stress` 三个编译目标，`tests` 汇总三者（`:529`）。运行层由 `test/test.sh.in` 承担：

```text
make check(:553) -> check_unit(:539) + check_integration(:549)
                   + check_integration_decay(:546) + check_integration_prof(:541)
  -> test/test.sh.in:34 逐二进制循环
       ├─ [存在同名 .sh] source 该脚本以设置 MALLOC_CONF(:39-48)
       ├─ export_malloc_conf 合并 MALLOC_CONF_ALL(:14-24)
       └─ $JEMALLOC_TEST_PREFIX <bin> @abs_srcroot@ @abs_objroot@(:48/:52)
  -> 结果码 0/1/2 对应 pass/skip/fail(:26-33/:55-65)
  -> fail_count>0 则 exit 1(:77-80)
```

`test/test.sh.in:3-12` 按 `@abi@` 设置动态库搜索路径：`macho` 用 `DYLD_FALLBACK_LIBRARY_PATH`，`pecoff` 追加 `PATH`，其余分支为空。异常结果会打印 `Use prefix to debug, e.g. JEMALLOC_TEST_PREFIX="gdb --args" sh test/test.sh ${t}`（`test/test.sh.in:67`）。

| make 目标 | 注入的 MALLOC_CONF | 锚点 |
|---|---|---|
| `check_unit` | 无额外注入（沿用测试自身 `.sh`） | `Makefile.in:540` |
| `check_integration` | 无额外注入 | `Makefile.in:550` |
| `check_integration_decay` | `dirty_decay_ms:-1,muzzy_decay_ms:-1` 与 `dirty_decay_ms:0,muzzy_decay_ms:0` 两组 | `Makefile.in:546-548` |
| `check_integration_prof` | `prof:true` 与 `prof:true,prof_active:false` 两组（仅 `enable_prof=1`） | `Makefile.in:541-545` |
| `stress` | 无额外注入 | `Makefile.in:551-552` |

## 3. 两个代表性测试样例

`test/unit/mallctl.c:6-31` 的 `test_mallctl_errors` 逐项断言：`mallctl("no_such_name", ...)` → `ENOENT`（`:10-11`）；对只读 `version` 写入 → `EPERM`（`:13-15`）；`epoch` 写入长度 `sizeof(epoch)-1` 与 `+1` → `EINVAL`（`:17-22`），读取长度不匹配 → `EINVAL`（`:24-29`）。同文件 `test_mallctlnametomib_errors`（`:33-41`）断言 `mallctlnametomib("no_such_name", ...)` → `ENOENT`（`:38-39`）。这与 [mallctl 命名空间](../contracts/api/mallctl.md)的错误表一一对应。

`test/integration/malloc.c:3-10` 的 `test_zero_alloc` 验证 `malloc(0)` 返回非空、`malloc_usable_size(res) > 0`、`free(res)` 正常；`main` 仅注册该用例（`:12-16`）。这是检查零大小分配快路径与 `isalloc` 口径的最小集成门禁。

## 4. `run_tests.sh` 与生成脚本的真实状态（不宣称可用）

`run_tests.sh` **只有一行**：`$(dirname "$)")/scripts/gen_run_tests.py | bash`（`run_tests.sh:1`）。其中 `"$)"` 不是合法的 shell 参数展开，`dirname` 的参数被破坏；该文件按字面记录为**畸形脚本**，不能作为可靠测试入口。

`scripts/gen_run_tests.py:1` 的 shebang 是 `#!/usr/bin/env python`；脚本使用 `xrange(len(items) + 1)`（`scripts/gen_run_tests.py:19`），这是 **Python 2 语法**，在 Python 3 下直接 `NameError`。其余逻辑是组合编译器（`gcc/g++`、`clang/clang++`，`:25-31`）、选项 `-m32`（`:33`）与配置选项 `--enable-debug` 等（`:35` 起）生成交叉测试矩阵，`nparallel = cpu_count() * 2`（`:12`），并按 `uname` 选择 `gmake` 或 `make`（`:16-20`）。结论：应使用第 1、2 节的 `autogen.sh` / `make` 目标作为唯一可信命令，不把这两个脚本当作通过门禁。

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 增删测试二进制：同步 `Makefile.in:526-528` 的 `TESTS_UNIT/TESTS_INTEGRATION/TESTS_STRESS` 变量、`tests/check` 依赖链（`:529`、`:553`）、`test/test.sh.in:34-53` 的传参约定；新增同名 `.sh` 可覆盖 `MALLOC_CONF`。
2. 改注入配置：联动 `Makefile.in:541-548` 的 `MALLOC_CONF` 字面串与 [配置字典](../contracts/configuration.md)的键名，确保键在 `src/jemalloc.c:1170-1414` 可解析。
3. 改 mallctl 返回码：联动 `test/unit/mallctl.c:6-41` 与 [mallctl 契约](../contracts/api/mallctl.md)错误表、`src/ctl.c:1292-1364`。
4. 改构建产物名：联动 `Makefile.in:586-589` 的 `distclean` 清单与 `bin/jemalloc-config.in` / `bin/jemalloc.sh.in` / `bin/jeprof.in` 模板。
5. 触碰 `run_tests.sh` 或 `scripts/gen_run_tests.py` 前先确认 Python 2 兼容性问题是否仍在（`scripts/gen_run_tests.py:19`）；未修复时不得把其输出当作测试通过证据，边界说明见 [边界页](boundary.md)。
