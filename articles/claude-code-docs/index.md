# Claude Code Docs Map: Where Each Topic Lives (2026)

> Which official Claude Code doc page covers memory, skills, hooks, settings, MCP, costs and install. The page to read for each, from the docs index, read 2026-11-16.

Source: https://aiagentmemory.org/articles/claude-code-docs/ · Updated 2026-11-16 · aiagentmemory.org


## Where are the Claude Code docs?

The official docs are at **code.claude.com/docs**. Anthropic also publishes an index of every English page at code.claude.com/docs/llms.txt (read 2026-11-16). The index groups pages into sections: Getting started, Build with Claude Code, Plugins, Administration, Configuration, Reference, Agent SDK and Resources. It also links to translated indexes, French, German, Chinese and others, each with 205 pages.

## Which page covers which topic?

Each row names the official page, the words the docs use for it (from the index), and the guide on this site that goes with it.

| Topic | Official page | What the docs say it covers | Guide on this site |
| :- | :- | :- | :- |
| Install and sign in | `/en/quickstart` | "Install Claude Code in your terminal, sign in, and use the CLI to explore your codebase and make your first code change." | [How to install Claude Code](/articles/install-claude-code/) |
| Memory | `/en/memory` | "Give Claude persistent instructions with CLAUDE.md or AGENTS.md files, and let Claude accumulate learnings automatically with auto memory." | [Claude Code memory](/articles/claude-code-memory/) |
| Settings | `/en/settings` and `/en/settings-reference` | "Complete reference for every Claude Code settings.json key: where each one goes, its type and default." | [Claude Code settings](/articles/claude-code-settings/) |
| Skills | `/en/skills` | "Create, manage, and share skills to extend Claude's capabilities in Claude Code." | [Claude Code skills](/articles/claude-code-skills/) |
| Subagents | `/en/sub-agents` | "Create and use specialized AI subagents in Claude Code for task-specific workflows and improved context management." | [Claude Code subagents](/articles/claude-code-subagents/) |
| Hooks | `/en/hooks-guide` and `/en/hooks` | "Run shell commands automatically when Claude Code edits files, finishes tasks, or needs input." The reference lists the events and exit codes. | [Claude Code hooks and memory](/articles/claude-code-hooks-memory/) |
| Commands | `/en/commands` | "Complete reference for commands available in Claude Code, including built-in commands and bundled skills." | [Claude Code commands](/articles/claude-code-commands/) |
| MCP servers | `/en/mcp-quickstart` and `/en/mcp` | "Add an MCP server to Claude Code, verify the connection, and find the configuration on disk." | [Claude Code MCP](/articles/claude-code-mcp/) |
| Plugins | `/en/plugins/overview` | "Understand what a Claude Code plugin is, when you need one instead of a standalone skill or MCP server." | [Claude Code plugins](/articles/claude-code-plugins/) |
| Costs | `/en/costs` | "Track token usage, set team spend limits, and reduce Claude Code costs with context management, model selection, extended thinking settings, and preprocessing hooks." | [Claude Code cost](/articles/claude-code-cost/) |
| Context window | `/en/context-window` | "An interactive simulation of how Claude Code's context window fills during a session." | [Claude Code commands](/articles/claude-code-commands/) (`/context`) |
| Config not taking effect | `/en/debug-your-config` | "Diagnose why CLAUDE.md, settings, hooks, MCP servers, or skills aren't taking effect. Use /context, /doctor, /hooks, and /mcp." | [Claude Code settings](/articles/claude-code-settings/) |
| Release notes | `/en/changelog` | "Release notes for Claude Code, including new features, improvements, and bug fixes by version." | [Claude Code release notes and memory](/articles/claude-code-release-notes-memory/) |

## How do you find a topic fast?

1. Open code.claude.com/docs/llms.txt and search the page for the topic word.
2. Read the one-line description under each link. It tells you what the page covers.
3. If a setting does not work, read `/en/debug-your-config` first. It lists the commands that show what loaded.
4. Check the changelog before you blame a bug. A fix may be in a newer version.
5. Use the `.md` links in the index. Each one points to the page as Markdown, with no site chrome to skip.

## Where Hindsight fits

The docs above do not mention Hindsight. [Hindsight](https://github.com/vectorize-io/hindsight) is an MIT-licensed memory server from Vectorize. Its Coding Agents package adds memory to Claude Code with one command, `npx @vectorize-io/hindsight-coding-agents install claude-code` (Hindsight coding agents page, read 2026-11-12). Capture is automatic, and each repo gets one bank. The same package covers 22 CLI agents in all.

The limits matter for this topic:

- **Writes cost more.** Each retain calls an LLM to pull out facts, so the docs' cost advice on [Claude Code cost](/articles/claude-code-cost/) applies to memory writes too.
- **The old plugin is retired.** The per-agent Claude Code plugin gets no more updates. The new package does not carry over the old plugin's recall and retain settings.
- **Vendor source.** The install steps come from Hindsight's own page. Check the official Claude Code pages above for what Claude Code itself does with CLAUDE.md and auto memory.


## FAQ

### Where are the official Claude Code docs?

The English docs are at code.claude.com/docs. Anthropic publishes an index of every page at code.claude.com/docs/llms.txt, grouped by section (getting started, build, plugins, administration, configuration, reference, Agent SDK). Read 2026-11-16.

### Which Claude Code doc page explains memory?

The page is titled 'How Claude remembers your project' at code.claude.com/docs/en/memory. It covers CLAUDE.md or AGENTS.md files, which load at the start of each session, and auto memory, which Claude writes for itself.

### Do the Claude Code docs cover third-party memory servers?

No. The official docs cover CLAUDE.md, auto memory and MCP servers. Third-party memory servers such as Hindsight publish their own setup pages. Hindsight's Coding Agents page describes its Claude Code install.

