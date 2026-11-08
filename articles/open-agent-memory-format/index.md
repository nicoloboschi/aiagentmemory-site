# Is There a Standard Format for AI Agent Memory? (2026)

> No standard AI agent memory format exists (Nov 2026). What 12 systems store, the .mem, Agent Memory Repo and OMP proposals, and how to share memory across agents now.

Source: https://aiagentmemory.org/articles/open-agent-memory-format/ · Updated 2026-11-08 · aiagentmemory.org


There is **no standard format for AI agent memory** as of November 2026. A note an agent saves in one tool stays in that tool's shape: a Markdown file, a cited fact on GitHub's servers, a labeled block, an extracted sentence with a `user_id`, a JSONL knowledge graph. Three open proposals try to fix that. None has real adoption yet, so sharing memory between agents today means picking one place all of them can reach.

This page lists what each system actually stores, compares the proposals, and gives the working options for one memory across several agents. The question got a fresh airing on 8 November 2026 when LoreKit's "[Agent memory needs its OpenTelemetry moment](https://www.lorekit.io/blog/open-agent-memory-format)" reached the Hacker News front page (87 points, 73 comments, [thread](https://news.ycombinator.com/item?id=50411467)).

## What is an agent memory format?

**An agent memory format is the shape a memory system gives to one remembered item: what the unit is (a file, a fact, a block, a graph edge), how it is addressed (a path, a user id, a label, a namespace), what it contains, and when it stops being true.** Two tools can share memory only when they agree on those four answers, or when one of them can translate the other's. Today they rarely agree, which is why each new coding agent, chatbot and memory library ships its own store.

The post that started the current debate borrows the comparison from observability. Before OpenTelemetry (2019), every monitoring vendor had its own agent and data format. Vendors then agreed on a data model, shared naming conventions, one wire protocol and reference implementations, and kept competing on storage and analysis. Agent memory, the post argues, is where observability was before that agreement: many implementations and nothing agreed underneath them.

## What each memory system stores (November 2026)

The table below is from each project's own docs or README, read on 8 November 2026. "Forgetting" means what happens to an item nobody touches.

