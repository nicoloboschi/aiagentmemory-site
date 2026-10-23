# Hermes Agent Plugins: Install, Catalog, Memory Providers

> How Hermes Agent plugins work: plugin types, hermes plugins install, the SHA-pinned catalog, how memory provider plugins hook in, and which ones exist (Oct 2026).

Source: https://aiagentmemory.org/articles/hermes-agent-plugins/ · Updated 2026-10-23 · aiagentmemory.org


**Hermes Agent plugins** are folders of Python code with a `plugin.yaml` manifest that add tools, hooks, slash commands, memory backends and other features without changing Hermes's core. You install them with `hermes plugins install <name>` from a human-reviewed, commit-pinned catalog, or from any Git repo. Memory provider plugins are a special kind: only one runs at a time.

[Hermes Agent](https://github.com/NousResearch/hermes-agent) is Nous Research's open-source (MIT) agent, with about 256,000 GitHub stars on October 21, 2026. This page covers the plugin system: plugin types, install commands, the catalog, how memory providers hook in, and which ones exist. For how Hermes's built-in memory works (MEMORY.md, USER.md, session search), see [Hermes Agent memory](/articles/hermes-agent-memory/). Facts come from the [Hermes plugin docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) and the repo's `plugin-catalog/` folder, checked October 21, 2026.

## What is a Hermes Agent plugin?

**A Hermes plugin is a directory with a `plugin.yaml` manifest and an `__init__.py` that defines `register(ctx)`. Hermes calls that function at startup, and the plugin uses `ctx` to register tools the model can call, lifecycle hooks, slash commands, CLI subcommands, bundled skills or backends such as a memory store.**

User plugins live in `~/.hermes/plugins/<name>/`. Hermes also discovers bundled plugins in the repo's `plugins/` folder, pip packages that expose a `hermes_agent.plugins` entry point, and NixOS plugins declared in the module config. Project-local `./.hermes/plugins/` folders are not loaded: the docs say the gateway that runs every session discovers plugins once per process, not per working folder.

Third-party code is off by default. Discovery finds a plugin, but nothing with tools or hooks loads until its name is in `plugins.enabled` in `~/.hermes/config.yaml`. The docs say this "stops third-party code from running without your explicit consent."

## Types of Hermes plugins

The [Hermes plugin docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) name four kinds:

| Type | What it does | How many run | How it's switched on |
|---|---|---|---|
| **General plugins** | Add tools, hooks, slash commands, CLI commands, skills | Any combination | `plugins.enabled` / `hermes plugins enable` |
| **Memory providers** | Add an external memory store next to MEMORY.md and USER.md | One | `memory.provider` in `config.yaml` |
| **Context engines** | Replace the built-in context compressor | One | `context.engine` in `config.yaml` |
| **Model providers** | Declare an inference backend (OpenRouter, Anthropic, and so on) | Many load; you pick one | `--provider` or `config.yaml` |

Inside general plugins, `ctx` has more entry points: gateway platforms (Discord, Telegram, IRC), image and video generation backends, terminal sandboxes, computer-use drivers, approval transports, TTS and STT backends, and `ctx.llm.complete()` for borrowing the user's active model. Some extensions aren't Python plugins at all: MCP servers are declared under `mcp_servers` in `config.yaml`, and shell hooks under `hooks:`.

Hooks are the main way a general plugin reacts to the agent. The docs list hooks such as `pre_tool_call`, `post_tool_call`, `pre_llm_call`, `post_llm_call`, `on_session_start`, `on_session_end` and `post_compaction`. A hook callback that blocks longer than `plugins.hook_callback_timeout` (default 30 seconds) is abandoned; a timed-out `pre_tool_call` blocks the tool.

## How to install Hermes Agent plugins

The usual path, from the docs:

1. **Find it.** `hermes plugins search <term>` searches the catalog by name, description and declared tools. `hermes plugins browse` lists every entry and `hermes plugins info <name>` shows one.
2. **Install it.** `hermes plugins install <name>` clones the entry's repo at its reviewed commit. The prompt shows the declared tools, hooks and environment variables before anything is cloned.
3. **Answer the enable prompt.** Hermes asks `Enable 'name' now? [y/N]` (default no). Use `--enable` or `--no-enable` in scripts. For a memory provider it asks `Use 'name' as the memory provider now?` instead.
4. **Grant capabilities if asked.** Plugins that declare privileged capabilities (for example `tools.override`) get a one-time consent prompt. Grants are saved under `plugins.entries.<id>.granted_capabilities`. Without a terminal, they stay ungranted. Python dependencies have their own consent step that `--enable` doesn't skip: a declined or non-interactive dependency install leaves the plugin installed but disabled. Hermes v0.22.4 (released October 21, 2026) skips that step for a plugin that installs no dependencies, per its release notes.
5. **Restart or start a new session.** Run `/plugins` in a session to see what loaded, or `hermes plugins list` for enabled, disabled and not-enabled states.
6. **Update on purpose.** `hermes plugins update <name>` moves a catalog install to the catalog's current pin; with no name (v0.22.3) it lists every plugin with an update and asks "Update all N?" (`--yes` skips the question). `hermes plugins check-updates` only reports.
7. **Remove it.** `hermes plugins remove <name>` uninstalls it, drops it from `plugins.enabled` and `plugins.disabled`, and resets `memory.provider` if it was the live provider.

The full command set:

```bash
hermes plugins                          # interactive toggle UI
hermes plugins list                     # all plugins and their state
hermes plugins search memory            # search the catalog
hermes plugins install <name>           # catalog entry at its pinned SHA
hermes plugins install owner/repo       # any Git repo (unreviewed)
hermes plugins install owner/repo --ref <40-char-sha>   # pinned custom install
hermes plugins enable <name>
hermes plugins disable <name>
hermes plugins update [<name>]
hermes plugins capabilities [<name>]    # declared vs granted
hermes plugins remove <name>
```

Every install and update runs a static security scan first. A `safe` verdict installs, `caution` asks `Install anyway? [y/N]`, and `dangerous` is blocked even with `--force`. A catalog install at its reviewed pin skips the `caution` question, since a reviewer already saw those findings. Config lives in `~/.hermes/config.yaml` under `plugins:` (`enabled`, `disabled`, `clone_timeout_seconds`, `hook_callback_timeout`, `load_timeout_seconds`); install provenance goes to `~/.hermes/plugins/.install-metadata.json`.

## The Hermes plugin catalog

The catalog is a folder of YAML files in the hermes-agent repo, one per plugin, published as one JSON feed that clients refresh at most every six hours. The live feed listed 754 entries on October 21, 2026, according to `plugin-catalog.json` on the Hermes docs site. By shelf, that was 241 tools, 174 desktop, 67 memory, 66 models, 52 general, 48 automation, 42 web, 41 platform and 21 voice, with two entries uncategorized.

The rules in the [catalog README](https://github.com/NousResearch/hermes-agent/tree/main/plugin-catalog) are what make a bare name safe to install:

- **Human-merged.** Entries are added only by pull requests a maintainer reviews. No self-serve registry.
- **Exact SHA pins.** Each entry pins a full 40-character commit. A new pin is a new reviewed PR.
- **No self-updating code**, no runtime patching of Hermes core, no compiled binaries that a reviewer can't read.
- **Declared capabilities must match** what the plugin registers at the pinned commit.
- **A removed list.** Pulled plugins go in `removed.yaml`, and the installer refuses them by name or repo URL.

Each entry has a tier. `official` means maintained by Nous Research, `verified` means Nous checked it comes from the company whose product it integrates, and `community` is everything else. All 67 Memory-shelf entries were `community` tier on October 21, 2026, including Honcho, Mem0, Supermemory and Hindsight, which their own companies maintain. So the tier doesn't separate memory providers from each other.

Hermes is also moving its own features out of core into `official` plugins. In [v0.22.5](https://github.com/NousResearch/hermes-agent/releases/tag/v0.22.5) (October 22, 2026), `hermes import-agent` and the OpenClaw migration became the `agent-importers` plugin, and `hermes mcp serve` left core too. Run an old command without its plugin and Hermes prints the install line (`hermes plugins install agent-importers`). The importer is how memory moves in from OpenClaw, Claude Code, Codex CLI or Cursor; [OpenClaw vs Hermes Agent memory](/articles/openclaw-vs-hermes-agent-memory/#migrating-between-them) shows what it does with each file.

The docs are blunt about what the catalog is not: "Cataloged ≠ audited." Review checks the pinned commit and the declared capabilities at admission. It isn't a security audit, so read the code of anything you give credentials to. A bare name that isn't in the catalog is an error; `owner/repo` installs skip the catalog and show an unvetted-code warning.

## How memory provider plugins hook in

**A memory provider plugin subclasses `MemoryProvider` from `agent/memory_provider.py` and registers it with `ctx.register_memory_provider()`.** It runs alongside the built-in MEMORY.md and USER.md, never instead of them. Only one is active, chosen by `memory.provider`; the developer guide says a second registration is rejected with a warning. The [memory providers page](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) describes what an active provider does.

When a provider is active, Hermes injects its context into the system prompt, prefetches memories before each turn, syncs each turn after the response, extracts memories at session end where supported, mirrors built-in memory writes to it, and adds the provider's own tools. The [provider developer guide](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) lists the methods behind each step:

| Method | When Hermes calls it | Required? |
|---|---|---|
| `name`, `is_available()`, `initialize(session_id, **kwargs)` | Startup; `is_available()` must make no network calls | Yes |
| `get_tool_schemas()`, `handle_tool_call()` | Tool injection, then when the agent uses a tool | Yes (handler if it has tools) |
| `get_config_schema()`, `save_config()` | `hermes memory setup` | Yes (save unless env-only) |
| `system_prompt_block()` | System prompt assembly | Optional |
| `prefetch(query)` | Before each API call; returns recalled context | Optional |
| `queue_prefetch(query)` | After each turn, to pre-warm the next | Optional |
| `sync_turn(user, assistant)` | After each completed turn; must not block | Optional |
| `on_session_end(messages)` | Conversation ends | Optional |
| `on_pre_compress(messages)` | Before context compression | Optional |
| `on_memory_write(action, target, content)` | Built-in MEMORY.md / USER.md writes | Optional |
| `shutdown()` | Process exit | Optional |

Two details trip people up. First, `hermes plugins enable <provider>` doesn't activate a memory provider; only `memory.provider` does, and `hermes memory setup` writes it. Second, memory providers resolve name collisions the opposite way from general plugins: the bundled one wins, so a home-made provider needs a unique name. Everything Hermes hands a provider passes through the same secret scrub as chat delivery first, so a key echoed into tool output is masked before the provider stores it.

## Memory provider plugins for Hermes Agent

The Hermes docs document nine external providers. Six install from the catalog, and the rest don't:

| Provider | Install | Where data lives | What its docs or catalog entry say |
|---|---|---|---|
| **Honcho** (Plastic Labs) | `hermes plugins install honcho` | Honcho Cloud or self-hosted | User modeling with dialectic reasoning; 5 tools; needs `HONCHO_API_KEY` |
| **OpenViking** (Volcengine) | `hermes plugins install openviking` | Local, managed cloud or your server | Captures conversations, recalls facts, searches your knowledge; 6 tools |
| **Mem0** | `hermes plugins install mem0` | Platform, self-hosted or local OSS | Server-side LLM fact extraction, semantic search, dedup, opt-in reranking; 4 tools |
| **Hindsight** (Vectorize) | `hermes plugins install hindsight` | Hindsight Cloud, local embedded Postgres, or your own server | Knowledge graph, entity resolution, multi-strategy retrieval; retain, recall and reflect tools |
| **Holographic** | `hermes plugins install holographic` | Local SQLite | FTS5 fact store with trust scores and HRR recall; no network unless optional embeddings are on |
| **Supermemory** | `hermes plugins install supermemory` | Supermemory Cloud or self-hosted | Profile recall, semantic search, per-turn capture; 4 tools |
| **RetainDB** | `hermes memory setup` | RetainDB Cloud ($20/month listed) | Hybrid vector + BM25 search; 10 tools |
| **ByteRover** | `brv` CLI, then `hermes memory setup` | Local, optional cloud sync | Knowledge tree with tiered retrieval; 3 tools |
| **Memori** | external `hermes-memori` CLI | Memori Cloud | Structured recall; 5 tools |

Honcho, Mem0, Supermemory, OpenViking, Hindsight and Holographic all used to ship inside Hermes. If `memory.provider` already names one, `hermes update` installs its catalog plugin and keeps the same settings and data. The docs say RetainDB and ByteRover leave core on October 15, 2026, with their repos "unmaintained and open for a new maintainer"; their folders were still in the repo's `plugins/memory/` on October 21. **Mnemosyne** (`hermes plugins install mnemosyne`) is another local option, SQLite with vector + FTS5 hybrid recall, that declares 41 tools in its catalog entry; see [Mnemosyne vs Hindsight](/articles/mnemosyne-vs-hindsight/). Holographic's design is explained in [holographic memory for AI agents](/articles/holographic-memory-ai-agent/), and Honcho in [Honcho LLM memory](/articles/honcho-llm-memory/).

### Other Memory-shelf entries

The rest of the 67, one line each from their catalog descriptions (October 21, 2026). Not every one is a `memory.provider`: a few are hooks, context engines or MCP packages, as marked.

| Entry | What its catalog entry says |
|---|---|
| agent_memory_es | Recall, retain and reflect tools over an Elasticsearch service, hybrid BM25 + kNN |
| agentplaybooks-memory | Recall and memory-write mirroring to agentplaybooks.ai |
| betterdb | Semantic recall on Valkey / valkey-search |
| bilinc | Hosted memory with a full history of what changed and why |
| birkin-mnemosyne | Local Markdown memory with multilingual keyword recall |
| brethof-brain | Session briefing, recall every turn, one memory shared across your agents |
| cashew | Thought-graph memory with local embeddings and decay |
| chronicle | Local SQLite memory plus memory-aware context compression |
| cognition | Facts regex-extracted at session end into local SQLite |
| cognee | Knowledge-graph memory on a local cognee server or Cognee Cloud |
| compartment | Encrypted offline vault with a bundled embedding model |
| cortexlayer | Page-linked memory with link-expansion retrieval |
| deja-memory | Memory built from coding-agent sessions already on the machine |
| dexio | Shared-wiki memory your team and agents also edit |
| entropicmem | SQLite FTS5 facts plus a Markdown vault with graph export |
| excel_line | Excel-backed hierarchical memory tree |
| gbrain-plugin | GBrain over MCP as a local memory layer (status command and setup scripts) |
| gbrain-pointer | Read-only provider that searches a GBrain MCP server |
| gbrain-retrieval-reflex | `pre_llm_call` hook that injects GBrain context; not a `memory.provider` |
| hermes-dreaming | Nightly consolidation that proposes facts worth keeping |
| hermes-lcm-x | Lossless session store; a context engine (`context.engine: lcm-x`) |
| hermes-pubky | Setup helper that syncs agent state to a Pubky homeserver |
| hermes-snapcompact | Context engine that archives history as images |
| hicortex | Fleet memory shared by every connected agent |
| hwiki | Markdown wiki memory with SQLite FTS5 search |
| hydradb | Recall before each turn and background saves to HydraDB |
| hyatlas | Self-hosted Go server with hybrid vector + keyword search |
| icarus | Self-memory and replacement models |
| intelligent_memory | Local Arabic/English memory with canonical facts |
| jev-memory-selector | Skill plus an MCP server; registers no Hermes tools or hooks |
| lancedb | Embedded LanceDB vector + FTS recall |
| lancedb-suite | Local LanceDB memory; needs Ollama + nomic-embed-text |
| limbic | Local SQLite with sqlite-vec + FTS5 and trust scoring |
| loreconvo | Cross-surface session memory in a local SQLite store |
| lumberroom | One store shared with all your agents; self-hosted or cloud |
| mem-os | MCP package for local persistent facts |
| memanto | Typed memory with RAG answers on Moorcheh cloud |
| memobase | Self-hosted user profile + event timeline recall |
| memex8 | Self-hosted Qdrant vector memory |
| memoria | Memoria Cloud with scoped recall |
| memory-zvec | In-process HNSW + RocksDB FTS store |
| metronix | Prefetch recall and built-in memory write-through |
| mindgraph | Semantic graph memory with goal and decision tracking |
| mnemostack | Hybrid vector + lexical + graph recall over Qdrant |
| nachos | Local SQLite memory and context compression |
| nexus | Self-hosted Qdrant memory with a knowledge graph |
| openltm | Local SQLite with FTS5, optional vectors and decay |
| optchat | Append-only log plus a summary tree |
| optmem-hermes | Local append-only memory with regex/BM25 recall |
| pastdotdev | Recall from past.dev; sessions sent at session end |
| penny | Core memory and notes on app.mypenny.ai |
| post-cortex | Typed records in an entity graph via a local daemon |
| qdrant | Verbatim turns in a Qdrant collection you host |
| qdrant-memory | LLM fact extraction into Qdrant |
| reme | File-native memory with HTTP and embedded backends |
| stack | Git-backed Markdown memory with background curation |
| sugar | Local SQLite memory for coding agents |
| tam | total-agent-memory over MCP |
| xmemo | Cloud memory on xmemo.dev |
| yantrikdb | Dedups memories and surfaces contradictions |

None of these projects publishes a head-to-head benchmark inside Hermes, and the catalog doesn't rank them. Pick by where your data may go (each entry has a disclosure line saying what leaves the machine) and by who maintains it.

## Where Hindsight fits

**[Hindsight](https://github.com/vectorize-io/hindsight)** is Vectorize's open-source memory server, MIT licensed with every feature in the self-hosted version. It installs with `hermes plugins install hindsight` and is maintained by Vectorize in its own repo; the old `hindsight-hermes` pip plugin is deprecated. It has three tools (`hindsight_retain`, `hindsight_recall`, `hindsight_reflect`), three modes (Hindsight Cloud, a local daemon with built-in Postgres that stops after 5 minutes idle, or your own Hindsight server) and three `memory_mode` settings: `hybrid` (automatic context plus tools, the default), `context` or `tools`.

Of the providers above, Hindsight and Mem0 are the two with published head-to-head numbers. On Mem0's own comparison page (checked October 2026), Hindsight and Mem0 tie on LongMemEval (94.6 vs 94.4) and LoCoMo (92 vs 92.5). On the same page, Hindsight scores 73.9 vs Mem0's 64.1 on the BEAM 1M long-history test. On the same page, Hindsight scores 64.1 vs Mem0's 48.6 on BEAM 10M. All of these are vendor-run results, not independent audits.

What that means for a Hermes agent:

- **Long-running agents.** A Hermes agent on Telegram or Discord can run for months. The BEAM 1M and 10M gaps are the data most relevant to that.
- **Learning, not just storage.** Hindsight consolidates facts into **observations** (evidence-backed beliefs with quotes and a proof count) and **mental models**, and `reflect` returns a reasoned answer instead of raw snippets.
- **Fully local if needed.** Hindsight supports 25+ LLM providers, including Ollama, LM Studio and llama.cpp, so the embedded mode can keep memory on the machine.

The limits that matter here. Hindsight's server returns more context per retrieval than Mem0 at its own defaults: 24K to 44K tokens vs under 7K, by Mem0's measurement. The Hermes plugin caps that: recall is limited to 4,096 tokens per turn by default (`recall_max_tokens`) and returns only consolidated observations unless you set `recall_types` to include raw facts, per Hindsight's Hermes integration docs (October 2026). Hermes prefetches recall every turn, so lower `recall_max_tokens` if prompt cost matters. Each retain and consolidation pass calls an LLM, so writes cost more than a plain FTS store like Holographic. Its catalog entry is community tier like every memory entry, and in cloud mode every turn goes to Hindsight's API, as the catalog disclosure says. More in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/) and [Hindsight explained](/articles/ai-memory-hindsight/).

## How to write a Hermes plugin

A general plugin needs two files. This is the docs' minimal example, with the hook written to the `post_tool_call` signature from the [hooks reference](https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks) (Hermes passes the hook's fields as keyword arguments).

`~/.hermes/plugins/hello-world/plugin.yaml`:

```yaml
name: hello-world
version: "1.0"
description: A minimal example plugin
```

`~/.hermes/plugins/hello-world/__init__.py`:

```python
import json


def register(ctx):
    schema = {
        "name": "hello_world",
        "description": "Returns a friendly greeting for the given name.",
        "parameters": {
            "type": "object",
            "properties": {"name": {"type": "string", "description": "Name to greet"}},
            "required": ["name"],
        },
    }

    def handle_hello(params, **kwargs):
        name = params.get("name", "World")
        return json.dumps({"success": True, "greeting": f"Hello, {name}!"})

    ctx.register_tool(name="hello_world", toolset="hello_world",
                      schema=schema, handler=handle_hello)

    def on_tool_call(tool_name, **kwargs):
        print(f"[hello-world] tool called: {tool_name}")

    ctx.register_hook("post_tool_call", on_tool_call)
```

Then run `hermes plugins enable hello-world` and start a new session. The model sees the text in `schema["description"]`, so write the tool's description there.

A memory provider follows the same `register(ctx)` pattern. The skeleton from the developer guide:

```python
from agent.memory_provider import MemoryProvider
from agent.secret_scope import get_secret


class MyMemoryProvider(MemoryProvider):
    @property
    def name(self) -> str:
        return "my-provider"

    def is_available(self) -> bool:
        return bool(get_secret("MY_API_KEY"))  # no network calls here

    def initialize(self, session_id: str, **kwargs) -> None:
        self._session_id = session_id
        self._home = kwargs.get("hermes_home")

    def get_tool_schemas(self):
        return []

    def prefetch(self, query: str, *, session_id: str = "") -> str:
        return ""  # recalled context for this turn


def register(ctx) -> None:
    ctx.register_memory_provider(MyMemoryProvider())
```

Read secrets with `get_secret()`, not `os.environ`, so each Hermes profile gets its own key. Run slow work in `sync_turn()` on a thread from `agent.memory_provider.spawn_context_thread`, since the docs say a bare thread can write into the wrong profile's store. To list a plugin in the catalog, open a PR adding one `plugin-catalog/<name>.yaml` with an exact SHA, following the submission guide. Since prompt-injected memory is a real attack path, read [AI memory injection](/articles/ai-memory-injection/) before shipping a provider that stores untrusted chat content.


## FAQ

### How do I install a plugin in Hermes Agent?

Run hermes plugins search <term> to find it in the catalog, then hermes plugins install <name>. Catalog installs check out the exact commit Nous Research reviewed. General plugins also need hermes plugins enable <name>; a memory provider is switched on by memory.provider in ~/.hermes/config.yaml, which the install prompt or hermes memory setup writes.

### Which memory provider plugins does Hermes Agent have?

On October 21, 2026 the Hermes plugin catalog had 67 entries on its Memory shelf, all community tier. The Hermes docs document Honcho, OpenViking, Mem0, Hindsight, Holographic and Supermemory as catalog providers, plus RetainDB, ByteRover and Memori outside the catalog. Only one external provider can be active at a time.

### What is the difference between plugins.enabled and memory.provider in Hermes?

plugins.enabled is the allow-list for general plugins such as tools, hooks and slash commands. Memory providers ignore it: Hermes treats them as exclusive, and the one named in memory.provider is the only one that runs. hermes plugins enable does not activate a memory provider.

