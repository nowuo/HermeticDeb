# HermeticDeb

> **An Agentic, Zero-Host-Pollution Package Lifecycle Manager for Debian Systems**

**HermeticDeb** 是一套面向 Debian 桌面与工作站环境的本地 AI 驱动软件包全生命周期管理框架（Agentic Package Management Framework）。本项目旨在解决非 Debian 官方源独立应用程序（Out-of-Tree Applications）在本地安装、依赖追踪、源码编译与版本升级过程中的环境污染与维护成本问题。

通过将命令行 AI Agent（如 Claude Code、Aider、OpenHands 等）约束在严格的工程沙盒与硬件级 Cgroups 护栏内，HermeticDeb 实现了从上游版本探测、预编译包兼容性预检、同源容器增量构建、动态共享库依赖自动萃取，到双版本回滚归档的标准化闭环。

## 核心特性（Core Features）

### 1. 宿主机零污染（Zero Host Pollution）

- **工具链完全隔离**：宿主机无需安装 `cargo`、`zig`、`cmake`、`build-essential` 或任何 `-dev` 开发头文件。所有编译探查、依赖安装与打包操作均在单一常驻 Docker 容器（`deb-builder`）内完成。

- **纯血同源产物**：基建容器动态对齐宿主机 `/etc/os-release` 版本代号（如 `trixie`、`bookworm`），编译产物 100% 链接宿主机同源动态库，开启本机 CPU 指令集极限优化（`-march=native` / `target-cpu=native`），并通过 `dpkg-shlibdeps` 自动计算精准的运行时依赖（Runtime `Depends`）。

- **严禁魔改绕过**：坚决杜绝 `LD_PRELOAD`、非标动态库软链接或 Wrapper 脚本注入。所有交付物均为符合 Debian 规范的标准 `.deb` 安装包，由系统原生 `apt`/`dpkg` 统一纳管。

### 2. 三级漏斗演进策略（Three-Tier Update Funnel）

针对纳入管理的独立应用程序，HermeticDeb 严格遵循资源消耗最小化的三级演进路径：

- **Tier 1：原生自更新（Native Self-Updater）**：优先识别并调用软件内置的自更新机制。

- **Tier 2：官方预编译包（Official Pre-built `.deb`）**：拉取上游官方 Release 预编译包，经静态依赖与安装模拟预检（`apt-get install -s`）通过后直接交付。

- **Tier 3：常驻容器增量源码编译（Incremental Source Build）**：当无预编译包或预检/运行失败时，自动唤醒 `deb-builder` 常驻容器，基于持久化源码树与本地补丁库执行增量编译与 `.deb` 封装。

### 3. 硬件级防死机护栏（Hardware-Level Cgroups Guardrails）

- **零 Swap 颠簸铁律**：容器强制设定 `--memory == --memory-swap`（禁用容器 Swap）与 `--oom-score-adj=1000`。即使面对 LLVM/Zig/Rust LTO 等极端内存风暴，超限进程也将在容器内部被内核瞬间精准终止，确保宿主机 Wayland/X11 桌面合成器与前台应用零卡顿、零死机。

- **双因子并发控制**：每次构建前实时读取宿主机 `nproc` 与 `/proc/meminfo` 的 `MemAvailable`，按“CPU 核心数折半”与“每并发线程预留 3GB 可用内存”双因子取小计算并发度，并挂载 `nice -n 19 ionice -c 3` 空闲级调度策略。

### 4. 补丁持久化与双版本秒级回滚（Patch Persistence & Instant Rollback）

- **补丁自动重放（`patches/`）**：针对低内存机型对上游构建脚本的适配修改（如拆分编译单元、单文件优化降级），强制导出为标准 Git Patch 持久保存。跨版本升级时自动重置源码树并按序重放补丁，彻底消除增量升级中的“修改失忆”与合并冲突。

- **分应用双版本留存（`dist/<pkg>/`）**：每个软件包在独立子目录下固定保留当前生效版本（`current_deb`）与上一历史版本（`previous_deb`），支持随时通过单条命令完成故障降级回滚。

### 5. 隐私沙盒与供应链安全（Privacy Sandbox & Supply Chain Security）

- **严格工作区边界**：AI Agent 的文件读写权限严格锁定在项目根目录内，严禁刺探宿主机环境变量、网络配置、进程状态或用户家目录敏感文件。

- **供应链白名单锁死**：仅允许从 `versions.toml` 登记的官方上游仓库拉取代码与资产，严禁私自搜索或引入未经授权的第三方个人打包二进制。

- **人工权限裁决**：所有涉及宿主机的 `sudo` 操作均输出完整绝对路径命令，交由用户手动输入密码执行；未预期的编译异常强制触发三段式结构化报告，由用户拍板裁决。

## 目录架构（Directory Structure）

