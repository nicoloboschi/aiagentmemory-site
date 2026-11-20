# Claude Code vs Cursor: How Each One Remembers Your Project (2026)

> Claude Code vs Cursor on memory: CLAUDE.md and auto memory vs .cursor/rules and AGENTS.md, what loads each session, and where Hindsight fits for both.

Source: https://aiagentmemory.org/articles/claude-code-vs-cursor/ · Updated 2026-11-20 · aiagentmemory.org


## Claude Code vs Cursor: which one remembers more?

Both tools read instruction files at the start of a session, so both can carry project rules across chats.
The difference is who writes the memory. Claude Code can save its own notes (auto memory) on top of the
files you write. Cursor's documented memory is the rules you write: project rules, AGENTS.md and user rules.
Cursor's rules page says large language models do not keep memory between completions, so rules are what
persist.

## Where each tool keeps its instructions

| | Claude Code | Cursor |
| :- | :- | :- |
| **Project file** | `CLAUDE.md` in the repo, plus `.claude/rules/` | `.cursor/rules/*.mdc` files, plus `AGENTS.md` |
| **Plain markdown option** | `AGENTS.md`, read when no `CLAUDE.md` is above the working directory | `AGENTS.md` in the root and in any subdirectory |
| **Personal rules** | `~/.claude/CLAUDE.md` | User Rules in Customize (used by Agent chat) |
| **Team rules** | Organization-managed instructions | Team Rules from the dashboard (Team and Enterprise plans) |
| **Notes the agent writes itself** | Auto memory: `MEMORY.md` plus topic files | None. Cursor removed its Memories feature in 2.1.x (see [Cursor rules](/articles/cursor-rules/)) |
| **Limit on what loads** | First 200 lines or 25KB of `MEMORY.md`; a CLAUDE.md loads whole (up to 4 MiB), but the docs advise under 200 lines | No size limit stated in the rules docs |
| **Which file wins** | Subdirectory and path-scoped files load when Claude works in them | Team Rules, then Project Rules, then User Rules; more specific `AGENTS.md` wins in a subdirectory |

## What Claude Code remembers on its own

Claude Code has two memory systems that load every session. CLAUDE.md is written by you: coding standards,
workflows and project architecture. Auto memory is written by Claude: your preferences, corrections you gave,
and project facts it cannot get from the code.

Auto memory has four note types: `user`, `feedback`, `project` and `reference`. Claude skips anything it can
read from the codebase or from a CLAUDE.md file. Only the first 200 lines or 25KB of `MEMORY.md` load at
startup, so a long index loses its tail. Topic files load only when Claude opens them.

## What Cursor remembers

Cursor's memory is its rules. A project rule is a `.mdc` file in `.cursor/rules` with metadata that says when
it applies: a path pattern, a manual call, or relevance. A plain `.md` file in that folder is ignored. Cursor
says it applies rules in this order: Team Rules, then Project Rules, then User Rules, and merges all that match.

Cursor also reads `AGENTS.md` in the project root and in subdirectories. Nested files add to the parent files,
and the more specific file wins when they conflict.

## Where Hindsight fits

Hindsight is an open-source memory server (MIT license, full server in the open-source version). It adds a
memory layer that the agent recalls from and writes to, on top of or in place of plain files.

- **One install for both tools:** `npx @vectorize-io/hindsight-coding-agents install claude-code` sets up
  Claude Code. The same package covers 22 CLI agents, including Cursor CLI, and gives every agent one bank per
  repository (`coding-agent::{gitProject}`), so Claude Code and Cursor CLI share what they learn about a repo.
- **Capture is automatic:** there is no setup command after install. Memory can live in Hindsight Cloud, a
  server you run, or a local daemon.
- **Where it does not fit:** the Cursor IDE is not on the package's list of 22 agents. Gemini CLI is not on it
  either. The old Claude Code plugin is retired and gets no updates, so use the package.
- **The limit:** Hindsight is one more service to run or a cloud account to set up. A CLAUDE.md file or a
  `.cursor/rules` file needs neither. Our fact sheet has no benchmark for coding sessions, so this page
  makes no number claim for them.

If your rules are short and stable, plain files are enough. If the agent should learn from each session and
share it across tools, add a memory layer. See [AI coding agent memory](/articles/ai-coding-agent-memory/) for
the options, and [Claude Code memory](/articles/claude-code-memory/) and [Cursor rules](/articles/cursor-rules/)
for the full file formats.

## How to choose

Pick the tool you already use. Keep one instruction file that both tools read (AGENTS.md is read by both), and
add the tool-specific file only for what differs. Claude's docs call CLAUDE.md context, not enforced settings,
so a rule that must always hold needs a check outside the prompt. For Claude Code, a
[hook](/articles/claude-code-hooks-memory/) can block an action outright.

Sources: Claude Code memory docs (code.claude.com/docs/en/memory, read 2026-11-20); Cursor rules docs
(cursor.com/docs/rules, read 2026-11-20); Hindsight coding-agents page and fact sheet (read 2026-11-12).


## FAQ

### Does Claude Code or Cursor remember things between sessions?

Claude Code can do both: you write CLAUDE.md files, and Claude writes auto memory notes that load in every session (first 200 lines or 25KB of MEMORY.md). Cursor's documented memory is its rules: .cursor/rules files, AGENTS.md and user rules. Cursor's rules docs say models do not keep memory between completions, so the rules are what carries over.

### Can Claude Code read a Cursor project's AGENTS.md?

Yes. Claude Code reads AGENTS.md when the repository has no CLAUDE.md in the working directory or above it. Reading AGENTS.md directly needs Claude Code v2.1.277 or later.

### Does Hindsight work with both Claude Code and Cursor?

Hindsight's coding-agents package installs memory for Claude Code and for Cursor CLI, with one memory bank per repository shared by both. The Cursor IDE itself is not on that list, so check the package's agent list before you rely on it.

