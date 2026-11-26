# OpenClaw Config File: Main Keys and Defaults (2026)

> The main sections of OpenClaw's openclaw.json, with the documented default and a one-line meaning for the keys people change most. Checked against OpenClaw's configuration docs on 2026-11-26, including the session, cron, hooks, logging, messages and talk keys.

Source: https://aiagentmemory.org/articles/openclaw-config-reference/ · Updated 2026-11-26 · aiagentmemory.org


OpenClaw reads its settings from `~/.openclaw/openclaw.json`. The file is JSON5, and every field is optional. Leave a field out and OpenClaw uses its built-in default.

This page lists the top-level sections and the keys people change most. Defaults come from OpenClaw's configuration docs, read on 2026-11-25. The docs spread the full key list over about 20 subpages, so this page is not the full list. The reference page of the docs says the same: it points to the subpage for each area.

## How to change the file

OpenClaw has three ways to edit the file, and the docs recommend the first two for most people:

- `openclaw onboard` runs the setup wizard. `openclaw configure` edits a few chosen settings.
- `openclaw config get agents.defaults.workspace` reads one key. `openclaw config set agents.defaults.userTimezone "Europe/Vienna"` writes one key and keeps the rest of the file. `openclaw config unset <path>` removes one key.
- Edit the file by hand. The Gateway watches the file and applies most changes right away (hot reload).

The Control UI at `http://127.0.0.1:18789` also has a Config tab. It builds a form from the live schema, with a raw JSON editor as an escape hatch. To check what a key accepts, run `openclaw config schema`.

## Top-level sections

| Section | What it controls | Docs page |
|---|---|---|
| `agents` | Agent defaults, per-agent overrides, multi-agent routing, sessions, messages and Talk | Configuration: agents |
| `channels` | Access rules and pairing for each chat app (Discord, Telegram, WhatsApp and others) | Configuration: channels |
| `tools` | Which tools are on, tool policy and custom model providers | Configuration: tools |
| `models` | Model routing and the model catalog (`models.catalogRefresh` turns catalog downloads on or off) | Configuration: runtime basics |
| `gateway` | Port, bind address, auth, roles, Control UI, TLS and reload | Configuration: gateway |
| `memory` | Memory search and its embedding provider | Memory configuration reference |
| `plugins` | Plugin entries and their own settings | Configuration: MCP, skills and plugins |
| `skills` | Skill limits, such as the size cap on the skills prompt | Configuration: MCP, skills and plugins |
| `hooks` | Outside HTTP calls that start or feed agent runs | Configuration: hooks |
| `cron` | Scheduled automations, their delivery and failure alerts | Configuration: automations |
| `session` | Session lifecycle, reset rules and retention | Configuration: agents |
| `messages` | How replies are delivered, queued and read aloud | Configuration: agents |
| `talk` | Voice (Talk) mode settings | Configuration: agents |
| `update` | Version checks and automatic updates | Configuration: runtime basics |
| `acp` | The ACP runtime that dispatches agent sessions | Configuration: runtime basics |
| `secrets` and `env` | Secret providers, secret references and inline environment variables | Configuration: environment and secrets |
| `logging`, `diagnostics`, `audit`, `telemetry` | What gets logged, traced or exported | Configuration: audit, logging and telemetry |
| `database` and `storage` | SQLite (default) or PostgreSQL, and named storage locations | Database engines |

## Agent defaults

These keys sit under `agents.defaults`. Each one can be overridden for a single agent under `agents.entries.<id>`.

