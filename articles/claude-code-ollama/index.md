# Claude Code with Ollama (2026): Local Models Setup and Memory

> How to run Claude Code with Ollama local or cloud models: the ollama launch command, the manual env vars, the 64k context rule, and how memory works. Ollama docs read 2026-11-17.

Source: https://aiagentmemory.org/articles/claude-code-ollama/ · Updated 2026-11-17 · aiagentmemory.org


**Claude Code with Ollama** means running Anthropic's coding agent against a model served by Ollama, either on your own machine or in Ollama's cloud. Ollama's docs say it connects Claude Code to local and cloud models through an Anthropic-compatible API, and the quickest start is one command: `ollama launch claude`. The model does the thinking, but Claude Code still reads your files and runs commands, so the quality of the result depends on the model you pick.

## Quick setup: the launch command

Ollama's Claude Code page (docs.ollama.com/integrations/claude-code, read 2026-11-17) gives this as the way to start:

```
ollama launch claude
```

For scripts, Docker or CI, skip the menus and name the model. The docs say `--yes` pulls the model when needed and requires `--model`. Anything after `--` goes straight to Claude Code:

```
ollama launch claude --model gemma4:cloud --yes -- -p "how does this repository work?"
```

## Manual setup with a local model

If you want to point Claude Code at Ollama yourself, install Claude Code first ([how to install Claude Code](/articles/install-claude-code/)), then set three variables. These are the values from Ollama's manual setup section:

```
export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_API_KEY=""
export ANTHROPIC_BASE_URL=http://localhost:11434
claude --model qwen3.5
```

`ANTHROPIC_AUTH_TOKEN` is the bearer token. `ANTHROPIC_API_KEY` is set empty so Claude Code does not send a real Anthropic key. Settings in Claude's own config files can override these shell variables. If Claude Code connects to the wrong place, check the settings files, then run `/status` inside Claude Code.

## Local or cloud: which to pick

| | Local model (`localhost:11434`) | Cloud model (Ollama's cloud) |
|---|---|---|
| Install | Ollama on your machine, model pulled | Only Claude Code and an Ollama API key |
| Setup from Ollama's docs | Manual variables above, or `ollama launch claude` | `ANTHROPIC_BASE_URL=https://ollama.com` and `ANTHROPIC_AUTH_TOKEN` set to `OLLAMA_API_KEY` |
| Context | Set 64k tokens or more for larger repos | No download, larger models |
| Known gaps | Speed and quality depend on your hardware | Hosted WebSearch and some advanced tool controls are not fully supported, per Ollama's docs |

Ollama's docs list what works in Claude Code through Ollama: chat, command line runs with the permission flow, tool calling with compatible models, file edits, subagents, web search, web fetch and images. Tool calling depends on the model. Pick one that supports tools.

## Memory with a local model

Claude Code has two built-in memory systems. CLAUDE.md files hold instructions you write, and auto memory holds notes Claude writes for itself in `~/.claude/projects/`. Both are plain files that Claude Code reads at the start of a session ([Claude Code memory guide](/articles/claude-code-memory/)). Ollama's docs do not describe memory at all, so do not assume anything about how a local model uses these files. Test it on your own repo.

If you want memory that lasts across sessions and machines, a separate memory store is the usual route. [Hindsight](https://github.com/vectorize-io/hindsight) is one option. It is MIT-licensed and has a Coding Agents package that covers Claude Code. Install it from your repo folder with `npx @vectorize-io/hindsight-coding-agents install claude-code`, per the [Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12. The package gives each repo its own memory bank.

Hindsight's README says its own server can use a local model through Ollama, LM Studio or llama.cpp, so the memory side can also run without a cloud LLM. That is a claim about the server's options. This page did not test how well a small local model extracts facts for memory, so check the result on your own sessions.

Where Hindsight fits, with its sourced strengths:

- **Long histories.** On the BEAM benchmark it scores 73.9 at 1M tokens and 64.1 at 10M. Mem0's Platform scores 64.1 at 1M and 48.6 at 10M. Those numbers come from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight), and they are vendor-run.
- **Open source.** Hindsight's server is MIT-licensed with the full feature set, so you can run the whole stack on your own machine.

The limits: each write calls an LLM, so a local model adds time to every save. Hindsight also returns more context per recall than Mem0 by default, so set a lower `max_tokens` if local context is tight. For a single short project, the built-in CLAUDE.md files may be enough.

## Which memory to start with

1. Get Claude Code running on Ollama with the launch command. Confirm it reads your repo.
2. Set the context window to 64k tokens or more before you judge quality.
3. Keep CLAUDE.md for rules you always want followed.
4. Add a memory store only when sessions need to remember each other. [Hindsight](/articles/ai-memory-hindsight/) is the one this site covers in depth.

Sources: [Ollama, Claude Code integration](https://docs.ollama.com/integrations/claude-code) (read 2026-11-17); [Claude Code docs](https://code.claude.com/docs/en/overview); [Hindsight README](https://github.com/vectorize-io/hindsight); [Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents) (read 2026-11-12).


## FAQ

### How do I run Claude Code with Ollama?

Run ollama launch claude, which starts Claude Code against Ollama's local or cloud models. For a manual setup, set ANTHROPIC_BASE_URL to http://localhost:11434, set ANTHROPIC_AUTH_TOKEN to ollama and ANTHROPIC_API_KEY to an empty string, then run claude --model with your model name. Ollama's Claude Code page, read 2026-11-17, gives these commands.

### What context window do local models need for Claude Code?

Ollama's docs say to set a context window of 64k tokens or higher for local models, so Claude Code can read a repository. Larger repositories may need more. Cloud models do not need a local download.

### Does Claude Code memory work with Ollama models?

Claude Code's CLAUDE.md files and auto memory are plain files it loads at the start of a session. Ollama's docs say nothing about memory. For a memory store that outlasts the session, Hindsight's Coding Agents package can add one, with a local Ollama model as its LLM if you run your own server.

