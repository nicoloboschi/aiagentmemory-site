# 如何安装 Claude Code（2026）：macOS、Windows、Linux

> 如何在 macOS、Windows、WSL 和 Linux 上安装 Claude Code：原生安装器、Homebrew、WinGet 或 npm，如何检查安装、登录、更新和卸载。命令来自官方安装文档，读取日期 2026-11-12。

Source: https://aiagentmemory.org/articles/install-claude-code-zh/ · Updated 2026-11-12 · aiagentmemory.org


## 什么是 Claude Code？

**Claude Code** 是 Anthropic 的编程 agent，在终端里运行。它读取你的项目、编辑文件并运行命令。你在电脑上安装一次，然后在任何项目文件夹里运行 `claude` 启动它。它的记忆来自 CLAUDE.md 等文件，以及可选的插件。

## 如何安装 Claude Code？

打开终端，运行适合你系统的命令。Anthropic 推荐的是原生安装器，它会自己保持最新。

| 系统 | 命令 | 更新方式 |
|---|---|---|
| macOS、Linux、WSL | `curl -fsSL https://claude.ai/install.sh \| bash` | 自动，在后台进行 |
| Windows PowerShell | `irm https://claude.ai/install.ps1 \| iex` | 自动，在后台进行 |
| Windows CMD | `curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd` | 自动，在后台进行 |
| Homebrew（macOS、Linux） | `brew install --cask claude-code` | 手动：`brew upgrade claude-code` |
| WinGet（Windows） | `winget install Anthropic.ClaudeCode` | 手动：`winget upgrade Anthropic.ClaudeCode` |
| npm（任何装有 Node.js 22+ 的系统） | `npm install -g @anthropic-ai/claude-code` | 手动：`npm install -g @anthropic-ai/claude-code@latest` |

Anthropic 的安装文档还列出了 Debian、Ubuntu、Fedora、RHEL 和 Alpine 的 apt、dnf 和 apk 软件源。这些安装同样需要手动升级。npm 安装不要加 `sudo`。

原生安装器不需要管理员权限，在 Windows 上也不需要以管理员身份运行。npm 是最容易出现权限错误的方式，所以只有在其他方式都失败时才用它。

## 安装前需要什么？

