# Claude Code MCP: Add, Scope and Remove Servers (2026)

> How to add an MCP server to Claude Code with claude mcp add, pick a scope (local, project or user), check status with /mcp, and keep secrets out of .mcp.json. Commands from the official Claude Code MCP docs, read 2026-11-12. Includes a memory section.

Source: https://aiagentmemory.org/articles/claude-code-mcp/ · Updated 2026-11-12 · aiagentmemory.org


## What is an MCP server in Claude Code?

**MCP** (Model Context Protocol) lets Claude Code use outside tools. An MCP server is a small program or remote service that offers tools, such as "search notes" or "save this fact". Once you add a server, Claude can call its tools in any session that loads it. You set servers up with the `claude mcp` commands, not by editing settings by hand.

## How do you add an MCP server?

There are three common routes. Pick the one that matches your server.

| Server type | Command | Use it for |
|---|---|---|
| Remote HTTP | `claude mcp add --transport http <name> <url>` | Cloud services. The docs recommend this type. |
| Local (stdio) | `claude mcp add <name> -- <command> [args...]` | A program that runs on your machine, such as `npx -y <package>` |
| Raw JSON | `claude mcp add-json <name> '<json>'` | Types the other commands cannot set, such as WebSocket |

Two details trip people up:

- The `--` (double dash) separates Claude's own options, such as `--transport`, `--env` and `--scope`, from the command that starts the server. Put options before `--`, and the server's command after it.
- Use `-e KEY=value` (or `--env`) to pass a setting such as an API key to a local server. Use `-H "Header: value"` (or `--header`) for a remote server that needs a header, such as a bearer token.

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp

claude mcp add --env AIRTABLE_API_KEY=YOUR_KEY --transport stdio airtable \
  -- npx -y airtable-mcp-server
```

The SSE transport still works, but the docs call it deprecated. Use HTTP where the service offers it.

## Which scope should you pick?

The scope decides where a server loads and who else gets it. The default is `local`.

| Scope | Loads in | Shared with your team | Stored in |
|---|---|---|---|
| `local` (default) | Current project only | No | `~/.claude.json`, under that project's path |
| `project` | Current project only | Yes, through version control | `.mcp.json` at the project root |
| `user` | All your projects | No | `~/.claude.json` |

Use `--scope user` for a personal memory server you want everywhere. Use `--scope project` for a server the whole team should share. The team gets a prompt to approve it the first time.

## How do you check, change and remove a server?

```bash
claude mcp list              # list all servers and their status
claude mcp get notion        # show one server's details
claude mcp remove notion     # remove a server
```

Inside a session, run `/mcp` to see status and to log in to servers that need it. A server can show `Connected`, `Needs authentication` or `Failed to connect`. A project server you have not approved shows `Pending approval`. To reset your approval choices for project servers, run `claude mcp reset-project-choices`.

Removing a remote server deletes the OAuth tokens that Claude Code stored for it, but it does not revoke them on the provider's side. To revoke first, run `claude mcp logout <name>`, then remove it.

## How do you write .mcp.json by hand?

Project servers live in `.mcp.json`. A remote server looks like this:

```json
{
  "mcpServers": {
    "shared-server": {
      "type": "http",
      "url": "https://example.com/mcp"
    }
  }
}
```

A local server uses `command`, `args` and `env` instead of `url`. Keep secrets out of the file: use `${VAR}` to read an environment variable, or `${VAR:-default}` to fall back to a default when the variable is not set.

```json
{
  "mcpServers": {
    "api-server": {
      "type": "http",
      "url": "${API_BASE_URL:-https://api.example.com}/mcp",
      "headers": {
        "Authorization": "Bearer ${API_KEY}"
      }
    }
  }
}
```

If a variable is not set and has no default, the config still loads, the text `${VAR}` stays as written, and `claude mcp list` shows a warning. Setting a variable to an empty string counts as set, so the default does not apply. To use the default, unset the variable.

## Memory: which MCP servers give Claude Code memory?

An MCP server gives Claude memory only if it saves facts and returns them later. Two options:

- **The official Knowledge Graph Memory Server** (`@modelcontextprotocol/server-memory`). It stores entities and relations in a local JSONL file. Set `MEMORY_FILE_PATH`, because the default file sits in the server's own install folder:

  ```bash
  claude mcp add --scope user --env MEMORY_FILE_PATH=$HOME/.claude/memory.jsonl \
    memory -- npx -y @modelcontextprotocol/server-memory
  ```

  The command above is the form from the Claude Code memory guide on this site. Check the server's README for its current flags before you rely on it.

- **Hindsight.** Hindsight is an open-source memory server with MIT-licensed self-hosting and the full server in the open-source release. For Claude Code, the supported route is its Coding Agents package, `npx @vectorize-io/hindsight-coding-agents install claude-code`, which captures memory through hooks. Hindsight Cloud also runs an MCP server at `https://api.hindsight.vectorize.io/mcp`, which uses OAuth and needs no API key, according to its Grok Bot integration docs (read 2026-10-24). We did not test adding that URL to Claude Code, so check the Hindsight docs for the exact steps.

  On the numbers, Hindsight reports 64.1 on BEAM 10M against Mem0's 48.6. Mem0's own comparison page gives both figures, so this is Mem0's data, not an independent test. Hindsight's limits matter too: it returns more context per retrieval than Mem0 by default, and each save calls an LLM. Set `max_tokens` lower if the context is too big.

Claude only calls memory tools when it decides to. Put a line in CLAUDE.md such as "Search memory before you answer questions about past decisions", or it may never call them. For the full memory picture, see [Claude Code memory](/articles/claude-code-memory/) and [AI memory MCP servers](/articles/ai-memory-mcp-server/). To install Claude Code first, see [How to install Claude Code](/articles/install-claude-code/).

## Sources

- Claude Code MCP docs, code.claude.com/docs/en/mcp (read 2026-11-12): commands, scopes, `.mcp.json`, environment variable expansion.
- Claude Code memory guide on this site, for the Knowledge Graph Memory Server command.
- Hindsight facts, coding agents: Hindsight's coding-agents docs (read 2026-11-12) and the Grok Bot integration README (read 2026-10-24) for the MCP URL.
- Benchmark figures: Mem0's comparison page, not Hindsight's own report.


## FAQ

### How do I add an MCP server to Claude Code?

Run claude mcp add --transport http <name> <url> for a remote server, or claude mcp add <name> -- <command> [args] for a local one. The default scope is local, which means only the current project. Use --scope user to load it in all your projects. These commands come from the Claude Code MCP docs, read on 2026-11-12.

### Where does Claude Code store MCP servers?

Local and user servers are stored in ~/.claude.json. Project servers are stored in a .mcp.json file at the project root, so you can commit them to version control. Claude Code asks you to approve project servers the first time you use them.

### How do I check whether an MCP server is connected in Claude Code?

Run claude mcp list in your terminal, or run /mcp inside a Claude Code session. The list shows Connected, Needs authentication, or Failed to connect for each server.

### Does an MCP server give Claude Code a memory?

Yes, if the server stores and returns facts. The official Knowledge Graph Memory Server stores entities in a local file. Hindsight can also be reached over MCP. Claude only uses memory tools when it decides to call them, so tell it when to use them in CLAUDE.md.

