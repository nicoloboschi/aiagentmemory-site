# Knowledge Graphs for AI Agent Memory: How They Work

> How knowledge graph memory works for AI agents: extraction, entity resolution, temporal edges, Graphiti, Neo4j, Cognee and GraphRAG, versus vector memory.

Source: https://aiagentmemory.org/articles/ai-memory-knowledge-graph/ · Updated 2026-10-12 · aiagentmemory.org


**A knowledge graph memory** stores what an AI agent learns as **entities** (a user, a company, a product) connected by **relationships** ("works at", "prefers", "reported bug in"). An LLM extracts those entities and relationships from conversations and documents, and the agent later retrieves facts by searching the graph and following its links. The graph makes connections and changes explicit, which flat vector memory can't do.

This page explains how graph memory is built, how retrieval works, the main tools (Graphiti, Neo4j Agent Memory, Cognee, Hindsight, GraphRAG), and when a graph beats a vector store.

## What is an AI memory knowledge graph?

**An AI memory knowledge graph is a long-term memory store for an LLM agent in which facts are saved as nodes and typed edges, extracted automatically from the agent's interactions. Each edge is a fact that links two entities, often with a timestamp and a pointer back to the message it came from.**

Compare two ways of storing "Dana moved from Acme to Globex in May":

- **Vector memory** stores the sentence as an embedding. A later search for "where does Dana work" may return it, or may return an older "Dana works at Acme" message that's just as similar.
- **Graph memory** stores `Dana -WORKS_AT-> Globex` (valid from May) and marks `Dana -WORKS_AT-> Acme` as ended. The answer is a lookup, and the history is kept.

That's the core appeal. Graphs give memory structure: who is related to what, and since when. For where this fits among other memory types, see [AI agent memory explained](/articles/ai-agent-memory-explained/).

## How knowledge graph memory is built

Every graph memory system runs some version of this write pipeline:

1. **Ingest an episode.** A message, a document or a JSON record arrives with a timestamp.
2. **Extract entities.** An LLM names the people, organizations, objects and concepts in it.
3. **Resolve entities.** "Dana", "Dana K." and "she" must map to one node. The system compares candidates with existing nodes, usually by embedding similarity plus an LLM check.
4. **Extract relationships.** The LLM writes facts as edges between resolved entities.
5. **Handle conflicts.** New edges are compared with existing ones. Contradicted facts are updated, deleted or marked invalid.
6. **Index.** Node names and edge facts get embeddings and full-text indexes for search.
7. **Summarize (optional).** Clusters of related entities get community summaries for broad questions.

