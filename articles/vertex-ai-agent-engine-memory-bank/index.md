# Vertex AI Agent Engine Memory Bank: Guide and Pricing

> Vertex AI Agent Engine Memory Bank, now Agent Platform Memory Bank: how memory generation, scopes, topics, profiles and revisions work, ADK code, and 2026 pricing.

Source: https://aiagentmemory.org/articles/vertex-ai-agent-engine-memory-bank/ · Updated 2026-10-12 · aiagentmemory.org


**Vertex AI Agent Engine Memory Bank** is Google Cloud's managed long-term memory for AI agents. It uses Gemini models to pull facts and preferences out of conversations, merges them with what it already knows about that user, and returns them later by scope or similarity search. In April 2026 Google renamed it **Agent Platform Memory Bank**, part of the **Gemini Enterprise Agent Platform**.

Old links and code still use the Vertex name; the ADK class is still `VertexAiMemoryBankService` and takes an `agent_engine_id`. This page uses Google's current docs, [release notes](https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes) and pricing page, checked on 8 October 2026.

## What is Memory Bank?

**Memory Bank is a managed service that turns conversations between a user and an agent into long-term memories. An LLM extracts the meaningful information, consolidates it with existing memories in the same scope (adding, updating or deleting facts), and stores each memory as a self-contained fact you can fetch in full or by similarity.**

