# LLM Memory With Obsidian: Karpathy's LLM Wiki Pattern

> How to use Obsidian as LLM memory, based on Andrej Karpathy's LLM Wiki gist: raw sources, an LLM-maintained wiki and a schema file, plus search and MCP tools.

Source: https://aiagentmemory.org/articles/llm-memory-obsidian/ · Updated 2026-10-08 · aiagentmemory.org


**Using Obsidian as LLM memory** means letting an AI agent write and maintain a folder of linked Markdown notes that you browse in Obsidian. The best-known version is **Andrej Karpathy's LLM Wiki**, published as a gist in April 2026. You drop sources into a folder, the agent compiles them into wiki pages, and both of you read the wiki instead of re-searching raw documents.

This page explains what Karpathy actually proposed, how to set it up with Obsidian, the tools that help it scale, and when a memory server fits better. The main source is Karpathy's [LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f), dated April 4, 2026.

## What is Karpathy's LLM Wiki?

**The LLM Wiki is a pattern where an LLM agent incrementally builds and maintains a persistent, interlinked Markdown wiki from source documents you collect. Knowledge is compiled once and kept current, rather than retrieved from raw text on every question. Obsidian is the viewer; the agent does the writing.**

Karpathy's one-line summary in the gist: "Obsidian is the IDE; the LLM is the programmer; the wiki is the codebase." He calls the wiki "a persistent, compounding artifact."

The gist is an "idea file," not code. Karpathy says it's deliberately abstract so you can paste it into your own agent and have it build a version for your domain. That's why many slightly different implementations exist.

## The three layers

| Layer | Who writes it | What it holds |
|---|---|---|
| **Raw sources** | You | Articles, papers, transcripts, images. Immutable; the LLM reads but never edits them |
| **The wiki** | The LLM | Summaries, entity and concept pages, comparisons, an overview. "You read it; the LLM writes it." |
| **The schema** | You and the LLM, over time | A config file like `CLAUDE.md` or `AGENTS.md` that sets structure, conventions and workflows |

Two special files keep it navigable:

- **`index.md`**: a catalog of every page with a link and a one-line summary. The agent reads it first when answering a question. Karpathy says this works well at moderate scale without embedding-based RAG.
- **`log.md`**: an append-only record of ingests, queries and lint passes, with greppable headings like `## [2026-04-02] ingest | Article Title`.

## The three operations

1. **Ingest.** Add one source. The agent reads it, discusses takeaways with you, writes a summary page, updates the index, revises related pages and appends to the log. Karpathy says one source may touch 10 to 15 pages, and he prefers ingesting one at a time.
2. **Query.** Ask a question. The agent reads the index and relevant pages and answers with citations. Good answers get filed back into the wiki, so exploration compounds.
3. **Lint.** Periodically, the agent checks for contradictions, stale claims, orphan pages, missing concepts and missing cross-references.

Karpathy's argument for why this works: knowledge bases fail on maintenance, because "the maintenance burden grows faster than the value." LLMs do the bookkeeping at near-zero cost. He links it to Vannevar Bush's 1945 Memex: "The part he couldn't solve was who does the maintenance. The LLM handles that."

## How to set it up with Obsidian

A minimal setup, following the gist:

1. **Create a vault** and add folders: `raw/` for sources, `wiki/` for pages.
2. **Write a schema file** (`CLAUDE.md` or `AGENTS.md`) at the root: page types, naming, link style (`[[wikilinks]]`), what goes in `index.md` and `log.md`, and the ingest, query and lint steps. Tell the agent never to edit `raw/`.
3. **Clip sources** with the Obsidian Web Clipper extension, which saves web articles as Markdown. Karpathy also binds a hotkey to download attachments so images are stored locally.
4. **Run a coding agent in the vault folder.** Claude Code, Codex or any agent that reads and writes files will work, because the schema file is the same kind of instruction file they already load; see [AI coding agent memory](/articles/ai-coding-agent-memory/). For Codex, name the schema file `AGENTS.md` so it loads on every run ([how Codex loads AGENTS.md](/articles/codex-memory/)).
5. **Browse in Obsidian.** Graph view shows hubs and orphan pages. Dataview can build tables from page frontmatter, and Marp turns pages into slides.
6. **Commit to git.** The wiki is a repo of Markdown, so you get history and can review what the agent changed.

Ingest one source, read what changed, correct the schema, repeat. The schema co-evolves with you.

