# Hermes Agent config.yaml Reference: Main Sections and Keys (2026)

> The main sections and keys of Hermes Agent's config.yaml, with the example value and a one-line meaning for each. Covers model, memory, compression, terminal, agent and delegation. Checked against the Hermes docs and its example config on 2026-11-26.

Source: https://aiagentmemory.org/articles/hermes-agent-config-reference/ · Updated 2026-11-26 · aiagentmemory.org


Hermes Agent keeps its settings in `~/.hermes/config.yaml`. Secrets (API keys, bot tokens) go in `~/.hermes/.env`. The config file wins for every non-secret setting when both are set.

This page lists the main sections and the keys people change most. The values are the ones in Hermes's own example file, read on 2026-11-24. The file has 28 top-level sections and many more child keys. The first table lists all 28 sections. The key table covers only the main child keys, not the full list.

## Where settings come from

From highest priority to lowest (Hermes docs, Configuration Precedence):

1. CLI flags, for one run, such as `hermes chat --model ...`
2. `~/.hermes/config.yaml`
3. `~/.hermes/.env`, the fallback for environment variables
4. Built-in defaults

Useful commands, from the Hermes docs:

- `hermes config get model` prints one value
- `hermes config set terminal.backend docker` writes one value. Names in capitals (such as `OPENROUTER_API_KEY`) go to `.env` instead
- `hermes config check` finds missing or stale options
- `hermes config migrate` adds missing options

## Top-level sections

| Section | What it controls |
|---|---|
| `_config_version` | Schema version. Hermes manages it; don't edit it by hand |
| `database` | SQLite journal mode for the state database (`wal` or `delete`) |
| `runtime` | Process limits for the gateway and `hermes serve` |
| `attachments` | Where uploaded files are staged |
| `plugins` | Plugin install settings, such as the Git clone timeout |
| `model` | Which model and provider to use |
| `kanban` | Kanban board worker settings |
| `cron` | Scheduled jobs, such as catch-up after downtime |
| `terminal` | Where the agent's shell commands run (local, Docker and others) |
| `browser` | Browser tool settings |
| `tool_loop_guardrails` | Limits and warnings for repeated tool calls |
| `compression` | When and how old context is compacted |
| `memory` | Built-in notes (MEMORY.md, USER.md) and their size limits |
| `max_concurrent_sessions` | Cap on chat sessions active at once across CLI, TUI and gateway |
| `group_sessions_per_user` | In group chats, one session per person (`true`) or one shared session |
| `hosted_rooms` | Shared rooms and their discussion limits (`max_rounds`, `max_messages`) |
| `streaming` | Streams replies to chat platforms as they are written. Off in the example (`enabled: false`) |
| `skills` | Skill settings, including how often the agent is nudged to create skills |
| `agent` | Turn limits, reasoning effort and gateway timeouts |
| `gateway` | Settings for the messaging gateway |
| `platform_toolsets` | Which toolsets each platform (CLI, Telegram, Discord, Slack and others) gets |
| `stt` | Speech-to-text settings |
| `code_execution` | Limits for code run through the agent |
| `delegation` | Limits for sub-agents |
| `display` | What the interface shows: busy-input mode, streaming, skin, bell |
| `telemetry` | Telemetry settings |
| `auth` | Login flow for provider accounts |
| `updates` | Update checks and backups |

## Keys people change most

Example values are from `cli-config.yaml.example` in the Hermes repo. Where that file states a default, the table says so.

