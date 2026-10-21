# Zep Docker in 2026: Self-Hosting Options Explained

> Can you run Zep memory with Docker? Zep Community Edition is deprecated. The self-hosting options now: Graphiti server and MCP containers, legacy CE, BYOC.

Source: https://aiagentmemory.org/articles/zep-memory-docker/ · Updated 2026-10-12 · aiagentmemory.org


**You can't self-host today's Zep with Docker.** Zep Community Edition, the Docker-based self-hosted Zep, has been deprecated since April 2025. What you can run in Docker now is **Graphiti**, Zep's open-source temporal graph engine, as a REST server with Neo4j or an MCP server with FalkorDB. The full Zep product runs in your own cloud only on the Enterprise plan.

Many guides still show `docker compose up` for Zep. They describe an edition that gets no updates or support. This page lays out the real options as of 8 October 2026, with working Compose setups, based on the [getzep/zep](https://github.com/getzep/zep) and [Graphiti](https://github.com/getzep/graphiti) repositories.

## What are the Zep self-hosting options in 2026?

**There are three ways to run Zep-style memory on your own infrastructure: Graphiti in Docker (supported, open source, Apache 2.0), the legacy Zep Community Edition Compose stack (deprecated and unsupported), or Zep's Bring Your Own Cloud deployment (Enterprise plan only). Zep Cloud, the main product, is managed.**

| Option | Status | What you get | What you run |
|---|---|---|---|
| **Graphiti REST server** (`zepai/graphiti`) | Supported, Apache 2.0 | Temporal graph: add messages, search facts, delete episodes or groups | Graphiti container + Neo4j, LLM key |
| **Graphiti MCP server** (`zepai/knowledge-graph-mcp`) | Supported, Apache 2.0 | Graph memory for Claude, Cursor and other MCP clients | One container with FalkorDB bundled, or + Neo4j |
| **Graphiti as a library** (`graphiti-core`) | Supported, Apache 2.0 | Full Python API inside your app | Your app + a graph database |
| **Zep Community Edition** (`zepai/zep`) | Deprecated April 2025, unsupported | Old Zep server with users and sessions | Zep + Graphiti 0.3 + Postgres containers |
| **Zep BYOC** | Enterprise plan | The real Zep, inside your VPC | Arranged with Zep |
| **Hindsight** (not Zep) | Supported, MIT | Full memory server: graph and time-range recall, consolidation, `reflect`, UI | One container with embedded Postgres, LLM key |

## What happened to Zep Community Edition

On 2 April 2025, Zep [announced](https://www.getzep.com/blog/announcing-a-new-direction-for-zeps-open-source-strategy/) it had "decided to stop maintaining and releasing Zep Community Edition." The code stayed public under Apache 2.0, but with no updates or support. Zep's open-source work moved to Graphiti.

Today the getzep/zep README opens with "This repository is **not** Zep's product or service." It holds examples, framework integrations and benchmarks for Zep Cloud. The old server lives in `legacy/`, labeled "Deprecated Zep Community Edition (unsupported)."

That folder still has `docker-compose.ce.yaml`. It starts the `zepai/zep:latest` image, a pinned `zepai/graphiti:0.3` image and Postgres with pgvector (`ankane/pgvector:v0.5.1`). It may still start, but it's frozen at an old Graphiti and gets no security fixes. The CE API also differs from Zep Cloud's v3 SDK, which is built around users and threads. **Don't start a new project on it.**

## Option 1: Graphiti REST server with Neo4j

The Graphiti repo includes a FastAPI service, published to Docker Hub as `zepai/graphiti` with tags that match each `graphiti-core` release (0.30.2 as of September 2026). It needs a Neo4j database and an LLM key.

This Compose file follows the [server README](https://github.com/getzep/graphiti/tree/main/server) and the repo's own `docker-compose.yml`:

```yaml
services:
  graph:
    image: zepai/graphiti:latest
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - NEO4J_URI=bolt://neo4j:7687
      - NEO4J_USER=neo4j
      - NEO4J_PASSWORD=${NEO4J_PASSWORD}
    depends_on:
      - neo4j
  neo4j:
    image: neo4j:5.26.2
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      - NEO4J_AUTH=neo4j/${NEO4J_PASSWORD}
    volumes:
      - neo4j_data:/data
volumes:
  neo4j_data:
```

Steps to run it:

1. Save the file as `docker-compose.yml` and create a `.env` with `OPENAI_API_KEY` and `NEO4J_PASSWORD`.
2. Run `docker compose up -d`.
3. Open `http://localhost:8000/docs` for the Swagger UI.
4. Open `http://localhost:7474` to browse the graph in Neo4j.
5. Send messages to `POST /messages` and query with `POST /search`.

A `group_id` plays the role of a Zep user: each group is a separate graph partition. This Python client adds a conversation and searches it:

```python
import time
import httpx

api = httpx.Client(base_url="http://localhost:8000", timeout=30)

api.post("/messages", json={
    "group_id": "user-dana",
    "messages": [
        {"content": "I moved to Lisbon last month.", "role_type": "user", "role": "Dana"},
        {"content": "Noted, I'll use your Lisbon address.", "role_type": "assistant", "role": "bot"},
    ],
}).raise_for_status()  # 202: processed by a background worker

time.sleep(20)  # extraction runs LLM calls asynchronously

result = api.post("/search", json={
    "group_ids": ["user-dana"],
    "query": "Where does Dana live?",
    "max_facts": 5,
}).json()

for fact in result["facts"]:
    print(fact["fact"], fact["valid_at"], fact["invalid_at"])
```

Each fact comes back with `valid_at` and `invalid_at`, the validity window that makes Graphiti temporal. When Dana later says she moved again, the Lisbon fact gets an `invalid_at` instead of disappearing. For why that matters, see [temporal reasoning in AI memory](/articles/temporal-reasoning-ai-memory/).

The server also has `POST /get-memory` (facts relevant to a list of messages), `GET /episodes/{group_id}`, and deletes for edges, episodes and whole groups.

## Option 2: Graphiti MCP server in Docker

If the goal is memory for Claude Desktop, Cursor or another MCP client, use the Graphiti MCP server. A pre-built image is published as `zepai/knowledge-graph-mcp`. Its default setup runs **FalkorDB and the MCP server in a single container**:

```bash
git clone https://github.com/getzep/graphiti.git
cd graphiti/mcp_server
cp .env.example .env      # set OPENAI_API_KEY (or another provider's key) here
cd docker && docker compose up
```

The Compose file reads keys from `mcp_server/.env`, so set them there rather than only exporting them in your shell.

That exposes the MCP endpoint at `http://localhost:8000/mcp/` over HTTP, FalkorDB on port 6379, and FalkorDB's web UI on port 3000. A Neo4j variant is in `docker-compose-neo4j.yml` in the same folder. Point an HTTP-capable MCP client at the endpoint; stdio-only clients can run the server directly with `uv`, per the [MCP server README](https://github.com/getzep/graphiti/tree/main/mcp_server).

The MCP server supports OpenAI, Anthropic, Gemini, Groq and Azure OpenAI for the LLM, and OpenAI, Voyage, Sentence Transformers and Gemini for embeddings, configured in `config.yaml`. More on this pattern in [AI memory MCP servers](/articles/ai-memory-mcp-server/).

## Running Zep-style memory fully local

"Local" usually means no cloud LLM. Graphiti defaults to OpenAI for both extraction and embeddings, but its README says other providers, including local servers such as **Ollama, vLLM, llama.cpp and LM Studio**, work through their OpenAI-compatible endpoints. It also warns that Graphiti "works best with LLM services that support Structured Output." Small local models often fail to produce valid entity and edge JSON, so test extraction quality before you commit.

Graph database choices for Graphiti 0.30: Neo4j 5.26, FalkorDB 1.1.2, or Amazon Neptune. Kuzu support is deprecated because the upstream Kuzu project is no longer maintained. FalkorDB also has an embedded "lite" install extra (`graphiti-core[falkordblite]`) for single-process setups.

## Option 3: a full memory server without a graph database

If what you miss from Community Edition is a complete server (per-user scoping, retrieval, a UI) rather than Zep's exact graph model, **[Hindsight](https://github.com/vectorize-io/hindsight)** is an MIT-licensed alternative that runs in one container. Its README's quick start:

```bash
docker run -it --pull always --name hindsight --restart unless-stopped -p 8888:8888 -p 9999:9999 \
  -e HINDSIGHT_API_LLM_API_KEY=$OPENAI_API_KEY \
  -v hindsight-data:/home/hindsight/.pg0 \
  ghcr.io/vectorize-io/hindsight:latest
```

The API comes up on port 8888 and a UI on 9999. Storage is an embedded PostgreSQL in the volume, or point it at your own Postgres with pgvector. Memory banks play the role of Zep users. Recall runs semantic, BM25, graph and time-range searches in parallel, and every feature, including consolidation into observations and `reflect`, is in the open-source release. It supports 25+ LLM providers, including fully local Ollama, LM Studio and llama.cpp.

The tradeoffs: facts don't carry Zep-style `valid_at`/`invalid_at` windows, writes cost more because retain and consolidation both call an LLM, and the README says it "may be overkill" for simple n8n-style flows. More in [what Hindsight is](/articles/ai-memory-hindsight/).

## Graphiti vs the Zep you'd get in the cloud

Self-hosted Graphiti is the engine, not the product. The Graphiti README is explicit about the split:

| | Zep Cloud | Self-hosted Graphiti |
|---|---|---|
| Users and threads | Built in | Use `group_id`; build the rest |
| Context Block (prompt-ready summary) | Built in | Build your own from search results |
| Graph database | Zep's managed engine | You run Neo4j, FalkorDB or Neptune |
| Dashboard, API logs | Yes | Neo4j Browser or FalkorDB UI only |
| Framework packages (LangGraph, CrewAI, ADK...) | Yes | No; call the REST API or library |
| Cost | Credits per ingested data (from $125/month for Flex) | Your infrastructure plus LLM and embedding calls |

If you were on Community Edition and want the closest thing to what you had, Graphiti plus a thin layer for users and sessions is the honest answer. If you'd rather not run a graph database, other self-hostable memory servers exist: Hindsight (above) and Mem0's Docker server both run on Postgres with pgvector, and Cognee runs with local file-based stores. These are compared in [Zep alternatives](/articles/zep-alternatives/), and Zep's own product is explained in [what is Zep memory](/articles/what-is-zep-memory/).


## FAQ

### Can I still self-host Zep with Docker?

Not the current Zep product. Zep stopped maintaining Zep Community Edition in April 2025, and its Docker Compose files sit unsupported in the legacy folder of getzep/zep. The supported self-hosted path is Graphiti, Zep's open-source graph engine, which ships Docker images for a REST server and an MCP server. Full Zep in your own cloud is an Enterprise BYOC option.

### What is the Docker image for Graphiti?

The Graphiti REST server is published as zepai/graphiti on Docker Hub and needs a Neo4j database. The Graphiti MCP server is published as zepai/knowledge-graph-mcp and by default bundles FalkorDB in the same container. Both need an LLM key, OpenAI by default.

### Is Graphiti the same as self-hosted Zep?

No. Graphiti is the temporal knowledge graph engine Zep is built on, but it doesn't include Zep's users, threads, Context Block assembly, dashboard or managed graph engine. You get the graph and its search; you build user and conversation management yourself.

