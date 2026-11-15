# Claude Agent SDK: Install, Sessions and Memory (2026)

> What the Claude Agent SDK does, how to install it in Python or TypeScript, how sessions resume or fork, where CLAUDE.md loads, and how to add long-term memory with Hindsight. From the official docs, read 2026-11-15.

Source: https://aiagentmemory.org/articles/claude-agent-sdk/ · Updated 2026-11-15 · aiagentmemory.org


## What is the Claude Agent SDK?

The **Claude Agent SDK** is a library for Python and TypeScript that runs the Claude Code agent inside your own program. You get the built-in tools (read and edit files, run commands, search the web), the agent loop, permissions, sessions and hooks that Claude Code uses, but you start them from code instead of a terminal. The Python package is `claude-agent-sdk`. The TypeScript package is `@anthropic-ai/claude-agent-sdk`.

Use the SDK when your app should run an agent on its own. Use the CLI for daily work in a terminal, and the Client SDK when you want to write the tool loop yourself.

## How do the Claude tools differ?

The Agent SDK sits between the CLI and the raw API. The official docs compare them like this:

| You want to | Use | What you get |
|---|---|---|
| Embed Claude Code's agent in your own Python or TypeScript app | Agent SDK | A library that runs the Claude Code binary, with its tools, permissions, sessions and hooks |
| Work in a terminal or run one-off tasks | Claude Code CLI | The interactive interface |
| Call the Claude API and write the tool loop yourself | Client SDK | Direct API access; you write the tool loop, or use the beta tool runner |
| Have Anthropic host the agent | Managed Agents | A hosted agent harness, configured through the API |

The Claude Code SDK packages used to be the way to do this. The Agent SDK replaces them, and the migration guide covers the change.

## How do you install and run it?

1. Check the prerequisites: Node.js 18.14 or newer, or Python 3.10 or newer, and an Anthropic API key.
2. Install the package: `pip install claude-agent-sdk` (or `uv add claude-agent-sdk`), or `npm install @anthropic-ai/claude-agent-sdk`. Both bundle the Claude Code binary, so you usually do not install the CLI too.
3. Export the key in the shell that runs your agent: `export ANTHROPIC_API_KEY=your-api-key`. The SDK does not read `.env` files on its own.
4. Run a first agent. This one reads a file, fixes what it finds and reports the result:

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage


async def main():
    async for message in query(
        prompt="Review utils.py for bugs that would cause crashes. Fix any issues you find.",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Glob"],
            permission_mode="acceptEdits",
        ),
    ):
        if isinstance(message, ResultMessage):
            print(f"Done: {message.subtype}")