```text
HermeticDeb/
├── README.md                  # 项目说明文档
├── AGENTS.md                  # AI Agent 核心架构规范与安全沙盒指令集
├── Dockerfile.base            # 由 Agent 按思维链动态维护的同源基建容器定义
├── versions.toml              # 软件包状态、构建配方、补丁索引与护栏状态数据库
├── src/                       # 官方上游持久化源码库（挂载入容器，支持增量编译）
│   └── <pkg_name>/            # 各软件独立 Git 源码树
├── patches/                   # 本地构建适配补丁库（跨版本升级时自动 git apply）
│   └── <pkg_name>/
│       └── 01-xxx.patch
└── dist/                      # 产出的标准 .deb 归档库（按应用隔离，双版本留存）
    └── <pkg_name>/
        ├── <pkg>_<current>_amd64.deb
        └── <pkg>_<previous>_amd64.deb
```

## 环境要求（Prerequisites）

- **操作系统**：Debian GNU/Linux（Bookworm / Trixie / Sid）及衍生发行版。

- **容器引擎**：Docker Engine（需支持 Docker BuildKit 与 Cgroups v2 内存限制）。

- **系统工具**：`aptitude`（用于外来独立包扫描）、`dpkg-dev`、`curl`、`git`。

- **AI 客户端**：支持读取项目上下文规范的 CLI Agent（如 Claude Code、Aider、Cursor、Codex CLI 等）。

## 快速开始（Quick Start）

### 1. 初始化工作区

克隆或创建项目目录，并将核心规范文件置于根目录：

```bash
mkdir -p ~/Tools/HermeticDeb && cd ~/Tools/HermeticDeb
# 确保 AGENTS.md 已放置在当前目录
# 若使用 Claude Code，可创建软链接以自动加载上下文：
ln -s AGENTS.md CLAUDE.md
```

### 2. 扫描并纳管系统存量外来软件

在项目根目录下启动 AI Agent，下达存量审计指令：

```text
按照 AGENTS.md 的【流程 A】，扫描当前系统中非 Debian 官方源安装的独立 Application，过滤掉依赖库与已有 apt 源接管的软件，汇报清单并初始化 versions.toml。
```

### 3. 日常检查更新与全自动构建（开启新周期）

当需要检查所有纳管软件的更新或安装新软件时，直接向 Agent 下达自然语言指令：

```text
开启新的维护周期：按照 AGENTS.md 规范对齐基建容器护栏，检查 versions.toml 中所有软件的上游更新并完成升级构建。
```

或新增特定软件：

```text
按照 AGENTS.md 规范，将上游仓库为 <owner>/<repo> 的软件 <pkg_name> 纳入管理并完成首次构建。
```

### 4. 安装与版本回滚

Agent 完成预检或源码打包后，将在终端输出包含**完整绝对路径**的安装指令。用户仅需复制并手动输入密码执行：

- **安装/升级最新版本**：

  ```bash
  sudo apt install /abs/path/to/HermeticDeb/dist/<pkg_name>/<pkg_name>_<new_version>_amd64.deb
  ```

- **故障秒级回滚至上一版本**：

  ```bash
  sudo apt install --allow-downgrades /abs/path/to/HermeticDeb/dist/<pkg_name>/<pkg_name>_<old_version>_amd64.deb
  ```

## 状态数据库示例（`versions.toml` Schema）

`versions.toml` 由 AI Agent 自动维护，完整记录基建护栏状态与各软件的构建生命周期元数据：

```toml
[base_environment]
host_codename = "trixie"
cgroups_guardrail = "cpus=4,mem=2g,swap=2g,oom_adj=1000"
extra_setup_cmds = [
    "python3 -m venv /opt/custom-tools && /opt/custom-tools/bin/pip install 'blueprint-compiler>=0.20.0'"
]
last_aligned_cycle = "2026-03-30 03:00:00"

[packages.ghostty]
repo = "ghostty-org/ghostty"
trusted_prebuilt_url = ""
current_version = "1.4.0"
self_update = false
self_update_cmd = ""
install_method = "source_build"
heavy_memory_build = true
patches = ["patches/ghostty/01-low-mem-glslang.patch"]
build_recipe = "zig build -Doptimize=ReleaseFast -Dcpu=native -j1"
current_deb = "dist/ghostty/ghostty_1.4.0-1_amd64.deb"
previous_deb = "dist/ghostty/ghostty_1.3.1-1_amd64.deb"
updated_at = "2026-03-30"
```

## 安全与治理准则（Governance & Safety）

HermeticDeb 的所有自动化行为均受控于 `AGENTS.md` 定义的六大工程铁律：

1.  **沙盒隔离**：严禁越出工作目录读写宿主机文件，严禁操作非 `deb-builder` 之外的任何 Docker 容器与镜像。

2.  **隐私保护**：严禁在排障过程中读取宿主机环境变量（`env`）、网络拓扑（`ip`/`ss`）或个人凭证。

3.  **资源受控**：每次任务周期结束后，`deb-builder` 容器必须执行 `docker stop` 熄火停机，平时占用 `0 MB` 内存与 `0%` CPU。

4.  **异常熔断**：遇到未知编译错误、补丁冲突或单线程极限 OOM，Agent 必须立即暂停并输出【问题罗列】、【根因分析】与【可选方案】，严禁私自无痕修改源码或调高硬件配额。

## 许可证（License）

本项目采用 [MIT License](https://www.google.com/search?q=LICENSE) 开源授权。
