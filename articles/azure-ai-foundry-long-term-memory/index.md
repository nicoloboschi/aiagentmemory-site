# Azure AI Foundry Memory: Agent Service Long-Term Memory

> How long-term memory works in Azure AI Foundry Agent Service (Microsoft Foundry): memory stores, memory types, scope, TTL, quotas, regions, pricing and Python code.

Source: https://aiagentmemory.org/articles/azure-ai-foundry-long-term-memory/ · Updated 2026-10-12 · aiagentmemory.org


**Azure AI Foundry long-term memory** is a managed feature of Foundry Agent Service that lets an agent remember users across sessions. You create a **memory store**, attach the **memory search tool** to an agent, and the service extracts facts, preferences and summaries from conversations, merges duplicates, and retrieves the relevant ones in later chats. As of October 2026 it is in **public preview**.

Microsoft has since renamed the product family to **Microsoft Foundry**, so current docs say "Memory in Foundry Agent Service." Older posts call it Azure AI Foundry memory or Azure AI Agent Service memory. They mean the same feature. Everything below comes from the [Microsoft Learn memory overview](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/what-is-memory) and the [how-to guide](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/memory-usage), checked on 8 October 2026.

## What is long-term memory in Azure AI Foundry?

**Memory in Foundry Agent Service is a managed long-term memory store for agents. An LLM extracts key information from conversations, consolidates it so duplicates merge and conflicting facts get resolved, and stores it as memory items per scope. Agents then search the store to personalize later sessions, across devices and workflows.**

Microsoft draws the same line most frameworks do. **Short-term memory** is the current session's conversation, which the orchestration layer keeps. **Long-term memory** is distilled knowledge that survives across sessions, and that's the part this feature handles. If you're new to the split, the [AI agent memory guide](/articles/ai-agent-memory-explained/) covers it in depth.

The service announced memory at Ignite 2025. Microsoft's [launch post](https://devblogs.microsoft.com/foundry/introducing-memory-in-foundry-agent-service/) (25 November 2025) says retrieval uses hybrid search and that bring-your-own storage is planned but not yet available.

## How Foundry Agent Service memory works

Memory runs in three phases, per the Learn overview:

1. **Extraction.** As a user talks to the agent, the system pulls out preferences, facts and context, such as "allergic to dairy," plus summaries of recent activity.
2. **Consolidation.** An LLM merges similar or duplicate topics so the store doesn't fill with repeats. Conflicting facts, like a new allergy, get resolved.
3. **Retrieval.** When the agent needs context, it searches the store for the most relevant memories.

Writes aren't instant. After each agent response the service schedules a memory update, but it waits for a period of inactivity set by `update_delay`. The default is 300 seconds (five minutes). That debounce means a fact a user states now may not show up in a new chat for a few minutes.

### The three memory types

| Memory type | What it holds | When to retrieve it |
|---|---|---|
| **User profile memory** | Durable preferences and personal context: language, product defaults, accessibility needs | Near the start of each conversation |
| **Chat summary memory** | Distilled summaries of earlier topics and threads | Each turn, using the current messages |
| **Procedural memory** | Reusable routines and patterns inferred from past interactions | When the user asks for a recurring task |

All three are enabled by default. You can turn each off when you create the store, and you can steer extraction with a `user_profile_details` instruction such as "flight carrier preference and dietary restrictions," or tell it what to avoid, such as financials or precise location.

### Static and contextual retrieval

The docs make a useful point: profile facts often aren't semantically similar to the user's message, so similarity search alone can miss them. The fix is two calls. A search with a `scope` and no messages returns **static** memories (the user profile), which you inject at the start of a conversation. A search with the latest messages returns **contextual** memories, both profile and chat summary, for each turn. The memory search tool does both for you.

## Scope: how memory is partitioned per user

Each **scope** inside a memory store is an isolated set of memory items. For a support agent, each customer gets their own scope.

- **With the memory search tool**, set `scope` to `{{$userId}}`. The service resolves the user from an `x-memory-user-id` request header if you send one, or falls back to the caller's Microsoft Entra tenant and object IDs.
- **With the low-level Memory Store API**, you must pass `scope` explicitly on every call. Automatic identity resolution isn't supported there.

Microsoft's best-practice list warns against giving agents memories shared across all users. If you do share a scope, use `user_profile_details` to stop personal data from being stored.

## Using memory from Python

You need the `azure-ai-projects` package (version 2.3.0 or later in the current docs), a Foundry project, a chat model deployment and an embedding deployment. This sketch follows the Learn how-to: create a store, attach the memory search tool, then search the store directly.

```python
import os
from datetime import timedelta
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (
    MemoryStoreDefaultDefinition,
    MemoryStoreDefaultOptions,
    MemorySearchPreviewTool,
    MemorySearchOptions,
    PromptAgentDefinition,
)

project = AIProjectClient(
    endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"],
    credential=DefaultAzureCredential(),
)

# 1. Create a memory store (options can only be set at creation time)
store = project.beta.memory_stores.create(
    name="support_memory",
    definition=MemoryStoreDefaultDefinition(
        chat_model=os.environ["MEMORY_STORE_CHAT_MODEL_DEPLOYMENT_NAME"],
        embedding_model=os.environ["MEMORY_STORE_EMBEDDING_MODEL_DEPLOYMENT_NAME"],
        options=MemoryStoreDefaultOptions(
            user_profile_enabled=True,
            chat_summary_enabled=True,
            procedural_memory_enabled=True,
            default_ttl_seconds=timedelta(days=30),
            user_profile_details="Avoid financials, precise location and credentials",
        ),
    ),
)

# 2. Attach the memory search tool to a prompt agent
agent = project.agents.create_version(
    agent_name="SupportAgent",
    definition=PromptAgentDefinition(
        model=os.environ["MEMORY_STORE_CHAT_MODEL_DEPLOYMENT_NAME"],
        instructions="You are a helpful support assistant.",
        tools=[MemorySearchPreviewTool(
            memory_store_name=store.name,
            scope="user_123",
            update_delay=300,  # seconds of inactivity before memories are written
        )],
    ),
)

# 3. Or query the store yourself
results = project.beta.memory_stores.search_memories(
    name=store.name,
    scope="user_123",
    items=[{"role": "user", "content": "What do I usually order?", "type": "message"}],
    options=MemorySearchOptions(max_memories=5),
)
for m in results.memories:
    print(m.memory_item.content)
```