asyncio.run(main())
```

The official docs say third-party products may not offer claude.ai logins or rate limits on top of the SDK without approval. Use an API key.

## How do sessions work?

A session is the conversation the SDK keeps while the agent works: your prompts, each tool call, each tool result and each reply. It saves the session to disk, so you can return to it. A session keeps the conversation, not the files. To undo file edits, the docs point to file checkpointing.

| Option | What it does | How it finds the session |
|---|---|---|
| `continue_conversation` (Python) / `continue` (TypeScript) | Adds to the most recent session in the directory | No ID needed |
| `resume` | Adds to one named session | You store the session ID from the result message |
| `fork_session` with `resume` | Starts a new session with a copy of the history; the original stays the same | You get the new ID from the result message |

In Python, `ClaudeSDKClient` keeps a multi-turn chat going in one process, with no ID handling. The sessions live in `~/.claude/projects/<encoded-cwd>/`, one `.jsonl` file per session. If you set `CLAUDE_CONFIG_DIR`, the folder moves under that path. To resume on another machine, copy the file or attach a `SessionStore` adapter that mirrors transcripts to your own storage.

One limit matters for apps that change instructions. If you pass a new `system_prompt` or `append` text on a resumed session, Claude does not see it until the session is compacted or you start a new session.

## Where does CLAUDE.md load?

The SDK reads CLAUDE.md through its setting sources. `'project'` loads `CLAUDE.md` or `.claude/CLAUDE.md` from the working directory. `'user'` loads `~/.claude/CLAUDE.md`. The default options enable both, so the files load without extra code. If you set `setting_sources` (Python) or `settingSources` (TypeScript) yourself, list the sources you want. An empty list turns CLAUDE.md off.

For the files the terminal version reads, see [Claude Code memory](/articles/claude-code-memory/).

The system prompt is separate. When you do not set one, the SDK uses a short default that covers tool calls only. The `claude_code` preset adds Claude Code's full instructions, and `append` adds your own text after them.

## Where does Hindsight fit?

Sessions remember one thread of work. They do not give you a memory of the user or the project that you can search across many threads and agents. That is the job of a memory server.

- **Hindsight's Claude Agent SDK package** (`pip install hindsight-claude-agent-sdk`) exposes Hindsight to the agent. Its docs describe retain, recall and reflect as in-process MCP tools that the agent calls when it decides to. You create the server with `create_hindsight_server(bank_id=..., hindsight_api_url=...)`, pass it as `mcp_servers={"hindsight": server}`, and allow `mcp__hindsight__*` in `allowed_tools`. The docs also describe hooks that inject memories before each turn and save each turn afterward.
- **Memory is scoped by bank.** One `bank_id` holds one agent's or one user's memory. Hindsight Cloud needs an API key. A self-hosted server works too; the guide uses `http://localhost:8888`.
- **Strength:** on BEAM 10M, a long-history test, Hindsight scores 64.1 and Mem0 scores 48.6. Both figures come from Mem0's own comparison page, not from an independent test.
- **What the Hindsight page does not say:** it does not name the hook functions or the exact tool names, and it does not explain how subagent runs are saved or recalled. Test those before you rely on them. Ask the agent a question that only an older session can answer.
- **Limit:** Hindsight returns more context per recall than Mem0 by default, and each save calls an LLM. Lower the token budget if the agent's prompt grows too large. For a single-shot script, a session alone may be enough.

For the wider picture, see [Hindsight as agent memory](/articles/ai-memory-hindsight/) and [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/). Hooks in the terminal version are covered in [Claude Code hooks for memory](/articles/claude-code-hooks-memory/).

## Sources

- Claude Agent SDK overview, quickstart, sessions and system prompt pages, code.claude.com/docs/en/agent-sdk/ (read 2026-11-15). Install commands, option names, session paths and CLAUDE.md loading all come from these pages.
- Hindsight Claude Agent SDK page, hindsight.vectorize.io/sdks/integrations/claude-agent-sdk (read 2026-11-15): package name, install command, `create_hindsight_server` and the MCP wiring.
- BEAM 10M figures (64.1 and 48.6): Mem0's comparison page, mem0.ai/compare/mem0-vs-hindsight.


## FAQ

### What is the Claude Agent SDK?

The Claude Agent SDK is a Python and TypeScript library that runs the Claude Code agent inside your own program. You get Claude Code's built-in tools, agent loop, permissions, sessions and hooks, called from code. Install it with pip install claude-agent-sdk or npm install @anthropic-ai/claude-agent-sdk, and set ANTHROPIC_API_KEY.

### Does the Claude Agent SDK remember things between runs?

Only inside one session. A session keeps the conversation on disk so you can continue, resume or fork it. It does not keep facts about the user or the project across unrelated sessions. For that, add a memory server such as Hindsight, or use CLAUDE.md files, which the SDK loads on each run.

### Does the Claude Agent SDK load CLAUDE.md?

Yes, by default. The default options turn on the user and project setting sources, so ~/.claude/CLAUDE.md and the project CLAUDE.md load. If you set setting_sources or settingSources yourself, list the sources you need. An empty list turns CLAUDE.md off.

