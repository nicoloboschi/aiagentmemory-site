# Claude Code vs Codex: Memory, Files and Config (2026)

> Claude Code vs Codex side by side: instruction files, memory defaults, skill folders, MCP config paths, and one memory bank both tools can share. Sourced.

Source: https://aiagentmemory.org/articles/claude-code-vs-codex/ · Updated 2026-11-16 · aiagentmemory.org


Claude Code vs Codex comes down to files and defaults more than features. Both are terminal coding agents that read a markdown instruction file when a session starts. Claude Code's auto memory is on by default in local sessions. Codex memories are off until you turn them on.

## Claude Code vs Codex at a glance

| | Claude Code | Codex |
| :- | :- | :- |
| Instruction file | `CLAUDE.md`, and `AGENTS.md` in current versions | `AGENTS.md` (and `AGENTS.override.md`) |
| Size limit | Loads files up to 4 MiB; docs advise under 200 lines per CLAUDE.md | 32 KiB by default (`project_doc_max_bytes`) |
| Built-in memory | Auto memory, on by default in local sessions | Codex memories, off by default |
| Where memory is stored | `~/.claude/projects/<project>/memory/` | Under `~/.codex` (`memories_v2/` in a fresh home) |
| Skill folders | `.claude/skills/` and `~/.claude/skills/` | `.agents/skills/` and `~/.agents/skills/` |
| Main config file | `~/.claude/settings.json` | `~/.codex/config.toml` |
| MCP servers | `.mcp.json` in the project, or `claude mcp add` | `[mcp_servers.<name>]` in `config.toml`, or `codex mcp add` |

Sources for each row are in the sections below.

## Instruction files

**Claude Code** loads `CLAUDE.md` and `CLAUDE.local.md` from the working directory and every directory above it. Files in subdirectories load when Claude works there. An `@path` line imports another file, up to four hops deep. The docs say to keep each CLAUDE.md under 200 lines, because longer files cost context and are followed less closely. Claude Code can also read a repository's `AGENTS.md` in place of CLAUDE.md. Some older sessions read CLAUDE.md only, so the docs suggest an `@AGENTS.md` import inside CLAUDE.md for them ([Claude Code memory docs](https://code.claude.com/docs/en/memory)).

