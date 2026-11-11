# OpenClaw Skills（2026）：从 ClawHub 安装、编写 SKILL.md、安全注意事项

> 如何查找、安装、更新和编写 OpenClaw skills：openclaw skills 命令、SKILL.md 格式和元数据字段，以及安装第三方 skill 前要检查什么。内容来自 OpenClaw ClawHub 和 skill 格式文档，读取日期 2026-11-11。

Source: https://aiagentmemory.org/articles/openclaw-skills-zh/ · Updated 2026-11-11 · aiagentmemory.org


**OpenClaw skills** 是一些文件夹，里面放着教 agent 完成某项任务的指令，例如日历、Todoist 客户端或数据库备份。你可以从公共注册表 ClawHub 安装，也可以自己编写。skill 不是记忆：它不会保存 agent 学到的内容。关于记忆，请看 [OpenClaw 记忆](/articles/openclaw-ai-agent-memory-zh/) 和 [OpenClaw 指南](/articles/openclaw-guide-zh/)。

来源：OpenClaw ClawHub 文档（documentation.openclaw.ai/clawhub）和 skill 格式页面（documentation.openclaw.ai/clawhub/skill-format）。两者都在 2026-11-11 读取。

## 命令

| 你想做什么 | 命令 | 文档中的说明 |
|---|---|---|
| 查找 skill | `openclaw skills search "calendar"` | 在 ClawHub 中搜索 |
| 安装一个 | `openclaw skills install @openclaw/demo` | 记录来源，之后的更新仍然走 ClawHub |
| 更新全部 | `openclaw skills update --all` | OpenClaw 原生命令 |
| 发布或删除 | `clawhub` CLI（`npm i -g clawhub`） | 需要先运行 `clawhub login`。用于发布、删除和恢复 |
| 安装插件（不是 skill） | `openclaw plugins install clawhub:<package>` | 插件和 skill 是两回事 |

文档说，原生 `openclaw` 命令会安装到当前 OpenClaw 工作区。单独的 `clawhub` CLI 会安装到你运行命令的文件夹下的 `./skills`，并写入 `.clawhub/lock.json`。每个 skill 只用其中一种方式安装，这样你才知道它存在哪里。

## skill 文件夹里有什么

| 组成部分 | 是否必需 | 作用 |
|---|---|---|
| `SKILL.md` | 是 | 带可选 YAML frontmatter 的 Markdown。agent 读取的指令 |
| 其他文件 | 否 | 文件夹里的任何普通文件，例如脚本或模板。打包总大小上限 50 MB |
| `.clawhubignore` 或 `.gitignore` | 否 | 发布时排除的文件 |

文档也接受 `skill.md`；旧的 `skills.md` 仍然可用。对于可移植的 Agent Skills，`name` 应与文件夹名一致，使用 1 到 64 个小写字母、数字或连字符。

## 最重要的 frontmatter 字段

```yaml
---
name: todoist
description: Manage tasks via the Todoist API.
version: 1.0.0
metadata:
  openclaw:
    requires:
      env:
        - TODOIST_API_KEY
      bins:
        - curl
    primaryEnv: TODOIST_API_KEY
    emoji: "✅"
    homepage: https://github.com/example/todoist-cli
---
```

- `description` 是注册表和搜索中的摘要。
- `requires.env` 列出 skill 不能没有的变量。可选变量应放在 `envVars` 下并设置 `required: false`，不要放进 `requires.env`。
- `requires.bins` 列出必须全部存在的命令行工具。`requires.anyBins` 表示至少存在其中一个即可。
- `install` 可以用 `brew`、`node`、`go` 或 `uv` 安装依赖。
- `os` 把 skill 限定在 `macos` 或 `linux` 等系统上。

文档说，ClawHub 的安全分析会检查 skill 声明的内容是否与它实际使用的内容一致。如果代码读取了某个 API 密钥，但 frontmatter 没有声明它，分析会标记出来。

## 安装 skill 之前

1. 阅读它的 `SKILL.md`。注意 shell 命令、网络请求，以及它要求你设置的任何变量。
2. 查看 ClawHub 页面上公开的扫描摘要。ClawHub 会对已发布的 skill 运行自动检查。被扫描拦截或阻止的版本可能从目录中消失。第三方指南提醒，"Pass" 状态并不是保证。
3. 只给它需要的密钥。先用测试密钥。

