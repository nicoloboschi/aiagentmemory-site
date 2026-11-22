# Local LLM Statistics (November 2026)

> Local LLM runner statistics for 10 projects: Ollama, llama.cpp, vLLM, LM Studio and others. GitHub stars, forks, licenses and last-month PyPI and npm downloads. Collected 2026-11-22.

Source: https://aiagentmemory.org/articles/local-llm-statistics/ · Updated 2026-11-22 · aiagentmemory.org


On 2026-11-22, Ollama had the most GitHub stars of the 10 local LLM projects we checked (187,830) and the most PyPI downloads (12,622,442 for its `ollama` package last month). These local LLM statistics were collected by aiagentmemory.org from the GitHub REST API, pypistats.org and the npm registry on 2026-11-22 at 09:21 UTC. We did not run any model ourselves.

Local runners do not keep memory. Ollama, LM Studio and llama.cpp process each request on its own. If you want a local model to remember past chats, see [how to give a local LLM memory](/articles/how-to-give-local-llm-memory/), [Claude Code with Ollama](/articles/claude-code-ollama/) and [Codex with Ollama](/articles/codex-ollama/).

For the same kind of numbers on agent memory projects, see [agent memory statistics](/articles/agent-memory-statistics/). For coding agents, see [AI coding agent statistics](/articles/ai-coding-agent-statistics/).

## Local LLM GitHub statistics

**Table 1. GitHub statistics for 10 local LLM projects, collected 2026-11-22 from the GitHub REST API.** Sorted by stars. Open WebUI is a chat front end, not a model runner. The PyPI and npm columns are last-month downloads for the package each project publishes; see the notes below the table.

| # | Project | Repository | Stars | Forks | Created | Last push | License | PyPI/month | npm/month |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Ollama | ollama/ollama | 187,830 | 19,138 | 2023-06-26 | 2026-11-21 | MIT | 12,622,442 | 5,101,054 |
| 2 | Open WebUI | open-webui/open-webui | 159,245 | 23,109 | 2023-10-06 | 2026-11-21 | NOASSERTION | 1,090,409 | - |
| 3 | llama.cpp | ggml-org/llama.cpp | 135,992 | 25,891 | 2023-03-10 | 2026-11-22 | MIT | 585,369 | - |
| 4 | vLLM | vllm-project/vllm | 97,328 | 25,128 | 2023-02-09 | 2026-11-22 | Apache-2.0 | 2,762,854 | - |
| 5 | AnythingLLM | Mintplex-Labs/anything-llm | 68,502 | 7,730 | 2023-06-04 | 2026-11-21 | MIT | - | - |
| 6 | LocalAI | mudler/LocalAI | 50,495 | 4,634 | 2023-03-18 | 2026-11-22 | MIT | - | - |
| 7 | Jan | janhq/jan | 46,012 | 3,236 | 2023-08-17 | 2026-11-18 | NOASSERTION | - | - |
| 8 | SGLang | sgl-project/sglang | 38,922 | 10,752 | 2024-01-08 | 2026-11-22 | Apache-2.0 | 678,365 | - |
| 9 | MLX LM | ml-explore/mlx-lm | 7,729 | 1,213 | 2025-03-11 | 2026-11-21 | MIT | 593,711 | - |
| 10 | LM Studio CLI | lmstudio-ai/lms | 5,445 | 479 | 2024-04-15 | 2026-11-17 | MIT | 144,148 | 277,913 |

"NOASSERTION" means GitHub did not match the repository's license file to a standard license, so read the file before you rely on it. This applies to Open WebUI and Jan. "-" means the list has no package for that project, or the project does not publish one.

## What the numbers show

