# Hermes Agent 技能：程序性记忆如何工作

> Hermes Agent 技能如何工作：agent 如何从经验中写出 SKILL.md 文件、存放位置、加载方式、Skills Hub、curator 和限制。

Source: https://aiagentmemory.org/articles/hermes-agent-skills-zh/ · Updated 2026-11-11 · aiagentmemory.org


**Hermes Agent 技能**是 agent 的程序性记忆：记录怎么完成一项任务的 markdown 文件。Hermes 在完成一项多步骤工作后自己写出技能，遇到新的坑时修改它，任务需要时才加载某一个。每个技能是 `~/.hermes/skills/` 下的一个 `SKILL.md` 文件夹，采用开放的 Agent Skills 格式。

[Hermes Agent](https://github.com/NousResearch/hermes-agent) 是 Nous Research 开发的 MIT 许可证 agent，2026 年 10 月 21 日在 GitHub 上约有 256,000 颗星。本页说明它的技能如何产生、存放、加载、分享和清理。内容来自官方的 [Skills System 文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)、[Curator 文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/curator) 以及 v0.22 发行说明，2026 年 10 月 21 日核对。关于事实文件和会话搜索，见 [Hermes Agent 记忆](/articles/hermes-agent-memory/)。

## Hermes Agent 技能是什么？

**Hermes 技能是一个带 `SKILL.md` 文件的文件夹：开头是 YAML 前置信息（name、description、version、tags），正文写明何时使用、步骤、坑点和如何检查结果。agent 能看到每个技能的名称和描述，只有任务匹配时才读取完整文件。**

文档说得很直接：agent “整理出一个非简单的工作流程时，会把这个方法保存为技能，供以后重复使用”。这就是技能和记忆的区别。记忆说的是什么是真的（“用户在太平洋时区”）。技能说的是怎么做一件事（“如何部署到 Kubernetes”）。Hermes 指南给出的经验法则是：“如果你会把它放进参考文档，它就是技能。如果你会把它写在便利贴上，它就是记忆。”

技能并不是 Hermes 独有的。这种格式遵循 [Agent Skills 规范](https://agentskills.io/specification)，Claude Code、Codex 等工具也读取它，所以一个技能文件夹可以在不同工具之间搬用。Hermes 的不同之处在于围绕它的循环：agent 自己写和改技能，后台任务还会清理没人用的技能。更广义的概念见英文页面 [procedural memory in AI agents](/articles/ai-agent-procedural-memory/)。

## Hermes 如何创建和改进技能

技能有四种产生方式：

1. **任务进行中。** 系统提示要求 agent 用 `skill_manage` 工具保存非简单的工作流程。文档列出三种触发情况：它整理出值得重复的多步骤流程，它碰到死胡同后找到了可行的路径，或者你纠正了它的做法。
2. **提醒。** 默认的 `skills.creation_nudge_interval` 是 15，所以 Hermes 每 15 次工具调用迭代提醒模型考虑保存一个技能（依据仓库中的 `cli-config.yaml.example`，2026 年 10 月）。设为 0 可以关闭提醒。
3. **后台复盘。** 一轮对话结束后，agent 的一个分叉副本可以保存记忆，或创建和修改技能。从 v0.22.0（2026 年 10 月 14 日）起，发行说明写着“后台复盘可以改进任何技能”，纠正的内容会写进该任务对应的技能里。
4. **用 `/learn` 主动要求。** 给它一个文件夹、一个网址、一段粘贴的流程，或者“我刚才怎么部署 staging 服务器的”，agent 就会据此写出技能。像一本书这样的大型资料，会变成一个简短的 `SKILL.md` 加上 `references/` 下每个主题一个文件。

`skill_manage` 有六个操作：`create`、`patch`（定向的查找替换，更新时的首选方式）、带完整 `content` 的 `patch`（整体重写）、`delete`、`write_file` 和 `remove_file`。文档称定向 patch “比整体重写更省 token”。

Hermes 还告诉 agent 好技能应该是什么样：“经验，不是日志。”坑点应该写成一条通用规则加一个简短原因，而不是事件的经过。每次写入都会运行一个提示性的检查器，对三种情况发出警告：正文里有大量 PR 或 issue 编号，参考文件超过 60 个，`SKILL.md` 正文超过约 24,000 个字符。它只发出警告，从不阻止写入。

## 技能存放在哪里，以及 SKILL.md 格式

所有内容都在 `~/.hermes/skills/` 下，文档称之为“主目录和唯一的真实来源”。内置技能在安装时复制到这里，从 hub 安装的技能和 agent 生成的技能也放在这里。技能按分类文件夹组织：

```text
~/.hermes/skills/
├── devops/
│   └── deploy-k8s/
│       ├── SKILL.md        # 必需
│       ├── references/     # 额外文档，按需加载
│       ├── templates/
│       ├── scripts/
│       └── assets/
├── .hub/                   # lock.json、quarantine/、audit.log
├── .usage.json             # 每个技能的查看、使用和修改次数
└── .bundled_manifest       # 已植入的内置技能的哈希值
```

一个最简的 `SKILL.md`，格式与文档中的示例一致：

```markdown
---
name: deploy-k8s
description: Deploy a service to our Kubernetes cluster
version: 1.0.0
platforms: [macos, linux]
metadata:
  hermes:
    tags: [kubernetes, deploy]
    category: devops
    requires_toolsets: [terminal]
---

# Deploy to Kubernetes

## When to Use
## Procedure
## Pitfalls
## Verification
```

Hermes 会从不止一个位置读取技能。名称冲突时，文档给出的顺序是 `项目 → 本地（~/.hermes/skills/）→ skills.create_dir → external_dirs`：

- **项目技能**位于 `<repo>/.hermes/skills/` 或 `<repo>/.agents/skills/`，只有在该仓库中运行 `hermes skills trust` 后才会加载，并且每个技能都会先被扫描。扫描器判定为危险的技能会被隔离。
- **`skills.create_dir`** 可以把 agent 新建的技能送到别的地方，比如一个由 git 管理的文件夹。
- **`skills.external_dirs`** 可以加入 `~/.agents/skills` 这类共享文件夹。文档警告，这些文件夹“不是写保护边界”：只要 Hermes 能写入，agent 就能改它们。

每个 profile 都有自己的技能文件夹和 manifest，所以 `hermes -p coder` 使用一个独立的技能库。

## 技能如何进入上下文

技能通过**渐进式披露**加载，分三级：

| 级别 | 调用 | agent 得到的内容 | 成本 |
|---|---|---|---|
| 0 | `skills_list()` | 每个技能的名称、描述和分类 | 会话开始时约 3k token |
| 1 | `skill_view(name)` | 完整的 `SKILL.md` | 不固定 |
| 2 | `skill_view(name, path)` | 一个参考文件 | 不固定 |

Hermes 文档估计第 0 级的技能列表约为 3,000 token（Skills System 文档，2026 年 10 月核对）。模型读这个列表，决定打开哪个技能。Hermes 没有记录选择时用到任何向量嵌入或排序步骤，因此应把这种选择看作模型自己的判断。

你也可以自己加载技能：

- **斜杠命令。** 每个已安装的技能都是一个命令，例如 `/github-pr-workflow create a PR for the auth refactor`。一条消息里最多可以叠加 5 个开头的 `/skill`。
- **捆绑包。** `~/.hermes/skill-bundles/` 里的一个 YAML 文件可以把多个技能放进一个命令，例如 `/backend-dev`。
- **自动加载。** `config.yaml` 中的 `skills.auto_load` 会在每次会话开始时完整加载指定的技能。
- **条件激活。** 前置信息中的字段，例如 `fallback_for_toolsets: [web]`，会在某个工具存在时隐藏该技能，所以免费搜索技能只在付费的网页工具缺失时才出现。

技能列表是系统提示的一部分，系统提示保持不变，这样服务商的提示缓存才能生效。对话中新增的技能会在你的下一条消息里附带一条简短通知，系统提示本身要到下一次压缩时才会更新。

## Skills Hub：安装和分享技能

**Skills Hub** 是 Hermes 的技能安装器。它会搜索多个注册表，安装时进行安全扫描。在 [v0.23.0](https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0)（2026 年 11 月 5 日）这个版本，Hermes 在 `skills/` 下附带 44 个内置 `SKILL.md`，在 `optional-skills/` 下有 142 个可选技能（按仓库目录树统计），而 10 月 21 日分别是 58 和 153。发行说明称，Word、PowerPoint 和 Excel 合并为一个 `microsoft-office` 技能，12 个小众技能变成可选安装，已有厂商仓库或 MCP 目录覆盖的可选技能被删除。

| 来源 | 示例标识 | 信任级别 |
|---|---|---|
| `official`（Hermes 仓库中的可选技能） | `official/security/1password` | 内置信任 |
| `github`，默认 tap | `openai/skills/k8s`（另有 `anthropics/skills`、`huggingface/skills`、`NVIDIA/skills`） | 受信任 |
| `skills-sh`（Vercel 的目录） | `skills-sh/vercel-labs/json-render/json-render-react` | 社区 |
| `well-known`（网站的 `/.well-known/skills/index.json`） | `well-known:https://mintlify.com/docs/.well-known/skills/mintlify` | 社区 |
| `url`（任何直接的 `SKILL.md` 链接） | `https://example.com/SKILL.md` | 社区 |
| `clawhub`、`lobehub`、`browse-sh` | 视来源而定 | 社区 |

安装和分享一个技能的步骤：

1. 用 `hermes skills search kubernetes` 搜索（聊天中用 `/skills search`）。
2. 用 `hermes skills inspect openai/skills/k8s` 预览。
3. 用 `hermes skills install openai/skills/k8s` 安装。扫描会检查数据外泄、提示注入、破坏性命令和供应链信号。`--force` 可以覆盖警告，但不能覆盖 `dangerous` 的判定。
4. 开始新会话（或 `/reset`），让技能出现在列表中。
5. 之后运行 `hermes skills check` 和 `hermes skills update`。你在本地改过的技能会被跳过，除非加上 `--force`。
6. 要分享自己的技能，就推送一个带 `skills/<name>/SKILL.md` 结构的 GitHub 仓库。别人可以运行 `hermes skills tap add owner/repo`，或者只安装一个技能：`hermes skills install owner/repo/skills/my-workflow`。

Hub 操作通过 GitHub API 进行，未认证的请求每小时只有 60 次，所以如果安装时遇到速率限制，请设置 `GITHUB_TOKEN`（Skills System 文档，2026 年 10 月）。插件也可以附带带命名空间的技能（`plugin:skill`），这些技能在按名称加载之前不会出现在技能列表中，见 [Hermes Agent 插件](/articles/hermes-agent-plugins/)。

## 维护技能：curator、审批和限制

会自己写技能的技能库会越来越大。Curator 文档点明了这个风险：“大量狭窄的近似重复技能，会污染目录并浪费 token。”Hermes 有四种控制手段。

**Curator** 是一个后台流程，在距离上次运行已过 7 天、且 agent 空闲 2 小时后运行（Curator 文档，2026 年 10 月）。默认情况下它只做清理：agent 创建的技能如果 14 天没人用，会被标记为陈旧；30 天没人用，会被移到 `~/.hermes/skills/.archive/`。v0.22.0 发行说明（2026 年 10 月 14 日）写道，未使用的技能现在 30 天后归档，之前是 90 天。还有一个用 LLM 把相近技能合并成总技能的步骤，但默认关闭（`curator.consolidate: false`）。文档说完整跑一次“通常需要 50–100 次 API 调用”。Curator 从不直接删除，会为每次改动保留记录，`hermes curator rollback` 可以撤销一次运行或单次修改。

**写入审批。** 设置 `skills.write_approval: true` 后，每次技能的创建、修改或删除都会先暂存在 `~/.hermes/pending/skills/`，然后用 `/skills pending`、`/skills diff <id>` 和 `/skills approve <id>` 审核。这个选项默认关闭。文档建议“容易误判自己学到了什么的小模型”开启它。另有一个独立的内容扫描器 `skills.guard_agent_created`，同样默认关闭。

**固定。** `hermes curator pin <skill>` 会阻止 curator 和 agent 删除该技能，修改仍然可以进行。

**锁定**（v0.22.6，2026 年 11 月 2 日）。`hermes curator lock <skill>` 会阻止 agent 修改该技能，`hermes curator unlock <skill>` 可以撤销。根据 [PR #147557](https://github.com/NousResearch/hermes-agent/pull/147557)，被锁定的技能会拒绝所有 agent 的写入途径，包括普通对话轮次、cron、子 agent、复盘分叉和 curator：`skill_manage`、对其文件夹内文件的 `write_file` 和 `patch`，以及会写入该处的 shell 命令（重定向、`cp`/`mv`/`rm`、`sed -i`、`git`、内联 `python -c`），即使加了 `--yolo` 也一样。agent 不能自己运行 `unlock`，curator 归档时也会跳过它。读取、加载和运行该技能自己的脚本仍然可以，你自己的修改（编辑器、Desktop 技能编辑器、CLI）从不受阻。如果你有一个写好且信任的手写技能，这就是针对下面“质量漂移”限制的修复方法。

| 控制手段 | 阻止 agent 删除 | 阻止 agent 修改 | 默认 |
|---|---|---|---|
| 写入审批 | 直到你批准 | 直到你批准 | 关闭 |
| 固定 | 是 | 否 | 关闭 |
| 锁定 | 是 | 是，所有工具 | 关闭 |

值得了解的限制：

- **上下文成本。** 技能列表每次会话约占 3k token，加载过的 `SKILL.md` 会在会话剩余时间里一直留在上下文中。检查器会对超过约 24,000 个字符的正文发出警告，原因就在这里。
- **质量漂移。** agent 会修改技能，包括你写的或安装的技能。一次会话中的错误经验，可能会写进每次之后都要执行该任务时所依据的技能。如果这一点很重要，就开启写入审批。
- **复盘成本。** 后台复盘会重放整段对话。除非你设置 `max_input_tokens`，它的输入预算是复盘模型上下文窗口的 75%，上限为 600,000 token（memory 文档，2026 年 10 月）。你可以把它交给更便宜的模型，或直接关闭。
- **Curator 不管大多数技能。** 它只管理被标记为 agent 创建的技能，目前就是后台复盘生成的技能。你或前台 agent 创建的技能（包括 `/learn` 生成的）不会被管，除非你运行 `hermes curator adopt`。
- **文档在一个数字上有出入。** Curator 页面的配置里写的是 14 天和 30 天，但页面下面又写“30 天未使用 → 陈旧 → 90 天未使用 → 归档”。配置默认值和 v0.22.0 发行说明都写的是 30 天后归档。

因为技能是 agent 要遵循的指令，被投毒的技能就是一条攻击路径。Hermes 的安装扫描和项目信任步骤就是为此设计的。更广泛的风险见 [AI 记忆注入](/articles/ai-memory-injection/)。

## 技能与其他记忆层

Hermes 按用途划分记忆。技能是四层之一：

| 层 | 存放内容 | 何时加载 | 谁写入 |
|---|---|---|---|
| `MEMORY.md` / `USER.md` | 关于环境和你的简短事实（分别为 2,200 和 1,375 个字符） | 每次会话，固定快照 | agent，通过 `memory` 工具 |
| 会话搜索 | SQLite 中完整的历史会话 | agent 调用 `session_search` 时 | 自动 |
| **技能** | 流程：步骤、命令、坑点 | 开始时加载列表，需要时加载完整文件 | agent、后台复盘、`/learn`、你、hub |
| 外部服务商 | 来自某个插件的长期记忆（Honcho、Mem0、Hindsight 等） | 每轮对话前预取 | 服务商 |

同一个后台复盘同时写入记忆和技能，文档称之为“自我改进循环”：“记忆存放应始终在上下文中的小而持久的事实，而技能存放只在相关时才加载的较长流程。”外部记忆服务商是否会看到技能内容，文档没有说明，因此应假设它看不到。

其他系统学习流程的方式不同。LangMem 的提示优化器会重写系统提示本身，它的目标是“根据对话日志或 agent 的执行轨迹更新系统提示”（[LangMem 文档](https://langchain-ai.github.io/langmem/guides/optimize_memory_prompt/)）。这会改变每一次调用，而 Hermes 技能只在相关时才加载。Claude Code、Codex 和其他 Agent Skills 客户端读取同样的 `SKILL.md` 格式，所以为一个工具写的技能，通常可以直接放进另一个工具的技能文件夹，见 [Claude Code 记忆](/articles/claude-code-memory/)。

**[Hindsight](https://github.com/vectorize-io/hindsight)** 属于外部服务商这一层，不属于技能层。它用 `hermes plugins install hindsight` 安装，不会写入 `SKILL.md` 文件。它与技能相比的新增之处，是不需要有人写成流程的已学知识：它把相关事实合并成 **observations**（带引文和证据计数的、有依据的信念），维护 **mental models**（它在后台刷新的标准答案，读取时不调用 LLM），并用 `reflect` 回答需要推理的问题。这适合那些不属于某一个技能的经验，比如用户希望怎样审核所有项目中的 PR。它是 MIT 许可证，自托管服务器包含全部功能。限制是：每次 `retain` 都要调用一次 LLM，而 Hermes 每轮都会预取召回内容。Hermes 插件默认把召回限制在 4,096 个 token（`recall_max_tokens`，依据 Hindsight 的 Hermes 文档，2026 年 10 月）；如果提示成本重要，就把它调低。更多内容见 [Hindsight 详解](/articles/ai-memory-hindsight/)，服务商对比见 [OpenClaw 与 Hermes Agent 记忆对比](/articles/openclaw-vs-hermes-agent-memory/)。


## FAQ

### Hermes Agent 怎么创建技能？

Hermes 用 skill_manage 工具保存技能。系统提示会要求它记录非简单的工作流程，默认每 15 次工具调用迭代提醒一次，一轮对话结束后的后台复盘可以创建或修改技能。你也可以用 /learn 让它生成一个。新技能是 ~/.hermes/skills/ 下的 SKILL.md 文件。

### 怎么让 Hermes Agent 不再修改某个技能？

从 v0.22.6（2026 年 11 月 2 日）起，运行 hermes curator lock <skill>。之后 agent 不能通过 skill_manage、write_file、patch 或 shell 命令修改这个技能，即使加了 --yolo 也一样，而且 agent 自己无法解锁。你自己的修改仍然有效。hermes curator pin 只阻止删除，skills.write_approval 会把 agent 的每次修改暂存起来等你审核。

### Hermes Agent 的技能存在哪里？

存在 ~/.hermes/skills/ 里，每个技能一个文件夹，必须有 SKILL.md，还可以有可选的 references/、templates/、scripts/、examples/ 和 assets/。在受信任的 git 仓库中，Hermes 还会读取 .hermes/skills/ 或 .agents/skills/ 里的项目技能，以及 config.yaml 中 skills.external_dirs 列出的文件夹。

### Hermes 技能和 Hermes 记忆有什么区别？

记忆（MEMORY.md 和 USER.md）存几条简短的事实，每次会话都会加载。技能存的是较长的流程，也就是怎么做一件事。开始时只加载技能列表，任务需要时才加载完整的 SKILL.md。Hermes 文档把技能称为 agent 的程序性记忆。

