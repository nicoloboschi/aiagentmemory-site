# Honcho LLM Memory: Peers, Reasoning and Setup

> What Honcho is: Plastic Labs' open-source memory for LLM agents. How peers, sessions and background reasoning work, Python SDK, self-hosting, pricing, evals.

Source: https://aiagentmemory.org/articles/honcho-llm-memory/ · Updated 2026-10-23 · aiagentmemory.org


**Honcho** is an open-source memory service for LLM agents, built by Plastic Labs. Instead of only storing extracted facts, it **reasons about the people and agents in your conversations**. You save messages; a background worker derives conclusions about each participant; then you ask Honcho what it knows, in plain language or as prompt-ready context.

In short: Honcho memory is a peer model, not a fact list. It is AGPL-3.0, self-hostable on Postgres with pgvector, priced at $2 per million ingested tokens on the managed service, and built into Hermes Agent as a memory provider. The current server is **3.4.0** (22 October 2026).

This page covers Honcho's data model, how its reasoning pipeline works, the Python SDK, self-hosting, what changed in 3.4, pricing and its benchmark claims. Sources are the [Honcho GitHub README](https://github.com/plastic-labs/honcho), the [Honcho docs](https://honcho.dev/docs/v3), [honcho.dev](https://honcho.dev/) and the [v3.4.0 release notes](https://github.com/plastic-labs/honcho/releases/tag/v3.4.0), checked on 23 October 2026.

## What is Honcho?

**Honcho is memory infrastructure for stateful agents. It stores messages and events on sessions, reasons over them asynchronously, and maintains a representation of every participant, called a peer. Agents query those representations, search history, or ask natural-language questions such as "what learning style does this user respond to?"**

Its README describes it as memory for agents "that understand changing people, agents, groups, projects, and ideas over time." The server is a FastAPI app, licensed **AGPL-3.0**, at version 3.4.0 (22 October 2026), with about 7,700 GitHub stars. The Python SDK is `honcho-ai` (2.7.0) and the TypeScript SDK is `@honcho-ai/sdk`.

The emphasis is **user modeling** rather than retrieval. Most memory layers answer "which stored facts match this query?" Honcho also tries to answer "what kind of person is this, and what do they likely want?"

## Honcho's data model: workspaces, peers and sessions

| Concept | What it is |
|---|---|
| **Workspace** | Top-level container that isolates one app or use case (formerly "App") |
| **Peer** | Any participant, human or AI agent (formerly "User") |
| **Session** | A conversation; many-to-many with peers |
| **Scope** | A named group of sessions that limits what recall can see |
| **Message** | One unit of data: a chat turn, an event, or an ingested document chunk |

The **peer model** is Honcho's distinctive choice. Humans and agents are both first-class, sessions can have many of each, and you can configure which peers observe which. Internally, observations live in collections keyed by an `(observer, observed)` pair, so Honcho can hold both a peer's view of itself and **peer X's understanding of peer Y**. That matters for multi-agent setups where agents shouldn't share everything.

## How Honcho's memory pipeline works

Honcho splits into two services, per its architecture section:

- **Storage** is synchronous: workspaces, peers, sessions and messages, written through the API.
- **Insights** is asynchronous: a background worker called the **deriver** consumes a queue and produces conclusions, representations, summaries and peer cards. It also runs **dreaming**, a background consolidation task.

The loop the README describes:

1. **Store** conversations, events, documents or tool traces as messages on a session.
2. **Reason.** The deriver processes the queue and updates peer representations.
3. **Query.** Ask for context, search results, a representation, or a natural-language answer.
4. **Inject** the result into any LLM call.

What you can read back:

- **Conclusions**: deductive and inductive statements Honcho has derived about a peer.
- **Representations**: low-latency snapshots of what Honcho knows, optionally per session.
- **Peer cards**: compact identity summaries.
- **Session context**: prompt-ready bundles with summaries, sized to a token budget.
- **Chat endpoint** (the "dialectic" settings in the config): a reasoning model that answers questions about a peer, at a reasoning level from `minimal` to `max`.

Because reasoning is asynchronous, new messages take a moment to show up in chat answers. The docs suggest the representation endpoint for low-latency reads. Search over messages is hybrid BM25 plus vector.

## Using Honcho from Python

This follows the README quickstart, which is unchanged in `honcho-ai` 2.7.0:

```python
import os
from honcho import Honcho
from openai import OpenAI

honcho = Honcho(
    workspace_id="tutoring-app",
    api_key=os.environ["HONCHO_API_KEY"],  # self-hosted: base_url="http://localhost:8000"
)

student = honcho.peer("student-42")
tutor = honcho.peer("tutor")
session = honcho.session("lesson-1")

session.add_messages([
    student.message("Can you explain fractions with a picture? Text walls lose me."),
    tutor.message("Sure, let's draw a pizza cut into eight slices."),
])

# Natural-language question about the peer (answered by Honcho's reasoning)
print(student.chat("How does this student prefer to learn?", reasoning_level="low"))

# Prompt-ready context for the next turn, capped at a token budget
context = session.context(summary=True, tokens=4_000)
reply = OpenAI().chat.completions.create(
    model="gpt-4o-mini",
    messages=context.to_openai(assistant=tutor),
)
print(reply.choices[0].message.content)
```

`peer.search(...)`, `peer.representation(...)` and `peer.card()` give the other read paths, and `honcho.chat(...)` asks a question across the whole workspace instead of one peer. `student.chat(..., target=tutor)` asks what one peer knows about another. A `to_anthropic()` formatter exists alongside `to_openai()`.

## Self-hosting Honcho

Three ways to run it:

1. **Managed** at api.honcho.dev. Each organization gets its own dedicated Honcho instance, per the README.
2. **Local stack with the CLI**: `uv tool install honcho-cli`, then `honcho start --setup`. It writes your LLM provider key and starts the API, the deriver, Postgres and Redis in Docker.
3. **From source**: clone the repo, copy `docker-compose.yml.example` and `.env.template`, add LLM keys, and run `docker compose up`.

Honcho needs **PostgreSQL with pgvector**. It uses several model providers by default: Gemini for the deriver, summaries and low reasoning levels; Anthropic for higher reasoning levels and dreaming; and OpenAI for embeddings. Each can be reconfigured. Running a self-hosted instance means running those LLM calls on your own keys.

The **AGPL-3.0** license is worth checking with your legal team. It requires you to share source changes if you offer a modified Honcho to users over a network.

## Integrations for coding agents

Honcho ships first-party memory plugins for **Claude Code, Codex, Cursor, OpenCode, OpenClaw** and the DeepSeek Harness, and it's built into **Hermes Agent** as a memory provider (`hermes memory setup`). All of them read one `~/.honcho/config.json`, so pointing two tools at the same workspace gives them shared memory. A hosted MCP server at mcp.honcho.dev covers other clients. [Hindsight](https://github.com/vectorize-io/hindsight) covers the same ground with native Claude Code, Codex, Cursor and Copilot plugins, an MCP server per memory bank and 60+ integrations in total. See [Hermes Agent memory](/articles/hermes-agent-memory/) and OpenClaw memory for how Honcho compares with other providers there.

## What changed in Honcho 3.4

