# Memori by GibsonAI: SQL-Native Memory Storage for Agents

> Memori (GibsonAI, now Memori Labs) stores agent memory in your SQL database. How it works, Python setup, LoCoMo scores by category, and how it compares with Mem0 and Hindsight.

Source: https://aiagentmemory.org/articles/ai-agents-memory-storage-memori-gibsonai/ · Updated 2026-10-25 · aiagentmemory.org


**Memori** is an open-source memory layer for AI agents that stores memory in **ordinary SQL databases** rather than a dedicated vector store. It started in 2025 as a GibsonAI project billed as a "SQL-native memory engine," and is now developed by **Memori Labs**. You wrap your LLM client with Memori, and it records conversations, extracts facts and relationships, and injects relevant memories into later calls automatically.

This page explains where Memori came from, how it stores and recalls memory today, how to set it up in Python with your own database, and how to read its benchmark claims. Sources are the [Memori GitHub repo](https://github.com/MemoriLabs/Memori), the [Memori docs](https://memorilabs.ai/docs/) and the [Memori paper](https://arxiv.org/abs/2603.19935), checked on 25 October 2026.

## What is Memori?

**Memori is an LLM-, database- and framework-agnostic memory layer. It intercepts calls made through a registered LLM client, stores the conversation, runs background "Advanced Augmentation" to extract facts, preferences, relationships and rules, and recalls matching memories into future prompts. Memories can live in Memori Cloud or in your own SQL database.**

The core idea is that agent memory is a **data structuring problem**. The Memori paper argues that compact, structured records (semantic triples plus conversation summaries) beat bigger context windows on both accuracy and cost.

The repository is Apache 2.0, written in Python with a Rust core, and had about 17,100 GitHub stars on 25 October 2026.

## From GibsonAI to Memori Labs

The history explains why search results mix the two names.

- **2025, GibsonAI.** The repo was created in July 2025 under GibsonAI, a company building AI-managed databases. Its early README called Memori an "Open-Source Memory Engine for LLMs, AI Agents & Multi-Agent Systems," installed with `pip install memorisdk`. It offered **dual-mode memory**: "conscious" short-term memory promoted into context, and "auto" mode that searched the database per query. Storage was SQLite, PostgreSQL or MySQL.
- **2026, Memori Labs.** The project moved to the `MemoriLabs/Memori` repo (the old GibsonAI URL redirects). Memori Labs launched **Memori Cloud**, a hosted version, in March 2026, per [Open Source For You](https://www.opensourceforu.com/2026/03/open-source-memory-engine-from-memori-labs-goes-fully-hosted-with-memori-cloud/), with deployment options for cloud, BYODB, VPC and on-prem. The company's CEO is Adam B. Struck and its CTO is Michael Montero, both authors of the Memori paper.

The current package is `memori` on PyPI (3.3.6, May 2026) and `@memorilabs/memori` on npm. Code written for `memorisdk`, including `conscious_ingest=True` and `memori.enable()`, doesn't match the current API.

## How Memori stores and recalls memory

### Attribution: entity, process, session

Memori tracks memory at three levels, and it **won't create memories without attribution**:

| Level | Meaning | Example |
|---|---|---|
| **Entity** | A person, place or thing | `user_123` |
| **Process** | Your agent, program or LLM workflow | `support_agent` |
| **Session** | One run of interactions between them | Set automatically, or with `set_session()` |

### Advanced Augmentation

After each LLM call returns, Memori queues the conversation for **Advanced Augmentation** in the background, so the response isn't delayed. Per the [augmentation docs](https://github.com/MemoriLabs/Memori/blob/main/docs/memori-byodb/concepts/advanced-augmentation.mdx), it:

1. reads the user messages and AI responses;
2. identifies facts, preferences, skills, attributes and events;
3. extracts semantic triples (subject, predicate, object);
4. generates embeddings for semantic search;
5. stores the results.

It also captures **agent execution traces**, such as tool calls, decisions and outcomes, which is what the README means by "memory from what agents do, not just what they say."

One detail matters for privacy. By default (Memori 3.3.6), the Python SDK's BYODB mode still sends conversations to Memori's hosted augmentation API and writes the results to your database. The README says augmentation works without an account but is rate-limited per IP address, and an API key raises the limits. So BYODB keeps the **stored** memory in your database; it doesn't keep extraction off Memori's servers.

### Recall

When a new call goes through the wrapped client, Memori searches stored entity facts for the query, ranks candidates by vector similarity (the docs mention FAISS), and adds the matches to the prompt. Agents running under OpenClaw or Hermes also get explicit recall tools, such as `memori_recall` and `memori_recall_summary`.

### What the tables look like

In BYODB mode, Memori creates its own tables in your database, including `memori_conversation_message` (raw messages), `memori_entity_fact` (extracted facts used for recall), `memori_process_attribute` and `memori_knowledge_graph`. Because it's plain SQL, you can query, back up and audit memory with tools you already run. For the tradeoffs of SQL versus vector stores, see [vector databases for LLM memory](/articles/vector-database-for-llm-memory/).

## Setting up Memori in Python with your own database

This follows the BYODB quickstart and matches `memori` 3.3.6:

```python
import sqlite3
from memori import Memori
from openai import OpenAI

client = OpenAI()  # needs OPENAI_API_KEY

mem = Memori(conn=lambda: sqlite3.connect("memori.db")).llm.register(client)
mem.config.storage.build()  # create Memori's tables
mem.attribution(entity_id="user_123", process_id="support_agent")

client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "My favorite color is blue."}],
)
mem.augmentation.wait()  # short scripts: let background extraction finish

reply = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "What's my favorite color?"}],
)
print(reply.choices[0].message.content)  # Memori injects the stored fact
```

Swap the connection factory for PostgreSQL, MySQL or another supported engine. For Memori Cloud, drop `conn` and set `MEMORI_API_KEY`.

Supported pieces, per the README and docs:

- **LLMs:** OpenAI (Chat Completions and Responses), Anthropic, Gemini, Bedrock, DeepSeek and xAI Grok; streamed and unstreamed, sync and async.
- **Frameworks:** Agno, LangChain and Pydantic AI.
- **Databases (BYODB):** PostgreSQL, MySQL, MariaDB, SQLite, CockroachDB, TiDB, OceanBase, Oracle and MongoDB, plus managed services such as AWS RDS, Neon and Supabase.
- **Agents and tools:** an OpenClaw plugin, a Hermes Agent memory provider, and an MCP server for Claude Code, Cursor and Codex.

The OpenClaw plugin (`openclaw plugins install @memorilabs/openclaw-memori`) and the Hermes provider are set up with a **Memori API key**, so they store memory in Memori Cloud. A June 2026 change to the [BYODB docs](https://github.com/MemoriLabs/Memori/blob/main/docs/memori-byodb/index.mdx) lists "Support for Hermes Agent BYODB Implementation" and "Support for OpenClaw BYODB Implementation" as **roadmap** items, so you can't yet point either agent at your own database.

## Memori benchmark claims

Memori reports results on [LoCoMo](https://arxiv.org/abs/2402.17753), and two sources give different numbers:

- The **README** and the [results page](https://github.com/MemoriLabs/Memori/blob/main/docs/memori-cloud/benchmark/results.mdx) (updated July 2026) report **87.0% overall accuracy** at an average of **721 tokens per query**, which they call 2.8% of the full-context footprint. Memori's score is the average of three rounds, graded by an LLM judge.
- The **paper** ([arXiv 2603.19935](https://arxiv.org/abs/2603.19935), March 2026) reports **81.95% accuracy** at **1,294 tokens per query**, about 5% of full context.

The July results page breaks the score down by question type, next to the baselines Memori ran itself:

| Method (Memori's run) | Single-hop | Multi-hop | Open-domain | Temporal | Overall | Tokens added per query |
|---|---|---|---|---|---|---|
| Memori | 88.5 | 83.7 | 72.9 | 90.3 | **87.0** | **721** |
| Zep | 79.43 | 69.16 | 73.96 | 83.33 | 79.09 | 3,911 |
| LangMem | 74.47 | 61.06 | 67.71 | 86.92 | 78.05 | not given |
| Mem0 | 62.41 | 57.32 | 44.79 | 66.47 | 62.47 | 1,764 |
| Full context (ceiling) | 88.53 | 77.70 | 71.88 | 92.70 | 87.52 | 26,031 |

How to read it:

- **It's self-reported.** Memori Labs ran every row, including the competitors. Mem0's own [comparison page](https://mem0.ai/compare/mem0-vs-hindsight) lists Mem0's platform at **92.5** on LoCoMo, 30 points above the 62.47 Memori measured. Scores from different setups can't be compared directly.
- **Memori is about level with full context** (87.0 vs 87.52) while sending about 1/36 of the tokens. That's the real claim: cost, not beating full context.
- **Open-domain questions are its weakest category** (72.9), which the page says is because broad questions lack clear anchors for small extracted facts.
- **Its own numbers don't agree.** The README says Memori cuts prompt size "roughly 67% vs. Zep"; the results page says 82%, which matches its table (721 vs 3,911 tokens). The paper's 81.95% isn't explained either.

Our [LLM memory comparison](/articles/llm-memory-comparison/) lists LoCoMo claims from several vendors side by side.

## Memori Enterprise

Since August 2026 the README has a **Memori Enterprise** section, offered as a preview by email. Its one case study is an unnamed "major project management platform" whose QA agents ran in a private VPC with Memori capturing execution traces, and it claims a **$2.1M per year** cut in token spend on one workflow. That's a vendor customer story with no public data behind it. No new Python release has shipped since 3.3.6 (28 May 2026); the main branch has had only fixes and README changes since.

## When Memori fits

Memori fits teams that want agent memory in **infrastructure they already run**: an existing Postgres or MySQL, with memories they can inspect with SQL. The client-wrapping approach means almost no code changes, and attribution by entity and process maps well onto multi-user, multi-agent apps.

It fits less well if conversations must never leave your network, since default BYODB extraction still goes through Memori's API in the Python SDK, or if you want to control the extraction step yourself. Fully self-run alternatives include Mem0's Docker server, Graphiti and [Hindsight](https://github.com/vectorize-io/hindsight); see [open-source memory systems compared](/articles/open-source-memory-systems-compared/).

| | Memori | Mem0 (open source) | Hindsight |
|---|---|---|---|
| License | Apache 2.0 | Apache 2.0 | MIT |
| Where memory is stored | Memori Cloud, or your SQL database (BYODB) | 22 vector store options (Qdrant by default) | PostgreSQL + pgvector, or Oracle AI Database 23ai |
| Where extraction runs | Memori's hosted API by default, even with BYODB | Your LLM, on your side | Your server, any of 25+ LLM providers incl. local |
| LoCoMo (self-reported) | 87.0 | 92.5 (platform, Mem0's page) | 92 |
| OpenClaw / Hermes | Yes, Memori Cloud only for now | Yes | Yes, cloud or self-hosted |

Hindsight is the closest match if you liked Memori for the SQL storage but need extraction on your own machines. It stores memory in PostgreSQL with pgvector (or Oracle AI Database 23ai), runs extraction on your server with any of 25+ LLM providers, local Ollama or llama.cpp included, and ships graph, temporal recall and consolidation in its MIT release. Its vendor-reported LoCoMo score is 92. The tradeoffs: it supports fewer databases than Memori (no MySQL or MongoDB), and every write calls an LLM, so writes cost more. More in [Hindsight AI memory](/articles/ai-memory-hindsight/). For the concepts behind memory storage in agents, start with [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What is Memori by GibsonAI?

Memori is an open-source (Apache 2.0) memory layer for LLM agents that was first released in 2025 under GibsonAI, a database company, as a 'SQL-native' memory engine. It is now developed by Memori Labs. It wraps your LLM client, records conversations, extracts structured memories, and recalls them automatically on later calls.

### Where does Memori store agent memory?

Either in Memori Cloud or in your own database (BYODB). BYODB supports PostgreSQL, MySQL, MariaDB, SQLite, CockroachDB, TiDB, OceanBase, Oracle and MongoDB, plus managed services built on them. Memories land in tables such as memori_entity_fact and memori_knowledge_graph.

### Is Memori the same as memorisdk?

memorisdk was the original PyPI package name under GibsonAI, with a 'conscious' and 'auto' memory mode. The current package is memori (version 3.3.6 in May 2026) with a different API built around registering your LLM client and setting entity and process attribution.

### How does Memori score on LoCoMo?

Memori Labs reports 87.0% overall on LoCoMo with an LLM judge, using 721 tokens per query, about level with full context (87.52%) at 2.8% of the tokens. Its weakest category is open-domain (72.9%). The paper reports 81.95% at 1,294 tokens. All rows, including the Zep (79.09%) and Mem0 (62.47%) baselines, were run by Memori Labs; Mem0's own page reports 92.5 for Mem0's platform.

### Does Memori work with OpenClaw and Hermes Agent?

Yes, through an OpenClaw plugin (@memorilabs/openclaw-memori) and a Hermes memory provider (hermes-memori). Both are configured with a Memori API key, so memory lives in Memori Cloud. Running them against your own database (BYODB) is listed on Memori's roadmap, not shipped.

