# Embedding Models for RAG and Agent Memory (2026)

> How to choose an embedding model for RAG and agent memory: MTEB scores, dimensions, context length and cost for OpenAI, Gemini, Voyage, Cohere, Qwen3, BGE-M3.

Source: https://aiagentmemory.org/articles/embedding-models-for-rag/ · Updated 2026-10-20 · aiagentmemory.org


**The best embedding model for RAG and agent memory** is the one that ranks the right passages highest on your own data, at a cost and latency you can live with. Public benchmarks like **MTEB** narrow the field. As of October 2026, strong API choices are Voyage 4, Gemini Embedding 2, Cohere Embed and OpenAI text-embedding-3. Strong open-weight choices are Qwen3-Embedding, BGE-M3 and, for on-device use, EmbeddingGemma 2 (released 6 October 2026, Apache-2.0).

This page explains what to compare, lists current models with specs from their own docs, and shows how to test them.

## What is an embedding model?

**An embedding model turns text (and, for some models, images or audio) into a fixed-length vector of numbers so that texts with similar meaning end up close together. RAG systems and agent memory use it twice: once to index stored content, and once per query to find the nearest stored vectors.**

The embedding model decides what "similar" means for your system. If it doesn't place "my flight got cancelled" near "refund for the canceled trip", no vector database or reranker downstream can fix that. That makes it one of the cheapest components to change and one of the most important to get right.

For agent memory specifically, the model embeds extracted facts and past messages that are then searched on every turn. The rest of that pipeline is covered in [AI agent memory explained](/articles/ai-agent-memory-explained/).

## How to read the MTEB leaderboard