以下是官方要求，来自[高级安装文档](https://code.claude.com/docs/en/setup)：

- **操作系统：** macOS 13.0 或更高版本，Windows 10（1809）或更高版本，Ubuntu 20.04 或更高版本，Debian 10 或更高版本，或 Alpine Linux 3.19 或更高版本。
- **硬件：** 4 GB 或更多内存，x64 或 ARM64 处理器。
- **网络：** 需要联网。
- **Git：** 可选。在原生 Windows 上，安装 [Git for Windows](https://git-scm.com/downloads/win) 后 Claude Code 才能使用 Bash 工具。没有它时，Claude Code 改用 PowerShell。
- **账户：** Pro、Max、Ultra、Team、Enterprise 或 Console 套餐。免费套餐不能使用 Claude Code。

在 Windows 上，你可以原生运行 Claude Code，也可以在 WSL 2 里运行。WSL 2 支持沙箱化的命令执行，原生 Windows 不支持。

## 如何检查安装是否成功？

打开一个新的终端，因为安装器会把 Claude Code 加到你的 PATH 里。然后运行：

```bash
claude --version
```

安装成功时会显示版本号，例如 `2.1.211 (Claude Code)`。如果看到 `command not found`，说明安装目录还没有加入 PATH。要做更深入的检查，运行 `claude doctor`。它只读取诊断信息，包括安装状态、设置错误和更新状态，而且不会启动会话。

## 如何登录并开始使用？

在项目文件夹里运行 `claude`。第一次运行时，Claude Code 会打开浏览器登录。如果你设置了 `ANTHROPIC_API_KEY` 环境变量，并在提示时确认该密钥，就会跳过登录。

## 如何更新 Claude Code？

运行 `claude update` 立即更新。原生安装在启动时会检查更新，并在后台安装。新版本在你下次打开 Claude Code 时生效。

你可以用 `autoUpdatesChannel` 设置选择发布通道。默认是 `"latest"`。`"stable"` 提供的版本通常晚一周左右，并跳过有重大回归问题的版本。要关闭自动更新，在设置文件的 `env` 里把 `DISABLE_AUTOUPDATER` 设为 `"1"`。

## 如何卸载 Claude Code？

macOS 或 Linux 上的原生安装，删除二进制文件和版本文件：

```bash
rm -f ~/.local/bin/claude
rm -rf ~/.local/share/claude
```

Homebrew、WinGet、npm 或 Linux 包管理器的卸载命令请使用对应的方法。删除 `~/.claude` 和 `~/.claude.json` 会同时删除你的设置、MCP 服务器配置和会话历史，所以先备份。如果删除后 `claude` 仍能运行，检查是否还有第二个安装。

## 安装后，记忆在哪里？

Claude Code 自带的记忆不需要安装。把构建命令和规则写进 CLAUDE.md，Claude Code 还会保存简短的自动记忆笔记。两者的说明见 [Claude Code 记忆](/articles/claude-code-memory/)，保存记忆的 hook 事件见 [Claude Code hooks](/articles/claude-code-hooks-memory/)。

内置记忆只留在一台机器上，不会搜索过去的会话。附加插件可以补上这一点。Hindsight 是 Vectorize 开发的开源记忆服务器，也是其中一个选择：

- **安装方式。** Claude Code 的 Hindsight 插件已经退役。它的替代品是 npm 包 `@vectorize-io/hindsight-coding-agents`。在你的 repo 文件夹里运行 `npx @vectorize-io/hindsight-coding-agents install claude-code`。它的 hooks 会记录每个会话，Claude Code 还会获得用于搜索的 `hindsight_*` 工具。记录是自动完成的，没有单独的设置命令。
- **表现如何。** 在 1000 万 token 规模的 BEAM 基准上，Hindsight 得分 64.1，Mem0 平台得分 48.6。这些数字来自 [Mem0 自己的对比页面](https://mem0.ai/compare/mem0-vs-hindsight)，所以是厂商自己跑的结果。该服务器是 MIT 许可证，完整功能可以在你自己的机器上运行。
- **限制。** 默认服务器是 Hindsight Cloud。本地守护进程需要 LLM 密钥来提取事实。如果没有设置密钥，安装器会退回到 Claude Code CLI，它不需要密钥。每次写入都会调用 LLM，比普通日志的成本更高。召回返回的上下文比简单存储多，所以要设置 token 上限，让提示保持小。

三种 Claude Code 记忆方案的并排对比，见[内置记忆 vs claude-mem vs Hindsight](/articles/claude-code-memory/#built-in-memory-vs-claude-mem-vs-hindsight)。通过 MCP 接入 Claude Code 的其他记忆服务器，见 [AI 记忆 MCP 服务器](/articles/ai-memory-mcp-server/)。OpenAI 终端 agent 的同样步骤，见 [如何安装 Codex CLI](/articles/codex-cli-install/)。

来源：[Claude Code 安装文档](https://code.claude.com/docs/en/setup)，读取日期 2026-11-12。[Hindsight Coding Agents 文档](https://hindsight.vectorize.io/sdks/integrations/coding-agents)，读取日期 2026-11-12。[GitHub 上的 Hindsight](https://github.com/vectorize-io/hindsight)。


## FAQ

### 安装 Claude Code 的命令是什么？

在 macOS、Linux 或 WSL 上运行 curl -fsSL https://claude.ai/install.sh | bash。在 Windows PowerShell 上运行 irm https://claude.ai/install.ps1 | iex。然后打开新的终端，运行 claude --version 检查是否安装成功。这些命令来自 Anthropic 的安装文档，读取日期 2026-11-12。

### 使用 Claude Code 需要付费套餐吗？

需要。文档说 Claude Code 需要 Pro、Max、Ultra、Team、Enterprise 或 Console 账户。免费的 claude.ai 套餐不包含 Claude Code。你也可以通过 Amazon Bedrock、Google Cloud 或 Microsoft Foundry 等第三方 API 服务商使用它。

### 如何更新 Claude Code？

原生安装会在后台自动更新。Homebrew、WinGet、npm 和 Linux 包管理器安装需要手动更新。运行 claude update 立即更新，或者分别运行 brew upgrade claude-code、winget upgrade Anthropic.ClaudeCode，或 npm install -g @anthropic-ai/claude-code@latest。