| Key | Example value | What it does |
|---|---|---|
| `model.default` | `anthropic/claude-opus-4.6` | The model to use. `--model` overrides it for one run |
| `model.provider` | `"auto"` | Provider. `auto` picks one from the credentials you have. Other values include `openrouter`, `anthropic`, `gemini` and `custom` |
| `model.base_url` | `https://openrouter.ai/api/v1` | API address. Used with the `custom` provider for local or self-hosted servers |
| `model.streaming` | `true` (commented out in the example) | Streams replies from the provider. Set `false` to turn streaming off for the whole session |
| `terminal.backend` | `"local"` | Where shell commands run |
| `terminal.cwd` | `"."` | Working folder for the local backend |
| `terminal.timeout` | `180` | Timeout value. The example file does not explain it further |
| `terminal.docker_mount_cwd_to_workspace` | `false` | Mounts the launch folder into Docker. Off by default for security |
| `compression.enabled` | `true` | Turns automatic context compression on |
| `compression.threshold` | `0.50` | Compression starts at this share of the context limit. Models under 512K are floored at 0.75 |
| `compression.protect_last_n` | `20` | Most recent messages kept intact. The file's default is 20, about 10 full turns |
| `compression.protect_first_n` | `3` | Number of first messages that compression never touches |
| `compression.checkpoint_required` | `false` | If `true`, compaction stops unless the memory provider confirms a checkpoint |
| `memory.memory_enabled` | `true` | Turns on the agent's own notes (MEMORY.md) |
| `memory.user_profile_enabled` | `true` | Turns on the user profile notes (USER.md) |
| `memory.memory_char_limit` | `2200` | Size limit for MEMORY.md, about 800 tokens |
| `memory.user_char_limit` | `1375` | Size limit for USER.md, about 500 tokens |
| `memory.nudge_interval` | `10` | Reminds the agent to save memories every N user turns. `0` turns it off |
| `agent.max_turns` | `500` | Maximum tool-calling steps in one conversation |
| `agent.reasoning_effort` | `"medium"` | How hard the model thinks, for models that support it |
| `delegation.max_iterations` | `250` | Maximum tool-calling turns for each sub-agent |
| `cron.catch_up_missed` | `true` | Runs recurring jobs missed during planned downtime. `false` skips past-grace misses |
| `display.busy_input_mode` | `interrupt` | What happens when you type while the agent is busy: `interrupt` (default), `queue` or `steer` |
| `display.streaming` | `true` | Shows the reply as it is written |
| `display.skin` | `default` | Visual theme of the interface |
| `updates.check` | `true` | Checks for new Hermes versions |
| `updates.backup_keep` | `5` | Number of pre-update backup zips kept |
| `max_concurrent_sessions` | `null` | No cap on active sessions. `null`, `0` or omitting it means unlimited |
| `group_sessions_per_user` | `true` | Each person in a group chat gets their own session |

## More keys, one level down

These are second-level keys from the same example file, read on 2026-11-26. Values are the example's. Where the example comments the line out, the value is marked "(commented)": the file shows it as a sample, not an active setting. "Unset" means the example leaves the key out.

| Key | Example value | What it does |
|---|---|---|
| `database.journal_size_limit` | `67108864` (commented) | Caps the SQLite WAL file size, in bytes (64 MB). Unset uses SQLite's default |
| `database.wal_autocheckpoint` | `1000` (commented) | Pages written before SQLite folds the WAL file back in. Unset uses SQLite's default |
| `plugins.clone_timeout_seconds` | `300` | Time limit for each Git clone, fetch or checkout when a plugin installs |
| `model.context_length` | unset | Total context window, input and output together. Leave unset so Hermes detects it; set it only when detection is wrong |
| `model.ollama_num_ctx` | unset | Ollama only: the context size sent on each request. Set it to pin VRAM use |
| `terminal.container_cpu` | `1` | CPU cores for the Docker sandbox |
| `terminal.container_memory` | `5120` | Sandbox memory in MB (5 GB) |
| `terminal.container_disk` | `51200` | Sandbox disk in MB (50 GB) |
| `terminal.container_persistent` | `true` | Keeps the sandbox filesystem between sessions. `false` makes it temporary |
| `browser.inactivity_timeout` | `120` | Seconds without activity before a browser session closes |
| `tool_loop_guardrails.warnings_enabled` | `true` | Warns the agent when it keeps repeating a failing tool call |
| `tool_loop_guardrails.hard_stop_enabled` | `false` | Stops the run on repeated tool calls. Off in the example |
| `compression.target_ratio` | `0.20` | Share of the threshold kept as the most recent messages |
| `compression.max_attempts` | `3` | Compression retries before a turn fails with "max compression attempts reached" |
| `compression.idle_compact_after_seconds` | `0` | When above 0, compacts old history after a session resumes following this many idle seconds. `0` is off |
| `streaming.edit_interval` | `0.3` (commented) | Seconds between message edits while a reply streams to a chat platform |
| `skills.creation_nudge_interval` | `15` | Reminds the model to save a skill every N tool-calling steps. `0` turns it off |
| `agent.gateway_timeout` | `1800` (commented) | Inactivity limit for gateway runs, in seconds. `0` means no limit |
| `agent.api_max_retries` | `3` (commented) | Retries for API errors (dropped connections, timeouts, 5xx) before the failure is shown |
| `agent.verify_on_stop` | `false` (commented) | After code edits without a fresh check, nudges the agent to verify before it finishes. `true` turns this on everywhere |
| `agent.service_tier` | `""` | Fast mode on providers that offer it, such as OpenAI Priority Processing. Empty means normal speed, and fast mode costs more |
| `code_execution.timeout` | `300` | Seconds a script may run before it is stopped |
| `code_execution.max_tool_calls` | `50` | Tool calls a single script may make |
| `delegation.max_concurrent_children` | `10` (commented) | Sub-agents that may run in parallel in one batch |
| `delegation.max_spawn_depth` | `1` (commented) | How many levels of sub-agents may start sub-agents. Allowed range 1 to 3 |
| `stt.provider` | `"local"` (commented) | Speech-to-text engine. Auto-detected if omitted |
| `display.show_reasoning` | `true` (commented) | Shows the model's thinking above each response |
| `display.bell_on_complete` | `false` | Terminal bell when the agent finishes. Works over SSH |
| `display.timestamps` | `false` (commented) | Adds `[HH:MM]` to user and assistant messages |
| `auth.codex_login_flow` | `device_code` | How `hermes auth add openai-codex` signs in: a code to enter (default) or a browser |
| `auth.adopt_external_logins` | `true` | Reuses Codex CLI and Claude Code logins when Hermes has none of its own |
| `updates.pre_update_backup` | `quick` (commented) | Backup before `hermes update`: `quick` (default), `full` (also a zip of the whole Hermes home folder) or `off` |
| `updates.non_interactive_local_changes` | `"stash"` | For updates run without a terminal: `stash` saves local source edits and restores them; `discard` drops them |

