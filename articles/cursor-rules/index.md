# Cursor Rules: .mdc Files, AGENTS.md and Memory (2026)

> Cursor rules are .mdc files in .cursor/rules, plus User, Team and AGENTS.md rules. The four apply modes, the 500-line guide and Hindsight's Cursor CLI.

Source: https://aiagentmemory.org/articles/cursor-rules/ · Updated 2026-11-20 · aiagentmemory.org


**Cursor rules** are instruction files that Cursor adds to the agent's context. You write them once, and Cursor reads them in later chats. Project rules are `.mdc` files in `.cursor/rules`, which Cursor's docs say is version-controlled and scoped to the codebase. Cursor also reads `AGENTS.md` files, and it has User and Team rules for wider scopes.

This page covers the rule types, how each one is applied, and where Hindsight fits for Cursor users. For a side-by-side with Claude Code's files, see [Claude Code vs Cursor](/articles/claude-code-vs-cursor/), and for Codex's files see [Codex vs Cursor](/articles/codex-vs-cursor/). Facts come from Cursor's rules docs (read 2026-11-15) and Hindsight's coding agents page (read 2026-11-12 and 2026-11-15).

## What are Cursor rules?

**Cursor rules are saved instructions that Cursor includes in the agent's prompt when a rule applies.** The docs say a language model does not keep memory between completions, so rules are how a project's conventions carry from one chat to the next. A rule can state a coding style, a test command or a list of files to avoid. Rules change what the agent sees. They do not make it learn from what happened.

## Which kinds of Cursor rules are there?

Cursor's docs list four types.

| Rule type | Scope | Where it is kept | How it is set |
|---|---|---|---|
| Project Rules | One codebase | `.cursor/rules/*.mdc`, version-controlled | Files in the folder, or `/create-rule` in chat |
| User Rules | All your projects | Your Cursor environment | Customize, then Rules |
| Team Rules | Whole organization | Cursor dashboard, Team and Enterprise plans | Admins; a rule can be enforced so users cannot turn it off |
| AGENTS.md | Project root and its subfolders | Plain Markdown, no frontmatter | Just a file in the folder |

Project rules are the only type the docs place in your repo. Plain `.md` files in `.cursor/rules` are ignored: project rules must use the `.mdc` extension.

## How does Cursor decide when a rule applies?

Each `.mdc` file has three settings in its frontmatter: `description`, `globs` and `alwaysApply`. The Rules screen sets them for you, and the combination picks one of four modes.

1. **Always Apply** (`alwaysApply: true`): included in every chat session.
2. **Apply Intelligently** (a `description` is set): the agent includes the rule when it judges it relevant.
3. **Apply to Specific Files** (`globs` is set): included when a matching file is in context.
4. **Apply Manually** (no description and no globs): included only when you `@`-mention the rule in chat.

The docs give one size limit: "Keep rules under 500 lines." They also suggest splitting a large rule into smaller ones. Cursor does not state a hard technical cap.

## Does Cursor have memories as well?

Older Cursor versions had a Memories feature that saved notes from chats. Cursor staff said on the Cursor forum that it was removed in version 2.1.x, and they suggested exporting the memories and moving them into Rules. The docs link for Memories now opens the Rules page. Treat rules you write as the supported way to keep instructions The [AI coding agent memory](/articles/ai-coding-agent-memory/) guide covers the same change.

## Where Hindsight fits

Cursor's rules are text you maintain by hand, so they hold only what you wrote down. [Hindsight](https://github.com/vectorize-io/hindsight) is an open-source memory server that records what happened in each session. Its Coding Agents package has a Cursor CLI installer:

```bash
npx @vectorize-io/hindsight-coding-agents install cursor-cli
```

According to Hindsight's coding agents page, the installer writes hooks to `~/.cursor/hooks.json` and MCP settings to `~/.cursor/mcp.json`. Capture is automatic, and each repo gets one memory bank, `coding-agent::{gitProject}`, shared with the other agents that work in that repo. The same package covers 22 CLI agents in all, so one bank can hold Cursor and Claude Code sessions for the same project.

- **What it does not cover:** the page documents Cursor CLI only, not the Cursor IDE. It also says past Cursor chats cannot be imported, because Cursor keeps its history in internal SQLite databases with unversioned schemas.
- **Cost:** Hindsight calls an LLM on writes, so each captured session costs more than a line added to a `.mdc` file.
- **Vendor source:** the Hindsight page is written by its maker. Test it on one repo before you rely on it.

For how the other coding agents keep their instructions, see [AI coding agent memory](/articles/ai-coding-agent-memory/). For Claude Code's files, see [Claude Code memory](/articles/claude-code-memory/). For Codex, see [Codex memory](/articles/codex-memory/).

## Sources

- Cursor project rules, rule types and AGENTS.md: [cursor.com/docs/context/rules](https://cursor.com/docs/context/rules) (read 2026-11-15)
- Hindsight Coding Agents, Cursor CLI installer and bank naming: [hindsight.vectorize.io/sdks/integrations/coding-agents](https://hindsight.vectorize.io/sdks/integrations/coding-agents) (read 2026-11-15)


## FAQ

### Where do Cursor rules live?

Project rules are .mdc files in the .cursor/rules folder of your project, and Cursor's docs say they are version-controlled. User Rules are global and are set in Cursor's Customize, then Rules screen. Cursor also reads AGENTS.md files in the project root and its subfolders.

### What is the difference between Always Apply and Apply Intelligently in Cursor?

Always Apply includes the rule in every chat session. Apply Intelligently includes it only when the agent decides the rule is relevant to the task. Apply to Specific Files attaches it when a matching file is in context, and Apply Manually includes it only when you @-mention it.

### Can Hindsight remember what Cursor did across sessions?

Hindsight's Coding Agents package has a Cursor CLI installer that writes hooks into ~/.cursor/hooks.json and captures sessions automatically. Its page documents Cursor CLI only, not the Cursor IDE, and it cannot import past Cursor chats.

