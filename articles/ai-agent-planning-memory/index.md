# AI Agent Planning Memory: Keeping and Reusing Plans

> How AI agents remember plans: to-do lists in working memory, plans stored as episodes, and plan reuse through workflow memory and plan caching, with code.

Source: https://aiagentmemory.org/articles/ai-agent-planning-memory/ · Updated 2026-10-12 · aiagentmemory.org


**AI agent planning memory** is the memory an agent uses to make, follow and reuse plans. It has three parts: the **active plan** kept in working memory (usually a to-do list), **past plans and outcomes** stored as episodes, and **reusable plan templates** extracted from earlier runs. The first keeps long tasks on track; the others make the next plan cheaper.

"Planning memory" isn't a separate store in the standard taxonomy. It's a job that working, episodic and procedural memory do together. This page shows how each one supports planning, with research results and a working example. For the memory types themselves, see [AI agent memory explained](/articles/ai-agent-memory-explained/).

## What is planning memory in AI agents?

**Planning memory in an AI agent is the information it keeps to form and execute multi-step plans: the current goal and steps with their status, records of how earlier plans worked out, and reusable plan templates or workflows. It lets the agent stay on task across many steps and avoid replanning from scratch.**

In the CoALA framework ([Sumers et al., 2023](https://arxiv.org/abs/2309.02427)), planning is a stage of every decision cycle. The agent uses reasoning and retrieval to **propose** candidate actions, **evaluate** them, and **select** one, then executes it. Retrieval is what links planning to memory: CoALA says episodes "may be retrieved into working memory to support reasoning" during planning, and argues that interleaving memory search with planning "can help to better ground planning."

| Part of planning memory | Memory type (CoALA) | Lifetime | Example |
|---|---|---|---|
| Active plan and step status | Working memory | One task | A to-do list with pending, in-progress and done items |
| Past plans and outcomes | Episodic memory | Across tasks | "Last migration plan failed at step 3: lock timeout" |
| Reusable plans and workflows | Procedural memory | Across tasks | "To book a flight: search, filter by date, compare, checkout" |

## Keeping the active plan in working memory

Agents lose the plot on long tasks. The first instruction ends up thousands of tokens back, surrounded by tool output (why that hurts is covered in [short-term memory in AI agents](/articles/short-term-memory-ai-agents/)). The fix most production agents use is an explicit plan the agent keeps rewriting.

### To-do lists as recitation

Manus reported that a typical task takes "around 50 tool calls on average," and that the agent often creates a `todo.md` file and checks items off as it goes ([Manus, 2025](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)). Rewriting the list pushes the objectives into the end of the context, where the model attends most. Manus calls this "manipulating attention through recitation."

LangChain packages the same idea as `TodoListMiddleware`, which gives an agent a `write_todos` tool and a prompt that tells it to mark items done as soon as each is finished. Deep Agents makes it an opt-in harness feature (opt-in since v0.7) and recommends it for long tasks and for "less capable models that benefit from an explicit accountability tool" ([Deep Agents docs](https://docs.langchain.com/oss/python/deepagents/overview)).

### Hierarchical plans stored in memory

[Generative Agents](https://arxiv.org/abs/2304.03442) (Park et al., 2023) plan top-down. Each agent drafts a day in roughly five to eight chunks, then breaks each chunk into hour-long actions, then into 5 to 15 minute actions. Plans go into the same memory stream as observations, so retrieval can surface them. When something unexpected happens, the agent regenerates the plan from that point.

Storing the plan in memory, not just in the prompt, has a practical benefit: the plan survives summarization and trimming of the message history.

## A to-do list agent with LangChain

The plan lives in agent state under `todos`, so a checkpointer keeps it across turns of the same thread:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware
from langchain_core.tools import tool
from langgraph.checkpoint.memory import InMemorySaver

@tool
def run_tests(path: str) -> str:
    """Run the test suite at the given path and return a summary."""
    return "42 passed, 1 failed: test_orders_migration"

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[run_tests],
    middleware=[TodoListMiddleware()],  # adds the write_todos tool
    checkpointer=InMemorySaver(),
)

config = {"configurable": {"thread_id": "release-checklist"}}
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Prepare the release: run tests, fix failures, update the changelog."}]},
    config,
)
for item in result.get("todos", []):
    print(item["status"], "-", item["content"])  # pending / in_progress / completed
```

The middleware's own prompt warns that writing todos costs time and tokens and should be skipped for simple requests. Use it for tasks with many steps or many tools.

## Remembering how past plans went

The second part of planning memory is episodic: what was tried before and what happened.

- **Keep failures visible.** Manus's guidance is to "keep the wrong stuff in": leaving failed actions and error traces in context helps the model avoid repeating them. Hiding errors removes the evidence it needs.
- **Reflect after failure.** [Reflexion](https://arxiv.org/abs/2303.11366) (Shinn et al., 2023) has the agent write a short verbal lesson after a failed attempt and feeds it into the next try. It reported 91% pass@1 on HumanEval this way, against 80% for GPT-4 at the time.
- **Store plan outcomes across sessions.** Saving "plan, result, reason" records lets a later task retrieve similar attempts before planning. The patterns for this are in [episodic memory in AI agents](/articles/episodic-memory-in-ai-agents/). Memory servers can hold these records for you: [Hindsight](https://github.com/vectorize-io/hindsight), for example, stores what the agent did as **experiences**, merges repeated outcomes into evidence-backed **observations**, and can answer "what went wrong last time we tried this?" with its `reflect` call. Each write costs an LLM call, so log outcomes per task, not per tool call.

## Reusing plans: workflows and plan caches

The third part is reuse. If an agent has solved a kind of task before, it shouldn't have to plan from zero.

| Approach | What it stores | How it's matched | Reported result (authors' own numbers) |
|---|---|---|---|
| [Agent Workflow Memory](https://arxiv.org/abs/2409.07429) (Wang et al., 2024) | Reusable workflows induced from past trajectories | Workflows relevant to the new task are added to the prompt | +24.6% and +51.1% relative success on Mind2Web and WebArena, fewer steps |
| [Agentic Plan Caching](https://arxiv.org/abs/2506.14852) (Zhang et al., 2025) | Plan templates extracted from completed runs | Keyword extraction matches new requests; a small model adapts the template | 50.31% lower cost and 27.28% lower latency on average, with performance maintained |
| Voyager skill library (Wang et al., 2023) | Working code for each solved subtask | Embedding similarity | Reused skills to solve new tasks in a new Minecraft world |

Plan caching targets cost. The APC authors note that normal LLM caches don't fit agents well, because agent outputs depend on external data and context, so they cache the *plan* rather than the answer. Workflow memory targets quality: the agent follows a routine that already worked.

Once a plan template is reused across many tasks, it's effectively procedural memory. That means it needs the same care as a system prompt: versioning, review and removal when it goes stale. See [procedural memory in AI agents](/articles/ai-agent-procedural-memory/).

## Design tips for planning memory

1. **Make the plan explicit.** A structured list with statuses beats a plan buried in a chain of thought.
2. **Keep it near the end of the context.** Rewrite or re-inject the plan each step instead of leaving it at the top.
3. **Store the plan in state, not only in messages.** Then trimming or summarizing history doesn't erase it.
4. **Log outcomes per step.** Success, failure and the reason, so later plans can learn from them.
5. **Retrieve before planning.** Search past episodes and workflows for similar tasks first.
6. **Promote proven plans carefully.** Turn a plan into a reusable workflow only after it has worked more than once.
7. **Skip planning for short tasks.** For a two-step request, a to-do list is overhead.


## FAQ

### What is planning memory in AI agents?

Planning memory is the memory an agent uses to make, track and reuse plans. It covers the active plan held in working memory (often a to-do list), past plans and their outcomes stored as episodes, and reusable plan templates or workflows extracted from earlier successful runs.

### Why do AI agents write to-do lists?

Long tasks push the original goal far back in the context, where the model attends to it less. Rewriting a to-do list keeps the goal and remaining steps near the end of the context. Manus describes this as manipulating attention through recitation, and LangChain ships it as the write_todos tool.

### Can AI agents reuse plans from earlier tasks?

Yes. Agent Workflow Memory (Wang et al., 2024) extracts reusable workflows from past runs and reported 24.6% and 51.1% relative success gains on Mind2Web and WebArena. Agentic Plan Caching (Zhang et al., 2025) stores plan templates and reported 50.31% lower cost and 27.28% lower latency on average.

