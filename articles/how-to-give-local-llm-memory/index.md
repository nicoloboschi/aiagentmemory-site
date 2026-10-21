# How to Give a Local LLM Memory (Ollama, Python)

> Local LLMs don't remember between chats. How to add persistent memory offline: Open WebUI, AnythingLLM and MCP options, plus tested Python with Ollama and Mem0.

Source: https://aiagentmemory.org/articles/how-to-give-local-llm-memory/ · Updated 2026-10-12 · aiagentmemory.org


A **local LLM has no memory of its own**. Ollama, LM Studio and llama.cpp process each request and keep nothing. To give a local LLM memory, use a chat app with a memory feature (Open WebUI, AnythingLLM), connect a local MCP memory server, or build it: save facts to SQLite or a local vector store, embed them with a local model, and add the relevant ones to each prompt.

## Does a local LLM have memory?

**Local LLM memory is information your app stores on your machine and puts back into the model's prompt, since the model itself retains nothing between calls.** There are two kinds: the current chat history, which the app resends each turn, and long-term memory, which survives a new chat or a restart.

You can see the statelessness in [Ollama's chat API](https://docs.ollama.com/api/chat): the `messages` field is the "chat history as an array of message objects," and there's no session ID. The client sends the whole conversation every time.

Even within one chat, the history is cut at the context length. Per [Ollama's context length docs](https://docs.ollama.com/context-length), the default depends on VRAM: 4k tokens under 24 GiB, 32k for 24-48 GiB, and 256k at 48 GiB or more. On a typical laptop, that means a long chat silently drops its early turns. You can raise it with `OLLAMA_CONTEXT_LENGTH=64000 ollama serve` or the app's slider, at the cost of more memory. The trade-offs of bigger windows on local models are in [largest context window open-source LLMs](/articles/largest-context-window-llm-open-source/).

A bigger window still empties when the chat ends. Long-term memory needs storage outside the model.

## Local AI apps with long-term memory

If you just want a local chat app that remembers you, these are the options we could confirm in each project's docs (October 2026).

| App | Built-in memory | How it works | Where it's stored |
|---|---|---|---|
| [Open WebUI](https://docs.openwebui.com/features/chat-conversations/memory) | Yes | You add memories in Settings > Personalization > Memory; with native function calling, the model manages them with tools like `add_memory` and `search_memories` | Your Open WebUI database, per user |
| [AnythingLLM](https://docs.anythingllm.com/features/memories) | Yes, off by default | Workspace memories (20 max) and global memories (5 max); optional automatic extraction from idle chats | Your AnythingLLM instance |
| Jan | Not yet | Homepage lists Memory as "Coming Soon"; assistants give fixed instructions only | n/a; can connect MCP servers |
| LM Studio | No built-in feature found | Search across chats (0.3.27); MCP host since 0.3.17; community memory plugins on the LM Studio Hub | Depends on the plugin or MCP server |

Two caveats from the docs themselves. Open WebUI warns that memory quality "depends heavily on the model" and that "only very small models struggle with appropriate memory selection." AnythingLLM's automatic extraction needs a chat model that supports tool calling. A 1-3B model may store junk or nothing.

**MCP memory servers** work with any app that can host MCP, including Jan and LM Studio. The reference [Knowledge Graph Memory Server](https://github.com/modelcontextprotocol/servers/tree/main/src/memory) stores entities, relations and observations in a local JSONL file you can point anywhere with `MEMORY_FILE_PATH`. More options are in [AI memory MCP servers](/articles/ai-memory-mcp-server/).

## How to give a local LLM memory in your own app

1. **Run a chat model and an embedding model.** For example `ollama pull llama3.2` and `ollama pull nomic-embed-text`.
2. **Pick a local store.** SQLite is enough for one user; Chroma, Qdrant (local mode) or Postgres with pgvector scale further.
3. **Save after each turn.** Store what the user said, or extract short facts with the local model.
4. **Embed and search before each reply.** Find the few memories closest to the new message.
5. **Inject them into the system prompt.** Keep the block small; small models get confused by long prompts.
6. **Keep recent turns too.** Send the last few messages for the flow of the current chat.
7. **Add deletion.** A local store is still personal data.

## Example 1: local memory from scratch with Ollama and SQLite

No memory library, no cloud. This stores every user message with a local embedding, then pulls the three closest ones into the system prompt. Tested with the `ollama` Python package 0.6 against Ollama 0.34:

```python
import json, math, sqlite3
import ollama

CHAT_MODEL = "llama3.2"
EMBED_MODEL = "nomic-embed-text"

db = sqlite3.connect("memory.db")
db.execute("CREATE TABLE IF NOT EXISTS memories (user_id TEXT, text TEXT, vec TEXT)")

def embed(text: str) -> list[float]:
    return ollama.embed(model=EMBED_MODEL, input=text).embeddings[0]

def cosine(a: list[float], b: list[float]) -> float:
    dot = sum(x * y for x, y in zip(a, b))
    return dot / (math.sqrt(sum(x * x for x in a)) * math.sqrt(sum(y * y for y in b)))

def remember(user_id: str, text: str) -> None:
    db.execute("INSERT INTO memories VALUES (?, ?, ?)", (user_id, text, json.dumps(embed(text))))
    db.commit()

def recall(user_id: str, query: str, k: int = 3) -> list[str]:
    q = embed(query)
    rows = db.execute("SELECT text, vec FROM memories WHERE user_id = ?", (user_id,))
    scored = sorted(((cosine(q, json.loads(v)), t) for t, v in rows), reverse=True)
    return [t for _, t in scored[:k]]

def chat(user_id: str, message: str, history: list[dict]) -> str:
    memories = "\n".join(f"- {m}" for m in recall(user_id, message))
    system = {"role": "system", "content": f"Things you remember about this user:\n{memories or '- nothing yet'}"}
    reply = ollama.chat(model=CHAT_MODEL, messages=[system, *history[-10:], {"role": "user", "content": message}])
    text = reply.message.content
    history += [{"role": "user", "content": message}, {"role": "assistant", "content": text}]
    remember(user_id, message)  # store what the user said for later sessions
    return text

print(chat("ana", "I'm allergic to peanuts and I live in Lisbon.", []))
# New session: empty history, but the fact comes back from SQLite.
print(chat("ana", "Suggest a snack for my flight home.", []))
```

In our run, even a 1.5B model (`qwen2.5:1.5b-instruct`) answered the second, fresh chat with peanut-free snack ideas. The file `memory.db` is the whole memory; copy it, inspect it with `sqlite3`, or delete a user's rows.

One gotcha: embeddings need an embedding model. Asking a chat model for embeddings on current Ollama returned "This server does not support embeddings" in our test, so pull `nomic-embed-text` or another embedding model. For other small local options, such as EmbeddingGemma 2 (Apache-2.0, released October 2026), see [embedding models for RAG and agent memory](/articles/embedding-models-for-rag/).

This stores raw messages, so it grows forever and never updates a changed fact. That's fine to start; for cleaner memories, use a library that extracts facts.

## Example 2: Mem0 fully local with Ollama

[Mem0](https://github.com/mem0ai/mem0) can use Ollama for both fact extraction and embeddings, with Qdrant running embedded on disk. Nothing leaves your machine. This config follows Mem0's own [self-hosted companion cookbook](https://docs.mem0.ai/cookbooks/companions/local-companion-ollama), switched to a local Qdrant path. Tested with `mem0ai` 2.2.1:

```python
import os
os.environ["MEM0_TELEMETRY"] = "False"  # mem0 sends anonymous usage data unless disabled
from mem0 import Memory

config = {
    "llm": {"provider": "ollama", "config": {"model": "llama3.1:8b", "temperature": 0,
                                             "ollama_base_url": "http://localhost:11434"}},
    "embedder": {"provider": "ollama", "config": {"model": "nomic-embed-text",
                                                  "ollama_base_url": "http://localhost:11434"}},
    "vector_store": {"provider": "qdrant", "config": {"collection_name": "memories", "path": "./qdrant_data",
                                                      "on_disk": True, "embedding_model_dims": 768}},
    "history_db_path": "./mem0_history.db",
}
memory = Memory.from_config(config)

memory.add([{"role": "user", "content": "I'm vegetarian and I run Debian on my laptop."}], user_id="ana")
hits = memory.search("What should I cook for dinner?", filters={"user_id": "ana"}, top_k=3)
for h in hits["results"]:
    print(round(h["score"], 2), h["memory"])
```

Our run (with `qwen2.5:1.5b-instruct` as the LLM) printed `0.43 User is vegetarian and runs Debian on their laptop`: Mem0 had the local model rewrite the message as a fact before storing it. Notes from testing:

- `embedding_model_dims` must match the embedder (768 for `nomic-embed-text`).
- Without `on_disk: True`, local Qdrant keeps vectors in memory only.
- The base install logs that BM25 keyword search is disabled; `pip install "mem0ai[extras]"` adds it, and `mem0ai[nlp]` adds spaCy.

## Other memory systems that run locally

Several agent memory servers accept a local model as their LLM, so you can self-host the whole stack:

- **[Hindsight](https://github.com/vectorize-io/hindsight)** lists `ollama`, `lmstudio` and `llamacpp` as fully local LLM providers. It runs under Docker, pip or embedded in Python, and can use an embedded Postgres (`pg0`), so you don't need a database server. Clients are Python, TypeScript and Go, and its MCP server works with MCP hosts like LM Studio and Jan. It's MIT-licensed with every feature in the open-source release: graph and time-aware search, consolidation of facts into observations, and `reflect`. The local catch: each write and each consolidation pass calls your LLM, so a small model on a laptop does a lot more work than with a plain SQLite store.
- **Mem0's self-hosted server** wraps the library above in a FastAPI service with Postgres and pgvector.
- **Letta** and **Cognee** can also be self-hosted; check each project's docs for current local-model support before you commit.

How they differ is covered in [open-source memory systems compared](/articles/open-source-memory-systems-compared/), and the general build (with hosted models) is in [how to give AI agents memory](/articles/how-to-give-ai-agents-memory/).

## Choosing a model for local memory

Memory puts extra demands on a small model:

- **Extraction needs instruction-following.** Turning chat into clean facts, or deciding when to call `add_memory`, is where tiny models fail first. Both Open WebUI and AnythingLLM say so in their docs.
- **The prompt gets longer.** Memories plus recent history plus the system prompt can exceed a 4k default window. Raise the context length if your hardware allows.
- **Split the jobs.** A small model for chat and a slightly larger one for background extraction is a sensible setup, since extraction isn't latency-sensitive.

For the concepts behind all of this, see [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### Does a local LLM have memory?

No. A local model running in Ollama, LM Studio or llama.cpp keeps nothing between requests. The chat app resends the conversation each time, and once you start a new chat the model knows nothing about the old one unless the app saves facts and adds them back to the prompt.

### What is the best way to add long-term memory to a local AI?

If you use a chat app, turn on its memory feature: Open WebUI and AnythingLLM both have one. If you build your own app, store memories in SQLite or a local vector store with embeddings from a local model such as nomic-embed-text, or run Mem0 with Ollama as its LLM and embedder. Everything then stays on your machine.

### Does Jan have long-term memory?

Not as a finished feature as of October 2026. Jan's homepage lists Memory as Coming Soon. Jan supports MCP servers, so you can connect a local memory server in the meantime.

