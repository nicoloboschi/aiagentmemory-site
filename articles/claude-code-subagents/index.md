# Claude Code Subagents: Create, Scope and Memory (2026)

> How Claude Code subagents work: where to store one (project, user or plugin), the fields that matter, how to call one, and the memory field that keeps notes across sessions. From the official docs, read 2026-11-14.

Source: https://aiagentmemory.org/articles/claude-code-subagents/ · Updated 2026-11-14 · aiagentmemory.org


## What is a Claude Code subagent?

A **Claude Code subagent** is a helper that the main session starts to do one kind of task. It runs in its own context window, with its own system prompt, its own tool list and its own permissions. When it finishes, only its summary returns to the main conversation.

Use one when a side task would fill your main chat with output you will not need again, such as long search results or log files. A subagent that is not a fork starts fresh. It does not see your chat history or the files Claude already read.

A subagent is not a skill. A [skill](/articles/claude-code-skills/) adds instructions to the main conversation. A [plugin](/articles/claude-code-plugins/) can ship subagents, along with skills and hooks, in one install.

## Which subagents come built in?

Claude Code ships with a few subagents that Claude uses on its own. You do not have to create them.

| Agent | What it does | Tools |
|---|---|---|
| Explore | Searches and reads code. Asked for a quick, medium or very thorough pass. | Read-only; Write and Edit denied |
| Plan | Gathers facts for a plan while you are in plan mode. | Read-only; Write and Edit denied |
| general-purpose | Handles multi-step work that both explores and changes code. | Every tool available to subagents |
| claude-code-guide, statusline-setup | Answers questions about Claude Code features, and sets up the status line. | Built-in helpers |

Explore and Plan start without your CLAUDE.md files and without the git status snapshot, which keeps them fast. Custom subagents load both, unless you set `omitClaudeMd`.

## Where should a subagent file live?

Claude Code reads subagent files from five places. When two share a name, the one with the higher priority wins.

| Location | Scope | Priority |
|---|---|---|
| Managed settings | Whole organization | 1 (highest) |
| `--agents` flag | This session only, not saved to disk | 2 |
| `.claude/agents/` | This project, shared through version control | 3 |
| `~/.claude/agents/` | All your projects | 4 |
| Plugin `agents/` folder | Wherever the plugin is enabled | 5 (lowest) |

Project subagents are the usual choice for a team. Claude Code scans subfolders too, but identity comes only from the `name` field, not from the folder. Keep names unique across the tree, because two files with the same name can load in an order you cannot count on.

## How do you create one?

1. Ask Claude to write one, or create the file by hand in `.claude/agents/` or `~/.claude/agents/`. A subagent file is Markdown with a short YAML header. If Claude cannot find a new file, restart Claude Code: a running session does not notice a folder created after it started.
2. Set `name` and `description`. These are the only required fields. The description tells Claude when to hand off a task, so write it as a trigger.
3. Add the tools it may use in `tools`. If you leave it out, the subagent inherits every tool available to subagents.
4. Pick a `model`: `haiku`, `sonnet`, `opus`, `fable`, a full model ID, or `inherit`. If you leave it out, the subagent runs on the model of the conversation that starts it.
5. Write the system prompt in the body, below the closing `---`. Say what to check and how to format the output.
6. Start a new task and check that Claude calls it. Field names must match exactly, because Claude Code ignores a field it does not recognize without an error. Run `claude --debug` and search the log for `does not recognize or apply` to find a typo.

A minimal read-only reviewer looks like this:

```markdown
---
name: code-reviewer
description: Reviews code for quality, security and missing tests. Use right after code changes.
tools: Read, Grep, Glob, Bash
model: inherit
---

You are a senior code reviewer. Run git diff, review the changed files,
and group your findings by priority.
```

The reviewer has no Edit or Write tool, so it cannot change the code it reviews.

## Which fields matter most?

| Field | What it controls |
|---|---|
| `tools` / `disallowedTools` | Which tools it may use, or which to remove |
| `permissionMode` | How it asks for permission (`default`, `acceptEdits`, `plan` and others) |
| `maxTurns` | The turn limit. At the limit, the output comes back marked as partial. |
| `skills` | Skills whose full text loads into its context at start |
| `isolation: worktree` | Runs it in a temporary git worktree, an isolated copy of the repository, cleaned up if it makes no changes |
| `memory` | A notes folder that survives across sessions (see below) |
| `background` | Keeps it running in the background when Claude would otherwise wait |

