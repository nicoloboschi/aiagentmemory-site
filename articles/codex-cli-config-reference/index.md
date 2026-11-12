# Codex CLI Config Reference: config.toml Keys and Defaults (2026)

> Where Codex CLI reads config.toml, which file wins, and the main keys people change: model, approval_policy, sandbox_mode, web_search, memories and MCP. Taken from OpenAI's config reference, November 2026.

Source: https://aiagentmemory.org/articles/codex-cli-config-reference/ · Updated 2026-11-12 · aiagentmemory.org


**Codex CLI reads its settings from `config.toml`.** The main file is `~/.codex/config.toml`. Set a key there and every project uses it. Put a `.codex/config.toml` in a repo to change one project only. This page lists the keys people change most, their allowed values and defaults, using OpenAI's configuration reference (checked 2026-11-10). Built-in memories are one key group; the [Codex memory guide](/articles/codex-memory/) covers how they behave.

## What is the Codex CLI config file?

The Codex CLI config file is a TOML file that sets the model, approval rules, sandbox, web search, memories, MCP servers and lifecycle hooks. OpenAI's Config basics page says you copy only the keys you need into the file. Keys you leave out keep their built-in value, so a short file is normal.

## Where Codex reads config, in order

Codex resolves each value from the first place that sets it, highest first (from OpenAI's Config basics page):

| Rank | Source | Notes |
|---|---|---|
| 1 | Command-line `--config` overrides | Wins over everything else |
| 2 | Project `.codex/config.toml` | Closest to your folder wins; trusted projects only |
| 3 | Profile file from `--profile` | `~/.codex/<name>.config.toml` |
| 4 | User `~/.codex/config.toml` | Your defaults |
| 5 | Cloud-managed defaults | Only when your workspace delivers them |
| 6 | System `/etc/codex/config.toml` | Unix only, if present |

An untrusted project skips its `.codex/` files, including project hooks and rules. User and system config still load.

## Key settings and their values

These are the keys from OpenAI's reference that people change most. Defaults are as the reference states them; where it gives none, the table says so.

| Key | Allowed values | What it does |
|---|---|---|
| `model` | model name, e.g. `gpt-6.1-sol` | The model for the session. The reference gives an example, not a default. |
| `review_model` | model name | Model for `/review`. Defaults to the session model. |
| `model_provider` | id from `model_providers` | Default `openai`. Built-in ids `openai`, `ollama` and `lmstudio` cannot be overridden. |
| `model_context_window` | number | Context tokens for the active model. |
| `approval_policy` | `on-request`, `never`, or a `granular` table | When Codex stops to ask before running a command. `untrusted` is unsupported; `on-failure` is deprecated. |
| `sandbox_mode` | `read-only`, `workspace-write`, `danger-full-access` | What commands may touch on disk and network. |
| `sandbox_workspace_write.network_access` | true or false | Allows outbound network inside the workspace sandbox. |
| `default_permissions` | `:read-only`, `:workspace`, `:danger-full-access`, or a custom name | Named permission profile. Do not combine it with `sandbox_mode`. |
| `web_search` | `disabled`, `cached`, `indexed`, `live` | Default `cached`. Full-access sandboxes default to `live`. |
| `model_reasoning_effort` | `low` to `ultra`, depending on model | Levels depend on the model and client. |
| `project_doc_max_bytes` | number | Largest `AGENTS.md` read into project instructions. |
| `notify` | array of strings | Command run for notifications; it gets a JSON payload. |
| `features.memories` | true or false | Turns on Codex memories. Off by default. |
| `memories.use_memories` | true or false | When false, old memories are not injected into new sessions. Default true. |
| `memories.generate_memories` | true or false | When false, new threads are not used to make memories. Default true. |
| `mcp_servers` | table | Allowlist of MCP servers that may run. A server not on the list is disabled. |
| `hooks` | table | Lifecycle hooks inline in the file, for example `PreToolUse` and `Stop`. |

## Memory keys

Codex memories are on a separate switch. `features.memories` must be true first. The `memories` table then sets how they work. The reference lists these defaults:

- `memories.version`: `v2` for a new home; `v1` if `$CODEX_HOME/memories/` already exists.
- `memories.max_raw_memories_for_consolidation`: 256, capped at 4096.
- `memories.max_unused_days`: 30, clamped to 0 to 365.
- `memories.max_rollout_age_days`: 30, clamped to 0 to 90.
- `memories.min_rollout_idle_hours`: 6, clamped to 1 to 48.
- `memories.min_rate_limit_remaining_percent`: 25.
- `memories.disable_on_external_context`: false. When true, threads with MCP calls or web search stay out of memory.

The memory extraction and consolidation models can be set with `memories.extract_model` and `memories.consolidation_model`. The reference gives no default model for them.

## Custom model providers

Use `[model_providers.<id>]` to point Codex at another API. The main fields are `base_url`, `env_key` (the variable that holds the key), `wire_api` (only `responses` is supported), and retry settings. `request_max_retries` defaults to 4 and `stream_idle_timeout_ms` to 300000. A custom entry replaces the whole provider with that id; fields are not merged.

## Limits of this reference

- The reference is long (about 150 KB of Markdown). This page covers only the keys above. Search the full page for anything else.
- Some keys are for managed or enterprise setups (`requirements.toml`, Work Cloud). They are not part of a personal config.
- Earlier third-party guides list keys that OpenAI's reference no longer names. Trust the reference.

## Where Hindsight fits

Codex's own memories keep short notes that the model writes from past threads. They are local and simple to switch off. Hindsight is a separate memory server that you run or host. It plugs into Codex through its Coding Agents package (`npx @vectorize-io/hindsight-coding-agents install codex`), which also covers Claude Code, Cursor, Copilot and other CLI agents with one memory bank per repo ([docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents)).

- **Where it leads:** on the BEAM 10M benchmark, Hindsight scored 64.1 against 48.6 for Mem0. Mem0 published this comparison on its own site, so it is vendor-run, not independent. Hindsight's own site also reports LongMemEval S at 94.6.
- **Open source:** the full server is MIT-licensed, including graph recall, consolidation and mental models. You can self-host it with Docker.
- **Limits:** Hindsight's writes call an LLM, so they cost more than Codex's built-in memories. Its recall returns more context per query by default: about 27K tokens per retrieval on BEAM 10M, against under 7K for Mem0 by Mem0's measurement. Set the token budget lower if that is too much.

For a Codex-only setup with a small rule set, the built-in memories are enough. For long histories across tools, see [Hindsight's memory guide](/articles/ai-memory-hindsight/) and [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Related pages

- [Codex memory: AGENTS.md, memories and add-ons](/articles/codex-memory/) explains how the memory keys act at runtime.
- [Claude Code settings](/articles/claude-code-settings/) is the same kind of reference for Claude Code.

## Sources

- OpenAI, [Codex configuration reference](https://developers.openai.com/codex/config-reference), read 2026-11-10.
- OpenAI, [Codex config basics](https://developers.openai.com/codex/config-basic), read 2026-11-10 (precedence order).
- Hindsight facts (benchmarks, token figures, plugin list): Hindsight README and Mem0's [Hindsight comparison](https://mem0.ai/compare/mem0-vs-hindsight), checked 2026-10-12 and 2026-11-08.


## FAQ

### Where is the Codex CLI config file?

Your personal settings live in ~/.codex/config.toml. A project can add .codex/config.toml files, but Codex loads them only when you trust the project. On Unix, /etc/codex/config.toml is the system file.

### Which Codex config setting wins?

Command-line --config overrides win first, then project files (closest to the working folder wins), then a --profile file, then ~/.codex/config.toml, then cloud-managed defaults, then the system file. Check the precedence list in OpenAI's Config basics page before you debug a value.

### How do I turn on Codex memories?

Set features.memories = true under the [features] table in config.toml. It is off by default. Codex then uses memories.generate_memories and memories.use_memories, both on by default once the feature is on.

### What are the safe Codex approval and sandbox settings?

approval_policy accepts on-request or never, plus a granular table. sandbox_mode accepts read-only, workspace-write or danger-full-access. untrusted is no longer supported, and on-failure is deprecated.

