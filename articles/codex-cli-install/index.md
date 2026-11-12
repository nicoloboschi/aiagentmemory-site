# How to Install Codex CLI (2026): macOS, Windows, Linux

> How to install OpenAI Codex CLI on macOS, Linux and Windows with the standalone installer, npm or Homebrew, sign in with ChatGPT, update it, and add Hindsight memory. Commands from the official README, read 2026-11-12.

Source: https://aiagentmemory.org/articles/codex-cli-install/ · Updated 2026-11-12 · aiagentmemory.org


**To install Codex CLI, run one command for your system.** On macOS or Linux, run `curl -fsSL https://chatgpt.com/codex/install.sh | sh`. On Windows, use the PowerShell line below. Then run `codex` in a project folder and choose **Sign in with ChatGPT**. Codex CLI is OpenAI's coding agent that runs in your terminal, on your own machine.

## What is Codex CLI?

**Codex CLI is a coding agent from OpenAI that runs locally.** It reads the files in your project folder, edits them, and runs the tools already on your machine. It is separate from the Codex web agent at chatgpt.com/codex and from the Codex desktop app. The official README lists the IDE extension, the desktop app (`codex app`) and the cloud agent as other ways to use Codex.

## Install commands

Use the command that matches your system. All of them come from the [Codex CLI README on GitHub](https://github.com/openai/codex), read 2026-11-12.

| Route | Command | Platforms |
|---|---|---|
| Standalone installer | `curl -fsSL https://chatgpt.com/codex/install.sh \| sh` | macOS, Linux |
| Standalone installer | `powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 \| iex"` | Windows |
| npm | `npm install -g @openai/codex` | Any system with npm |
| Homebrew | `brew install --cask codex` | macOS |
| Manual binary | Download from the [latest GitHub Release](https://github.com/openai/codex/releases/latest) | macOS (arm64, x86_64), Linux (x86_64, arm64) |

The standalone installers download from `releases.openai.com/codex` by default. If that download fails, they fall back to GitHub Releases. To use GitHub Releases from the start, set `CODEX_INSTALLER_USE_RELEASES_OPENAI_COM=false` before you run the script. The README shows the exact line for each shell.

For the manual route, each archive holds one file with the platform in its name, such as `codex-x86_64-unknown-linux-musl`. Rename it to `codex` after you extract it.

## Steps after install

1. **Open a project folder.** Run `cd` into a repository, then run `codex`. Codex reads the folder it starts in.
2. **Sign in.** On first run, pick **Sign in with ChatGPT**. The docs say that Codex is included in the Plus, Pro, Business, Edu and Enterprise plans. An API key also works, but it needs [additional setup](https://developers.openai.com/codex/auth#sign-in-with-an-api-key).
3. **Give it a first task.** Ask it to explain the project, make one small change, or find a bug. Create a Git commit before a big task so you can go back.
4. **Learn the commands.** `codex exec` runs one task without the chat screen. It is the option for scripts and CI. The Codex docs at [developers.openai.com/codex](https://developers.openai.com/codex) list the rest.

Codex CLI reads its settings from `~/.codex/config.toml`. The [Codex CLI config reference](/articles/codex-cli-config-reference/) lists the keys, with their defaults.

## Update and uninstall

- **Update.** Run the same install command again. The docs list the standalone installer as the update route on macOS and Linux.
- **Uninstall.** For npm, run `npm uninstall -g @openai/codex`. For Homebrew, run `brew uninstall --cask codex`. The README gives no uninstall command for the standalone installer or the manual binary. For those, find the file with `which codex` and delete it.

## Check the install

Run `codex --version` in a terminal. It prints the installed version. If the command is not found, open a new terminal window and try again, since the install may not be on your `PATH` yet. If it still fails, run the install again with the same route.

## Memory: built in and add-ons

Codex has two built-in memory layers. **AGENTS.md** files hold rules you write, and Codex loads them at the start of every run. **Codex memories** are off by default. When you turn them on, Codex turns useful context from past chats into local files under `~/.codex/memories/`. The full rules, limits and config keys are in [Codex memory: AGENTS.md, memories and add-ons](/articles/codex-memory/).

Built-in memory stays on one machine. For searchable memory that several agents share, add a hook-based plugin.

**Hindsight** is open-source memory for AI agents, with a server you can run yourself. Its Coding Agents package covers 22 command-line agents, Codex CLI and Claude Code among them, from one install command. Per the [Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12:

- `npx @vectorize-io/hindsight-coding-agents install codex` writes three hooks to `~/.codex/hooks.json` and an `[mcp_servers]` entry to `config.toml`. The Codex hooks need `codex_hooks = true` in that file.
- Memory goes into one bank per repository, `coding-agent::{gitProject}`. Claude Code and Codex working on the same repo share that bank.
- The memory can live in Hindsight Cloud, on a server you run, or in a local daemon.

**Limits to know:** the Codex hooks only work when `codex_hooks = true` is set, and the memory needs a Hindsight server (cloud, your own, or a local daemon) to be running. For a single small project, built-in AGENTS.md may be enough. The per-agent Claude Code plugin is retired. The same package covers Claude Code now, as described in [Claude Code memory](/articles/claude-code-memory/).

Hindsight is open source at [github.com/vectorize-io/hindsight](https://github.com/vectorize-io/hindsight).

## Related

- [How to install Claude Code](/articles/install-claude-code/): the same steps for the other terminal coding agent.
- [How to install Gemini CLI](/articles/gemini-cli-install/): Google's terminal agent, with its GEMINI.md memory files.
- [Codex memory](/articles/codex-memory/): AGENTS.md, memories and how to add Hindsight.
- [Codex CLI config reference](/articles/codex-cli-config-reference/): the keys in `config.toml`.


## FAQ

### How do I install Codex CLI?

On macOS or Linux, run curl -fsSL https://chatgpt.com/codex/install.sh | sh. On Windows, run powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 | iex". You can also run npm install -g @openai/codex or brew install --cask codex. Then run codex in a project folder.

### How do I sign in to Codex CLI?

Run codex and choose Sign in with ChatGPT. A Plus, Pro, Business, Edu or Enterprise plan includes Codex. You can also sign in with an API key, which needs extra setup.

### How do I add memory to Codex CLI?

Codex has built-in AGENTS.md files and optional Codex memories. For searchable memory shared across repos and agents, install Hindsight with npx @vectorize-io/hindsight-coding-agents install codex. It writes hooks to ~/.codex/hooks.json and needs codex_hooks = true.

