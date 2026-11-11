# Hermes Agent 插件：安装、目录与记忆提供方

> Hermes Agent 插件如何工作：插件类型、hermes plugins install、固定 SHA 的目录、记忆提供方插件，以及 v0.23.0（2026 年 11 月）的变化。

Source: https://aiagentmemory.org/articles/hermes-agent-plugins-zh/ · Updated 2026-11-11 · aiagentmemory.org


**Hermes Agent 插件**是包含 `plugin.yaml` 清单的 Python 代码文件夹。它们可以添加工具、钩子、斜杠命令、记忆后端等功能，而不需要改动 Hermes 的核心。你可以用 `hermes plugins install <名称>` 从经人工审核、固定提交的目录中安装，也可以从任意 Git 仓库安装。记忆提供方插件比较特殊：同一时间只有一个在运行。

[Hermes Agent](https://github.com/NousResearch/hermes-agent) 是 Nous Research 推出的开源（MIT）agent，截至 2026 年 10 月 21 日在 GitHub 上约有 256,000 颗星。本页介绍插件系统：插件类型、安装命令、目录、记忆提供方如何接入，以及有哪些提供方。Hermes 内置记忆如何工作（MEMORY.md、USER.md、会话搜索）见 [Hermes Agent 记忆](/articles/hermes-agent-memory/)。事实来自 [Hermes 插件文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) 以及仓库中的 `plugin-catalog/` 文件夹，2026 年 10 月 21 日核对。

## 什么是 Hermes Agent 插件？

**Hermes 插件是一个目录，里面有 `plugin.yaml` 清单和一个定义了 `register(ctx)` 的 `__init__.py`。Hermes 启动时会调用这个函数，插件通过 `ctx` 注册模型可以调用的工具、生命周期钩子、斜杠命令、CLI 子命令、随附的技能，或者像记忆存储这样的后端。**

用户插件放在 `~/.hermes/plugins/<名称>/`。Hermes 还会在仓库的 `plugins/` 文件夹中发现内置插件、暴露 `hermes_agent.plugins` 入口点的 pip 包，以及 NixOS 模块配置中声明的插件。项目本地的 `./.hermes/plugins/` 文件夹不会被加载：文档说，运行每个会话的网关每个进程只发现一次插件，而不是每个工作目录发现一次。

第三方代码默认关闭。能发现插件，但在它的名称写入 `~/.hermes/config.yaml` 的 `plugins.enabled` 之前，带有工具或钩子的插件不会加载。文档说这样能“防止第三方代码在你未明确同意的情况下运行”。

## Hermes 插件的类型

[Hermes 插件文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) 列出了四种类型：

| 类型 | 作用 | 同时运行几个 | 如何开启 |
|---|---|---|---|
| **通用插件** | 添加工具、钩子、斜杠命令、CLI 命令、技能 | 任意组合 | `plugins.enabled` / `hermes plugins enable` |
| **记忆提供方** | 在 MEMORY.md 和 USER.md 之外添加一个外部记忆存储 | 一个 | `config.yaml` 中的 `memory.provider` |
| **上下文引擎** | 替换内置的上下文压缩器 | 一个 | `config.yaml` 中的 `context.engine` |
| **模型提供方** | 声明一个推理后端（OpenRouter、Anthropic 等） | 可加载多个，你选一个 | `--provider` 或 `config.yaml` |

在通用插件中，`ctx` 还有更多入口：网关平台（Discord、Telegram、IRC）、图像和视频生成后端、终端沙箱、computer-use 驱动、审批通道、TTS 和 STT 后端，以及借用用户当前模型的 `ctx.llm.complete()`。有些扩展根本不是 Python 插件：MCP 服务器在 `config.yaml` 的 `mcp_servers` 下声明，shell 钩子在 `hooks:` 下声明。

钩子是通用插件响应 agent 的主要方式。文档列出的钩子有 `pre_tool_call`、`post_tool_call`、`pre_llm_call`、`post_llm_call`、`on_session_start`、`on_session_end` 和 `post_compaction`。钩子回调如果阻塞超过 `plugins.hook_callback_timeout`（默认 30 秒），就会被放弃；超时的 `pre_tool_call` 会阻止这次工具调用。从 v0.22.6 起，`pre_llm_call` 钩子还能阻止整个回合（[见下文](#what-changed-for-plugins-in-hermes-v0226)）。

## 如何安装 Hermes Agent 插件

按文档，常用流程如下：

1. **查找。** `hermes plugins search <关键词>` 按名称、描述和声明的工具搜索目录。`hermes plugins browse` 列出所有条目，`hermes plugins info <名称>` 显示单个条目。
2. **安装。** `hermes plugins install <名称>` 会以审核过的提交克隆该条目的仓库。安装前，提示会先显示声明的工具、钩子和环境变量。
3. **回答启用提示。** Hermes 会问 `Enable 'name' now? [y/N]`（默认是否）。脚本中可用 `--enable` 或 `--no-enable`。如果是记忆提供方，它会改问 `Use 'name' as the memory provider now?`。
4. **需要时授予权限。** 声明了特权能力（例如 `tools.override`）的插件会弹出一次性的同意提示。授权保存在 `plugins.entries.<id>.granted_capabilities` 下。没有终端时，权限保持未授予。Python 依赖有自己的同意步骤，`--enable` 也跳不过：拒绝或在非交互模式下安装依赖，会让插件保持已安装但未启用。根据发布说明，Hermes v0.22.4（2026 年 10 月 21 日发布）对不安装依赖的插件跳过了这一步。
5. **重启或开启新会话。** 在会话中运行 `/plugins` 查看已加载的内容，或运行 `hermes plugins list` 查看已启用、已禁用和未启用的状态。
6. **有意识地更新。** `hermes plugins update <名称>` 会把目录安装的插件移到目录当前的固定版本；不带名称时（v0.22.3 起）会列出所有可更新的插件并询问 “Update all N?”（`--yes` 跳过询问）。`hermes plugins check-updates` 只报告，不更新。从 v0.22.6 起，需要更新 Hermes 才能使用的更新会被跳过，旧的固定版本保留。
7. **移除。** `hermes plugins remove <名称>` 卸载插件，把它从 `plugins.enabled` 和 `plugins.disabled` 中删除，如果它是当前的记忆提供方，还会重置 `memory.provider`。

完整命令如下：

```bash
hermes plugins                          # 交互式开关界面
hermes plugins list                     # 所有插件及其状态
hermes plugins search memory            # 搜索目录
hermes plugins install <名称>           # 目录条目，使用固定的 SHA
hermes plugins install owner/repo       # 任意 Git 仓库（未审核）
hermes plugins install owner/repo --ref <40 位 SHA>   # 固定版本的自定义安装
hermes plugins enable <名称>
hermes plugins disable <名称>
hermes plugins update [<名称>]
hermes plugins capabilities [<名称>]    # 声明的与已授予的
hermes plugins remove <名称>
```

每次安装和更新都会先运行静态安全扫描。`safe` 结论直接安装，`caution` 会询问 `Install anyway? [y/N]`，`dangerous` 即使加 `--force` 也会被阻止。以审核固定版本进行的目录安装会跳过 `caution` 询问，因为审核者已经看过这些结果。配置位于 `~/.hermes/config.yaml` 的 `plugins:` 下（`enabled`、`disabled`、`clone_timeout_seconds`、`hook_callback_timeout`、`load_timeout_seconds`）；安装来源记录在 `~/.hermes/plugins/.install-metadata.json`。

## Hermes 插件目录

目录是 hermes-agent 仓库中的一组 YAML 文件，每个插件一个，并发布为一个 JSON 数据源，客户端最多每六小时刷新一次。2026 年 10 月 21 日，在线数据源列出了 754 个条目，数据来自 Hermes 文档站的 `plugin-catalog.json`。按分类看，有 241 个工具、174 个桌面、67 个记忆、66 个模型、52 个通用、48 个自动化、42 个网络、41 个平台、21 个语音，另有两个未分类。

[目录 README](https://github.com/NousResearch/hermes-agent/tree/main/plugin-catalog) 中的规则，是让一个裸名称可以安全安装的原因：

- **由人合并。** 条目只能通过维护者审核的 PR 加入。没有自助注册表。
- **精确的 SHA 固定。** 每个条目固定一个完整的 40 位提交。新的固定版本就是一个新的已审核 PR。
- **没有自我更新的代码**，不在运行时修补 Hermes 核心，也没有审核者读不懂的编译二进制。
- **声明的能力必须与** 固定提交中注册的内容一致。
- **移除名单。** 被下架的插件列在 `removed.yaml` 中，安装器会按名称或仓库 URL 拒绝它们。

每个条目都有一个级别。`official` 表示由 Nous Research 维护，`verified` 表示 Nous 确认其来源是它所集成产品的公司，`community` 是其他所有条目。2026 年 10 月 21 日，记忆分类的 67 个条目全部是 `community` 级别，包括 Honcho、Mem0、Supermemory 和 Hindsight，尽管这些项目由各自的公司维护。所以级别无法区分记忆提供方之间的差别。

Hermes 也在把自己的功能从核心移到 `official` 插件中。在 [v0.22.5](https://github.com/NousResearch/hermes-agent/releases/tag/v0.22.5)（2026 年 10 月 22 日）中，`hermes import-agent` 和 OpenClaw 迁移变成了 `agent-importers` 插件，`hermes mcp serve` 也离开了核心。运行没有对应插件的旧命令时，Hermes 会打印安装命令（`hermes plugins install agent-importers`）。导入器是记忆从 OpenClaw、Claude Code、Codex CLI 或 Cursor 迁入的方式；[OpenClaw 与 Hermes Agent 的记忆对比](/articles/openclaw-vs-hermes-agent-memory/#migrating-between-them) 说明了它对每个文件的处理。

文档直言目录不是什么：“Cataloged ≠ audited”（收录不等于审计）。审核只检查固定的提交和收录时声明的能力，不是安全审计。所以对于要交出凭据的插件，请阅读它的代码。不在目录中的裸名称会报错；`owner/repo` 安装会跳过目录，并显示未审核代码的警告。

## Hermes v0.23.0 中插件的变化

[Hermes v0.23.0](https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0)（2026 年 11 月 5 日）是 v0.22.x 补丁之上的完整版本，因此下面的 v0.22.6 变化也包含在内。插件方面的变化来自发布说明和 PR 内容：

| 变化 | v0.23.0 中的情况 | 来源 |
|---|---|---|
| 目录规模 | v0.23.0 标签时有 1,050 个条目（2026 年 10 月 21 日为 754）。按分类：389 个工具、225 个桌面、96 个模型、79 个记忆、65 个通用、62 个自动化、53 个平台、52 个网络、27 个语音、2 个未分类 | 2026 年 11 月 6 日，我们对该标签上 `plugin-catalog/*.yaml` 的统计（不含 `removed.yaml`） |
| 级别 | 1,019 个 `community`、21 个 `official`、9 个 `verified`、1 个无级别。79 个记忆分类条目仍全部是 `community`，包括 Honcho、Mem0、Supermemory 和 Hindsight | 同一统计 |
| 全部更新 | `hermes plugins update` 不带名称时只询问一次，并更新全部；一个失败不影响其余。没有终端时，除非加 `--yes`，否则什么都不更新；加了 `--yes` 后，新能力保持未授予 | [#141513](https://github.com/NousResearch/hermes-agent/pull/141513) |
| 自动更新 | `plugins.auto_apply`（默认关闭），也可用 `/plugins auto-update`，并在桌面版和仪表板中有开关；目录安装的插件也会更新 | 发布说明，[#85794](https://github.com/NousResearch/hermes-agent/pull/85794) |
| 新的插件 API | `ctx.post_message` 向当前聊天发送一条消息，并能原地编辑它；`ctx.side_question` 让你允许的插件在一个回合后向会话自己的 agent 提一个文本问题（同一份记忆，完整对话） | [#141005](https://github.com/NousResearch/hermes-agent/pull/141005)、[#143791](https://github.com/NousResearch/hermes-agent/pull/143791) |
| 新的钩子 | `post_compaction`（每次压缩完成时触发一次，附带压缩前后的 token 估算）和 `on_session_delete`（删除、归档、清理；不保证恰好一次） | [#141017](https://github.com/NousResearch/hermes-agent/pull/141017)、[#140995](https://github.com/NousResearch/hermes-agent/pull/140995) |
| 不兼容的改名 | `hermes mcp serve` 改为 `hermes mcp-serve`；启动 Hermes 的 MCP 客户端必须修改参数，Hermes 不会自动改写 | 发布说明 |

对记忆提供方来说，这两个钩子补上了下文 `MemoryProvider` 接口中的空缺。提供方原本已经有 `on_pre_compress`，现在还可以在压缩删除内容后重新注入，并在你删除会话时清理自己的副本。`ctx.side_question` 在允许之前值得仔细考虑：回答来自一个加载了你记忆的 agent，所以拥有这项授权的插件可以问到 agent 知道的任何内容。本周期目录中的新记忆提供方有 Mnemoverse、post-cortex、Membase、conxt、Inspeximus 和 nyx。

## Hermes v0.22.6 中插件的变化

[Hermes v0.22.6](https://github.com/NousResearch/hermes-agent/releases/tag/v0.22.6)（2026 年 11 月 2 日）是一个补丁版本，汇总了数百个已合并的 PR；v0.23.0 后来对它做了完整记录。以下三项插件变化在 v0.22.6 标签中（已对照合并提交核对）：

| 变化 | 之前 | v0.22.6 中 | PR |
|---|---|---|---|
| 目录中的 `requires_hermes` 下限 | 只显示；安装和更新可能固定到需要更新版 Hermes 的提交 | 安装被拒绝，并提示“`<插件>` 需要 Hermes `<版本>`；当前是 `<版本>`”，同时给出适合你安装方式的更新命令。`hermes plugins update` 保留已安装的固定版本，并把插件列为“更新 Hermes 后可更新”。`search` 仍会显示它，并带有 `needs Hermes` 一行 | [#143468](https://github.com/NousResearch/hermes-agent/pull/143468) |
| `pre_llm_call` 可以阻止回合 | 插件或 shell 钩子只能添加上下文，不能停止回合 | 返回 `{"action": "block", "message": "..."}`（shell 钩子：`{"decision": "block", "reason": "..."}`），回合会在任何模型调用之前结束，用户会看到该消息。存储的提示词会变成被扣留的占位符，因此压缩、`session_search`、后台复盘，以及记忆提供方的 `on_session_end` / `on_pre_compress` 都不会看到它 | [#81434](https://github.com/NousResearch/hermes-agent/pull/81434) |
| `hermes update` 会关闭插件 | 如果该插件是记忆提供方，`memory.provider` 会被清空，原有选择丢失 | 保留 `memory.provider`，变更记录在 `plugins.disabled_by_update` 下；之后的更新在插件符合要求时会重新开启它。`hermes memory status` 和 `hermes doctor` 现在会说明插件已关闭，而不是显示“All checks passed” | [#144257](https://github.com/NousResearch/hermes-agent/pull/144257) |

阻止指令对记忆很重要：数据防泄漏插件现在可以阻止包含密钥的提示词，使其既不会进入模型，也不会进入外部记忆存储，而不只是阻止当前这一轮的回复。该 PR 用真实插件做的测试显示，被阻止的回合发出了 0 次 API 调用。

## 记忆提供方插件如何接入

**记忆提供方插件继承 `agent/memory_provider.py` 中的 `MemoryProvider`，并通过 `ctx.register_memory_provider()` 注册。** 它与内置的 MEMORY.md 和 USER.md 并行运行，从不取代它们。同一时间只有一个处于激活状态，由 `memory.provider` 选择；开发者指南说，第二次注册会被拒绝并给出警告。[记忆提供方页面](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) 说明了激活后提供方的作用。

提供方激活后，Hermes 会把它的上下文注入系统提示词，在每个回合前预取记忆，在回复后同步每个回合，在支持的情况下于会话结束时提取记忆，把内置记忆的写入同步到它那里，并添加提供方自己的工具。[提供方开发者指南](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) 列出了每一步背后的方法：

| 方法 | Hermes 何时调用 | 是否必需 |
|---|---|---|
| `name`、`is_available()`、`initialize(session_id, **kwargs)` | 启动时；`is_available()` 不得发起网络调用 | 是 |
| `get_tool_schemas()`、`handle_tool_call()` | 注入工具时，以及 agent 使用工具时 | 是（有工具则需要处理函数） |
| `get_config_schema()`、`save_config()` | `hermes memory setup` | 是（除非仅用环境变量，则需保存） |
| `system_prompt_block()` | 组装系统提示词时 | 可选 |
| `prefetch(query)` | 每次 API 调用之前；返回召回的上下文 | 可选 |
| `queue_prefetch(query)` | 每个回合之后，为下一轮预热 | 可选 |
| `sync_turn(user, assistant)` | 每个完成的回合之后；不得阻塞 | 可选 |
| `on_session_end(messages)` | 对话结束 | 可选 |
| `on_pre_compress(messages)` | 上下文压缩之前 | 可选 |
| `on_memory_write(action, target, content)` | 内置 MEMORY.md / USER.md 的写入 | 可选 |
| `shutdown()` | 进程退出 | 可选 |

有两个细节容易出错。第一，`hermes plugins enable <提供方>` 不会激活记忆提供方；只有 `memory.provider` 才能激活，而 `hermes memory setup` 会写入它。第二，记忆提供方处理名称冲突的方式与通用插件相反：内置的那个优先，所以自制提供方需要一个唯一的名称。Hermes 交给提供方的所有内容，都会先经过与聊天输出相同的密钥清理，因此工具输出中回显的密钥在提供方保存之前就已被遮蔽。

## Hermes Agent 的记忆提供方插件

Hermes 文档记录了九个外部提供方。其中六个可以从目录安装，其余不行：

| 提供方 | 安装方式 | 数据存放位置 | 文档或目录条目的说明 |
|---|---|---|---|
| **Honcho**（Plastic Labs） | `hermes plugins install honcho` | Honcho Cloud 或自托管 | 用户建模，带辩证推理；5 个工具；需要 `HONCHO_API_KEY` |
| **OpenViking**（火山引擎） | `hermes plugins install openviking` | 本地、托管云或你的服务器 | 记录对话，召回事实，搜索你的知识；6 个工具 |
| **Mem0** | `hermes plugins install mem0` | Platform、自托管或本地开源版 | 服务端 LLM 事实提取、语义搜索、去重、可选的重排序；4 个工具 |
| **Hindsight**（Vectorize） | `hermes plugins install hindsight` | Hindsight Cloud、本地内嵌 Postgres，或你自己的服务器 | 知识图谱、实体解析、多策略检索；retain、recall 和 reflect 工具 |
| **Holographic** | `hermes plugins install holographic` | 本地 SQLite | FTS5 事实存储，带信任分数和 HRR 召回；除非开启可选嵌入，否则不联网 |
| **Supermemory** | `hermes plugins install supermemory` | Supermemory Cloud 或自托管 | 用户画像召回、语义搜索、每回合记录；4 个工具 |
| **RetainDB** | `hermes memory setup` | RetainDB Cloud（列出价格 每月 $20） | 向量加 BM25 混合搜索；10 个工具 |
| **ByteRover** | 先装 `brv` CLI，再运行 `hermes memory setup` | 本地，可选云同步 | 带分层检索的知识树；3 个工具 |
| **Memori** | 外部 `hermes-memori` CLI | Memori Cloud | 结构化召回；5 个工具 |

Honcho、Mem0、Supermemory、OpenViking、Hindsight 和 Holographic 原来都内置在 Hermes 中。如果 `memory.provider` 已经指定了其中一个，`hermes update` 会安装对应的目录插件，并保留原有设置和数据。文档说，RetainDB 和 ByteRover 将于 2026 年 10 月 15 日离开核心，其仓库已“不再维护，正在寻找新维护者”；2026 年 10 月 21 日，它们的文件夹仍在仓库的 `plugins/memory/` 中。**Mnemosyne**（`hermes plugins install mnemosyne`）是另一个本地选项，基于 SQLite，采用向量加 FTS5 混合召回，其目录条目声明了 41 个工具；见 [Mnemosyne 与 Hindsight 对比](/articles/mnemosyne-vs-hindsight/)。Holographic 的设计在 [AI agent 的全息记忆](/articles/holographic-memory-ai-agent/) 中有说明，Honcho 在 [Honcho LLM 记忆](/articles/honcho-llm-memory/) 中有说明。

### 记忆分类中的其他条目

其余 67 项中（2026 年 10 月 21 日），每项用目录描述的一行概括。并非每一项都是 `memory.provider`：少数是钩子、上下文引擎或 MCP 包，已在表中标明。

| 条目 | 目录条目的说明 |
|---|---|
| agent_memory_es | 基于 Elasticsearch 服务的召回、保存和反思工具，混合 BM25 + kNN |
| agentplaybooks-memory | 召回，并把记忆写入同步到 agentplaybooks.ai |
| betterdb | 基于 Valkey / valkey-search 的语义召回 |
| bilinc | 托管记忆，完整记录发生了什么变化以及原因 |
| birkin-mnemosyne | 本地 Markdown 记忆，支持多语言关键词召回 |
| brethof-brain | 会话简报，每回合召回，多个 agent 共享一份记忆 |
| cashew | 思维图记忆，使用本地嵌入和衰减 |
| chronicle | 本地 SQLite 记忆，并带有记忆感知的上下文压缩 |
| cognition | 会话结束时用正则提取事实，存入本地 SQLite |
| cognee | 本地 cognee 服务器或 Cognee Cloud 上的知识图谱记忆 |
| compartment | 加密的离线保险库，附带内置嵌入模型 |
| cortexlayer | 页面链接式记忆，带链接扩展检索 |
| deja-memory | 从机器上已有的编码 agent 会话构建的记忆 |
| dexio | 你的团队和 agent 都能编辑的共享维基记忆 |
| entropicmem | SQLite FTS5 事实，加上带图谱导出的 Markdown 库 |
| excel_line | 基于 Excel 的分层记忆树 |
| gbrain-plugin | 通过 MCP 使用 GBrain，作为本地记忆层（有状态命令和安装脚本） |
| gbrain-pointer | 只读提供方，搜索 GBrain MCP 服务器 |
| gbrain-retrieval-reflex | 注入 GBrain 上下文的 `pre_llm_call` 钩子；不是 `memory.provider` |
| hermes-dreaming | 每晚整理，提议值得保留的事实 |
| hermes-lcm-x | 无损会话存储；是上下文引擎（`context.engine: lcm-x`） |
| hermes-pubky | 安装辅助工具，把 agent 状态同步到 Pubky homeserver |
| hermes-snapcompact | 上下文引擎，把历史存档为图片 |
| hicortex | 所有已连接 agent 共享的团队记忆 |
| hwiki | 基于 SQLite FTS5 搜索的 Markdown 维基记忆 |
| hydradb | 每回合前召回，后台保存到 HydraDB |
| hyatlas | 自托管 Go 服务器，混合向量与关键词搜索 |
| icarus | 自我记忆和替换模型 |
| intelligent_memory | 本地阿拉伯语/英语记忆，带标准事实 |
| jev-memory-selector | 技能加一个 MCP 服务器；不注册 Hermes 工具或钩子 |
| lancedb | 内嵌 LanceDB 向量 + FTS 召回 |
| lancedb-suite | 本地 LanceDB 记忆；需要 Ollama + nomic-embed-text |
| limbic | 本地 SQLite，带 sqlite-vec + FTS5 和信任评分 |
| loreconvo | 跨多个入口的会话记忆，存于本地 SQLite |
| lumberroom | 所有 agent 共用一个存储；自托管或云端 |
| mem-os | 用于本地持久事实的 MCP 包 |
| memanto | 类型化记忆，在 Moorcheh 云上做 RAG 回答 |
| memobase | 自托管用户画像加事件时间线召回 |
| memex8 | 自托管 Qdrant 向量记忆 |
| memoria | Memoria Cloud，带范围限定的召回 |
| memory-zvec | 进程内 HNSW + RocksDB FTS 存储 |
| metronix | 预取召回，并写穿内置记忆 |
| mindgraph | 语义图记忆，跟踪目标和决策 |
| mnemostack | 基于 Qdrant 的向量 + 词法 + 图混合召回 |
| nachos | 本地 SQLite 记忆和上下文压缩 |
| nexus | 自托管 Qdrant 记忆，带知识图谱 |
| openltm | 本地 SQLite，带 FTS5、可选向量和衰减 |
| optchat | 只追加的日志加摘要树 |
| optmem-hermes | 本地只追加记忆，带正则/BM25 召回 |
| pastdotdev | 从 past.dev 召回；会话在会话结束时发送 |
| penny | 在 app.mypenny.ai 上的核心记忆和笔记 |
| post-cortex | 通过本地守护进程存入实体图中的类型化记录 |
| qdrant | 存在你托管的 Qdrant 集合中的逐字对话记录 |
| qdrant-memory | 用 LLM 提取事实后存入 Qdrant |
| reme | 文件原生记忆，支持 HTTP 和内嵌后端 |
| stack | 基于 Git 的 Markdown 记忆，后台整理 |
| sugar | 面向编码 agent 的本地 SQLite 记忆 |
| tam | 通过 MCP 提供的 total-agent-memory |
| xmemo | xmemo.dev 上的云记忆 |
| yantrikdb | 去重记忆，并标出相互矛盾之处 |

这些项目都没有在 Hermes 内部发布正面对比的基准，目录也没有对它们排名。选择时，请看数据可能流向哪里（每个条目都有一行披露，说明什么会离开你的机器），以及由谁维护。

## Hindsight 适合的位置

**[Hindsight](https://github.com/vectorize-io/hindsight)** 是 Vectorize 推出的开源记忆服务器，采用 MIT 许可，自托管版包含全部功能。用 `hermes plugins install hindsight` 安装，由 Vectorize 在自己的仓库中维护；旧的 `hindsight-hermes` pip 插件已弃用。它提供三个工具（`hindsight_retain`、`hindsight_recall`、`hindsight_reflect`），三种模式（Hindsight Cloud、带内嵌 Postgres 的本地守护进程（空闲 5 分钟后停止），或你自己的 Hindsight 服务器），以及三种 `memory_mode` 设置：`hybrid`（自动上下文加工具，默认）、`context` 或 `tools`。

上述提供方中，只有 Hindsight 和 Mem0 有公开的正面对比数据。在 Mem0 自己的对比页面上（2026 年 10 月核对），Hindsight 与 Mem0 在 LongMemEval（94.6 对 94.4）和 LoCoMo（92 对 92.5）上基本持平。同一页面上，在长历史测试 BEAM 1M 中，Hindsight 得 73.9，Mem0 得 64.1。同一页面上，在 BEAM 10M 中，Hindsight 得 64.1，Mem0 得 48.6。以上都是厂商自行运行的结果，不是独立审计。

这对 Hermes agent 意味着什么：

- **长时间运行的 agent。** 在 Telegram 或 Discord 上运行的 Hermes agent 可以连续跑几个月。与之最相关的数据，是 BEAM 1M 和 10M 上的差距。
- **会学习，而不只是存储。** Hindsight 把事实整合为 **observations**（带引文和证明次数的、有证据支撑的信念）和 **mental models**，`reflect` 返回的是经过推理的答案，而不是原始片段。
- **需要时可以完全本地运行。** Hindsight 支持 25 个以上的 LLM 提供方，包括 Ollama、LM Studio 和 llama.cpp，因此内嵌模式可以把记忆留在机器上。

需要注意的限制。Hindsight 服务器在默认设置下，每次召回返回的上下文比 Mem0 多：按 Mem0 的测量，是 24K 到 44K token，对比 Mem0 的不到 7K。Hermes 插件对此做了限制：默认每回合召回上限为 4,096 token（`recall_max_tokens`），并且除非你把 `recall_types` 设为包含原始事实，否则只返回已整合的 observations。这是 Hindsight 的 Hermes 集成文档（2026 年 10 月）的说明。Hermes 每个回合都会预取召回，所以如果在意提示词成本，请调低 `recall_max_tokens`。每次保存和整合都会调用一次 LLM，所以写入比 Holographic 这种纯 FTS 存储更贵。它的目录条目和其他记忆条目一样是 community 级别；在云模式下，每个回合都会发送到 Hindsight 的 API，目录披露中也有说明。更多内容见 [Mem0 与 Hindsight 对比](/articles/mem0-vs-hindsight/) 和 [Hindsight 详解](/articles/ai-memory-hindsight/)。

## 如何编写 Hermes 插件

通用插件只需要两个文件。以下是文档中最小的示例，钩子按照 [钩子参考](https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks) 中的 `post_tool_call` 签名编写（Hermes 会把钩子的字段作为关键字参数传入）。

`~/.hermes/plugins/hello-world/plugin.yaml`：

```yaml
name: hello-world
version: "1.0"
description: A minimal example plugin
```

`~/.hermes/plugins/hello-world/__init__.py`：

```python
import json


def register(ctx):
    schema = {
        "name": "hello_world",
        "description": "Returns a friendly greeting for the given name.",
        "parameters": {
            "type": "object",
            "properties": {"name": {"type": "string", "description": "Name to greet"}},
            "required": ["name"],
        },
    }

    def handle_hello(params, **kwargs):
        name = params.get("name", "World")
        return json.dumps({"success": True, "greeting": f"Hello, {name}!"})

    ctx.register_tool(name="hello_world", toolset="hello_world",
                      schema=schema, handler=handle_hello)

    def on_tool_call(tool_name, **kwargs):
        print(f"[hello-world] tool called: {tool_name}")

    ctx.register_hook("post_tool_call", on_tool_call)
```

然后运行 `hermes plugins enable hello-world`，并开启新会话。模型看到的是 `schema["description"]` 中的文字，所以工具的说明要写在那里。

记忆提供方遵循相同的 `register(ctx)` 模式。下面是开发者指南中的骨架：

```python
from agent.memory_provider import MemoryProvider
from agent.secret_scope import get_secret


class MyMemoryProvider(MemoryProvider):
    @property
    def name(self) -> str:
        return "my-provider"

    def is_available(self) -> bool:
        return bool(get_secret("MY_API_KEY"))  # 这里不要发起网络调用

    def initialize(self, session_id: str, **kwargs) -> None:
        self._session_id = session_id
        self._home = kwargs.get("hermes_home")

    def get_tool_schemas(self):
        return []

    def prefetch(self, query: str, *, session_id: str = "") -> str:
        return ""  # 本回合召回的上下文


def register(ctx) -> None:
    ctx.register_memory_provider(MyMemoryProvider())
```

用 `get_secret()` 读取密钥，不要用 `os.environ`，这样每个 Hermes 配置档都有自己的密钥。慢的工作放在 `sync_turn()` 中，并通过 `agent.memory_provider.spawn_context_thread` 启动线程，因为文档说，普通线程可能会把数据写入错误配置档的存储。要把插件加入目录，请提交一个 PR，新增一个带精确 SHA 的 `plugin-catalog/<名称>.yaml`，并遵循提交指南。由于被注入提示词的记忆是一条真实的攻击路径，在发布会存储不可信聊天内容的提供方之前，请先阅读 [AI 记忆注入](/articles/ai-memory-injection/)。


## FAQ

### 如何在 Hermes Agent 中安装插件？

先运行 hermes plugins search <关键词> 在目录中查找，再运行 hermes plugins install <名称>。目录安装会检出 Nous Research 审核过的那个精确提交。通用插件还需要运行 hermes plugins enable <名称>；记忆提供方通过 ~/.hermes/config.yaml 中的 memory.provider 开启，安装提示或 hermes memory setup 会写入这一项。

### Hermes Agent 有哪些记忆提供方插件？

在 Hermes v0.23.0 标签（2026 年 11 月 5 日）中，插件目录的 Memory 分类有 79 项，2026 年 10 月 21 日为 67 项，全部为 community 级别。Hermes 文档把 Honcho、OpenViking、Mem0、Hindsight、Holographic 和 Supermemory 列为目录中的提供方，RetainDB、ByteRover 和 Memori 不在目录中。同一时间只能启用一个外部提供方。

### plugins.enabled 和 memory.provider 有什么区别？

plugins.enabled 是通用插件的允许列表，通用插件包括工具、钩子和斜杠命令。记忆提供方不看这个列表：Hermes 把它们视为互斥的，只有 memory.provider 中指定的那一个会运行。hermes plugins enable 不会启用记忆提供方。

### Hermes 插件为什么安装或更新失败？

从 Hermes v0.22.6（2026 年 11 月 2 日）起，目录中的 requires_hermes 最低版本要求会被强制执行。如果插件需要更新的 Hermes，安装会被拒绝，并显示所需版本和适合你安装方式的更新命令。hermes plugins update 会保留已安装的提交，直到你更新 Hermes。

