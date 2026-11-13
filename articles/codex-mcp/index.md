# Codex MCP: Add, Configure and Check Servers (2026)

> How to add an MCP server to Codex CLI with codex mcp add or config.toml, the timeout and tool-filter settings, how to check servers with /mcp, and which MCP servers give Codex memory. From the official Codex MCP docs, read 2026-11-13.

Source: https://aiagentmemory.org/articles/codex-mcp/ · Updated 2026-11-13 · aiagentmemory.org


## What is MCP in Codex CLI?

**MCP** (Model Context Protocol) lets Codex use tools that live outside it. An MCP server is a program or a remote service that offers tools, such as "search notes" or "save this fact". Codex CLI, the desktop app and the IDE extension all load the same MCP servers, so you set a server up once.

## How do you add an MCP server?

Codex has two kinds of server, and each one has its own setup:

| Server type | Command | Use it for |
|---|---|---|
| Local (stdio) | `codex mcp add <name> -- <command> [args]` | A program that runs on your machine, such as `npx -y <package>` |
| Remote (Streamable HTTP) | `codex mcp add <name> --url <url>` | A service you reach at an address |

The official example adds Context7, a free documentation server:

```bash
codex mcp add context7 -- npx -y @upstash/context7-mcp
```

Put the server's own command after the `--`. Pass a setting such as an API key to a local server with `--env KEY=VALUE`. Remote servers can also take a bearer token from an environment variable. You can set that in `config.toml` with `bearer_token_env_var`.

## What does config.toml look like?

The CLI command writes a table to `config.toml`. You can also write it by hand. Each server gets its own `[mcp_servers.<name>]` table:

```toml
[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]

[mcp_servers.example]
url = "https://mcp.example.com"
```

A local server uses `command`, `args`, `env` and `cwd`. A remote server uses `url`, plus `bearer_token_env_var` or `http_headers` for auth. Leave secrets out of the file where you can: `env_vars` forwards a variable from your shell instead of writing its value down.

Four settings are worth knowing:

| Setting | Default | What it does |
|---|---|---|
| `startup_timeout_sec` | 10 | How long Codex waits for the server to start |
| `tool_timeout_sec` | 60 | How long one tool call may run |
| `enabled` | true | Set `false` to turn a server off without deleting it |
| `enabled_tools` / `disabled_tools` | none | Allow list, then a deny list, for the server's tools |

Codex has a top-level `mcp_optional_startup_grace_ms` setting too (default 1000 ms). It controls how long Codex waits for optional servers when it builds the first tool list.

## How do you check, log in and remove a server?

```bash
codex mcp list          # list configured servers
codex mcp login <name>  # sign in to a server that uses OAuth
codex mcp --help        # all MCP commands
```

Inside the `codex` TUI, type `/mcp` to see the active servers. If the terminal has no local browser, use `codex mcp login <name> --no-browser`, open the printed link yourself and paste the redirect URL back. To turn a server off, set `enabled = false` in its table. To remove it, delete its table from `config.toml`.

## Memory: which MCP servers give Codex memory?

An MCP server gives Codex memory only if it saves facts and returns them later. Two options:

- **A file-based memory server.** The official Knowledge Graph Memory Server (`@modelcontextprotocol/server-memory`) stores entities in a local JSONL file. Point it at a file in your home folder with `MEMORY_FILE_PATH`, and check the server's README for its current settings:

  ```toml
  [mcp_servers.memory]
  command = "npx"
  args = ["-y", "@modelcontextprotocol/server-memory"]

  [mcp_servers.memory.env]
  MEMORY_FILE_PATH = "/Users/you/.codex/memory.jsonl"
  ```

- **Hindsight.** Hindsight is an open-source memory server, MIT licensed, and the full server is in the open-source release. Its Coding Agents package supports Codex CLI: `npx @vectorize-io/hindsight-coding-agents install codex` writes three hooks to `~/.codex/hooks.json` and an `[mcp_servers]` entry to `config.toml`. The hooks need `codex_hooks = true` in that file. The package's steps are in [Codex CLI install](/articles/codex-cli-install/). Hindsight Cloud also runs an MCP server at `https://api.hindsight.vectorize.io/mcp` with OAuth and no API key, according to its Grok Bot integration docs (read 2026-10-24). We did not test that URL in Codex, so check the Hindsight docs before you rely on it.

  On the numbers, Hindsight reports 64.1 on BEAM 10M against Mem0's 48.6. Mem0's own comparison page gives both figures, so this is Mem0's data, not an independent test. The limits matter too: each save calls an LLM, and retrieval can return a lot of context by default. Lower the context size if the prompt gets too big.

Codex only uses memory tools when it decides to. Add a line to `AGENTS.md`, such as "Search memory before you answer questions about past decisions". Otherwise it may never call them. For the full picture, see [Codex memory](/articles/codex-memory/) and [AI memory MCP servers](/articles/ai-memory-mcp-server/). Skills are a different feature: they add instructions, not memory. See [Codex skills](/articles/codex-skills/). For every config key, see the [Codex CLI config reference](/articles/codex-cli-config-reference/).

## Sources

- OpenAI, Codex MCP docs, developers.openai.com/codex/mcp (read 2026-11-13; the page now redirects to the ChatGPT Learn site): add command, `config.toml` keys, timeouts, `enabled_tools`, `codex mcp login`, `/mcp`.
- Codex CLI install page on this site, for the Hindsight Coding Agents command.
- Hindsight Grok Bot integration README (read 2026-10-24), for the hosted MCP URL. Not tested in Codex.
- Benchmark figures: Mem0's comparison page, not Hindsight's own report.


## FAQ

### How do I add an MCP server to Codex CLI?

Run codex mcp add <name> -- <command> [args] for a local server, or codex mcp add <name> --url <url> for a remote one. Codex saves the server in ~/.codex/config.toml under [mcp_servers.<name>]. Run codex mcp list to check it. These commands come from the Codex MCP docs, read on 2026-11-13.

### Where does Codex CLI store MCP server settings?

In config.toml. By default this is ~/.codex/config.toml. A trusted project can also have its own .codex/config.toml. The desktop app, CLI and IDE extension all read the same MCP configuration.

### How do I see which MCP servers Codex is using?

Run codex mcp list in your terminal, or type /mcp in the codex TUI. Both show the servers you have configured and whether they are active.

### Does an MCP server give Codex a memory?

Only if the server saves facts and returns them later. Codex does not remember a fact just because a server is connected. The model must choose to call the memory tools, so add a line to AGENTS.md that tells it when to search memory.

