# Hermes Agent Desktop：安装、记忆与设置

> Hermes Agent Desktop 是 Nous Research 为 Hermes 做的原生应用。介绍如何安装、它怎样和命令行共享记忆与会话、Memory Graph 功能以及限制。

Source: https://aiagentmemory.org/articles/hermes-agent-desktop-zh/ · Updated 2026-11-12 · aiagentmemory.org


**Hermes Agent Desktop** 是 Nous Research 的 Hermes Agent 官方原生应用，支持 macOS（Apple Silicon）、Windows 和 Linux。它不是另一个 Agent：底层运行的是和 `hermes` 命令行相同的核心，读取的是同一套配置、会话、技能和记忆。你可以从 Hermes 官网安装，如果已经装了命令行，也可以直接运行 `hermes desktop`。

[Hermes Agent](https://github.com/NousResearch/hermes-agent) 使用 MIT 许可证，2026 年 10 月 24 日在 GitHub 上约有 256,500 颗星。下面的内容都来自官方的 [Hermes Desktop 文档](https://hermes-agent.nousresearch.com/docs/user-guide/desktop) 和 [安装指南](https://hermes-agent.nousresearch.com/docs/getting-started/installation)，2026 年 10 月核对，当时 Hermes 版本为 v0.22.5。

## 什么是 Hermes Agent Desktop？

**Hermes Agent Desktop 是一个 Electron 应用，在标准 Hermes Agent 运行时之上提供一个以聊天为主的窗口。** 用 Nous Research 的话说，它使用的是“与命令行相同的配置、相同的 API 密钥、相同的会话、相同的技能、相同的记忆”，并且“不是一个单独的产品，也不是轻量克隆”。

Nous Research 于 2026 年 6 月 2 日以 v0.15.2 公开预览版的形式发布了它，[Decrypt 报道过此事](https://decrypt.co/369952/hermes-ai-agent-official-app-terminal)。应用默认会启动并管理自己的本地后端。它也可以连接到另一台机器上的 Hermes 后端（比如 VPS、家用服务器，或者通过 Tailscale 连接的 Mac mini），前提是那台机器上已经运行了 `hermes serve`。应用不会替你启动它。

现在 Hermes 有四种前端，共用同一份状态：

| 界面 | 如何打开 | 是否共享记忆和会话 |
|---|---|---|
| 桌面应用 | 安装包，或 `hermes desktop` | 是 |
| 命令行 | `hermes` | 是 |
| TUI | `hermes --tui` | 是 |
| Web 控制台 | `hermes dashboard` | 是 |

## 如何安装 Hermes Agent Desktop

按照安装指南和 `install.sh` 脚本，有三种方式：

1. **安装包。** 从 Hermes 官网下载：macOS 上是 DMG（仅 Apple Silicon），Windows 上是 `.appinstaller`。
2. **在已有的命令行安装上。** 运行 `hermes desktop`。它会安装 Node 依赖，为你的系统构建 Electron 应用，然后启动。第一次运行时会从 GitHub 下载 Electron（约 114 MB）；如果网络被屏蔽，请设置 `ELECTRON_MIRROR`。
3. **作为全新安装的一部分。** 给安装脚本加上桌面参数：

```bash
# macOS / Linux / WSL2
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash -s -- --include-desktop
```

在原生 Windows 上，PowerShell 安装脚本（`iex (irm https://hermes-agent.nousresearch.com/install.ps1)`）接受 `-IncludeDesktop` 参数。

如果用 Ollama，`ollama launch hermes-desktop` 会在缺少 Hermes 时先安装它，让你选一个模型，把 Hermes Desktop 指向该模型，然后打开应用（[Ollama 文档](https://docs.ollama.com/integrations/hermes-desktop)）。加上 `--model <model>` 可以跳过选择模型的步骤。

`hermes desktop` 常用的参数：

| 参数 | 作用 |
|---|---|
| `--skip-build` | 直接启动已经构建好的应用 |
| `--force-build` | 即使没有变化也重新构建 |
| `--build-only` | 只构建不启动（`hermes update` 会用到） |
| `--cwd PATH` | 在指定的项目目录中打开 |
| `--ignore-existing` | 不启动本地后端 |

要卸载它，`hermes uninstall --gui` 只删除应用，`hermes uninstall` 删除应用和 Agent 但保留你的数据，`hermes uninstall --full` 全部删除。加上 `--dry-run` 可以先预览。

## Hermes Desktop 如何使用记忆

应用使用的运行目录和命令行安装相同：`~/.hermes`，Windows 上是 `%LOCALAPPDATA%\hermes`，结构也一样。所以它的记忆就是 Hermes 的普通记忆：`MEMORY.md`（2,200 个字符的 Agent 笔记）和 `USER.md`（1,375 个字符的用户资料），会在会话开始时作为固定快照加载进提示词；过去的会话保存在 SQLite 中，Agent 需要时再搜索。完整的机制、限制和修复方法见 [Hermes Agent 记忆](/articles/hermes-agent-memory-zh/)。

应用额外提供的，是一种查看这些记忆的方式：

- **Memory Graph。** 一个可缩放的节点图，展示 Hermes 学到的技能和记忆，可以按 All、Used 和 Learned 筛选，还带时间线。可以从命令面板或状态栏打开。它的分享按钮会把布局导出为一段代码；文档说，这段代码不包含任何记忆或技能的文字。
- **Learning Journey。** `/journey`（也可以用 `/learning` 或 `/memory-graph`）会按时间回放学到的技能和记忆。你可以在这里编辑或删除节点：删除技能会把它归档，删除记忆则会直接移除。
- **Memory & Context 设置。** 和 Model、Workspace、Safety、Tools & Keys 并列的一个设置页，背后是配置文件。如果有两个或更多配置档案，“Applies to”选择器会决定你正在编辑哪个档案的 `config.yaml`。每个档案都有自己的配置、技能和会话。

由于共用同一份状态，会产生两个结果。在桌面聊天中写入的记忆，下一次终端会话里也能看到，反过来同样成立。另外，固定快照的规则仍然适用：在聊天中途保存的事实，要到下一次会话才会出现在提示词里，所以想让它生效，请新开一个聊天。

## 添加外部记忆提供商

桌面文档没有介绍记忆提供商。因为应用读取的是和命令行相同的 `config.yaml` 和插件，所以你在终端里配置的提供商，应用也会使用：

```bash
hermes plugins install hindsight   # 也可以是 honcho、mem0、supermemory 或 openviking
hermes memory setup                # 选择提供商，填入凭据
hermes memory status               # 检查当前启用的是哪一个
```

除了 `MEMORY.md` 和 `USER.md` 之外，同一时间只运行一个外部提供商。[Hermes Agent 插件](/articles/hermes-agent-plugins-zh/)页面介绍了它们如何安装和更新。

[Hindsight](https://github.com/vectorize-io/hindsight) 是列出的提供商之一，适合你会长期使用的桌面 Agent。在 Mem0 自己的对比页面上，Hindsight 在长历史的 BEAM 测试中领先：在 1M tokens 时是 73.9 对 64.1，在 10M tokens 时是 64.1 对 48.6（这些是厂商自己跑出的数字，不是独立测试）。在 LongMemEval 上两者持平（94.6 对 94.4）。Hindsight 使用 MIT 许可证，自托管服务器包含全部功能；它的 Hermes 插件有一个 `local_embedded` 模式，会运行带内置 Postgres 的本地守护进程，所以记忆留在你的机器上。通过 `bank_id_template`，你可以为每个 Hermes 档案分配独立的记忆库。限制也要说清楚：每次写入都会调用一次 LLM；插件默认每轮最多注入 4,096 个 tokens 的回忆内容（`recall_max_tokens`），每条消息都要为此付费。如果内置文件已经够用，就不必接入提供商。关于取舍的详细说明，见 [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/)。

## 官方应用与社区版 Hermes Desktop

在网上搜索 “Hermes Desktop” 还会找到 [fathah/hermes-desktop](https://github.com/fathah/hermes-desktop)，它自称是 “Desktop Companion for Hermes Agent”。这是一个独立的社区项目，不是 Nous Research 做的。

| | 官方 Hermes Desktop | fathah/hermes-desktop |
|---|---|---|
| 开发者 | Nous Research | 社区（fathah） |
| 代码位置 | NousResearch/hermes-agent 中的 `apps/desktop` | 独立的仓库 |
| 许可证 | MIT | MIT |
| 星标数（2026 年 10 月 24 日） | 属于 hermes-agent 的一部分（约 256,500） | 约 14,400 |

如果你想要一个跟随 Hermes 发布和上面这些文档的应用，请选官方版本。

## 已知限制

桌面文档列出了以下几点：

- **桌面插件没有沙箱。** `$HERMES_HOME/desktop-plugins/` 中的任何东西都以应用的全部权限运行，所以只安装你信任的插件。记忆本身也是一个攻击面，见 [AI 记忆注入](/articles/ai-memory-injection/)。
- **在聊天中途切换模型会重置提示词缓存**，所以下一轮会更贵。
- **检查更新受速率限制**：没有令牌时，每小时最多 60 次 GitHub 请求；设置 `GITHUB_TOKEN` 或用 `gh` 登录后可达 5,000 次。
- **Wayland 的特殊问题**：在一些原生 Wayland 合成器上，窗口置顶和吸附到指针的功能无法使用。
- **macOS 版本只支持 Apple Silicon。**

日志保存在 `~/.hermes/logs/desktop.log`；`hermes logs gui -f` 可以实时跟踪。关于 Hermes 与 OpenClaw 在记忆上的比较，见 [OpenClaw vs Hermes Agent 记忆](/articles/openclaw-vs-hermes-agent-memory-zh/)。


## FAQ

### 如何安装 Hermes Agent Desktop？

从 Hermes 官网下载 DMG（仅 macOS Apple Silicon）或 .appinstaller（Windows）。如果已经装了 Hermes 命令行，运行 hermes desktop，它会构建并启动应用。全新安装时可以用 curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash -s -- --include-desktop 一并安装。

### Hermes Desktop 和 Hermes 命令行共享记忆吗？

共享。Nous Research 的文档说，桌面应用和命令行、网关运行的是同一个 Agent，使用同样的配置、API 密钥、会话、技能和记忆，都在 ~/.hermes 下（Windows 上是 %LOCALAPPDATA%\hermes）。在应用里开始的会话可以在终端里接着用，反过来也一样。

### Hermes Desktop 和 fathah/hermes-desktop 是同一个东西吗？

不是。官方应用由 Nous Research 在 hermes-agent 仓库里开发，2026 年 6 月 2 日发布。fathah/hermes-desktop 是另一个独立的、MIT 许可的社区伴侣应用，2026 年 10 月在 GitHub 上约有 14,400 颗星。