- **Ollama leads on stars and on both package counts.** It has the most stars, and the most npm and PyPI downloads of the runners. It does not have the most forks: llama.cpp (25,891), vLLM (25,128) and Open WebUI (23,109) all have more.
- **Two of the top four are not libraries.** Open WebUI is a chat front end that talks to a runner. vLLM is a serving engine. Its PyPI package installs the server itself, so its download count is not comparable with a client library such as the `ollama` package.
- **llama.cpp's PyPI number is for a different project.** The 585,369 figure is for `llama-cpp-python`, a Python binding made by a separate developer (abetlen), not by the llama.cpp project. Read it as a proxy for Python use of llama.cpp, not as llama.cpp's own downloads.
- **LM Studio's download counts come from SDK packages.** The `lmstudio` Python package and the `@lmstudio/sdk` npm package are the libraries LM Studio publishes; both are maintained by the LM Studio team.

Download counts include CI runs, container builds and packages installed as dependencies, so they are not user counts. Monthly counts also swing from one month to the next. Read a change of a few tens of percent as a signal, not a trend.

## Memory: where Hindsight fits

Hindsight is not a local runner. It is an open-source memory server (MIT license, the full server included, from its [README](https://github.com/vectorize-io/hindsight)). It uses an LLM for writes and for reasoning, and it can use a local model for that:

- **Local models work.** Hindsight supports 25+ LLM providers, including local ones: Ollama, LM Studio and llama.cpp (Hindsight README, checked 2026-10-12). A self-hosted setup can keep the whole stack on one machine.
- **Its default embeddings are local.** The self-hosted default is the `local` embeddings provider with `BAAI/bge-small-en-v1.5`, so no embedding API is needed.
- **The limit: every write calls the LLM.** Retain and consolidation both call a model. With a local model, each write runs at your machine's speed, and a slow local model slows every memory write. We have no measured speed figures for Hindsight with local models, so none are quoted here.

## Methodology

**When.** All numbers were collected on 2026-11-22, starting at 09:21 UTC, by aiagentmemory.org with the same script used for [agent memory statistics](/articles/agent-memory-statistics/).

**Which APIs.**

1. **GitHub:** `GET https://api.github.com/repos/{owner}/{repo}`, through the `gh` CLI. Fields used: `stargazers_count`, `forks_count`, `created_at`, `pushed_at`, `license.spdx_id`.
2. **PyPI:** `GET https://pypistats.org/api/packages/{package}/recent`, last-month total.
3. **npm:** `GET https://api.npmjs.org/downloads/point/last-month/{package}`, last-month total.

**Which packages.** Each package was checked against the project before we counted it. `ollama` on PyPI and npm is published by the Ollama project. `lmstudio` on PyPI and `@lmstudio/sdk` on npm are published by LM Studio. `llama-cpp-python` on PyPI is a third-party binding by abetlen, not the llama.cpp project, and the table says so. `vllm`, `sglang`, `mlx-lm` and `open-webui` on PyPI install the project's own software. LocalAI, AnythingLLM and Jan have no package in our list, so their package cells are empty.

**What's excluded.** Hosted services, model weights on Hugging Face, and the model families (Llama, Qwen, Gemma) are not counted. We did not measure speed or quality of any runner.

**Not yet tracked.** The GitHub numbers are one snapshot. We will re-run the list in two weeks and add the change.


## FAQ

### Which local LLM runner has the most GitHub stars?

On 2026-11-22, Ollama had the most GitHub stars of the 10 local LLM projects we checked (187,830), followed by Open WebUI (159,245) and llama.cpp (135,992). Counts come from the GitHub REST API, collected by aiagentmemory.org.

### Which local LLM package is downloaded the most from PyPI?

Ollama's Python package `ollama` had 12,622,442 PyPI downloads in the last month on 2026-11-22, the most of the packages we checked. vLLM's `vllm` package was second with 2,762,854, but it is the server itself, not a client library, so the two counts measure different things.

### Do local LLMs keep memory between chats?

No. Ollama, LM Studio, llama.cpp and vLLM process each request and keep nothing between requests. A memory layer has to store facts outside the model and add them back to the prompt. Hindsight can use Ollama, LM Studio or llama.cpp as its LLM, but it is a memory server, not a local runner.