Google [announced the public preview](https://cloud.google.com/blog/products/ai-machine-learning/vertex-ai-memory-bank-in-public-preview) on 9 July 2025. The launch post says the approach is grounded in Google Research work accepted at ACL 2025, the paper [In Prospect and Retrospect: Reflective Memory Management for Long-term Personalized Dialogue Agents](https://arxiv.org/abs/2503.08026) (Tan et al.). That paper stores summaries at utterance, turn and session level and refines retrieval with online reinforcement learning, reporting more than a 10% accuracy gain on LongMemEval over a baseline without memory management.

### The 2026 rename

The 22 April 2026 release notes moved everything under one brand:

| Old name | New name |
|---|---|
| Vertex AI Agent Engine | Agent Runtime |
| Vertex AI Agent Engine Memory Bank | Agent Platform Memory Bank |
| Agent Builder Sessions | Agent Platform Sessions |

Current code samples use `agentplatform.Client` from the `google-cloud-agentplatform` package. Older tutorials use the `vertexai` SDK and `agent_engines` calls.

## How Memory Bank generates memories

Memory generation has two steps, per the [Memory Bank overview](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/memory-bank):

1. **Extraction.** A Gemini model reads the source data and keeps only what matters for the configured **memory topics**.
2. **Consolidation.** New facts are compared with existing memories in the same scope. Each result is labeled `CREATED` (a new concept), `UPDATED` (same concept, fact may change) or `DELETED` (contradicted by new information).

You can feed it three kinds of input: raw conversation events (`direct_contents_source`), a stored Sessions conversation (`vertex_session_source`), or up to five pre-extracted facts (`direct_memories_source`) that skip extraction and go straight to consolidation. Generation can block or run in the background with `wait_for_completion=False`, which Google recommends for production. Since July 2026 the **IngestEvents** API is generally available: you stream events in and Memory Bank triggers generation based on batching rules you set.

On 29 June 2026 the default generation model moved from Gemini 2.5 Flash to Gemini 3.5 Flash.

### Memory topics

Topics define what counts as worth remembering. There are four managed topics, all on by default:

- `USER_PERSONAL_INFO`: names, relationships, hobbies, key dates
- `USER_PREFERENCES`: stated or implied likes, dislikes and styles
- `KEY_CONVERSATION_DETAILS`: milestones and outcomes in a conversation
- `EXPLICIT_INSTRUCTIONS`: things the user asks the agent to remember or forget

You can add **custom topics** with a label and a description that goes into the extraction prompt, plus few-shot examples.

### Profiles, revisions and TTL

- **Memory profiles** (GA since 15 July 2026) are structured records that follow a schema you define, for example from a Pydantic model. Each field holds one value that an LLM merges when new data arrives, so the agent can read a user's profile without running a search.
- **Revisions** keep an immutable history of how each memory changed. They count toward billed storage.
- **TTL** settings delete stale memories automatically. Without one, memories don't expire.
- **IAM Conditions** can restrict which principals read or write memories for a given scope.

## Scope and retrieval

Every memory has a **scope**, a dictionary such as `{"user_id": "123"}`. Matching is exact: a request only sees memories whose scope is identical, regardless of key order. Scope is set when a memory is created and can't be changed. ADK uses `{"user_id": ..., "app_name": ...}`, so memories are isolated per user and per app.

Two retrieval modes exist:

- **Retrieve all** memories for a scope, up to 100 per page.
- **Similarity search** with a query, ranked by Euclidean distance between embeddings. `top_k` defaults to 3. Since July 2026 you can configure `gemini-embedding-2` for this.

```python
import agentplatform

client = agentplatform.Client(project="PROJECT_ID", location="us-central1")
memory_bank = client.memory_banks.create()  # default configuration

scope = {"user_id": "123"}

# Add a pre-extracted fact; Memory Bank consolidates it with existing memories
client.memory_banks.memories.generate(
    name=memory_bank.name,
    direct_memories_source={"direct_memories": [{"fact": "Prefers aisle seats"}]},
    scope=scope,
)

# Fetch the memories most relevant to the current turn
results = client.memory_banks.memories.retrieve(
    name=memory_bank.name,
    scope=scope,
    similarity_search_params={"search_query": "book me a flight", "top_k": 3},
)
for memory in results.page:
    print(memory)
```

## Using Memory Bank with Google ADK

The [ADK quickstart](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/memory-bank/adk-quickstart) wires Memory Bank in through three pieces: a memory service, a retrieval tool and a callback that sends events for generation.

```python
from google import adk
from google.adk.memory import VertexAiMemoryBankService
from google.adk.sessions import VertexAiSessionService
from google.adk.tools.preload_memory_tool import PreloadMemoryTool

async def generate_memories(callback_context):
    # Send recent events to Memory Bank for incremental generation
    await callback_context.add_events_to_memory(
        events=callback_context.session.events[-5:-1])
    return None

agent = adk.Agent(
    model="gemini-3.5-flash",
    name="stateful_agent",
    instruction="Use what you remember about the user.",
    tools=[PreloadMemoryTool()],  # retrieves memories at the start of every turn
    after_agent_callback=generate_memories,
)

runner = adk.Runner(
    agent=agent,
    app_name="travel_app",
    session_service=VertexAiSessionService(
        project="PROJECT_ID", location="us-central1", agent_engine_id="SESSIONS_ID"),
    memory_service=VertexAiMemoryBankService(
        project="PROJECT_ID", location="us-central1", agent_engine_id="MEMORY_BANK_ID"),
)
```

`PreloadMemoryTool` retrieves on every turn. `LoadMemoryTool` lets the model decide when to look. Google recommends `add_events_to_memory` for incremental updates and `add_session_to_memory` only at the end of a session, so events aren't processed twice. When you deploy to Agent Runtime, an empty Memory Bank instance comes with the runtime.

### Google's agent memory options beyond Memory Bank

People searching for a "Google agent memory framework" usually mean ADK's memory layer. ADK separates **session state** from **memory**. State keys without a prefix live in one session; `user:` keys are shared across a user's sessions in the same app; `app:` keys are shared by every user; `temp:` keys vanish after one invocation. Memory services implement `BaseMemoryService`:

| ADK memory service | Storage | Use |
|---|---|---|
| `InMemoryMemoryService` | Process memory, keyword matching | Prototyping only |
| `VertexAiMemoryBankService` | Memory Bank, LLM-extracted facts | Production user memory |
| `VertexAiRagMemoryService` | RAG corpus, vector search | Searching past session content |

See [ADK's memory docs](https://adk.dev/sessions/memory/). Google Research's separate Titans architecture is a model-level idea, not a product; the [Titans explainer](/articles/google-titans-give-ai-human-like-memory/) covers it.

## Memory Bank pricing (October 2026)

Google consolidated Agent Platform billing in 2026. From the [pricing page](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing), Memory Bank billing on the new structure began **1 September 2026**:

| What you pay for | Rate |
|---|---|
| Storage (all memory data, **including revisions**) | $0.30 per GiB-month (Agent Storage) |
| Read API requests (search, fetch) | $0.085 per vCPU-hour, 1 vCPU-hour per 3 million reads |
| Write API requests (generate, update, delete expired) | $0.085 per vCPU-hour, 1 vCPU-hour per 1 million writes |
| Generation and embedding tokens | Billed separately at the selected model's rates |

So a million memory writes costs about $0.085 in compute, and three million reads about the same. In practice the Gemini tokens used for extraction and consolidation will often be the larger line item, so check the model price for your generation config. Sessions are billed the same way.

Under the old SKUs, Google's [SKU transition notice](https://services.google.com/fh/files/emails/b_502770571_gemini_enterprise_agent_platform_pricing.pdf) lists Memory Bank at $0.25 per 1,000 memories stored per month and $0.50 per 1,000 memories retrieved, with embedding usage not billed. Guides quoting those figures predate September 2026.

## When Memory Bank fits, and alternatives

Memory Bank fits teams already on Google Cloud that want user memory with enterprise controls. Its docs list VPC Service Controls, CMEK, data residency at rest, HIPAA, Access Transparency and Access Approval support. Google also warns about **memory poisoning** and recommends Model Armor and adversarial testing.

It's less of a fit if you need to run outside Google Cloud or want to inspect the memory engine's code. The other clouds have direct equivalents: [Azure AI Foundry long-term memory](/articles/azure-ai-foundry-long-term-memory/) and [AWS AgentCore Memory](/articles/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/). Open-source memory servers such as Mem0, Graphiti, Cognee and Hindsight run anywhere and work with ADK; the [LLM memory comparison](/articles/llm-memory-comparison/) puts them side by side.

**[Hindsight](https://github.com/vectorize-io/hindsight)** is the closest open-source match for an ADK team. It has a native [Google ADK integration](https://hindsight.vectorize.io/sdks/integrations/google-adk), can use Gemini (including through Vertex AI) as its extraction model, and is MIT-licensed with every feature in the open-source server. Like Memory Bank it extracts facts and consolidates them, but it also searches by keyword, entity graph and time range next to similarity, and `reflect` reasons over memories to answer. It's strongest on long histories: 73.9 on BEAM 1M and 64.1 on BEAM 10M (vendor-reported). The trade-off: you run it yourself (or use Hindsight Cloud) instead of getting Google Cloud's built-in controls such as VPC Service Controls and CMEK, and every write calls an LLM, just as Memory Bank's generation does. For the concepts behind all of these, start with [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What is Vertex AI Agent Engine Memory Bank?

It's Google Cloud's managed long-term memory service for agents. It uses Gemini to extract facts from conversations, consolidate them with existing memories for the same user, and return them by scope or similarity search. Since April 2026 it is called Agent Platform Memory Bank, part of the Gemini Enterprise Agent Platform.

### How much does Memory Bank cost?

From 1 September 2026, Google bills storage at $0.30 per GiB-month (including revisions), and operations as Agent Compute at $0.085 per vCPU-hour, where one vCPU-hour covers 3 million reads or 1 million writes. Model tokens for memory generation and embeddings are billed separately at the model's rates. Before that, the old SKUs were $0.25 per 1,000 memories stored per month and $0.50 per 1,000 retrieved.

### Can I use Memory Bank without ADK?

Yes. ADK has a built-in VertexAiMemoryBankService, but Google's docs say other frameworks such as LangGraph and CrewAI can call the Memory Bank API directly by wrapping it in tools and callbacks, and memories can be accessed from Agent Runtime, local code or other deployments.

