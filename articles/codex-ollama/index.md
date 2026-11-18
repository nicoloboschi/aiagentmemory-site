# Codex with Ollama (2026): Local Models Setup and Memory

> How to run OpenAI Codex CLI with Ollama local or cloud models: the ollama launch codex command, codex --oss, the 64k context rule, and how memory works. Ollama docs read 2026-11-18.

Source: https://aiagentmemory.org/articles/codex-ollama/ · Updated 2026-11-18 · aiagentmemory.org


**Codex with Ollama** means running OpenAI's Codex CLI against a model that Ollama serves, either on your own machine or in Ollama's cloud. Ollama's Codex page says you can start it with one command, `ollama launch codex`, or with `codex --oss`, which uses Ollama as the local provider. Codex still edits your files and runs your commands. The model decides how well it does that, so pick one that handles tool calls.

## Quick setup

Ollama's Codex integration page (docs.ollama.com/integrations/codex, read 2026-11-18) gives two ways in.

The launch command sets up Codex for you:

```
ollama launch codex
```

To set it up without starting Codex, add `--config`:

```
ollama launch codex --config
```

The manual way uses Codex's `--oss` flag, which tells Codex to use Ollama. Pass a model name with `-m`:

```
codex --oss -m gpt-oss:120b
```

For a cloud model, Ollama's page uses the `-cloud` suffix in the model name, for example `gpt-oss:120b-cloud`. Codex web-search requests that go through the Ollama profile are run by Ollama, for both local and cloud models, according to the same page.

## Keep the setup in a profile

If you want the setup to stick, Ollama's page says to create `~/.codex/ollama-launch.config.toml`. It sets the model, the provider and the base URL. The page's example uses `http://localhost:11434/v1/` with `wire_api = "responses"`, and then you start Codex with:

```
codex --profile ollama-launch
```

Codex's own config options are in [the Codex CLI config reference](/articles/codex-cli-config-reference/). Install steps are in [the Codex CLI install guide](/articles/codex-cli-install/).

## Local or cloud: which to pick

| | Local model (Ollama on your machine) | Cloud model (`-cloud` suffix) |
|---|---|---|
| Start with | `ollama launch codex` or `codex --oss -m <model>` | Same commands, with a cloud model name |
| Model download | Yes, you pull it first | No |
| Context | Set at least 64k tokens, per Ollama's page | Not covered on the page |
| Main limit | Speed and quality depend on your hardware | Runs on Ollama's cloud, not your machine |

Ollama's page says each model keeps its own context window, vision and thinking settings, and that you can switch between your installed, tool-capable Ollama models inside Codex with `/model`.

## Memory with a local model

Ollama's Codex page does not mention memory. Codex has its own memory and AGENTS.md files. For what those do, see [Codex memory](/articles/codex-memory/). Ollama's page says nothing about how a local model uses them, so test them on your own repo.

If you want facts to carry across sessions and tools, a separate memory store is the usual route. [Hindsight](https://github.com/vectorize-io/hindsight) is one option. Its Coding Agents package covers Codex CLI, along with 21 other CLI agents. Install it from your repo folder with `npx @vectorize-io/hindsight-coding-agents install all`, or name one agent, per the [Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12. Each repo gets its own memory bank, and every agent in that package shares it.

Hindsight's README lists Ollama among its 25+ LLM providers. So if you host the Hindsight server yourself, its memory writes can run on a local Ollama model. This page did not test how well a small local model extracts facts, so check the result on your own sessions.

Where Hindsight fits, with its sourced strengths:

- **Long histories.** On the BEAM benchmark it scores 73.9 at 1M tokens and 64.1 at 10M. Mem0's Platform scores 64.1 at 1M and 48.6 at 10M. Both figures come from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight), so they are vendor-run.
- **Open source.** The Hindsight server is MIT-licensed with the full feature set, so you can run the whole stack on your own machine.

The limits: each memory write calls an LLM, so a local model makes every save slower. Recall also returns more context than a simple file, so set a lower `max_tokens` if your local context is tight. For one small project, Codex's own memory and an AGENTS.md file may be enough.

## Which setup to start with

1. Run `ollama launch codex` and check that Codex reads your repo.
2. Set the context window to at least 64k tokens before you judge quality.
3. Keep AGENTS.md for rules you always want followed.
4. Add a memory store only when sessions need to remember each other. [Hindsight](/articles/ai-memory-hindsight/) is the one this site covers in depth.

Sources: [Ollama, Codex integration](https://docs.ollama.com/integrations/codex) (read 2026-11-18); [Hindsight README](https://github.com/vectorize-io/hindsight); [Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents) (read 2026-11-12); [Mem0 vs Hindsight](https://mem0.ai/compare/mem0-vs-hindsight).


## FAQ

### How do I run Codex CLI with Ollama?

Run ollama launch codex to start Codex against Ollama's models, or run codex --oss to use the local Ollama provider directly. Pick a model with -m, for example codex --oss -m gpt-oss:120b. Ollama's Codex page, read 2026-11-18, gives these commands.

### What context window does Codex need with Ollama?

Ollama's Codex page says to use a context window of at least 64k tokens for Codex. Set it on the local model before you judge the results.

### Does Codex memory work with Ollama models?

Ollama's Codex page does not mention memory. Codex has its own memory and AGENTS.md files, but a store that keeps facts across sessions and tools needs a separate tool. Hindsight's Coding Agents package covers Codex CLI and can run its LLM through Ollama if you host the server yourself.