## Nested blocks

These are the blocks with their own keys. Same source and date as above. "(commented)" means the example comments the line out.

| Key | Example value | What it does |
|---|---|---|
| `compression.model_thresholds` | unset (commented) | Per-model compression trigger. Keys match part of the model name; the longest match wins. The example shows `"gpt-5": 0.30` as a sample |
| `compression.threshold_tokens` | `200000` (commented) | Absolute token cap for compression. Compression fires at the lower of this and the ratio threshold. Null or unset means off |
| `skills.external_dirs` | unset (commented) | Extra folders of skills, read-only. New skills are still written to `~/.hermes/skills/`. Local skills win on name clashes |
| `hosted_rooms.discussion.max_rounds` | `3` | Rounds of discussion in a hosted room |
| `hosted_rooms.discussion.max_messages` | `10` | Total visible bot replies in a room, across all members |
| `platform_toolsets` | `cli: [hermes-cli]`, `telegram: [hermes-telegram]` and so on | Which toolset each platform gets: CLI, Telegram, Discord, WhatsApp, Slack, Signal and QQ Bot |
| `agent.personalities` | `{}` (commented sample) | Named personas. Each has a `system_prompt`, `tone` and `style`. Empty in the example |
| `stt.enabled` | `true` | Turns speech-to-text on |
| `stt.language` | `"en"` | Language hint for speech-to-text. A provider's own language setting wins |
| `stt.local.model` | `"base"` | Whisper model for local speech-to-text: `tiny`, `base`, `small`, `medium`, `large-v3` or `turbo` |
| `stt.local.vad` | `true` (commented) | Silero filter that keeps silence away from Whisper |
| `stt.openai.model` | `"whisper-1"` | OpenAI transcription model. Other options in the file include `gpt-4o-transcribe` |
| `stt.openai.timeout` | `60` | Request timeout in seconds |
| `stt.openai.max_retries` | `1` | Retries on transport errors. `0` turns them off |
| `stt.providers` | unset (commented) | Your own speech command. The example runs `custom-asr` with `{input_path}` and `{output_path}` filled in |
| `delegation.backend` | `"default"` (commented) | How sub-agent tasks are routed. `default` uses the keys in this section |
| `delegation.model` | unset (commented) | Model for sub-agents. Empty means the parent's model. The file advises a cheaper model here, since sub-agents use most tokens |
| `delegation.provider` | unset (commented) | Provider for sub-agents. Supported in the file: `openrouter`, `nous`, `zai`, `kimi-coding` and `minimax` |
| `delegation.fallback_providers` | unset (commented) | Fallback chain for sub-agents. `[]` turns fallback off |
| `delegation.orchestrator_enabled` | `true` (commented) | `false` makes every sub-agent a leaf that cannot start more |
| `delegation.subagent_auto_approve` | `false` (commented) | When a sub-agent hits an approval prompt: `false` denies it, `true` approves it once. Set `true` only for cron or batch jobs |
| `delegation.inherit_mcp_toolsets` | `true` (commented) | Sub-agents keep the parent's MCP tools when their toolsets are narrowed. `false` makes the match strict |

