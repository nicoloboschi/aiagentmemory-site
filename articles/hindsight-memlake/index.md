# Hindsight Memlake vs Postgres, turbopuffer and LanceDB

> Hindsight Memlake vs pgvector, turbopuffer, LanceDB and Pinecone for agent memory: architecture, writes, cold start, licensing, Vectorize-run numbers, limits.

Source: https://aiagentmemory.org/articles/hindsight-memlake/ · Updated 2026-10-20 · aiagentmemory.org


**Hindsight Memlake** is an object-storage engine built only for agent memory, sitting under Hindsight's extraction, consolidation and recall. Postgres with pgvector, turbopuffer, LanceDB and Pinecone serverless are general retrieval stores you build memory on yourself. In Vectorize-run benchmarks Memlake beats Hindsight's own Postgres backend by a wide margin on large banks, but it's Enterprise only, and for banks under ~50k memories Vectorize says Postgres is fine.

So the comparison is partly apples to oranges. This page compares them where they do overlap (where the data lives, how writes become visible, which searches run on one copy, cold start, licensing) using each vendor's own docs, checked on 20 October 2026. No benchmark here is invented, and no head-to-head between Memlake and turbopuffer, LanceDB or Pinecone exists.

## What is Hindsight Memlake?

**Hindsight Memlake is a Rust storage engine that keeps an agent's memories, the links between them, their indexes, source documents and mental models in object storage, one bucket per tenant. Servers are disposable caches, coordination is done with conditional writes on the bucket, and one stored copy serves vector, keyword, graph and time search.**