The [Massive Text Embedding Benchmark](https://arxiv.org/abs/2210.07316) (Muennighoff et al., 2022) is the standard comparison. The original release spanned 8 task types, 58 datasets and 112 languages. Its main finding still holds: "no particular text embedding method dominates across all tasks."

Four things to keep in mind when reading it:

1. **Look at the retrieval column**, not the overall average. RAG and memory are retrieval tasks; clustering and classification scores matter less.
2. **Check which benchmark version a number comes from.** There are now English, multilingual, code and other variants. OpenAI's 64.6% for text-embedding-3-large and Qwen's 70.58 for Qwen3-Embedding-8B come from different benchmarks and can't be compared directly.
3. **Scores are mostly self-reported** by the model's authors.
4. **Benchmarks aren't your data.** Short user facts, support tickets and code all behave differently from Wikipedia-style passages.

## Embedding models compared

Specs from each provider's docs or model card, checked 8 October 2026. Prices change often, so check the provider's page.

| Model | Provider | Open weights | Max input | Dimensions | Notes |
|---|---|---|---|---|---|
| text-embedding-3-small | OpenAI | No | 8,192 tokens | 1,536 (shortenable) | ~62,500 pages per dollar; MTEB 62.3% (OpenAI) |
| text-embedding-3-large | OpenAI | No | 8,192 tokens | 3,072 (shortenable) | ~9,615 pages per dollar; MTEB 64.6% (OpenAI) |
| gemini-embedding-2 | Google | No | 8,192 tokens | 128-3,072 (768, 1,536, 3,072 recommended) | Text, images, video, audio, PDF in one space |
| gemini-embedding-001 | Google | No | 2,048 tokens | 128-3,072 | Text only; `task_type` parameter |
| voyage-4-large / voyage-4 / voyage-4-lite | Voyage AI | No | 32,000 tokens | 1,024 default; 256, 512, 2,048 | int8 and binary output; the three are compatible |
| voyage-4-nano | Voyage AI | Yes | 32,000 tokens | 1,024 default; 256, 512, 2,048 | Open-weight member of the family |
| embed-v4.0 | Cohere | No | 128k tokens | 256-1,536 | Text and images |
| Qwen3-Embedding (0.6B / 4B / 8B) | Alibaba Qwen | Yes, Apache-2.0 | 32K tokens | Up to 1,024 / 2,560 / 4,096; 32 and up | 100+ languages; 8B scored 70.58 on MTEB multilingual (June 2025) |
| BGE-M3 | BAAI | Yes, MIT | 8,192 tokens | 1,024 | Dense, sparse and multi-vector output from one model |
| EmbeddingGemma | Google | Yes, Gemma license | 2,048 tokens | 768; 512, 256, 128 | 300M parameters, built for on-device use |
| EmbeddingGemma 2 | Google | Yes, Apache-2.0 | 8,192 tokens | 768; 512, 256, 128 | Released 6 Oct 2026; 740M parameters (270M text-only); text, code, images, video, audio |

Sources: [OpenAI embeddings guide](https://developers.openai.com/api/docs/guides/embeddings), [Gemini embeddings docs](https://ai.google.dev/gemini-api/docs/embeddings), [Voyage docs](https://docs.voyageai.com/docs/embeddings), [Cohere Embed docs](https://docs.cohere.com/docs/cohere-embed), and the Hugging Face model cards for [Qwen3-Embedding-8B](https://huggingface.co/Qwen/Qwen3-Embedding-8B), [BGE-M3](https://huggingface.co/BAAI/bge-m3) and [EmbeddingGemma](https://huggingface.co/google/embeddinggemma-300m). Cohere's docs also list newer `embed-v5.0-pro` and `embed-v5.0-fast` models with 128k context and up to 2,048 dimensions.

**EmbeddingGemma 2 is the newest open option, released by Google on 6 October 2026** ([announcement](https://blog.google/innovation-and-ai/technology/developers-tools/embeddinggemma-2/), [model card](https://huggingface.co/google/embeddinggemma-2)). Google's model card reports MTEB English v2 68.46, Multilingual v2 61.36 and Code 78.68 at 768 dimensions; the first EmbeddingGemma scored 61.15 multilingual and 68.76 on code. These are Google's own numbers. Google says the text-only weights need about 191MB of active RAM on a Pixel 11 Pro with quantization, which makes it a fit for local or on-device memory where nothing should leave the machine. Because its context grew to 8K tokens and the license moved to Apache-2.0, it removes the two main limits of the first version.

## What to compare when choosing

### Retrieval quality on your data

This is the only number that really matters, and you have to measure it yourself. See the evaluation steps at the end of this page.

### Dimensions and storage

Every dimension costs 4 bytes per vector in float32. A million 3,072-dimension vectors is about 12 GB before index overhead. Most current models are trained so vectors can be cut short (**Matryoshka representation learning**). OpenAI's guide says a text-embedding-3-large vector "can be shortened to a size of 256" and still beats unshortened ada-002 at 1,536. Note that truncated vectors need re-normalizing unless the API does it; Google says `gemini-embedding-2` normalizes automatically and `gemini-embedding-001` doesn't below 3,072.

Quantized output cuts storage further. Voyage's 4-series returns `int8`, `uint8`, `binary` and `ubinary` vectors directly.

### Context length

Long limits (32K for Voyage 4 and Qwen3, 128k for Cohere) let you embed whole documents. That isn't always a good idea: one vector for a 20-page document blurs its topics. For RAG, chunks of a few hundred tokens usually retrieve better. For memory, the units are already short facts or messages, so context length rarely matters.

### Query and document prompts

Many models embed queries and documents differently. Qwen3-Embedding is "instruction-aware" and expects a query prompt. `gemini-embedding-001` takes a `task_type` such as `RETRIEVAL_QUERY` or `RETRIEVAL_DOCUMENT`, while `gemini-embedding-2` takes the task as a text prefix. Skipping these usually costs accuracy.

### Hybrid search

Embeddings handle meaning but miss exact strings like order IDs and product names. **BGE-M3** returns dense, sparse (lexical weights) and multi-vector (ColBERT-style) outputs from one model, which covers keyword and semantic matching together. Otherwise, pair a dense model with BM25 in your database.

### Hosting

APIs mean no infrastructure but a network call per query and data leaving your system. Open-weight models run locally. That's what people usually mean by an **in-memory embedding model**: one loaded into the application's own process, with no API call. EmbeddingGemma (300M parameters) and Qwen3-Embedding-0.6B are small enough for a laptop CPU. Pairing one with an [in-memory vector database](/articles/best-in-memory-vector-database/) gives a fully local memory stack.

## Embedding models for agent memory

Memory differs from document RAG in ways that affect the model choice:

- **Units are short.** "User is vegetarian" is five words. Pick a model that does well on short-text retrieval, and test with real memory items.
- **Queries and memories look different.** The query is a new user message; the memory is a terse fact. Asymmetric query/document prompts help here.
- **Everything is re-embedded on a switch.** A memory store grows for years. Changing models later means re-embedding all of it, and Google's docs state that `gemini-embedding-001` and `gemini-embedding-2` spaces are "incompatible."
- **Embeddings don't know time.** "I moved to SF" and "I live in NYC" are both close to "where do I live?" Handling changed facts is the job of the memory system, not the embedding model. [Hindsight](https://github.com/vectorize-io/hindsight), for example, treats the embedding as one of four recall strategies: it runs semantic, BM25 keyword, entity-graph and time-range search in parallel, fuses them and reranks with a cross-encoder, so a weak embedding match on an ID or a date can still be found. The cost is an LLM call on every write to extract entities and timestamps. See [vector databases for LLM memory](/articles/vector-database-for-llm-memory/) for what a vector store does and doesn't cover.

## Code: local and API embeddings

A local open-weight model with sentence-transformers (6.x), using the model's query prompt and Matryoshka truncation:

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("Qwen/Qwen3-Embedding-0.6B")

memories = ["User is vegetarian", "User's flight to Tokyo is on 12 May", "User prefers short answers"]
query = "Suggest a restaurant for dinner"

doc_vecs = model.encode_document(memories, truncate_dim=256)
query_vec = model.encode_query(query, truncate_dim=256)  # applies the model's "query" prompt

print(model.similarity(query_vec, doc_vecs))  # cosine similarity, one row per query
```

The same with OpenAI's API, asking for shorter vectors at creation time:

```python
from openai import OpenAI

client = OpenAI()
response = client.embeddings.create(
    model="text-embedding-3-small",
    input=["User is vegetarian", "Suggest a restaurant for dinner"],
    dimensions=512,
)
vectors = [item.embedding for item in response.data]
```

## How to evaluate embedding models on your data

1. **Collect 50-200 real queries** from logs or users, not invented ones.
2. **Label the right answers.** For each query, mark which stored chunks or memories should come back.
3. **Pick two to four candidates** from the table, including one cheap baseline.
4. **Embed the same corpus with each**, using the right query/document prompts.
5. **Measure recall@k** (is a correct item in the top k?) for the k you'll actually use, often 5 or 10.
6. **Check cost and latency** per 1,000 queries and for indexing the full corpus.
7. **Try truncated dimensions** on the winner to see how much quality you lose for the storage you save.

Retrieval quality also depends on how text is chunked and stored. How memory and document retrieval differ is covered in [RAG vs agent memory](/articles/rag-vs-agent-memory/).


## FAQ

### What is the best embedding model for RAG?

There's no single best one. MTEB's authors found no method dominates every task, and leaderboard ranks change monthly. Strong 2026 choices include Voyage 4, Gemini Embedding 2, Cohere Embed and OpenAI text-embedding-3 as APIs, and Qwen3-Embedding or BGE-M3 as open weights. Test two or three on your own data.

### How many dimensions should an embedding have?

Usually 768 to 1,536 is enough for RAG and memory. Many current models support Matryoshka-style truncation, so you can store 256 or 512 dimensions to save space. OpenAI reports text-embedding-3-large cut to 256 dimensions still beats ada-002 at 1,536 on MTEB.

### Can I switch embedding models later?

Yes, but you have to re-embed everything. Vectors from different models live in different spaces and can't be compared; Google's docs say Gemini Embedding 001 and 2 spaces are incompatible, for example. Store the model name and version with each vector so a migration is planned, not discovered.