ClawHub 默认开放：只要 GitHub 账号够旧、能通过上传检查，任何人都能发布。请把 skill 当作从互联网下载的脚本来对待。

## 编写自己的 skill

1. 新建一个以 skill 命名的文件夹，里面放一个 `SKILL.md`。
2. 添加 `name` 和 `description`。把每个变量和命令行工具都写进 `metadata.openclaw.requires`。
3. 发布前先在聊天会话中测试。
4. 用 `clawhub skill publish <path> --slug <slug> --name "<name>" --version 1.0.0` 发布。

每次发布都会生成一个新的 semver 版本。skill 使用 MIT-0 许可证。ClawHub 不支持收费 skill。

## Hindsight 的位置

skill 可以调用记忆工具，但它本身不带来记忆。Hindsight 是单独的插件，即 `@vectorize-io/hindsight-openclaw`。从 v1.2.0（2026-11-03 发布）起，它为 OpenClaw 提供 `hindsight_retain`、`hindsight_recall` 和 `hindsight_reflect` 工具，并有一个 `memoryMode` 设置：`hybrid`（默认）、`context` 或 `tools`。

- **最有出处的数字：** 在 10M tokens 的 BEAM 基准上，Hindsight 得分 64.1，Mem0 得分 48.6。这个对比是 Mem0 在自己的网站上做的，因此属于厂商自测。
- **开源部分：** 完整服务端是 MIT 许可，包括图谱召回和 reflect。
- **它的限制：** 默认情况下，一次检索返回的上下文比 Mem0 多（可以把 `max_tokens` 调低）。每次保存都要调用 LLM，写入成本更高。保存失败的内容只保存在内存中，服务器宕机时如果进程崩溃就会丢失。（这一条的来源是 Hermes 插件的 v1.2.0 说明；依赖之前请先确认 OpenClaw 插件的情况。）
- **skill 的风险是另一回事：** 读取你记忆的第三方 skill，又多了一项需要审查的东西。

只有当文件记忆不够用时，才安装 Hindsight。先阅读 [OpenClaw 记忆页面](/articles/openclaw-ai-agent-memory-zh/)。

## 常见问题

**如何安装 OpenClaw skill？** 用 `openclaw skills search` 搜索，然后运行 `openclaw skills install @owner/<slug>`。用 `openclaw skills update --all` 更新全部。

**SKILL.md 文件是什么？** 放在 skill 文件夹顶部的指令文件。它的 frontmatter 写明 skill 的名称、摘要和所需变量。

**从 ClawHub 安装 skill 安全吗？** 先读文件和扫描结果。把第三方 skill 当作不可信的代码。

**skill 能给 agent 记忆吗？** 不能。请使用工作区里的记忆文件，或 Hindsight 这类记忆插件。

2026-11-11 对照 OpenClaw ClawHub 和 skill 格式文档检查。命令可能随 OpenClaw 版本变化；运行 `openclaw skills --help` 核对你的版本。


## FAQ

### 如何安装 OpenClaw skill？

运行 openclaw skills search "<主题>" 查找，然后运行 openclaw skills install @owner/<slug> 安装。运行 openclaw skills update --all 更新全部 skill。这些命令来自 OpenClaw ClawHub 文档，读取日期 2026-11-11。

### OpenClaw 里的 SKILL.md 文件是什么？

skill 是一个包含 SKILL.md 文件的文件夹。SKILL.md 是带可选 YAML frontmatter 的 Markdown。description 字段是在注册表和搜索中显示的摘要。frontmatter 还可以列出 skill 需要的环境变量、命令行工具和配置文件。

### 从 ClawHub 安装 skill 安全吗？

把第三方 skill 当作不可信的内容。ClawHub 默认开放，任何人都可以发布。安装前请阅读 SKILL.md 并查看公开的扫描结果。第三方指南提醒，"Pass" 状态并不是保证。请自己检查扫描详情。

### OpenClaw skill 会给 agent 记忆吗？

本身不会。skill 是一组指令和工具。OpenClaw 的记忆是工作区里的 Markdown 文件，或者 Hindsight 这类记忆插件，而插件需要单独安装。

