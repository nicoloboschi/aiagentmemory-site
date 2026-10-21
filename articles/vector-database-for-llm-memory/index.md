# Vector Databases for LLM Memory: How to Choose One

> How vector databases store long-term memory for LLM agents, what memory needs beyond RAG, and how pgvector, Qdrant, Chroma, Milvus, Pinecone and Redis compare.

Source: https://aiagentmemory.org/articles/vector-database-for-llm-memory/ · Updated 2026-10-20 · aiagentmemory.org


**A vector database for LLM memory** stores past messages, facts and documents as embeddings, then finds the ones closest in meaning to the current query so they can be added to the prompt. It's the storage layer under most long-term memory systems. For most teams the best choice is the database they already run (pgvector in Postgres, or Redis), with a dedicated vector database like Qdrant, Milvus or Pinecone once scale or multi-tenancy demands it.

This page explains how vector memory works, what agent memory needs that plain RAG doesn't, and how the main options compare as of October 2026.

## What is a vector database for LLM memory?

**A vector database for LLM memory is a store that keeps text as numerical embeddings alongside metadata (user, time, type) and answers nearest-neighbor queries fast. An agent uses it to save what it learns and, on each turn, retrieve the few memories most similar to the current request.**

The model itself is stateless: it remembers nothing between calls. A vector store gives it a searchable external memory. The [MemGPT paper](https://arxiv.org/abs/2310.08560) (Packer et al., 2023), for example, kept its "archival storage" in PostgreSQL and searched it through function calls. Today's memory frameworks do the same with a choice of backends.

For the bigger picture of where storage fits next to extraction and retrieval, see [AI agent memory explained](/articles/ai-agent-memory-explained/).

## How vector memory works

The loop has two halves.

**Write path:**

1. Take a message, or a fact extracted from it by an LLM.
2. Turn it into an embedding with an embedding model.
3. Store the vector with metadata: `user_id`, timestamp, memory type, source message ID.

**Read path:**

1. Embed the current query.
2. Run an approximate nearest-neighbor (ANN) search, filtered to the right user and time range.
3. Optionally combine with keyword search and rerank.
4. Put the top few results into the prompt.

### ANN indexes

Searching every vector is exact but slow at scale, so databases build an index. Two types dominate:

- **HNSW** (hierarchical navigable small world) builds a layered graph of neighbors. It's fast and accurate but uses more memory. pgvector's defaults are `m = 16` and `ef_construction = 64`, with `hnsw.ef_search = 40` at query time ([pgvector README](https://github.com/pgvector/pgvector)).
- **IVF** (inverted file) clusters vectors into lists and searches only the closest lists. It builds faster and uses less memory but needs tuning (`lists`, `probes`).

Redis's [vector search docs](https://redis.io/docs/latest/develop/ai/search-and-query/vectors/) give a useful rule of thumb: use a `FLAT` (exact) index for small datasets "(< 1M vectors)" or when perfect accuracy matters, and `HNSW` for larger ones.

## What agent memory needs beyond RAG

A RAG index over documents is written once and read many times. Agent memory is different, and those differences should drive the choice of database.

| Need | Why memory needs it | What to look for |
|---|---|---|
| Per-user isolation | Users must never see each other's memories | Fast metadata filters, tenant partitioning |
| Frequent small writes | Every conversation turn can add memories | Good upsert performance, no full reindex |
| Updates and deletes | Facts change; users ask to be forgotten | Delete by ID or filter, quick visibility |
| Time filters | "What did I say last week?" | Range filters on timestamp fields |
| Exact matches | Names, order IDs, product codes | Hybrid search (BM25 or sparse vectors plus dense) |
| Filtered recall | Filters shouldn't silently drop results | Filter-aware ANN (pgvector iterative scans, Qdrant filtering) |

Two of these trip people up most. **Multi-tenancy**: Qdrant's [multitenancy guide](https://qdrant.tech/documentation/manage-data/multitenancy/) says "Creating a separate collection for each tenant is rarely the most efficient approach" and recommends one collection with a tenant field indexed with `is_tenant=true`. Pinecone's docs recommend "one namespace per tenant." **Filtered search**: with plain HNSW, a strict filter can leave fewer results than requested. pgvector 0.8.0 added **iterative index scans** (`hnsw.iterative_scan`) to keep scanning until enough rows pass the filter.

## Vector databases compared for LLM memory

Facts from each project's docs and GitHub, checked 8 October 2026.

| Database | Type | License | Good fit for memory when |
|---|---|---|---|
| [pgvector](https://github.com/pgvector/pgvector) | Postgres extension | PostgreSQL License | You already run Postgres and want memory next to app data, with SQL joins and transactions |
| [Qdrant](https://github.com/qdrant/qdrant) | Dedicated vector DB | Apache-2.0 | You need rich payload filters, tenant partitioning and sparse+dense hybrid search |
| [Chroma](https://github.com/chroma-core/chroma) | Embedded or server | Apache-2.0 | You want the simplest local start (in-memory or on-disk client) |
| [Milvus](https://github.com/milvus-io/milvus) | Distributed vector DB | Apache-2.0 | You expect very large collections; Milvus Lite covers prototypes |
| [Weaviate](https://github.com/weaviate/weaviate) | Dedicated vector DB | BSD-3-Clause (parts under the Weaviate License) | You want built-in hybrid search and modules for embedding |
| Pinecone | Managed service only | Commercial | You want no infrastructure, with namespaces per user |
| [LanceDB](https://github.com/lancedb/lancedb) | Embedded library on the Lance columnar format | Apache-2.0 | You want an embedded store that also handles multimodal data |
| Redis / Amazon MemoryDB | In-memory database with vector search | Redis 8: AGPLv3, RSALv2 or SSPLv1; MemoryDB is managed | Latency matters most and you already use Redis |

A few notes:

- **pgvector** supports HNSW and IVFFlat, and up to 2,000 dimensions per indexed `vector` column (4,000 with `halfvec`). Distance operators include `<=>` for cosine and `<#>` for negative inner product. It needs Postgres 13 or later.
- **Redis** supports `FLAT`, `HNSW` and, since Redis 8.2, `SVS-VAMANA` indexes, with metadata filters on text, tag, numeric and geo fields.
- **Amazon MemoryDB** adds vector search to a durable Valkey/Redis-compatible service. AWS's [docs](https://docs.aws.amazon.com/memorydb/latest/devguide/vector-search.html) list retrieval-augmented generation among its use cases and claim, as of June 2024, the "fastest vector search performance at the highest recall rates among popular vector databases on AWS" (a vendor benchmark).
- **Microsoft Kernel Memory**, once a common .NET option, is now archived. Its README says it "is an archived research project" and the code "serves as a learning resource, not production software."

If your memory must live in RAM inside the app process, with no server, see [in-memory vector databases](/articles/best-in-memory-vector-database/). At the other end, for very large memory banks kept on object storage, [Hindsight Memlake vs Postgres, turbopuffer and LanceDB](/articles/hindsight-memlake/) compares the object-storage engines.

## What memory frameworks use

You often don't pick the vector database directly; the memory framework does, or offers a list.

- **Mem0**'s Python library supports 22 vector stores, from Qdrant, Chroma and pgvector to Pinecone, Redis, Elasticsearch, Turbopuffer and Amazon S3 Vectors. Its [docs](https://docs.mem0.ai/components/vectordbs/overview) say that with no configuration, "`Qdrant` will be used as the vector database."
- **Hindsight** stores memories in PostgreSQL with pgvector (or Oracle AI Database 23ai), with an embedded Postgres (`pg0`) for local use, so teams on Postgres don't add a new database.
- **Honcho** runs on Postgres with pgvector.
- **Letta** keeps archival passages in its own storage behind its API.
- **Graphiti** and other graph-based systems store embeddings inside the graph database (Neo4j, FalkorDB or Neptune) instead of a separate vector store. See [knowledge graphs for AI memory](/articles/ai-memory-knowledge-graph/).

## Example: per-user memory with Qdrant

This stores memories with a user ID and timestamp, then retrieves only that user's memories, optionally within a time range. It uses `qdrant-client` 1.19 and OpenAI embeddings.

```python
import time
import uuid

from openai import OpenAI
from qdrant_client import QdrantClient, models

oai = OpenAI()
qdrant = QdrantClient(":memory:")  # QdrantClient(url="http://localhost:6333") in production

qdrant.create_collection(
    "memories",
    vectors_config=models.VectorParams(size=1536, distance=models.Distance.COSINE),
)
qdrant.create_payload_index(  # co-locates each user's vectors (server mode)
    "memories",
    field_name="user_id",
    field_schema=models.KeywordIndexParams(type=models.KeywordIndexType.KEYWORD, is_tenant=True),
)


def embed(text: str) -> list[float]:
    return oai.embeddings.create(model="text-embedding-3-small", input=text).data[0].embedding


def remember(user_id: str, text: str, kind: str = "fact") -> None:
    qdrant.upsert("memories", points=[models.PointStruct(
        id=str(uuid.uuid4()),
        vector=embed(text),
        payload={"user_id": user_id, "text": text, "kind": kind, "ts": int(time.time())},
    )])


def recall(user_id: str, query: str, since: int | None = None, k: int = 5) -> list[str]:
    must = [models.FieldCondition(key="user_id", match=models.MatchValue(value=user_id))]
    if since:
        must.append(models.FieldCondition(key="ts", range=models.Range(gte=since)))
    hits = qdrant.query_points(
        "memories", query=embed(query), query_filter=models.Filter(must=must), limit=k
    ).points
    return [h.payload["text"] for h in hits]


remember("u42", "Prefers aisle seats on long flights")
print(recall("u42", "Book a flight to Tokyo"))
```

The `user_id` filter is the line that matters most. Leave it out and one user's memories can show up in another user's prompt. Payload indexes only take effect on a Qdrant server, not in local mode. Which embedding model to use is covered in [embedding models for RAG and memory](/articles/embedding-models-for-rag/).

## What a vector database won't do for memory

A vector store answers "what's similar to this?" Agent memory needs more:

- **Deciding what to save.** Storing every message makes search noisy. Extracting facts needs an LLM step.
- **Handling change.** "I live in NYC" and "I moved to SF" are both similar to "where do I live?" The database can't tell which is current.
- **Time reasoning.** Timestamps support filters, not questions like "what changed since my last order?"
- **Relationships.** Multi-hop questions ("which of my teammates use the old API?") need links between facts.

That's the gap memory frameworks fill on top of the database, and why many add a graph or keyword index.

[Hindsight](https://github.com/vectorize-io/hindsight) is one example of all four layered on pgvector. Its `retain` step uses an LLM to extract facts, time, entities and relationships instead of storing raw messages. Related facts are consolidated into observations that get refined as things change. Recall runs vector search alongside BM25 keyword, entity-graph and time-range search, then fuses and reranks the results. On long histories that pays off: it reports 64.1 on BEAM at 10M tokens, where Mem0's comparison page lists Mem0 at 48.6 (vendor-run). The cost is LLM calls on every write and more tokens per retrieval than a plain top-k vector query, so set its `max_tokens` budget. The difference between plain retrieval and memory is covered in [RAG vs agent memory](/articles/rag-vs-agent-memory/).

## How to choose

1. **Start with what you run.** Postgres means pgvector; Redis, MongoDB or Elasticsearch all have vector search.
2. **Count tenants.** Thousands of users need filter-friendly tenant partitioning, not a collection per user.
3. **Estimate size.** Under a few million vectors, almost any option works. At hundreds of millions, look at Milvus, Qdrant, Weaviate or Pinecone.
4. **Check hybrid search.** If users mention IDs, names or codes, you'll want keyword matching too.
5. **Test deletes.** Make sure you can remove one user's data completely and quickly.
6. **Pick the framework first if you use one.** Check which backends it supports before choosing a database.


## FAQ

### Do LLMs need a vector database for memory?

Not strictly, but most long-term memory systems use one. A vector database lets an agent find past messages or facts by meaning, so it can pull a few relevant memories into the prompt instead of resending the whole history. Small apps can use pgvector in an existing Postgres or an embedded store like Chroma.

### What is the best database for AI memory?

For most teams it's the database they already run: pgvector if you use Postgres, or the vector search in Redis, MongoDB or Elasticsearch. Choose a dedicated vector database such as Qdrant, Milvus, Weaviate or Pinecone when you need very large scale, many tenants or advanced filtering. Memory frameworks like Mem0 support over twenty backends.

### Is a vector database enough for agent memory?

Usually not on its own. It finds similar text, but it doesn't decide what to remember, merge duplicates, or know that a newer fact replaced an older one. Memory frameworks add extraction, conflict handling and time-aware ranking on top of the vector store.

