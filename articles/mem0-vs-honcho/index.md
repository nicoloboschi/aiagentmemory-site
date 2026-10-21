# Mem0 vs Honcho: Fact Memory vs User Modeling

> Mem0 vs Honcho compared (Oct 2026): fact extraction vs peer reasoning, multi-agent support, license, self-hosting, benchmarks, pricing, and which to choose.

Source: https://aiagentmemory.org/articles/mem0-vs-honcho/ · Updated 2026-10-12 · aiagentmemory.org


**Mem0 vs Honcho** is a choice between remembering facts and understanding people. Mem0 pulls short facts out of conversations and returns the ones that match a query. Honcho keeps the full messages, runs background reasoning over them, and builds a model of each participant that your agent can ask questions about in plain language.

Both are open source and both have a managed service. They differ in license (Apache 2.0 vs AGPL-3.0), in how much LLM work happens after each write, and in how they treat multi-agent conversations. Many teams weighing the two also look at **[Hindsight](https://github.com/vectorize-io/hindsight)**, an MIT-licensed memory server that, like Honcho, reasons over memories in the background, and posts the highest vendor-reported scores of the three on LongMemEval and BEAM. Sources are each project's README, docs, pricing page and published evals, checked on 12 October 2026.

## What are Mem0 and Honcho?

**Mem0 is an Apache 2.0 memory layer that uses an LLM to extract durable facts from conversations, stores them per user, agent or run, and returns the relevant ones by hybrid search. Honcho is an AGPL-3.0 memory service from Plastic Labs that stores messages on sessions, reasons over them asynchronously, and keeps a queryable representation of every participant.**

**[Mem0](https://github.com/mem0ai/mem0)** runs as a Python or TypeScript library, a self-hosted Docker server, or the hosted Mem0 Platform. It has about 67,000 GitHub stars, the most of any agent memory project. See [what Mem0 is](/articles/what-is-mem0-ai/).

**[Honcho](https://github.com/plastic-labs/honcho)** is a FastAPI server (version 3.3.0) with Python (`honcho-ai`) and TypeScript SDKs, about 7,500 stars, and a managed service at api.honcho.dev. Our [Honcho guide](/articles/honcho-llm-memory/) covers its data model and setup in depth.

## Mem0 vs Honcho at a glance

| | Mem0 | Honcho | Hindsight |
|---|---|---|---|
| License | Apache 2.0 | AGPL-3.0 | MIT |
| What gets stored | Short extracted facts with entities | Raw messages, plus derived conclusions per peer | Facts, experiences, consolidated observations, mental models |
| Scope keys | `user_id`, `agent_id`, `run_id` | Workspace, peer, session, scope | Memory bank (`bank_id`) |
| Write path | One LLM extraction pass, add-only | Message stored at once; background "deriver" reasons later | LLM extraction of facts, time, entities, relations; background consolidation |
| Read paths | `search` (semantic + BM25 + entity) | Context bundle, representation, peer card, search, natural-language chat | `recall` (semantic + BM25 + graph + temporal, reranked), `reflect`, mental models |
| Multi-agent views | Facts tagged by agent | Per-observer views: what peer X knows about peer Y | Separate banks per user or agent |
| Self-host needs | Library: your LLM, embedder, vector store. Server: Postgres + pgvector | Postgres + pgvector, Redis, and LLM keys (Gemini, Anthropic, OpenAI by default) | PostgreSQL + pgvector (or embedded), any of 25+ LLM providers, local ones included |
| Coding agent support | Agent skills for Claude Code, Codex, Cursor and others; MCP | Plugins for Claude Code, Codex, Cursor, OpenCode, OpenClaw; built into Hermes; MCP | Native plugins for Claude Code, Codex, Cursor, Copilot; MCP server per bank |
| Hosted pricing | Free; Starter $19/mo; Pro $249/mo | $2 per 1M tokens ingested; chat queries $0.001 to $0.50 each | Hindsight Cloud: pay as you go, retain $10 and recall $0.75 per 1M tokens, free credits |

Sources: the [Mem0 README](https://github.com/mem0ai/mem0), the [Honcho README](https://github.com/plastic-labs/honcho), the [Hindsight README](https://github.com/vectorize-io/hindsight), [Mem0 pricing](https://mem0.ai/pricing) and [honcho.dev](https://honcho.dev/).

## How each one builds memory

**Mem0** does its work at write time. After an exchange, your code sends the messages to `add`. Since the token-efficient algorithm Mem0 introduced in April 2026, an LLM extracts facts in a single pass and Mem0 only adds them; a changed fact sits next to the old one until you call `update` or `delete`. On `search`, it scores semantic similarity, BM25 keywords and entity overlap and fuses them. You get back a short list of facts such as "prefers short answers."

**Honcho** splits the job in two. Storage is synchronous: messages land on a session right away. Insights are asynchronous: a background worker called the **deriver** reads the queue and writes **conclusions** (deductive and inductive statements about a peer), **representations**, summaries and **peer cards**. A consolidation task called **dreaming** runs on top. Because reasoning lags the write, a brand-new message may not show up in chat answers for a moment.

The practical difference: Mem0 gives you facts you can inspect and edit one by one. Honcho gives you an evolving picture of a person, including things nobody said outright, such as "learns better from pictures than text." For the general pattern behind both, see [AI agent memory explained](/articles/ai-agent-memory-explained/).

## Multi-agent memory: Honcho's peer model

Mem0 tags each memory with a `user_id`, `agent_id` or `run_id`, so different agents can keep separate memories. Every fact is still one shared statement about the world.

Honcho treats humans and agents the same way: both are **peers**, and a session can hold many of each. Observations are stored by an `(observer, observed)` pair, so Honcho can keep a peer's view of itself and, separately, what peer X has learned about peer Y. `student.chat("...", target=tutor)` asks what one peer knows about another. If your agents shouldn't all know the same things, for example in a negotiation or a game, this is a real difference.

## Code: the same task in each

Store something about a user, then fetch context before the next turn. Mem0, with the open-source library (defaults to OpenAI, needs `OPENAI_API_KEY`):

```python
from mem0 import Memory

memory = Memory()
memory.add("Dana prefers short answers and works in Go.", user_id="dana")

hits = memory.search("How should I answer Dana?", filters={"user_id": "dana"}, top_k=5)
context = "\n".join(h["memory"] for h in hits["results"])
```

Honcho, with the `honcho-ai` 2.5.1 SDK (needs `HONCHO_API_KEY`, or `base_url="http://localhost:8000"` when self-hosted):

```python
from honcho import Honcho

honcho = Honcho(workspace_id="my-app")
dana = honcho.peer("dana")
assistant = honcho.peer("assistant")
session = honcho.session("chat-1")

session.add_messages([
    dana.message("Keep it short please. I mostly write Go."),
    assistant.message("Got it."),
])

# Ask Honcho's reasoning about the user (answered after the deriver has run)
print(dana.chat("How should I answer Dana?", reasoning_level="low"))

# Or pull prompt-ready context capped at a token budget
context = session.context(summary=True, tokens=2_000)
```

Mem0 returns a list of facts you format yourself. Honcho can return an answer, a summary or a ready-made message list (`context.to_openai(...)`).

## Benchmarks: both self-reported

| Benchmark | Mem0 | Honcho | Hindsight |
|---|---|---|---|
| LongMemEval | 94.4 | 90.4% (LongMemEval-S); 92.6% with Gemini 3 Pro | **94.6** (LongMemEval S) |
| LoCoMo | **92.5** | 89.9% | 92 |
| BEAM 1M | 64.1 | 0.618 | **73.9** |
| BEAM 10M | 48.6 | 0.409 | **64.1** |
| BEAM 100K | Not listed | 0.630 | **75** |
| Tokens per retrieval | About 6,700 to 7,000 | Set by you (`tokens=` budget) | 24K-44K on Mem0's measurement; set by you (`max_tokens`) |

Mem0's figures come from its [README](https://github.com/mem0ai/mem0) and [research page](https://mem0.ai/research) and measure its managed Platform. Honcho's come from [honcho.dev/evals](https://honcho.dev/evals/), with gemini-2.5-flash-lite as the deriver and claude-haiku-4.5 for dreaming and chat unless stated. Hindsight's come from its [benchmark site](https://benchmarks.hindsight.vectorize.io/); Mem0's [Hindsight comparison](https://mem0.ai/compare/mem0-vs-hindsight) lists the same LongMemEval, LoCoMo and BEAM 1M/10M scores, and Hindsight's README says research collaborators at Virginia Tech and The Washington Post reproduced them. Mem0 and Hindsight report BEAM on a 0-100 scale and Honcho on 0-1. Honcho's evals page also plots its own runs of Mem0 and Zep; neither Mem0 nor Hindsight publishes a Honcho score.

Both sets are vendor-run, with different answer models and LLM judges, so a 3 to 4 point gap is not proof that one is better on your data. The BEAM 10M gap is wider: 64.1 for Hindsight against 48.6 for Mem0 and 0.409 (40.9) for Honcho. Run both on a sample of your own conversations; [LLM memory evaluation](/articles/llm-memory-evaluation/) explains how these benchmarks work and where they mislead.

## Cost and self-hosting

**Mem0 open source** costs only your LLM, embedder and vector store. The hosted Platform has a free tier (10,000 adds and 1,000 retrievals a month), Starter at $19/month and Pro at $249/month with graph memory. Mem0 lists SOC 2 and HIPAA for the Platform.

**Honcho managed** bills $2.00 per million tokens ingested (storage plus reasoning). Natural-language chat queries cost $0.001 at the `minimal` reasoning level up to $0.50 at `max`; `context()` retrieval and dreaming have no separate charge. Each organization gets a dedicated instance. Startups under $5M raised get $1,000 in credits.

**Self-hosting Honcho** means running Postgres with pgvector, Redis, the API and the deriver, and paying for the deriver's LLM calls on your own keys. The `honcho start --setup` CLI sets up all of it in Docker. Mem0's library has no server at all, which makes it the lighter of the two to run yourself.

## Where Hindsight fits

Hindsight sits closer to Honcho than to Mem0 in design: it does background reasoning over what it stores. It's the strongest of the three when:

- **Histories are long.** It posts the top BEAM scores here at every size it reports, with the widest gap at 10M tokens (64.1 vs 48.6 for Mem0 and 40.9 for Honcho).
- **You want a reasoned answer under a permissive license.** `reflect` answers questions like "what does this user want?" by reasoning over the bank, much like Honcho's chat, but under MIT instead of AGPL-3.0.
- **You want standing answers about a user.** **Mental models** are answers you define, kept up to date in the background and read without a retrieval or LLM call. **Observations** merge facts into evidence-backed beliefs, refined as new facts arrive.
- **You want everything open.** Graph links, temporal recall, consolidation and `reflect` all ship in the open-source server; Mem0 keeps graph and temporal features on its paid Platform.

Its limits for this reader: it has no per-observer peer model like Honcho's, so "what agent X knows about agent Y" means separate banks. It returns about 4x more context per retrieval than Mem0 by Mem0's numbers (set `max_tokens` lower), and writes cost more than Mem0's single pass. More in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Mem0, Honcho or Hindsight: how to choose

| If you... | Pick |
|---|---|
| Want a permissive license for a product you ship | Mem0 or Hindsight |
| Want the lightest setup: a library, no background workers | Mem0 |
| Want facts you can list, edit and delete one by one | Mem0 |
| Want the smallest token cost per retrieval | Mem0 |
| Build tutors, coaches or companions that adapt to how a person thinks | Honcho |
| Run several agents that should know different things about each other | Honcho |
| Want to ask "what does this user want?" and get a reasoned answer | Honcho, or Hindsight (`reflect`) under MIT |
| Have long histories (hundreds of thousands to millions of tokens per user) | Hindsight |
| Want graph, temporal recall and consolidation self-hosted, free | Hindsight |

Zep and Graphiti track how facts change over time, and Letta lets the agent edit its own memory blocks. See [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/), [Mem0 vs Supermemory](/articles/mem0-vs-supermemory/) and [Mem0 alternatives](/articles/mem0-alternatives-compared/) for other pairings.


## FAQ

### What is the difference between Mem0 and Honcho?

Mem0 extracts short facts from conversations and returns the relevant ones by search. Honcho stores the messages themselves, reasons over them in the background, and builds a model of each participant (a peer) that you can query in plain language. Mem0 answers 'which facts match?'; Honcho also answers 'what is this person like?'

### Are Mem0 and Honcho open source?

Both are. Mem0 is Apache 2.0, a permissive license. The Honcho server is AGPL-3.0, which requires you to share source changes if you offer a modified Honcho to users over a network. Both also run as managed services.

### Which scores higher on benchmarks, Mem0 or Honcho?

On their own published numbers, Mem0 reports 94.4 on LongMemEval and 92.5 on LoCoMo; Honcho reports 90.4% on LongMemEval-S and 89.9% on LoCoMo with its default models. Each vendor ran its own tests with different models and judges, so the gap is not a controlled comparison.