It was announced in the [Memlake launch post](https://hindsight.vectorize.io/blog/2026/10/13/memlake) on 13 October 2026, alongside open-source Hindsight 1.0. It plugs into Hindsight's database layer, so the API, SDKs and agent integrations don't change. Postgres still holds the small relational parts: banks and their settings, the background job queue and the audit log.

Memlake isn't sold as a standalone database. It's part of **Hindsight Enterprise**. Open-source [Hindsight](https://github.com/vectorize-io/hindsight) (MIT) stays on Postgres.

## Why put agent memory on object storage?

Agent memory has an odd shape for a database. Records are tiny (about 4-5 KB each in Memlake's bucket, per Vectorize), they only accumulate, and every recall runs **four searches over the same data**: by meaning, by keyword, through relationships and through time. Each write should also be visible to the next query.

On a relational store, every memory pays three times: a vector index entry, a full-text index entry and rows in a link table. Vectorize's launch post reports that on Hindsight's Postgres backend, ingest topped out at 55-80 memories per second. The cost grows with the bank, so a store that only grows needs a bigger primary forever.

Object storage flips that trade. Storage is cheap and durable, and compute can be added or thrown away. The price is latency on first reads, which every engine on this page handles with a local cache (RAM and NVMe) and routing each bank or namespace to the same server.

The design Memlake uses, as described in the launch post:

1. Each retain is written to an append-only **log** in the bucket with a conditional create, so two writers can't claim the same slot.
2. A **head pointer** is advanced with another conditional write, and only then is the write acknowledged. The next recall from any server sees it.
3. Separate **indexers** fold new log entries into small immutable **segments** and publish them by swapping a **manifest**, again with a conditional write.
4. On recall, a server reads only the segment blocks it needs, scans the not-yet-indexed tail of the log, and runs all four searches.
5. Vectors use an IVF index with RaBitQ compression (~60 bytes per vector), then rerank at full precision. Keyword search is BM25 on [tantivy](https://github.com/quickwit-oss/tantivy), including Chinese segmentation.
6. The write, indexing and garbage-collection protocols are specified in TLA+ and model-checked within a bounded scope.

## Memlake vs pgvector, turbopuffer, LanceDB and Pinecone: architecture

Every non-Memlake cell comes from that product's own docs, linked below the table. "Not documented" means we couldn't find it in their docs, not that it's missing.

| | Hindsight Memlake | Postgres + pgvector | turbopuffer | LanceDB | Pinecone serverless |
|---|---|---|---|---|---|
| What it is | Storage engine under a full memory system | Postgres extension for vector search | Managed vector and full-text search on object storage | Open-source embedded retrieval library; commercial Enterprise | Managed vector database |
| Source of truth | Object storage, one bucket per tenant | Postgres's own storage | Object storage (WAL plus indexes per namespace) | Lance files on local disk, block, file or object storage (S3, GCS, Azure) | "Distributed object storage" |
| Coordination | Conditional writes on a manifest and head pointer; no database or consensus cluster beside the bucket | Postgres (single primary, ACID transactions) | "All concurrency control is delegated to object storage" | S3 atomic writes; "no external commit coordinator is required" | Not documented (write log with sequence numbers, run by Pinecone) |
| Searches on one copy | Vector, BM25, graph, time | Vector (HNSW, IVFFlat), Postgres full-text, SQL filters and joins | Vector ANN, BM25, filters; server-side RRF fusion | Vector, full-text (BM25), hybrid, scalar filters | Dense and sparse in one index; BM25 full-text mentioned |
| Graph search | Built in | Not built in; model links as tables and join | Not documented | Not documented | Not documented |
| Write ack and visibility | Durable in the bucket before ack; next recall from any server sees it | Visible on transaction commit | Durable in object storage on return; strongly consistent by default; one WAL commit per second per namespace | Writes consistent; other readers refresh per `read_consistency_interval` (default: no automatic refresh) | Logged and acked as durable; searchable immediately from the memtable |
| Indexing of new data | Async indexers; unindexed log tail is scanned | Synchronous with the insert | Async; unindexed data is scanned exhaustively | OSS: manual `optimize()`; Enterprise: automatic in the background | Background builder moves memtable data into slabs and compacts them |
| Cold start | 8.6 s p95 to open a 10M-memory bank; ~1 s p99 until blocks are cached | Not documented | p50 874 ms cold vs 14 ms warm (1M docs) | Object storage: "hundreds of milliseconds and higher p95"; local NVMe p95 under 10 ms | Uncached slabs load from object storage; no figure given |
| License / offer | Commercial, Hindsight Enterprise only | Open source, PostgreSQL License | Commercial managed; BYOC and single-tenancy on the enterprise plan | Apache-2.0 library; Enterprise commercial | Commercial managed; BYOC on AWS, GCP and Azure |

Sources: [pgvector README](https://github.com/pgvector/pgvector); turbopuffer [architecture](https://turbopuffer.com/docs/architecture), [guarantees](https://turbopuffer.com/docs/guarantees), [hybrid search](https://turbopuffer.com/docs/hybrid) and [pricing](https://turbopuffer.com/pricing); LanceDB [storage](https://docs.lancedb.com/storage/), [consistency](https://docs.lancedb.com/tables/consistency), [reindexing](https://docs.lancedb.com/indexing/reindexing), [Enterprise architecture](https://docs.lancedb.com/enterprise/architecture) and [GitHub](https://github.com/lancedb/lancedb); Pinecone [database architecture](https://docs.pinecone.io/guides/get-started/database-architecture), [hybrid search](https://docs.pinecone.io/guides/search/hybrid-search) and [BYOC](https://docs.pinecone.io/guides/production/bring-your-own-cloud).

Two details matter for memory workloads. turbopuffer says that once a namespace has more than 128 MiB of outstanding writes, new writes aren't visible until they're indexed, and staleness of up to about an hour can follow very large writes. pgvector's README notes that with approximate indexes, filters are applied after the index scan, so strict filters can return fewer rows unless you turn on iterative index scans (0.8.0+).

## The key difference: a store vs a memory system

turbopuffer, LanceDB, Pinecone and pgvector answer "which stored items are closest to this query?" That's the bottom layer. To get **agent memory** out of them you still build:

- **Extraction:** an LLM step that turns a conversation into facts, entities and timestamps.
- **Consolidation:** merging and updating facts when they repeat or contradict each other.
- **Links:** a graph of which memories share an entity, caused each other, or happened in sequence.
- **Recall:** running several searches, fusing them and fitting the result into a token budget.
- **Reasoning:** answering a question over the memories, not just returning them.

Hindsight does all five (retain, observations, four-way recall, mental models and `reflect`), and Memlake is only the storage under it. So "Memlake vs turbopuffer" is really "Hindsight Enterprise vs Hindsight-like logic you write on turbopuffer". The stores are good at their job. turbopuffer and LanceDB share Memlake's core idea of object storage as the source of truth with caching servers in front, and LanceDB's library is free to run anywhere.

What Memlake adds for this one workload is that the graph and time indexes live in the same segments as the vector and keyword indexes, and that the memory system's ranking runs next to the data. With a general store you'd keep the link graph elsewhere (often a Postgres table or a graph database) and fuse results in your own code. That's how it's commonly done today; see [vector databases for LLM memory](/articles/vector-database-for-llm-memory/) and [RAG vs agent memory](/articles/rag-vs-agent-memory/).

## Memlake's numbers (Vectorize-run benchmarks)

All figures below come from Vectorize's own [launch post](https://hindsight.vectorize.io/blog/2026/10/13/memlake). They are **Vectorize-run benchmarks, not independent**, and every recall number was measured without the cross-encoder reranker.

| What was measured | Result |
|---|---|
| Fleet scale | 100M memories over 20 banks on 4 servers of 6 vCPU, latency flat from 1 to 200 recalls/s |
| One 3M-memory bank | 71 ms p50, 90 ms p99 |
| Largest single bank run | 10M memories |
| Bank growth | 100k to 3M memories (30x) made recall about 6x slower under load |
| Ingest per bank | 1,200-1,500 memories/s, 15-27x Hindsight's Postgres backend (55-80/s) |
| Retrieval quality | Within 0.3% of an unnamed "established vector database" on BEIR (SciFact, NFCorpus), July engine |
| Storage | About 4-5 KB per memory |

The ingest gap rarely matters on its own: every Hindsight retain goes through an LLM first, so the LLM is usually the bottleneck, not the store.

## Limits and caveats

These matter as much as the wins:

- **Enterprise only.** Memlake isn't in open-source Hindsight, which stays on Postgres. You can't self-host it for free the way you can pgvector or LanceDB.
- **Cold start is real.** Opening a 10M-memory bank takes 8.6 s at p95, once per server, and queries can take around 1 s at p99 until blocks are cached.
- **The Postgres comparison isn't on identical hardware.** Postgres ran on a 4-vCPU primary shared with other development tenants, so read the ratios as indicative.
- **Costs are modelled, not billed.** Vectorize's cost figures come from its internal sizing calculator.
- **The quality check is against an unnamed database** on two BEIR datasets, with an older engine.
- **No head-to-head with turbopuffer, LanceDB or Pinecone** has been published by anyone.
- **Small banks don't need it.** Vectorize says banks under about **50,000 memories** each are fine on Postgres, and small deployments can be cheaper there.

## Which one fits?

- **Postgres + pgvector:** you already run Postgres, banks are small to medium, and you want memory next to app data with transactions and joins. It's also what open-source Hindsight uses, so you get the full memory system for free.
- **turbopuffer:** you're building your own retrieval or memory layer, want a managed object-storage engine with vector plus BM25 and server-side fusion, and accept a commercial service with monthly minimums.
- **LanceDB:** you want an open-source embedded library that can sit on local disk or object storage, especially with multimodal data, and you're fine running reindexing yourself (or paying for Enterprise).
- **Pinecone serverless:** you want a fully managed vector database with dense and sparse search and don't want to run infrastructure.
- **Hindsight Memlake:** you want a complete agent memory system, not just a store, and your banks are large or numerous: long-running autonomous agents, or one bank per employee across a big company, with data in a bucket you control.

If you're starting out, run open-source Hindsight on Postgres and move to Memlake only if your banks outgrow it. How Hindsight's memory layer works in general is covered in [Hindsight AI memory: how it works](/articles/ai-memory-hindsight/).


## FAQ

### What is Hindsight Memlake?

Memlake is a storage engine for Hindsight agent memory, written in Rust and launched on 13 October 2026. It keeps memories, links, indexes and mental models in object storage (S3, GCS or a certified S3-compatible store) and serves vector, keyword, graph and time search from one copy. It is part of Hindsight Enterprise, not the open-source release, which stays on Postgres.

### Is Memlake faster than turbopuffer or LanceDB?

Nobody has published that comparison. Vectorize's numbers compare Memlake with Hindsight's own Postgres backend and, on BEIR quality, with an unnamed vector database. turbopuffer and LanceDB are also object-storage retrieval engines, but they are general stores you would build memory on, while Memlake sits under a full memory system, so a fair head-to-head would need the same memory logic on top of each.

### Do I need Memlake, or is Postgres with pgvector enough for agent memory?

For most teams Postgres is enough. Vectorize itself says banks under about 50,000 memories each are fine on Postgres and that small deployments can be cheaper there. Memlake's gains show up when single banks grow large or there are many of them, and it is only available in Hindsight Enterprise.

