# How to Install Claude Code (2026): macOS, Windows, Linux

> How to install Claude Code on macOS, Windows, WSL and Linux with the native installer, Homebrew, WinGet or npm, check the install, log in, update, and uninstall. Commands from the official setup docs, read 2026-11-12.

Source: https://aiagentmemory.org/articles/install-claude-code/ · Updated 2026-11-12 · aiagentmemory.org


## What is Claude Code?

**Claude Code** is Anthropic's coding agent that runs in your terminal. It reads your project, edits files and runs commands. You install it once on your machine, then start it with `claude` in any project folder. Its memory comes from files such as CLAUDE.md and from optional add-ons.

## How do you install Claude Code?

Open a terminal and run the command for your system. Anthropic's native installer is the recommended route, and it keeps itself up to date.

| System | Command | Updates |
|---|---|---|
| macOS, Linux, WSL | `curl -fsSL https://claude.ai/install.sh \| bash` | Automatic, in the background |
| Windows PowerShell | `irm https://claude.ai/install.ps1 \| iex` | Automatic, in the background |
| Windows CMD | `curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd` | Automatic, in the background |
| Homebrew (macOS, Linux) | `brew install --cask claude-code` | Manual: `brew upgrade claude-code` |
| WinGet (Windows) | `winget install Anthropic.ClaudeCode` | Manual: `winget upgrade Anthropic.ClaudeCode` |
| npm (any system with Node.js 22+) | `npm install -g @anthropic-ai/claude-code` | Manual: `npm install -g @anthropic-ai/claude-code@latest` |

Anthropic's setup docs also list apt, dnf and apk repositories for Debian, Ubuntu, Fedora, RHEL and Alpine. Those installs need a manual upgrade too. Do not use `sudo` with the npm install.

The native installer needs no admin rights, and on Windows you do not need to run as Administrator. Npm is the route that most often hits permission errors, so use it only if the others fail.

## What do you need before you install?

The official requirements, from the [advanced setup docs](https://code.claude.com/docs/en/setup):

- **Operating system:** macOS 13.0 or later, Windows 10 (1809) or later, Ubuntu 20.04 or later, Debian 10 or later, or Alpine Linux 3.19 or later.
- **Hardware:** 4 GB of RAM or more, on an x64 or ARM64 processor.
- **Network:** an internet connection.
- **Git:** optional. On native Windows, [Git for Windows](https://git-scm.com/downloads/win) lets Claude Code use the Bash tool. Without it, Claude Code uses PowerShell instead.
- **Account:** a Pro, Max, Ultra, Team, Enterprise or Console plan. The free plan has no Claude Code access.

On Windows you can run Claude Code natively or inside WSL 2. WSL 2 supports sandboxed command execution. Native Windows does not.

## How do you check the install?

Open a new terminal, because the installer adds Claude Code to your PATH. Then run:

```bash
claude --version
```

A working install prints a version number such as `2.1.211 (Claude Code)`. If you see `command not found`, the install folder is not on your PATH yet. For a deeper check, run `claude doctor`. It prints read-only diagnostics, including install health, settings errors and update status, without starting a session.

## How do you log in and start?

Run `claude` in your project folder. Claude Code opens a browser login the first time. If you have set the `ANTHROPIC_API_KEY` environment variable and approve that key when asked, it skips the login.

## How do you update Claude Code?

Run `claude update` to update now. Native installs check for updates on startup and install them in the background. The new version starts the next time you open Claude Code.

You can choose a release channel with the `autoUpdatesChannel` setting. `"latest"` is the default. `"stable"` serves a version that is typically about a week old and skips releases with major regressions. To turn auto-updates off, set `DISABLE_AUTOUPDATER` to `"1"` under `env` in your settings file.

## How do you uninstall Claude Code?

For a native install on macOS or Linux, remove the binary and the version files:

```bash
rm -f ~/.local/bin/claude
rm -rf ~/.local/share/claude
```

Use the matching command for Homebrew, WinGet, npm or your Linux package manager. Removing `~/.claude` and `~/.claude.json` also deletes your settings, MCP server setup and session history, so back those up first. Check for a second install if `claude` still runs after you remove it.

## Where does memory fit after the install?

Claude Code's own memory needs no install. Put build commands and rules in CLAUDE.md, and Claude Code also keeps short auto-memory notes. Both are described in [Claude Code memory](/articles/claude-code-memory/), and the hook events that save memory are in [Claude Code hooks](/articles/claude-code-hooks-memory/).

Built-in memory stays on one machine and does not search past sessions. Add-ons fill that gap. Hindsight, the open-source memory server from Vectorize, is one of them:

- **Install path.** The Hindsight plugin for Claude Code is retired. Its replacement is the npm package `@vectorize-io/hindsight-coding-agents`. Run `npx @vectorize-io/hindsight-coding-agents install claude-code` from your repo folder. Its hooks capture each session, and Claude Code gets `hindsight_*` tools for search. Capture is automatic, with no setup command.
- **Where it leads.** On the BEAM benchmark at 10 million tokens, Hindsight scores 64.1 against 48.6 for Mem0's platform. Those figures come from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight), so they are vendor-run. The server is MIT-licensed, and the full feature set runs on your own machine.
- **The limits.** The default server is Hindsight Cloud. A local daemon needs an LLM key for fact extraction. If no key is set, the installer falls back to the Claude Code CLI, which needs no key. Each write calls an LLM, which costs more than a plain log. Recall returns more context than a simple store, so set token limits to keep prompts small.

For a side-by-side of the three Claude Code memory options, see [Built-in memory vs claude-mem vs Hindsight](/articles/claude-code-memory/#built-in-memory-vs-claude-mem-vs-hindsight). For other memory servers that plug into Claude Code over MCP, see [AI memory MCP servers](/articles/ai-memory-mcp-server/).

Sources: [Claude Code setup docs](https://code.claude.com/docs/en/setup), read 2026-11-12. [Hindsight Coding Agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12. [Hindsight on GitHub](https://github.com/vectorize-io/hindsight).


## FAQ

### What is the command to install Claude Code?

On macOS, Linux and WSL, run curl -fsSL https://claude.ai/install.sh | bash. On Windows PowerShell, run irm https://claude.ai/install.ps1 | iex. Then open a new terminal and run claude --version to check it worked. These commands come from Anthropic's setup docs, read on 2026-11-12.

### Do I need a paid plan to use Claude Code?

Yes. The docs say Claude Code needs a Pro, Max, Ultra, Team, Enterprise or Console account. The free claude.ai plan does not include Claude Code. You can also use it with a third-party API provider such as Amazon Bedrock, Google Cloud or Microsoft Foundry.

### How do I update Claude Code?

Native installs update themselves in the background. Homebrew, WinGet, npm and Linux package installs need a manual update. Run claude update to update now, or brew upgrade claude-code, winget upgrade Anthropic.ClaudeCode, or npm install -g @anthropic-ai/claude-code@latest for the others.

