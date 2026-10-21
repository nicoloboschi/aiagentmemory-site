# AI Agent Architecture Patterns: ReAct to Multi-Agent

> The main AI agent architecture patterns compared: ReAct, plan-and-execute, ReWOO, LLMCompiler, Reflexion, workflows, multi-agent, and where memory fits in each.

Source: https://aiagentmemory.org/articles/ai-agent-architecture-patterns/ · Updated 2026-10-12 · aiagentmemory.org


**AI agent architecture patterns** are the standard ways to arrange LLM calls, tools and memory into an agent. The main ones are **ReAct** (think, act, observe in a loop), **plan-and-execute** (plan first, then run steps), **ReWOO** and **LLMCompiler** (plan all tool calls up front), **Reflexion** (learn from failed attempts), fixed **workflows**, and **multi-agent** setups. Each puts memory in a different place.

## What are AI agent architecture patterns?

**An agent architecture pattern is a reusable control structure that decides when the LLM is called, what it sees, how it uses tools, and what state carries between steps.** The model is the same in every pattern. What changes is the loop around it.

Anthropic's [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) (December 2024) draws the main line. **Workflows** are "systems where LLMs and tools are orchestrated through predefined code paths." **Agents** are "systems where LLMs dynamically direct their own processes and tool usage." Both start from the same block: "an LLM enhanced with augmentations such as retrieval, tools, and memory."

The research patterns below are mostly agents in that sense. The workflow patterns come after.

## Pattern comparison

| Pattern | Control flow | LLM calls | Memory it relies on | Good for | Weak at |
|---|---|---|---|---|---|
| ReAct | Reason, act, observe, repeat | One per step | Growing scratchpad in context | Open-ended tool use, Q&A over APIs | Long tasks, cost, losing track |
| Plan-and-execute | Plan once, execute steps, replan | Few planner calls plus cheap executors | Plan and step results as state | Multi-step tasks with clear subgoals | Plans that break on surprises |
| ReWOO | Plan with variables, run tools, solve | Planner + solver (2 calls) | Plan with placeholders for evidence | Token efficiency, predictable tools | Steps that depend on surprises |
| LLMCompiler | Plan a task graph, run in parallel | Planner plus joiner | Task DAG and results | Independent tool calls, latency | Highly sequential tasks |
| Reflexion | Try, evaluate, reflect, retry | Several per attempt | Episodic memory of reflections | Tasks with a clear success signal | Tasks with no feedback |
| Multi-agent | Lead delegates to specialized agents | Many | Shared state or isolated contexts | Broad, parallel research | Token cost, coordination |

## Core reasoning patterns

### ReAct: reason and act in a loop