| System | Unit of memory | Addressed by | Forgetting | Who writes it |
|---|---|---|---|---|
| [AGENTS.md](https://agents.md/) | One Markdown file, no required fields | File path; the closest file to the edited code wins | Never, unless a person edits it | A person |
| Claude Code auto memory | Markdown notes under `~/.claude/projects/<project>/memory/`, indexed by `MEMORY.md` | Project directory | Never automatic | The model |
| Codex memories | Files under `~/.codex/memories/`, "generated state" | Project | Never automatic | The model, after a chat goes idle |
| [Anthropic memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) | A file under `/memories`, operated with `view`, `create`, `str_replace`, `insert`, `delete` | A path the model picks; your handler maps it to real storage | Never, unless the model deletes it | The model |
| [GitHub Copilot Memory](https://docs.github.com/en/copilot/concepts/prompting-and-context/copilot-memory) | A repository-level fact or a user-level preference, with citations | One repository, or one user per billing entity | Expires 90 days after the last upvote by default (7 to 180 configurable for repo facts); retrieval does not reset it | Copilot, from its own work |
| Letta memory blocks | A block with `label`, `description`, `value` and a character `limit`, always in context | Block label; one block can be attached to several agents | Never; the agent rewrites it | The agent (or your code) |
| Mem0 | A fact an LLM extracts from messages ("User's dog Biscuit turned 5") | `user_id`, `agent_id`, `run_id`, `app_id` | Optional `expiration_date` (`YYYY-MM-DD`); expired items are hidden, not deleted; default never | A service, from the conversation |
| LangGraph `BaseStore` | A JSON dict value: `store.put(namespace, key, value)` | A namespace tuple such as `(user_id, "memories")` plus a key | Optional TTL; by default a read refreshes it (`refresh_on_read` is `True`) | Your code or the agent |
| [MCP memory server](https://github.com/modelcontextprotocol/servers/blob/main/src/memory/README.md) | Entities with observations, and typed relations | Entity name | Never | The model, through tools |
| Graphiti / Zep | A fact on a graph edge with a validity window | Entities and the edge | Facts are "invalidated, not deleted" when they change | A service, from episodes |
| LoreKit | A lesson record (scope, key, lifecycle) stored as a Markdown file | Scope string (`branch::`, `repo::`, `project::`, `global`) plus key; the most specific copy wins | Archived by measured use, not deleted | The agent, when it hits friction |
| Hindsight | World facts, experiences, observations and mental models in a bank | Bank id, shared by any client with access | Observations are refined as evidence arrives; nothing expires automatically | An LLM extracts on `retain`; consolidation runs in the background |

Two corrections to the LoreKit post, from the primary docs. Copilot's expiry is 90 days by default, not 28, and it is reset by an upvote, not by reading the memory. And the MCP reference server does handle two clients on one file: every write takes an exclusive lock file and saves atomically, so writes take turns instead of overwriting each other.

The split that matters most is the last column. In instruction files a person writes the memory. In the memory tool, Letta, LoreKit and the coding agents' auto memory, the model writes it. In Mem0, Graphiti and Hindsight a service extracts it from the conversation. A reader who cannot tell which of the three produced a record cannot know how much to trust it.

## The three proposals, compared

| Proposal | Backer | What it fixes | Shape | Status (8 Nov 2026) | License |
|---|---|---|---|---|---|
| [Portable Memory](https://github.com/MacPaw/portable-memory) (`.mem`) | MacPaw Research | Export: moving one person's memory between vendors, with verifiable deletion | A folder: `manifest.json`, `items/<kind>.jsonl` (episode, entity, edge, fact, core, procedure...), `audit/tombstones.jsonl`, `CHECKSUMS` | Paper and Python + Swift reference SDKs (byte-identical output); adapters for Mem0, ChatGPT and Claude exports; 7 GitHub stars | MIT |
| [Agent Memory Repo](https://github.com/AgentMemoryRepo/agentmemoryrepo) | Cognition (also published at cognition.com) | Live sharing: many sessions and agents reading and writing one memory | A git repository: `MEMORY.md` entry point, Markdown bullets with `[source: ...; added: YYYY-MM-DD]`, `[[path]]` cross-links; clone, grep, update, push after every edit; a periodic "Dreaming" agent merges and prunes | Spec plus a skill for Devin, Claude Code and Cursor; 855 stars, last push 6 October 2026 | MIT |
| [Open Memory Protocol](https://github.com/askmohanty/open-memory-protocol) (OMP) | Quefly Enterprises LLP | The wire contract: five verbs (`recall`, `remember`, `link`, `observe`, `validate`) with a provenance schema and latency targets | Transport-agnostic verb shapes; storage, ranking and embeddings left to the implementer | Editor's Draft v0.1; needs two independent implementations before "Candidate Recommendation"; the reference implementation (memex) is one; 2 stars, last push May 2026 | CC BY 4.0 (spec), Apache 2.0 (code) |

They solve different layers. Portable Memory is an interchange format: its headline feature is a deletion tombstone that must propagate to every derived copy (indexes, embeddings, caches) before a product can claim its level-2 conformance badge. Agent Memory Repo is an addressing convention that reuses git for identity, history and conflicts. OMP is a protocol with no data model for the memory itself beyond provenance fields. The LoreKit post's own sketch is a fourth shape, a Markdown file with YAML front matter (`address`, `key`, `kind`, `written_by`, `lifecycle`), offered as an illustration rather than a proposal.

Letta's [Agent File](https://github.com/letta-ai/agent-file) (`.af`, Apache 2.0, 1,215 stars) is often named in this discussion but solves a different problem: it serializes a whole agent (system prompt, memory blocks, tool code and schemas, model settings) for import into a Letta server, not a person's memory for use by a different vendor.

None of the three has a neutral home. The [Agentic AI Foundation](https://aaif.io/projects) under the Linux Foundation lists AGENTS.md, MCP, Agent Skills, A2A and six other projects, and no memory format among them, as of 8 November 2026.

## Why there is no standard yet

The Hacker News thread is a fair sample of the objections, and they are not trivial:

- **The reader is a model.** Several commenters argued that a unified format matters less when the consumer is an LLM that reads any text. Agents already read `CLAUDE.md`, `.cursorrules` and `AGENTS.md` without a schema.
- **It is early.** Others said the field does not yet know how long memories should live or what the right abstraction is, and that standardizing now risks a SOAP-style spec nobody wants.
- **A plain directory works.** A recurring practical answer was a memory directory the agent manages with minimal rules, which is what Anthropic's memory tool and the coding agents' auto memory already are.
- **Shared memory is an injection channel.** Anything an agent reads can steer it, and a shared store widens who gets to write. A format can record who wrote a memory and let records be signed, but it cannot make poisoning impossible. See [AI memory injection](/articles/ai-memory-injection/).
- **Concurrency.** One commenter asked how a spec would stop two parallel sessions silently overwriting the same memory. Agent Memory Repo answers with git conflicts; the MCP server with a lock file; Hindsight with observations that merge new evidence instead of replacing it. Per-file formats with none of these lose writes.

The counter-argument from the post is also fair: every system above answers the same four questions (where a memory is true, what its identity is, what it says, when it stops being true). They disagree on the answers, not on the questions, which is where OpenTelemetry started too.

## How to share memory between agents today

Until a format wins, interoperability is a choice of place rather than shape. These work now, in rising order of capability and dependency:

1. **Keep the human layer in `AGENTS.md`.** Build commands, conventions and rules that must always apply go in the file every coding agent reads (Claude Code reads it when no `CLAUDE.md` is present). Our [Claude Code memory](/articles/claude-code-memory/) and [Codex memory](/articles/codex-memory/) pages cover each tool's load order.
2. **For agent-written notes several tools must read, use one Markdown store at a known path.** Agent Memory Repo's layout (a `MEMORY.md` index, one-line entries with a source link and date, `[[links]]`) is a reasonable convention, and git gives you history and conflict detection for free. [Everything Claude Code](/articles/everything-claude-code-memory/)'s Memory Vault takes the same approach for handoffs between harnesses.
3. **For structured memory, run one MCP memory server and point every client at it.** The reference server stores a JSONL graph; set `MEMORY_FILE_PATH` to a path you own, because the default sits inside `node_modules`. Mem0, Basic Memory and Hindsight also expose MCP servers; see [AI memory MCP servers](/articles/ai-memory-mcp-server/).
4. **For memory that is extracted, consolidated and queried, use one memory server with plugins for each agent.** Hindsight's plugins for Claude Code, Codex, Cursor, Copilot, Hermes and OpenClaw, and its MCP server for Grok Bot, write to and read from the same banks, so a fact learned in one harness is recalled in another without any format conversion.
5. **Record who wrote each memory and when.** Whatever the store, keep a source link and a date on each entry (Agent Memory Repo's `[source: ...; added: ...]`, Copilot's citations, Hindsight's quotes on each observation). It is the one field every proposal agrees on.
6. **Decide forgetting explicitly.** Copilot forgets after 90 days without reinforcement; LangGraph refreshes a TTL on read; Mem0 hides expired items; most file formats never forget. Pick one rule and apply it, or the store fills with stale lessons. See [AI memory decay](/articles/ai-memory-decay/).
7. **Export on a schedule, and test the import.** Vendor exports exist (Letta `.af`, Hindsight transfer archives, Copilot CLI import, ChatGPT and Claude memory export prompts). A backup you have never restored is not a backup.

## Where Hindsight fits: one server instead of one format

Hindsight's answer to the interoperability problem is not a file format but a shared server. [Hindsight](https://github.com/vectorize-io/hindsight) is an MIT-licensed memory server with the full feature set (graph, temporal recall, consolidation, reflect, mental models) in the open-source version, and its 60+ integrations include plugins for Claude Code, Codex, Cursor, Copilot, Cline, Hermes Agent and OpenClaw, an MCP server, and a Grok Bot setup built on that MCP server. They all speak to the same **banks**: the Hermes and OpenClaw plugins expose the same `hindsight_retain`, `hindsight_recall` and `hindsight_reflect` tools, the Hermes plugin's `bank_id_template` can key banks by profile, project or user, and Grok Bot's setup documents reading banks that Claude Code, ChatGPT, Hermes and OpenClaw wrote. In the LoreKit post's terms, that is "live addressing" solved by making the address a bank id rather than a file path.

It also covers the export problem within its own ecosystem. Per its [OpenAPI spec](https://api.hindsight.vectorize.io/openapi.json) (read 8 November 2026), `POST /v1/default/banks/{bank_id}/transfer/export` writes a bank, or chosen documents, to a transfer ZIP with documents, facts, observations, mental models, knowledge pages and optional audit history. Embeddings and database ids are never included: the import re-embeds with the target bank's model and re-resolves entities with no LLM extraction, so a bank can move to an instance that uses a different embedding model, and the import "costs no tokens and invents no new facts". A separate endpoint exports a bank's knowledge base as a Markdown bundle, and a subject-access export assembles everything about one tagged subject for data-protection requests.

The limits are real. The transfer archive is Hindsight's own format, not a cross-vendor standard: another vendor cannot read it without an adapter, exactly as with everyone else. Every `retain` and every consolidation pass calls an LLM, so writes cost more than appending to a Markdown file. And a memory server is a dependency; the README itself says Hindsight "may be overkill" for simple workflows. For a single developer with one agent, `AGENTS.md` plus the agent's own auto memory is enough. Hindsight earns its place when several agents or people must share what was learned, when the history runs to months, or when you want to ask "why" over it. See [what Hindsight is](/articles/ai-memory-hindsight/) and the broader [memory types in AI agents](/articles/ai-agents-memory-types/) for how observations and mental models relate to the formats above.


## FAQ

### Is there a standard format for AI agent memory?

No. As of November 2026 every memory system has its own shape: Markdown instruction files, files under /memories, Copilot's cited facts, Letta's labeled blocks, Mem0's extracted facts, LangGraph's namespace-and-key records, the MCP memory server's JSONL graph. Three published proposals (MacPaw's Portable Memory, Cognition's Agent Memory Repo, the Open Memory Protocol draft) exist, none is widely adopted, and no memory format sits under the Agentic AI Foundation, which stewards AGENTS.md and MCP.

### How can several coding agents share one memory today?

Three working options: a shared Markdown file at a known path (AGENTS.md, or a git memory repo like Agent Memory Repo), a shared MCP memory server that every client calls, or a memory server such as Hindsight where Claude Code, Codex, Cursor, Hermes and OpenClaw plugins write to and read from the same bank. The file route needs no server but has no provenance or expiry; the server routes add both but add a dependency.

### Can I export my agent's memory and import it somewhere else?

Only within one vendor, or through an adapter. Letta exports agents as .af files, Hindsight exports a bank as a transfer ZIP that another Hindsight instance restores (re-embedding with its own model), Copilot imports memories through its CLI, and MacPaw's portable-memory tool converts ChatGPT, Claude and Mem0 exports into a .mem bundle. No two vendors read each other's native format.