These tables cover the keys people change most, not every key in the file. Still not listed: the `display` skin and banner keys, the `telemetry` keys, the `delegation.backends`, `delegation.profiles` and `model.providers` blocks, and the other `stt` providers (Groq, Mistral, xAI, DeepInfra).

## The memory section and Hindsight

The `memory` block in `config.yaml` only controls Hermes's built-in notes. Those are two plain files in `~/.hermes/memories/`: `MEMORY.md` for the agent's own notes and `USER.md` for what it knows about you. The two size limits cap how much of each file goes into the prompt.

Hindsight is a different thing. It is a memory provider plugin, installed with `hermes plugins install hindsight`. Its settings live in `~/.hermes/hindsight/config.json`, not in `config.yaml`. The Hermes plugin docs list these modes and tools:

- Modes: `cloud` (the default), `local_embedded` (a local daemon with built-in Postgres; it needs an LLM key and stops after 5 minutes idle) and `local_external`.
- `memory_mode`: `hybrid` (the default: automatic context plus tools), `context` or `tools`.
- Tools: `hindsight_retain`, `hindsight_recall` and `hindsight_reflect`.
- `bank_id_template` can give each profile or repository its own memory bank.
- Update the plugin with `hermes plugins update hindsight`. Running the install command again does not update it.

**Where Hindsight is strongest.** On long histories it leads Mem0 on the BEAM benchmark: 73.9 against 64.1 at 1M tokens, and 64.1 against 48.6 at 10M tokens. These numbers come from Mem0's own comparison page, not from Hindsight's marketing. The same page shows Hindsight and Mem0 close on LongMemEval (94.6 and 94.4) and LoCoMo (92 and 92.5). The graph, temporal recall and consolidation parts are open source.

**Limits.**
- Hindsight uses more tokens per retrieval on Mem0's measurement: about 27K at BEAM 10M, against under 7K for Mem0. Mem0 measured this, not Hindsight.
- The Hindsight v1.2.0 release notes say a crash while the server is down loses the Hermes retain backlog, because the backlog is kept in memory only (PR #6013).
- The `local_embedded` mode stops after 5 minutes idle.

Sources: [Hermes configuration docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration), [Hermes example config](https://github.com/NousResearch/hermes-agent/blob/main/cli-config.yaml.example), [Hermes memory docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers), [Hindsight GitHub](https://github.com/vectorize-io/hindsight). The BEAM and LongMemEval numbers are from Mem0's own comparison page. Benchmarks are vendor-reported.

## Related pages

- [Hermes Agent memory](/articles/hermes-agent-memory/): how MEMORY.md and USER.md work
- [Hermes Agent plugins](/articles/hermes-agent-plugins/): the plugin catalog and install commands
- [Hermes Agent guide](/articles/hermes-agent/)


## FAQ

### Where is the Hermes Agent config file?

It is ~/.hermes/config.yaml. Secrets such as API keys go in ~/.hermes/.env. Run `hermes config` to see the current values and `hermes config edit` to open the file.

### What is the default context compression threshold in Hermes?

The example config sets compression.threshold to 0.50, which means compression starts at 50% of the model's context limit. The file says models with context windows below 512K are floored at 0.75, so for those models the 0.50 value does not apply.

### Does config.yaml hold the Hindsight settings?

No. The Hindsight plugin reads ~/.hermes/hindsight/config.json. The memory block in config.yaml only turns the built-in notes (MEMORY.md and USER.md) on or off and sets their size limits.

