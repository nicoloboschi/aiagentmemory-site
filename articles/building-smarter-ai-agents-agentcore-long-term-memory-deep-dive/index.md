# AgentCore Memory: AWS Long-Term Memory Deep Dive

> How Amazon Bedrock AgentCore Memory works: events, short-term vs long-term memory, the four built-in strategies, direct ingestion, record TTL, harness managed memory, record streaming, metadata filters, Python code and pricing (November 2026).

Source: https://aiagentmemory.org/articles/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/ · Updated 2026-11-08 · aiagentmemory.org


**AgentCore long-term memory** is the part of Amazon Bedrock AgentCore Memory that turns raw conversation events into durable records an agent can search later. You write turns as **events** (short-term memory). Configured **memory strategies** then extract facts, preferences, summaries or episodes from those events and store them as **memory records** in namespaces, where any later session can retrieve them.

AgentCore became generally available in October 2025. This page follows the [AgentCore Memory developer guide](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html), the API reference and the [pricing page](https://aws.amazon.com/bedrock/agentcore/pricing/), checked on 8 November 2026. The [new additions since October](#whats-new-in-agentcore-memory-october-and-november-2026) (harness managed memory, record streaming, metadata filters, cross-account access, redrive) are covered below.

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

### Direct ingestion: long-term memory without events

Since 8 September 2026 you don't have to store data as a short-term event to get long-term records from it. The **`IngestData`** API ([AWS announcement](https://aws.amazon.com/about-aws/whats-new/2026/09/agentcore-memory-direct-ingest)) sends content straight to the memory's configured long-term strategies.

| | `CreateEvent` | `IngestData` |
|---|---|---|
| Stored as a short-term event | Yes, kept until the event expiry | No |
| Runs long-term strategies | Yes | Yes, the same extraction pipeline |
| Payloads | Conversation turns or blobs, per actor and session | Conversation messages (`USER` / `ASSISTANT` roles) or JSON, such as behavior events, activity logs and system events, with optional metadata |
| Check the result | `ListMemoryRecords`, `RetrieveMemoryRecords` | The same, plus Kinesis notifications and `ListMemoryExtractionJobs` to retry failed extractions |
| Regions | All AgentCore Memory regions | All AgentCore Memory regions |

Use it to backfill memory from history you already keep elsewhere (old chat logs, CRM activity, app events) without replaying it as sessions, or when you only want the extracted facts and not the raw turns. The announcement doesn't say how direct ingestion is billed, so check the [pricing page](https://aws.amazon.com/bedrock/agentcore/pricing/) before a large backfill. In open source, [Hindsight](https://github.com/vectorize-io/hindsight)'s `retain` call is the equivalent single step: send it content and a model extracts facts, time, entities and relationships into long-term memory, with no separate event store in front.

### Expiring long-term records (TTL)

Events always expired (3 to 365 days), but until October 2026 long-term records stayed forever. Since 13 October 2026 you can give them a **time-to-live** ([AWS announcement](https://aws.amazon.com/about-aws/whats-new/2026/10/agentcore-memory-long-term-memory-ttl/), [developer guide](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/long-term-record-ttl.html)). The default is still no expiry.

| | Strategy-level TTL | Request-level TTL |
|---|---|---|
| Set on | `CreateMemory` / `UpdateMemory`, per strategy | `BatchCreateMemoryRecords` / `BatchUpdateMemoryRecords`, per call |
| Applies to | Records that strategy extracts (built-in or with overrides) | Every record in that batch; the strategy's TTL is never used here |
| Value | `recordExpiryDurationDays`, minimum 31 | Same |
| Must match | Semantic and user preference share one value; summary and episodic share one value | n/a |
| Self-managed strategy | Not allowed | The only way to set retention |
| Remove it | `"action": "REMOVE"` | Same |

Three details matter in practice:

- **Changes are not retroactive.** A TTL is stamped on each record when it's made. Changing or removing it affects only records created afterwards, and expired records can't be recovered.
- **Expiry is not immediate.** Background deletion "can typically take a few days", and until then `RetrieveMemoryRecords`, `GetMemoryRecord` and `ListMemoryRecords` can still return the record. For a privacy deletion, call `DeleteMemoryRecord` or `BatchDeleteMemoryRecords` instead.
- **It needs its own permission.** Any request carrying a TTL also needs `bedrock-agentcore:UpdateMemoryRecordExpirySpecification`, or it fails with `AccessDeniedException`. AWS's reason: a short TTL is a delayed delete, so without this permission anyone who can update a memory could delete records they aren't allowed to delete directly.

```bash
aws bedrock-agentcore-control create-memory \
  --name "SupportAgentFactMemory" \
  --event-expiry-duration 90 \
  --memory-strategies '[{"semanticMemoryStrategy": {
      "name": "FactExtractor",
      "namespaceTemplates": ["/support_cases/{sessionId}/facts/"],
      "recordExpirySpecification": {"recordExpiryDurationDays": 60}}}]'
```

A TTL deletes records by age, not by whether they're still true or used. For memory that should fade by relevance instead, see [AI memory decay](/articles/ai-memory-decay/).

## What's new in AgentCore Memory (October and November 2026)

Five additions landed in the developer guide after the TTL release. None changed the pricing table below; all were read in the docs on 8 November 2026.

| Addition | What it does | Where it's documented |
|---|---|---|
| **Harness managed memory** | New harnesses get an AgentCore Memory resource by default, with automatic retrieval before each turn | [Harness: Memory](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-memory.html) (page dated 28 October 2026) |
| **Memory record streaming** | Pushes `MemoryRecordCreated` / `Updated` / `Deleted` events to a Kinesis Data Stream in your account | [Memory record streaming](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-record-streaming.html) |
| **Metadata filtering** | Up to 10 indexed keys per memory, up to 5 `AND` filters per retrieval; the LLM can fill keys from the conversation | [Structured metadata](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/long-term-memory-metadata.html) |
| **Cross-account access** | A resource policy lets another AWS account call the memory's data-plane APIs; S3, SNS and Kinesis destinations can live in another account | [Cross-account memory access](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-cross-account-access.html) |
| **Redrive failed ingestions** | `ListMemoryExtractionJobs` shows failed extractions with a `failureReason`; `StartMemoryExtractionJob` re-runs one | [Redrive failed ingestions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/long-term-redrive.html) |

### The harness turns memory on by default

The AgentCore **harness** (GA, documented alongside the [AgentCore CLI 1.0 release](https://aws.amazon.com/about-aws/whats-new/2026/10/amazon-bedrock-agentcore-cli-version-1-0-available/) of 28 October 2026) is AWS's managed agent loop: you declare model, tools, skills and instructions, and AWS runs the compute, filesystem, identity and memory. The memory part is this service:

- **Managed memory is the default.** Creating a harness without a memory configuration provisions an AgentCore Memory resource for it. The CLI does the same unless you pass `--memory` with mode `disabled` or `existing`. The strategy names are `SEMANTIC`, `SUMMARIZATION`, `USER_PREFERENCE` and `EPISODIC`, set with `eventExpiryDuration` for the raw events; you change them later through `UpdateHarness`, and can still read and write events and records through the Memory APIs directly.
- **Bring your own memory** when you need custom namespace templates or one memory shared by several harnesses: attach an existing resource by ARN, or by name if the same project declares it. Only the by-name form accepts a `retrievalConfig` and survives `agentcore export harness`.
- **Retrieval is automatic.** With active strategies, the harness queries long-term memory on every invocation with `topK=10` and `relevanceScore=0.2` per strategy namespace and injects the results before the model reasons. Pass your own `retrievalConfig` to change `topK`, the score threshold or the namespaces, or to skip a strategy. If you add or remove strategies on a BYO memory, call `UpdateHarness` or the retrieval configuration goes stale.
- **Per-user scoping** comes from the `actorId` you pass at invoke time; each actor gets isolated short-term and long-term memory.
- **Context truncation** is separate from memory: `sliding_window` (default, keeps the last N messages), `summarization` (compresses older turns) or `none`.
- **Cost.** The harness itself is free; managed memory bills at the standard AgentCore Memory rates below for events, stored records and retrievals. AWS's own advice: "To avoid persistent Memory charges, disable Memory when you create the harness."

```bash
agentcore add harness --name Support \
  --memory '{"mode":"managed","strategies":["SEMANTIC","SUMMARIZATION"],"eventExpiryDuration":30}'
# or, no memory at all:
agentcore add harness --name Support --memory '{"mode":"disabled"}'
```

### Streaming, metadata and the other operational additions

- **Record streaming** fires on extraction from events, direct `BatchCreateMemoryRecords`, `BatchUpdateMemoryRecords`, explicit deletes and, notably, on **consolidation**: when the pipeline de-duplicates or supersedes a record, you get a `MemoryRecordDeleted` event. Choose `METADATA_ONLY` (ids and timestamps) or `FULL_CONTENT` (adds `memoryRecordText`) per stream; deletion events carry only the ids either way. The memory needs a `memoryExecutionRoleArn` that can write to your stream, and a `StreamingEnabled` test event confirms the setup.
- **Metadata filtering** adds structured attributes inside a namespace. You declare up to 10 indexed keys per memory (`STRING`, `STRINGLIST` or `NUMBER`); a key can't be removed once added, and existing records aren't backfilled. A query combines up to 5 filters with `AND`, using `EQUALS_TO`, `CONTAINS`, numeric comparisons, and `BEFORE` / `AFTER` on the system timestamps `x-amz-agentcore-memory-createdAt` and `updatedAt`, which need no indexed key. Each strategy can carry a metadata schema that tells the LLM what to extract from the conversation (for example `sentiment`, or an escalated `priority`, with allowed values capped at 10). Keys marked `STRICTLY_CONSISTENT` are copied from the event without LLM inference, and they also **partition extraction and consolidation**: a record tagged `compliance_level: hipaa` never merges with one tagged `standard`. Changing which keys are strict isolates old records from new ones, so AWS says to plan this before ingesting, and to start with 3 to 5 keys.
- **Cross-account** access attaches a resource-based policy to the memory with `PutResourcePolicy`, so a role in another account can call `BatchCreateMemoryRecords`, `RetrieveMemoryRecords` and the other data-plane actions by passing the full memory ARN as the memory id. Self-managed strategy and streaming destinations in another account need the execution role plus bucket, topic or stream policies on the other side.
- **Redrive** covers the case where extraction from events to records fails. For built-in strategies AWS says "the only failure scenario is hitting the ingestion limit"; overrides add model and permission failures. Watch the `FailedExtraction` CloudWatch metric, list the failed jobs, fix the reason, then re-run each job by id.

What this adds up to: AgentCore Memory now has the operational surface of a database (change feed, typed filters, cross-account IAM, dead-letter redrive) on top of the extraction pipeline. The open-source equivalents are partial. [Hindsight](https://github.com/vectorize-io/hindsight) has no change stream; its data-movement path is a bank export to a transfer ZIP that another Hindsight instance restores or merges, in its own format rather than a cross-vendor one. It scopes memory with isolated banks and a retrieval token budget rather than per-record indexed keys. If you need Kinesis-style events on every memory change today, AgentCore is the one that ships them.

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

## AgentCore Memory pricing (checked November 2026)

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

### Can I write to AgentCore long-term memory without creating events?

Yes. Since 8 September 2026 the IngestData API sends conversation messages or JSON payloads (activity logs, behavior or system events) straight to the memory's long-term strategies without storing them as short-term events. Check results with ListMemoryRecords or RetrieveMemoryRecords and retry failed extractions with ListMemoryExtractionJobs.

### Do AgentCore long-term memory records expire?

Not by default. Since 13 October 2026 you can set a time-to-live of at least 31 days per strategy (CreateMemory or UpdateMemory) or per batch write (BatchCreateMemoryRecords or BatchUpdateMemoryRecords). It applies only to records created after the change, deletion can take a few days, and setting it needs the bedrock-agentcore:UpdateMemoryRecordExpirySpecification permission.

### Does the AgentCore harness use AgentCore Memory?

Yes. A harness created without a memory configuration gets a managed AgentCore Memory resource by default, with SEMANTIC, SUMMARIZATION, USER_PREFERENCE and EPISODIC strategies available, and it retrieves long-term memories automatically before each turn (topK 10, relevance score 0.2 per strategy). You can attach an existing memory instead, or pass --memory with mode disabled. The harness is free but managed memory bills at normal AgentCore Memory rates.

### Can I filter AgentCore long-term memory by metadata?

Yes, since late 2026. Declare up to 10 indexed keys per memory (string, string list or number), attach metadata to events or let the strategy's metadata schema have the LLM extract it, then combine up to 5 AND filters on RetrieveMemoryRecords or ListMemoryRecords. Keys marked STRICTLY_CONSISTENT are copied without LLM inference and also keep records with different values from merging during consolidation.

### What is the best way to give an LLM memory on AWS?

For new projects AWS points to Amazon Bedrock AgentCore Memory, a managed service for both session history and extracted long-term memory. The older Bedrock Agents memory feature (session summaries kept up to 365 days) belongs to Bedrock Agents Classic, which is closed to new customers. You can also run an open-source memory server on your own AWS infrastructure.

