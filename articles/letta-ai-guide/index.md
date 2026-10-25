# Letta (formerly MemGPT): Memory, Setup and Pricing

> What Letta is today: the MemGPT successor, how its memory works (MemFS, memory blocks, dreaming, archival memory), how to use it from Python, and its pricing.

Source: https://aiagentmemory.org/articles/letta-ai-guide/ · Updated 2026-10-25 · aiagentmemory.org


**Letta** is the open-source platform for stateful agents built by the team behind **MemGPT**. A Letta agent keeps its memory, identity and conversation history across sessions, and rewrites its own memory as it learns. In 2026 the main product is **Letta Code**, an agent harness with a CLI, desktop app and TypeScript SDK, backed by optional Letta Cloud hosting.

That's a big change from the Letta most tutorials describe. The old Python API server is retired, and a lot of guides online now point at code paths Letta no longer supports. This page sticks to what the current [Letta Code repository](https://github.com/letta-ai/letta-code) and [docs](https://docs.letta.com) say as of October 2026.

## What is Letta AI?

**Letta is a harness for stateful AI agents: the runtime that connects an LLM to persisted context, manages turns, and runs tools.** Each agent has long-lived memory stored as files it can read and edit, a searchable message history, and background "dreaming" that consolidates lessons. It grew out of the 2023 MemGPT research paper.