Plugin subagents ignore `hooks`, `mcpServers` and `permissionMode`, for security. If you need those, copy the file into `.claude/agents/` or `~/.claude/agents/`.

## How do you call a subagent?

- **Let Claude decide.** Name it in your prompt: "Use the code-reviewer subagent to check my changes." Claude matches the request against each `description`.
- **@-mention it.** Type `@` and pick it from the list. This guarantees that subagent runs for the task, but your full message still goes to it.
- **Run the whole session as it.** Start with `claude --agent code-reviewer`. The main thread then takes on its tool limits and model.

Claude Code caps nesting. A subagent can start subagents up to three layers below the main conversation by default. It also caps concurrent subagents at 20 by default.

## What is the memory field?

The `memory` field gives a subagent a folder that survives across sessions. Pick a scope:

- `user`: `~/.claude/agent-memory/<agent-name>/`, for learnings you want in every project.
- `project`: `.claude/agent-memory/<agent-name>/`, shared through version control. Anthropic's docs call this the recommended default.
- `local`: `.claude/agent-memory-local/<agent-name>/`, for one project, kept out of version control.

When memory is on, Read, Write and Edit are turned on for the subagent, and its prompt gets the first 200 lines or 25 KB of its `MEMORY.md`. If auto memory is off in your settings, the field has no effect.

This is a set of plain files that the subagent edits. It stores what the subagent writes down, and nothing else. You can ask it to check its memory before a task and to save what it learned after. For the main session's own memory, see [Claude Code memory](/articles/claude-code-memory/). For the Codex equivalent, see [Codex subagents](/articles/codex-subagents/).

## Where does Hindsight fit?

The `memory` field and Hindsight solve different problems:

- **The `memory` field** is a folder of notes that one subagent writes and reads. Claude Code loads the start of its `MEMORY.md` into the prompt.
- **[Hindsight](/articles/ai-memory-hindsight/)** is a memory server. Its **Coding Agents** package (one npm install, `npx @vectorize-io/hindsight-coding-agents install claude-code`) covers 22 CLI agents, Claude Code among them, and captures sessions automatically, with one memory bank per repository. The old per-agent Claude Code plugin is retired.
- **What the Coding Agents page does not say:** the page I read on 2026-11-14 does not explain how Claude Code subagent runs are captured or recalled. Do not assume a subagent sees the same memory as the main session. Test it: ask the subagent a question that only a past session can answer.
- **Strength:** Hindsight reports 64.1 on BEAM 10M against Mem0's 48.6. The BEAM figures come from Mem0's own comparison page, not from an independent test, so treat them as the vendors' numbers.
- **Limit:** Hindsight returns more context per recall than Mem0 by default, and each save calls an LLM. Lower `max_tokens` if recall is too large. For a short, single-task subagent, its own notebook may be enough.

## Sources

- Claude Code subagent docs, code.claude.com/docs/en/sub-agents (read 2026-11-14): built-in agents, scopes, frontmatter fields, the memory field, model order, concurrency and depth limits, and the examples.
- Hindsight Coding Agents docs, hindsight.vectorize.io/sdks/integrations/coding-agents (read 2026-11-14): the install command and the one-bank-per-repository rule. The page does not describe subagent capture.
- Benchmark figures (BEAM 10M, 64.1 and 48.6): Mem0's comparison page, not Hindsight's own report.


## FAQ

### What is a Claude Code subagent?

A subagent is a helper that Claude Code starts in its own context window, with its own system prompt and tool list. It does the side task and returns a summary, so the search results and file reads stay out of your main conversation. Its requests count toward the same usage limits.

### Where do I put a Claude Code subagent file?

Put it in .claude/agents/ in a project to share it through version control, or in ~/.claude/agents/ to use it in every project. A subagent needs a name and a description in its frontmatter. Only those two fields are required.

### Does a Claude Code subagent remember things between sessions?

Only if you set the memory field to user, project or local. Claude Code then gives it a folder for notes, and the first 200 lines or 25 KB of its MEMORY.md load into its prompt. This is a file-based notebook, not a database, and it is separate from Hindsight.

### How many subagents can run at once in Claude Code?

By default 20 subagents can run at the same time in one session. Spawning another fails until one finishes. Set CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS to change the limit. The limit does not apply to sessions with ultracode active.