Graphiti, the engine inside Zep, is the most documented example. The [Zep paper](https://arxiv.org/abs/2501.13956) (Rasmussen et al., 2025) describes three layers: an **episode subgraph** of raw inputs ("a non-lossy data store"), a **semantic entity subgraph** of extracted entities and facts, and a **community subgraph** of clustered entities with summaries.

### Temporal edges

Facts change, so good graph memory tracks time. Graphiti uses a **bi-temporal model** with four timestamps per edge: `t_valid` and `t_invalid` for when the fact was true in the world, and `t'_created` and `t'_expired` for when the system recorded or retired it. When an LLM finds that a new edge contradicts an older one and their time ranges overlap, the old edge's `t_invalid` is set to the new edge's `t_valid`. More on this in [temporal reasoning in AI memory](/articles/temporal-reasoning-ai-memory/).

## How agents retrieve from a graph

Graph memory rarely relies on graph queries alone. Retrieval usually mixes three searches and merges the results:

- **Semantic search** over embeddings of facts and entity names.
- **Keyword search** (BM25) for exact names, IDs and rare terms.
- **Graph traversal** from matched nodes to their neighbors, to pull in connected facts.

Graphiti implements cosine similarity, Okapi BM25 and breadth-first search, then reranks (options include reciprocal rank fusion, maximal marginal relevance and a cross-encoder). Hindsight's recall similarly runs semantic, BM25, graph and temporal retrieval in parallel and fuses them.

**HippoRAG** ([Gutiérrez et al., 2024](https://arxiv.org/abs/2405.14831)) takes a different route. It's inspired by hippocampal indexing theory: it builds a graph of extracted phrases and runs **Personalized PageRank** from the query's entities. On multi-hop QA it reports up to 20% better results than prior methods, and single-step retrieval that's 10-20 times cheaper and 6-13 times faster than iterative retrieval like IRCoT.

## Knowledge graph vs vector memory

| | Vector memory | Knowledge graph memory |
|---|---|---|
| Unit stored | Text chunk + embedding | Entity nodes and fact edges (often with embeddings) |
| Write cost | One embedding call | Several LLM calls per episode (extract, resolve, dedupe) |
| Good at | Fuzzy recall of related text | Multi-hop questions, entity lookups, how facts connect |
| Changing facts | Old and new both match | Can update or invalidate edges explicitly |
| Time | Metadata filter at best | Validity windows on facts (in temporal graphs) |
| Failure mode | Returns similar but wrong chunk | Extraction errors become false facts |
| Infra | Vector DB or pgvector | Graph DB (Neo4j, FalkorDB, Neptune) plus indexes |

In practice the line is blurry. Most graph memory systems also store embeddings and run vector search. Most vector memory systems add some entity linking. Choose based on the questions your agent must answer. A guide to the vector side is in [vector databases for LLM memory](/articles/vector-database-for-llm-memory/).

## Knowledge graph memory tools

| Tool | What it is | Graph backend | License |
|---|---|---|---|
| [Graphiti](https://github.com/getzep/graphiti) | Temporal context graph framework, core of Zep | Neo4j 5.26, FalkorDB, Amazon Neptune (Kuzu deprecated) | Apache-2.0 |
| Zep | Managed memory service built on Graphiti | Managed | Commercial |
| [Neo4j Agent Memory](https://github.com/neo4j-labs/agent-memory) | Neo4j Labs library: conversations, entity graph, reasoning traces | Neo4j / AuraDB, or hosted service | Apache-2.0 |
| [Cognee](https://github.com/topoteretes/cognee) | Memory platform turning text and code into a graph plus vectors | Several, including Postgres | Apache-2.0 |
| MCP Knowledge Graph Memory Server | Reference MCP server: entities, relations, observations | A local JSONL file | MIT, moving to Apache-2.0 |
| [Hindsight](https://github.com/vectorize-io/hindsight) | Agent memory server: entity, temporal and causal links plus vectors, with consolidation | PostgreSQL + pgvector (or Oracle AI Database 23ai), no separate graph DB | MIT |
| [Microsoft GraphRAG](https://github.com/microsoft/graphrag) | Graph index and community summaries over a document corpus | Its own index files | MIT |

Some notes on each, from their READMEs and docs:

- **Graphiti** tracks how facts change, keeps provenance to source episodes, and supports custom entity types through Pydantic models. Its README contrasts it with GraphRAG: GraphRAG targets "static document summarization" with batch processing, while Graphiti handles "continuous, incremental updates." It works best with LLMs that support structured output. Zep's hosted version is covered in [what is Zep memory](/articles/what-is-zep-memory/).
- **Neo4j Agent Memory** splits memory into short-term (conversation history), long-term (an entity graph of people, places, facts and preferences) and reasoning memory (tool use and decisions). It integrates with LangChain, Pydantic AI, LlamaIndex, CrewAI, OpenAI Agents and others. Its README says it is "actively maintained, but not officially supported" by Neo4j.
- **Cognee** builds "entities, relationships, and searchable chunks" from text, and a graph of symbols and dependencies from code. Its current API centers on `remember`, `recall`, `improve` and `forget`, and it can run locally without an LLM key.
- **The MCP memory server** from the Model Context Protocol project is the simplest graph memory there is: entities with observations, directed relations, and tools like `create_entities`, `add_observations` and `search_nodes`, all saved to a JSONL file.
- **Hindsight** keeps its graph inside Postgres: `retain` extracts facts, time, entities and relationships, and `recall` runs graph traversal next to semantic, BM25 and time-range search, fused and reranked. On top of the graph it consolidates facts into **observations** (beliefs with supporting quotes, refined as evidence arrives), and `reflect` reasons over them. On the long-history BEAM test it scores 73.9 at 1M tokens and 64.1 at 10M, versus Mem0's 64.1 and 48.6 (per [Mem0's own comparison](https://mem0.ai/compare/mem0-vs-hindsight); all vendor-run). The trade-off is the one every graph system has: each write calls an LLM. More in [what Hindsight is](/articles/ai-memory-hindsight/).
- **GraphRAG** ([Edge et al., 2024](https://arxiv.org/abs/2404.16130)) isn't agent memory. It builds a graph and community summaries over a fixed corpus to answer "global" questions like "What are the main themes in the dataset?" Its community idea was later reused in Zep.
- **Mem0** had graph memory in its open-source SDK, but its v3 migration guide says "Graph memory is removed from the open-source SDK" and is now a Mem0 Platform feature.

## Using Graphiti with Neo4j in Python

This follows Graphiti's quickstart (graphiti-core 0.30). It needs a running Neo4j 5.26 instance and an `OPENAI_API_KEY`, since OpenAI is the default for extraction and embeddings.

```python
import asyncio
from datetime import datetime, timezone

from graphiti_core import Graphiti
from graphiti_core.nodes import EpisodeType


async def main() -> None:
    graphiti = Graphiti("bolt://localhost:7687", "neo4j", "password")
    try:
        await graphiti.build_indices_and_constraints()

        await graphiti.add_episode(
            name="support-chat-1",
            episode_body="Dana: I just moved from Acme to Globex, so update my billing email.",
            source=EpisodeType.message,
            source_description="support chat",
            reference_time=datetime(2026, 5, 2, tzinfo=timezone.utc),
            group_id="user-dana",  # one graph partition per user
        )

        edges = await graphiti.search("Where does Dana work?", group_ids=["user-dana"])
        for edge in edges:
            print(edge.fact, edge.valid_at, edge.invalid_at)
    finally:
        await graphiti.close()


asyncio.run(main())
```

Each result is an edge with a natural-language `fact` and its validity window. If a later episode says Dana left Globex, the earlier edge gets an `invalid_at` date instead of being deleted.

## Problems with graph memory

**Extraction is expensive.** Each episode costs several LLM calls for entities, resolution, edges and conflict checks. Write latency is seconds, not milliseconds, so most systems ingest in the background.

**Extraction errors become facts.** A misread sentence produces a wrong edge, and that edge looks as authoritative as a correct one. Keeping links from each fact to its source episode lets you audit and fix it.

**Entity resolution is hard.** Two people with the same first name, or one company with three spellings, can split or merge nodes wrongly. Errors here spread to every connected fact.

**Schemas need thought.** Free-form extraction yields many near-duplicate relationship types. Custom entity and edge types (Graphiti supports them through Pydantic) keep the graph consistent but take design work.

**Small models struggle.** Graphiti's README warns that smaller models may fail ingestion because the pipeline depends on reliable structured output.

## When to use a knowledge graph for agent memory

A graph is worth it when:

- Questions connect several facts ("which of my customers on the old plan reported this bug?").
- Facts change and you need to know what was true when.
- Entities matter more than text: accounts, people, products, tickets.
- You need to explain where an answer came from.

A vector store is enough when:

- Memories are mostly free text, like notes or chat snippets.
- Questions are "find something like this."
- Write cost and latency have to stay low.

Many teams start with vectors and add a graph once relationship questions start failing.


## FAQ

### What is a knowledge graph memory for AI agents?

It's long-term memory stored as entities (people, products, places) connected by typed relationships, usually extracted from conversations by an LLM. The agent retrieves facts by searching the graph and following links, which helps with questions that connect several facts or ask how something changed over time.

### Is a knowledge graph better than a vector database for LLM memory?

Not in general. Graphs do better on multi-hop questions, entity-centric lookups and changing facts. Vector search is cheaper to build and better for fuzzy recall of unstructured text. Most graph memory systems, including Graphiti and Cognee, also store embeddings and combine both.

### How do you use Neo4j for LLM memory?

Common options are Graphiti (Zep's open-source temporal graph engine, which runs on Neo4j 5.26 among other backends) and Neo4j Labs' Agent Memory library, which stores conversations, an entity graph and reasoning traces in Neo4j. Both extract entities with an LLM and search with vectors, keywords and graph traversal.