To write memories without an agent, call `begin_update_memories(name=..., scope=..., items=[...], update_delay=0)`, which returns a poller. Microsoft says this long-running operation can take about a minute. The API also offers item-level create, read, update, list and delete, plus deleting all memories for one scope, which is how you'd handle a user's data deletion request.

When a user says "remember that my seat is aisle" or "forget my seat," the memory search tool applies it at once and returns a `memory_command_call` item in the response. These direct commands still respect the store's TTL.

## Limits, regions and pricing

**Quotas (Learn, October 2026):**

- 100 scopes per memory store
- 10,000 memories per scope
- 1,000 search requests per minute
- 1,000 update requests per minute

**Other limits:** memory needs compatible Azure OpenAI chat and embedding deployments. Virtual network integration isn't supported for memory stores. Procedural memory and default TTL are set at creation; changing them later may need a new store, depending on API version. A TTL of `0` means memories never expire.

**Regions:** 19 at the time of writing, including East US 2, West US, West US 2, West US 3, North Central US, Canada East, Brazil South, UK South, France Central, Sweden Central, Switzerland North, Norway East, Italy North, Japan East, Korea Central, Australia East, South India, South Africa North and UAE North.

**Pricing:** Microsoft says memory pricing "can change during preview" and that you're billed for the chat and embedding model usage the store consumes. There's no separate memory meter listed. Check the [Foundry Agent Service pricing page](https://azure.microsoft.com/pricing/details/foundry-agent-service/) before you plan costs.

## Security and when to use something else

Memory written by an LLM can be poisoned. Microsoft lists **prompt injection** and **memory corruption** as the main risks and recommends Azure AI Content Safety with prompt-injection detection, plus regular adversarial testing. In practice that means treating anything a user says as untrusted before it becomes a stored fact.

Microsoft also says where memory doesn't fit. Use a Foundry IQ knowledge base for curated company content, and the file search tool for documents a user uploads during a chat. Memory is for things learned about a user or task over time.

How it compares with the other managed options:

| | Foundry Agent Service memory | Vertex AI / Agent Platform Memory Bank | AWS AgentCore Memory |
|---|---|---|---|
| Status (Oct 2026) | Public preview | Billed service; event ingestion and profiles GA | Generally available |
| Memory types | User profile, chat summary, procedural | Natural-language facts, custom topics, structured profiles | Semantic, user preference, summary, episodic strategies |
| Partition key | `scope` | `scope` dictionary | Namespaces built from actor and session IDs |
| Expiry | Store-level default TTL | TTL config | Event expiry 3-365 days |

See the deep dives on [Vertex AI Agent Engine Memory Bank](/articles/vertex-ai-agent-engine-memory-bank/) and [AgentCore long-term memory](/articles/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/) for the details.

If you build with Microsoft Agent Framework rather than prompt agents, Foundry memory is one of its listed context provider integrations, next to Mem0, Redis, Neo4j and Cosmos DB. The [Microsoft Agent Framework memory](/articles/microsoft-agent-framework-memory/) page covers that route. If you need memory outside Azure, or self-hosted, open-source systems such as Mem0, Graphiti, Cognee or [Hindsight](https://github.com/vectorize-io/hindsight) run anywhere; the [open-source memory systems comparison](/articles/open-source-memory-systems-compared/) lists them.

Hindsight follows the same extract-and-consolidate design as Foundry memory, without the preview's network limit. It's MIT-licensed with the full server included and runs on PostgreSQL from Docker, pip or Helm, so you can deploy it inside your own virtual network. Like Foundry, it extracts and consolidates memories, but it also keeps the evidence behind each consolidated belief and offers `reflect` for answers that need reasoning. It works with 25+ LLM providers (Azure OpenAI through its LiteLLM gateway option) and has a Microsoft Agent Framework integration. On the long-history BEAM test it scores 73.9 at 1M tokens versus Mem0's 64.1, per [Mem0's own comparison](https://mem0.ai/compare/mem0-vs-hindsight). The cost: you run it yourself (or use usage-based Hindsight Cloud), and each write makes LLM calls, as Foundry's does. See [what Hindsight is](/articles/ai-memory-hindsight/).


## FAQ

### Does Azure AI Foundry Agent Service have long-term memory?

Yes. Memory in Foundry Agent Service is a managed long-term memory feature, in public preview as of October 2026. You create a memory store, attach the memory search tool to an agent (or call the Memory Store API), and the service extracts, consolidates and retrieves memories per scope, usually one scope per user.

### How much does Foundry Agent Service memory cost?

Microsoft's docs say that during the preview you're billed for the chat and embedding model deployments the memory store uses, and that memory pricing can change before general availability. There is no separate per-memory price listed as of October 2026.

### What are the limits of Foundry memory stores?

Per Microsoft Learn, a memory store holds up to 100 scopes, each scope up to 10,000 memories, and search and update calls are each limited to 1,000 requests per minute. Memory needs Azure OpenAI chat and embedding deployments, and virtual network integration isn't supported for memory stores.