**Codex** reads one file per directory: `AGENTS.override.md` first, then `AGENTS.md`, then any fallback names you set in `project_doc_fallback_filenames`. It starts at the Git root and goes down to the current directory, and files closer to you come later and win. Codex stops adding files once the total reaches 32 KiB. Its global file lives in `~/.codex`, or in `CODEX_HOME` if you set it ([AGENTS.md guide](https://learn.chatgpt.com/docs/agent-configuration/agents-md)).

What this means in practice: a repo with only an `AGENTS.md` works with both tools. A repo with only a `CLAUDE.md` is read by Claude Code alone.

## Memory

**Claude Code auto memory** is a set of notes Claude writes itself from your corrections. It is on by default in local sessions. Each project gets a folder under `~/.claude/projects/<project>/memory/`, and git worktrees of one repo share it. The first 200 lines or 25 KB of `MEMORY.md` load at every session start. You can turn it off with `/memory`, with `autoMemoryEnabled: false` in a project's settings, or with the environment variable `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` ([Claude Code memory docs](https://code.claude.com/docs/en/memory)).

**Codex memories** turn useful context from past chats into local files. They are off by default. You can turn them on in the desktop app under Settings > Personalization, or with `memories = true` under `[features]` in `config.toml`. Updates run in the background and can lag, because Codex waits for a chat to go idle. `/memories` controls, per chat, whether that chat uses or feeds memory ([Codex memories docs](https://learn.chatgpt.com/docs/customization/memories)).

Both tools' built-in memory is a recall aid. Neither docs page treats it as enforced rules. For a rule that must always apply, Claude Code's docs point to a PreToolUse hook. The Codex docs say to keep required team guidance in `AGENTS.md`.

## Skills

**Claude Code** reads a skill from a folder with a `SKILL.md` file: `~/.claude/skills/<name>/` for all your projects, or `.claude/skills/<name>/` for one repo. Only `description` is recommended in the front matter, because Claude uses it to decide when to load the skill. Skills follow the open [Agent Skills](https://agentskills.io) standard, with extra Claude Code fields ([skills docs](https://code.claude.com/docs/en/skills)).

**Codex** reads skills from `.agents/skills/` in the current folder and each parent up to the repo root, from `~/.agents/skills`, and from `/etc/codex/skills` for admins. Its `SKILL.md` must include both `name` and `description`. You call a skill with `$` or `/skills` in the CLI, or Codex picks it when the task matches the description ([skills guide](https://learn.chatgpt.com/docs/build-skills)).

The folders differ. A skill kept only in `.claude/skills/` is not loaded by Codex, and the reverse is also true. Copy it or link it into both places.

## Config and MCP servers

**Claude Code** keeps user settings in `~/.claude/settings.json`. An MCP server is stored in one of three scopes: `local` (default, in `~/.claude.json`), `project` (in `.mcp.json` at the project root, shared through Git, and Claude asks you to approve it), or `user` (all your projects). You add one with `claude mcp add`. The docs list `http`, `sse` (deprecated) and `stdio` transports ([MCP docs](https://code.claude.com/docs/en/mcp)).

**Codex** keeps everything in `~/.codex/config.toml`, shared by the desktop app, the CLI and the IDE extension. A project can add `.codex/config.toml`, but only for trusted projects. Each MCP server gets a table, `[mcp_servers.<name>]`, with `command` for local servers or `url` for remote ones. You add one with `codex mcp add`. Codex also sets approval (`approval_policy`, such as `on-request` or `never`) and sandbox (`sandbox_mode`, such as `workspace-write`) in the same file ([config docs](https://learn.chatgpt.com/docs/config-file/config-basic), [MCP docs](https://learn.chatgpt.com/docs/extend/mcp)). This page does not compare Claude Code's permission settings.

For the full key list, see [Codex CLI config reference](/articles/codex-cli-config-reference/) and [Claude Code settings](/articles/claude-code-settings/).

## Memory that both tools can share

Both tools can write to the same external memory. [Hindsight](https://github.com/vectorize-io/hindsight), an open-source agent memory server (MIT license, full server included), has one npm package for 22 CLI agents. Claude Code and Codex CLI are both on the list. Install with `npx @vectorize-io/hindsight-coding-agents install claude-code`, or `install all` for every agent on the list. Capture is automatic after install, and every agent writes to one bank per repository, named `coding-agent::<git project>` ([coding agents page](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12).

**Where Hindsight fits:**
- **One bank for both tools.** A fact you save while using Claude Code is there for Codex in the same repo.
- **Self-hosting is free.** The server is MIT licensed, so you run it on your own machine with no usage fee.
- **Secret scanning on writes is opt-in.** Hindsight's Memory Defense checks each write against 45 secret and PII patterns, then redacts or blocks the match.

**Limits:**
- **Hosted costs tokens.** Hindsight Cloud charges $10.00 per 1M tokens retained and $0.75 per 1M tokens recalled. The price page was checked on 2026-11-16 ([vectorize.io/pricing](https://vectorize.io/pricing)).
- **Old Claude Code plugin is retired.** Switching to the package needs `--import-conversations`, and the old recall and retain settings do not carry over.
- **No coding-agent benchmark.** The BEAM 10M figures (64.1 for Hindsight, 48.6 for Mem0) both come from [Mem0's comparison page](https://mem0.ai/compare/mem0-vs-hindsight), not an independent test. BEAM is a long-memory test, not a coding-session test.

For how memory works across other coding agents, see [AI coding agent memory](/articles/ai-coding-agent-memory/). For Hindsight in general, see [Hindsight and AI memory](/articles/ai-memory-hindsight/).

## Which should you choose?

Neither vendor's docs compare the two tools on quality, so this page does not pick one. Choose by the setup you already have:

- **If you keep a CLAUDE.md:** add an `AGENTS.md` for Codex, or an `@AGENTS.md` import in CLAUDE.md, and keep one set of rules.
- **If you want memory without setup:** Claude Code's auto memory is on by default. Codex needs a settings change first.
- **If you use both tools in one repo:** one memory bank through Hindsight avoids keeping two copies of the same facts.

## Sources

- [Claude Code: how Claude remembers your project](https://code.claude.com/docs/en/memory), checked 2026-11-16
- [Claude Code: skills](https://code.claude.com/docs/en/skills), checked 2026-11-16
- [Claude Code: MCP](https://code.claude.com/docs/en/mcp), checked 2026-11-16
- [Codex: AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md), checked 2026-11-16
- [Codex: memories](https://learn.chatgpt.com/docs/customization/memories), checked 2026-11-16
- [Codex: skills](https://learn.chatgpt.com/docs/build-skills), checked 2026-11-16
- [Codex: config basics](https://learn.chatgpt.com/docs/config-file/config-basic), checked 2026-11-16
- [Codex: MCP](https://learn.chatgpt.com/docs/extend/mcp), checked 2026-11-16
- [Hindsight coding agents](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12


## FAQ

### Is Claude Code or Codex better for memory?

Neither vendor's docs compare them, so no winner is sourced. Claude Code's auto memory is on by default in local sessions. Codex memories are off by default and you turn them on in settings or config.toml.

### Does Codex read CLAUDE.md?

No. The Codex AGENTS.md guide names AGENTS.override.md and AGENTS.md, plus any fallback names you set. Current Claude Code versions read AGENTS.md too, so one AGENTS.md file can serve both tools.

### How do I share one memory between Claude Code and Codex?

Hindsight's coding agents package installs with one npm command for 22 CLI agents, Claude Code and Codex CLI included. Both tools write to one memory bank per git repository.

