# Claude Code Cost: Real Spend Numbers and What Memory Adds (2026)

> What Claude Code costs per developer: average $13 per active day, $150-250 per month, and how CLAUDE.md, MCP servers and memory tools add tokens. From the official costs page, read 2026-11-16.

Source: https://aiagentmemory.org/articles/claude-code-cost/ · Updated 2026-11-16 · aiagentmemory.org


## What does Claude Code cost?

**Claude Code costs are token costs.** The more text Claude reads and writes, the more you pay, or the more of your plan limit you use. Anthropic's cost page (code.claude.com/docs/en/costs, read 2026-11-16) gives one reference point: across enterprise deployments, the average is about **$13 per developer per active day**, or **$150 to $250 per developer per month**. For 90% of users, the cost stays below $30 per active day.

Subscription prices for Pro, Max, Ultra, Team and Enterprise are on claude.com/pricing. Those prices change, so this page does not copy them. The cost page says Claude Code charges by API token use, and the plan you pick decides how those tokens are billed.

## How can you see what a session costs?

Use these tools to measure before you cut anything:

| Tool | What it shows | Source |
| :- | :- | :- |
| `/usage` | Session tokens and cost, and a breakdown of plan usage by skill, subagent, plugin and MCP server | Anthropic costs page |
| `/context` | What fills the context window, by category | Anthropic costs page |
| Status line | Cost and context fill, shown all the time | Anthropic costs page |
| OpenTelemetry export | Per-user token counts and cost, for a team's own tools | Anthropic costs page |
| Console Cost page | Spend for API workspaces | Anthropic costs page |

The `/usage` figure is an estimate at list price. An admin can set a `modelPricing` table in managed settings so the figure uses contracted rates. The cost page says the figure is not an invoice.

## Why does memory add tokens?

Every request sends the context again. Memory is more context. The cost page lists the parts that add tokens at the start of, or during, a session:

- **CLAUDE.md** ([how Claude Code memory files load](/articles/claude-code-memory/)): "Your CLAUDE.md file is loaded into context at session start." Its tokens are there even for unrelated work. The page says to keep it under 200 lines.
- **MCP servers** ([Claude Code and MCP](/articles/claude-code-mcp/)): tool definitions are deferred by default, so only tool names and server instructions enter context until Claude uses a tool. Each server still adds something, and the page says to disable servers you do not use.
- **Skills:** these load only when invoked, so moving workflow steps from CLAUDE.md into skills keeps the base context small.
- **Compaction:** `/compact` sends the whole conversation it summarizes, so a large compact is itself a large request.

Prompt caching lowers the cost of repeated text. The cost page notes that a cache miss, such as a first message after a long break, reprocesses the whole context.

## How to cut the cost without losing memory

1. **Clear between unrelated tasks.** `/clear` starts a new session and costs nothing. Use `/rename` first if you want to come back later.
2. **Pick the model for the job.** The cost page says Sonnet handles most coding tasks and costs less than Opus. Set `model: haiku` for simple subagent work.
3. **Trim CLAUDE.md.** Keep the lines that apply to every task. Move the rest into skills.
4. **Turn off MCP servers you do not use.** Run `/mcp` to see them. Prefer CLI tools such as `gh` when they do the job.
5. **Check `/usage` after a long day.** It shows which skill, subagent or MCP server took the largest share of recent usage.

For the commands behind these steps, see [Claude Code commands](/articles/claude-code-commands/). For the settings that control them, see [Claude Code settings](/articles/claude-code-settings/).

## Where Hindsight fits

[Hindsight](https://github.com/vectorize-io/hindsight) is an open-source memory server. It is an alternative to keeping everything in CLAUDE.md. Its Coding Agents package installs into Claude Code with one command, `npx @vectorize-io/hindsight-coding-agents install claude-code`, and captures sessions automatically (Hindsight coding agents page, read 2026-11-12).

On cost, the trade-off is clear:

- **Writes cost more.** Hindsight calls an LLM to extract facts on every retain. Hindsight Cloud bills retain and recall per token. Self-hosting is free, but the LLM calls are yours.
- **Reads return more context by default.** Mem0's own measurement, on its comparison page, puts Hindsight at about 27K tokens per retrieval on BEAM 10M, against under 7K for Mem0. Hindsight lets you set a lower token budget.
- **Long histories are its strength.** On BEAM 10M, Hindsight scored 64.1 and Mem0 scored 48.6. Both figures come from Mem0's comparison page (mem0.ai/compare/mem0-vs-hindsight). They are vendor numbers, not an independent test.
- **The old Claude Code plugin is retired.** Switch to the Coding Agents package. It does not carry over the old plugin's recall and retain settings.

For a small project, a short CLAUDE.md is the cheaper option. A memory server makes more sense when many sessions need the same facts back. Measure both on your own repo with `/usage` before you choose.

## Sources

- Claude Code costs: [code.claude.com/docs/en/costs](https://code.claude.com/docs/en/costs) (read 2026-11-16)
- Hindsight Coding Agents: [hindsight.vectorize.io/sdks/integrations/coding-agents](https://hindsight.vectorize.io/sdks/integrations/coding-agents) (read 2026-11-12)
- Mem0 vs Hindsight, Mem0's comparison page: [mem0.ai/compare/mem0-vs-hindsight](https://mem0.ai/compare/mem0-vs-hindsight) (BEAM 10M figures and token counts, vendor-reported)
- Hindsight: [github.com/vectorize-io/hindsight](https://github.com/vectorize-io/hindsight)


## FAQ

### How much does Claude Code cost per developer?

Anthropic's cost page says the average across enterprise deployments is about $13 per developer per active day, or $150 to $250 per developer per month. Costs stay below $30 per active day for 90% of users. These are averages, not a price list.

### Does CLAUDE.md make Claude Code more expensive?

Yes, a little. The CLAUDE.md file loads into context at the start of each session, so its tokens are sent with every request. Anthropic's costs page advises keeping it under 200 lines and moving workflow steps into skills, which load only when invoked.

### Does memory add to the Claude Code bill?

It can. Memory that sits in files adds tokens on every request. A memory server such as Hindsight adds its own cost: it calls an LLM to write memories, and its recall returns extra context. Its page does not give a total per session, so measure it on your own repo.

