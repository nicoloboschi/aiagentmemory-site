# MCP Server Config File Locations: Claude Code, Claude Desktop, Codex, Cursor, VS Code, Hermes, OpenClaw (2026)

> Where seven AI clients keep their MCP server list, the file path per OS and the top-level key, with a minimal example. Each row is from that client's own docs, read 2026-11-25.

Source: https://aiagentmemory.org/articles/mcp-config-file-locations/ · Updated 2026-11-25 · aiagentmemory.org


Each AI client keeps its MCP server list in its own file. This page lists the file for seven clients, the top-level key inside it, and where the file sits. Each row comes from that client's own documentation, read on 2026-11-25. Where a client's docs give no path for an OS, the table says so.

## Where each client keeps its MCP servers

| Client | File | Top-level key | Path |
|---|---|---|---|
| Claude Code | `.mcp.json` (project, shared) and `~/.claude.json` (user and local) | `mcpServers` | `.mcp.json` at the project root. `~/.claude.json` in your home folder. The docs give no separate Windows path. |
| Claude Desktop | `claude_desktop_config.json` | `mcpServers` | macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`. Windows: `%APPDATA%\Claude\claude_desktop_config.json`. Linux: `~/.config/Claude/claude_desktop_config.json`. |
| Codex | `config.toml` | `[mcp_servers.<name>]` tables | `~/.codex/config.toml`. Per project: `.codex/config.toml`, trusted projects only. The docs give no Windows path. |
| Cursor | `mcp.json` | `mcpServers` | Global: `~/.cursor/mcp.json`. Per project: `.cursor/mcp.json` in the project folder. |
| VS Code | `.vscode/mcp.json` (workspace) | `servers` | The workspace file is `.vscode/mcp.json` in the project folder. VS Code's docs say new Copilot setups use other files: `~/.copilot/mcp-config.json` (top key `mcpServers`) or a workspace-root `.mcp.json`. |
| Hermes Agent | `config.yaml` | `mcp_servers` | `~/.hermes/config.yaml`. The docs give no Windows path. |
| OpenClaw | `openclaw.json` | `mcp.servers` | `~/.openclaw/openclaw.json`. The `openclaw mcp` commands read and write this section. |

Sources: [Claude Code MCP docs](https://code.claude.com/docs/en/mcp) (scopes: local, project, user); [Claude Desktop local MCP servers, Model Context Protocol docs](https://modelcontextprotocol.io/docs/develop/connect-local-servers) (the three OS paths; the VS Code configuration reference repeats them); [Codex MCP docs](https://developers.openai.com/codex/mcp); [Cursor MCP docs](https://cursor.com/docs/mcp) (`mcp.json`, STDIO fields) and the VS Code configuration reference, which lists the Cursor global and workspace paths; [VS Code MCP servers](https://code.visualstudio.com/docs/agent-customization/mcp-servers) and [VS Code MCP configuration reference](https://code.visualstudio.com/docs/agents/reference/mcp-configuration); [Hermes MCP docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp); [OpenClaw MCP registry docs](https://docs.openclaw.ai/cli/mcp/registry). The OpenClaw file name comes from [OpenClaw's configuration docs](https://docs.openclaw.ai/gateway/configuration).

## Minimal examples

The same server, a local filesystem server, written for each client. Replace the path with your own.

**Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/you/Desktop"]
    }
  }
}
```

**Claude Code** (`.mcp.json` at the project root, or the same entry added with `claude mcp add`):

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/project"]
    }
  }
}
```

**Codex** (`~/.codex/config.toml`):

```toml
[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/project"]
```

**Cursor** (`~/.cursor/mcp.json` or `.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/project"]
    }
  }
}
```

**Hermes Agent** (`~/.hermes/config.yaml`):

```yaml
mcp_servers:
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"]
```

VS Code's workspace file uses the key `servers`, not `mcpServers`. OpenClaw's section is `mcp.servers`, and the docs show its entries through the `openclaw mcp` commands, so this page gives no hand-written example for it.

## Things that catch people out

- **Wrong key.** Claude, Cursor and VS Code use different top-level keys (`mcpServers` or `servers`). Codex and Hermes use their own format. Check the key against the table above before you edit a file.
- **Project files need approval.** Claude Code asks before it uses a server from `.mcp.json`. Codex reads a project `.codex/config.toml` only in a trusted project.
- **Hosted clouds read other files.** Cursor's docs say Cursor's hosted Cloud Agents do not read `~/.cursor/mcp.json`. They use servers set on the Cursor website.
- **Plugins are separate.** Installing a VS Code agent plugin can bring its own servers. Those do not appear in your own file.

## Where memory fits

An MCP server is a way to give an agent memory tools. Hindsight has an MCP endpoint: the Grok Bot integration README lists `https://api.hindsight.vectorize.io/mcp` (OAuth, no API key) as its server. For Hermes Agent and OpenClaw, the Hindsight plugin is the usual route and does not need an MCP entry; see [the Hermes Agent config reference](/articles/hermes-agent-config-reference/) and [the OpenClaw config reference](/articles/openclaw-config-reference/). Limit: if you run your own Hindsight server and want an MCP client to reach it, the MCP endpoint must be public HTTPS with OAuth 2.1, according to Hindsight's integration notes.

## Scope

This page lists the main file per client. Clients can read more than one file (for example Claude Code's user and local scopes both live in `~/.claude.json`). The docs for each client are the authority if a path changes.


## FAQ

### Where is the Claude Desktop MCP config file?

macOS: ~/Library/Application Support/Claude/claude_desktop_config.json. Windows: %APPDATA%\Claude\claude_desktop_config.json. Linux: ~/.config/Claude/claude_desktop_config.json. The servers go under the mcpServers key.

### Where does Claude Code keep MCP servers?

Shared with your team: .mcp.json at the project root. Personal and user-wide servers: ~/.claude.json. Claude Code asks you to approve project servers from .mcp.json before it uses them.

### Where does Codex keep MCP servers?

In ~/.codex/config.toml, one [mcp_servers.<name>] table per server. A project can add .codex/config.toml, but only in trusted projects.

### Where is the Cursor mcp.json file?

Global: ~/.cursor/mcp.json. Per project: .cursor/mcp.json in the project folder. The servers go under the mcpServers key.

