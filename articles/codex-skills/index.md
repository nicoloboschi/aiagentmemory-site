# Codex Skills: Folders, Install and Disable (2026)

> How Codex skills work: the SKILL.md folder, where Codex loads it from, how to call a skill with $, install one, and turn one off. Sourced from the OpenAI Codex skills docs, read 2026-11-13.

Source: https://aiagentmemory.org/articles/codex-skills/ · Updated 2026-11-13 · aiagentmemory.org


**A Codex skill is a folder with a `SKILL.md` file.** The file holds a name, a description and the steps for one job. Codex sees the name and description at the start, and reads the full steps only when a task matches. Put the skill in the right folder, and Codex finds it on its own or when you type `$` and its name.

## What is in a skill folder?

A skill is a folder. Only `SKILL.md` is required. The other parts are optional:

```text
my-skill/
  SKILL.md          required: instructions and metadata
  scripts/          optional: executable code
  references/       optional: documentation
  assets/           optional: templates and resources
  agents/openai.yaml   optional: UI text, invocation policy, tool dependencies
```

The top of `SKILL.md` needs two fields, `name` and `description`:

```yaml
---
name: skill-name
description: Explain exactly when this skill should and should not trigger.
---
```

Write the description so Codex can match it. The docs say to put the key use case and trigger words first, because a host may shorten the text.

## How Codex decides to use a skill

Codex starts with each skill's name, description and file path. This list uses at most 2% of the model's context window, or 8,000 characters when the window size is unknown. If you have many skills, Codex shortens the descriptions first, and it can drop some from the list with a warning. Set `skills.max_context_tokens` (up to 10,000 tokens) to change the budget.

A skill runs in one of two ways:

- **Explicit:** you name it in the prompt. In the Codex CLI or the IDE extension, type `$` and the skill name, or run `/skills`.
- **Implicit:** Codex picks the skill because your task matches its description.

Set `allow_implicit_invocation: false` in `agents/openai.yaml` if a skill should run only when you name it.

## Where Codex loads skills from

Codex reads skills from five places. The table is from the docs, read 2026-11-13.

| Scope | Location | Use it for |
|---|---|---|
| Repo, working folder | `$CWD/.agents/skills` | Skills for one module or service |
| Repo, folders above | `$CWD/../.agents/skills` up to the repo root | Skills shared by a parent folder |
| Repo, root | `$REPO_ROOT/.agents/skills` | Skills for everyone in the repository |
| User | `$HOME/.agents/skills` | Your own skills, in any repository |
| Admin | `/etc/codex/skills` | Machine-wide skills for scripts and shared machines |
| System | Bundled with Codex | `skill-creator` and `skill-installer`, on by default |

Codex supports symlinked skill folders. It notices changes on its own. If a new skill does not show up, restart Codex.

## Create a skill

You have two options:

1. **Use the built-in creator.** In Codex, run `$skill-creator`. It asks what the skill does, when it should trigger, and whether it needs scripts. The default is instructions only.
2. **Write it by hand.** Make the folder, add `SKILL.md`, and keep it to one job. The docs advise instructions over scripts, unless you need a fixed result or an outside tool.

## Install a skill

The bundled `skill-installer` adds curated skills to your local Codex. For example, `$skill-installer linear` installs the Linear skill from the `openai/skills` repository. You can also point it at other repositories.

Installed skills are for your own setup. To share a skill with other people, or to bundle two or more skills with a connector, package them as a plugin.

## Turn a skill off

Add an entry to `~/.codex/config.toml`:

```toml
[[skills.config]]
path = "/path/to/skill/SKILL.md"
enabled = false
```

Use one selector per entry: a full path to one `SKILL.md`, or a name that matches every skill with that name. If more than one entry matches, the last one wins. Restart Codex after the change. To turn off the bundled system skills in one project, set `[skills.bundled] enabled = false` in that project's trusted `.codex/config.toml`.

## Do skills give Codex memory?

No. A skill holds steps that you wrote or installed. Codex reads the same file each time, and the docs do not describe Codex writing new skills from past work. The `skill-creator` only runs when you ask it to.

Codex's own memory is a separate feature. [Codex memory: AGENTS.md, memories and add-ons](/articles/codex-memory/) covers AGENTS.md rules and the optional memories under `~/.codex/memories/`.

For memory that lasts across sessions, Hindsight's Coding Agents package can add it to Codex CLI. It captures each session automatically and recalls it for the same repository. This is a different tool from skills. Its setup and limits are in [how to install Codex CLI](/articles/codex-cli-install/#memory-built-in-and-add-ons). Two limits matter here: the package needs a Hindsight server (cloud, your own, or a local daemon) to be running, and every recall adds tokens to the prompt. Skills cost nothing to keep, but they only hold what you wrote.

Hermes Agent goes further: it writes and edits its own skills. See [Hermes Agent skills](/articles/hermes-agent-skills/). For the Claude Code version of the same format, see [Claude Code skills](/articles/claude-code-skills/).

## Related

- [How to install Codex CLI](/articles/codex-cli-install/)
- [Codex CLI config reference](/articles/codex-cli-config-reference/)
- [Codex memory](/articles/codex-memory/)
- [Claude Code skills](/articles/claude-code-skills/)
- [Procedural memory in AI agents](/articles/ai-agent-procedural-memory/)

Sources: [Codex skills docs](https://developers.openai.com/codex/skills), read 2026-11-13. Hindsight facts: [hindsight.vectorize.io coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12. Hindsight is open source at [github.com/vectorize-io/hindsight](https://github.com/vectorize-io/hindsight).


## FAQ

### What is a Codex skill?

A Codex skill is a folder with a SKILL.md file. The file has a name, a description and instructions for one job. Codex reads the name and description first, and loads the full instructions only when it decides to use the skill. Skills follow the open agent skills standard.

### Where does Codex look for skills?

Codex scans .agents/skills in your working folder and each parent folder up to the repository root. It also reads $HOME/.agents/skills for your user, /etc/codex/skills for the machine, and the bundled system skills. Two skills with the same name both show up; Codex does not merge them.

### How do I turn off a Codex skill?

Add a [[skills.config]] entry to ~/.codex/config.toml with the skill's path or name and enabled = false, then restart Codex. The skill folder stays on disk.