| Key | Default | What it does |
|---|---|---|
| `workspace` | `~/.openclaw/workspace` (or the `OPENCLAW_WORKSPACE_DIR` value) | The folder the agent works in and where its bootstrap files live |
| `contextInjection` | `"always"` | When workspace bootstrap files are put into the prompt |
| `bootstrapMaxChars` | `20000` | Characters per bootstrap file before it is cut short |
| `bootstrapTotalMaxChars` | `60000` | Characters for all bootstrap files together |
| `imageMaxDimensionPx` | `1200` | Largest side, in pixels, of an inbound image |
| `imageQuality` | `"auto"` | Image resize level. In auto mode, many images in one turn are compressed more |
| `userTimezone` | not set | The timezone the agent uses for dates (example in the docs: `America/Chicago`) |
| `model` | not set | The main model, with a `primary` and a list of `fallbacks`. The docs example uses `anthropic/claude-opus-4-6` |
| `utilityModel` | not set | A cheaper model for short internal jobs, such as session titles and transcript summaries |
| `timeoutSeconds` | 48 hours per attempt | How long one model attempt may run. Each fallback gets a fresh budget. `0` means no limit |

## Gateway, update and ACP

| Key | Default | What it does |
|---|---|---|
| `gateway.mode` | `"local"` | `local` runs the Gateway on this machine. `remote` connects to another one |
| `gateway.port` | `18789` | Port for the WebSocket and HTTP server |
| `gateway.bind` | `loopback` on a regular host | Which addresses accept connections. Non-loopback binds require Gateway auth |
| `gateway.auth.mode` | token (required by default) | `none`, `token`, `password` or `trusted-proxy`. The onboarding wizard makes a token |
| `gateway.tls.enabled` | `false` | Turns on HTTPS and WSS at the Gateway listener |
| `update.checkOnStart` | `true` | Checks for a new version at start and at most once every 24 hours. Set `false` to turn the check off |
| `update.auto.enabled` | `false` | Background auto-update for stable, beta and dev installs. Needs `checkOnStart` on |
| `acp.enabled` | `true` | Global switch for ACP agent sessions |
| `acp.dispatch.enabled` | `true` | Separate switch: `false` keeps ACP commands but blocks running them |

Anonymous feature statistics are sent only when `telemetry.enabled` is `true`. The runtime docs say that setting `update.checkOnStart` to `false`, or setting `OPENCLAW_NO_AUTO_UPDATE=1`, stops all automatic update requests and feature statistics.

## Session, cron and hooks keys

These come from the OpenClaw docs pages for sessions, automation and hooks, read 2026-11-26. Only keys with a default stated on the page are listed.

| Key | Default | What it does |
|---|---|---|
| `session.scope` | `"per-sender"` | How direct chats are grouped into sessions |
| `session.groupScope` | `"per-group"` | How groups, rooms and channels are grouped |
| `session.reset` | `none` | Automatic session reset. `none` is the default, so sessions do not reset on their own. Other policy: `daily` or `idle` |
| `session.maintenance.mode` | `enforce` | `enforce` cleans up old session entries. `warn` only reports them |
| `session.maintenance.pruneAfter` | `30d` | Age at which a stale session entry is removed |
| `session.maintenance.maxEntries` | `5000` | Cap on session entries that are not archived |
| `session.maintenance.maxDiskBytes` | `10gb` | Disk budget for one agent's session store |
| `session.maintenance.coldStorage.afterDays` | `30` | Days of inactivity before a transcript can move to compressed cold storage. `coldStorage.enabled` defaults to `true` |
| `cron.enabled` | `true` | Runs the stored automation jobs |
| `cron.skipMissedJobs` | `false` | When `true`, skips recurring slots missed while the Gateway was down |
| `cron.sessionRetention` | `24h` | How long finished isolated cron run sessions are kept. `false` or `"0h"` turns cleanup off |
| `cron.failureAlert.after` | `1` | Consecutive failures before a failure alert is sent |
| `cron.failureAlert.cooldownMs` | `3600000` | Minimum gap between failure alerts, in milliseconds (1 hour) |
| `hooks.enabled` | `false` | Turns on the HTTP hook endpoints. Needs a non-empty `hooks.token` |
| `hooks.path` | `/hooks` | Base path for the hook endpoints |
| `hooks.allowRequestSessionKey` | `false` | Lets a caller pass its own session key. Off by default |
| `hooks.mappings` | `[]` | Ordered list of hook mappings. The first match wins |

