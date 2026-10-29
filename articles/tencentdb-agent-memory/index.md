# TencentDB Agent Memory (v2.0.3): Layers, Team Memory, Setup

> TencentDB Agent Memory explained (v2.0.3, Oct 2026): what 2.0.3 changes, L0-L3 memory layers, Team Memory and Memory Hub, proxy setup for Claude Code and OpenClaw, and how it compares to Hindsight and Mem0.

Source: https://aiagentmemory.org/articles/tencentdb-agent-memory/ · Updated 2026-10-29 · aiagentmemory.org


**TencentDB Agent Memory** (repo `TencentDB-Agent-Memory`) is an open-source memory hub for AI agents from Tencent Cloud Database. It sits as a proxy between a coding agent and its LLM, saves each conversation, distills it into four layers (L0 to L3), and injects the right memory, skills, docs and code knowledge into every request. Its Team Memory release lets several agents and people share those assets with access control.

It is MIT-licensed, written mostly in TypeScript, and had about 30,080 GitHub stars on 29 October 2026, up from 29,600 on 19 October. This page covers how its memory layers work, how to install it, what Team Memory adds, which agents it supports, and how it compares with **[Hindsight](https://github.com/vectorize-io/hindsight)** and Mem0. Facts come from the project's [README](https://github.com/TencentCloud/TencentDB-Agent-Memory), INSTALL.md, CHANGELOG.md, the GitHub releases list and the SDK README, rechecked on 19 October 2026; the 2.0.3 section comes from the [v2.0.3 release notes](https://github.com/TencentCloud/TencentDB-Agent-Memory/releases/tag/v2.0.3), read on 29 October 2026.

## What is TencentDB Agent Memory?

**TencentDB Agent Memory is a self-hosted memory system that turns agent conversations, documents and code into four reusable "memory assets": Chat Memory, Skills, Wiki and CodeGraph. A Memory Hub panel decides which agent can use which asset, and a proxy injects them into agents like Claude Code without plugins or code changes.**

The README frames the goal as cutting repeated work: "Any information that helps the next Agent avoid reinventing the wheel should be saved, organized, and reused." So it stores more than chat facts. The four assets are:

- **Chat Memory:** preferences, facts, decisions and history, refined from raw conversation into layers.
- **Skill:** a reusable procedure pulled from finished work, with versions, resource files, trigger boundaries, steps and validation rules. Part of this code comes from Nous Research's Hermes Agent, which the README credits.
- **Wiki:** documents turned into structured pages with a link graph, based on Andrej Karpathy's "LLM Wiki" idea.
- **CodeGraph:** an index of symbols, files, call relationships and impact paths for a repository, built on the open-source `colbymchenry/codegraph` project.

**Timeline.** The project started as an OpenClaw memory plugin: the npm package `@tencentdb-agent-memory/memory-tencentdb` was first published on 31 March 2026, and Tencent's press release says the project was open-sourced in May 2026. The [releases list](https://github.com/TencentCloud/TencentDB-Agent-Memory/releases) shows `2.0.0-beta.1` (22 July 2026) as the first public release of the team version and `2.0.0` stable on 3 August 2026. Tencent Cloud announced **Team Memory** in a PR Newswire release on 13 August 2026. The latest stable version is **`2.0.3` (26 October 2026)**; see [what 2.0.3 changes](#whats-new-in-203-26-october-2026) below.

Two version lines run in parallel, which is easy to misread: the `0.x` numbers belong to the original OpenClaw plugin (latest `0.3.6`, May 2026), while `1.x` and `2.x` are the self-hosted server stack in this repo. If you are deploying the containers, you want the 2.x line.

### What's new in 2.0.3 (26 October 2026)

`2.0.3` is the first stable release with the `2.0.3-beta.1` features listed further down. On top of the beta it adds default session bindings and rich documents in the Wiki, raises the default upstream timeout, and fixes a proxy that could not recover from a Redis outage. The release notes are in Chinese; this is a plain-English summary.

| Area | Before 2.0.3 | In 2.0.3 |
|---|---|---|
| Starting a session | You pick a Team, Agent and Task every time | You can save a default Team/Agent/Task combination; new sessions use it, and you can lock it so later sessions can't change it. If the saved default stops being valid, it falls back to the picker and says why |
| Wiki imports | Markdown, plain text, HTML | Adds Word, Excel, PowerPoint and PDF: import, preview and text extraction. Also fixes Markdown paste imports that sometimes failed |
| Upstream wait (no `forwardTimeoutMs` set) | 10 minutes | 1 hour. Configs that set `forwardTimeoutMs` keep their value |
| Stream with no new output for 60 s | Could end the reply | Only logs a warning; the reply keeps going |
| Redis goes down | Proxy could not recover on its own; you had to restart it | Short outages queue briefly; longer ones degrade at once instead of holding requests; recovers without a restart |
| DeepSeek Harness helper endpoints | Token counting and embeddings returned 404 | Fixed |
| Old memories with bad timestamps | Some could not be turned into L1 memories | Fixed |

Smaller fixes: the one-line macOS install script no longer exits on Chinese punctuation, the Knowledge service starts again (an OpenTelemetry dependency had drifted), and a few panel fixes (keyboard-closable tabs, a message when copying a User Key fails).

**Should you upgrade?** Yes if you run the proxy against Redis in production or see long agent replies cut off around the 10-minute mark: those are the two fixes that change behaviour for existing setups. If you rely on the old 10-minute cut-off as a safety limit, set `forwardTimeoutMs` explicitly before upgrading.

**What `2.0.3-beta.1` added** (13 October 2026, now part of `2.0.3`), all off by default so existing deployments don't change:

- **On-demand skill selection.** The proxy picks the skills that fit each turn and loads them through one client tool call, instead of injecting the whole skill catalogue into the system prompt. That is the fix for the obvious cost of a growing skill library.
- **A pluggable decision service** behind skill selection and the L3 memory-update judgement. JEV is built in and named by the `provider` setting, and other decision APIs can be plugged in. A failed decision call does not block the conversation or the memory write.
- **Session trajectory logging.** A new `/v3/session/add` endpoint stores the messages sent upstream and the model's reply in MemoryCore, so the L0 layer captures proxied turns.
- **SSH clone for private repos** (including non-standard ports) when building a CodeGraph.
- **`ftsAvailable` in `/health`**, which tells you whether BM25 keyword recall is actually working. Worth checking first if retrieval quality looks off, since the L1/L0 fallback depends on it.

## How TencentDB Agent Memory works: the L0-L3 memory layers

Chat Memory is stored in four layers. New conversation is saved as L0 first, then a background pipeline refines it upward once thresholds are reached:

| Layer | What it stores (README) | Used for |
|---|---|---|
| **L0 Conversation** | Raw conversation with full context | Checking exact wording, timestamps and sources |
| **L1 Atom** | Facts, preferences, constraints and events | Precise recall of specific information |
| **L2 Scenario** | Knowledge blocks grouped by project or scenario | Restoring a working context quickly |
| **L3 Core / Persona** | Long-term profile, stable patterns | Getting an agent into a user's and team's context at once |

**Retrieval is layered too.** By default the proxy puts L2 and L3 into the system prompt as a quick bootstrap. When a specific fact is needed, it falls back to L1 and L0 with BM25 keyword search plus vector search, merged by reciprocal rank fusion (RRF). Results are capped by item count, character budget and timeout so memory doesn't flood the context window.

**Documents and code are called on demand.** Wiki pages and CodeGraph assets are not pasted into every prompt. The agent discovers them through `/v3/tools/list` and reads pages, source or impact paths through `/v3/tools/call`. Storage defaults to SQLite inside the container volume; a MongoDB backend is available but marked experimental and off by default.

This layered design is close to the ideas behind [MemoryOS](/articles/memory-os-ai-agent/), the EMNLP 2025 system from BUPT and Tencent AI Lab with short, mid and long-term tiers, though the two are separate projects.

## How to install and use TencentDB Agent Memory

The recommended setup starts three Docker containers: Memory Core (port 8420), Memory Hub with the panel (8125) and knowledge service (8424), and the proxy (8096). Images are published for `linux/amd64` and `linux/arm64`.

1. Clone the repo and enter the deploy folder: `git clone https://github.com/TencentCloud/TencentDB-Agent-Memory.git`, then `cd TencentDB-Agent-Memory/deploy/global-images`.
2. Run `./start-all.sh`. It asks for two LLM settings: a **memory** model (used to extract layers, skills and wiki pages) and a **proxy upstream** model (the LLM your agent will really talk to). It tests both connections before starting.
3. On first boot the script creates an admin user and saves an `sk-mem-...` key to `.admin-key`.
4. Open `http://localhost:8125`, log in with that key, create a **Team**, then create a business user inside it. The INSTALL guide recommends not driving agents with the admin key.
5. Inside the Team, create at least one **Agent** (with a role description and system prompt) and, optionally, a **Task**.
6. Point Claude Code at the proxy:

```bash
export ANTHROPIC_BASE_URL=http://127.0.0.1:8096/claude-code/default
export ANTHROPIC_AUTH_TOKEN='sk-mem-...'   # the business user's key
claude --model <your PROXY_UPSTREAM_MODEL>
```

7. In each new session, the proxy uses Claude Code's `AskUserQuestion` tool to let you pick the Team, Agent and Task. From then on, every turn gets that agent's L2/L3 memory, matching skills and knowledge injected.

To read or write memory from your own code, the repo has a Python SDK in `sdk/memory-core/python`. It wasn't on PyPI when we checked, so install it from GitHub. The v3 client requires `team_id`, `agent_id` and `user_id` on every call:

```python
# pip install "git+https://github.com/TencentCloud/TencentDB-Agent-Memory.git#subdirectory=sdk/memory-core/python"
from tencentdb_agent_memory.v3 import MemoryClient

client = MemoryClient(
    endpoint="http://127.0.0.1:8420",
    api_key="",              # MEMORY_CORE_GATEWAY_API_KEY; empty by default
    service_id="default",
    team_id="team-1", agent_id="agent-1", user_id="user-1",
)

# L0: save a conversation (writes need a session_id)
client.add_conversation(
    [{"role": "user", "content": "Don't refactor the old auth module, mobile still uses it."}],
    session_id="sess-1",
)

# L1: search extracted facts (filled in later by the background pipeline)
hits = client.search_atomic("auth module", limit=5)

# L3: read the agent's long-term profile
persona = client.read_core()["content"]
```

A TypeScript SDK (`@tencentdb-agent-memory/memory-sdk-ts-v2`) exposes the same `MemoryClient`, `SkillClient` and `MetadataClient` classes.

## Team Memory and Memory Hub

**Team Memory** is the 2.0 feature set that moves memory from one user to a team of people and agents. Its core idea is an **agent loadout**: instead of one global memory, each agent gets the assets it needs. The README's example is a one-person company with a Scout, a Builder and a Reviewer agent, each with its own Wiki, CodeGraph, Skills and Chat Memory.

Access control is explicit. New Chat Memory and Skills are private by default, and sharing is a separate action:

| Visibility | Who can read |
|---|---|
| `private` | Only the owner, not even team admins |
| `team` | All team members; owner and admin can manage |
| `restricted` | Users, roles or agents on an ACL |
| `agent` | Specific agents in the same team |

Memory Hub tracks owner, version, status, visibility, usage count and agent bindings for each asset. A system admin creates teams and users; team admins and members manage assets inside a team. Version 2.0.2-beta.1 added OAuth2 login so teams can connect an internal SSO system.

**Cold start** is the other half. You can import existing repositories (indexed into CodeGraph), documents (turned into Wiki pages) and past agent sessions (mined for Skills and Chat Memory). An `asset-import.ts` script scans local skills, memory and sessions from each supported client.

## Supported agents and integrations

Integration works through the proxy, not through plugins, hooks or MCP. The agent's base URL points to the proxy, which speaks both the Anthropic Messages API (`/claude-code/<spaceId>/v1/messages`) and the OpenAI Chat Completions API. The repo's `agents/` folder has a guide for each client:

| Agent | Protocol | How the session picks Team/Agent |
|---|---|---|
| Claude Code | Anthropic Messages | Interactive form (`AskUserQuestion`) |
| CodeBuddy (Tencent) | OpenAI Chat Completions | Interactive form |
| Codex | OpenAI Responses API | Interactive form |
| WorkBuddy | Responses / Chat | Interactive form |
| DeepSeek Harness (dsh) | OpenAI Chat Completions | Interactive form |
| Hermes | OpenAI Chat Completions | Preset headers in config |
| OpenClaw | OpenAI Chat Completions | Preset headers (`x-team-id`, `x-agent-id`...) in `~/.openclaw/openclaw.json` |

The 2.0.2 release also added OpenCode and Pi. For OpenClaw's own built-in memory, see [OpenClaw memory](/articles/openclaw-ai-agent-memory/); for how coding agents handle memory in general, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

**What the proxy design means in practice.** You don't install anything in the agent, and any client that lets you change its base URL can work. In exchange, all of the agent's LLM traffic passes through the proxy, and you configure the upstream model and key on the server. The `.env.example` file also notes that setting a gateway API key currently breaks the proxy's auth flow, so the default local setup runs with that gate off.

## TencentDB Agent Memory vs Hindsight vs Mem0

| | TencentDB Agent Memory | Hindsight | Mem0 |
|---|---|---|---|
| License | MIT | MIT, full server included | Apache 2.0; graph and temporal features are Platform-only |
| Main unit | Team / Agent / Task, with four asset types | Memory bank (`bank_id`) | `user_id`, `agent_id`, `run_id` |
| What it stores | L0-L3 Chat Memory, Skills, Wiki, CodeGraph | Facts, experiences, consolidated observations, mental models | Short extracted facts with entities |
| Retrieval | L2/L3 injected; BM25 + vector + RRF over L1/L0 | Semantic + BM25 + graph + temporal, fused, cross-encoder rerank | Semantic + BM25 + entity matching, fused |
| Reasoning over memory | No | `reflect` returns a reasoned answer | No |
| Docs and code | Built-in Wiki and CodeGraph | No; memory only | No |
| Team sharing | `private` / `team` / `restricted` ACL / `agent`, review panel | Isolated banks | Scoping by IDs |
| How agents connect | LLM proxy (base URL swap) | Native plugins (Claude Code, Codex, Cursor, Copilot), MCP, 60+ integrations | SDKs, REST, hosted MCP server |
| Clients | Python, TypeScript, REST | Python, TypeScript, Go, REST, CLI | Python, TypeScript, REST, CLI |
| Storage | SQLite (default), MongoDB (experimental) | PostgreSQL + pgvector (embedded option) | Your vector store, or Postgres in the server |
| Hosted option | None listed in the README | Hindsight Cloud, usage-based | Mem0 Platform |
| Published benchmark | PersonaMem 48% to 76% with memory on | LongMemEval 94.6, BEAM 1M 73.9, BEAM 10M 64.1 | LongMemEval 94.4, BEAM 1M 64.1, BEAM 10M 48.6 |

All benchmark numbers are vendor-reported. TencentDB's README gives one result, PersonaMem from 48% without memory to 76% with it, but doesn't name the model, dataset size or setup. Hindsight's site reports 86.6 on PersonaMem 32K, but with different setups the two numbers can't be compared directly. The Hindsight and Mem0 rows come from [Mem0's own Hindsight comparison](https://mem0.ai/compare/mem0-vs-hindsight). More detail in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Where Hindsight fits

TencentDB Agent Memory's real strength is team knowledge management: shared skills, wikis and code graphs, with an owner and an ACL on every asset. If that is your problem, it is a strong choice.

Hindsight is the stronger option when:

- **Histories are long.** Hindsight publishes 73.9 on BEAM 1M and 64.1 on BEAM 10M, leading Mem0 by 9.8 and 15.5 points on Mem0's own comparison page. TencentDB Agent Memory publishes no long-history benchmark.
- **The agent should reason over memory.** Observations merge related facts into evidence-backed beliefs, mental models keep standing answers up to date, and `reflect` answers questions that need thinking, not lookup.
- **You don't want a proxy in the LLM path.** Hindsight has native plugins for Claude Code, Codex, Cursor and Copilot, an MCP server, and 60+ integrations, so your agent keeps talking to its LLM directly.
- **You want model freedom or a managed service.** It supports 25+ LLM providers, including local ones, and Hindsight Cloud exists if you don't want to run it yourself.

Its limits: it returns more context per retrieval than Mem0 by default (about 4x, by Mem0's numbers), writes cost LLM calls for extraction and consolidation, and it is a memory system only, with no built-in document wiki or code graph. Mem0 remains the leanest choice for a simple per-user memory API. For a wider view, see [open-source AI memory systems compared](/articles/open-source-memory-systems-compared/).


## FAQ

### What is TencentDB Agent Memory?

TencentDB Agent Memory is an MIT-licensed, self-hosted memory system from Tencent Cloud Database. It turns agent conversations, documents and code into four shared assets (Chat Memory, Skills, Wiki and CodeGraph) and injects them into coding agents such as Claude Code, Codex, CodeBuddy and OpenClaw through an LLM proxy.

### How do I connect Claude Code to TencentDB Agent Memory?

Run ./start-all.sh in deploy/global-images to start Memory Core, Memory Hub and the proxy. Create a Team and an Agent in the panel at localhost:8125, then set ANTHROPIC_BASE_URL to http://127.0.0.1:8096/claude-code/default and ANTHROPIC_AUTH_TOKEN to your sk-mem user key, and start claude.

### Is TencentDB Agent Memory better than Mem0 or Hindsight?

It solves a different problem. TencentDB Agent Memory is strongest for teams that want shared skills, document wikis, code graphs and access control for several coding agents. Mem0 is a lean per-user memory API. Hindsight publishes the strongest long-history benchmark results of the three and adds reasoning over memory with reflect.

### What changed in TencentDB Agent Memory 2.0.3?

Version 2.0.3 (26 October 2026) lets you save a default Team, Agent and Task for new sessions, adds Word, Excel, PowerPoint and PDF imports to the Wiki, raises the default upstream timeout from 10 minutes to 1 hour (unless forwardTimeoutMs is set), and fixes a proxy that could not recover from a Redis outage without a restart. It also ships the on-demand skill selection and session logging from 2.0.3-beta.1.

