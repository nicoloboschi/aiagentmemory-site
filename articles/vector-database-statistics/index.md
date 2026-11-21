# Vector Database Statistics (November 2026)

> GitHub stars, forks, licenses and monthly npm and PyPI downloads for 8 vector databases: Milvus, FAISS, Qdrant, Chroma, pgvector, Weaviate, LanceDB and Pinecone. Collected 2026-11-21.

Source: https://aiagentmemory.org/articles/vector-database-statistics/ · Updated 2026-11-21 · aiagentmemory.org


On 2026-11-21, Milvus had the most GitHub stars of the 8 vector databases we checked: 46,711. The pgvector Python package had the most PyPI downloads: 43,629,382 in the last month. Pinecone's npm package had the most npm downloads: 4,830,781. Downloads count installs of a client package, not servers running in production.

These numbers were pulled by aiagentmemory.org on 2026-11-21 from the GitHub, PyPI and npm APIs. The table is the script's output, pasted without edits. The exact calls are in the [Methodology](#methodology) section. For how these databases fit into agent memory, read [best in-memory vector databases](/articles/best-in-memory-vector-database/) and [vector databases for LLM memory](/articles/vector-database-for-llm-memory/).

## What is a vector database?

A vector database stores numbers that stand for the meaning of text, called embeddings, and finds the closest ones to a query. Agents use it to search old notes or facts by meaning. Some of these projects are servers, some are libraries that run in your program, and one (pgvector) is an extension for PostgreSQL.

## Vector database GitHub and package statistics

Collected 2026-11-21 07:33 UTC from the GitHub REST API, pypistats.org (last month) and api.npmjs.org (last month).

| # | Project | Repository | Stars | Forks | Created | Last push | License | PyPI/month | npm/month |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Milvus | milvus-io/milvus | 46,711 | 4,342 | 2019-09-16 | 2026-11-21 | Apache-2.0 | 3,609,687 | 973,839 |
| 2 | FAISS | facebookresearch/faiss | 41,840 | 4,658 | 2017-02-07 | 2026-11-20 | MIT | 12,819,715 | - |
| 3 | Qdrant | qdrant/qdrant | 35,652 | 2,833 | 2020-05-30 | 2026-11-20 | Apache-2.0 | 16,124,922 | 4,124,925 |
| 4 | Chroma | chroma-core/chroma | 29,721 | 2,612 | 2022-10-05 | 2026-11-20 | Apache-2.0 | 8,584,496 | 1,277,244 |
| 5 | pgvector | pgvector/pgvector | 23,758 | 1,402 | 2021-04-20 | 2026-11-20 | NOASSERTION | 43,629,382 | 4,516,155 |
| 6 | Weaviate | weaviate/weaviate | 16,959 | 1,440 | 2016-03-30 | 2026-11-20 | NOASSERTION | 5,120,088 | 2,144,213 |
| 7 | LanceDB | lancedb/lancedb | 11,957 | 1,174 | 2023-02-28 | 2026-11-21 | Apache-2.0 | 6,930,155 | 2,589,651 |
| 8 | Pinecone | pinecone-io/python-sdk | 456 | 140 | 2021-09-16 | 2026-11-03 | Apache-2.0 | 4,580,641 | 4,830,781 |

"NOASSERTION" means GitHub did not detect a license, so check the LICENSE file. Pinecone's row is its open-source Python SDK repository, and its download counts are for the SDK packages. They do not measure the Pinecone service.

## What the numbers show

- **Stars:** Milvus leads, then FAISS and Qdrant. FAISS is a search library with no server, so its stars do not mean a database. Chroma has 29,721.
- **Client downloads:** pgvector's Python package had 43,629,382, and Qdrant's Python client had 16,124,922. Pinecone's npm package had 4,830,781, the most npm downloads in the table.
- **Smallest:** LanceDB has 11,957 stars. Pinecone's SDK repository has 456 stars.
- **Recent activity:** Milvus and LanceDB were last pushed on 2026-11-21. Pinecone's SDK was last pushed on 2026-11-03, the oldest push in the table.

## Where Hindsight fits