The same docs say `hooks.allowedAgentIds` is unrestricted when unset, and that `[]` denies all agents. The `session` and `cron` pages also list many keys with no default stated (for example `session.store` and `cron.webhookToken`). They are not in this table.

## Logging, messages and talk keys

From OpenClaw's observability, messages-and-talk docs, read 2026-11-26. Only keys with a default or example stated on the page are listed.

| Key | Default or example | What it does |
|---|---|---|
| `logging.level` | `"info"` | Log level written to the log file |
| `logging.file` | unset: a dated file in the state folder's `logs/` | A fixed log path. Dated default files are deleted after 24 hours |
| `logging.consoleLevel` | `"info"` | Log level shown in the console. Becomes `debug` when you run with `--verbose` |
| `logging.consoleStyle` | `"pretty"` | Console format: `pretty` or `json`. The old `compact` value is retired |
| `logging.maxFileBytes` | `104857600` (100 MB) | Size at which the log file rotates. Up to five numbered archives are kept |
| `logging.redactPatterns` | default list | Regexes that mask secrets in console output and log tails. Setting it replaces the default string list |
| `messages.responsePrefix` | `"🦞"` (example) | Text put in front of replies. `"auto"` is the other option |
| `messages.ackReaction` | `"👀"` | Emoji added to a message to show it was received. `""` turns it off |
| `messages.ackReactionScope` | `"group-mentions"` | Where the acknowledgment reaction appears. Other values: `group-all`, `direct`, `all`, `off` |
| `messages.queue.mode` | `"steer"` | What happens to a new message while a run is active. `steer` adds it to the run. `followup` runs it after the run ends. `collect` and `interrupt` are also listed |
| `messages.queue.cap` | `20` | Queued messages allowed before the drop rule applies |
| `messages.queue.drop` | `"summarize"` | Which queued messages go when the cap is reached. `old` drops the oldest, `new` rejects the newest |
| `messages.queue.allowCollaborativeSteering` | `true` | Lets other people's messages steer the active run. `false` makes them wait for their own turn |
| `messages.inbound.debounceMs` | unset | Groups quick text messages from one sender into one turn. Telegram waits 300 ms when unset. `0` turns it off |
| `messages.tts.auto` | `"off"` | Automatic spoken replies: `off`, `always`, `inbound` or `tagged` |
| `talk.silenceTimeoutMs` | unset: 700 ms on macOS and Android, 900 ms on iOS | Pause after the user stops speaking before Talk sends the transcript |
| `talk.speechLocale` | unset: device language | Speech recognition language, as a BCP 47 id such as `en-US` |
| `talk.realtime.consultRouting` | `"provider-direct"` | `provider-direct` keeps the provider's own reply. `force-agent-consult` sends the request through OpenClaw |

The docs list `messages.queue.byChannel` too (the example sends WhatsApp and Telegram to `followup`). The `talk.realtime` vadThreshold, silenceDurationMs and prefixPaddingMs keys are documented, but their defaults are "the provider's default", so they are not in the table.

## Memory keys

The memory settings live under the top-level `memory` block. Search defaults are in `memory.search`. Per-agent overrides are in `agents.entries.<id>.memory.search`.

| Key | Default | What it does |
|---|---|---|
| `memory.search.enabled` | `true` | Turns memory search on or off |
| `memory.search.provider` | `"openai"` | The embedding adapter, such as `openai`, `ollama`, `local`, `gemini` or `bedrock`. Set it to use another provider |
| `memory.search.model` | the provider's default | The embedding model name |
| `memory.search.fallback` | `"none"` | Adapter to use if the main one fails |
| `memory.search.rememberAcrossConversations` | on for personal installs; off when DM isolation is set | Lets the agent use context from its other private chats. Groups and channels are never used as sources |

