# AI Coding Agent Pricing: Claude Code, Codex, GitHub Copilot Plans and Prices (November 2026)

> Plan prices and usage limits for Claude Code, OpenAI Codex and GitHub Copilot, read from each vendor's pricing page on 2026-11-25. Cursor and Windsurf are left out because their pages did not give a clean price list.

Source: https://aiagentmemory.org/articles/ai-coding-agent-pricing/ · Updated 2026-11-25 · aiagentmemory.org


# AI Coding Agent Pricing: Claude Code, Codex and GitHub Copilot (November 2026)

**Short answer:** the cheapest paid plans are $10 (Copilot Pro), $17 to $20 (Claude Pro) and $20 (ChatGPT Plus, which includes Codex). The top tiers are $100 to $200 a month. Every price below was read from the vendor's own pricing page on 2026-11-25. Prices can change, so check the page before you buy.

## Plan prices and limits

| Tool | Plan | Price per month | What it includes for coding | Source |
| --- | --- | --- | --- | --- |
| Claude Code | Free | $0 | Claude Code is not included | claude.com/pricing |
| Claude Code | Pro | $17 billed yearly ($200 up front), $20 billed monthly | Claude Code included, 1x usage per 5 hours and per week | claude.com/pricing |
| Claude Code | Max | $100 | Claude Code included, 5x Pro usage per 5 hours and per week | claude.com/pricing |
| Claude Code | Ultra | $200 | Claude Code included, 10x Pro usage per week, no 5-hour limit for verified accounts | claude.com/pricing |
| Claude Code | Team, standard seat | $20 per seat billed yearly, $25 billed monthly | Claude Code included, more usage than Pro | claude.com/pricing |
| Claude Code | Team, premium seat | $100 per seat billed yearly, $125 billed monthly | 5x the usage of a standard seat | claude.com/pricing |
| Codex | Free | $0 | Limited Codex access | chatgpt.com/pricing |
| Codex | Plus | $20 | Codex on the web, CLI, IDE extension and iOS; about 15 to 160 local messages per 5 hours on GPT-6.1 Sol | chatgpt.com/pricing, learn.chatgpt.com/docs/pricing |
| Codex | Pro | from $100, three usage tiers | Longer Codex sessions; the Codex page says Pro has no five-hour limit | chatgpt.com/pricing, learn.chatgpt.com/docs/pricing |
| Codex | API key | Pay per token at API rates | Codex CLI, SDK or IDE extension; no cloud features such as GitHub review or Slack | learn.chatgpt.com/docs/pricing |
| GitHub Copilot | Pro | $10 | 1,000 base GitHub AI Credits, a selection of models | docs.github.com (plans for GitHub Copilot) |
| GitHub Copilot | Pro+ | $39 | 3,900 base GitHub AI Credits, premium models | docs.github.com |
| GitHub Copilot | Max | $100 | 10,000 base GitHub AI Credits, priority access to premium models | docs.github.com |
| GitHub Copilot | Business | $19 per user | 1,900 GitHub AI Credits per user, premium models | docs.github.com |
| GitHub Copilot | Enterprise | $39 per user | 3,900 GitHub AI Credits per user, priority access to premium models | docs.github.com |

Notes on the table:

- **Claude API prices are per token.** On claude.com/pricing, read 2026-11-25: Opus 5.5 is $4 per million input tokens and $20 per million output; Sonnet 5.5 is $2 and $10; Haiku 5.5 is $0.10 and $0.50. Batch processing is half price.
- **Codex limits are estimates.** OpenAI says its per-5-hour message counts are estimates, not fixed limits. Check the usage dashboard for your real numbers.
- **Copilot uses credits.** GitHub's plans now count usage in AI credits, so a monthly price does not give a fixed number of requests. Business and Enterprise credits are pooled, and extra usage costs $0.01 per credit.

## Left out, and why

- **Cursor:** the pricing page we read on 2026-11-25 lists plans with Grok and X features (Plus $8, Premium $30, Super $100, Ultra $200). We could not tell whether those are the coding plans, so we did not put them in the table. Check cursor.com/pricing before quoting Cursor.
- **Windsurf:** windsurf.com/pricing now redirects to devin.ai/pricing. That page lists Devin plans (Pro $20, Max $200, Teams $80). We could not confirm they are the same as Windsurf's plans, so we did not list them.

## Memory across coding agents

Plan prices do not include memory. A coding agent forgets a repo's decisions when a session ends, unless it keeps notes or uses a memory layer.

Hindsight, an open-source memory server, has one install for 22 CLI coding agents, including Claude Code, Codex CLI and Copilot CLI (`npx @vectorize-io/hindsight-coding-agents install claude-code`). Source: hindsight.vectorize.io/sdks/integrations/coding-agents, read 2026-11-12. One memory bank is used per repo, and every agent in that repo shares it.

- **Cost:** self-hosting Hindsight is free under the MIT license, with no usage limits and no telemetry. Hindsight Cloud is billed by use (retain $10 per million tokens, recall $0.75 per million tokens, per Hindsight's pricing page, checked 2026-11-16).
- **Where it leads (vendor-run numbers):** on BEAM, a long-history benchmark, Hindsight scores 64.1 at 10M tokens against Mem0's 48.6, and 73.9 against 64.1 at 1M. Both figures come from Mem0's comparison page, not an independent test.
- **Limits:** Hindsight's retain and consolidation steps call an LLM, so they add cost. Memory is only as good as the model that reads it. Gemini CLI is not on the 22-agent list.

## Related

- [MCP Server Config File Locations](/articles/mcp-config-file-locations/): where Claude Code, Codex and other clients keep their MCP server list.
- [Codex CLI config reference](/articles/codex-cli-config-reference/): the config keys Codex reads.

## How this was checked

Each price was read from the vendor's page on 2026-11-25: claude.com/pricing (Claude), chatgpt.com/pricing and learn.chatgpt.com/docs/pricing (Codex), docs.github.com plans for GitHub Copilot. Prices exclude tax. Pages change without notice.


## FAQ

### How much does Claude Code cost per month?

Claude Code is included in the Claude Pro plan ($17 a month billed yearly, $20 billed monthly), Max ($100) and Ultra ($200). It is not included in the Free plan. Source: claude.com/pricing, read 2026-11-25.

### How much does Codex cost per month?

Codex is included in ChatGPT Plus at $20 a month and in ChatGPT Pro from $100 a month. You can also use Codex with an API key and pay per token at API rates. Sources: chatgpt.com/pricing and learn.chatgpt.com/docs/pricing, read 2026-11-25.

### How much does GitHub Copilot cost per month?

Copilot Pro is $10 a month, Copilot Pro+ is $39, and Copilot Max is $100. Copilot Business is $19 per user and Copilot Enterprise is $39 per user. Source: GitHub Docs plan table, read 2026-11-25.

### Is there a memory option for coding agents?

Yes. Hindsight has one npm package, @vectorize-io/hindsight-coding-agents, that installs memory into 22 CLI agents, including Claude Code, Codex CLI and Copilot CLI. Its memory is separate from the plan price: the agent plan does not include it, and self-hosting Hindsight is free under the MIT license.

