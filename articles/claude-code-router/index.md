# Claude Code Router (2026): Setup, Providers and Memory

> What Claude Code Router does, how to start it on port 3456 and point Claude Code at it, and what Anthropic says about routing to non-Claude models. Sources read 2026-11-17.

Source: https://aiagentmemory.org/articles/claude-code-router/ · Updated 2026-11-17 · aiagentmemory.org


**Claude Code Router** (CCR) is a local program that sits between Claude Code and the model provider. Claude Code sends its requests to CCR on your machine. CCR then sends them on to the provider you chose, such as a Claude API key, OpenRouter or DeepSeek. The project is MIT-licensed, on GitHub at `musistudio/claude-code-router`. Its README says it also works with Codex, OpenCode and other agents, but this page covers only Claude Code.

## Setup in short

There are two ways to start it. The desktop app is the first one in the README. The command-line route is:

```
npm install -g @musistudio/claude-code-router
ccr ui
```

The README says the command line needs Node.js 22 or newer. Then open the management page at `http://127.0.0.1:3458`, add a provider with its API key, and start the server. The model gateway listens on `http://127.0.0.1:3456`. Then apply the profile for Claude Code (step 4 of the README's desktop steps).

Anthropic's gateway docs name the variables for any gateway: `ANTHROPIC_BASE_URL` for the address and `ANTHROPIC_AUTH_TOKEN` or `ANTHROPIC_API_KEY` for the credential. The README does not show them for CCR, so check the router's CLI docs for the exact values before you set them by hand.

## What it does and what Anthropic says

| | Claude Code Router | Anthropic's position |
|---|---|---|
| What it is | Open-source local gateway, MIT license | Gateways are your own choice; Anthropic does not maintain third-party ones |
| Models | Many providers, including OpenRouter, DeepSeek, Mistral and Z.AI, per the README | Routing Claude Code to non-Claude models through any gateway is not supported |
| Failover | Retries and an ordered list of fallback models, per the README | Not described |
| Logs | Request logs with the provider, model, tokens and latency, per the README | Not described |

Read the last column carefully. The reason many people install a router is to use cheaper or local models. Anthropic's docs say it does not support that. Other things can still work, but if something breaks, Anthropic will not help.

## Memory with a router

Claude Code Router does not add memory. Claude Code keeps its usual files: CLAUDE.md for rules you write, and auto memory for notes it keeps itself in `~/.claude/projects/`. The [Claude Code memory guide](/articles/claude-code-memory/) explains both. Nothing in CCR's README mentions memory, so do not expect a router to keep facts between sessions.

If you need that, [Hindsight](https://github.com/vectorize-io/hindsight) offers one option. Its Coding Agents package installs into Claude Code with `npx @vectorize-io/hindsight-coding-agents install claude-code`, per the [Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12. The docs do not say whether the package works when Claude Code goes through a router. This page did not test it, so test it before you rely on it.

Where Hindsight fits, with its sourced strengths:

- **Long histories.** On the BEAM benchmark it scores 73.9 at 1M tokens and 64.1 at 10M. Mem0's Platform scores 64.1 at 1M and 48.6 at 10M. Those numbers come from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight), so they are vendor-run.
- **Open source.** The Hindsight server is MIT-licensed with the full feature set.

The limits: each write calls an LLM, which adds cost and time to every save. By default it returns more context per recall than Mem0, so set a lower `max_tokens` if you need room.

## Which to try first

1. Run Claude Code on its own, with your Claude login, to check it works.
2. Add the router only if you have a clear reason, such as a provider you already pay for. Know that Anthropic does not support non-Claude models through a gateway.
3. Keep CLAUDE.md for the rules you always want followed. Add a memory store only when sessions need to remember each other.

Sources: [Claude Code Router README](https://github.com/musistudio/claude-code-router) (read 2026-11-17); [Anthropic, other LLM gateways](https://code.claude.com/docs/en/llm-gateway) and [connect Claude Code to a gateway](https://code.claude.com/docs/en/llm-gateway-connect) (read 2026-11-17); [Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents) (read 2026-11-12); [Hindsight README](https://github.com/vectorize-io/hindsight).


## FAQ

### What is Claude Code Router?

Claude Code Router (CCR) is an open-source MIT project by musistudio. It runs a local gateway on http://127.0.0.1:3456. You point Claude Code at that address, and CCR sends each request to the model provider you set up, with retries and fallback models. Its README, read 2026-11-17, describes it this way.

### Does Anthropic support Claude Code Router?

Anthropic does not endorse, maintain or audit third-party gateways, and says it does not support routing Claude Code to non-Claude models through any gateway. Its gateway docs, read 2026-11-17, say this. Using CCR with non-Claude models is outside what Anthropic supports.

### Does Claude Code Router add memory to Claude Code?

No. The router passes requests between Claude Code and a model. It does not store facts between sessions. Claude Code's own CLAUDE.md files and auto memory still work the way they did before. For memory that lasts across sessions, you need a separate memory store, such as Hindsight's Coding Agents package. Using that package through the router is not tested here.

