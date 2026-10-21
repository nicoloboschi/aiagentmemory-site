# Procedural Memory in AI Agents: Prompts, Skills, Workflows

> Procedural memory in AI agents and LLMs: weights, code, prompts and skill files, how Voyager, AWM and Memp learn procedures, and LangMem prompt optimization.

Source: https://aiagentmemory.org/articles/ai-agent-procedural-memory/ · Updated 2026-10-12 · aiagentmemory.org


**Procedural memory in AI agents** is the agent's knowledge of how to do things: the rules and skills that shape how it acts. For LLM agents it lives in three places: the model's weights, the agent's code and its prompts. Most current work on learning procedures edits the text parts, because rewriting text is cheaper and safer than retraining the model.

It's the memory type agents use on every single step, and the one developers change most carefully. This page covers what counts as procedural memory, how research agents learn new procedures, and how to optimize prompts from feedback. For the full set of memory types, see [AI agent memory explained](/articles/ai-agent-memory-explained/).

## What is procedural memory in AI agents?

**Procedural memory in an AI agent is stored knowledge of how to perform tasks: implicit skill in the LLM's weights, plus explicit instructions in the agent's code, system prompt, tool definitions and skill files. Unlike episodic or semantic memory, it shapes how the agent acts rather than what it knows.**

