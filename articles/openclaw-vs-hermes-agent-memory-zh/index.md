# OpenClaw 与 Hermes Agent 记忆对比（2026）：文件、上限、迁移

> OpenClaw 和 Hermes Agent 记忆对比：文件结构、字符上限、搜索、整理、外部记忆提供方和迁移。内容来自两个项目的文档，2026 年 10 月核对。

Source: https://aiagentmemory.org/articles/openclaw-vs-hermes-agent-memory-zh/ · Updated 2026-11-11 · aiagentmemory.org


**OpenClaw 与 Hermes Agent 的记忆**本质上是两种思路。**Hermes** 把记忆做得很小、很固定：两个有硬性上限的文件，以冻结快照的形式加载，外加对旧会话的全文搜索。**OpenClaw** 让记忆不设上限：Markdown 文件和每日笔记，支持混合搜索，并由后台任务把最好的笔记提升到 `MEMORY.md`。

两者都是开源（MIT）的 agent，都在你自己的电脑上运行，并通过聊天软件交互。Hermes 还有官方的[桌面应用](/articles/hermes-agent-desktop/)，与命令行共用同一套记忆。[OpenClaw](https://github.com/openclaw/openclaw) 来自 Peter Steinberger 和 OpenClaw Foundation；[Hermes Agent](https://github.com/NousResearch/hermes-agent) 来自 Nous Research。下面的事实来自两个项目的文档，2026 年 10 月核对。

## OpenClaw 和 Hermes 的记忆有什么不同？

**两者都把记忆存成 agent 自己写的纯文本。Hermes 把记忆限制得很死，始终在上下文里约 1,300 个 token，其余内容都当作可搜索的历史。OpenClaw 让记忆以文件的形式增长，启动时只加载整理过的文件，其余内容通过语义和关键词搜索找回。**

两者没有哪个“更好”。Hermes 的做法让提示词保持小巧，也让提示缓存保持稳定。OpenClaw 的做法能存更多内容，代价是需要 embedding 提供方和索引。两者都处在 [AI agent 记忆入门](/articles/ai-agent-memory-explained/) 所讲的光谱的简单一端。

## 并排对比

| | Hermes Agent | OpenClaw | Hindsight（两者都可用的插件） |
|---|---|---|---|
| **始终加载的文件** | `MEMORY.md`（agent 笔记）、`USER.md`（用户资料） | `MEMORY.md`（持久事实）、`USER.md`（可选资料） | 无；记忆库在服务器上，按查询召回 |
| **位置** | `~/.hermes/memories/` | agent 工作区，默认 `~/.openclaw/workspace` | Hindsight Cloud、你自己的服务器，或本地嵌入式守护进程 |
| **大小上限** | 2,200 和 1,375 个字符，由工具强制执行 | 启动预算；超出则注入的副本被截断 | 没有固定上限；召回内容按你设的 `max_tokens` 预算裁剪 |
| **会话中途的更新** | 写入磁盘；提示在下次会话才更新（冻结快照） | 文件写入磁盘；建立索引后搜索才能看到新笔记 | `retain` 存入事实；之后的召回都能看到 |
| **每日或情景笔记** | 无；完整会话存在 SQLite 中 | `memory/YYYY-MM-DD.md` 每日笔记 | 经历和世界事实，带时间和实体 |
| **搜索** | `session_search`：对旧会话做 SQLite FTS5 全文搜索 | `memory_search`：向量 + BM25 混合，带时间衰减、MMR | 语义 + BM25 + 图 + 时间并行搜索，融合后重排序 |
| **需要 embedding 吗** | 不需要 | 可选（默认 OpenAI；可用本地 GGUF） | 需要，由服务器处理 |
| **写入工具** | `memory` 工具：添加、替换、删除 | agent 写 Markdown 文件；`memory_get` 读取 | `retain`：LLM 提取事实、时间、实体和关系 |
| **整理** | 记忆满了时 agent 必须先整理；后台复查 | dreaming 任务把笔记提升到 `MEMORY.md`；压缩前先 flush | 后台整理为观察和心智模型 |
| **写入审批** | `memory.write_approval: true` 会暂存写入 | dreaming 的输出进入 `DREAMS.md` 等待审阅 | 未列出 |
| **安全扫描** | 条目会扫描注入、外泄和不可见 Unicode | 不可信的候选内容不会被提升 | 未列出 |
| **外部提供方** | 同一时间只能用一个：Honcho、[OpenViking](/articles/openviking-agent-memory/)、Mem0、Hindsight、Holographic、RetainDB、ByteRover、Supermemory、[Memori](/articles/llm-memory-comparison/#honcho-memos-and-memori) | 内置、Honcho、LanceDB 引擎；Memory Wiki；第三方插件（Mem0、Hindsight） | 它本身就是提供方：`hermes plugins install hindsight` 或 `@vectorize-io/hindsight-openclaw` |

## Hermes 如何处理记忆

Hermes 把始终在线的部分保持得很小。agent 通过记忆工具添加、替换或删除简短的条目。如果某次写入会超出上限，就返回错误，让 agent 先做整理。[Hermes 文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) 解释了冻结快照：它“保留 LLM 的前缀缓存”，所以系统提示在会话中途不会变化，缓存的 token 也保持便宜。

更大的内容放进会话历史。每个命令行和聊天会话都存进 `~/.hermes/state.db`，`session_search` 用全文搜索找到原始消息。不需要 embedding，也不需要提取模型。Skills 保存 agent 学到的流程，是 agent 自己写和修改的 `SKILL.md` 文件；[Hermes Agent skills](/articles/hermes-agent-skills-zh/) 讲了它们如何创建、加载和修剪。完整说明见 [Hermes Agent 记忆](/articles/hermes-agent-memory-zh/)。

## OpenClaw 如何处理记忆

OpenClaw 写得更多，也找得更多。每日笔记记录正在进行的上下文。`MEMORY.md` 和 `USER.md` 在启动时加载。所有内容都建立了 SQLite 索引，`memory_search` 把向量相似度和 BM25 合并，再乘以 30 天半衰期的时间衰减和重要性分数，并用 MMR 去掉近似重复的内容。

有两个后台步骤让它保持整洁：压缩前的**记忆 flush** 会请 agent 保存重要内容；**dreaming** 会把通过分数、频次和多样性检查的短期笔记提升到 `MEMORY.md`。完整说明见 [OpenClaw AI agent 记忆](/articles/openclaw-ai-agent-memory-zh/)。

## 哪个记得更好？

两个内置系统之间没有公开的正面对比基准，所以任何说某个“记得更好”的说法都只是观点。从设计可以推断的是：

- **提示中的准确近期事实**：差不多。两者都在启动时加载整理过的文件。
- **三周前说过、但措辞不同的内容**：OpenClaw 更有优势，因为向量搜索匹配含义，而 Hermes 的 FTS5 匹配词语。
- **精确的标识符和错误字符串**：两者都可以；FTS5 和 BM25 都能匹配精确的词元。
- **每轮的 token 成本**：Hermes 更低，也更可预测，这是设计上的选择。
- **会话中途看到你刚添加的事实**：Hermes 会在工具响应里立即显示，但要等到 `/new` 之后才会出现在系统提示里。

如果以后不够用，两者都可以接入外部提供方。选哪一个，和选任何记忆层是同一个决定。[LLM 记忆对比](/articles/llm-memory-comparison/) 讲了 Mem0、Honcho、Hindsight、Supermemory 等；[Honcho LLM 记忆](/articles/honcho-llm-memory/) 讲了它 3.4 版本的变化。

## Hindsight 适合哪里

两个内置系统都刻意保持简单。当 agent 运行几个月，历史超出了 2,200 字符的文件或一堆每日笔记时，外部提供方就该接手。**[Hindsight](https://github.com/vectorize-io/hindsight)** 就是其中之一，两个 agent 都能接入：它在 Hermes 插件目录里（`hermes plugins install hindsight`），它的 [OpenClaw 插件](https://hindsight.vectorize.io/sdks/integrations/openclaw) 可以运行本地嵌入式守护进程，复用你已经付费的 LLM。

它比内置系统多出的能力：

- **长历史。** 它报告 LongMemEval 94.6 和 BEAM 10M 64.1；Mem0 自己的对比页给出 Mem0 在 BEAM 10M 上为 48.6。分数均为厂商报告。
- **保留证据的整理。** 相关事实会合并成带原文引用和证明计数的观察，而不是让 agent 手工改写 `MEMORY.md`。
- **四路搜索，而不是一路。** 召回同时进行语义、关键词、图和时间范围搜索，所以“我上周说过的部署内容”不需要原文措辞也能找到。
- **开源、可本地运行。** MIT 许可，所有功能都在开源服务器中，支持 25 家以上 LLM 提供方，包括 Ollama。

### 两个 agent 使用同样的工具（Hindsight v1.2.0）

从 [Hindsight v1.2.0](https://github.com/vectorize-io/hindsight/releases/tag/v1.2.0)（2026 年 11 月 3 日）起，OpenClaw 插件提供与 Hermes 插件相同的三个记忆工具，名称和参数一致（[#5965](https://github.com/vectorize-io/hindsight/pull/5965)）。所以同一个记忆库可以在两个 agent 之间迁移，而不需要改变 agent 使用它的方式。两个插件现在的功能如下：

| | Hermes 插件 | OpenClaw 插件 |
|---|---|---|
| **agent 工具** | `hindsight_retain`、`hindsight_recall`、`hindsight_reflect` | 相同的三个，另加七个知识库工具（浏览、搜索、读取、创建和编辑页面；只有设置 `allowKnowledgeDelete: true` 才能删除）（[#6004](https://github.com/vectorize-io/hindsight/pull/6004)） |
| **工具与自动召回** | `memory_mode` | `memoryMode`：`hybrid`（默认：自动召回加工具）、`context`（只自动召回）、`tools`（只用工具） |
| **后台运行** | Cron 运行、看板工作者和 `delegate_task` 子 agent 不会保存到记忆库（[#6000](https://github.com/vectorize-io/hindsight/pull/6000)） | Cron、心跳和子 agent 会话默认跳过；设置 `skipOperationalSessions: false` 后它们可以使用记忆；会话模式可以把它们设为只读（[#5963](https://github.com/vectorize-io/hindsight/pull/5963)） |
| **保存失败时** | 自动保存的内容会保留并重发，最多尝试 5 次，且不会重复存储；内容保存在内存中，所以服务器宕机时如果进程崩溃，这些内容仍会丢失（[#6013](https://github.com/vectorize-io/hindsight/pull/6013)） | v1.2.0 的更新说明中没有相关变化 |

这里最需要注意的限制是：它比 Hermes 固定的约 1,300 个 token 消耗更多 token。每次 `retain` 都会调用 LLM，而且召回默认返回的上下文比 Mem0 多（按 Mem0 的测量约 4 倍）。如果提示成本很重要，把 `max_tokens` 设低。更多内容见 [Hindsight 详解](/articles/ai-memory-hindsight/)。

## 两者之间如何迁移

两个项目都让切换变得容易：

1. **OpenClaw 到 Hermes**：从 Hermes v0.22.5（2026 年 10 月 22 日）起，导入器是插件，不再属于核心。运行 `hermes plugins install agent-importers`，然后运行 `hermes import-agent openclaw`（插件装好后，旧命令 `hermes claw migrate` 仍然可用）。全新安装时，`hermes setup` 会找到 `~/.openclaw`，询问“Import your setup from OpenClaw?”，并替你安装插件。它总会先显示预览；加上 `--dry-run` 就只停在预览。
2. **Hermes 到 OpenClaw**：在 OpenClaw 控制界面中，打开 **Settings → Import Memory**。它会把 Hermes 的 `MEMORY.md` 和 `USER.md` 复制到 `memory/imports/hermes/`，可以搜索，但不会合并进 OpenClaw 自己的 `MEMORY.md`。

根据 [Hermes 迁移指南](https://hermes-agent.nousresearch.com/docs/guides/migrate-from-openclaw)，OpenClaw 的导入会这样处理记忆：

| OpenClaw | 在 Hermes 中去向 | 方式 |
|---|---|---|
| `workspace/MEMORY.md` | `~/.hermes/memories/MEMORY.md` | 拆分成条目，与已有内容合并，去掉重复项 |
| `workspace/USER.md` | `~/.hermes/memories/USER.md` | 同样的合并方式 |
| `workspace/memory/*.md` 每日笔记 | `~/.hermes/memories/MEMORY.md` | 所有每日文件合并进主记忆 |
| `workspace/SOUL.md` | `~/.hermes/SOUL.md` | 原样复制 |
| Skills（4 个位置） | `~/.hermes/skills/openclaw-imports/` | 默认跳过重名的 skill |
| API keys | `~/.hermes/.env` | 只有加上 `--migrate-secrets` 才会导入 |

在写入任何内容之前，它会把 `~/.hermes/` 打包成 zip，存到 `~/.hermes/backups/`（加上 `--no-backup` 可关闭），所以糟糕的导入可以用 `hermes import` 撤销。

迁移后，请检查哪些内容被带过来了。2,200 字符的 Hermes 记忆可以干净地映射到 OpenClaw。反方向才是问题：所有每日笔记都会合并进一个 `MEMORY.md`，而 Hermes 把这个文件限制在 2,200 字符。请把它们精简成几条长期有效的条目，或者把长历史转到外部提供方（见 [Hindsight 适合哪里](#hindsight-适合哪里)）。

这个插件也能从 Claude Code、Codex CLI 和 Cursor 导入。它们的指令文件（`CLAUDE.md`、`AGENTS.md`、Cursor rules）以及 Codex 的 `memories/*.md` 会变成 Hermes 的记忆条目，而 `hermes import-agent --sync` 会同步之后的改动。

## 如何选择

如果你想要一份一分钟就能读完的记忆、可预测的 token 用量，以及很强的内置 skill 系统，选 **Hermes**。如果你想让 agent 记一本不断增长的日记，并能按含义找回旧笔记，而且你能接受运行 embedding 提供方（或本地模型），选 **OpenClaw**。两者里，记忆都只是指导，不是强制规则；OpenClaw 的文档也明确这么说。对于硬性规则，请使用各 agent 自己的审批和沙箱设置。如果其中任何一个在几个月的使用中忘得太多，就添加 **Hindsight** 作为提供方，而不是换 agent。另外，由于两个 agent 都会把聊天和网页中的内容写入记忆，在给它们访问不可信输入之前，请先阅读 [记忆注入攻击](/articles/ai-memory-injection/)。


## FAQ

### OpenClaw 和 Hermes Agent 的记忆主要区别是什么？

Hermes 的记忆很小，有硬性上限（2,200 + 1,375 个字符），每次会话以冻结快照的形式加载，旧会话用全文搜索查找。OpenClaw 的记忆是不限长度的 Markdown 文件和每日笔记，支持向量和关键词混合搜索，后台的 dreaming 任务会把笔记提升到 MEMORY.md。

### 我能把记忆从 OpenClaw 迁到 Hermes，或者反过来吗？

可以。从 Hermes v0.22.5 起，导入功能是 agent-importers 插件：运行 hermes plugins install agent-importers，然后运行 hermes import-agent openclaw（hermes claw migrate 在装好插件后仍然可用）。它会把 MEMORY.md、USER.md 和所有每日笔记合并进 Hermes 记忆，复制 SOUL.md 和 skills，并且先显示预览。反方向，OpenClaw 的 Settings > Import Memory 会把 Hermes 的 MEMORY.md 和 USER.md 复制到 memory/imports/hermes/，成为可搜索的笔记。

### OpenClaw 和 Hermes 支持 Mem0、Honcho 或 Hindsight 吗？

两者都支持，通过插件实现。Hermes 列出了九个外部提供方，包括 Honcho、Mem0、Hindsight 和 Supermemory，同一时间只能启用一个。OpenClaw 内置 Honcho 和 LanceDB 引擎，Mem0 和 Hindsight 各自发布了 OpenClaw 插件。

### Hindsight 能给 OpenClaw 和 Hermes 同样的记忆工具吗？

能，从 Hindsight v1.2.0（2026 年 11 月 3 日）起。OpenClaw 插件提供 hindsight_retain、hindsight_recall 和 hindsight_reflect，名称和参数与 Hermes 插件相同，还有 memoryMode 设置（hybrid、context 或 tools）和七个知识库工具。Cron、心跳和子 agent 会话默认不进入记忆，除非把 skipOperationalSessions 设为 false。

