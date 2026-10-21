# Best In-Memory Vector Databases: FAISS, Chroma, More

> The best in-memory vector databases by language: FAISS, hnswlib, USearch, Chroma, Qdrant local mode, LangChain and Spring AI stores, with Python code.

Source: https://aiagentmemory.org/articles/best-in-memory-vector-database/ · Updated 2026-10-12 · aiagentmemory.org


**An in-memory vector database** keeps embeddings and their search index in RAM, usually inside your application's process, so there's no server to deploy and lookups are very fast. The best choice depends on what you need around the search. For raw speed in Python or C++, use **FAISS** or **USearch**. For a database-style API with metadata, use **Chroma**'s in-memory client or **Qdrant**'s local mode. For framework prototypes, LangChain, LangChain4j and Spring AI each ship a simple in-memory store.

This page compares the main open-source options by language, shows working Python code, and explains when in-memory storage is the right call.

## What is an in-memory vector database?

**An in-memory vector database is a store that holds vectors and their index in RAM and answers nearest-neighbor queries without disk reads or network calls. Most run embedded in the application. They trade durability and scale for speed and zero setup, and many can save to a file and reload.**

The term covers two different kinds of tool:

- **Vector search libraries** (FAISS, hnswlib, USearch) do one thing: index vectors and find the nearest ones. You store text and metadata elsewhere.
- **Embedded vector stores** (Chroma, Qdrant local mode, LangChain's stores) add documents, metadata filters and a collection API on top.

A third group, **in-memory database servers** like Redis, keep data in RAM but run as a separate service. They're covered in [vector databases for LLM memory](/articles/vector-database-for-llm-memory/).

## In-memory vector databases compared

Facts from each project's README and docs, checked 8 October 2026.

| Tool | Kind | Index | Languages | Persistence | License |
|---|---|---|---|---|---|
| [FAISS](https://github.com/facebookresearch/faiss) | Library | Flat, IVF, PQ, HNSW, NSG; GPU support | C++, Python | `write_index` / `read_index` | MIT |
| [hnswlib](https://github.com/nmslib/hnswlib) | Library | HNSW | Header-only C++, Python | Save/load index file | Apache-2.0 |
| [USearch](https://github.com/unum-cloud/usearch) | Library | HNSW | C++, C, Python, JavaScript, Java, Rust, Go, Swift, C# and more | Save, load, or memory-map a file | Apache-2.0 |
| [Chroma](https://github.com/chroma-core/chroma) | Embedded store | HNSW (local) | Python (in-process); JS needs a server | `PersistentClient` writes to disk | Apache-2.0 |
| [Qdrant](https://github.com/qdrant/qdrant-client) local mode | Embedded store | Same API as server | Python client | `path=` persists to disk | Apache-2.0 |
| LangChain `InMemoryVectorStore` | Framework store | Exact cosine over all vectors | Python (JS: `MemoryVectorStore`) | Python: `dump` / `load` to JSON | MIT |
| LangChain4j `InMemoryEmbeddingStore` | Framework store | Brute force | Java | `serializeToFile` / `fromFile` | Apache-2.0 |
| Spring AI `SimpleVectorStore` | Framework store | Cosine over all vectors | Java | `save(File)` / `load(File)` to JSON | Apache-2.0 |

Notes that matter when choosing:

- **FAISS** is "a library for efficient similarity search and clustering of dense vectors," developed mainly at Meta's FAIR lab. Its compressed indexes (quantization codes) "can scale to billions of vectors in memory." It doesn't store text or metadata.
- **USearch** has the widest language coverage. Its README notes that USearch and FAISS "both employ the same HNSW algorithm." An index can be served from a memory-mapped file with `Index.restore(path, view=True)`, which avoids loading it all into RAM.
- **Qdrant**'s client README says local mode lets you "run same code in local mode without running Qdrant server" and is "useful for development, prototyping and testing." Switching to a server later means changing one line.
- **The framework stores** are explicitly for demos. Spring AI's source says `SimpleVectorStore` "is not designed for production use." LangChain.js docs say `MemoryVectorStore` is ephemeral and doesn't support IDs or deletion.

## Best in-memory vector store by language

### Python

Use **FAISS** or **USearch** when you need speed on large sets and manage metadata yourself. Use **Chroma** (`chromadb.EphemeralClient()`) or **Qdrant** (`QdrantClient(":memory:")`) when you want filters and stored documents. Inside LangChain, `InMemoryVectorStore` from `langchain_core.vectorstores` is the zero-setup option; its source computes cosine similarity against every stored vector, so it slows down linearly as data grows.

### JavaScript and Node.js

**USearch** ships a JavaScript binding. In LangChain.js, `MemoryVectorStore` (imported from `@langchain/classic/vectorstores/memory`) "does an exact, linear search." Chroma's JS client connects to a running Chroma server rather than running in-process.

### Java and Spring AI

**Spring AI**'s `SimpleVectorStore` keeps vectors in a `ConcurrentHashMap`, searches with cosine similarity over all of them, and saves to or loads from a JSON file. **LangChain4j**'s `InMemoryEmbeddingStore` works the same way, with `serializeToFile` and `fromFile`. For an approximate index in Java, USearch has Java bindings.

### Rust, C and C++

**USearch** is written in C++ and has C and Rust bindings. **hnswlib** is a header-only C++ library. **FAISS** is C++ at its core. For an embedded store with SQL, `sqlite-vec` is a SQLite extension with Python, Node.js, Ruby, Go and Rust packages; its README says it's pre-v1 and to "expect breaking changes."

## In-memory vector search in Python

These three snippets run as-is (tested with faiss-cpu 1.15, chromadb 1.5 and usearch 2.26). The first two show the library route and the store route.

**FAISS with an HNSW index:**

```python
import faiss
import numpy as np

d = 384
vectors = np.random.default_rng(0).random((10_000, d), dtype=np.float32)
faiss.normalize_L2(vectors)  # on unit vectors, inner product = cosine similarity

index = faiss.IndexIDMap(faiss.IndexHNSWFlat(d, 32, faiss.METRIC_INNER_PRODUCT))
index.add_with_ids(vectors, np.arange(10_000, dtype=np.int64))

scores, ids = index.search(vectors[:1], 5)  # top 5 neighbors of the first vector
print(ids[0], scores[0])

faiss.write_index(index, "memories.faiss")  # reload later with faiss.read_index()
```

`32` is the HNSW graph's neighbors per node (M). `IndexIDMap` lets you use your own integer IDs, which you map to text in a dict or table.

**Chroma's in-memory client with metadata filters:**

```python
import chromadb

client = chromadb.EphemeralClient()  # nothing written to disk
memories = client.create_collection("memories", configuration={"hnsw": {"space": "cosine"}})

memories.add(
    ids=["m1", "m2"],
    documents=["User prefers window seats", "User is allergic to peanuts"],
    metadatas=[{"user_id": "u1"}, {"user_id": "u1"}],
)
result = memories.query(query_texts=["seat preference"], n_results=1, where={"user_id": "u1"})
print(result["documents"])
```

Chroma embeds the documents with its default local model (downloaded on first use) unless you pass your own embedding function. Replace `EphemeralClient()` with `PersistentClient(path="./chroma")` to keep data between runs.

**USearch:**

```python
import numpy as np
from usearch.index import Index

vectors = np.random.default_rng(0).random((10_000, 384), dtype=np.float32)
index = Index(ndim=384, metric="cos")
index.add(np.arange(10_000), vectors)

matches = index.search(vectors[0], 5)
print(matches.keys, matches.distances)
index.save("memories.usearch")
```

## How much RAM do you need?

The raw vectors are easy to estimate: **count × dimensions × 4 bytes** for float32.

| Vectors | 384 dims | 768 dims | 1,536 dims |
|---|---|---|---|
| 100,000 | 154 MB | 307 MB | 614 MB |
| 1 million | 1.5 GB | 3.1 GB | 6.1 GB |
| 10 million | 15 GB | 31 GB | 61 GB |

An HNSW graph adds neighbor lists on top, growing with M. If that doesn't fit, you have three levers: a smaller embedding model or truncated dimensions, half precision or int8 vectors, or compressed indexes like FAISS's IVF-PQ. Choosing a model with the right size is covered in [embedding models for RAG](/articles/embedding-models-for-rag/).

## When to use an in-memory vector database

In-memory stores are a good fit when:

- **You're prototyping or testing.** Tests that need vector search shouldn't depend on a running server.
- **Data is small or rebuildable.** A few hundred thousand vectors, rebuilt from a source of truth at startup.
- **Latency must be minimal.** No network hop, no disk read.
- **The memory is short-lived.** A single agent session's scratch memory, discarded when the session ends.
- **The app runs on a device.** Desktop apps, edge devices, notebooks.

They're a poor fit when:

- **Several processes or servers share the data.** Each process has its own copy.
- **Data must survive crashes.** Saving to a file now and then isn't the same as durable writes.
- **You need multi-user isolation and deletes.** Libraries have no access control, and some stores don't support deletion.
- **Data outgrows RAM.**

For agent long-term memory, which must persist across sessions and users, an in-memory store usually serves as a cache or a dev setup in front of a durable database. Memory servers take the durable route: [Hindsight](https://github.com/vectorize-io/hindsight), for example, stores memories in PostgreSQL with pgvector and offers an embedded Postgres for local setups, so you get zero-setup development without losing data on exit. How long-term memory is built overall is explained in [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What is an in-memory vector database?

It's a vector store that keeps embeddings and its search index in RAM, usually inside your application's process, so there's no server to run and queries take microseconds to milliseconds. Data is lost when the process exits unless you save it to a file.

### Is FAISS an in-memory vector database?

FAISS is an in-memory vector search library, not a full database. It builds indexes in RAM and searches them very fast, on CPU or GPU, but vectors are identified only by integers. You keep text and metadata yourself, and you save the index to disk with faiss.write_index.

### What is the best in-memory vector store for Python?

For raw speed and control, FAISS or USearch. For a database-like API with metadata filters and document storage, Chroma's in-memory client or Qdrant's local mode (QdrantClient(':memory:')). For quick LangChain prototypes, InMemoryVectorStore, which does exact cosine search over all vectors.