The memory reference also says that this setting needs a memory provider that supports protected transcript recall. The built-in provider does. Other providers may not.

## Memory plugin: Hindsight

OpenClaw can use Hindsight as a memory plugin, `@vectorize-io/hindsight-openclaw`. Its v1.2.0 release, dated 3 November 2026, added the same three tools as the Hermes plugin: `hindsight_retain`, `hindsight_recall` and `hindsight_reflect`. Its settings are:

- `memoryMode` picks `hybrid` (the default: automatic recall plus the tools), `context` (automatic recall only) or `tools` (tools only).
- Deleting from the knowledge base needs `allowKnowledgeDelete: true`.
- Cron, heartbeat and sub-agent sessions stay out of memory unless you set `skipOperationalSessions: false`.

The plugin can run a local embedded daemon, so you do not need a server to try it. This page does not list the plugin's own keys. Read its package README for them.

**The win.** On the BEAM benchmark at 10 million tokens, Hindsight scores 64.1 and Mem0 scores 48.6. At 1 million tokens the scores are 73.9 and 64.1. Both figures come from [Mem0's comparison page](https://mem0.ai/compare/mem0-vs-hindsight). Mem0 ran that comparison, so it is a vendor-run result.

**The limit.** Saving costs more than with plain notes. Retain and consolidation both call an LLM, so every saved turn can add model cost. For a short history, the daily notes and `MEMORY.md` that OpenClaw already keeps may be enough. Hindsight's advantage shows up on long histories.

For how the built-in memory works, see [OpenClaw memory: how it works and which plugins](/articles/openclaw-ai-agent-memory/).

## Sources

- OpenClaw configuration docs: [configuration](https://docs.openclaw.ai/gateway/configuration), [configuration reference](https://docs.openclaw.ai/gateway/configuration-reference), [agents](https://docs.openclaw.ai/gateway/config-agents), [workspace and bootstrap](https://docs.openclaw.ai/gateway/config-agents/workspace-and-bootstrap), [models](https://docs.openclaw.ai/gateway/config-agents/models), [runtime basics](https://docs.openclaw.ai/gateway/config-runtime) and [gateway](https://docs.openclaw.ai/gateway/config-gateway). All read 2026-11-25.
- Logging, messages and talk keys: [observability](https://docs.openclaw.ai/gateway/config-observability) and [messages and talk](https://docs.openclaw.ai/gateway/config-agents/messages-and-talk), read 2026-11-26.
- Session, cron and hooks keys: [sessions](https://docs.openclaw.ai/gateway/config-agents/sessions), [automation](https://docs.openclaw.ai/gateway/config-automation) and [hooks](https://docs.openclaw.ai/gateway/config-hooks), read 2026-11-26.
- Memory reference: [docs.openclaw.ai/reference/memory-config](https://docs.openclaw.ai/reference/memory-config), read 2026-11-25.
- Hindsight plugin: [Hindsight v1.2.0 release notes](https://github.com/vectorize-io/hindsight/releases/tag/v1.2.0), read 2026-11-05.
- BEAM figures: [Mem0 vs Hindsight](https://mem0.ai/compare/mem0-vs-hindsight), vendor-run.


## FAQ

### Where is the OpenClaw config file?

It is ~/.openclaw/openclaw.json, in JSON5 (comments and trailing commas are allowed). If the file is missing, OpenClaw uses safe defaults. Run `openclaw config file` to see the active path.

### What is the default OpenClaw Gateway port and bind?

The Gateway listens on port 18789. The docs give loopback as the default bind on a regular host, so it accepts only local connections. Binding to a non-loopback address requires Gateway auth.

### Which OpenClaw memory setting controls cross-conversation recall?

memory.search.rememberAcrossConversations. The docs say it is on by default for personal installs and off when DM isolation is configured. Set it per agent under agents.entries.<id>.memory.search.

