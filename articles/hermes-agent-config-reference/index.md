# Hermes Agent config.yaml Reference: Main Sections and Keys (2026)

> The main sections and keys of Hermes Agent's config.yaml, with the example value and a one-line meaning for each. Covers model, memory, compression, terminal, agent and delegation. Checked against the Hermes docs and its example config on 2026-11-24.

Source: https://aiagentmemory.org/articles/hermes-agent-config-reference/ · Updated 2026-11-24 · aiagentmemory.org


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

