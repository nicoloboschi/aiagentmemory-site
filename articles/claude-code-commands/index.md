# Claude Code Commands: Memory, Context and Session (2026)

> Claude Code slash commands for memory, context and sessions: /init, /memory, /compact, /clear and /resume. What each does, from the official docs read 2026-11-15.

Source: https://aiagentmemory.org/articles/claude-code-commands/ · Updated 2026-11-15 · aiagentmemory.org


## What are Claude Code commands?

**Claude Code commands** are the words you type after a `/` inside a session, such as `/memory` or `/compact`. They control the session: they switch models, edit memory files, summarize or clear the chat, and run workflows. A command is only recognized at the start of your message. Type `/` in a session to see the commands available to you, since some depend on your plan and platform.

The official commands page lists 121 rows (read 2026-11-15). This page covers the ones that affect what Claude remembers, how full the context window gets, and how sessions are kept.

## Which commands handle memory and context?

| Command | What it does for memory |
| :- | :- |
| `/init` | Writes a starter `CLAUDE.md` for the project. Set `CLAUDE_CODE_NEW_INIT=1` for an interactive flow that also covers skills, hooks and personal memory files. |
| `/memory` | Opens your `CLAUDE.md` files to edit, turns auto memory on or off, and shows the auto memory entries. |
| `/context [all]` | Shows how full the context window is, by category. It lists which `CLAUDE.md` and auto memory files loaded and suggests cuts for memory bloat. |
| `/compact [instructions]` | Summarizes the chat to free space. Optional text tells the summary what to focus on. |
| `/clear [name]` | Starts a new conversation with empty context. The docs say it keeps project memory. Aliases: `/reset`, `/new`. |
| `/rewind` | Rolls the code and the chat back to an earlier point, or summarizes from a chosen message. Aliases: `/checkpoint`, `/undo`. |

## Which commands manage sessions?

| Command | What it does |
| :- | :- |
| `/resume [session]` | Reopens an earlier conversation by ID or name, or opens the session picker. Alias: `/continue`. |
| `/branch [name]` | Branches the chat so you can try another direction. The original stays; go back to it with `/resume`. |
| `/fork [prompt]` | Copies the chat into a new background session and keeps you here. |
| `/rename [name]` | Names the current session. |
| `/export [filename]` | Saves the conversation as plain text. |
| `/btw [question]` | Asks a side question without adding it to the conversation history. |

## Which commands run helpers and settings?

| Command | What it does |
| :- | :- |
| `/agents` | Reminds you to ask Claude to create or manage subagents, or to edit the agent folders directly. |
| `/subtask <task>` | Starts a forked subagent that inherits the chat. Its result comes back into this conversation. |
| `/tasks` | Lists background work, including subagents that have finished. |
| `/skills` | Lists the available skills. `/reload-skills` re-scans the skill folders without a restart. |
| `/hooks` | Shows the hook configurations. |
| `/permissions` | Manages allow, ask and deny rules for tools. |
| `/config [key=value]` | Opens Settings, or sets one value, such as `/config model=sonnet`. Alias: `/settings`. |
| `/mcp` | Manages MCP server connections and their sign-in. |
| `/plugin` | Manages plugins. |
| `/usage` | Shows session cost and plan limits. Aliases: `/cost`, `/stats`. |
| `/doctor` | Checks your setup, including `CLAUDE.md` problems, and proposes fixes that Claude applies after you confirm. Alias: `/checkup`. |

## What happens to memory on /compact and /clear?

The context window docs say what survives a compaction (read 2026-11-15). The system prompt still applies. Project-root `CLAUDE.md` and auto memory are read from disk again. Skill descriptions are rebuilt from your current skills. The plan from plan mode is read back in. To turn plan mode on and off, see [Claude Code plan mode](/articles/claude-code-plan-mode/).

The chat summary replaces the old messages. So a detail that you said only in chat stays only as far as the summary keeps it. Write lasting facts into `CLAUDE.md` or auto memory, not into the chat. The commands page says `/clear` keeps project memory. It does not say how much of a cleared chat a later session can see, so test that yourself.

## Where Hindsight fits

Claude Code's own memory is a set of plain files. [Hindsight](https://github.com/vectorize-io/hindsight) is an open-source memory server that can sit beside them. Its Coding Agents package covers Claude Code and 21 other CLI agents in one npm install: `npx @vectorize-io/hindsight-coding-agents install claude-code`. Capture is automatic, so no command is needed, and each repo gets one memory bank (read on Hindsight's coding agents page, 2026-11-12).

- **Long histories:** on BEAM 10M, Hindsight scored 64.1 and Mem0 scored 48.6. Both figures come from Mem0's comparison page, and they are vendor numbers, not an independent test.
- **Limits:** the old per-agent Claude Code plugin is retired. Hindsight calls an LLM on every write, so writes cost more than a line in a file. Its page does not say whether `/compact`, `/clear` or `/branch` change what gets captured, so test it on your repo before you rely on it.

For how Claude Code's own memory files work, see [Claude Code memory](/articles/claude-code-memory/). For the hook side, see [Claude Code hooks and memory](/articles/claude-code-hooks-memory/). For the wider picture, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

## Sources

- Claude Code commands: [code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands) (read 2026-11-15)
- Context window and what survives compaction: [code.claude.com/docs/en/context-window](https://code.claude.com/docs/en/context-window) (read 2026-11-15)
- Hindsight Coding Agents: [hindsight.vectorize.io/sdks/integrations/coding-agents](https://hindsight.vectorize.io/sdks/integrations/coding-agents) (read 2026-11-12)


## FAQ

### What is a Claude Code command?

A Claude Code command is a word you type after a slash inside a session, such as /memory or /compact. Commands switch models, edit memory files, summarize or clear the chat, and run workflows. A command is only recognized at the start of your message.

### Which Claude Code command edits CLAUDE.md?

The /memory command opens your CLAUDE.md files for editing. It also turns auto memory on or off and shows the auto memory entries. /init creates a starter CLAUDE.md for a project.

### Does /compact delete my CLAUDE.md or auto memory?

No. According to the Claude Code context window docs, project-root CLAUDE.md and auto memory are read from disk again after compaction. What is lost is any detail that the chat summary leaves out.

