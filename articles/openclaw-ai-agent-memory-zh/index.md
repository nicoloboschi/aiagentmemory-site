# OpenClaw 记忆：文件、搜索与嵌入模型

> OpenClaw 记忆如何工作：MEMORY.md 与每日笔记、混合 memory_search、嵌入模型、dreaming，以及 8 个记忆插件的对比（OpenViking、Hindsight）。

Source: https://aiagentmemory.org/articles/openclaw-ai-agent-memory-zh/ · Updated 2026-11-10 · aiagentmemory.org


**OpenClaw 的记忆是一组文件。** agent 写下它要记住的内容，再读回来。`MEMORY.md` 存持久的事实和决定，`USER.md` 存你的偏好，按日期命名的每日笔记存正在进行的上下文。前两个文件在会话开始时加载。所有内容都会被存入 SQLite 建立索引，agent 通过 `memory_search`（向量搜索和关键词搜索的混合）找回更早的笔记。

[OpenClaw](https://github.com/openclaw/openclaw) 是开源（MIT）的个人 AI 助手，由 Peter Steinberger 和社区贡献者构建，现在由 OpenClaw Foundation 维护。它运行在你自己的电脑上，通过 WhatsApp、Telegram、Slack、Discord、iMessage 以及 20 多个其他渠道与你交流。本页内容来自官方[记忆文档](https://docs.openclaw.ai/concepts/memory)和[记忆搜索文档](https://docs.openclaw.ai/concepts/memory-search)，2026 年 10 月核对。第一次使用 OpenClaw？请先看 [OpenClaw 指南](/articles/openclaw-guide-zh/)，里面有安装、入门引导和 Gateway。

## 什么是 OpenClaw 记忆？

**OpenClaw 记忆基于文件。** 模型只能记住写到磁盘上、以 Markdown 形式保存在 agent 工作区里的内容。记忆插件（默认是 memory-core）为这些文件建立索引，并给 agent 提供搜索和读取它们的工具，所以知识在重启、压缩和新会话之后仍然保留。

因为记忆是纯文本，你可以像处理任何文件一样打开、编辑、对比和备份。这和把事实存在数据库里的记忆层不同。关于这类通用模式，参见 [AI agent 记忆详解](/articles/ai-agent-memory-explained/)。

## 记忆文件

| 文件 | 存什么 | 何时加载 |
|---|---|---|
| `MEMORY.md` | 精选的长期事实和决定 | 会话开始时 |
| `USER.md`（可选） | 稳定的偏好、风格、人际关系、进行中的项目 | 会话开始时，预算较小 |
| `memory/YYYY-MM-DD.md` | 每日笔记和正在进行的上下文 | 在裸 `/new` 或 `/reset` 时加载今天和昨天的；其他时候通过搜索 |
| `memory/YYYY-MM-DD-<slug>.md` | 带后缀的每日笔记，例如 session-memory 钩子生成的 | 同每日笔记 |
| `DREAMS.md`（可选） | 供人工审阅的 dreaming 摘要 | 不注入 |

工作区默认是 `~/.openclaw/workspace`。如果 `MEMORY.md` 超过启动预算，磁盘上的文件保持完整，但注入的副本会被截断。`/context list`、`/context detail` 和 `openclaw doctor` 会显示大小和截断情况。

文档还给出一条实用规则：影响行动的笔记，要记下什么时候可以行动（需要批准、有效期、谁说的），而不只是记事实。文档也明说，记忆“引导行为，但不强制执行策略”。硬性限制应放在批准设置和沙箱里。

### 导入其他 agent 的记忆

控制界面里有 **Settings → Import Memory**。它会把 Codex（`~/.codex/memories`）、Claude Code（`~/.claude/projects/*/memory`）和 Hermes（`MEMORY.md` 与 `USER.md`）的 Markdown 记忆复制到 `memory/imports/`。导入的内容可以搜索，但不会合并进 `MEMORY.md`，并且导入前会先做备份。

## memory_search 如何工作

默认的 memory-core 插件给 agent 三个工具：

- **`memory_search`**：对笔记做语义搜索，也可选择搜索会话记录。
- **`memory_get`**：读取某个文件或某段行号。
- **`intent`**：创建、列出或取消由事件触发的常驻意图。

配置了嵌入提供商之后，搜索是**混合**的。向量搜索（语义）和 BM25 关键词搜索（ID、环境变量、代码符号等精确词）并行运行，另外还有文件名搜索。如果其中一条路径失败，其余的照常运行。

最终得分是三项相乘：混合相关度、**时间衰减**和写入时设定的重要性分数。衰减的半衰期是 30 天，所以一个上个月的笔记只算一半。`MEMORY.md`、`USER.md` 和没有日期的文件不衰减。之后有一个固定的 MMR 步骤（lambda 0.7），去掉近似重复的片段，让结果不会是同一条笔记的五份副本。

会话记录是可选的：在 `sources` 里加上 `"sessions"`，并设置 `experimental.sessionMemory: true`。关于记忆系统如何按时间排序，参见[AI 记忆中的时间推理](/articles/temporal-reasoning-ai-memory/)。

## OpenClaw 使用哪个嵌入模型

默认的嵌入提供商是 OpenAI。你可以用 `memory.search.provider` 修改：

| 提供商 | 配置 ID | 是否需要 API key | 默认模型或说明 |
|---|---|---|---|
| OpenAI | `openai` | 视情况 | API key，或符合条件的 Codex OAuth |
| Gemini | `gemini` | 需要 | 多模态（图片和音频）用 `gemini-embedding-2` |
| Voyage | `voyage` | 需要 | `voyage-4-large` |
| Mistral | `mistral` | 需要 | `mistral-embed` |
| DeepInfra | `deepinfra` | 需要 | `BAAI/bge-m3` |
| Bedrock | `bedrock` | 不需要 | AWS 凭证链 |
| GitHub Copilot | `github-copilot` | 不需要 | 需要 Copilot 订阅 |
| 本地 | `local` | 不需要 | 托管的 llama.cpp GGUF，约 0.3 GB |
| Ollama / LM Studio | `ollama` / `lmstudio` | 不需要 | 你本地的模型 |
| OpenAI 兼容 | `openai-compatible` | 通常需要 | 任何 `/v1/embeddings` 接口 |

设为 `none` 则只用关键词搜索；设为 `auto` 则由 OpenClaw 自动选择，嵌入失败时退回关键词搜索。要用本地嵌入，先安装 `@openclaw/llama-cpp-provider`；之后 OpenClaw 会安装 `llama-server` 并自动下载 GGUF 文件。

如何选择：如果你已经在用 OpenAI，就保持默认。如果笔记绝不能离开本机，用**本地**或 **Ollama**。如果你想索引图片和音频，用 **Gemini**。更换提供商或模型后，需要用 `openclaw memory index --force` 重建索引，因为不同模型产生的向量无法比较。关于如何选择模型的背景知识，参见 [RAG 的嵌入模型](/articles/vector-database-for-llm-memory/#embedding-models-for-agent-memory)。

## 刷新、dreaming 与压缩

有两个后台进程让记忆保持最新：

1. **记忆刷新。** 在[压缩](https://docs.openclaw.ai/concepts/compaction)缩短对话之前，OpenClaw 会运行一轮静默的对话，让 agent 保存所有重要内容。这个功能默认开启。你可以用 `agents.defaults.compaction.memoryFlush.enabled: false` 关闭它，或者把它指向更便宜的模型，例如 `ollama/qwen3:8b`。只读沙箱会话会跳过它。
2. **Dreaming。** 一个定期运行的任务会给短期召回信号打分，只有通过分数、频率和查询多样性阈值的条目才会晋升到 `MEMORY.md`。不可信的和系统生成的候选会被排除。摘要写入 `DREAMS.md`，供你审阅。用 `plugins.entries.memory-core.config.dreaming.enabled: false` 关闭它。

Dreaming 是 OpenClaw 版的[记忆巩固](/articles/memory-consolidation-ai-agents/)：把一堆每日笔记变成一小组持久的事实。

## 记忆引擎和插件

记忆插槽是可以替换的。由于有些插件也接管上下文组装，记忆选项可以放在三个位置：记忆插槽、`contextEngine` 插槽，或两者之外。

| 选项 | 插槽 | 记忆存在哪里 | 增加了什么 |
|---|---|---|---|
| 内置 memory-core（默认） | 记忆 | 工作区里的 SQLite | 对 Markdown 笔记做关键词、向量和混合搜索；无需额外依赖 |
| PostgreSQL 上的 memory-core | 记忆 | 你的 PostgreSQL agent 数据库 | 同样的搜索；需要 `unaccent`、`pg_trgm`，向量还需要 `pgvector` |
| Honcho | 记忆（插件） | Honcho 服务器 | 跨会话的用户建模、语义搜索、多 agent 感知 |
| LanceDB | 记忆（插件） | LanceDB | 兼容 OpenAI 的或本地 Ollama 嵌入，自动召回，自动捕获 |
| Memory Wiki（自带） | 记忆旁边 | Wiki 库 | 带证据的论断，跟踪矛盾和时效；不替代记忆插件 |
| OpenViking | 上下文引擎 | OpenViking 服务器（`viking://` 树） | 会话归档和摘要，每次回复前召回，接管压缩 |
| Mem0 | 记忆（插件） | Mem0 平台或自托管 | 提取出的事实，而不是文件搜索 |
| Hindsight | 记忆（插件） | Hindsight Cloud、你自己的服务器，或本地嵌入式守护进程 | 提取出的事实、巩固后的 observations、`reflect` |

来源：前五行来自 [OpenClaw 记忆文档](https://docs.openclaw.ai/concepts/memory)（2026 年 10 月 24 日核对）；OpenViking 的 [OpenClaw 集成指南](https://github.com/volcengine/OpenViking/blob/main/docs/en/agent-integrations/03-openclaw.md)；以及各厂商的插件页面。

**PostgreSQL 选项**是最新的内置选择。SQLite 仍然是默认，但 memory-core 也可以索引并搜索 PostgreSQL 里的 agent 数据库。使用 `memory.search.provider: "none"` 即可只做关键词搜索，此时不需要 `vector` 扩展。

**OpenViking** 与其他选项不同。它的插件（`openclaw plugins install clawhub:@openviking/openclaw-plugin`，需要 OpenClaw 2026.5.27 或更新版本，Node.js 22+）占用 OpenClaw 的 `contextEngine` 插槽。每一轮之前，它用服务器端的会话摘要、最近的消息、用户画像和召回的记忆来组装模型的上下文。每一轮之后，它捕获对话，并在待提交 token 超过预算一半时提交到 OpenViking 服务器。它还设置 `ownsCompaction: true`，所以压缩由 OpenViking 而不是 OpenClaw 完成。`peer_role` 设置决定记忆是按用户共享，还是按助手或发送者分开。[OpenViking 页面](/articles/openviking-agent-memory/)介绍服务器本身。

OpenViking 自己在 2026 年 5 月发布的基准文章称，OpenClaw 在 LoCoMo 上从原生记忆的 24.20% 提升到使用插件的 82.08%，输入 token 减少 91.0%。这些是该项目自己的数字：文章没有说明评判模型和回答模型，而且对照组是 OpenClaw 的文件记忆，不是另一个记忆系统。

Mem0（[ClawHub](https://clawhub.ai/mem0/plugins/openclaw-mem0) 上的 `@mem0/openclaw-mem0`）和 Hindsight（`@vectorize-io/hindsight-openclaw`）用基于提取的存储取代了文件搜索。和 OpenViking 一样，它们每轮都会增加 LLM 调用，所以主要在 agent 积累了数月历史、每日笔记和 `MEMORY.md` 已经覆盖不了的时候才值得使用。

对于这种数月历史的情况，[Hindsight](https://github.com/vectorize-io/hindsight) 有最强的已公开数据：在长历史基准 BEAM 上，1M token 时它以 73.9 对 64.1 领先 Mem0，10M token 时以 64.1 对 48.6 领先，数据来自 [Mem0 自己的对比页](https://mem0.ai/compare/mem0-vs-hindsight)（由厂商自己运行）。它也像 dreaming 那样在后台工作：相关事实合并成有证据支持、会被不断完善而不是覆盖的 **observations**。它的插件设置向导（`openclaw plugins install @vectorize-io/hindsight-openclaw`）提供 Hindsight Cloud、你自己的服务器，或嵌入式本地守护进程；后者可以跑在 Ollama 上，让所有内容都留在本机。代价是每次写入都会产生更多 LLM 调用，每次召回返回的上下文也比 Mem0 多。另外，记忆会从你能读写的 Markdown 文件，转移到数据库里。

**OpenClaw 插件的变化（Hindsight v1.2.0，2026 年 11 月 3 日）。** 插件现在给 OpenClaw 提供与 Hermes 插件相同的三个工具：`hindsight_retain`、`hindsight_recall` 和 `hindsight_reflect`，名称和参数一致（[#5965](https://github.com/vectorize-io/hindsight/pull/5965)）。`memoryMode` 设置可以选 `hybrid`（默认：自动召回加工具）、`context`（只自动召回）或 `tools`（只用工具）。新增了七个知识库工具，删除需要设置 `allowKnowledgeDelete: true`（[#6004](https://github.com/vectorize-io/hindsight/pull/6004)）。除非把 `skipOperationalSessions` 设为 `false`，否则 cron、心跳和子 agent 会话不进入记忆（[#5963](https://github.com/vectorize-io/hindsight/pull/5963)）。来源：[Hindsight v1.2.0 发布说明](https://github.com/vectorize-io/hindsight/releases/tag/v1.2.0)，2026-11-05 读取。同一版本的重试改动（保存失败后最多重发 5 次，且只保存在内存中）写在 Hermes 插件的说明里；OpenClaw 插件的重试行为不在这些说明中，依赖之前请先确认。

## OpenClaw 2026.10.10 的变化（2026 年 11 月）

OpenClaw 2026.10.10 于 2026 年 11 月 3 日发布。它的[更新日志](https://github.com/openclaw/openclaw/blob/main/CHANGELOG/2026.10.10.md)中有一大段“Memory and search”。升级后有两处变化可能让记忆表现不同：搜索排序变了，新的对话文本需要更长时间才能被搜到。

| 变化 | 你会注意到什么 | 该怎么做 | 来源 |
|---|---|---|---|
| 混合搜索中，关键词只是部分重合的结果，权重低于语义上更接近的结果 | 结果顺序变化；罕见的 ID 或文件名匹配可能被排到后面 | 如果你设置了最低搜索分数，有用的结果消失时可以调低 | [#177450](https://github.com/openclaw/openclaw/pull/177450)、[#167004](https://github.com/openclaw/openclaw/pull/167004) |
| 严格的正文和文件名搜索找不到结果时，启用自然语言兜底；前几个向量窗口之外的关键词匹配也能得分 | 过去返回空的问题，现在可能找到笔记 | 无需操作 | [#161165](https://github.com/openclaw/openclaw/pull/161165) |
| 相关的较早日期笔记，即使有时间衰减，仍然可以被召回 | 旧的每日笔记会再次出现，不过较新的笔记可能仍排在前面 | 无需操作 | [#168692](https://github.com/openclaw/openclaw/pull/168692) |
| 会话记录在安静一段时间后才建立索引，活跃轮次中不建立；搜索使用已发布的索引 | 刚说的话可能还不在 `memory_search` 里 | 等 agent 空闲，或用 `openclaw memory status` 检查 | [#170404](https://github.com/openclaw/openclaw/pull/170404) |
| 不断增长的记录只对变化的分块重新嵌入和写入；基本未变的来源复用已有向量 | 长会话的重新嵌入工作减少 | 无需操作 | [#103201](https://github.com/openclaw/openclaw/pull/103201)、[#171402](https://github.com/openclaw/openclaw/pull/171402) |
| 失败的重建会保留之前可用的索引；完整重建失败后会按退避策略重试 | 重建出错时搜索仍然可用，不会变成空的 | 修复嵌入错误；`openclaw doctor` 会解释原因 | [#168263](https://github.com/openclaw/openclaw/pull/168263)、[#138786](https://github.com/openclaw/openclaw/pull/138786) |
| 无效、空的或全零的查询嵌入现在会显示错误或 Gateway 警告 | 以前向量匹配会被悄悄丢弃 | 检查嵌入服务，然后重试 | [#168263](https://github.com/openclaw/openclaw/pull/168263)、[#183475](https://github.com/openclaw/openclaw/pull/183475) |
| 向量维度不对的查询嵌入，会报错并给出 agent 专用的强制重建命令 | 更换模型后有明确报错，而不是错误结果 | 运行它打印的重建命令 | [#168465](https://github.com/openclaw/openclaw/pull/168465) |
| 记忆嵌入遵循提供商的私有网络规则，包括重定向 | 放在私有地址上的自托管嵌入服务器可能被拒绝 | 在嵌入提供商的 ID 上明确允许该路由 | [#169847](https://github.com/openclaw/openclaw/pull/169847)、[#170173](https://github.com/openclaw/openclaw/pull/170173) |
| 私有记忆读取会检查角色和管理员权限；所有者会话和跨 agent 子 agent 可以重新访问 | 关联到旧聊天的自动任务可以再次读取记忆 | 无需操作 | [#181359](https://github.com/openclaw/openclaw/pull/181359) |
| Memory Wiki 在重新生成概念页时，会把手写文字保留在有标记的 Notes 部分内 | 旧的生成页上，该部分之外的手工编辑没有受到保护 | 把手工编辑移到 Notes 部分 | [#175707](https://github.com/openclaw/openclaw/pull/175707) |

更新日志谨慎地说，排序变化“不保证每个精确匹配都排在最前”。[关于精确召回的报告](https://github.com/openclaw/openclaw/issues/162764)仍然开着。如果你的 agent 要查找工单号或环境变量名等精确词，升级后请测试这些查询。

### OpenClaw 2026.11.1（2026 年 11 月 5 日）

下一个稳定版 [2026.11.1](https://github.com/openclaw/openclaw/releases/tag/v2026.11.1) 没有新的记忆功能。它修复了长对话如何在压缩后保留，以及大型记录如何被读取。记忆搜索和记忆刷新都依赖这两者。

| 变化 | 你会注意到什么 | 来源 |
|---|---|---|
| 压缩会在多次摘要之间保留仍在处理的人类请求。以前，之后的内部更新（工具结果、后台运行）可能会把它替换掉，或者已完成的工作会重新出现 | 一个长任务经过多次压缩后，agent 仍然知道你最初要求什么 | [#180214](https://github.com/openclaw/openclaw/pull/180214) |
| 记忆索引会等归档的记录完整写入后再建立索引 | 旧会话是用完整文本建立索引，而不是半写的副本 | [发布说明](https://github.com/openclaw/openclaw/releases/tag/v2026.11.1) |
| 很长的记录改为分页读取，不再一次性全部读取（之前在 8 MiB 时失败，“在新会话中恢复”时工作进程内存耗尽） | 大会话可以恢复和加载，不再崩溃 | [#188450](https://github.com/openclaw/openclaw/pull/188450)、[#182091](https://github.com/openclaw/openclaw/pull/182091)、[#190690](https://github.com/openclaw/openclaw/pull/190690) |

这些都不改变排序或保存的内容，所以升级后不需要重新测试。上面 2026.10.10 的搜索变化仍然适用。

## 常用命令和修复方法

```bash
openclaw memory status          # 索引状态和当前提供商
openclaw memory search "query"  # 在命令行中搜索
openclaw memory index --force   # 重建索引
```

如果搜索没有结果，先检查文件是否为空、提供商是否已设置，然后强制重建索引。文档还建议，如果中文文本搜不到，也要重建索引。如果 agent 忽略了你知道在 `MEMORY.md` 里的内容，用 `/context detail` 检查是否被截断。要比较 OpenClaw 和 Nous Research 的 agent，参见 [OpenClaw 与 Hermes Agent 的记忆对比](/articles/openclaw-vs-hermes-agent-memory/)。


## FAQ

### OpenClaw 的记忆是怎么工作的？

OpenClaw 把记忆存成 agent 工作区（默认 ~/.openclaw/workspace）里的普通 Markdown 文件。MEMORY.md 存长期事实，USER.md 存偏好，memory/ 下按日期命名的笔记存每日上下文。MEMORY.md 和 USER.md 在会话开始时加载；每日笔记会被建立索引，由 memory_search 工具查找。

### OpenClaw 记忆默认用哪个嵌入模型？

默认是 OpenAI。你可以用 memory.search.provider 切换到 Gemini、Voyage（voyage-4-large）、Mistral（mistral-embed）、Bedrock、DeepInfra（BAAI/bge-m3）、Ollama、LM Studio、GitHub Copilot、通用的 OpenAI 兼容接口，或大约 0.3 GB 的本地 llama.cpp GGUF 模型。设为 none 则只用关键词搜索。

### OpenClaw 的 memory_search 返回空结果，怎么办？

用 openclaw memory status 检查索引和提供商，确认记忆文件不是空的，然后用 openclaw memory index --force 重建索引。如果嵌入提供商出错，OpenClaw 会退回到只用关键词搜索，所以原文里的词仍然能找到。

### 升级到 2026.10.10 后，OpenClaw 记忆搜索结果为什么变了？

OpenClaw 2026.10.10（2026 年 11 月 3 日发布）改了混合搜索：关键词只是部分重合的结果，权重低于语义上更接近的结果，因此排序会变，罕见的 ID 可能被排到后面。如果你设置了最低搜索分数，可以调低。现在会话记录也会在安静一段时间后才建立索引，所以刚说的话可能暂时搜不到。

### 哪些记忆插件可以用于 OpenClaw？

除了默认的 SQLite memory-core（也可以用 PostgreSQL），OpenClaw 文档列出的还有 Honcho、LanceDB 和自带的 Memory Wiki。Mem0 和 Hindsight 也发布了记忆插件。OpenViking 的插件占用的是 OpenClaw 的 contextEngine 插槽，负责召回、会话摘要和压缩。

