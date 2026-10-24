# Hermes Agent Desktop: Install, Memory and Setup

> Hermes Agent Desktop is Nous Research's native app for Hermes. How to install it, how it shares memory and sessions with the CLI, the Memory Graph, and limits.

Source: https://aiagentmemory.org/articles/hermes-agent-desktop/ · Updated 2026-10-24 · aiagentmemory.org


**Hermes Agent Desktop** is the official native app for Nous Research's Hermes Agent, for macOS (Apple Silicon), Windows and Linux. It isn't a separate agent: it runs the same core as the `hermes` CLI and reads the same config, sessions, skills and memory. Install it from the Hermes website, or run `hermes desktop` if the CLI is already installed.

[Hermes Agent](https://github.com/NousResearch/hermes-agent) is MIT-licensed and had about 256,500 GitHub stars on October 24, 2026. Everything below comes from the official [Hermes Desktop docs](https://hermes-agent.nousresearch.com/docs/user-guide/desktop) and [installation guide](https://hermes-agent.nousresearch.com/docs/getting-started/installation), checked October 2026, with Hermes at v0.22.5.

## What is Hermes Agent Desktop?

**Hermes Agent Desktop is an Electron app that puts a chat-first window on top of the normal Hermes Agent runtime. In Nous Research's words it uses "same config, same API keys, same sessions, same skills, same memory" as the CLI, and it is "not a separate product or a lightweight clone".**

Nous Research launched it on June 2, 2026 as a public preview at v0.15.2, [Decrypt reported](https://decrypt.co/369952/hermes-ai-agent-official-app-terminal). By default the app starts and manages its own local backend. It can also connect to a Hermes backend on another machine (a VPS, a home server, or a Mini behind Tailscale), as long as `hermes serve` is already running there; the app doesn't start it for you.

Hermes now has four front ends over one state:

| Interface | How you open it | Shares memory and sessions |
|---|---|---|
| Desktop app | Installer, or `hermes desktop` | Yes |
| CLI | `hermes` | Yes |
| TUI | `hermes --tui` | Yes |
| Web dashboard | `hermes dashboard` | Yes |

## How to install Hermes Agent Desktop

There are three routes, per the installation guide and the `install.sh` script:

1. **Packaged app.** Download from the Hermes website: a DMG on macOS (Apple Silicon only) or an `.appinstaller` on Windows.
2. **From an existing CLI install.** Run `hermes desktop`. It installs the Node dependencies, builds the Electron app for your OS, then launches it. The first run downloads Electron (about 114 MB) from GitHub; on a blocked network set `ELECTRON_MIRROR`.
3. **As part of a fresh install.** Pass the desktop flag to the install script:

```bash
# macOS / Linux / WSL2
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash -s -- --include-desktop
```

On native Windows the PowerShell installer (`iex (irm https://hermes-agent.nousresearch.com/install.ps1)`) takes `-IncludeDesktop`.

With Ollama, `ollama launch hermes-desktop` installs Hermes if it's missing, lets you pick a model, points Hermes Desktop at it and opens the app ([Ollama docs](https://docs.ollama.com/integrations/hermes-desktop)). Add `--model <model>` to skip the picker.

Useful `hermes desktop` flags:

| Flag | What it does |
|---|---|
| `--skip-build` | Launch the already-built app |
| `--force-build` | Rebuild even if nothing changed |
| `--build-only` | Build without launching (used by `hermes update`) |
| `--cwd PATH` | Open in a given project directory |
| `--ignore-existing` | Don't start a local backend |

To remove it, `hermes uninstall --gui` deletes only the app, `hermes uninstall` removes the app and agent but keeps your data, and `hermes uninstall --full` removes everything. Add `--dry-run` to preview.

## How memory works in Hermes Desktop

The app uses the same runtime home as a CLI install: `~/.hermes`, or `%LOCALAPPDATA%\hermes` on Windows, with the same layout. So its memory is Hermes's normal memory: `MEMORY.md` (2,200 characters of agent notes) and `USER.md` (1,375 characters about you), loaded into the prompt as a frozen snapshot at session start, plus past sessions in SQLite that the agent searches on demand. The full mechanics, limits and fixes are in [Hermes Agent memory](/articles/hermes-agent-memory/).

What the app adds is a way to see that memory:

- **Memory Graph.** A zoomable node map of the skills and memories Hermes has learned, filterable by All, Used and Learned, with a timeline. Open it from the command palette or the status bar. Its share button exports the layout as a code; the docs say the code contains no memory or skill text.
- **Learning Journey.** `/journey` (also `/learning` or `/memory-graph`) plays back learned skills and memories over time. You can edit or delete nodes from there: a deleted skill is archived, a deleted memory is removed.
- **Memory & Context settings.** One of the config-backed settings pages, next to Model, Workspace, Safety and Tools & Keys. With two or more profiles, an "Applies to" selector picks which profile's `config.yaml` you're editing. Each profile keeps its own config, skills and sessions.

Two consequences follow from the shared state. Memory written in a desktop chat is in the terminal's next session too, and the other way round. And the frozen snapshot still applies: a fact saved mid-chat shows up in the prompt only in the next session, so start a new chat to see it used.

## Adding an external memory provider

The desktop docs don't cover memory providers. Because the app reads the same `config.yaml` and plugins as the CLI, a provider you set up in a terminal is the one the app uses:

```bash
hermes plugins install hindsight   # or honcho, mem0, supermemory, openviking
hermes memory setup                # pick the provider, enter credentials
hermes memory status               # check which one is active
```

Only one external provider runs at a time, next to `MEMORY.md` and `USER.md`. The [Hermes Agent plugins](/articles/hermes-agent-plugins/) page covers how they install and update.

[Hindsight](https://github.com/vectorize-io/hindsight) is one of the listed providers, and it's a good fit for a desktop agent you keep for months. On Mem0's own comparison page, Hindsight leads on the long-history BEAM test: 73.9 vs 64.1 at 1M tokens and 64.1 vs 48.6 at 10M tokens (vendor-run numbers, not independent). They tie on LongMemEval (94.6 vs 94.4). Hindsight is MIT with every feature in the self-hosted server, and its Hermes plugin has a `local_embedded` mode that runs a local daemon with built-in Postgres, so memories stay on your machine. With `bank_id_template` you can give each Hermes profile its own memory bank. The limits: each retain calls an LLM, and the plugin injects up to 4,096 tokens of recall per turn by default (`recall_max_tokens`), which you pay for on every message. If the built-in files already hold what you need, skip providers. For the trade-offs in detail, see [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Official app vs the community Hermes Desktop

Searching "Hermes Desktop" also finds [fathah/hermes-desktop](https://github.com/fathah/hermes-desktop), a "Desktop Companion for Hermes Agent". It's a separate community project, not made by Nous Research.

| | Official Hermes Desktop | fathah/hermes-desktop |
|---|---|---|
| Made by | Nous Research | Community (fathah) |
| Where the code lives | `apps/desktop` in NousResearch/hermes-agent | Its own repository |
| License | MIT | MIT |
| Stars (Oct 24, 2026) | Part of hermes-agent (~256,500) | ~14,400 |

If you want the app that tracks Hermes releases and the docs above, use the official one.

## Known limits

The desktop docs list these:

- **Desktop plugins aren't sandboxed.** Anything in `$HERMES_HOME/desktop-plugins/` runs with the app's full authority, so install only plugins you trust. Memory is an attack surface too; see [AI memory injection](/articles/ai-memory-injection/).
- **Switching models mid-chat resets the prompt cache**, so the next turn costs more.
- **Update checks are rate limited** to 60 GitHub requests per hour without a token; set `GITHUB_TOKEN` or log in with `gh` for 5,000.
- **Wayland quirks**: always-on-top and snap-to-pointer don't work on some native Wayland compositors.
- **macOS builds are Apple Silicon only.**

Logs are in `~/.hermes/logs/desktop.log`; `hermes logs gui -f` follows them. For how Hermes compares with OpenClaw on memory, see [OpenClaw vs Hermes Agent memory](/articles/openclaw-vs-hermes-agent-memory/).


## FAQ

### How do I install Hermes Agent Desktop?

Download the DMG (macOS, Apple Silicon only) or the .appinstaller (Windows) from the Hermes website. If the Hermes CLI is already installed, run hermes desktop, which builds and launches the app. A fresh install can include it with curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash -s -- --include-desktop.

### Does Hermes Desktop share memory with the Hermes CLI?

Yes. Nous Research's docs say the desktop app runs the same agent as the CLI and gateway, with the same config, API keys, sessions, skills and memory, all under ~/.hermes (or %LOCALAPPDATA%\hermes on Windows). A session started in the app can be resumed in the terminal, and the other way round.

### Is Hermes Desktop the same as fathah/hermes-desktop?

No. The official app is built by Nous Research inside the hermes-agent repository and launched on June 2, 2026. fathah/hermes-desktop is a separate, MIT-licensed community companion app with about 14,400 GitHub stars in October 2026.

