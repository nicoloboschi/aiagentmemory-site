# How to Install Gemini CLI (2026): npm, Homebrew, GEMINI.md Memory

> How to install Google's Gemini CLI with npm, Homebrew or npx, sign in, update it, and use GEMINI.md files for memory. Commands from the official README, read 2026-11-12, with the limits Hindsight has for Gemini CLI.

Source: https://aiagentmemory.org/articles/gemini-cli-install/ · Updated 2026-11-12 · aiagentmemory.org


**To install Gemini CLI, run one npm command.** Run `npm install -g @google/gemini-cli`, then run `gemini` in a project folder and sign in with Google. Gemini CLI is Google's open-source coding agent for the terminal.

## What is Gemini CLI?

**Gemini CLI is a command-line coding agent from Google.** It reads and edits files in your project folder, and it runs commands in your terminal. The official repository is [github.com/google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli). Read 2026-11-12.

One important limit to know first: the README says that free individual access and Google AI Pro and Ultra access to Antigravity CLI moved on June 18, 2026. Gemini CLI is aimed at Gemini Code Assist Standard and Enterprise users, and at Code Assist for GitHub on Google Cloud. Check which plan you have before you install.

## Install commands

All commands come from the README's Quick Install section, read 2026-11-12.

| Route | Command | Notes |
|---|---|---|
| npx | `npx @google/gemini-cli` | Runs it without installing. Good for a first try. |
| npm | `npm install -g @google/gemini-cli` | The usual route. Needs Node.js. |
| Homebrew | `brew install gemini-cli` | macOS and Linux |
| MacPorts | `sudo port install gemini-cli` | macOS |
| Restricted machines | `conda create -y -n gemini_env -c conda-forge nodejs`, then `conda activate gemini_env`, then the npm command | For machines where you cannot install Node.js directly |

Run `gemini` to start it.

## Sign in

The README lists three ways to sign in:

1. **Sign in with Google (OAuth).** Run `gemini` and follow the prompts. For a paid Code Assist license from your organization, first run `export GOOGLE_CLOUD_PROJECT="YOUR_PROJECT_ID"`.
2. **Gemini API key.** Run `export GEMINI_API_KEY="YOUR_API_KEY"`, then `gemini`.
3. **Vertex AI.** Run `export GOOGLE_API_KEY="YOUR_API_KEY"` and `export GOOGLE_GENAI_USE_VERTEXAI=true`, then `gemini`.

Keep API keys out of files you commit to Git. Put them in your shell profile or a secrets manager.

## Update and release channels

The README has no separate update command. It uses npm tags instead:

- `npm install -g @google/gemini-cli@latest` for the stable release.
- `@preview` for weekly preview builds.
- `@nightly` for daily builds.

Pick one tag and stay on it. Preview and nightly builds can change without notice.

## Check the install

Run `gemini --version` in a terminal. If the command is not found, open a new terminal window. The npm global folder may not be on your `PATH` yet.

## Memory: GEMINI.md files

**Gemini CLI remembers what you write in GEMINI.md files.** These files are plain text instructions. The CLI reads them and sends their text with every prompt. Per the [GEMINI.md docs](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/gemini-md.md), read 2026-11-12, it loads:

1. **Global:** `~/.gemini/GEMINI.md`. It holds default instructions for all your projects.
2. **Workspace:** `GEMINI.md` files in your workspace folders and their parent folders.
3. **Just in time:** when a tool touches a file, the `GEMINI.md` files in that folder and its parents, up to a trusted root.

Use `/memory show` to print everything the CLI has loaded. Use `/memory reload` after you edit a file. The file name can be changed with the `context.fileName` key in `settings.json`, for example to read `AGENTS.md` too.

**Gemini CLI does not save notes from your chats on its own**, as far as these docs say. Anything it should remember next time goes into a GEMINI.md file that you edit.

### Hindsight and Gemini CLI

Hindsight is open-source memory for AI agents. It can be searched across sessions and shared between agents. Its Coding Agents package, `npx @vectorize-io/hindsight-coding-agents install all`, covers 22 command-line agents, from Claude Code and Codex CLI to Cursor CLI and Qwen Code. **Gemini CLI is not on that list** in the [coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12. So that install command does not set up memory for Gemini CLI.

For Gemini CLI, use GEMINI.md for rules you want every session to follow. Hindsight is a separate option for memory that should search past sessions. Check its current agent list before you rely on it for Gemini CLI.

## Related

- [How to install Codex CLI](/articles/codex-cli-install/): the same kind of install guide for OpenAI's terminal agent.
- [How to install Claude Code](/articles/install-claude-code/): the install steps for Anthropic's terminal agent.
- [Claude Code memory](/articles/claude-code-memory/): how CLAUDE.md and hooks compare with GEMINI.md files.


## FAQ

### How do I install Gemini CLI?

Run npm install -g @google/gemini-cli, or brew install gemini-cli on macOS or Linux. You can also run npx @google/gemini-cli without installing it. Then run gemini in a project folder.

### How do I sign in to Gemini CLI?

Run gemini and sign in with Google (OAuth). You can also set GEMINI_API_KEY, or use Vertex AI with GOOGLE_API_KEY and GOOGLE_GENAI_USE_VERTEXAI=true.

### How does Gemini CLI memory work?

Gemini CLI reads GEMINI.md files that you write. It loads ~/.gemini/GEMINI.md for every project, plus GEMINI.md files in your workspace folders and their parents. The GEMINI.md docs describe no automatic memory from your chats. Use /memory show to see what is loaded and /memory reload after you edit a file.

