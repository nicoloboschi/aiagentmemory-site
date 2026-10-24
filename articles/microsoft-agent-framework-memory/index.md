# Microsoft Agent Framework Memory: Sessions and Providers

> How memory works in Microsoft Agent Framework (Python): AgentSession, history and context providers, and Mem0, Foundry, Cosmos DB, Redis, Hindsight compared.

Source: https://aiagentmemory.org/articles/microsoft-agent-framework-memory/ · Updated 2026-10-24 · aiagentmemory.org


**Microsoft Agent Framework memory** is built from two pieces. An **`AgentSession`** holds one conversation's state across `agent.run` calls. **Context providers** run before and after each call: history providers reload and save the transcript, and memory providers extract and recall lasting facts through Mem0, Redis, Cosmos DB, Neo4j or Microsoft Foundry.

Agent Framework is the [successor to Semantic Kernel and AutoGen](https://learn.microsoft.com/en-us/agent-framework/overview/), built by the same teams. The Python package hit 1.0 in April 2026 and is at 1.20 as of October 2026. Code below follows the [Agent Framework memory tutorial](https://learn.microsoft.com/en-us/agent-framework/get-started/memory) and the package's current signatures.

## What is memory in Microsoft Agent Framework?

**Memory in Microsoft Agent Framework is the context an agent carries between runs. An AgentSession groups a conversation and holds shared state. Context providers attached to the agent load that context before each model call and save what happened after it, either as a full transcript (history) or as selected durable facts (memory).**

The docs split providers into patterns:

| Pattern | What it does | Examples |
|---|---|---|
| Conversation storage | Reloads and saves the exact transcript | `InMemoryHistoryProvider`, Redis, Cosmos DB |
| Memory | Extracts and recalls selected durable facts | Mem0, Foundry, Redis, Neo4j, Cosmos DB |
| RAG | Retrieves from an external knowledge source | Azure AI Search, Foundry, Neo4j, Redis |

This split mirrors the general difference between [short-term and long-term memory](/articles/short-term-and-long-term-memory-agentic-ai/): one keeps the conversation going, the other remembers the user.

## Sessions: multi-turn conversations

Pass the same session to every run and the agent sees earlier turns:

```python
import asyncio

from agent_framework import Agent, AgentSession
from agent_framework.openai import OpenAIChatClient


async def main():
    agent = Agent(
        client=OpenAIChatClient(model="gpt-5.4-mini"),  # reads OPENAI_API_KEY
        instructions="You are a friendly assistant. Keep answers brief.",
    )
    session = agent.create_session()

    await agent.run("My name is Alice and I love hiking.", session=session)
    print(await agent.run("What do you remember about me?", session=session))

    # Save and restore across processes
    saved = session.to_dict()  # JSON-serializable dict
    restored = AgentSession.from_dict(saved)
    print(await agent.run("And my name?", session=restored))


asyncio.run(main())
```

Where the transcript lives depends on the model service. Services that keep conversation state, such as OpenAI Responses or Foundry, store it server-side and the session holds a `service_session_id`. Otherwise a local history provider stores it. The Python docs note that the agent may add an `InMemoryHistoryProvider` automatically in some cases, "but this is not guaranteed," so add one explicitly if you rely on it.

The [session docs](https://learn.microsoft.com/en-us/agent-framework/concepts/agents/conversations/session) flag a security point. Service-side IDs like `resp_*` or `conv_*` are scoped to your API key or project, not to your end users. In a multi-user app, map client session IDs to service IDs in your own storage and check ownership before resuming.

## History providers: persisting the transcript

`InMemoryHistoryProvider` keeps the transcript in process memory. For persistence, swap in a database-backed provider, such as Redis from the `agent-framework-redis` package:

```python
from agent_framework import Agent
from agent_framework.openai import OpenAIChatClient
from agent_framework.redis import RedisHistoryProvider

history = RedisHistoryProvider(
    source_id="chat_history",
    redis_url="redis://localhost:6379",
    application_id="support-bot",
    agent_id="support-bot",
    max_messages=50,  # keep the last 50 messages
)

agent = Agent(
    client=OpenAIChatClient(),
    instructions="You are a helpful support assistant.",
    context_providers=[history],
)
```

`InMemoryHistoryProvider` accepts flags that control what it saves: `load_messages`, `store_inputs`, `store_outputs` and `store_context_messages`. The docs warn that only **one** history provider should have `load_messages=True`, or you'll replay several transcripts into the same call. A common pattern adds a second provider with `load_messages=False, store_context_messages=True` as an audit log of everything the model saw.

## Context providers: custom and long-term memory

A **context provider** subclasses `ContextProvider` and implements `before_run` and `after_run`. Before the call it can add instructions, messages or tools. After the call it can save what it learned. Per-session state goes in the `state` dict, which serializes with the session:

```python
from typing import Any

from agent_framework import AgentSession, ContextProvider, SessionContext


class UserNameMemory(ContextProvider):
    def __init__(self) -> None:
        super().__init__("user_name_memory")

    async def before_run(self, *, agent: Any, session: AgentSession,
                         context: SessionContext, state: dict[str, Any]) -> None:
        name = state.get("user_name")
        context.extend_instructions(
            self.source_id,
            f"The user's name is {name}." if name else "Ask for the user's name.",
        )

    async def after_run(self, *, agent: Any, session: AgentSession,
                        context: SessionContext, state: dict[str, Any]) -> None:
        for message in context.input_messages:
            _, found, rest = (message.text or "").lower().partition("my name is")
            if found and rest.strip():
                state["user_name"] = rest.split()[0].capitalize()
```

That example only lasts as long as the session. For memory across sessions, use a provider backed by a memory service:

- **Mem0** (`pip install agent-framework-mem0`, then `from agent_framework.mem0 import Mem0ContextProvider`). Set storage scopes (`user_id`, `agent_id`, `application_id`) and retrieval scopes (`search_user_id`, and so on) separately. The docs warn that if no `search_*` scope is set, it stores memories but never recalls them.
- **Microsoft Foundry** (`FoundryMemoryProvider` in `agent_framework.foundry`): managed semantic memory in a Foundry memory store, scoped per user. Our page on [Azure AI Foundry long-term memory](/articles/azure-ai-foundry-long-term-memory/) covers the service side.
- **Azure Cosmos DB memory** (`pip install agent-framework-azure-cosmos-memory --pre`, then `from agent_framework_azure_cosmos_memory import CosmosMemoryContextProvider`). Built on the [Azure Cosmos DB Agent Memory Toolkit](https://github.com/AzureCosmosDB/AgentMemoryToolkit), it extracts facts, procedural knowledge and episodic memories from each turn, consolidates a cross-thread user profile, and deduplicates and resolves contradictions, per the [package README](https://github.com/microsoft/agent-framework/tree/main/python/packages/azure-cosmos-memory). One Foundry project endpoint powers both the memory extraction and the agent.
- **Redis** (`RedisContextProvider`, backed by a RedisVL search index) and **Neo4j** providers, listed on the [context provider integrations page](https://learn.microsoft.com/en-us/agent-framework/integrations/by-component/context-providers/).

| Provider | Install | Where memory lives | What it extracts or recalls |
|---|---|---|---|
| Mem0 | `agent-framework-mem0` | Mem0 Platform or self-hosted Mem0 | Facts, with separate storage and search scopes |
| Microsoft Foundry | `agent-framework-foundry` (`FoundryMemoryProvider`) | Managed Foundry memory store (preview) | User profile, chat summaries, procedures |
| Azure Cosmos DB memory | `agent-framework-azure-cosmos-memory` (pre-release) | Your Cosmos DB database | Facts, procedures, episodes, consolidated user profile |
| Redis | `agent-framework-redis` | Redis with a RedisVL index | Semantic search over stored context |
| Hindsight | `hindsight-agent-framework` | Hindsight server (self-hosted, MIT) or Hindsight Cloud | Facts, entities, consolidated observations; `reflect` |

The October 2026 releases mostly hardened these providers rather than adding new ones: Python 1.23.0 (22 October) rejects out-of-range Cosmos memory search parameters and closes the Redis search-index connection when the provider owns it, and .NET 1.26.0 (23 October) preserves caller cancellation in memory providers, per the [Agent Framework release notes](https://github.com/microsoft/agent-framework/releases).

Third-party memory systems also ship providers. [Hindsight](https://github.com/vectorize-io/hindsight) publishes `hindsight-agent-framework`, whose `HindsightProvider` recalls memories in `before_run` and retains the exchange in `after_run`, so the model doesn't have to remember to call a memory tool:

```python
# pip install hindsight-agent-framework
from agent_framework import Agent
from agent_framework.openai import OpenAIChatClient
from hindsight_agent_framework import HindsightProvider

agent = Agent(
    client=OpenAIChatClient(),
    instructions="You are a helpful assistant.",
    context_providers=[
        # one bank per user; omit hindsight_api_url to use Hindsight Cloud (HINDSIGHT_API_KEY)
        HindsightProvider(bank_id="user-123", hindsight_api_url="http://localhost:8888"),
    ],
)
```

Compared with the Mem0 provider, Hindsight's open-source server includes graph and temporal recall and background consolidation of facts into observations, features Mem0 keeps on its paid Platform. It also leads Mem0 on the long-history BEAM benchmark (64.1 vs 48.6 at 10M tokens, per [Mem0's own comparison](https://mem0.ai/compare/mem0-vs-hindsight); vendor-run). The tradeoff: it injects more context per run than Mem0 by default, so set `max_tokens` on the provider if prompt size matters. See [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Harness Agent memory

Agent Framework also has a **Harness Agent** (`create_harness_agent` in Python): an opinionated agent for long, multi-step tasks with planning, context compaction, file access and memory built in. It uses the same `AgentSession` lifecycle. Per the docs, the harness saves history after each model call inside a tool loop, not just at the end of a run. Reuse and serialize the session so todos, file memory and tool approvals survive a restart.

## Coming from Semantic Kernel

Microsoft's [Semantic Kernel migration guide](https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-semantic-kernel/) maps the conversation pieces directly. Semantic Kernel's `ChatHistoryAgentThread` becomes `agent.create_session()`, and `agent.get_session(service_session_id=...)` continues a service-managed conversation. There's no session delete API, since not every provider supports deleting hosted history, so track and delete server-side history through the provider's own SDK.

Existing Semantic Kernel vector stores aren't stranded either. With `semantic-kernel` 1.38 or later, `collection.create_search_function(...)` returns a kernel function, and `.as_agent_framework_tool()` turns it into an Agent Framework tool. Support differs by language: Mem0 is Python-only, and the Go SDK, in public preview, has no Harness Agent and none of the listed providers yet.

For how Agent Framework's approach compares with LangGraph, CrewAI and the OpenAI Agents SDK, see the [AI agent framework comparison](/articles/ai-agent-framework-comparison/) and the [AI agent memory guide](/articles/ai-agent-memory-explained/).


## FAQ

### How does memory work in Microsoft Agent Framework?

Each conversation is an AgentSession that you pass to agent.run. Context providers run before and after every call: history providers reload and save the transcript, and memory providers such as Mem0, Microsoft Foundry, Azure Cosmos DB or Redis extract and recall durable facts. Sessions serialize with to_dict and from_dict.

### Where does Microsoft Agent Framework store chat history?

Either in the model service, when it keeps server-side state such as OpenAI Responses or Foundry, or in a local history provider. InMemoryHistoryProvider is the built-in local one; RedisHistoryProvider and Cosmos DB providers persist history outside the process.

### Is Microsoft Agent Framework the replacement for Semantic Kernel and AutoGen?

Yes. Microsoft's overview calls Agent Framework the direct successor to Semantic Kernel and AutoGen, built by the same teams, and provides migration guides from both. Version 1.0 of the Python package shipped in April 2026.

