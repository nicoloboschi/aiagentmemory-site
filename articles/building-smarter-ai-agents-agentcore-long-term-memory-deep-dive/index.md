# AgentCore Memory: AWS Long-Term Memory Deep Dive

> How Amazon Bedrock AgentCore Memory works: events, short-term vs long-term memory, the four built-in strategies, namespaces, Python code and October 2026 pricing.

Source: https://aiagentmemory.org/articles/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/ · Updated 2026-10-12 · aiagentmemory.org


**AgentCore long-term memory** is the part of Amazon Bedrock AgentCore Memory that turns raw conversation events into durable records an agent can search later. You write turns as **events** (short-term memory). Configured **memory strategies** then extract facts, preferences, summaries or episodes from those events and store them as **memory records** in namespaces, where any later session can retrieve them.

AgentCore became generally available in October 2025. This page follows the [AgentCore Memory developer guide](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html), the API reference and the [pricing page](https://aws.amazon.com/bedrock/agentcore/pricing/), checked on 8 October 2026.

## What is AgentCore Memory?

**AgentCore Memory is a fully managed AWS service that stores an agent's conversation history as events and, through configurable strategies, extracts long-term memory records from them. Records are organized by namespaces built from actor and session IDs, and retrieved by semantic search so agents can personalize answers across sessions.**

It handles both layers of [short-term and long-term memory](/articles/ai-agent-memory-explained/):

- **Short-term memory** is turn-by-turn context inside one session. It's the raw events you send with `CreateEvent`. When the user asks "What about tomorrow?", the agent reads the recent events to know they meant tomorrow's weather in Seattle.
- **Long-term memory** is what strategies extract across sessions: "prefers window seats," an order number, a summary of last week's support case.

A memory resource with no strategies stores events only. AWS says plainly that without strategies, no long-term records are extracted.

### Key building blocks

| Concept | What it is |
|---|---|
| **Memory resource** | The container. Has a name, an event expiry (3 to 365 days), optional KMS key and strategies |
| **Event** | One write: conversation turns or other data, tied to an `actorId` and `sessionId` |
| **Actor** | Usually the end user, or an agent |
| **Strategy** | The rule for what to extract: semantic, user preference, summary, episodic or custom |
| **Memory record** | One extracted item of long-term memory |
| **Namespace** | A path such as `/strategy/{memoryStrategyId}/actors/{actorId}/` that groups records |

Events also support **branching**, so you can fork a conversation from an earlier event to try an alternative path without losing the original.

## The four built-in memory strategies

Each built-in strategy runs an LLM pipeline with up to three steps. **Extraction** pulls insights from events. **Consolidation** decides whether to write a new record or update an existing one. **Reflection**, only in the episodic strategy, draws insights across episodes.

| Strategy (API name) | What it keeps | Example | Steps |
|---|---|---|---|
| Semantic (`semanticMemoryStrategy`) | Standalone facts from the conversation | "The user runs version 2.1 of the software" | Extraction, consolidation |
| User preference (`userPreferenceMemoryStrategy`) | Preferences, choices and styles, with context and categories | "Prefers a formal tone" | Extraction, consolidation |
| Summary (`summaryMemoryStrategy`) | Condensed summaries of one session, by topic | "User reported order #XYZ-123; agent sent a replacement" | Consolidation |
| Episodic (`episodicMemoryStrategy`) | Structured episodes plus reflections | "Deploy failed with tool A, succeeded with tool B" | Extraction, consolidation, reflection |

The semantic and user preference strategies only read `USER` and `ASSISTANT` messages. Both default to the namespace `/strategy/{memoryStrategyId}/actors/{actorId}/`, so records are grouped per user. Summaries default to a per-session namespace, `/strategy/{memoryStrategyId}/actor/{actorId}/session/{sessionId}/`. You can combine several strategies on one memory resource.

### Episodic memory and reflections

The episodic strategy, added after GA, is the closest AgentCore comes to [episodic memory in AI agents](/articles/episodic-memory-in-ai-agents/). AgentCore detects when an episode is complete and writes one structured record with the situation, intent, assessment, justification and an episode-level reflection. Records only appear after an episode ends, so they lag behind the other strategies.

**Reflections** then analyze many episodes to find patterns: which tool combinations tend to succeed, which approaches fixed past failures. AWS recommends including `TOOL` results in your events for best results. Reflections can be stored at the actor level or the strategy level. The docs flag a privacy point: strategy-level reflections can span multiple users, so reflect at the actor level if that's a concern.

### Overrides and self-managed strategies

Built-ins are the easiest path but the least flexible. Two other options trade setup for control:

- **Built-in with overrides**: you change the extraction or consolidation prompts and choose a Bedrock model, which runs in your account. Storage is cheaper than built-in.
- **Self-managed**: AgentCore notifies your pipeline (via S3 and SNS) when trigger conditions are met, and you run your own extraction with any model, then write records back. You own the infrastructure.

## Using AgentCore Memory from Python

AWS's getting-started guide creates the memory resource with the AgentCore CLI (`agentcore add memory --strategies SEMANTIC` then `agentcore deploy`) and uses the `bedrock-agentcore` Python SDK for events. You can also create it with boto3:

```python
import boto3
from bedrock_agentcore.memory import MemorySessionManager
from bedrock_agentcore.memory.constants import ConversationalMessage, MessageRole

control = boto3.client("bedrock-agentcore-control", region_name="us-west-2")

# Create a memory resource with one semantic strategy (takes a few minutes)
memory = control.create_memory(
    name="SupportMemory",
    eventExpiryDuration=30,  # days to keep raw events (3-365)
    memoryStrategies=[{
        "semanticMemoryStrategy": {
            "name": "facts",
            "namespaceTemplates": ["/facts/{actorId}/"],
        }
    }],
)["memory"]

# Write conversation turns as events (short-term memory)
manager = MemorySessionManager(memory_id=memory["id"], region_name="us-west-2")
session = manager.create_memory_session(actor_id="user-123", session_id="order-42")
session.add_turns([
    ConversationalMessage("My order #35476 hasn't arrived.", MessageRole.USER),
    ConversationalMessage("Sorry to hear that. Let me check.", MessageRole.ASSISTANT),
])

# Later, in any session: search extracted long-term records
records = session.search_long_term_memories(
    query="open support issues", namespace_path="/facts/user-123/", top_k=3
)
for record in records:
    print(record)
```

Extraction is asynchronous, so records show up some time after the events. For Strands agents, the SDK ships an `AgentCoreMemorySessionManager` that loads history and retrieves records per namespace automatically.

## AgentCore Memory pricing (October 2026)

| Item | Price |
|---|---|
| Short-term ingestion | $1.00 per GB (from 6 Oct 2026; was $0.25 per 1,000 events) |
| Short-term storage | $0.10 per GB-month over each event's TTL |
| Short-term retrieval | $0.20 per GB retrieved |
| Long-term storage, built-in strategies | $0.75 per 1,000 records per month |
| Long-term storage, override or self-managed | $0.25 per 1,000 records per month, plus model use in your account |
| Long-term retrieval | $0.50 per 1,000 retrievals |

Short-term events are billed with a 12 KB minimum and 64 KB maximum per event, so lots of tiny events cost more than their raw size suggests. A rough example: 10,000 users with 50 built-in records each is 500,000 records, or $375 a month in storage. Every 100,000 retrieval calls add $50. There's no memory-specific free tier on the pricing page.

## LLM memory on AWS: the options

If you searched for "LLM memory on AWS," these are the realistic routes:

| Option | What it gives you | Status |
|---|---|---|
| **AgentCore Memory** | Managed events plus extracted long-term records, any framework | Current, GA |
| **Bedrock Agents memory** | Session summaries kept 1 to 365 days, keyed by `memoryId` | Bedrock Agents Classic, closed to new customers |
| **Self-hosted memory server** | Run an open-source system on EC2, ECS or EKS with your own database | Your responsibility |

The [Bedrock Agents memory docs](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-memory.html) now point new users to AgentCore. For self-hosting, Mem0, Graphiti, Cognee, Letta and [Hindsight](https://github.com/vectorize-io/hindsight) all run in containers and can use Bedrock models; the [open-source memory systems comparison](/articles/open-source-memory-systems-compared/) lists licenses and storage backends.

Of these, Hindsight covers the most of what AgentCore's strategies do, in open source. Its `retain` extracts facts, entities and time from each conversation, background consolidation turns them into **observations** (beliefs with the quotes behind them, refined as evidence arrives), and `reflect` reasons across memories, much like AgentCore's episodic reflections. It's MIT-licensed with every feature in the open-source server, deploys to EKS with its Helm chart, uses Bedrock as its LLM provider, and has an integration for AWS's Strands Agents. You pay for your database and model calls instead of per stored record. The trade-offs: you run and back it up yourself, and every write calls a model. See [what Hindsight is](/articles/ai-memory-hindsight/).

## How AgentCore compares with Azure and Google

All three clouds now sell a managed memory layer with the same basic loop: write conversation data, let an LLM extract and merge facts, retrieve per user. The differences are in the knobs.

- **AgentCore** gives the most control over extraction, from built-ins to fully self-managed pipelines, and is the only one with an episodic strategy that reflects across episodes.
- **[Azure AI Foundry long-term memory](/articles/azure-ai-foundry-long-term-memory/)** has user profile, chat summary and procedural memory, partitioned by `scope`, and was still in preview in October 2026.
- **[Vertex AI Agent Engine Memory Bank](/articles/vertex-ai-agent-engine-memory-bank/)** uses topics and structured profiles, keeps revision history, and integrates natively with Google ADK.

Pick the one in the cloud you already run on. Memory data, IAM and model access are much simpler when they sit next to your agent.


## FAQ

### What is the difference between short-term and long-term memory in AgentCore?

Short-term memory is the raw events (conversation turns) you write with CreateEvent, kept for a set expiry of 3 to 365 days. Long-term memory is the memory records that strategies extract from those events, such as facts, preferences, summaries or episodes, stored in namespaces and searched semantically across sessions.

### How much does AgentCore Memory cost?

As of October 2026, long-term storage is $0.75 per 1,000 records per month with built-in strategies or $0.25 with override or self-managed strategies, and retrieval is $0.50 per 1,000 calls. Short-term memory moved on 6 October 2026 from $0.25 per 1,000 events to $1.00 per GB ingested, $0.10 per GB-month stored and $0.20 per GB retrieved.

### What is the best way to give an LLM memory on AWS?

For new projects AWS points to Amazon Bedrock AgentCore Memory, a managed service for both session history and extracted long-term memory. The older Bedrock Agents memory feature (session summaries kept up to 365 days) belongs to Bedrock Agents Classic, which is closed to new customers. You can also run an open-source memory server on your own AWS infrastructure.

