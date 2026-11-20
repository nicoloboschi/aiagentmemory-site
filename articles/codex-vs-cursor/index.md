# Codex vs Cursor: How Each One Remembers Your Project (2026)

> Codex vs Cursor on memory: AGENTS.md and Codex memories vs .cursor/rules and AGENTS.md, the size limits, and where Hindsight fits for both.

Source: https://aiagentmemory.org/articles/codex-vs-cursor/ · Updated 2026-11-20 · aiagentmemory.org


## Codex vs Cursor: which one remembers more?

Both tools read instruction files at the start of work, so both can carry project rules from one chat to the next.
The difference is who writes the memory. Codex can save notes from past chats in local memory files, if you turn
that on. Cursor's documented memory is the rules you write: project rules, AGENTS.md and user rules. Cursor's
rules page says large language models do not keep memory between completions, so rules are what persist.

Codex's own memory docs give the same advice: keep required team guidance in AGENTS.md, and treat memories as a
recall layer, not the only source for rules that must always apply.

## Where each tool keeps its instructions

| | Codex | Cursor |
| :- | :- | :- |
| **Project file** | `AGENTS.override.md` or `AGENTS.md` in each folder from the project root down to where you work | `.cursor/rules/*.mdc` files, plus `AGENTS.md` |
| **Plain markdown option** | `AGENTS.md` is the file; add other names with `project_doc_fallback_filenames` | `AGENTS.md` in the root and in any subdirectory |
| **Personal rules** | `~/.codex/AGENTS.override.md` or `~/.codex/AGENTS.md` (first non-empty file wins) | User Rules in Customize (used by Agent chat) |
| **Team rules** | Checked-in `AGENTS.md` files | Team Rules from the dashboard (Team and Enterprise plans) |
| **Notes written from past chats** | Local memories, off by default; written in the background after a chat has been idle | None in the rules docs |
| **Size limit** | 32 KiB combined by default (`project_doc_max_bytes`); Codex stops adding files once the limit is reached | No size limit stated in the rules docs |
| **Which file wins** | Files closer to your folder come later and override earlier guidance | Team Rules, then Project Rules, then User Rules; all matching rules are merged |

## What Codex remembers on its own

Codex has two parts. The first is instruction files you write. Codex reads them before it does any work, and
it skips empty files. The second is local memories: after you turn them on, Codex turns useful context from
earlier chats into memory files under your Codex home folder (`~/.codex` by default). Codex skips active or
short-lived chats, removes secrets from the generated notes, and does not update right away when a chat ends.

You can control memory per chat with `/memories` in the Codex CLI or the desktop app. Two settings matter for
this comparison: `memories.use_memories` (inject old memories into new sessions) and
`memories.disable_on_external_context`, which keeps chats that used MCP tools or web search out of memory
generation. The docs say to review memory files before you share your Codex home folder.

## What Cursor remembers

Cursor's memory is its rules. A project rule is a `.mdc` file in `.cursor/rules` with metadata that says when it
applies: always, by file path, by a manual call, or by relevance. A plain `.md` file in that folder is ignored.
Cursor applies rules in this order: Team Rules, then Project Rules, then User Rules, and merges every rule that
matches. Cursor reads `AGENTS.md` too, in the root and in subfolders.

## Where Hindsight fits

Hindsight is an open-source memory server (MIT license, full server in the open-source version). It adds a
memory layer that the agent recalls from and writes to, on top of or in place of plain files.

- **One install for the coding agents:** `npx @vectorize-io/hindsight-coding-agents install all` sets up every
  agent on the package's list of 22 CLI agents. That list includes Codex CLI and Cursor CLI. They share one memory
  bank per repository (`coding-agent::{gitProject}`), so what Codex learns about a repo is there for Cursor CLI too.
- **Capture is automatic:** there is no setup command after install. Memory can live in Hindsight Cloud, a server
  you run, or a local daemon.
- **Where it does not fit:** the Cursor IDE is not on the package's list. Gemini CLI is not on it either.
- **The limit:** Hindsight is one more service to run, or a cloud account to set up. An AGENTS.md file or a
  `.cursor/rules` file needs neither. Our fact sheet has no benchmark for coding sessions, so this page makes no
  number claim for them.

If your rules are short and stable, plain files are enough. If the agent should learn from each session and
share what it learns across tools, add a memory layer. See [AI coding agent memory](/articles/ai-coding-agent-memory/)
for the options, [Codex memory](/articles/codex-memory/) and [Cursor rules](/articles/cursor-rules/) for the full
file formats, and [Claude Code vs Cursor](/articles/claude-code-vs-cursor/) for the Claude side of the same question.

## How to choose

Pick the tool you already use. Keep one AGENTS.md that both tools read, and add the tool-specific file only for
what differs. Codex's docs say its memories are a recall layer, so a rule that must always hold belongs in
AGENTS.md.
Sources: Codex AGENTS.md guide (developers.openai.com/codex/guides/agents-md, read 2026-11-20); Codex memories
docs (developers.openai.com/codex/memories, read 2026-11-20); Cursor rules docs (cursor.com/docs/rules, read
2026-11-20); Hindsight coding-agents page and fact sheet (read 2026-11-12).


## FAQ

### Does Codex or Cursor remember things between sessions?

Codex can do both: you keep AGENTS.md files, and Codex can write local memories from past chats. Local memories are off by default. Cursor's documented memory is its rules: .cursor/rules files, AGENTS.md and user rules. Cursor's rules docs say large language models do not keep memory between completions, so the rules carry over.

### Is AGENTS.md read by both Codex and Cursor?

Yes. Codex reads AGENTS.md from your home folder and from each folder between the project root and where you work. Cursor reads AGENTS.md in the project root and in subdirectories. Both tools use the same file name.

### Does Hindsight work with Codex and Cursor?

Hindsight's coding-agents package covers Codex CLI and Cursor CLI, with one memory bank per repository shared by every agent on the list. The Cursor IDE is not on that list, so check the package's agent list before you rely on it.

