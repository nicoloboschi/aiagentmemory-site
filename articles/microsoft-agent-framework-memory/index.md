# Microsoft Agent Framework Memory: Sessions and Providers

> How memory works in Microsoft Agent Framework (Python 1.25): AgentSession, history and context providers, the Harness Agent's MEMORY.md memory with extraction and consolidation, and Mem0, Foundry, Cosmos DB, Redis, Hindsight compared.

Source: https://aiagentmemory.org/articles/microsoft-agent-framework-memory/ · Updated 2026-11-09 · aiagentmemory.org


**Microsoft Agent Framework memory** is built from two pieces. An **`AgentSession`** holds one conversation's state across `agent.run` calls. **Context providers** run before and after each call: history providers reload and save the transcript, and memory providers extract and recall lasting facts through Mem0, Redis, Cosmos DB, Neo4j or Microsoft Foundry.

Agent Framework is the [successor to Semantic Kernel and AutoGen](https://learn.microsoft.com/en-us/agent-framework/overview/), built by the same teams. The Python package hit 1.0 in April 2026 and is at 1.25.0 as of 7 November 2026; the [October and November releases](#what-changed-in-the-october-and-november-2026-releases) changed the Redis memory index in a way that needs a re-ingest, dropped Python 3.10 and fixed two bookkeeping bugs in the harness's file memory. Code below follows the [Agent Framework memory tutorial](https://learn.microsoft.com/en-us/agent-framework/get-started/memory) and the package's current signatures.

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

### What changed in the October and November 2026 releases

The October and November 2026 releases mostly hardened these providers rather than adding new ones. One of them breaks existing Redis memory, and the 7 November release closes the door on Python 3.10, per the [Agent Framework release notes](https://github.com/microsoft/agent-framework/releases):

| Release | Date | Memory change | What you need to do |
|---|---|---|---|
| Python 1.23.0 | 22 Oct | Rejects out-of-range Cosmos memory search parameters; closes the Redis search-index connection when the provider owns it | Nothing |
| .NET 1.26.0 | 23 Oct | Memory providers keep the caller's cancellation | Nothing |
| .NET 1.27.0 | 30 Oct | Mem0 provider checks scopes on sessions restored from saved state ([#9827](https://github.com/microsoft/agent-framework/pull/9827)) | Upgrade if you restore sessions and use Mem0 |
| Python 1.24.0 | 30 Oct | **Breaking:** `RedisContextProvider` uses a new index schema for user, agent, application and session scopes ([#9461](https://github.com/microsoft/agent-framework/pull/9461)) | Delete the old index and its hashes, then re-ingest |
| Python 1.24.0 | 30 Oct | Experimental `AgentTriggeredCompactionStrategy` and `UnloadSkillsCompactionStrategy` ([#9384](https://github.com/microsoft/agent-framework/pull/9384)) | Optional |
| Python 1.24.0 | 30 Oct | Python 3.11 is the supported baseline; the next release blocks 3.10 installs | Move off Python 3.10 |
| .NET 1.28.0 | 6 Nov | `Mem0Client.ClearMemoryAsync` throws `ArgumentException` when all four scope IDs are blank instead of sending an unscoped DELETE to Mem0 ([#10109](https://github.com/microsoft/agent-framework/pull/10109)) | Upgrade if you clear Mem0 memories from .NET |
| Python 1.25.0 | 7 Nov | **Breaking:** Python 3.10 installs are refused across every `agent-framework-*` package ([#9948](https://github.com/microsoft/agent-framework/pull/9948)) | Be on 3.11 or later before `pip install -U` |
| Python 1.25.0 | 7 Nov | Harness memory: a session whose extraction was still running when a consolidation pass started is no longer dropped from the pending count ([#9294](https://github.com/microsoft/agent-framework/pull/9294)) | Nothing; consolidation now triggers on time |
| Python 1.25.0 | 7 Nov | Harness memory: a corrupt `state.json` is moved aside to `.state.json.<uuid>.corrupt` and the store restarts clean, instead of raising on every turn ([#9547](https://github.com/microsoft/agent-framework/pull/9547)) | Nothing; check for `.corrupt` files if memory bookkeeping ever reset |
| Python 1.25.0 | 7 Nov | Background agents keep finished tasks until cleared, so a session snapshot restored mid-task no longer marks the result `LOST` ([#9387](https://github.com/microsoft/agent-framework/pull/9387)) | Nothing |
| Python 1.25.0 | 7 Nov | Redis integration installs on Python 3.15; RedisVL range widened to include 0.31 ([#10143](https://github.com/microsoft/agent-framework/pull/10143)) | Optional |

**The Redis change.** Before 1.24.0, the Redis provider matched scopes through Redis TAG defaults that don't keep every ID exactly. Per PR #9461, distinct application IDs (for example, ones differing only in case or punctuation) could be treated as the same one, which breaks the isolation the scopes are meant to give. The new schema stores scopes as exact, case-sensitive tokens and enforces the configured scope on every write and search. It refuses to start on an index built by an older version, and `overwrite_index=True` does not migrate it. To upgrade: delete the old context index and its context hashes, let the provider create the new index, and re-ingest your context data. Plan for this before you bump the package: there is no in-place migration.

**The Mem0 fix (.NET).** The .NET `Mem0Provider` already refused empty storage and search scopes, but only when it created fresh state. A session restored from a saved state bag skipped that check. Per PR #9827, with empty restored scopes, clearing stored memories sent a delete request to Mem0 with no scope filter, and storage failures were only logged. From 1.27.0 the search, store and clear paths all run the same scope check and throw instead.

**Agent-triggered compaction.** Compaction strategies used to run only on rules the developer set, such as a token count. The new `AgentTriggeredCompactionStrategy` wraps an existing strategy and gives the agent one tool to ask for compaction itself, for example after it finishes planning. The wrapped strategy still decides what is removed. `UnloadSkillsCompactionStrategy` adds an `unload_skill` tool that drops loaded skills from the context once the agent is done with them. Neither changes the saved history in sessions or history providers; they only trim what the model sees during the run.

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

Its long-term memory is a `MemoryContextProvider`, a history provider that also keeps a Markdown memory store. These defaults come from the provider's [source](https://github.com/microsoft/agent-framework/blob/main/python/packages/core/agent_framework/_harness/_memory.py) at 1.25.0, read on 9 November 2026; every one is a constructor argument:

| Piece | Default | What it does |
|---|---|---|
| `MEMORY.md` | Up to 200 pointer lines of 150 characters (`index_line_limit`, `index_line_length`) | One line per topic: slug, summary, last update. Rebuilt from the topic files before every run |
| `topics/<slug>.md` | One file per topic | The durable memory lines for that topic, plus a summary |
| `transcripts/` | Saved every run | The conversation history, written by the same provider (`FileHistoryProvider` underneath) |
| `state.json` | Per owner | Consolidation bookkeeping: last pass time, sessions finished since, pending turn ids |
| Auto-loaded topics | 3 (`selection_limit`) | Topic files whose pointer matches keywords in the user's message are injected with `MEMORY.md`; 0 turns it off |
| Recent turns | 0 (`recent_turns`) | How many recent transcript turns to inject next to durable memory |
| Extraction | 5 items per turn (`max_extractions`) | After each run, the agent's own chat client reads the turn and returns `{"memories":[{"topic","memory"}]}`: durable facts, preferences, decisions, one sentence each, no "transient tasks ... or tool chatter" |
| Consolidation | 5 sessions and 24 hours (`consolidation_min_sessions`, `consolidation_interval`) | Rewrites each topic file "into a tighter durable form": dedupes, drops stale items, shortens the summary. `consolidation_client` lets the pass run on a cheaper model |
| Tools given to the model | `list_memory_topics`, `read_memory_topic`, `write_memory`, `delete_memory_topic`, `search_memory_transcripts`, `consolidate_memories` | All run without approval prompts; `consolidate_memories` forces an immediate pass across all topic files |

So the harness follows the same index-plus-topic-files layout as Claude Code's auto memory (also a 200-line `MEMORY.md`, see [Claude Code memory](/articles/claude-code-memory/)), with two differences: extraction is a separate model call after every turn rather than the agent writing notes mid-task, and consolidation is scheduled rather than left to the user. The 1.25.0 fixes above matter here. Before [#9294](https://github.com/microsoft/agent-framework/pull/9294), a consolidation pass that started while another session was still extracting cleared that session from the pending list, so its facts were saved but didn't count toward the next pass and "wait for unrelated sessions before they are consolidated". And before [#9547](https://github.com/microsoft/agent-framework/pull/9547), a half-written or hand-edited `state.json` raised `JSONDecodeError` on every later turn; it is now quarantined next to the original and the store restarts from a clean state. The source also warns that the storage directory must not be writable by untrusted users, since the final memory files could be swapped outside the store's own atomic writes, the same class of risk covered on [memory poisoning in AI agents](/articles/ai-memory-injection/).

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

### Why does RedisContextProvider reject my index after upgrading Agent Framework?

Python 1.24.0 (30 October 2026) changed the beta RedisContextProvider index schema so user, agent, application and session scopes match exactly, and it refuses indexes built by older versions. overwrite_index=True does not migrate them. Delete the old context index and its context hashes, let the provider create a new index, and re-ingest the context data.

### How does the Agent Framework Harness Agent store long-term memory?

As Markdown files: a MEMORY.md index of up to 200 one-line topic pointers, one file per topic under topics/, saved transcripts under transcripts/ and a state.json for bookkeeping. Before each run the provider injects MEMORY.md plus up to 3 topic files whose keywords match the user's message. After each run a model call extracts up to 5 durable memory items from the new turn, and a consolidation pass rewrites topic files once 5 sessions have finished and 24 hours have passed since the last pass. All of these are constructor arguments on MemoryContextProvider.