### A small lint script

The lint step can be partly mechanical. This stdlib Python script finds broken `[[wikilinks]]` and orphan pages that nothing links to, which you can hand to the agent as a to-do list:

```python
import re
import sys
from pathlib import Path

LINK = re.compile(r"\[\[([^\]|#]+)")

def lint(wiki_dir: str) -> None:
    pages = {p.stem: p for p in Path(wiki_dir).rglob("*.md")}
    linked = set()
    for name, path in pages.items():
        for target in LINK.findall(path.read_text(encoding="utf-8")):
            target = target.strip()
            linked.add(target)
            if target not in pages:
                print(f"broken link: {name} -> {target}")
    for name in sorted(set(pages) - linked - {"index", "log"}):
        print(f"orphan page: {name}")

if __name__ == "__main__":
    lint(sys.argv[1] if len(sys.argv) > 1 else "wiki")
```

## Scaling past index.md

An index file stops working once the wiki has thousands of pages. Options:

- **qmd.** Karpathy mentions [qmd](https://github.com/tobi/qmd), a local Markdown search engine with BM25, vector search and LLM reranking, all on-device. It has a CLI and an MCP server (`qmd mcp`) with tools like `query` and `get`.
- **Obsidian CLI.** Obsidian has an official command-line interface (it needs the 1.12 installer) that talks to the running app: `obsidian search query="..."`, `obsidian read`, `obsidian create`. An agent can call it from a shell.
- **Basic Memory.** An AGPL-3.0 MCP server that stores notes as Markdown with an SQLite index. Point Obsidian at its folder and both see the same files.
- **OpenClaw Memory Wiki.** OpenClaw's `memory-wiki` plugin compiles agent knowledge into a vault with structured claims and evidence, and can write Obsidian-friendly Markdown.
- **A memory server over the vault.** Hindsight's Obsidian plugin (beta) syncs a vault one way into a Hindsight bank, tags notes by vault, folder and date, and answers questions with citations to the source notes; the vault stays the source of truth.

More options are compared on [AI memory MCP servers](/articles/ai-memory-mcp-server/).

## LLM wiki vs RAG vs memory servers

| | LLM wiki (Obsidian) | RAG over documents | Agent memory server |
|---|---|---|---|
| **What's stored** | LLM-written pages that synthesize sources | Raw chunks of your documents | Extracted facts, events, entities |
| **When work happens** | At ingest: pages updated once | At query: fragments re-found each time | At write: facts extracted per message |
| **Human-readable** | Yes, browse in Obsidian | Only the source docs | Usually through an API or UI |
| **Best for** | Research, reading, personal knowledge | Large document sets | Chat and agent personalization |
| **Weak spot** | Errors in pages propagate; ingest is slow | No synthesis across sources | Hard to audit by hand |

The wiki pattern shines for one person or a small team building up understanding of a topic. It's a poor fit for remembering thousands of users' preferences, where a memory API is the right tool; see [RAG vs agent memory](/articles/rag-vs-agent-memory/) and [persistent memory in AI](/articles/persistent-memory-ai/).

## Risks to watch

- **Compounding errors.** If the LLM misreads a source, the mistake spreads to every page it touches. Keep `raw/` immutable so you can always check, and lint often.
- **Stale claims.** Pages written early may contradict newer sources. The lint step exists for this.
- **Untrusted sources.** A clipped page with hidden instructions can steer the agent while it edits your wiki. Review diffs before committing, especially for web clips.


## FAQ

### What is Karpathy's LLM Wiki?

It's a pattern Andrej Karpathy published as a GitHub gist on April 4, 2026. Instead of retrieving from raw documents on every question, an LLM agent builds and maintains an interlinked Markdown wiki from your sources. It has three layers: immutable raw sources, the LLM-written wiki, and a schema file such as CLAUDE.md or AGENTS.md.

### How do I use Obsidian as memory for an LLM?

Make the vault the agent's working folder, put source documents in a raw folder the agent never edits, and write a schema file that tells the agent how to ingest sources, update pages, keep index.md and log.md, and lint. Run a coding agent like Claude Code or Codex in that folder and browse the results in Obsidian.

### Is an LLM wiki better than RAG?

It's different. RAG re-finds fragments on every question. A wiki compiles knowledge once and keeps it updated, so cross-references and contradictions are already worked out. Karpathy notes an index file works at moderate scale; past that you add a search tool such as qmd.

