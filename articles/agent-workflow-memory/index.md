# Agent Workflow Memory (AWM): How It Works and Results

> Agent Workflow Memory (AWM, ICML 2025) explained: how agents induce reusable workflows from past runs, online vs offline modes, WebArena and Mind2Web results, limits.

Source: https://aiagentmemory.org/articles/agent-workflow-memory/ · Updated 2026-10-25 · aiagentmemory.org


**Agent workflow memory** means an LLM agent keeps reusable routines learned from its own past runs and follows them on new tasks. The best-known version is **Agent Workflow Memory (AWM)** by Wang, Mao, Fried and Neubig (CMU and MIT, [ICML 2025](https://proceedings.mlr.press/v267/wang25bx.html)). AWM induces workflows from past trajectories and puts the relevant ones in the agent's prompt.

On web navigation with GPT-4, the paper reports a 51.1% relative gain on WebArena and 24.6% on Mind2Web. This page explains how AWM builds workflows, the difference between its online and offline modes, the full results, the limits found by the authors and by later work, and how to build workflow memory into an agent today. Sources: the [arXiv paper](https://arxiv.org/abs/2409.07429), the ICML 2025 proceedings version, and the [official repo](https://github.com/zorazrw/agent-workflow-memory), checked on 12 October 2026.

## What is agent workflow memory?

**Agent workflow memory is a store of reusable task routines, called workflows, that an agent extracts from past action trajectories. Each workflow has a short goal description and a sequence of steps with example-specific values replaced by variables. Relevant workflows are added to the agent's context so it can follow a proven routine instead of planning from zero.**

The AWM paper defines a workflow as "commonly reused routines" and compares the idea to how people learn task routines from experience. A workflow is smaller than a whole task. "Buy dry cat food on Amazon" is a task; "search for a product on Amazon" is a workflow, because it shows up inside many tasks.

In memory terms, workflows are **procedural memory**: knowledge of how to do things, stored as text. Our [procedural memory guide](/articles/ai-agent-procedural-memory/) covers the wider category (prompts, skill files, code libraries). This page goes deep on AWM itself.

## How Agent Workflow Memory works

AWM adds two steps to a normal observe-and-act agent loop: **induce** workflows from experiences, then **integrate** them into memory. In the paper, memory is just text in the prompt, starting with the documentation of built-in actions like `click` and `type`.

### What a workflow looks like

Each workflow has two parts:

1. **A description**: a short summary of the goal, such as "This workflow aims to find a customer order with specified ID."
2. **A trajectory of steps**. Each step holds a plain-language description of the page state, the agent's reasoning, and an executable action like `click('126')` or `stop()`.

Here is one workflow the paper induced on Mind2Web (shopping domain):

```text
# shopping: search and sort
Given that you are on the Amazon search results page, this workflow searches for a product and sorts the results.
[textbox] Search Amazon -> TYPE: {search-term}
[button] Go -> CLICK
[span] Sort by: -> CLICK
[option] {sort-option} -> CLICK
```

### The induction step

An LLM reads one or more past experiences (instruction plus action trajectory) and writes workflows. The prompt, from the paper's appendix, asks the model to "find the repetitive subset of actions across multiple tasks," to avoid "similar or overlapping workflows," to give each workflow at least two steps, and to replace non-fixed values with descriptive variable names. The output is split on blank lines and each workflow is stored separately.

The authors also tried a rule-based induction (deduplicate trajectories by action sequence, drop invalid steps). On WebArena it scored about the same (35.6% vs 35.5%), but on Mind2Web the LLM version won by 1.7 step-success points, because abstract sub-routines transfer better than whole copied trajectories.

### Workflows build on workflows

Online, workflows get more complex over time. On the WebArena map site, the agent first learned "find a place by its name," then reused those steps to build "get the zip code of a place." The paper calls this a snowball effect and shows most of the gain arriving in the first 40 or so tasks.

### Online vs offline induction

| | AWM offline | AWM online |
|---|---|---|
| Input | Annotated training examples (human or synthetic) | Only the test queries, in a stream |
| When workflows are made | Once, before inference ("training" time) | After each task, if it passes the check |
| Quality check | None needed; examples are ground truth | An LLM evaluator labels each run success or failure |
| Memory during the run | Fixed set for every test task | Grows task by task |
| Strength | Best when training data matches the test tasks | Adapts to new websites and domains |
| Risk | Train-test gap (e.g. training on Amazon shopping, testing on Amazon careers) | A wrong "success" label becomes a wrong workflow |

Both modes run per website: examples are grouped by site and each site gets its own small workflow set. Combining the two (offline workflows to start, online ones added later) did not add up. It scored between the two, and the authors say the offline workflows seemed to hurt the online ones.

## Agent Workflow Memory results on WebArena and Mind2Web

All numbers below are from the paper's own experiments with `gpt-4-0613` (and `gpt-3.5-turbo` on Mind2Web), temperature 0.

### WebArena

WebArena has 812 tasks across five self-hosted websites. It has no training set, so only online AWM was tested. Task success rate (Table 1):

| Method | Total SR | Shopping | CMS | Reddit | GitLab | Maps | Steps |
|---|---|---|---|---|---|---|---|
| SteP (14 human-written workflows) | 33.0 | 37.0 | 24.0 | 59.0 | 32.0 | 30.0 | - |
| WebArena baseline | 14.9 | 14.0 | 11.0 | 6.0 | 15.0 | 16.0 | - |
| AutoEval | 20.2 | 25.5 | 18.1 | 25.4 | 28.6 | 31.9 | 46.7 |
| BrowserGym | 23.5 | - | - | - | - | - | - |
| BrowserGym (accessibility tree only) | 15.0 | 17.2 | 14.8 | 20.2 | 19.0 | 25.5 | 7.9 |
| **AWM (online)** | **35.5** | 30.8 | 29.1 | 50.9 | 31.8 | 43.3 | **5.9** |

The 51.1% relative gain is 35.5 against BrowserGym's 23.5. Against BrowserGym with the same page input as AWM (accessibility tree only), the gap is wider: 15.0 to 35.5. AWM also beat SteP, which uses human-written workflows, though SteP stays ahead on Shopping and Reddit. On a cross-template subset (one task per template, so workflows can't just copy a twin task) AWM scored 33.2% against 32.1% for SteP and 20.5% for BrowserGym.

The ICML version adds a cost breakdown: trajectory evaluation and workflow induction add **10.8%** compute on top of action generation (about 3,600 extra tokens on a ~33,700-token task).

### Mind2Web

Mind2Web scores each step (did the agent pick the right element and action?) and the whole task (were all steps right?). Offline results on the cross-task split (Table 3):

| Method | Element acc. | Action F1 | Step SR | Task SR |
|---|---|---|---|---|
| MindAct (GPT-3.5) | 20.3 | 56.6 | 17.4 | 0.8 |
| Synapse (GPT-3.5) | 34.0 | - | 30.6 | 2.4 |
| AWM (GPT-3.5) | 39.0 | 52.8 | 34.6 | 2.8 |
| MindAct (GPT-4) | 41.6 | 60.6 | 36.2 | 2.0 |
| **AWM (GPT-4)** | **50.6** | 57.3 | **45.1** | **4.8** |

The 24.6% relative gain is step success, 36.2 to 45.1. Synapse retrieves whole past examples; AWM's abstract workflows did better, which the authors explain as less bias toward copying elements from a concrete example.

Generalization with GPT-4, step success rate (Table 4):

| Method | Cross-task | Cross-website | Cross-domain |
|---|---|---|---|
| MindAct | 36.2 | 30.1 | 18.6 |
| AWM offline | 45.1 | 33.7 | 32.6 |
| AWM online | 43.6 | 33.9 | **35.5** |

Online AWM pulls ahead as test data moves further from training data, because it only learns from the test environment.

## Limits of Agent Workflow Memory

AWM is a strong idea with clear limits. Some come from the paper, some from later work.

- **Whole-task success stays low on Mind2Web.** Step success rose a lot, but task success went from 2.0% to 4.8%. Agents still fail most full tasks.
- **Following workflows too closely.** Action F1 dropped below MindAct's (57.3 vs 60.6). The authors say the agent struggles to know "where to diverge from the workflow guidelines."
- **Bad online workflows.** Online induction trusts an LLM judge. When it labels a wrong run as a success, a wrong workflow enters memory.
- **Only successes are used.** Failed runs are thrown away. The [ReasoningBank paper](/articles/reasoningbank-llm-agent-memory-framework/) (Google, 2025) found that feeding failures to AWM lowered its WebArena-Shopping success from 44.4 to 42.2.
- **Gains shrink with newer models.** In ReasoningBank's re-run on WebArena, AWM beat no memory with Gemini 2.5 Flash (44.1 vs 40.5), tied with Gemini 2.5 Pro (47.6 vs 46.7) and scored below no memory with Claude 3.7 Sonnet (40.8 vs 41.7). On tasks that span several websites, AWM fell to 3.4% from 10.3% (Gemini 2.5 Flash). These are another team's numbers in a different setup.
- **Workflows as tools are rarely used.** Wrapping workflows as callable actions (AWM-AS) added only 1.3 step-success points, and the agent called them in just 18.5% of tasks. Fixed action sequences also break on dynamic pages, like a flight form whose airport pop-up isn't known in advance.
- **Narrow testing.** The paper covers web navigation only, mostly GPT-4, with workflows kept per website. Nothing removes or updates a stale workflow; memory only grows.
- **Always in the prompt.** AWM puts every workflow for the site in the prompt before the agent starts. The [REM paper](https://arxiv.org/abs/2610.27377) (Chen et al., October 2026), on repository bug fixing rather than web tasks, argues the opposite: give a short phase-by-phase summary of a related past run at the start, hold the detailed record back until the agent seems stuck, and drop it after one use. On SWE-ContextBench Lite that fixed 27.6% of bugs against 24.9% for the prior memory system it was compared with, at 17% lower cost. The authors conclude that "the bottleneck in experience reuse is not retrieval fidelity" but how experience is stored and governed. Details are on our [coding agent memory](/articles/ai-coding-agent-memory/) page.

## How to implement workflow memory today

The [official repo](https://github.com/zorazrw/agent-workflow-memory) (Apache 2.0, about 480 stars) is research code: `pipeline.py` scripts for WebArena and Mind2Web, not a library. For your own agent, the pattern is small enough to build yourself:

1. **Log every run** as instruction plus steps (state, reasoning, action).
2. **Judge the outcome.** Use real signals first (tests passed, API returned 200, user confirmed). Fall back to an LLM judge only when you have nothing better.
3. **Induce workflows** from successful runs with an LLM, using a prompt like AWM's: sub-task level, variables for specific values, no overlap.
4. **Scope them.** AWM keeps one set per website. Use a scope that matches your agent: per site, per repo, per tool or per customer.
5. **Retrieve before acting.** Put the workflows for the current scope (or the top matches by similarity) in the prompt.
6. **Review and expire.** Keep a version, an owner and a last-used date on each workflow. Delete ones that stop working; AWM itself never does.
7. **Keep failures somewhere.** Store failed runs as lessons, not as workflows, so the agent learns what to avoid.

A minimal induction step with the OpenAI Python SDK, using the AWM prompt and a JSON file as the store:

```python
import json
from pathlib import Path
from openai import OpenAI

client = OpenAI()
STORE = Path("workflows.json")

INDUCE_PROMPT = """Given a list of web navigation tasks, your task is to extract the common workflows.
Each given task contains a natural language instruction, and a series of actions to solve the task.
You need to find the repetitive subset of actions across multiple tasks, and extract each of them out as a workflow.
Each workflow should be a commonly reused sub-routine of the tasks. Do not generate similar or overlapping workflows.
Each workflow should have at least two steps. Represent the non-fixed elements (input text, button strings)
with descriptive variable names as shown in the example."""

def load() -> dict:
    return json.loads(STORE.read_text()) if STORE.exists() else {}

def induce(scope: str, experiences: list[str]) -> list[str]:
    """Turn successful runs into workflows and save them under a scope (e.g. a website)."""
    resp = client.chat.completions.create(
        model="gpt-4o",
        temperature=0,
        messages=[
            {"role": "system", "content": INDUCE_PROMPT},
            {"role": "user", "content": "\n\n".join(experiences)},
        ],
    )
    # AWM splits the model output on blank lines, one workflow per block
    new = [w.strip() for w in resp.choices[0].message.content.split("\n\n") if w.strip()]
    store = load()
    store[scope] = store.get(scope, []) + new
    STORE.write_text(json.dumps(store, indent=2))
    return new

def workflow_context(scope: str) -> str:
    """Text to add to the agent's system prompt before it acts."""
    return "## Useful workflows\n" + "\n\n".join(load().get(scope, []))
```

The online loop is then: run the task with `workflow_context(site)` in the prompt, judge the result, and call `induce(site, [run_log])` when it succeeded.

## Workflow memory vs procedural memory and other experience memory

Workflow memory is one way to store procedural memory. Agents learn from past runs in several other ways, and they store different things:

| Approach | What it stores | Learns from failures | Where it's used |
|---|---|---|---|
| Agent Workflow Memory | Abstract step-by-step routines with variables | No | Prompt (or as callable actions) |
| Synapse | Whole past trajectories as examples | No | Prompt |
| [Voyager](https://arxiv.org/abs/2305.16291) | Working code for each solved sub-task | No (only saves code that works) | Callable skill library |
| ReasoningBank | Short strategy notes (title, description, content) | Yes | Prompt |
| Plan caching | Plan templates from finished runs | No | Adapted by a small model |
| [REM](https://arxiv.org/abs/2610.27377) | A small phase-structured summary per finished coding run, plus a detailed record | Not stated | Summary at task start; detailed record only when the agent struggles, then dropped |
| Hindsight | Experiences, consolidated observations, mental models | Yes (outcomes are stored like any experience) | Recall or `reflect` before acting |

Workflows answer "what steps worked for this kind of task?" Reasoning notes answer "what should I keep in mind?" Plans answer "what's the order of work for this task?", covered in [planning memory for AI agents](/articles/ai-agent-planning-memory/). Raw run logs are episodic memory; see [episodic memory in AI agents](/articles/episodic-memory-in-ai-agents/).

## Where Hindsight fits

[Hindsight](https://github.com/vectorize-io/hindsight) is an MIT-licensed memory server. It doesn't implement AWM and doesn't induce executable workflows. What it gives you is the storage and learning layer around the same idea: an agent that improves from past experience.

- **Experiences**: record each run and its outcome with `retain`; an LLM extracts the facts, time and entities.
- **Observations**: related experiences are merged in the background into evidence-backed beliefs with quotes and a proof count, refined as new runs arrive instead of overwritten. Repeated outcomes on the same site can end up as one statement with its supporting evidence.
- **Mental models**: standing answers kept up to date in the background, such as "how do we deploy service X?", that the agent reads with no retrieval or LLM call.
- **`reflect`**: reasons over many memories to answer "what went wrong last time we tried this?", which covers the failure lessons AWM drops.
- **Retrieval**: four strategies in parallel (semantic, BM25, graph, temporal) with reranking, inside a token budget you set.

```python
from hindsight_client import Hindsight

hs = Hindsight(base_url="http://localhost:8888")

hs.retain(
    bank_id="web-agent-gitlab",
    content="Task: list contributors of repo a11y-kit. Searched the repo, opened Contributors, "
            "read the table. Succeeded in 4 steps.",
    tags=["success"],
)

lessons = hs.reflect(bank_id="web-agent-gitlab",
                     query="What steps work for finding repo contributors, and what failed before?")
print(lessons.text)
```

You can also store your induced AWM workflows in a bank and recall them by similarity instead of loading a whole JSON file.

Hindsight's lead on long histories is real: on its own benchmark site it scores 94.6 on LongMemEval, and Mem0's comparison page shows it ahead of Mem0 by 9.8 and 15.5 points on BEAM 1M and 10M. Those are conversation-memory benchmarks, not web-agent tests, and they're vendor-run. The limits for this use: every `retain` and consolidation calls an LLM, so log once per task, not per click, and it returns more context per retrieval than Mem0 by default, so set `max_tokens`. For a single agent on one website, a JSON file of workflows like the one above may be all you need. More in [Hindsight explained](/articles/ai-memory-hindsight/).


## FAQ

### What is Agent Workflow Memory?

Agent Workflow Memory (AWM) is a method from CMU and MIT (Wang, Mao, Fried and Neubig, ICML 2025) that lets an LLM agent learn reusable routines. An LLM extracts common sub-tasks from past action trajectories, replaces specific values with variables, and adds these workflows to the agent's prompt for later tasks.

### What is the difference between online and offline AWM?

Offline AWM induces workflows from annotated training examples before the agent runs, then uses the same fixed set for every test task. Online AWM needs no training data: after each test task, an LLM evaluator judges whether the run succeeded, and successful runs become new workflows for the next task.

### How much does Agent Workflow Memory improve agents?

With GPT-4, the paper reports WebArena success of 35.5% versus 23.5% for BrowserGym (51.1% relative) and Mind2Web step success of 45.1% versus 36.2% for MindAct (24.6% relative), with about 2 fewer steps per WebArena task. These are the authors' own numbers. Later work (ReasoningBank) found smaller or negative gains with newer models.