Honcho 3.4.0 shipped on 22 October 2026. The changes that affect memory, from the [release notes](https://github.com/plastic-labs/honcho/releases/tag/v3.4.0):

| Change | What it does | Default |
|---|---|---|
| Named-scope API keys | A key bound to one scope can only recall from and write into that scope's sessions. Useful when one agent or tenant must not see the rest of a workspace. | New keys only; older servers reject them |
| `uid` on workspaces, peers and sessions | A fixed internal id next to your chosen `id`, so references survive if names become renameable | Additive, no migration |
| Dreamer working memory | The dreamer sees its remaining tool budget, a brief of how memory is laid out, and keeps capped notes between dreams; deduction and induction get separate iteration limits | Off (`DREAM_BRIEF_ENABLED`, `DREAM_NOTES_MODE` and others) |
| Stricter deriver | Malformed model output is retried and then fails, instead of being dropped silently as "nothing learned" | On |
| Database migration | Adds a counter and five partial indexes built `CONCURRENTLY`; one needs the `btree_gin` extension | Runs on upgrade |

For self-hosters, the migration is the part to plan. The index builds wait for open transactions on the queue table and fail after 60 minutes; on a large or busy database, the notes say to build the indexes ahead of time with the statements in the troubleshooting guide. Scoped keys are minted with `scripts/generate_jwt.py` on self-hosted instances. The high-level Python SDK and CLI helpers still try to create the workspace, so a restricted key has to go through REST or the SDKs' HTTP transports for now. [Hindsight](https://github.com/vectorize-io/hindsight) handles the same need differently: you give each user, project or tenant its own memory bank, and banks are isolated from each other.

The silent-drop fix matters for anyone running the deriver on an OpenAI-compatible local model: before 3.4, a batch the model answered in the wrong shape could vanish without an error, so memory had gaps you couldn't see.

## Honcho pricing

From [honcho.dev](https://honcho.dev/) in October 2026:

| Item | Price |
|---|---|
| Ingestion (storage plus reasoning) | $2.00 per 1M tokens |
| `context()` retrieval | No limit listed |
| Dreaming | Included |
| Chat (reasoning) queries | $0.001 (minimal), $0.01 (low), $0.05 (medium), $0.10 (high), $0.50 (max) per query |
| Startups under $5M raised | $1,000 credits and 12 months of subsidized pricing |

New accounts get free starter credits; the amount shown differs between the site and the docs, so check at sign-up.

## Honcho benchmark results

Plastic Labs publishes evals at [honcho.dev/evals](https://honcho.dev/evals/), with configurations in the `plastic-labs/honcho-benchmarks` repo:

| Benchmark | Honcho | Claude Haiku 4.5 with full context | Hindsight (its own site) |
|---|---|---|---|
| LongMemEval-S | 90.4% | 62.6% | 94.6% |
| LoCoMo | 89.9% | 75.6% | 92% |
| BEAM 100K | 0.630 | 0.533 | 75 (0.75) |
| BEAM 10M | 0.409 | not reported | 64.1 (0.641) |

The default setup used gemini-2.5-flash-lite for the deriver and claude-haiku-4.5 for dreaming and chat. These are **self-reported** results, graded by an LLM judge, and other vendors use different models and judges. The Hindsight column comes from its [benchmark site](https://benchmarks.hindsight.vectorize.io/), also vendor-run with its own setup, so read it as a rough guide rather than a head-to-head test. Our [LLM memory comparison](/articles/llm-memory-comparison/) lists claims side by side, and [LLM memory evaluation](/articles/llm-memory-evaluation/) explains the benchmarks.

## When Honcho fits

Honcho fits when understanding the person matters more than recalling a fact: tutors, coaches, companions, and support agents that should adapt to how each user thinks. It's also one of the few systems that models what one agent knows about another, which helps in multi-agent apps.

It fits less well when you need document RAG, explicit fact validity windows, or a permissive license. Mem0 and Graphiti (Apache 2.0) and Hindsight (MIT) are alternatives with different memory models.

Hindsight is the closest match if you like Honcho's background reasoning but want a permissive license. Its open-source server includes consolidation into evidence-backed **observations**, standing **mental models**, and a `reflect` call that reasons over memories, much like Honcho's chat endpoint. Its published scores are higher on every benchmark both report, with the widest gap at BEAM 10M (64.1 vs 40.9), though neither vendor ran the other's setup. Honcho's peer model of what one participant knows about another is its own; Hindsight's README doesn't list an equivalent. Like Honcho, it calls an LLM on every write. See [Mem0 vs Honcho](/articles/mem0-vs-honcho/), [Supermemory alternatives](/articles/supermemory-alternatives/) and [Mem0 alternatives](/articles/mem0-alternatives-compared/). For how user modeling fits into agent memory overall, start with [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What is Honcho?

Honcho is an open-source memory service for LLM agents from Plastic Labs. You store conversations as messages on sessions; Honcho reasons over them in the background and builds a representation of each participant (a peer), which you can query in natural language or pull as prompt-ready context.

### Is Honcho open source?

Yes. The Honcho server is licensed AGPL-3.0 on GitHub (plastic-labs/honcho). You can self-host it with Docker Compose or run a local stack with the honcho CLI. Plastic Labs also runs a managed service at api.honcho.dev.

### How is Honcho different from Mem0?

Mem0 extracts facts from messages and returns them by search. Honcho stores the messages themselves and runs background reasoning to derive conclusions about each peer, including what one peer knows about another, and answers natural-language questions about them through its chat endpoint. Mem0 is Apache 2.0; Honcho is AGPL-3.0.

### What is new in Honcho 3.4?

Honcho 3.4.0 (22 October 2026) adds API keys bound to one named scope, a fixed uid on workspaces, peers and sessions, opt-in working memory for the dreamer, and a deriver that fails loudly instead of silently dropping malformed batches. It includes a database migration with concurrent index builds that self-hosters should plan for on large databases.

