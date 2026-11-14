# Codex Subagents: Built-in Agents, Custom Files and Settings (2026)

> How Codex subagents work: the three built-in agents, how to ask Codex to spawn them, custom agent files in .codex/agents/, and the [agents] settings. From OpenAI's Codex docs, read 2026-11-14.

Source: https://aiagentmemory.org/articles/codex-subagents/ · Updated 2026-11-14 · aiagentmemory.org


## What is a Codex subagent?

A **Codex subagent** is a helper that Codex starts to handle one part of a task. Codex runs several of them in parallel, waits for all of them, then returns one answer in the main chat. Each subagent does its own model and tool work. Its thread is shown in the app, the CLI and the IDE extension, so you can open it and read what it did.

The use case is noise. Exploration notes, test logs and stack traces fill the main chat and make it less reliable over time. Subagents move that work off the main thread and send back only a summary. Use them for read-heavy work such as exploration, tests and triage. Be careful with several agents editing code at the same time, because their edits can conflict.

For a Claude Code version of the same idea, see [Claude Code subagents](/articles/claude-code-subagents/).

## Which agents come built in?

Codex ships with three agents. You do not have to write them.

| Agent | What it is for |
|---|---|
| `default` | General-purpose fallback |
| `worker` | Doing the work: implementation and fixes |
| `explorer` | Read-heavy look at the codebase |

If you name a custom agent with the same name as a built-in one, yours wins.

## How do you ask for subagents?

Say it in the prompt. Codex does not start subagents on its own unless a direct request or a project or skill instruction asks for them. The docs give this pattern:

> Review this branch with parallel subagents. Spawn one subagent for security risks, one for test gaps, and one for maintainability. Wait for all three, then summarize the findings by category with file references.

A good prompt says three things: how to split the work, whether Codex should wait for all agents, and what the summary should contain. Set the sandbox and approval mode on the parent session before you ask. Subagents inherit them, and a custom agent file cannot give a subagent a looser sandbox.

## How do you write a custom agent?

Add one TOML file per agent. Use `~/.codex/agents/` for your own agents, or `.codex/agents/` for a project you share. Each file needs three fields:

| Field | Required | What it does |
|---|---|---|
| `name` | Yes | The name Codex uses to spawn or refer to the agent |
| `description` | Yes | When Codex should use this agent |
| `developer_instructions` | Yes | How the agent behaves |

Optional fields change the model: `model`, `model_reasoning_effort`, `model_reasoning_summary` and `model_verbosity`. To cut what an agent can do, set these to `false`: `features.shell_tool`, `features.apps`, `features.plugins`, `features.memories`, `features.request_permissions_tool`. Individual skills can be turned off with `skills.config`.

A read-only explorer looks like this:

```toml
name = "pr_explorer"
description = "Read-only codebase explorer for gathering evidence before changes are proposed."
model_reasoning_effort = "high"
developer_instructions = """
Stay in exploration mode.
Do not edit files.
Trace the real execution path and cite files and symbols.
"""
```

Two things a custom agent cannot change: `sandbox_mode`, `approval_policy`, `model_provider` and `mcp_servers` stay as set in the parent session. The docs say that a custom agent "can't override those settings", and that instructions asking an agent not to edit do not create a read-only sandbox. Put MCP servers in the parent config so every agent can use them.

## Which global settings control subagents?

They live under `[agents]` in your config file:

| Setting | Default and effect |
|---|---|
| `agents.enabled` | `true`. Set to `false` to turn off multi-agent tools |
| `agents.max_concurrent_threads_per_session` | Caps open subagent threads. Unset means Codex picks the default |
| `agents.default_subagent_model` | Model for spawned agents. A spawn request can override it |
| `agents.default_subagent_reasoning_effort` | Reasoning effort for spawned agents |
| `agents.interrupt_message` | `true`. Adds a note to the agent's context when its turn is interrupted |

If you set no model or effort, a subagent inherits the parent's. Setting a model in a custom file wins over the `[agents]` default.

For the rest of the config keys, see the [Codex CLI config reference](/articles/codex-cli-config-reference/).

## Where does memory fit?

Codex has its own memory feature, and a custom agent can switch it off with `features.memories = false`. Subagent memory is a separate question:

- **Codex's own memory:** [Codex memory](/articles/codex-memory/) covers the main feature. A read-only agent can turn it off, as above.
- **Hindsight:** the Coding Agents package installs for Codex CLI (see [Codex CLI install](/articles/codex-cli-install/)). It captures sessions automatically, with one memory bank per repository. The Hindsight page does not say whether subagent threads are captured or recalled. Do not assume they are. Test it: ask a subagent a question that only an earlier session can answer.
- **MCP servers:** subagents inherit the parent's MCP servers. An MCP memory server set up in the parent config is therefore available to them too. See [Codex MCP](/articles/codex-mcp/) for how to add one.
- **Cost:** every subagent does its own model work, so a memory lookup inside each one adds tokens. Hindsight returns more context per recall than Mem0 by default, so set a lower token budget if recall is large.
- **Benchmarks:** Hindsight reports 64.1 on BEAM 10M against Mem0's 48.6. Both figures come from Mem0's own comparison page, not from an independent test.

## Sources

- OpenAI Codex docs, subagents: developers.openai.com/codex/subagents (it redirects to learn.chatgpt.com/docs/agent-configuration/subagents), read 2026-11-14. Built-in agents, spawning, the `[agents]` table, the custom agent fields and the feature keys.
- Hindsight Coding Agents docs, hindsight.vectorize.io/sdks/integrations/coding-agents, read 2026-11-14. The install command and the one-bank-per-repository rule. No subagent detail.
- BEAM 10M figures (64.1 and 48.6): Mem0's comparison page.


## FAQ

### What is a Codex subagent?

A subagent is a helper that Codex starts to do one task in parallel, such as exploring the code or running tests. It does its own model and tool work, then returns a summary to the main chat. Each subagent uses its own tokens, so a subagent run costs more than a single-agent run.

### How do I make Codex use subagents?

Ask for them in the prompt, for example 'spawn one agent per point, wait for all of them, and summarize'. Codex only spawns subagents after a direct request or when a project or skill instruction asks for it. Subagents are on by default in current releases; set agents.enabled to false to turn them off.

### Where do custom Codex agents go?

Put one TOML file per agent in .codex/agents/ for one project, or in ~/.codex/agents/ for all projects. Each file needs a name, a description and developer_instructions. Codex uses the name field, not the file name.

### Do Codex subagents remember things between sessions?

The Codex docs do not describe a per-subagent memory. Subagents inherit the parent's MCP servers, and a custom agent can turn off the features.memories setting. Whether a subagent's work is saved to a memory server such as Hindsight is not documented, so test it before you rely on it.

