# Hermes Agent 记忆：2,200 字符上限、记忆插件与故障排查

> Hermes Agent 用 MEMORY.md（2,200 字符）和 USER.md（1,375 字符）记住内容，每次会话开始时加载一次，并用 SQLite 搜索历史会话。说明它的 9 个外部记忆插件怎么选，以及 Hermes 为什么会忘事。

Source: https://aiagentmemory.org/articles/hermes-agent-memory-zh/ · Updated 2026-11-13 · aiagentmemory.org


**Hermes Agent 的记忆由两个小文件加一份可搜索的历史组成。** 这两个文件都由 agent 自己管理：`MEMORY.md`（2,200 字符）存 agent 的笔记，`USER.md`（1,375 字符）存你的资料。两者在会话开始时加载进系统提示词。较早的对话存在 SQLite 里，需要时再搜索。还可以再加一个外部插件。

[Hermes Agent](https://github.com/NousResearch/hermes-agent) 是 Nous Research 开发的开源（MIT）agent。它可以在终端、[Hermes Agent 桌面版](/articles/hermes-agent-desktop/) 或聊天应用里运行（所有入口共享同一份记忆），还能从自己的经验中创建可复用的技能。本页介绍它记忆的每一层、外部插件，以及记忆看起来不工作时该检查什么。细节来自官方 [Hermes 记忆文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory)，2026 年 10 月核对。

## Hermes Agent 的记忆是什么？

**Hermes Agent 的记忆是一个有容量上限、会跨会话保留的精选存储。** agent 决定保存什么，通过 memory 工具把简短条目写入 MEMORY.md 和 USER.md。每次会话开始时，Hermes 把两个文件作为冻结的一块注入系统提示词。

这个设计刻意保持小巧。两个文件合计约 1,300 个 token，始终在上下文里。其他内容（完整对话记录、旧任务）在 agent 搜索之前都不进入提示词。这种"始终加载的记忆 + 按需召回"的分工，与 [AI agent 记忆解释](/articles/ai-agent-memory-explained/) 中描述的相同。第一次用 Hermes？[Hermes Agent 指南](/articles/hermes-agent/) 介绍了安装、设置，以及每个记忆文件放在哪里。

## Hermes 记忆的各层

| 层 | 位置 | 大小 | 何时在上下文中 |
|---|---|---|---|
| **MEMORY.md** | `~/.hermes/memories/` | 2,200 字符（约 800 个 token） | 每次会话，作为冻结快照 |
| **USER.md** | `~/.hermes/memories/` | 1,375 字符（约 500 个 token） | 每次会话，作为冻结快照 |
| **会话历史** | `~/.hermes/state.db`（SQLite，FTS5） | 无上限，除非开启自动清理 | 仅当 agent 调用 `session_search` 时 |
| **技能** | 技能目录 | 每个技能不同 | 技能相关时 |
| **外部插件** | 取决于插件 | 取决于插件 | 每轮对话前预取 |

使用 profile 时，文件位于 `~/.hermes/profiles/<name>/memories/` 下。

### MEMORY.md 和 USER.md

**MEMORY.md** 存 agent 的工作知识：环境事实、项目约定、工具的怪癖，以及有效的做法。**USER.md** 存你的信息：姓名、角色、时区、沟通风格、讨厌的事情和技术水平。文档建议跳过琐碎的事实、容易再查到的东西、原始数据堆，以及 `SOUL.md` 或 `AGENTS.md` 里已有的内容。

条目之间用 `§` 分隔。注入的内容带有一个用量标题（例如 67%，2,200 字符中用了 1,474 个），让模型知道存储有多满。

### memory 工具

agent 用一个工具、三个动作编辑记忆：

1. **add**：追加一个新条目。
2. **replace**：用唯一的子串（`old_text`）找到条目，然后覆盖。
3. **remove**：删除匹配 `old_text` 的条目。

没有读取动作，因为内容已经在提示词里。完全相同的重复条目会被拒绝。如果写入会超过上限，工具会返回错误并列出当前条目，agent 必须用一批操作既腾出空间又添加新条目。Hermes 从不自动压缩记忆。文档建议用量超过 80% 时整理一次。

每个条目在保存前都会扫描提示词注入、凭据外泄、SSH 后门和不可见的 Unicode 字符。这很重要，因为记忆是已知的攻击面，见 [AI 记忆注入](/articles/ai-memory-injection/)。

### 会话搜索

过去的 CLI 和聊天会话存在 `~/.hermes/state.db` 里。`session_search` 工具对它们做全文搜索（SQLite FTS5），返回原始消息，agent 还可以翻阅找到的会话。`hermes sessions list` 可以在命令行里列出这些会话。这是 Hermes 对"我们上个月做了什么？"的回答：不存进记忆，但可以找回来。

**一次搜索胜过更好的摘要。** 第三方项目 [hermes-jev-skills](https://github.com/kerpopule/hermes-jev-skills)（2026 年 10 月约 1,100 个 GitHub 星标）在它自己的 7 次 Hermes 工作会话上测试了会话交接，每次用 15 道题的回忆测试。在其 [评分表](https://github.com/kerpopule/hermes-jev-skills/blob/main/evals/compaction/results/SCORECARD-2026-09-20.md) 中，让下一个会话多调用一次 `session_search`，对每种交接笔记都能把回忆率提高 20 到 33 个百分点。没有笔记、只搜索一次的会话（56.7%），胜过所有只读笔记而不搜索的情况。最好的笔记是根据整段对话、预算 1,200 字写成的：单独 58.7%，加一次搜索 75.0%；被它替换的 400 字笔记分别是 37.5% 和 68.3%。这些是作者自己在 7 个会话上的数字，不是独立基准。实际的经验是：让交接笔记写明旧会话的 id，方便下一个会话搜索，不要清理你可能还要找的会话。他们的 [交接笔记](https://github.com/kerpopule/hermes-jev-skills/blob/main/docs/hermes-compaction.md) 还提到，在 Hermes 0.21 上，同时传 `query` 和 `session_id` 时，`session_search` 会从头读取会话而忽略查询；应先用关键词搜索，再用 `around_message_id` 打开命中的位置。

这段历史只有在它还存在时才找得到。默认情况下 Hermes 保留所有会话，而 [会话文档](https://hermes-agent.nousresearch.com/docs/user-guide/sessions) 警告 `state.db` 会"无限增长"，网关和定时任务的安装中有多 GB 的报告。自动清理默认关闭（v0.22.3，2026 年 10 月 21 日改回需手动开启）：

```yaml
sessions:
  auto_prune: true      # delete ended sessions past retention_days, at startup
  retention_days: 90    # default
```

只删除已结束的会话，按最后活动时间计算，清理最多每天运行一次。如果 `config.yaml` 没有 `auto_prune` 项，而且存放 `state.db` 的磁盘空间不足（可用少于 2 GB，或可用低于 10% 且少于 20 GB），Hermes 会问一次；用 `/autoprune on`、`/autoprune off` 或 `/autoprune later` 回答。要做一次性清理，可以用 `hermes sessions prune --older-than 30`（加 `--dry-run` 先预览），`hermes sessions optimize` 会压缩文件但不删除任何内容。对记忆来说的代价是：被清理的会话会从 `session_search` 消失，所以超过 `retention_days` 仍需保留的内容，必须放进 `MEMORY.md`、技能，或外部插件（插件会保存自己的副本）。

### 后台审查和写入审批

一次对话结束后，后台审查可以更新记忆和技能。它默认开启（`auxiliary.background_review.enabled`），默认使用主模型，输入上限为模型上下文窗口的 75%（最多 600,000 个 token）。如果你想审批写入，设置 `memory.write_approval: true`；待写入的内容会出现在 `/memory pending` 下，可以批准或拒绝。

从 v0.22.6 起，审查只保存对话支持的事实：每条事实都必须有用户消息或工具结果作为依据。之前，一次性的"把那个改短一点"会在 3 次测试中 3 次变成长期的"用户偏好简洁的回答"；现在是 3 次中 0 次，而你明确说出的事实（"始终用摄氏度，记住这个"）和调试时发现的环境事实仍会保存（[PR #80788](https://github.com/NousResearch/hermes-agent/pull/80788)，作者用一个故意较弱的模型做的 A/B 测试）。明确说出的长期喜好或厌恶（"不要再做 Y"）仍会保存。

## 配置 Hermes 记忆

设置保存在 `~/.hermes/config.yaml`：

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  write_approval: false
  # provider: honcho   # optional external provider
```

如果同时关闭 `memory_enabled` 和 `user_profile_enabled`，memory 工具会被完全移除。你可以调高字符上限，但每个字符每轮都要付费，所以更大并不免费。

`/journey` 命令（也可以用 `hermes journey`）显示技能和记忆条目的时间线，`hermes journey delete <node>` 可以删除一条。

## 外部记忆插件

Hermes 可以在内置文件之外同时运行**一个外部插件**。启用后，Hermes 会在每轮对话前预取相关记忆，每次回复后同步对话，在支持的情况下于会话结束时提取记忆，并把内置记忆的写入同步到插件。[记忆插件页面](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) 截至 2026 年 10 月列出了九个：

| 插件 | 功能（据 Hermes 文档） | 托管方式 | 工具数 |
|---|---|---|---|
| **Honcho**（Plastic Labs） | 跨会话的用户建模，带辩证推理 | 云端或自托管 | 5 |
| **OpenViking**（火山引擎） | 带文件系统式层级的上下文数据库 | 自托管（AGPL-3.0）或火山引擎云 | 6 |
| **Mem0** | LLM 事实抽取，语义搜索和去重 | 云端、自托管或进程内 | 4 |
| **Hindsight**（Vectorize） | 带知识图谱和实体解析的长期记忆 | 云端、嵌入式 Postgres 或本地服务器 | 3 |
| **Holographic** | 本地 SQLite 事实库，带 FTS5、信任评分和 HRR 代数 | 仅本地 | 2 |
| **RetainDB** | 带混合搜索的云记忆 API | 云端（标价每月 20 美元） | 10 |
| **ByteRover** | 本地优先的记忆，通过 `brv` 命令行工具 | 本地，可选云同步 | 3 |
| **Supermemory** | 带用户画像召回的语义长期记忆 | 云端或自托管 | 4 |
| **Memori**（Memori Labs） | 结构化长期记忆 | Memori Cloud | 5 |

插件正在从 Hermes 代码树中移到各自维护者的仓库。Holographic 以前内置在代码树中，现在由 itpartypattaya 维护（[itpartypattaya/hermes-plugin-holographic](https://github.com/itpartypattaya/hermes-plugin-holographic)），通过同名的插件目录安装，设置和数据库保持不变。RetainDB 和 ByteRover 将于 2026 年 10 月 15 日移出核心，文档说它们的独立仓库"无人维护，欢迎新的维护者"，也没有列出目录安装命令。Honcho、OpenViking、Mem0、Hindsight、Holographic 和 Supermemory 用 `hermes plugins install <name>` 安装；Memori 使用它自己的 `hermes-memori` 命令行工具。Holographic 使用全息约简表示，在 [用于 AI agent 的全息记忆](/articles/holographic-memory-ai-agent/) 中有解释；[Honcho LLM 记忆](/articles/honcho-llm-memory/) 有专门的页面。

如果你用的插件不再随 Hermes 一起发布，不必手动重新安装。据文档，`hermes update` 会为每个指定了它的 profile 安装其目录插件（桌面版首次启动时也会这样做），插件名、设置、数据目录和工具都保持不变。如果安装无法完成（离线、被拒绝，或 `security.allow_lazy_installs: false`），Hermes 会打印准确的 `hermes plugins install <name>` 命令。

**插件能看到什么。** Hermes 会把每一轮的内容交给插件：包括工具输出在内的完整对话记录、召回查询、会话结束和压缩前的对话记录、同步的记忆写入、委派结果，以及插件自己工具的参数。所有内容先经过 Hermes 用于聊天投递的同一套密钥清理，即使设置了 `security.redact_secrets: false`，因此工具输出中回显的 API 密钥会在插件保存前被遮盖。文档还有一个限制：没有可识别格式的凭据（例如任意密码）不会被检测到。如果这很重要，请选择你能自托管的插件。

OpenViking 发布过一项针对 Hermes 自带记忆的测试。在其 2026 年 5 月的基准文章中，Hermes 使用原生记忆在 LoCoMo 上得分 33.38%，使用 OpenViking 时为 82.86%，输入 token 减少 34.3%。这些是 OpenViking 自己的数字，没有写明评判模型，并且比较的是 Hermes 那 1,300 个 token 的文件，而不是其他插件。OpenViking 的 Hermes 设置提供快速本地服务器或火山引擎的托管服务；记忆要等后台抽取完成后才会出现，详见 [OpenViking agent 记忆](/articles/openviking-agent-memory/)。

这些插件如何安装、更新并在每轮对话中接入，以及 v0.23.0 中插件目录里 79 项 Memory 分类，见 [Hermes Agent 插件](/articles/hermes-agent-plugins/)。

每个插件的设置方式相同。先把 Hermes 更新到 v0.23.1。根据其发布说明，v0.23.0 及更早的版本读取的插件目录上限是 2 MiB，所以 `hermes plugins install` 可能会悄悄安装一个旧的固定版本，漏掉你上次更新后新增的插件。v0.23.1 把上限提高到 16 MiB（PR #151860）。

```bash
hermes plugins install honcho   # or mem0, hindsight, supermemory, openviking
hermes memory setup             # interactive picker and credentials
hermes memory status            # confirm which provider is active
hermes memory off               # disable the external provider
```

### Hermes Agent 最好的记忆插件

没有唯一的最佳选择，取决于 agent 必须记住什么。Hermes 的[插件文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) 也没有推荐某一个，但说明了各自的特点（2026 年 10 月 28 日核对）：

| 你需要 | 选择 | Hermes 文档说的特点 | 运行位置 |
|---|---|---|---|
| 几个月的历史，记忆合并成结论 | **Hindsight** | `hindsight_reflect` "提供其他插件都没有的跨记忆综合" | 云端、嵌入式 Postgres 或本地服务器 |
| 跨会话的用户模型 | **Honcho** | "带辩证推理的跨会话用户建模" | 云端或自托管 |
| 不用手动管理的记忆 | **Mem0** | 最适合"免手动的记忆管理"：Mem0 自己做抽取 | 云端、自托管服务器或进程内 |
| 数据不离开本机，不需要额外服务 | **Holographic** | 最适合"仅本地记忆……没有外部依赖" | 本地 SQLite |
| 把召回的记忆排除在存储内容之外 | **Supermemory** | "自动上下文围栏"把召回的记忆从捕获的对话轮次中剥离 | 云端或自托管 |
| 文件系统式的上下文存储 | **OpenViking** | 带文件系统层级的上下文数据库 | 自托管或火山引擎云 |
| 除两个文件外不需要别的 | **不用插件** | 内置记忆始终开启 | 本地文件 |

如果你关心的是对用户建模，选 **Honcho**；想要托管的事实库、设置又少，选 **Mem0** 或 **Supermemory**；想把记忆整理成有证据支持的观察，并能调用 reflect，选 **Hindsight**；如果任何数据都不能离开本机，选 **Holographic**（或 Hindsight 的嵌入式 Postgres）。在 RetainDB 或 ByteRover 重新有人维护之前，不要新建在它们上面的配置。Mnemosyne 是单文件的 SQLite 记忆，Hermes 文档把它列为目录插件（`hermes plugins install mnemosyne`），也是一个本地选项，见 [Mnemosyne 与 Hindsight 对比](/articles/mnemosyne-vs-hindsight/)。如果内置的两个文件已经够用，就完全不用插件；插件会增加 LLM 调用和一个依赖。

九个插件中，Hindsight 和 Mem0 是仅有公开正面对比数字的两个。在 Mem0 自己的对比页上，两者在 LongMemEval（94.6 对 94.4）和 LoCoMo（92 对 92.5）上基本持平；在长历史的 BEAM 测试中，Hindsight 领先：1M token 时 73.9 对 64.1，10M token 时 64.1 对 48.6（均为厂商自测）。对于运行数月的 Hermes agent，这一点很重要。Hermes 文档还说，Hindsight 的 `hindsight_reflect` 工具"提供其他插件都没有的跨记忆综合"。[Hindsight](https://github.com/vectorize-io/hindsight) 是 MIT 许可，自托管服务器包含全部功能，并且可以通过 Ollama、LM Studio 或 llama.cpp 使用本地模型，所以在数据不能离开机器时也能用。代价是：Hindsight 服务器在默认设置下，每次检索返回的上下文比 Mem0 多（按 Mem0 的测量，24K 到 44K 对比 7K 以下的 token）。Hermes 插件把这个值限制在每轮 4,096 个 token（`recall_max_tokens`），并且按 Hindsight 的 Hermes 集成文档，默认只召回已整理的观察；由于 Hermes 每轮都会预取召回，如果你在意提示词成本，请调低这个上限。每次写入还要调用 LLM，所以写入比本地 FTS 存储更贵。详见 [Mem0 与 Hindsight 对比](/articles/mem0-vs-hindsight/)。

[Hindsight v1.2.0](https://github.com/vectorize-io/hindsight/releases/tag/v1.2.0)（2026 年 11 月 3 日）根据其 PR 修复了 Hermes 插件的主要缺口：

| v1.2.0 之前的问题 | 现在 | PR |
|---|---|---|
| 自动保存失败（服务器宕机、超时、5xx）后只记日志并丢弃 | 保留，并在下一次保存前和退出时重发，最多 5 次，不会重复存储。失败的保存暂存在内存中，因此服务器宕机时崩溃仍会丢失它们 | [#6013](https://github.com/vectorize-io/hindsight/pull/6013) |
| 服务器已接受、随后失败的保存被计为完成 | 会被监控直到结束；失败的会在服务器上重试，最多 3 次 | [#6013](https://github.com/vectorize-io/hindsight/pull/6013) |
| Kanban 工作者和 `delegate_task` 子 agent 的内容被存入记忆库 | 跳过，与定时任务相同 | [#6000](https://github.com/vectorize-io/hindsight/pull/6000) |
| Windows 上使用中文、日文或韩文区域设置时插件无法加载 | 可以加载 | [#6000](https://github.com/vectorize-io/hindsight/pull/6000) |
| Hermes Docker 镜像中插件找不到 `uvx` | 可以找到 | [#6000](https://github.com/vectorize-io/hindsight/pull/6000) |

同一版本给 Hindsight 的 OpenClaw 插件提供了相同的三个工具，所以同一个记忆库可以同时被两个 agent 使用，见 [OpenClaw 与 Hermes Agent 记忆对比](/articles/openclaw-vs-hermes-agent-memory/#same-tools-in-both-agents-hindsight-v120)。

## Hermes v0.23.0 的记忆变化

[v0.23.0](https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0)（2026 年 11 月 5 日）是完整版本，记录了 v0.22.1 到 v0.22.7 的补丁，因此下面 v0.22.6 表格中的内容也包含在其中。以下是那些未在其中说明的记忆相关改动，来自发布说明和 PR 内容：

| 改动 | 之前 | v0.23.0 中 | 来源 |
|---|---|---|---|
| 每个 profile 一个 gateway | CLI、TUI、桌面版、编辑器、机器人和定时任务各自可以运行一个 agent，并同时写入同一段对话的 `state.db`，导致消息被写两次或丢失 | 一个 gateway 进程管理该 profile 的会话；每个入口都接入它，每个会话只有一个队列和一个写入者 | 发布说明，[#106742](https://github.com/NousResearch/hermes-agent/pull/106742) |
| 压缩摘要 | "可能"之类的猜测常被丢掉，所以压缩后猜测会作为事实回来 | 每份摘要都有"未验证 / 工作假设"一节，每个不确定的论断占一行 `UNVERIFIED: <claim> — <basis>`；按 PR 的测试，每次摘要调用多约 79 个提示词 token | [#86019](https://github.com/NousResearch/hermes-agent/pull/86019) |
| 压缩后的技能 | 只有超过 5,000 字符的技能正文会留下"重新加载"提示，较小的技能会悄悄消失 | 每个被压缩掉的技能都会留下重新加载标记 | [#138428](https://github.com/NousResearch/hermes-agent/pull/138428) |
| 记忆的插件钩子 | 插件会收到 `on_pre_compress`，但压缩后或会话被删除时没有信号 | 每次压缩完成后触发一次 `post_compaction`，插件可以重新注入被摘要掉的内容；`on_session_delete` 在删除、归档和清理时触发（不保证恰好一次） | [#141017](https://github.com/NousResearch/hermes-agent/pull/141017)，[#140995](https://github.com/NousResearch/hermes-agent/pull/140995) |
| 插件目录中的 Memory 分类 | 67 项（10 月 21 日） | v0.23.0 标签时 79 项，全部为 `community` 级别；新增的有 Mnemoverse、post-cortex、Membase、conxt、Inspeximus 和 nyx | 我们统计的 `plugin-catalog/` 标签内容 |

最明显的变化是单一 gateway。之前，同一个聊天如果在两个地方打开，可能有两个 agent 同时写入同一段历史，重复或丢失的消息就是这样来的。现在，在任何地方打开的聊天都是同一段对话、只有一个写入者，所以 `session_search` 读到的是一份干净的历史。`UNVERIFIED:` 行对长会话很重要：agent 在压缩前做的猜测，不会再作为它"记得"的事实回来。插件方面的更多内容见 [Hermes Agent 插件](/articles/hermes-agent-plugins/#what-changed-for-plugins-in-hermes-v0230)。

## Hermes v0.22.6 的记忆变化

[v0.22.6](https://github.com/NousResearch/hermes-agent/releases/tag/v0.22.6)（2026 年 11 月 2 日）是一个汇总了 776 个已合并 PR 的补丁版本；v0.23.0 后来完整记录了它。下面的记忆改动都已对照 PR 核实，并确认在 v0.22.6 标签中：

| 改动 | 之前 | 现在 | PR |
|---|---|---|---|
| 后台审查 | 从一次性请求和随口的评论中猜测偏好 | 只保存有用户消息或工具结果支持的事实 | [#80788](https://github.com/NousResearch/hermes-agent/pull/80788) |
| Kanban 工作者 | 工作轮次被当作操作者本人的内容发给记忆插件，于是 "work kanban task t_..." 被存成关于你的事实 | 工作轮次带有 `kanban:dispatcher` 作者标记 | [#144861](https://github.com/NousResearch/hermes-agent/pull/144861) |
| 调低 `memory_char_limit` / `user_char_limit` | 超过新上限的条目会阻止所有替换和删除，直到你手动编辑文件 | 删除可以进行（会先做 `.bak` 快照）；只有增加文字的写入会被拒绝 | [#144240](https://github.com/NousResearch/hermes-agent/pull/144240) |
| `hermes update` 关闭了插件 | `memory.provider` 被清空，Hermes 悄悄回退到内置记忆 | 保留插件设置，之后的更新在插件适配时重新开启；`hermes memory status` 和 `hermes doctor` 会说明插件已关闭 | [#144257](https://github.com/NousResearch/hermes-agent/pull/144257) |
| 桌面版和控制台的记忆页面 | 只显示 MEMORY.md / USER.md，所以正在工作的插件看起来没有启用 | 显示当前启用的插件及其就绪状态，并把两个文件标注为始终开启的内置层 | [#145574](https://github.com/NousResearch/hermes-agent/pull/145574) |
| 压缩 | 压缩触发时才写摘要（相关 issue 中的中位停顿为 184 秒） | 可选的 `compression.prepare_ahead: true` 在接近触发点时后台写摘要；如果记忆插件会添加压缩上下文，它在第一次压缩后自动关闭 | [#144684](https://github.com/NousResearch/hermes-agent/pull/144684) |
| 技能 | agent 可以改写任何技能 | `hermes curator lock <skill>` 阻止所有 agent 写入该技能的路径；见 [Hermes Agent 技能](/articles/hermes-agent-skills/#keeping-skills-healthy-curator-approval-and-limits) | [#147557](https://github.com/NousResearch/hermes-agent/pull/147557) |

对"Hermes 记住了错误的东西"这一问题最重要的两项是：审查不再凭空编造偏好；Kanban 运行不再污染外部插件中的你的资料。如果你在 v0.22.6 之前用某个插件运行过 Kanban，请检查其中是否有被当作关于你的事实保存的任务笔记。

## Hermes 记忆不工作：检查什么

大多数"Hermes 忘了"的报告来自少数几个原因：

1. **快照是冻结的。** 写入会立即保存到磁盘，但只在下一次会话才出现在提示词中。运行 `/new` 重新加载。消息网关（Telegram、Discord）会一直是一个长会话，直到你重置它。
2. **存储已满。** 达到 2,200 或 1,375 字符后，新增会失败，直到 agent 整理。在标题处或直接打开文件查看用量。
3. **记忆被关闭。** 在 `config.yaml` 中确认 `memory_enabled` 和 `user_profile_enabled`，并确认 `memory` 没有出现在 `agent.disabled_toolsets` 里。
4. **写入在等待审批。** 设置了 `write_approval: true` 时，在你于 `/memory pending` 下批准之前，什么都不会写入。
5. **profile 不对。** 每个 profile 有自己的记忆目录。
6. **这个细节本来就不是记忆。** 长的任务历史属于会话搜索，所以让 agent 搜索过去的会话。如果开启了自动清理，超过 `retention_days` 的会话已经没了。
7. **插件出问题。** 运行 `hermes memory status`。同一时间只能启用一个插件，云端插件需要它们的凭据（例如 RetainDB 从 `~/.hermes/.env` 读取 `RETAINDB_API_KEY`）。从 v0.22.6 起，那里显示的 `Plugin: turned off` 表示某次更新停用了该插件；启动提示会给出原因，`hermes plugins enable <name>` 可以重新开启。

## Hermes 与其他 agent 的比较

Hermes 的内置记忆最接近 OpenClaw 的 Markdown 文件和 Claude Code 的自动记忆：纯文本、体积小、启动时加载。不同之处在于严格的字符上限和冻结快照。Hermes 还可以导入其他 agent 的记忆。从 v0.22.5 起，这是官方的 `agent-importers` 插件（`hermes plugins install agent-importers`，然后 `hermes import-agent`），全新的 `hermes setup` 在发现 `~/.openclaw`、`~/.claude`、`~/.codex` 或 `~/.cursor` 时会提供它。OpenClaw 的 `MEMORY.md`、`USER.md` 和每日笔记会合并进 Hermes 记忆；Claude Code 的 `CLAUDE.md`、Codex 的 `AGENTS.md` 和 `memories/*.md`，以及 Cursor 的规则，会变成 `MEMORY.md` 中的条目。每次导入都会先显示预览，`hermes import-agent --sync` 会拉取之后的变化（[Hermes 文档](https://hermes-agent.nousresearch.com/docs/user-guide/import-from-other-agents)）。大批量导入后请检查 `MEMORY.md`：memory 工具把它限制在 2,200 字符，而一份很长的 `CLAUDE.md` 本身就可能超过这个数。并排对比见 [OpenClaw 与 Hermes Agent 记忆对比](/articles/openclaw-vs-hermes-agent-memory/)。


## FAQ

### Hermes Agent 的记忆是怎么工作的？

Hermes 在 ~/.hermes/memories/ 里保存两个小文件：MEMORY.md 存 agent 自己的笔记（2,200 字符），USER.md 存用户资料（1,375 字符）。两者在会话开始时以冻结快照的形式加载进系统提示词。过去的会话存在 SQLite 里，需要时用 session_search 工具搜索。

### 为什么 Hermes 不记得我刚告诉它的事？

记忆写入会立即保存到磁盘，但只会在下一次会话的系统提示词里出现，因为快照是冻结的，以保持提示词缓存稳定。运行 /new 开始新会话即可重新加载。同时检查 memory_enabled 是否为 true，以及存储是否已满。

### Hermes Agent 支持哪些外部记忆插件？

截至 2026 年 10 月，文档列出 Honcho、OpenViking、Mem0、Hindsight、Holographic、RetainDB、ByteRover、Supermemory 和 Memori。RetainDB 和 ByteRover 已于 2026 年 10 月 15 日移出核心，无人维护；Holographic 现在是社区插件。同一时间只能启用一个外部插件，它与内置的 MEMORY.md 和 USER.md 并行运行。

### Hermes Agent 最好的记忆插件是哪个？

取决于你的需求，Hermes 的文档没有指定一个。需要几个月的历史时选 Hindsight，因为它的 hindsight_reflect 工具提供跨记忆综合，文档称其他插件都没有这个功能。需要建模用户时选 Honcho，想要托管的自动抽取时选 Mem0，想要完全本地且不依赖额外服务时选 Holographic。如果两个内置文件已经够用，就不用装插件。

### 为什么 Hermes 会保存一些我从没说过的事？

在 v0.22.6（2026 年 11 月 2 日）之前，后台记忆审查可能把一次性请求变成长期偏好，Kanban 工作轮次也会像你亲口说的一样发给外部插件。v0.22.6 修复了这两点：审查只保存有用户消息或工具结果支持的事实（PR #80788），Kanban 轮次带有 kanban:dispatcher 作者标记（PR #144861）。运行 hermes update，然后用 memory 工具或 hermes journey delete 删除错误条目。

### Hermes v0.23.0 对记忆做了哪些改动？

v0.23.0（2026 年 11 月 5 日）让每个 profile 只由一个 gateway 管理全部会话，因此 CLI、桌面版、机器人和定时任务不会同时写入同一个聊天（PR #106742）。压缩摘要现在把猜测列在 UNVERIFIED: 行中，避免它们以事实的身份回来（PR #86019）；插件还新增了 post_compaction 和 on_session_delete 钩子。插件目录里的 Memory 分类增加到 79 项。

