---
kind: component
scope: cli-tools
status: draft
origin: generated
source_paths: [bin/jemalloc-config.in, bin/jemalloc.sh.in, bin/jeprof.in, include/jemalloc/jemalloc.sh]
symbols: [usage, jemalloc-config, jemalloc.sh, jeprof]
upstream: [INDEX.md]
downstream: [operations/deployment-and-runtime.md, contracts/api-reference.md]
---

# 命令行工具（cli-tools）

## 1. 角色与边界

本组件覆盖 jemalloc 分发的三个 CLI 入口与一个头文件生成脚本：构建信息查询 `jemalloc-config`、LD_PRELOAD 包装器 `jemalloc.sh`、堆剖析报告器 `jeprof`，以及合并生成 `jemalloc.h` 的 `include/jemalloc/jemalloc.sh`。全站入口见 [INDEX.md](../INDEX.md)，部署与运行时注入见 [operations/deployment-and-runtime.md](../operations/deployment-and-runtime.md)，C API 契约见 [contracts/api-reference.md](../contracts/api-reference.md)，dump 产出侧见 [architecture/profiler-and-background.md](profiler-and-background.md)。本组件不含服务端：`jeprof` 的远程 profile 拉取目标是外部 HTTP 服务，记为 `[外部黑盒：本仓未包含服务端实现]`。

## 2. 入口与符号契约表

| 文件:行号 | 符号 | 类型 | 入 → 出 |
|---|---|---|---|
| `bin/jemalloc-config.in:3` | `usage` | shell 函数 | 无参 → here-doc 打印选项表 |
| `bin/jemalloc-config.in:6` | `jemalloc-config` | CLI 名（字面出现于 usage 文本） | 单选项 → 单行标量值 |
| `bin/jemalloc.sh.in:7` | `jemalloc.sh` | LD_PRELOAD 包装器 | 任意命令 → `exec "$@"` |
| `bin/jeprof.in:149` | `jeprof` | Perl CLI 用法串 | 选项 + profile → 文本/图形报告 |
| `include/jemalloc/jemalloc.sh:5` | `jemalloc.sh` | 头文件生成脚本 | `objroot` 参数 → 合并的 `jemalloc.h` |

`symbols` 中的四项为工具/函数名而非 C 导出符号：`usage` 是 `bin/jemalloc-config.in:3` 的 shell 函数；`jemalloc-config`、`jemalloc.sh`、`jeprof` 分别是各自脚本的工具名，已在上表给出逐字锚点。

## 3. 逐文件源码分析

`bin/jemalloc-config.in` 在 `usage`（`:3-25`）中列出 `--version`、`--revision`、`--config`、`--prefix`、`--bindir`、`--datadir`、`--includedir`、`--libdir`、`--mandir`、`--cc`、`--cflags`、`--cppflags`、`--cxxflags`、`--ldflags`、`--libs`；`:27-28` 定义 `prefix`/`exec_prefix`，`:30` 起以 `case "$1"` 逐项回显 `@...@` 占位符（如 `--version` 回显 `@jemalloc_version@`，`:35-37`）。这些占位符由 configure 替换，脚本本身不做路径推算。

`bin/jemalloc.sh.in` 仅 9 行：`:3-5` 取 `@prefix@`/`@exec_prefix@`/`@libdir@`，`:7` 将 `@LD_PRELOAD_VAR@` 设为 `${libdir}/libjemalloc.@SOREV@`，`:8` 导出，`:9` `exec "$@"`。变量名由平台决定（Linux 为 `LD_PRELOAD`，Darwin 为 `DYLD_INSERT_LIBRARIES`），解释见 [operations/deployment-and-runtime.md](../operations/deployment-and-runtime.md)。

`bin/jeprof.in` 为 Perl 脚本，shebang 在 `:1`（`#! /usr/bin/env perl`）。`usage_string`（`:146`）给出三种输入形态：程序加空格分隔的 profile 列表（`:149-150`）、符号化 profile（`:151-154`）、远程 `host:port` 形式（`:155-160`）；`:162-164` 列出 `$HEAP_PAGE`、`$PROFILE_PAGE`、`$GROWTH_PAGE`、`$CONTENTION_PAGE` 等服务端点，`:168` 提供 `--symbols` 地址映射模式。`JEPROF_TMPDIR` 默认值见 `:243`。

`include/jemalloc/jemalloc.sh` 与上面同名的运行时包装器**不是同一脚本**：它在 `:3` 取 `objroot`，`:5-12` 输出头文件起始围栏，`:14-20` 依次拼接 `jemalloc_defs.h`、`jemalloc_rename.h`、`jemalloc_macros.h`、`jemalloc_protos.h`、`jemalloc_typedefs.h`、`jemalloc_mangle.h`，并在 `:22-27` 收尾，产物是单一 `jemalloc.h`。

## 4. 调用链与失败语义

```text
jemalloc-config 选项 -> bin/jemalloc-config.in:30 case -> 回显 @...@ 标量
  └─ [默认] *) -> usage bin/jemalloc-config.in:81 -> exit 1 bin/jemalloc-config.in:82

jemalloc.sh 命令 -> bin/jemalloc.sh.in:7 设置 preload -> :8 export -> :9 exec
  └─ [外部黑盒] 动态链接器完成库注入

jeprof 选项 profile -> bin/jeprof.in:149 解析 -> 符号化 -> 文本/图形输出
  └─ [外部黑盒] host:port 远端服务拉取
```

`jemalloc-config` 的 `case` 在 `:80-82` 有 `*` 默认分支：未匹配选项调用 `usage` 并以 `exit 1` 退出，调用方据此判断选项合法性。

## 5. 共变耦合与修改清单（Co-Change Playbook）

- 新增 configure 输出变量 → 同步 `bin/jemalloc-config.in` 的 `usage`（`:3-25`）与 `case` 分支（`:30` 起），并更新 [operations/deployment-and-runtime.md](../operations/deployment-and-runtime.md) 的查询示例。
- 改动 `@SOREV@` 或库安装路径 → 同步 `bin/jemalloc.sh.in:7` 与 [operations/deployment-and-runtime.md](../operations/deployment-and-runtime.md) 的预加载前提。
- 改动堆转储文件名前缀 → 同步 `jeprof` 示例（`bin/jeprof.in:248`）与 [architecture/profiler-and-background.md](profiler-and-background.md) 的 `prof_prefix` 契约。
- 新增内部头文件 → 同步 `include/jemalloc/jemalloc.sh:14-15` 的 `for hdr in` 列表与 [contracts/api-reference.md](../contracts/api-reference.md) 的公开头清单。
- 验证命令：`make install` 后执行 `jemalloc-config --version`、`jemalloc.sh /bin/true`、`jeprof --help`；`jeprof` 需 Perl 运行时。
- 隐式陷阱：`include/jemalloc/jemalloc.sh` 是构建期脚本，不可作为运行时 LD_PRELOAD 包装器使用；两者同名但职责不同。