[ReAct](https://arxiv.org/abs/2210.03629) (Yao et al., ICLR 2023) has the model alternate between a **thought** (reasoning text), an **action** (a tool call), and an **observation** (the tool result) until it answers. The paper reports that reasoning traces "help the model induce, track, and update action plans as well as handle exceptions." On ALFWorld and WebShop, ReAct beat imitation and reinforcement learning baselines by 34% and 10% absolute success rate.

**Where memory sits:** the whole trajectory stays in the prompt as a scratchpad. That's simple, but each step resends everything before it, so long runs get expensive and the model can lose track of the goal. Most "tool-calling agent" loops in current frameworks are ReAct-style.

### Plan-and-execute and Plan-and-Solve

**Plan-and-execute** splits planning from doing. A planner LLM writes a step list; an executor (often a cheaper model or a ReAct sub-loop) runs each step; a replanner revises the plan when results come back. The idea traces to [Plan-and-Solve prompting](https://arxiv.org/abs/2305.04091) (Wang et al., ACL 2023), which first devises a plan that splits a task into subtasks and then carries them out, cutting missing-step errors compared with zero-shot chain-of-thought.

**Where memory sits:** the plan itself is state, stored outside the prompt and updated as steps finish. That gives long tasks an anchor. Anthropic's research system uses the same trick: its lead agent saves its plan to memory because context past 200,000 tokens gets truncated ([multi-agent research write-up](https://www.anthropic.com/engineering/multi-agent-research-system)). More on planning state in [AI agent planning memory](/articles/ai-agent-planning-memory/).

### ReWOO and LLMCompiler: plan the tool calls up front

**ReWOO** ([Xu et al., 2023](https://arxiv.org/abs/2305.18323), "Reasoning WithOut Observation") writes the whole plan with placeholders like `#E1` for evidence not yet fetched. Workers fill the placeholders by running tools, and a solver writes the answer. Because the LLM isn't re-prompted after every observation, the paper reports 5x token efficiency and a 4% accuracy gain on HotpotQA.

**LLMCompiler** ([Kim et al., ICML 2024](https://arxiv.org/abs/2312.04511)) goes further: a planner builds a dependency graph of tool calls, a task-fetching unit dispatches them, and an executor runs independent ones in parallel. Compared with ReAct, the authors report up to 3.7x lower latency, 6.7x cost savings and about 9% better accuracy.

**Where memory sits:** a structured plan with variable bindings. It's compact, but there's little room to react to an unexpected result mid-plan without a replanning step.

### Reflexion: learning from failed attempts

[Reflexion](https://arxiv.org/abs/2303.11366) (Shinn et al., 2023) improves an agent through verbal feedback instead of weight updates. After an attempt fails, the agent writes a reflection on what went wrong and stores it in **episodic memory**. The next attempt reads those reflections first. The paper reports 91% pass@1 on HumanEval, against 80% for GPT-4.

**Where memory sits:** this is the pattern where long-term memory is the core mechanism. It needs a success signal (tests, a checker, an evaluator) to know when to reflect. The same idea underlies agent memory systems that store lessons from past runs; see [episodic memory in AI agents](/articles/episodic-memory-in-ai-agents/).

Memory servers now do this across sessions. [Hindsight](https://github.com/vectorize-io/hindsight), for example, merges related facts from past runs into **observations** (beliefs backed by quotes and a proof count) and refines them as new evidence arrives, so a lesson learned once is recalled on later tasks. The cost is an extra LLM call on every write.

## Workflow patterns: predefined paths

Anthropic's guide lists five workflow patterns for cases where the steps are known in advance:

1. **Prompt chaining:** each LLM call processes the output of the previous one.
2. **Routing:** classify an input and send it to a specialized follow-up.
3. **Parallelization:** run LLM calls at the same time and combine the outputs in code.
4. **Orchestrator-workers:** a central LLM breaks down the task, delegates to worker LLMs, and merges results.
5. **Evaluator-optimizer:** one call generates, another evaluates and gives feedback, in a loop.

The guide's advice is to "find the simplest solution possible, and only increasing complexity when needed," and to start with LLM APIs directly before adding a framework. Workflows need little memory beyond passing outputs forward, which is part of why they're easier to debug.

## Multi-agent patterns

Multi-agent systems split work across several LLM agents. LangChain's [multi-agent docs](https://docs.langchain.com/oss/python/langchain/multi-agent) name the common shapes and how each handles context:

- **Subagents:** a main agent calls subagents as tools. Subagents are stateless and start fresh each time, which isolates context.
- **Handoffs:** agents pass control to each other via tool calls; state persists across turns.
- **Skills:** one agent loads specialized prompts and knowledge on demand.
- **Router:** a routing step sends input to one or more specialized agents and combines the results.

Anthropic's research system is an orchestrator-worker setup: a lead agent (Claude Opus 4) delegates to parallel subagents (Claude Sonnet 4). It outperformed single-agent Opus 4 by 90.2% on Anthropic's internal research eval, but multi-agent runs use about 15x the tokens of a chat, against about 4x for a single agent (self-reported).

**Where memory sits:** either in shared state every agent reads, or in separate per-agent contexts that only pass back condensed results. Isolation keeps each context small; shared state keeps agents consistent. Framework support for both is compared in [AI agent framework comparison](/articles/ai-agent-framework-comparison/).

## Choosing a pattern

1. **Fixed, known steps?** Use a workflow (chain, router, parallel).
2. **Open-ended tool use, short tasks?** Use ReAct.
3. **Long tasks with clear subgoals?** Use plan-and-execute with replanning.
4. **Many independent tool calls?** Use LLMCompiler-style parallel planning.
5. **Clear pass/fail signal and repeated attempts?** Add Reflexion-style episodic memory.
6. **Broad research that splits cleanly?** Use orchestrator-workers, and budget for the token cost.

Whatever the pattern, the agent forgets everything between sessions unless you add long-term memory. How to add it is covered in [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What are the main AI agent architecture patterns?

The core single-agent patterns are ReAct (interleaved reasoning and tool calls), plan-and-execute (plan first, then run steps), ReWOO and LLMCompiler (plan tool calls up front, run them without re-prompting, in parallel for LLMCompiler) and Reflexion (retry with stored self-critique). Above these sit fixed workflows like prompt chaining and routing, and multi-agent patterns such as orchestrator-workers and handoffs.

### What is the difference between ReAct and plan-and-execute?

ReAct decides one step at a time, calling the LLM after every tool result, so it adapts quickly but uses many calls. Plan-and-execute writes a full plan first and then executes it, which uses fewer expensive planning calls and keeps long tasks on track, but it needs a replanning step when results don't match the plan.

### Where does memory fit in an agent architecture?

Every pattern has working memory, the scratchpad of thoughts and tool results in the current context. Patterns differ in what they store beyond it: plan-and-execute keeps the plan as state, Reflexion stores lessons in episodic memory, and multi-agent systems keep shared state or give each subagent its own isolated context.

