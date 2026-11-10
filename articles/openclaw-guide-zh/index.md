# OpenClaw 指南：安装、入门引导、Gateway 与记忆

> 如何安装 OpenClaw、完成入门引导、打开控制界面并连接聊天应用，以及它的记忆存在哪里、Hindsight 插件如何工作。内容来自 OpenClaw README 和官方文档。

Source: https://aiagentmemory.org/articles/openclaw-guide-zh/ · Updated 2026-11-10 · aiagentmemory.org


**OpenClaw** 是一个开源 AI 助手，运行在你自己的电脑上，通过 Telegram、Discord、Slack、WhatsApp 等聊天应用和你交流。一个叫 Gateway 的进程负责运行会话、工具和聊天连接。它的记忆是 agent 工作区里的纯 Markdown 文件。如果需要几个月的历史记录，可以再接入 [Hindsight](https://github.com/vectorize-io/hindsight) 这样的外部记忆插件。

本指南涵盖安装、首次运行、Gateway 和记忆。官方文档在 [docs.openclaw.ai](https://docs.openclaw.ai)。内容核对于 2026 年 11 月 9 日，最新版本为 v2026.11.2（2026 年 11 月 8 日发布）。

## OpenClaw 是什么？

**OpenClaw 是一个你自己托管的个人 AI 助手，没有付费版，也没有托管服务。** 它的 README 说，状态、记忆和凭证都留在你的硬件上。模型和 agent 框架（比如 Claude、Codex 或本地模型）都是插件，可以替换，不需要改动其他设置。

| 组成部分 | 作用 | 运行位置 |
|---|---|---|
| Gateway | 管理会话、工具、事件和聊天连接的控制中心 | 你的机器或服务器 |
| 控制界面、CLI、TUI | 和 Gateway 对话、管理它的方式 | 你的机器 |
| 频道（Channels） | WhatsApp、Telegram、Slack、Discord、Signal、iMessage 等聊天应用 | 连接到 Gateway |
| 应用和节点 | 在支持的平台上提供语音、摄像头、屏幕和设备操作 | 手机、电脑 |

## 如何安装 OpenClaw？

安装脚本支持 macOS、Linux、WSL2 和 Windows。需要时，它会安装受支持的 Node.js 版本。

1. **macOS、Linux 或 WSL2：** `curl -fsSL https://openclaw.ai/install.sh | bash`
2. **Windows PowerShell：** `iwr -useb https://openclaw.ai/install.ps1 | iex`
3. **如果你自己管理 Node.js：** `npm install -g openclaw@latest`。README 要求 24.x 系列使用 Node 24.16.0 或更新版本，或者 26.1.0 或更新版本；为了安全修复，它推荐 24.22.1+ 或 26.12.1+。如果 npm 是 11.15 或更旧的版本，去掉较新 npm 版本使用的 `--allow-scripts=openclaw` 参数。
4. **完成入门引导：** 安装脚本会自动启动它。通过包管理器安装后，运行 `openclaw onboard --install-daemon`。入门引导会检查模型访问、创建工作区并设置 Gateway。
5. **检查并打开界面：** 运行 `openclaw gateway status`，然后运行 `openclaw dashboard`。在控制界面发一条消息，确认助手会回复。

Docker、Nix 和其他安装方式，请看[安装指南](https://docs.openclaw.ai/install)。

## 如何连接聊天应用？

每个聊天服务就是一个频道。按照[频道指南](https://docs.openclaw.ai/channels)里的步骤设置，然后就可以在该应用里给助手发消息。

默认情况下，OpenClaw 不信任你不认识的聊天发送者。在私聊频道里，每个新发送者出现时，OpenClaw 会要求你用 `openclaw pairing approve <channel> <code>` 批准。除非你知道谁能联系到这个机器人，否则请保持这个设置。

除非你开启沙箱，主会话的工具会直接在宿主机上运行。在你把别人接进来或通过网络暴露 Gateway 之前，请先读[安全指南](https://docs.openclaw.ai/gateway/security)。

## OpenClaw 会通过网络发送什么？

你的提示词会发送给你配置的模型提供商和聊天平台。默认情况下，OpenClaw 还会每天检查一次新版本。匿名使用统计默认关闭，需要你主动开启。设置 `update.checkOnStart: false` 可以同时关闭每日检查和统计。README 中有一个页面，列出了具体会发送的内容。

## OpenClaw 的记忆如何工作？

OpenClaw 把记忆保存为可以阅读和编辑的文件。默认工作区是 `~/.openclaw/workspace`。

| 层级 | 保存形式 | 使用方式 |
|---|---|---|
| 长期事实 | `MEMORY.md` | 每次会话开始时加载 |
| 偏好 | `USER.md` | 每次会话开始时加载 |
| 每日笔记 | `memory/` 下按日期命名的文件 | 建立索引，通过 `memory_search` 工具查找 |
| 记忆插件 | 插件的存储，例如 Hindsight | 通过插件自己的工具和设置回忆 |

搜索每日笔记时可以使用嵌入模型。默认提供商是 OpenAI，你可以换成其他提供商或本地模型。如果提供商失败，OpenClaw 会退回到关键词搜索。详细内容请看 [OpenClaw 记忆：文件、搜索与嵌入模型](/articles/openclaw-ai-agent-memory/)（英文）。

OpenClaw 的文档也说明，这些文件里的记忆只是给模型的参考，不是硬性规则。必须成立的规则，请用审批和沙箱设置来保证。

## 如何给 OpenClaw 添加 Hindsight？

[Hindsight](https://github.com/vectorize-io/hindsight) 是一个开源记忆服务器，使用 MIT 许可证。完整的服务器都在开源版本中，包括图召回、整合和 reflect。

1. 安装插件：`openclaw plugins install @vectorize-io/hindsight-openclaw`。
2. 选择它的运行位置：Hindsight 云服务、你自己的 Hindsight 服务器，或本地嵌入式守护进程。嵌入式守护进程可以运行 Ollama 这样的本地模型。
3. 选择 `memoryMode`：`hybrid`（默认）会自动加入记忆上下文，并给 agent 提供工具；`context` 只加入上下文；`tools` 只提供工具。
4. Agent 会获得 `hindsight_retain`、`hindsight_recall` 和 `hindsight_reflect`。从 v1.2.0（2026 年 11 月 3 日）起，它们的名称和参数与 Hermes 插件相同，所以同一个 bank 可以在两个 agent 之间迁移。
5. 默认会跳过定时任务、心跳和子 agent 会话。如果你希望它们也使用记忆，请设置 `skipOperationalSessions: false`。

**Hindsight 适合哪里：** OpenClaw 的文件记忆容易阅读，但搜索效果取决于嵌入设置。当 agent 运行几个月、笔记装不下时，Hindsight 会保存一个更大的存储，agent 需要时再去搜索。在长历史基准 BEAM 上，Hindsight 在 10M token 时得分 64.1，Mem0 为 48.6。这个对比是 Mem0 在它自己的网站上做的，所以请把它当作厂商自己跑出的结果。

**需要注意的限制：**
- 每次保存都会调用一次 LLM，所以写入比文件笔记更贵。
- 每次召回都会增加上下文。默认情况下，Hindsight 每次召回返回的内容比 Mem0 多。如果提示词成本重要，请把 `max_tokens` 调低。
- 保存失败的内容会暂存在内存里，所以如果服务器宕机时进程崩溃，这些内容会丢失。
- OpenClaw 插件的删除工具只有在 `allowKnowledgeDelete: true` 时才能使用。

关于 OpenClaw 和 Hermes 在记忆方面的比较，请看 [OpenClaw 与 Hermes Agent 记忆对比](/articles/openclaw-vs-hermes-agent-memory/)（英文）。Hermes 这一侧，请看 [Hermes Agent 指南](/articles/hermes-agent-zh/)。

## 先检查什么

1. 运行 `openclaw gateway status`，然后在控制界面发一条消息。
2. 只配对你信任的聊天发送者。
3. 时不时看看 `~/.openclaw/workspace` 里的文件。它们是纯 Markdown，所以你可以手动修正错误的笔记。
4. 只有当文件记忆不够用时，再添加 Hindsight。先在你的笔记副本上测试。
5. OpenClaw 和 Hindsight 插件各自按自己的节奏更新。请查看发布说明，因为 2026.10 和 2026.11 版本改动过记忆行为。

常见问题速查：

- **如何安装 OpenClaw？** 运行适合你系统的安装脚本，安装后会启动入门引导。
- **记忆在哪里？** 在 `~/.openclaw/workspace`：`MEMORY.md`、`USER.md`，以及 `memory/` 下的每日笔记。
- **能和 Hindsight 一起用吗？** 可以。安装 `@vectorize-io/hindsight-openclaw`，并选择 Hindsight 服务器或云账号。


## FAQ

### 如何安装 OpenClaw？

在 macOS、Linux 或 WSL2 上运行 curl -fsSL https://openclaw.ai/install.sh | bash。在 Windows 上，运行 openclaw.ai 提供的 PowerShell 安装脚本。安装脚本会启动入门引导。如果你自己管理 Node.js，运行 npm install -g openclaw@latest，然后运行 openclaw onboard --install-daemon。

### OpenClaw 把记忆存在哪里？

存在 agent 工作区里，默认是 ~/.openclaw/workspace。MEMORY.md 和 USER.md 会在每次会话开始时加载，memory/ 下按日期命名的每日笔记则通过 memory_search 工具搜索。除非模型提供商或聊天平台收到你的提示词，所有内容都留在你自己的机器上。

### OpenClaw 能和 Hindsight 一起用吗？

可以。用 openclaw plugins install @vectorize-io/hindsight-openclaw 安装插件。从 Hindsight v1.2.0（2026 年 11 月 3 日）起，它给 OpenClaw 提供和 Hermes 插件相同的 hindsight_retain、hindsight_recall 和 hindsight_reflect 工具。它需要 Hindsight 云服务、你自己的 Hindsight 服务器，或本地嵌入式守护进程。