The [MemGPT paper](https://arxiv.org/abs/2310.08560) (Packer et al., 2023) framed the LLM as an operating system. A small, fast "main context" sits in the prompt. Larger external stores sit outside it. The model itself decides what to page in and out, using function calls. That idea, an agent that manages its own memory, is still the core of Letta.

The project has changed form several times:

| Period | What "Letta" meant |
|---|---|
| 2023 | MemGPT, a research project and Python library |
| 2024 to 2026 | Letta, a Python API server (`letta-ai/letta`) with a REST API and Python/TypeScript clients |
| 2026 | Letta Code (`letta-ai/letta-code`): CLI, desktop app, App Server, TypeScript Agent SDK, Letta Cloud |

The `letta-ai/letta` README now says the current source lives in `letta-code`, and that the `archive` branch holds "the retired Letta V1 API server." Old tags still exist for reproducibility. The last release on that repository was 0.16.8, in May 2026.

## How Letta memory works

Letta's memory model has three layers: memory the agent always sees, memory it searches, and background processes that clean it up. The names changed between versions, so it helps to map old terms to new ones.

### Core memory: memory blocks, now memory files

In the classic Letta API, **core memory** was a set of **memory blocks**. Each block had a label (like `persona` or `human`), a value, and a size limit. Blocks were pinned into the context window on every turn, and the agent edited them with built-in memory tools. Blocks could also be shared between agents.

Letta Code keeps the idea but stores it differently. Memory lives in **MemFS**, which the [MemFS docs](https://docs.letta.com/concepts/memfs) call "a git-backed memory filesystem." Each memory is a Markdown file with YAML front matter, and its label maps to a path: a memory labeled `persona` is `persona.md`. Root-level files load into the system prompt on every turn. Subdirectories with their own `MEMORY.md` index stay out of context until the agent needs them.

Every memory edit is a git commit. That gives you version history, and you can sync an agent's memory to your own GitHub repository with `/memory-repository set`.

### What changed in Letta Code memory in October 2026

Letta Code ships several releases a week. These are the memory changes since early October, from the [release notes](https://github.com/letta-ai/letta-code/releases) and the linked pull requests:

| Version (date) | Change | What it means |
|---|---|---|
| 0.34.5 (Oct 8) | Local agents always start on the `MEMORY.md` layout | New local agents use **MemFS v2**: root files in the prompt, subfolders behind `MEMORY.md` indexes |
| 0.34.7 (Oct 8) | Built-in `Memory` tool ([PR 4884](https://github.com/letta-ai/letta-code/pull/4884)) | Opens deferred memory one level at a time: a folder's `MEMORY.md` plus up to 10 children, most recently edited first, each with its description. Before, agents guessed paths with `Read`/`LS`/`Bash` |
| 0.34.14 (Oct 14) | Local backend honors `memfs: false` | You can run a local agent without MemFS |
| 0.34.18 (Oct 19) | `maintaining-core-memory` and `maintaining-deferred-memory` skills ([PR 5156](https://github.com/letta-ai/letta-code/pull/5156)) | A written procedure for "remember this": merge corrections into existing principles, keep names, dates and quotes in the git commit message, not the memory file |
| 0.34.19 (Oct 20) | Subagents get a pointer to the parent's memory folder ([PR 5193](https://github.com/letta-ai/letta-code/pull/5193)) | Claude Code and Codex workers launched by Letta can read the user and project context the parent agent keeps |

The PR for the skills explains why they were needed: before, an agent asked to "remember this" in a normal chat had no procedure and often wrote subject knowledge into a root file, so it landed in the system prompt on every turn. As of 25 October 2026 the `letta-code` repository has about 3,700 GitHub stars and the archived `letta` server repository about 25,300.

### Archival memory and recall memory

The MemGPT design had two out-of-context stores:

- **Recall memory**: the full message history, searchable by the agent.
- **Archival memory**: a vector-indexed store of passages the agent (or your code) inserts and searches later.

Both still exist in the Letta REST API. The Python client exposes `agents.passages.create` and `agents.passages.search` on the `/archival-memory` endpoints. In Letta Code, message search is a first-class command (`/search`, `letta messages search`). On Letta Cloud it supports full-text, vector and hybrid search; local backends use full-text matching only, according to the MemFS docs.

### Sleep-time agents, now called dreaming

**Sleep-time compute** comes from a 2025 Letta paper ([Lin et al., arXiv 2504.13171](https://arxiv.org/abs/2504.13171)). The model does work between queries, so less work is needed when a question arrives. The paper reports about 5x less test-time compute for the same accuracy on its stateful math benchmarks.

In the old API this was the `enable_sleeptime` flag, which moved memory management to a background agent. In Letta Code it's called **dreaming**. Background subagents "review recent conversations, consolidate useful lessons, and update memory" (Letta memory docs). You configure it with `/sleeptime`, and it can run after a set number of agent steps or when the context window is compacted. There's an optional second review pass before updates apply, which costs more tokens.

### Skills and self-editing

Letta Code agents also learn **skills**: reusable instruction files stored in MemFS, in a project's `.agents/skills`, or globally in `~/.letta`. Commands like `/remember` tell the agent to file a lesson where it belongs, and `/doctor` audits memory for duplication and prompt bloat.

## How to get started with Letta

The supported path is the Letta Code CLI. Here's the shortest route from the README:

1. Install it: `npm install -g @letta-ai/letta-code`, or `uv tool install letta` (the PyPI `letta` package now ships the Letta Code CLI with a bundled Node runtime).
2. Run `letta` in your project directory.
3. Choose Letta Cloud or local mode on first launch. Change later with `letta backend cloud` or `letta backend local`.
4. Run `/connect` to add your own model keys (OpenAI, Anthropic and others) and `/model` to switch models.
5. Run `/init` so the agent inspects the repo and builds its first memory.
6. Use `letta server` to run the App Server for self-hosted or always-on agents.

### Using Letta from Python

This is where older guides go wrong. Letta's FAQ answers "Is there a Python version of the SDK?" with: "No. The Letta Agent SDK is currently available only for TypeScript." When Python is a hard requirement, Letta suggests running the App Server and connecting to its WebSocket protocol from Python.

The generated REST client, `letta-client` on PyPI (version 1.12.1, June 2026), still works against agents hosted in Letta Cloud with a `LETTA_API_KEY`. Its docs page says the package "is deprecated" and "receives no new features," so treat it as a bridge for existing code, not a base for new projects. With that caveat, this is what the classic memory model looks like through it:

```python
import os
from letta_client import Letta

client = Letta(api_key=os.environ["LETTA_API_KEY"])  # Letta Cloud

# Core memory: blocks pinned into the context window on every turn
agent = client.agents.create(
    name="support-agent",
    model="openai/gpt-4o-mini",  # handle format: provider/model-name
    memory_blocks=[
        {"label": "human", "value": "Name: Dana. Plan: Pro. Prefers short answers."},
        {"label": "persona", "value": "You are a concise support agent."},
    ],
)

# Archival memory: passages stored outside the context, searched on demand
client.agents.passages.create(agent.id, text="Refunds are processed within 5 business days.")

response = client.agents.messages.create(agent.id, input="How long do refunds take?")
for message in response.messages:
    if message.message_type == "assistant_message":
        print(message.content)

# Read back what the agent now holds in core memory
print(client.agents.blocks.retrieve("human", agent_id=agent.id).value)
```

The method names and parameters above match `letta-client` 1.12.1. Check the package changelog before relying on it, since no new features are planned.

## Letta pricing and hosting

Letta Code itself is Apache 2.0 and free to run. You pay your model provider. Hosted plans, from the [Letta pricing page](https://docs.letta.com/letta-code/pricing) as of October 2026:

| Plan | Price | What you get |
|---|---|---|
| Free | $0/month | Limited agents and Letta Auto usage; bring your own API keys |
| Pro | $20/month | Letta Auto weekly and monthly quotas, pay-as-you-go overage, up to 20 stateful agents |
| API | $20/month | Unlimited agents, $0.10 per active agent per month, $0.00015 per second of tool execution, API keys, pay-as-you-go LLM usage |
| Teams Pro | $20/seat/month | Shared agents with access control |
| Enterprise | Custom | RBAC, SAML/OIDC SSO, dedicated support |

Hosting choices:

- **Local backend**: agents and MemFS stay on your disk. Letta's docs warn you're responsible for backing them up.
- **Letta Cloud**: agent state, memory and conversations live in Letta's cloud, while tools run on your laptop, a remote computer or a managed sandbox.
- **Self-hosted App Server**: `letta server` runs the full runtime on your own machine.

One thing to know if you're migrating: the open-source Docker image "is no longer supported as a backend for Letta Code" (Letta deprecated docs). AgentFile (`.af`) import and export were also removed.

## When Letta fits, and when something else does

Letta fits when the agent itself is the product: a long-lived assistant or coding agent that edits its own memory, learns skills and runs on a schedule. Its memory is readable Markdown under git, which makes it easy to inspect and diff.

It fits less well when you want a memory layer to bolt onto an agent you already built in another framework, or a Python-first SDK. Tools like Mem0, Zep, Cognee and [Hindsight](https://github.com/vectorize-io/hindsight) are built as memory services that sit beside your agent rather than as the agent runtime.

Hindsight is the closest match to Letta's "learns over time" idea for teams that want to keep their own agent loop. Like dreaming, it consolidates in the background: related facts merge into **observations** that keep their evidence, and **mental models** hold standing answers that stay up to date. It has a maintained Python client (plus TypeScript and Go), is MIT-licensed with every feature in the open-source server, and ships plugins for Claude Code, Codex, Cursor and Copilot, so existing coding agents get memory without switching harness. On the long-history BEAM benchmark it reports 73.9 at 1M tokens and 64.1 at 10M (vendor-reported). What it doesn't do: run the agent, its tools or its schedule, which Letta does. Each write and consolidation pass also calls an LLM.

Those tradeoffs are covered in our guides to [Letta alternatives](/articles/letta-alternatives/), [Mem0 alternatives](/articles/mem0-alternatives-compared/), [Zep alternatives](/articles/zep-alternatives/) and [Cognee alternatives](/articles/cognee-alternatives/). For the idea Letta started from, see [hierarchical memory for LLMs](/articles/llm-hierarchical-memory/).


## FAQ

### Is Letta the same as MemGPT?

Yes. Letta is the company and project built by the MemGPT researchers, and the GitHub README calls it 'Letta (f.k.a. MemGPT)'. The MemGPT paper (Packer et al., 2023) introduced the idea of an LLM that manages its own memory tiers, and Letta is the product that grew from it.

### Does Letta have a Python SDK?

Not for new projects. Letta's FAQ says the Agent SDK is TypeScript only. The generated Python package, letta-client, still talks to the Letta REST API, but its docs page marks it as deprecated and unmaintained. For Python, Letta suggests running the App Server and talking to its WebSocket protocol.

### What is the Memory tool in Letta Code?

A built-in tool added in Letta Code 0.34.7 (October 2026) for MemFS v2 agents. Root memory files are always in the system prompt; everything in subfolders is deferred. Memory('folder') returns that folder's MEMORY.md index plus up to 10 children, most recently edited first, and Memory('folder/file.md') returns one file, so the agent no longer guesses paths with Read or Bash.

### How much does Letta cost?

Letta Code is open source (Apache 2.0) and free to run locally with your own model keys. Letta's hosted plans, as of October 2026, are Free ($0), Pro ($20/month), an API plan ($20/month plus $0.10 per active agent per month and tool execution time), Teams Pro ($20 per seat per month) and custom Enterprise pricing.

