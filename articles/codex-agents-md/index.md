# Codex AGENTS.md: Locations, Merge Order and 32 KiB Limit

> Where Codex CLI reads AGENTS.md, which file wins when several match, the 32 KiB default cap and how to raise it. From the OpenAI Codex AGENTS.md guide, read 2026-11-24.

Source: https://aiagentmemory.org/articles/codex-agents-md/ · Updated 2026-11-24 · aiagentmemory.org


**Codex reads `AGENTS.md` files before it does any work.** It looks in your Codex home folder, then in every folder from the Git repo root down to where you start it. Files closer to you come later and win. The combined text stops at 32 KiB unless you raise `project_doc_max_bytes`.

[Codex](https://github.com/openai/codex) is OpenAI's coding agent. [AGENTS.md](https://agents.md/) is a plain markdown file with standing instructions: how to run tests, what not to touch, which package manager to use. The facts below come from OpenAI's [AGENTS.md guide for Codex](https://developers.openai.com/codex/guides/agents-md), read on 24 November 2026.

## Where Codex looks for AGENTS.md

Codex builds its instruction chain once per run. In a TUI session that usually means once per launch.

| Scope | Files Codex checks, in order | Folder |
|---|---|---|
| Global | `AGENTS.override.md`, then `AGENTS.md`. Only the first non-empty file counts. | `~/.codex`, or `CODEX_HOME` if set |
| Project | In each folder from the repo root down to the current one: `AGENTS.override.md`, then `AGENTS.md`, then any fallback names | Git root to current folder |

If Codex cannot find a Git root, it checks only the current folder.

## How the files merge

1. Codex reads the global file first.
2. It walks from the project root down to your current folder, taking at most one file per folder.
3. It joins the files with blank lines, root first.
4. Files closer to your folder come later in the text, so their rules win on conflicts.
5. Empty files are skipped. Codex stops adding files once the total reaches the size limit.

## A minimal setup

Put shared rules at the repo root:

```md
# AGENTS.md

## Repository expectations

- Run `npm run lint` before opening a pull request.
- Document public utilities in `docs/` when you change behavior.
```

Put a rule for one service in a subfolder. This `AGENTS.override.md` replaces the `AGENTS.md` in the same folder, so write it in full:

```md
# services/payments/AGENTS.override.md

## Payments service rules

- Use `make test-payments` instead of `npm test`.
- Never rotate API keys without notifying the security channel.
```

Put personal defaults in `~/.codex/AGENTS.md`, for example "Prefer `pnpm` when installing dependencies." Every repo you open inherits them.

## Size limit and fallback names

The default cap is **32 KiB combined** (`project_doc_max_bytes`). Past that, Codex drops the rest. Raise the cap, or split rules into nested folders, if a long file gets cut off.

```toml
# ~/.codex/config.toml
project_doc_fallback_filenames = ["TEAM_GUIDE.md", ".agents.md"]
project_doc_max_bytes = 65536
```

Fallback names are checked after `AGENTS.md` in each folder. A file with any other name is ignored.

## Check what Codex loaded

Ask Codex to list its instructions. It should name the global file first, then the root file, then any subfolder file:

```bash
codex --ask-for-approval never "Summarize the current instructions."
```

To check a subfolder, add `--cd services/payments`. To keep a log of what loaded, run Codex with `-c log_dir=./.codex-log` and read `codex-tui.log` in that folder.

Common causes of "my rules are ignored": an `AGENTS.override.md` higher up is replacing the file you edited, the file is empty, or Codex was started before the edit (restart it; there is no cache to clear).

## Code review rules

For Codex code review on GitHub, add a `## Code Review Rules` section to the `AGENTS.md` closest to the code it covers. Keep repo-wide checks at the root and service checks in nested files. Keep rules short, and leave formatting and lint to CI.

## Where Hindsight fits

`AGENTS.md` holds rules you write by hand. Codex reads the file at the start of every run, so it only knows what someone wrote down. It cannot recall what happened in last week's sessions.

Hindsight adds that recall. Its coding agents package installs into Codex CLI with `npx @vectorize-io/hindsight-coding-agents install codex`. It keeps one memory bank per Git repo, and every agent that works in that repo reads the same bank. The package covers 22 CLI agents, per the [Hindsight coding agents page](https://hindsight.vectorize.io/sdks/integrations/coding-agents).

Limits to know:

- Keep `AGENTS.md` for rules that must always apply. A recalled memory is a hint, not a guaranteed rule.
- Recall adds tokens to each prompt. You set the budget with `max_tokens`.
- Memory can live in Hindsight Cloud (usage-based pricing), on a server you run, or in a local daemon. Self-hosting is free under the MIT license. Source: [github.com/vectorize-io/hindsight](https://github.com/vectorize-io/hindsight).

## Related pages

- [Codex memory](/articles/codex-memory/): the built-in memories layer, and how it differs from AGENTS.md.
- [Codex CLI config reference](/articles/codex-cli-config-reference/): every config key, including the two above.
- [Claude Code vs Codex](/articles/claude-code-vs-codex/): how each tool loads its instruction file.


## FAQ

### Where does Codex look for AGENTS.md?

In two places. First the Codex home folder, ~/.codex (or CODEX_HOME if you set it). Then every folder from your Git repo root down to the folder you start Codex in. In each folder Codex takes one file: AGENTS.override.md if it exists, else AGENTS.md, else a fallback name you set. Source: the OpenAI Codex AGENTS.md guide, read 2026-11-24.

### What is the size limit for Codex AGENTS.md files?

32 KiB combined by default, set by project_doc_max_bytes. Codex stops adding files once the total reaches that size. To raise it, set project_doc_max_bytes in ~/.codex/config.toml, for example to 65536, and restart Codex.

### Which AGENTS.md wins if two files disagree?

The one closer to your working folder. Codex joins the files from the root down, with blank lines between them, so later text overrides earlier text. An AGENTS.override.md in a subfolder replaces the AGENTS.md in that same folder.

