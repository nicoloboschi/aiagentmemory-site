# Claude Code Release Notes: Memory and Context Changes (2026)

> Claude Code releases 2.1.311 to 2.1.323 (Oct 26 to Nov 12, 2026) and the changes to CLAUDE.md, MEMORY.md, compaction, transcripts and /context. Updated with each release.

Source: https://aiagentmemory.org/articles/claude-code-release-notes-memory/ · Updated 2026-11-13 · aiagentmemory.org


## What are the Claude Code release notes for memory?

Claude Code's memory is made of files (CLAUDE.md, MEMORY.md, rules) plus compaction, which shortens a long
conversation when it nears the context limit. This page tracks the release notes that change how those parts
behave. It covers the 11 newest GitHub releases, read on 2026-11-13, and skips releases that touch no memory
or context behavior.

## Memory-related changes by version

| Version | Date | Change that matters for memory |
|---|---|---|
| 2.1.323 | 2026-11-12 | Fixed running `/compact` again, or after `/context`, summarizing away a message Claude had not answered yet. Fixed subagent transcripts and large tool results being saved apart from the session's transcript after entering a worktree. |
| 2.1.322 | 2026-11-10 | Fast mode on Vertex. No memory change. |
| 2.1.321 | 2026-11-09 | Fixed new messages not saved, and `claude -p` and SDK sessions hanging, when memory ran out while the transcript was read. `claude -p` on Linux now exits 1 (not 0) when an NFS export loses transcript lines. Fixed auto mode losing a background subagent's report when the conversation was compacted. |
| 2.1.319 | 2026-11-06 | `/context` no longer sends its full visual grid to Claude on every later turn. Its table keeps the 10 largest rows per list. `/rename` now warns when the name cannot be saved to the transcript. |
| 2.1.318 | 2026-11-04 | Fixed idle compaction replacing a conversation with a reply that was not a summary. Removed the fix that kept earlier thinking when resuming old conversations (that thinking can now be dropped). |
| 2.1.317 | 2026-11-03 | Fixed "Prompt is too long" after automatic compaction when a very large CLAUDE.md had changed since it was last sent. Fixed very long conversations losing their earlier summary when compaction had to shorten its own request. |
| 2.1.316 | 2026-10-31 | Fixed `/model <id>` not warning when the model is unknown. Auto-compact then assumes a 200K context window. Fixed the SDK not loading the CLAUDE.md of a directory outside the working folder. |
| 2.1.314 | 2026-10-30 | Added idle compaction to VS Code and the Claude Desktop Code tab: a conversation over 200K tokens that sits idle is summarized, and the summary is used on your next message if the prompt cache has expired. Each PreCompact hook's output is now cut at 10,000 characters. |
| 2.1.312 | 2026-10-27 | Fixed Claude overwriting a very large CLAUDE.md, rules file or @-included file after reading only its start. Write now asks Claude to read the whole file or edit it in place. |
| 2.1.311 | 2026-10-26 | Fixed Claude seeing stale MEMORY.md and CLAUDE.md content right after automatic compaction, when those files had been edited earlier in the session. Fixed freezes caused by a very long list item in CLAUDE.md. |

Version 2.1.322 has no memory change. The GitHub list has no release numbered 2.1.313 or 2.1.315.

## What the changes mean in practice

**CLAUDE.md and MEMORY.md after compaction.** Three releases fixed the same problem from different sides:
Claude could see old file content after compaction (2.1.311), a very large CLAUDE.md could cause a
"Prompt is too long" error after compaction (2.1.317), and a large file could be overwritten after only
partly being read (2.1.312). If you keep a long CLAUDE.md, update to 2.1.317 or later.

**Compaction.** Idle compaction is new in 2.1.314. It runs on a conversation left idle past 200K tokens, so
the next message starts from a summary. Two fixes followed in 2.1.317 and 2.1.318 for summaries that were lost
or replaced with a non-summary reply. Compaction is still lossy: a summary keeps what the model judges
important, so facts you need later belong in CLAUDE.md or an outside memory store.

**Transcripts and resume.** Several fixes (2.1.317, 2.1.321) stop sessions from losing transcript lines
when a disk, NFS export or quota runs out. Before 2.1.317 some of these runs exited with code 0, so a script
could not tell the write had failed. Now they print an error and exit 1.

**/context.** The 2.1.319 change keeps `/context` short. It matters if you check how much of the window
your files take up.

## Where Hindsight fits

Claude Code's built-in memory is the files and compaction above. Hindsight is a separate memory server.
Its Claude Code setup is the `@vectorize-io/hindsight-coding-agents` npm package, so Claude Code can save and recall facts outside the context window.
Setup and the plugin's behavior are in [Claude Code memory](/articles/claude-code-memory/).

The release notes do not mention Hindsight. The claims below come from Hindsight's own material, not from
Claude Code's changelog:

- Hindsight scored 94.6 on LongMemEval S and 73.9 on BEAM 1M, both on Vectorize's own site (vendor-run,
  not independent). Mem0's comparison page reports Hindsight at 64.1 on BEAM 10M against Mem0's 48.6, with
  both numbers from Mem0's own page.
- Hindsight's server is MIT-licensed and includes the full feature set. Self-hosting is free.
- The limits matter. Hindsight returns more context per recall than Mem0 by default: about 27K tokens for
  BEAM 10M, against under 7K for Mem0 on the same test (Mem0's measurement). Writes also cost more, because
  retain and consolidation call an LLM. Set `max_tokens` lower if the context is too large for your task.

For a project with a short CLAUDE.md, the built-in files are enough. Hindsight helps more when a project runs
for months and its facts no longer fit in one file. Compare
[Claude Code memory](/articles/claude-code-memory/) and [Codex memory](/articles/codex-memory/) for the other
coding agents.

## How this page is kept current

Each new Claude Code release gets a row in the table, or a note that it has no memory change. Version
numbers and dates come from the GitHub release. Release notes are a summary the vendor writes, so check the
full notes before you rely on a single line here.

Sources: [anthropics/claude-code releases](https://github.com/anthropics/claude-code/releases) (read 2026-11-11);
[Hindsight on GitHub](https://github.com/vectorize-io/hindsight).


## FAQ

### What changed in Claude Code memory recently?

From v2.1.311 to v2.1.323 (October 26 to November 12, 2026), Claude Code fixed stale MEMORY.md and CLAUDE.md content after compaction, stopped overwriting large CLAUDE.md files after reading only part of them, added idle compaction in VS Code and Claude Desktop, and made /context shorter. The table below lists each change with its version.

### Where do I read the Claude Code release notes?

The official notes are on the anthropics/claude-code GitHub releases page. This page copies only the lines that touch memory, compaction, transcripts and context, and gives the version and date for each.

