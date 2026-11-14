# Hermes Agent 安装指南（2026）：设置、记忆与插件

> 用一条 curl 命令在 Linux、macOS 或 WSL2 上安装 Hermes Agent，运行 hermes setup，然后设置它的内置记忆和 Hindsight 插件。内容来自官方文档，2026-11-10 核对。

Source: https://aiagentmemory.org/articles/hermes-agent-zh/ · Updated 2026-11-14 · aiagentmemory.org


**Hermes Agent** 是 Nous Research 开发的开源 AI Agent。它在终端里运行，也能接入 Telegram、Discord、Slack 等聊天应用。你可以用一条命令切换模型，它还会把过去工作中的技能和笔记保存下来。它的内置记忆是两个很小的文本文件，加上一份可以搜索的历史会话记录。如果需要更大的记忆层，可以再接入外部提供商，比如 [Hindsight](https://github.com/vectorize-io/hindsight)。

本指南介绍安装、首次运行、消息网关和记忆。官方文档在 [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/)。内容于 2026 年 11 月 10 日核对，最新版本为 v0.23.1（2026 年 11 月 10 日发布）。

## 什么是 Hermes Agent？

**Hermes Agent 是一个会自我改进的 Agent，你可以把它运行在自己的电脑或服务器上。** 它有终端界面（`hermes` 命令）、消息网关，以及一套技能系统：完成复杂任务后，它会写出可以复用的技能。它支持很多模型提供商，包括 Nous Portal、OpenRouter、OpenAI 和你自己的接口地址。官方 README 称，你可以"无需改代码、无锁定"地切换模型。

| 组成部分 | 作用 | 运行位置 |
|---|---|---|
| 终端界面 | 用 `hermes` 在终端里聊天 | 你的电脑 |
| 消息网关 | 一个进程，同时服务 Telegram、Discord、Slack、WhatsApp、Signal 和邮件 | 你的电脑或 VPS |
| 终端后端 | 命令在哪里执行：本地、Docker、SSH、Modal、Daytona、Vercel Sandbox | 本地或远程 |
| 桌面应用 | 基于同一份状态的原生应用，见 [Hermes Agent 桌面版](/articles/hermes-agent-desktop/) | 你的电脑 |

## 如何安装 Hermes Agent？

只需运行一条命令。安装程序会装好 Hermes 需要的 Python、Node.js 和其他工具。

1. **Linux、macOS 或 WSL2：** `curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash`
2. **原生 Windows：** 运行安装页面上的 PowerShell 一行命令。文件放在 `%LOCALAPPDATA%\hermes` 下。WSL2 安装使用 `~/.hermes`，和 Linux 相同。
3. **Android（Termux）：** 使用 Termux 指南和 APT 包，不用桌面脚本。
4. **运行设置向导：** `hermes setup` 一次完成所有配置。如果只想改模型，运行 `hermes model`。
5. **开始聊天：** 运行 `hermes`。要使用聊天应用，先运行 `hermes gateway setup`，再运行 `hermes gateway start`，然后给机器人发消息。

安装后常用的命令：`hermes tools`（开启或关闭工具）、`hermes config set` 和 `hermes config get`（读写单个设置），以及 `hermes gateway`（启动消息网关）。

## Hermes 把状态存在哪里？

所有内容都在 `~/.hermes` 下，包括配置文件（`config.yaml`）、API 密钥、技能、记忆文件和会话数据库。在桌面应用里开始的会话，可以在终端里接着继续，因为两者读的是同一个文件夹。

Hermes 还能从其他 Agent 导入内容。运行 `hermes setup` 时，它会查找 `~/.openclaw`、`~/.claude`、`~/.codex` 和 `~/.cursor`。它可以复制设置、记忆和技能，并且会先给你看预览。对 OpenClaw，它还能复制 API 密钥，但只有你同意时才会复制。

## Hermes Agent 的记忆是怎么工作的？

Hermes 有两层内置记忆，以及一层可选的外部记忆。

| 层 | 保存为 | 大小限制 | 使用方式 |
|---|---|---|---|
| Agent 笔记 | `~/.hermes/memories/MEMORY.md` | 2,200 个字符（约 800 个 token） | 每次会话开始时加载进系统提示词 |
| 用户资料 | `~/.hermes/memories/USER.md` | 1,375 个字符（约 500 个 token） | 同上，用来保存你的偏好 |
| 过去的会话 | `~/.hermes/state.db`，全文搜索（FTS5） | 没有固定限制 | Agent 需要历史时，用 `session_search` 搜索 |
| 外部提供商 | 如 Hindsight 等插件 | 由提供商决定 | 在每轮对话之前或期间回忆 |

字符限制是真实存在的。如果一次写入会超过限制，记忆工具会返回错误，Agent 必须先缩短一条内容。记忆不会自己压缩。Hermes 会自动写这些文件，Agent 在工作中也会整理它们。

文档还提醒了一个常见的设置错误：不要让两个 Agent 进程指向同一个 Hermes 主目录。它们的记忆写入会混在一起。如果需要第二个 Agent，请给它单独的 profile，或者使用外部记忆提供商。

关于内置文件的细节和调整方法，见 [Hermes Agent 记忆](/articles/hermes-agent-memory/)。

## 如何为 Hermes Agent 添加 Hindsight？

Hindsight 是一个独立的记忆系统。Hermes 有对应的插件，这个插件在 Hermes 的插件目录里。

1. 安装插件：`hermes plugins install hindsight`。请先把 Hermes 更新到 v0.23.1（2026 年 11 月 10 日）。在 v0.23.0 及更早版本上，插件目录超过了客户端 2 MiB 的上限，所以安装可能会退回到某个插件的旧固定版本，而这个插件是在你上次更新后才新增的。
2. 在 `~/.hermes/hindsight/config.json` 中选择模式：
   - `cloud`（默认）使用 Hindsight 云账号。
   - `local_embedded` 运行一个本地守护进程，带自己的 Postgres。它需要一个 LLM 密钥，闲置 5 分钟后停止。
   - `local_external` 连接你自己运行的 Hindsight 服务器。
3. 选择 `memory_mode`：`hybrid`（默认）自动添加记忆上下文，并提供工具；`context` 只添加上下文；`tools` 只提供工具。
4. Agent 会得到三个工具：`hindsight_retain`（保存）、`hindsight_recall`（搜索）和 `hindsight_reflect`（根据记忆给出综合回答）。
5. 要为每个项目或 profile 设置单独的记忆库，使用 `bank_id_template`，例如加上 `{profile}` 或 `{project}`。
6. 以后更新插件，运行 `hermes plugins update hindsight`。再次运行安装命令并不会更新它。

**Hindsight 适合什么场景：** 内置记忆很小，并且每次都会加载，所以适合放几条关键事实。Hindsight 保存的是更大的记忆库，Agent 按需搜索，适合跨多个项目、时间很长的历史。插件的 `recall_max_tokens` 默认值是 4,096 个 token，所以每次回忆都会增加提示词成本。如果你在意 token 用量，请把这个值调低。

**需要注意的限制：**
- 根据 v1.2.0 插件的发布说明，保存失败后的重试只保存在内存里。如果 Hindsight 服务器宕机，同时 Hermes 崩溃，这些内容会丢失。
- 同一份发布说明指出，看板工作进程（kanban workers）和 `delegate_task` 创建的子 Agent 不会被保存到记忆中。
- Hindsight 不会替代内置文件，两者并行运行。

关于 Hermes 和 OpenClaw 在记忆上的完整对比，见 [OpenClaw 与 Hermes Agent 记忆对比](/articles/openclaw-vs-hermes-agent-memory/)。关于插件，见 [Hermes Agent 插件](/articles/hermes-agent-plugins/)。

## Hermes 和技能、MCP 有什么关系？

技能是可复用的说明。Hermes 可以在完成复杂任务后写出技能，并在之后的使用中不断改进。技能遵循开放的 [agentskills.io](https://agentskills.io) 标准。你可以用 `/skills` 浏览技能，用 `/<技能名>` 运行某个技能。见 [Hermes Agent 技能](/articles/hermes-agent-skills/)。

Hermes 也能连接 MCP 服务器，把外部工具加进 Agent。MCP 的设置方法见官方文档的 MCP Integration 部分。定时任务的设置方式类似：它们按计划运行任务，并把结果发到聊天平台。

## 先检查这几件事

1. 运行 `hermes setup`，用一个简短问题确认模型能正常回答。
2. 用 `hermes tools` 只开启你需要的工具。工具越少，提示词越短。
3. 保持内置记忆开启。只有当你需要一个能跨多个会话搜索的存储时，再添加 Hindsight。
4. 时不时打开 `~/.hermes/memories/MEMORY.md` 看看。它是纯文本，错误的条目可以手动修改。
5. 每次 Hermes 发布新版本后，都检查一下插件，因为插件和 Agent 会一起变化。

常见问题速览：

- **如何安装 Hermes Agent？** 运行适合你系统的安装脚本，然后运行 `hermes setup`。
- **记忆在哪里？** 在 `~/.hermes/memories/` 中，过去的会话在 `~/.hermes/state.db` 中。
- **能和 Hindsight 一起用吗？** 可以，通过 Hindsight 插件实现，需要 Hindsight 云账号或你自己的服务器。


## FAQ

### 如何安装 Hermes Agent？

在 Linux、macOS 或 WSL2 上运行 curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash。原生 Windows 请使用 Hermes 文档中的 PowerShell 一行命令。然后运行 hermes setup 打开完整的设置向导，或运行 hermes model 选择模型提供商。

### Hermes Agent 把记忆存在哪里？

内置记忆是两个纯文本文件：MEMORY.md（Agent 的笔记，约 2,200 个字符）和 USER.md（用户资料，约 1,375 个字符），都在 ~/.hermes/memories/ 下。每一次 CLI 和消息会话也会保存到 ~/.hermes/state.db，并支持全文搜索。

### Hermes Agent 能和 Hindsight 一起用吗？

可以。Hermes 有 Hindsight 插件，用 hermes plugins install hindsight 安装。它会增加 hindsight_retain、hindsight_recall 和 hindsight_reflect 三个工具，并能自动保存和回忆记忆。它需要 Hindsight 云账号，或者你自己运行的 Hindsight 服务器。

