# Claude Code Plan Mode: How to Turn It On and Exit (2026)

> Claude Code plan mode: Claude reads and plans but does not edit. Four ways to turn it on, how to approve a plan and leave. From the official docs, read 2026-11-28.

Source: https://aiagentmemory.org/articles/claude-code-plan-mode/ · Updated 2026-11-28 · aiagentmemory.org


## What is plan mode?

**Plan mode** is a Claude Code permission mode where Claude researches and proposes changes without making them. It reads files and runs shell commands to explore, then writes a plan. It does not edit your source until you approve that plan. Use it when a change touches many files and you want to review the approach first.

## How to turn plan mode on

Claude Code docs list these ways to enter plan mode:

| Method | How | Scope |
|---|---|---|
| Shift+Tab | Press Shift+Tab until the status bar shows `⏸ plan mode on` | Current session |
| /plan | Start one prompt with `/plan` | One prompt |
| Command-line flag | `claude --permission-mode plan`; the same flag works with `-p` | One run |
| Project setting | Set `permissions.defaultMode` to `plan` in `.claude/settings.json` | Every session in that project |

In VS Code, the setting `claudeCode.initialPermissionMode` can start sessions in plan mode.

## How to approve a plan and leave

When the plan is ready, Claude Code shows three options:

1. **Yes, and use auto mode**: approve and switch to auto mode.
2. **Yes, manually approve edits**: approve and switch to the default mode, where you confirm each edit.
3. **No, keep planning**: stay in plan mode.

Press `Ctrl+G` to open the plan in your text editor before you choose. To leave plan mode without a plan, press Shift+Tab again when no approval prompt is open.

## Permission modes

The permission modes named in the docs are `default` (also called `manual`), `acceptEdits`, `plan`, `auto`, `dontAsk` and `bypassPermissions`. Plan mode is the one that holds all edits until you approve a plan.

One limit: in an interactive terminal session with bypass permissions available, a file edit or shell command that Claude tries during planning can run without a prompt. The docs call this out. Plan mode keeps its blocks in `-p` runs, Agent SDK sessions and the VS Code chat panel.

## Memory and plan mode

Plan mode changes what Claude may do, not what it remembers. A plan lives for the task. Rules that should hold across sessions belong in `CLAUDE.md`; see [Claude Code commands](/articles/claude-code-commands/) for `/memory` and `/init`.

If you want decisions from past sessions to come back, a memory layer is needed. [Hindsight](https://github.com/vectorize-io/hindsight) ships one Coding Agents package, `@vectorize-io/hindsight-coding-agents`, that covers 22 CLI agents including Claude Code, Codex CLI and Cursor. Its limit: it returns more context per retrieval by default, so set `max_tokens` lower if your sessions are short. For the rest of the coding-agent options, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

## Sources

- Claude Code permission modes: https://code.claude.com/docs/en/permission-modes (read 2026-11-28)
- Claude Code common workflows: https://code.claude.com/docs/en/common-workflows (read 2026-11-28)
- Hindsight coding agents integration: https://hindsight.vectorize.io/sdks/integrations/coding-agents (read 2026-11-12)


## FAQ

### What is plan mode in Claude Code?

Plan mode is a permission mode where Claude reads files, runs shell commands to explore, and writes a plan, but does not edit your source. Edits stay blocked until you approve the plan. The Claude Code permission modes docs describe it this way.

### How do I turn on plan mode in Claude Code?

Press Shift+Tab until the status bar shows plan mode on, start one prompt with /plan, or start with claude --permission-mode plan. You can also set permissions.defaultMode to plan in a project's .claude/settings.json.

### How do I leave plan mode in Claude Code?

With no approval prompt open, press Shift+Tab again. At the approval prompt, press Esc first, because Shift+Tab there approves the plan.