Hindsight (https://github.com/vectorize-io/hindsight) is not a vector database, so it is not in the table. It is an agent memory server. Its open-source version keeps memories in PostgreSQL with the pgvector extension, and the server can run its own embedded Postgres, so you do not need a separate database to start (Hindsight README, checked 2026-10-12).

Limits to know:

- The hosted Hindsight Memlake storage (Enterprise only, launched 2026-10-13) keeps memories in object storage with its own vector index. It is not open source. The open-source server stays on Postgres.
- No head-to-head benchmark against Chroma, Qdrant, Milvus, LanceDB or Pinecone has been published by Vectorize. Its own storage numbers are Vectorize-run and are not independent.
- Vectorize says banks under about 50,000 memories are fine on Postgres, and small deployments can be cheaper there.

## Frequently asked questions

**What is the most popular vector database?** By GitHub stars on 2026-11-21, Milvus had 46,711, FAISS had 41,840 and Qdrant had 35,652. Stars show interest, not production use.

**Which vector database has the most downloads?** The pgvector Python package had 43,629,382 PyPI downloads in the last month, the most in the table. Qdrant's Python client had 16,124,922. These are client packages, so they count installs.

**Is Hindsight a vector database?** No. It is an agent memory server. Its open-source version stores memories in PostgreSQL with pgvector, and it is not in the table.

## Methodology

**When.** Collected on 2026-11-21 at 07:33 UTC by aiagentmemory.org. The table is the output of `repo_stats.py`, a script in the site's repository that calls the APIs below and prints a Markdown table. No number was typed by hand.

**Which APIs.**

1. **GitHub:** `GET https://api.github.com/repos/{owner}/{repo}`. Fields used: `stargazers_count`, `forks_count`, `created_at`, `pushed_at`, `license.spdx_id`.
2. **PyPI:** `GET https://pypistats.org/api/packages/{package}/recent`, the last-month total.
3. **npm:** `GET https://api.npmjs.org/downloads/point/last-month/{package}`, the last-month total.

**Which projects.** Chosen by hand from the best-known vector databases and their official packages. This is not a ranking of use.

**Package names.** Each name was checked against its registry entry, which names the source repository:

- `chromadb` (PyPI and npm) points to `chroma-core/chroma`.
- `qdrant-client` (PyPI) and `@qdrant/js-client-rest` (npm) are the official clients of `qdrant/qdrant`.
- `weaviate-client` (PyPI and npm) are the official clients of `weaviate/weaviate`.
- `pymilvus` (PyPI) points to `milvus-io/milvus`. `@zilliz/milvus2-sdk-node` (npm) is the official Node.js SDK for Milvus.
- `pgvector` (PyPI) is the Python support package and `pgvector` (npm) the Node.js one, both for `pgvector/pgvector`.
- `lancedb` (PyPI) and `@lancedb/lancedb` (npm) point to `lancedb/lancedb`.
- `faiss-cpu` (PyPI) points to `facebookresearch/faiss`. FAISS has no npm package in this list.
- `pinecone` (PyPI) and `@pinecone-database/pinecone` (npm) are the official SDKs. The PyPI source is `pinecone-io/python-sdk`. The repository `pinecone-io/pinecone-python-client` redirects to it.

**Not counted.** Downloads from other tools that depend on these packages, and stars gained in the last 30 days.

**Next refresh.** 2026-12-01, with the same script and table. The change since today will be added as a dated section.


## FAQ

### What is the most popular vector database?

By GitHub stars on 2026-11-21, Milvus had the most at 46,711, then FAISS at 41,840 and Qdrant at 35,652. Stars show interest, not production use. Milvus is the largest by stars, but FAISS is a library, not a server.

### Which vector database has the most downloads?

The pgvector Python package had 43,629,382 PyPI downloads in the last month, the most in the table. Qdrant's Python client had 16,124,922. These are client packages, so they count installs, not servers running.

### Is Hindsight a vector database?

No. Hindsight is an agent memory server. Its open-source version stores memories in PostgreSQL with the pgvector extension. It is not in the table, because the table lists vector databases.