The term comes from psychology, where [procedural memory](https://en.wikipedia.org/wiki/Procedural_memory) covers skills like riding a bike. LangGraph's [memory guide](https://docs.langchain.com/oss/python/langgraph/memory) applies it directly: for AI agents, procedural memory is "a combination of model weights, agent code, and agent's prompt that collectively determine the agent's functionality."

## Implicit vs explicit procedural memory

The CoALA framework ([Sumers et al., 2023](https://arxiv.org/abs/2309.02427)) splits procedural memory into two forms.

| Form | Where it lives | Example | How it changes | Cost and risk of change |
|---|---|---|---|---|
| **Implicit** | LLM weights | Knowing how to write Python, follow a chat format | Fine-tuning, RLHF, distillation | High cost; hard to inspect or undo |
| **Explicit: code** | Agent source code | Tool functions, retrieval logic, the agent loop | A developer edits it, or the agent writes new code skills | Testable, but bugs break every run |
| **Explicit: prompts** | System prompt, instruction files | "Always run tests before opening a PR" | Edits by a developer, or prompt optimization from feedback | Cheap; easy to version and roll back |
| **Explicit: skills** | Skill files loaded on demand | A `SKILL.md` describing how to file an expense report | Added by people or saved by the agent | Cheap; loaded only when relevant |

CoALA makes two points that still hold. First, procedural memory can't start empty: "procedural memory must be initialized by the designer." Second, writing to it is "significantly riskier than writing to episodic or semantic memory, as it can easily introduce bugs or allow an agent to subvert its designers' intentions."

LangGraph's docs add a practical note: agents rarely change their weights or code, but "it is more common for agents to modify their own prompts."

## How research agents learn procedures

Three lines of work show agents building procedural memory from experience, without touching model weights. All numbers below are the authors' own reported results.

### Voyager: a library of code skills

[Voyager](https://arxiv.org/abs/2305.16291) (Wang et al., 2023) plays Minecraft with GPT-4 and keeps "an ever-growing skill library of executable code." When a generated program succeeds, it's saved as a named skill (CoALA's examples: `combatZombie`, `craftStoneSword`) and retrieved later by embedding similarity. Complex skills call simpler ones. The authors report 3.3x more unique items and tech-tree milestones up to 15.3x faster than prior methods, and the library carried over to a new world.

### Agent Workflow Memory: reusable routines

[Agent Workflow Memory](https://arxiv.org/abs/2409.07429) (AWM, Wang et al., 2024) induces "commonly reused routines, i.e., workflows" from past web-navigation runs and gives the relevant ones to the agent on new tasks. It reports relative success-rate gains of 24.6% on Mind2Web and 51.1% on WebArena, with fewer steps per solved task. It works offline (from training examples) or online (from test queries as they come).

### Memp: build, retrieve and update procedures

[Memp](https://arxiv.org/abs/2508.06433) (Fang et al., 2025) distills past trajectories into two forms: step-by-step instructions and higher-level, script-like abstractions. It studies build, retrieval and update strategies separately, including a regimen that "updates, corrects, and deprecates" entries. On TravelPlanner and ALFWorld, success rates rose as the repository improved. Procedural memory built with a stronger model also helped when moved to a weaker one.

A related approach, [ReasoningBank](/articles/reasoningbank-llm-agent-memory-framework/), stores distilled reasoning strategies from both successful and failed runs, which sits between episodic and procedural memory.

## Procedural memory in production agents

Most shipped agents store procedural memory as plain text files that people and agents can both edit.

- **System prompts and instruction files.** `AGENTS.md`, `CLAUDE.md` and similar files hold project rules that load every session. Deep Agents loads "persistent instructions and preferences" from `AGENTS.md` files at startup. More on this pattern in [AI coding agent memory](/articles/ai-coding-agent-memory/).
- **Agent Skills.** The [Agent Skills](https://agentskills.io/) format, first developed by Anthropic and released as an open standard, packages "procedural knowledge" as folders with a `SKILL.md` file. Agents load skills by **progressive disclosure**: only each skill's name and description at startup, the full instructions when a task matches, and bundled files only when needed. Claude Code, Codex, Gemini CLI, Cursor, GitHub Copilot and Letta are among the listed clients.
- **Self-editing agents.** Letta's docs describe agents that "learn skills, rewrite their own system prompts," with background "dreaming" subagents that consolidate lessons into memory.
- **Memory servers with standing answers.** [Hindsight](https://github.com/vectorize-io/hindsight) keeps **mental models**: answers to questions you define (for example "how does this team want PRs reviewed?") that it keeps up to date from new memories in the background. Reading one is a database read, with no retrieval or LLM call. The difference from a skill file is that nobody edits it by hand, so review what it says before you let it steer the agent.

Progressive disclosure matters for scale. A system prompt holds every rule on every call. A skill library can hold hundreds of procedures while spending only a line of context on each one until it's needed.

## LLM procedural memory optimization with LangMem

Prompt optimization is the most practical way to update procedural memory today: feed past conversations and feedback to an LLM, and let it propose a better system prompt. LangMem's `create_prompt_optimizer` does this. Adapted from LangMem's [prompt optimization guide](https://langchain-ai.github.io/langmem/guides/optimize_memory_prompt/):

```python
from langmem import create_prompt_optimizer

trajectories = [
    # A conversation with no annotation: the optimizer infers what went wrong
    (
        [
            {"role": "user", "content": "Tell me about Mars"},
            {"role": "assistant", "content": "Mars is the fourth planet..."},
            {"role": "user", "content": "I wanted more about its moons"},
        ],
        None,
    ),
    # A conversation with explicit feedback
    (
        [
            {"role": "user", "content": "What are Mars' moons?"},
            {"role": "assistant", "content": "Mars has two moons: Phobos and Deimos..."},
        ],
        {"score": 0.9, "comment": "Should include more details and recommended follow-up questions"},
    ),
]

optimizer = create_prompt_optimizer(
    "anthropic:claude-sonnet-4-5",
    kind="metaprompt",  # also "gradient" (2-10 LLM calls) or "prompt_memory" (1 call)
    config={"max_reflection_steps": 1, "min_reflection_steps": 0},
)
new_prompt = optimizer.invoke(
    {"trajectories": trajectories, "prompt": "You are a planetary science expert"}
)
print(new_prompt)
```

The three `kind` options trade cost for depth. `prompt_memory` makes a single LLM call. `gradient` splits the work between one call that proposes improvements and one that applies them. `metaprompt` uses reflection steps you can bound with `config`.

LangGraph's docs describe the same loop without a library: store the instructions in the memory store, have one node rewrite them from conversation feedback, and have the main node read the latest version each run.

## Guardrails for writable procedural memory

Because procedural memory runs on every future task, treat updates like code changes:

1. **Version every change.** Keep prompts and skills in git or a store with history, so you can diff and roll back.
2. **Separate proposal from approval.** Let the agent propose prompt or skill edits; have a person or an evaluation gate approve them.
3. **Evaluate before shipping.** Run the new prompt on a fixed test set and compare it with the old one.
4. **Scope updates.** A rule learned from one user's feedback shouldn't silently change the agent for everyone.
5. **Watch for poisoning.** An attacker who can get text into the agent's learned instructions controls future behavior. Don't let untrusted content write procedural memory directly.
6. **Deprecate stale procedures.** Memp's results suggest removing outdated entries matters as much as adding new ones.

For how procedural memory relates to the other types, and how to decide which ones an agent needs, see [types of AI agent memory](/articles/ai-agents-memory-types/).


## FAQ

### What is procedural memory in AI agents?

Procedural memory is the agent's knowledge of how to do things. In the CoALA framework it has two parts: implicit knowledge in the LLM's weights and explicit knowledge in the agent's code and prompts. In practice it shows up as the system prompt, tool code, skill files and learned workflows.

### How do LLM agents learn procedural memory?

Mostly by editing text, not weights. Agents rewrite their own instructions from feedback (LangMem's prompt optimizer), save working code as reusable skills (Voyager), or distill past trajectories into reusable workflows (Agent Workflow Memory, Memp). Fine-tuning the model is the costly alternative.

### Why is writing procedural memory risky?

Procedural memory controls behavior on every future task. CoALA notes that writing to it is significantly riskier than writing to episodic or semantic memory because it can introduce bugs or let an agent subvert its designers' intentions. Version prompt and skill changes and review them before they go live.

