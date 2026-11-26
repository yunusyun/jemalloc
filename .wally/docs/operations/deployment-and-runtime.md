---
kind: operations
scope: deployment-and-runtime
status: draft
origin: generated
source_paths: [Makefile.in, bin/jemalloc-config.in, bin/jemalloc.sh.in, INSTALL.md, configure.ac]
symbols: [install, prefix, DESTDIR]
upstream: [INDEX.md, contracts/configuration.md]
downstream: [operations/runbook-and-observability.md]
---

# 部署与运行时操作手册（库安装，非服务部署）

## 1. 部署对象定性

本仓交付 `libjemalloc` 与头文件、手册页、3 个 CLI 工具；**没有**任何服务进程、监听端口、HTTP 路由、健康检查端点或实例拓扑。`README:1-3` 明确其为 `malloc(3)` 实现；因此「部署」= 编译安装 + 应用链接/`LD_PRELOAD` + 运行期 `MALLOC_CONF`。本文不描述、也不臆造多实例服务架构。

## 2. 安装命令与前缀

| 路径 | 命令 | 字面依据 |
|---|---|---|
| 打包发行版 | `./configure && make && make install` | `INSTALL.md:4-6` |
| 开发者源码树 | `./autogen.sh && make dist && make && make install` | `INSTALL.md:11-14`；`autogen.sh:13` 会先执行 `./configure --enable-autogen` |
| 指定安装根 | `./configure --prefix=/usr/local` | `INSTALL.md:32-38`（写入 `/usr/local/include`、`/usr/local/lib`、`/usr/local/man`） |
| 暂存到别的根 | `make install DESTDIR="?"` | `INSTALL.md:371-374`：`DESTDIR` 前置到 `INCLUDEDIR/LIBDIR/DATADIR/MANDIR` |
| 分开安装 | `make install_bin` / `install_include` / `install_lib` / `install_doc` | `INSTALL.md:337-345`、`Makefile.in:520-524` |

`Makefile.in:520` 的 `install: install_bin install_include install_lib`，`enable_doc=1` 时追加 `install_doc`（`:522-524`）。`configure.ac:1-4` 要求 autoconf ≥ 2.68、`AC_INIT([Makefile.in])`、`AC_CONFIG_AUX_DIR([build-aux])`；`configure.ac` 只生成 `configure`，仓库不提交生成物。

```text
configure.ac -> configure -> Makefile.in:297 all: build_lib
  -> Makefile.in:520 install -> install_bin + install_include + install_lib [+ install_doc]
       -> prefix 决定绝对安装路径(:32 INSTALL.md)
       -> DESTDIR 仅在安装阶段做路径前置(:371-374 INSTALL.md)
```

## 3. 安装后自检工具

`bin/jemalloc-config.in:3` 的 `usage()` 列出全部选项（`:8-23`）；`case "$1"` 分派（`:30-83`）：`--version` 回显 `@jemalloc_version@`（`:35-37`）、`--prefix` 回显 `@PREFIX@`（`:44-46`）、`--libdir` 回显 `@LIBDIR@`（`:56-58`）、`--cc/--cflags/--ldflags/--libs` 分别回显编译链接参数（`:62-79`）。未知参数走 `usage` 并 `exit 1`（`:80-82`）。这些输出可用于确认安装前缀与实际编译选项是否与预期一致。

`bin/jemalloc.sh.in:7-9` 是进程级注入包装器：设置 `@LD_PRELOAD_VAR@=${libdir}/libjemalloc.@SOREV@` 并 `export`，再 `exec "$@"` 运行目标程序。它不改任何服务配置，仅影响当前命令的子进程。`bin/jeprof.in:1` 是 Perl 堆分析脚本，需与 prof 输出文件配合使用。

| 自检项 | 命令 | 期望 |
|---|---|---|
| 版本 | `jemalloc-config --version` | 与构建时 `@jemalloc_version@` 一致 |
| 前缀 | `jemalloc-config --prefix` | 与 `./configure --prefix` 一致 |
| 链接参数 | `jemalloc-config --ldflags --libs` | 返回构建时记录的 flags |
| 临时接管 | `./jemalloc.sh <app> [args...]` | 仅该子进程被 `LD_PRELOAD` 注入 |

## 4. 运行期配置投放

编译期 `--with-malloc-conf`（`INSTALL.md:112-120`）写入 `config_malloc_conf`（`include/jemalloc/internal/jemalloc_preamble.h.in:89`），其优先级低于全局 `malloc_conf`、`/etc/malloc.conf` 与 `MALLOC_CONF`（`src/jemalloc.c:951-1012`、`src/jemalloc.c:1024-1030`）。因此部署时最常见偏差是：环境变量静默覆盖构建期默认值。完整键表与优先级见 [配置字典](../contracts/configuration.md)。

| 部署开关（configure） | 字面锚点 | 运行期影响 |
|---|---|---|
| `--enable-debug` | `INSTALL.md:121-124` | 打开断言与校验，性能显著下降 |
| `--disable-stats` | `INSTALL.md:126-129` | 关闭 `config_stats`，`stats.*` 数据不可用 |
| `--enable-prof` | `INSTALL.md:131-140` | 打开堆分析，`prof.*` 节点可用 |
| `--disable-zone-allocator` | `INSTALL.md:165-168` | Darwin 下不再接管默认 malloc zone |
| `--disable-libdl` | `INSTALL.md:272-274` | 影响 lazy lock 的 `dlsym` 依赖 |
| `--disable-initial-exec-tls` | `INSTALL.md:263-270` | 允许 `dlopen` 后加载，但进程内可能并存两套分配器（`:268-270`） |

`INSTALL.md:397-415` 明确：非平凡改动应使用 `autogen.sh` 而非 `configure`，以便重新生成 `configure` 与自动生成的源文件；也支持 out-of-tree 构建（`autoconf` → `mkdir obj` → `cd obj` → `../configure --enable-autogen` → `make`）。排障与统计采集入口见 [排障与可观测](runbook-and-observability.md)。

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 改安装布局：联动 `Makefile.in:520-524` 的 `install` 依赖、`INSTALL.md:32-38 / 354-380` 的 `prefix`/`DESTDIR` 描述、`bin/jemalloc-config.in:27-28 / 44-58` 的路径回显。
2. 改 CLI 模板：联动 `bin/jemalloc-config.in:3-83`、`bin/jemalloc.sh.in:7-9`、`bin/jeprof.in:1`，以及 `Makefile.in:587-589` 的 `distclean` 清理清单。
3. 改编译期开关：联动 `configure.ac`、`include/jemalloc/internal/jemalloc_preamble.h.in:54-211`、`INSTALL.md:121-274` 与 [配置字典](../contracts/configuration.md)。
4. 改动默认 `MALLOC_CONF`：联动 `INSTALL.md:112-120`、`src/jemalloc.c:951-1012` 的四源解析与 [排障页](runbook-and-observability.md)中「配置未生效」分支。
5. 不得在本页新增服务拓扑、端口或健康检查描述；若上游架构变化，先核对 `README:1-3` 的库定位再改文档。
