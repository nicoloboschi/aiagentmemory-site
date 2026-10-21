# Claude Code Memory: CLAUDE.md, Auto Memory and Plugins

> How Claude Code memory works: CLAUDE.md locations and load order, @imports, auto memory and how to turn it off, /memory, plus memory MCP servers, claude-mem and Hindsight.

Source: https://aiagentmemory.org/articles/claude-code-memory/ · Updated 2026-10-17 · aiagentmemory.org


**Claude Code memory** is two built-in systems. **CLAUDE.md files** hold instructions you write, and Claude Code loads them at the start of every session. **Auto memory** holds notes Claude writes for itself in `~/.claude/projects/<project>/memory/`. You manage both with `/memory`. For more than notes, add a memory MCP server or a plugin such as claude-mem or Hindsight.

Each Claude Code session starts with an empty context window, so anything Claude should know has to come from a file that loads back in. This page covers how Claude Code's built-in memory works, from Anthropic's [Claude Code memory docs](https://code.claude.com/docs/en/memory), then the add-on options and how they compare. All facts were checked on 12 October 2026. For how other coding agents (Cursor, Copilot) handle memory, see [AI coding agent memory](/articles/ai-coding-agent-memory/), or [GitHub Copilot Memory](/articles/github-copilot-memory/) for Copilot's citation-checked repository facts in detail. The Claude chat app at claude.ai has its own, separate memory feature, covered in [Claude AI long-term memory](/articles/claude-ai-long-term-memory/).

## What is Claude Code memory?

**Claude Code memory is the set of files that Claude Code loads into a new session so Claude does not start from zero: CLAUDE.md instruction files that you write and commit, and auto memory notes that Claude writes from your corrections. Both are plain Markdown. Claude reads them as context, not as rules it is forced to obey.**

The docs compare the two like this:

| | CLAUDE.md files | Auto memory |
|---|---|---|
| Who writes it | You | Claude |
| What it holds | Instructions and rules | Learnings and patterns |
| Scope | Project, user or organization | One per repository, shared across worktrees |
| What loads | Every file in scope, in full | First 200 lines or 25 KB of `MEMORY.md` |
| Shared with the team | Yes, through git (project files) | No, machine-local |

The docs are clear about one limit: CLAUDE.md is "context, not enforced configuration." If an action must never happen, block it with a PreToolUse hook or a `permissions.deny` rule. Don't rely on a line in CLAUDE.md.

## CLAUDE.md files: locations and load order

Claude Code reads CLAUDE.md from four scopes. They load from the broadest to the most specific, so project instructions appear in context after user instructions:

| Scope | Location | Shared with |
|---|---|---|
| Managed policy | macOS `/Library/Application Support/ClaudeCode/CLAUDE.md`, Linux and WSL `/etc/claude-code/CLAUDE.md`, Windows `C:\Program Files\ClaudeCode\CLAUDE.md` | Everyone on the machine; can't be excluded |
| User | `~/.claude/CLAUDE.md` | Only you, all projects |
| Project | `./CLAUDE.md` or `./.claude/CLAUDE.md` | The team, through git |
| Local | `./CLAUDE.local.md` (add it to `.gitignore`) | Only you, this project |

How the files load:

- **Files above the working directory load at launch.** Run Claude in `foo/bar/` and it reads `foo/CLAUDE.md` and `foo/bar/CLAUDE.md`, plus any `CLAUDE.local.md` next to them.
- **Files are joined, not replaced.** Content runs from the filesystem root down to where you launched, so closer instructions come last. In each directory, `CLAUDE.local.md` comes after `CLAUDE.md`.
- **Files in subdirectories load later**, the first time Claude reads, writes or edits a file in that folder.
- **HTML comments are removed.** A block-level `<!-- note -->` costs no tokens, so you can leave notes for people.
- **Project-root CLAUDE.md survives `/compact`.** Claude Code reads it again from disk after compaction.

The docs suggest keeping each file under 200 lines. A file can be up to 4 MiB; anything larger is skipped. Run `/init` to generate a first CLAUDE.md from your codebase, and `/context` to see which files loaded under **Memory files**.

### Rules in .claude/rules/

For larger projects, put one topic per file in `.claude/rules/` (for example `testing.md` or `api-design.md`). A rule with no front matter loads at launch. A rule with a `paths:` list of globs loads only when Claude works on a matching file:

```markdown
---
paths:
  - "src/api/**/*.ts"
---

- All API endpoints must include input validation
```

Personal rules in `~/.claude/rules/` apply to every project on your machine. In big monorepos, the `claudeMdExcludes` setting skips other teams' CLAUDE.md files by path or glob.

### AGENTS.md

Since v2.1.277, Claude Code reads `AGENTS.md` directly when there is no `CLAUDE.md`, `.claude/CLAUDE.md` or `CLAUDE.local.md` in the working directory or above it. If you have both, it reads only CLAUDE.md by default. To read both, set **Project instructions** to `claude-md-and-agents-md` in `/config`, or put `@AGENTS.md` at the top of your CLAUDE.md. Claude Code does not read `AGENTS.override.md`, which is a Codex file.

## Imports with @path

A CLAUDE.md can pull in other files with `@path/to/file`:

```markdown
See @README for the project overview and @package.json for npm commands.

# Additional instructions
- git workflow @docs/git-instructions.md
- @~/.claude/my-project-instructions.md
```

The rules for imports, from the docs:

1. Relative paths resolve from the file that contains the import, not from the working directory.
2. Absolute paths and `~/` paths work too.
3. Imported files can import other files, up to **four hops** deep.
4. For a path with spaces, put a backslash before each space (`@Design\ Docs/api.md`). A quoted path is not imported.
5. Imports inside backticks or code blocks are ignored, so `` `@README` `` stays as text.
6. Imports load at launch, so splitting a long file into imports does not save tokens. Use path-scoped rules for that.
7. The first time a project imports a file outside the working directory, Claude Code asks you to approve it. This protects you from files someone else committed to a shared repo.

Importing from your home directory is also the way to share private notes across git worktrees, because a gitignored `CLAUDE.local.md` exists only in the worktree where you created it.

## Claude Code auto memory

**Auto memory** lets Claude save notes for itself as it works, without you writing anything. Each note is a Markdown file with a `type` field in its front matter, one of four types:

- `user`: your role, skills and working preferences
- `feedback`: corrections you gave and approaches you confirmed
- `project`: ongoing work, deadlines and decisions Claude can't learn from the code or git history
- `reference`: where to find outside information, such as an issue tracker or dashboard

Claude skips anything it can work out from the code, such as architecture or file paths, and anything your CLAUDE.md already says. It doesn't save something every session.

### Where auto memory is stored

Each project gets `~/.claude/projects/<project>/memory/`. The `<project>` name comes from the git repository, so all worktrees and subdirectories of one repo share one memory folder. Inside are a `MEMORY.md` index (one line per memory) and one topic file per memory, such as `feedback_testing.md`. Write times go into a `modified` front matter field.

At session start Claude Code loads the first **200 lines or 25 KB** of `MEMORY.md`, whichever comes first. Topic files are not loaded at start; Claude opens them when it needs them. If `MEMORY.md` grows past the limit, Claude Code tells Claude to shorten it, because anything past the limit is dropped on the next load.

Auto memory is **machine-local**. It is not shared across machines or cloud sessions. Memory files are kept when Claude Code cleans up old session transcripts. To move the folder, set `autoMemoryDirectory` in `settings.json` (an absolute path or one starting with `~/`).

### How to turn auto memory on or off

Auto memory is on by default in local sessions. To change it:

- **Everywhere:** open `/memory` and use the auto memory toggle. It saves `autoMemoryEnabled` to `~/.claude/settings.json`.
- **One project:** put `{"autoMemoryEnabled": false}` in that project's settings.
- **By environment:** set `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`.

The toggle can turn memory off, but not back on, in background sessions or in a session that another Claude Code session started. Run `claude` directly in a terminal to turn it back on.

## The /memory command

`/memory` is the control panel for both systems. It:

- lists your CLAUDE.md, CLAUDE.local.md and other memory files across user and project scopes, including ones that don't exist yet
- opens any file in your editor, creating it first if needed
- toggles auto memory on or off
- opens the auto memory folder so you can read, edit or delete what Claude saved

If you say "remember that the API tests need a local Redis," Claude saves it to auto memory. To put it in CLAUDE.md instead, say "add this to CLAUDE.md," or edit the file through `/memory`. To check what loaded at launch, use `/context`.

## Add-on memory for Claude Code

Built-in memory holds short notes on one machine. It doesn't keep a searchable history of past sessions, and it isn't shared with teammates. Three kinds of add-on fill those gaps.

### Memory MCP servers

Any MCP server can give Claude memory tools. The reference one is the **Knowledge Graph Memory Server** (`@modelcontextprotocol/server-memory`) from the [official MCP servers repo](https://github.com/modelcontextprotocol/servers/tree/main/src/memory). It stores entities, relations and observations in a local JSONL file and gives Claude tools such as `create_entities`, `add_observations` and `search_nodes`. Add it for all your projects:

```bash
claude mcp add --scope user --env MEMORY_FILE_PATH=$HOME/.claude/memory.jsonl \
  memory -- npx -y @modelcontextprotocol/server-memory
```

Set `MEMORY_FILE_PATH`, because the default file sits in the server's own install folder. Claude only uses the graph when it decides to call the tools, so tell it to in CLAUDE.md. Other memory servers (Mem0, Basic Memory, Hindsight and more) are compared in [AI memory MCP servers](/articles/ai-memory-mcp-server/).

### claude-mem

**[claude-mem](https://github.com/thedotmack/claude-mem)** is an Apache 2.0 plugin by Alex Newman, with 101,326 GitHub stars on 17 October 2026. Its README says it "captures everything your agent does during sessions, compresses it with AI, and injects relevant context back into future sessions." It works through Claude Code lifecycle hooks (SessionStart, UserPromptSubmit, PostToolUse, Stop, SessionEnd), so capture is automatic.

How it works, per the README:

- A local worker service (run with Bun) stores sessions, observations and summaries in **SQLite**, with **Chroma** for hybrid semantic and keyword search.
- Claude searches the history through MCP tools in three steps: `search` returns a compact index (about 50-100 tokens per result), `timeline` shows what happened around a result, and `get_observations` fetches full details only for chosen IDs.
- A web viewer shows the memory stream, and `<private>` tags keep content out of storage.
- It also installs for Codex, OpenCode, Gemini, Copilot and others.

Install it from inside Claude Code with `/plugin marketplace add thedotmack/claude-mem` and `/plugin install claude-mem`, or run `npx claude-mem install`. The `npx` installer asks you to sign in and offers a hosted "claude-mem observer" with a 30-day free trial. You can instead pick your own OpenRouter or Gemini key or your Anthropic plan, or pass `--provider` to skip sign-in. Summaries cost model calls, so check which provider you picked. Full detail, including the other agents it installs for and what the hosted observer costs, is on [claude-mem](/articles/claude-mem/).

### Hindsight plugin for Claude Code

**[Hindsight](https://github.com/vectorize-io/hindsight)** is an MIT-licensed memory server from Vectorize with a native Claude Code plugin (it has native plugins for Codex, Cursor and Copilot too). Instead of keeping a log of sessions, it extracts facts, times, entities and relations from what it retains, then merges related facts into **observations**: beliefs with supporting quotes that get refined as new sessions add evidence. Recall runs semantic, keyword, graph and time-range search in parallel and reranks the results. `reflect` reasons over many memories to answer questions like "why did we drop the Redis cache?"

Where it leads, by the numbers:

- **Long histories.** On the BEAM benchmark it scores 73.9 at 1M tokens and 64.1 at 10M, against 64.1 and 48.6 for Mem0's Platform. Those figures come from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight), and all are vendor-run. Months of coding sessions are this kind of history.
- **Everything open source.** Graph and temporal recall, observations, mental models and `reflect` are all in the MIT server, with no paid tier needed.
- **No extra API key.** It works with 25+ LLM providers, local Ollama included, or your existing Claude subscription.
- **Shared memory.** It runs as a server (Docker, pip or Hindsight Cloud), so several machines and agents can use the same memory bank, which built-in auto memory can't do.

The limits: it returns more context per retrieval than Mem0 by default (about 4x, by Mem0's measurement), so set a lower `max_tokens` to keep each prompt small. Each write calls an LLM, which costs more than a plain log. More on how it works in [Hindsight AI memory](/articles/ai-memory-hindsight/).

## Built-in memory vs claude-mem vs Hindsight

| | CLAUDE.md + auto memory | claude-mem | Hindsight plugin |
|---|---|---|---|
| Ships with Claude Code | Yes | No, plugin | No, plugin + server |
| License | Part of Claude Code | Apache 2.0 | MIT, full server included |
| Who writes memory | You (CLAUDE.md), Claude (auto memory) | Automatic, from hooks on every session | Automatic: extracted facts, consolidated into observations |
| What is stored | Instructions; short typed notes | Tool-use observations and AI-compressed session summaries | Facts, experiences, observations, mental models |
| Storage | Markdown in the repo and `~/.claude/projects/` | Local SQLite + Chroma; optional cloud sync | PostgreSQL (embedded option), local or Hindsight Cloud |
| How it loads | Files at session start; topic notes on demand | Context injected at start; MCP search on demand | Recall by semantic, keyword, graph and time search |
| Reasoning over memory | No | No | Yes (`reflect`) |
| Shared across machines | Only what you commit to git | Not by default; optional cloud backup to cmem.ai | Yes, through one server or Cloud |
| Extra cost | None | Model calls for summaries | LLM calls per write; more tokens per recall |
| Best for | Rules and conventions every session needs | Searching what Claude did in past sessions | Long project history, team memory, agents that learn |

Sources: [Claude Code memory docs](https://code.claude.com/docs/en/memory), the [claude-mem README](https://github.com/thedotmack/claude-mem), the [Hindsight README](https://github.com/vectorize-io/hindsight).

These layers stack. Keep build commands and hard conventions in CLAUDE.md in every case. Add claude-mem if you mostly want to look back at what happened in past sessions on one machine. Add Hindsight if the history runs to months, if several people or agents should share what was learned, or if you want the agent to answer "why" questions over that history.

## Claude Code memory vs Codex memory

OpenAI's Codex uses the same two-layer idea with different files:

| | Claude Code | Codex |
|---|---|---|
| Instruction file | `CLAUDE.md` (reads `AGENTS.md` if no CLAUDE.md) | `AGENTS.md`, `AGENTS.override.md`, fallback names |
| Global file | `~/.claude/CLAUDE.md` | `~/.codex/AGENTS.md` (or `AGENTS.override.md`) |
| Size limit | Under 200 lines advised; files over 4 MiB skipped | Stops at 32 KiB combined (`project_doc_max_bytes`) |
| Automatic memory | Auto memory, **on** by default | Memories, **off** by default |
| Stored in | `~/.claude/projects/<project>/memory/` | `~/.codex/memories/` |
| When it writes | During the session | In the background, after a chat has been idle |
| Per-chat control | `/memory` toggle | `/memories` |

Per OpenAI's [Codex memories docs](https://learn.chatgpt.com/docs/customization/memories), you turn memories on with `memories = true` under `[features]` in `config.toml`, or in **Settings > Personalization**. Codex skips short sessions, redacts secrets, and calls its memory files "generated state" that you shouldn't edit by hand. Claude Code's auto memory files, by contrast, are plain Markdown you are invited to edit. Both vendors give the same advice: put rules that must always apply in the instruction file, not in automatic memory. Codex's instruction chain is described in its [AGENTS.md guide](https://learn.chatgpt.com/docs/agent-configuration/agents-md). The full breakdown is on [Codex memory](/articles/codex-memory/).

If your team uses both tools, keep one `AGENTS.md` and let Claude Code read it, or import it with `@AGENTS.md`. Memory files are also an attack surface: a bad instruction committed to a shared CLAUDE.md loads into every session. See [AI memory injection](/articles/ai-memory-injection/).


## FAQ

### Where does Claude Code store its memory?

Instructions you write live in CLAUDE.md files: ~/.claude/CLAUDE.md for you, ./CLAUDE.md or ./.claude/CLAUDE.md for the project, ./CLAUDE.local.md for private project notes, plus an optional managed policy file. Auto memory, the notes Claude writes itself, lives in ~/.claude/projects/<project>/memory/ with a MEMORY.md index.

### How do I turn off Claude Code auto memory?

Open /memory in a session and use the auto memory toggle, which saves autoMemoryEnabled to ~/.claude/settings.json. For one project, set "autoMemoryEnabled": false in that project's settings. You can also set the environment variable CLAUDE_CODE_DISABLE_AUTO_MEMORY=1.

### Is claude-mem better than Claude Code's built-in memory?

They do different jobs. Built-in auto memory saves a few short notes about your preferences and project. claude-mem records what Claude did in every session, compresses it with an LLM and lets Claude search that history. Many people run both.

