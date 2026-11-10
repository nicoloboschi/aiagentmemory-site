# Claude Code Skills: How They Work, Where They Live and Memory

> How Claude Code skills work: SKILL.md location (personal, project, plugin), frontmatter fields and limits, how they load, and how skills differ from CLAUDE.md and memory. Sourced from the Claude Code docs.

Source: https://aiagentmemory.org/articles/claude-code-skills/ · Updated 2026-11-10 · aiagentmemory.org


**Claude Code skills are folders that hold a `SKILL.md` file with instructions for one kind of task.** Claude reads each skill's name and description at the start of a session. It loads the full instructions only when the task matches, or when you type `/skill-name`. Skills do not store facts about your work. For that, use CLAUDE.md files, auto memory, or a memory plugin.

This guide covers where skills live, the fields you can set, how they load, and what they do with memory. The source is the [Claude Code skills docs](https://code.claude.com/docs/en/skills), checked on 10 November 2026.

## Where do Claude Code skills live?

Put a skill in one of these folders. The scope decides who gets it.

| Level | Path | Who gets it |
|---|---|---|
| Personal | `~/.claude/skills/<skill-name>/SKILL.md` | You, in all projects on this machine |
| Project | `.claude/skills/<skill-name>/SKILL.md` | Anyone in that repository (commit it to share) |
| Nested | `<subdir>/.claude/skills/<skill-name>/SKILL.md` | Sessions started in or below that subfolder |
| Plugin | `<plugin>/skills/<skill-name>/SKILL.md` | Anyone with the plugin on, called as `/plugin-name:skill-name` |
| Enterprise | Managed settings folder, for example `/etc/claude-code/.claude/skills/` on Linux | Everyone on machines where the organization deploys it |

Old command files in `.claude/commands/<name>.md` still work and create the same `/name` command.

## How do you write a skill?

A skill is one file. It starts with YAML frontmatter between `---` lines on the first line, then plain Markdown instructions.

1. Make a folder: `mkdir -p ~/.claude/skills/release-notes`.
2. Write `SKILL.md` with a `name` and a `description`. Put the main use case first in the description.
3. Add the steps Claude should follow, in plain Markdown.
4. Start Claude Code and type `/release-notes`, or just ask for the task and let Claude load the skill when it matches.

```yaml
---
name: release-notes
description: Write release notes from the git log since the last tag. Use when asked for a changelog or release summary.
disable-model-invocation: true
allowed-tools: Bash(git log *)
---

Write release notes for $ARGUMENTS from the commits since the last tag.
1. Group commits into features, fixes and breaking changes.
2. Keep each line under 100 characters.
```

Every field is optional, but `description` is the one that matters, because Claude uses it to pick the skill. The fields you will use most:

| Field | What it does |
|---|---|
| `name` | Command name in the `/` menu. Defaults to the folder name. |
| `description` | What the skill does and when to use it. |
| `disable-model-invocation` | `true` means only you can load it, with `/name`. Default `false`. |
| `user-invocable` | `false` hides it from the `/` menu, so only Claude loads it. |
| `allowed-tools` | Tools Claude can use without asking while the skill runs. |
| `context` | `fork` runs the skill in a separate subagent. |
| `paths` | Glob patterns that limit when Claude loads the skill. |

## What are the limits?

- **Description length:** the description and `when_to_use` text are cut at **1,536 characters** in the skill list. Put the key use case first.
- **File length:** keep `SKILL.md` **under 500 lines**. Move long reference text into other files in the same folder.
- **Exact names:** field names must match exactly. Unknown fields are ignored without an error, so a typo fails silently.
- **Compaction:** when a long session is summarized, Claude Code re-attaches the latest use of each skill, keeping the first **5,000 tokens** of each, with **25,000 tokens** across all skills.

## How do skills load?

- **Descriptions are always in context.** Every skill's name and description takes a little room in every session. Many long descriptions add up.
- **The body loads on use.** The full `SKILL.md` enters the conversation as one message and stays there for later turns.
- **Control from settings.** The `skillOverrides` setting can switch a skill to `on`, `name-only`, `user-invocable-only` or `off` without editing the file. Permission rules such as `Skill(name)` can allow or deny specific skills.

## Do skills count as memory?

No. A skill holds instructions that you wrote. It does not learn from your sessions. Claude Code has two other places for knowledge that carries across sessions:

| | CLAUDE.md files | Auto memory | Skills |
|---|---|---|---|
| Who writes it | You | Claude | You (or a plugin) |
| What it holds | Rules and project facts | Notes from your corrections | How to do one kind of task |
| Loaded | Every session | Every session (first 200 lines or 25 KB) | Name and description always; body when used |

The rule of thumb: if Claude should always know it, put it in CLAUDE.md. If Claude should know how to do one job, make a skill. If Claude should remember facts across many sessions, add a memory system. See [Claude Code memory: CLAUDE.md, auto memory and plugins](/articles/claude-code-memory/) for the details.

## Where does Hindsight fit?

[Hindsight](https://github.com/vectorize-io/hindsight) is an open-source memory server (MIT license, full server included). It has a Claude Code integration, so Claude can recall and store facts through tools instead of only reading text files.

- **Where it helps:** long histories across many sessions and repositories. Its memory layer keeps facts in a store, and `reflect` can answer a question by reasoning over them. The skill layer does not do this on its own.
- **Limit 1, cost:** saving facts calls an LLM. Retain and consolidation both cost tokens, so a skill that saves on every turn costs more than one that saves rarely.
- **Limit 2, context:** recall returns more text per call by default. Set `max_tokens` lower if a session gets crowded.
- **Limit 3, overkill:** for a project with a few fixed rules, CLAUDE.md is enough. Add Hindsight when the rules change and the history gets too big to read by hand.

## FAQ

**What is a Claude Code skill?** A folder with a `SKILL.md` file: a description and instructions Claude loads when the task matches. Claude sees the name and description at all times and loads the body when it is used.

**Do Claude Code skills remember things?** Not by themselves. They are fixed instructions. Facts go in CLAUDE.md, auto memory, or a memory plugin.

**Where do Claude Code skills go?** Personal skills in `~/.claude/skills/`, project skills in `.claude/skills/`, and plugin skills inside the plugin folder.


## FAQ

### What is a Claude Code skill?

A skill is a folder with a SKILL.md file: a short description plus instructions Claude Code can load when the task matches. Claude sees every skill's name and description, and loads the full body only when it is used. Skills live in ~/.claude/skills/ (personal), .claude/skills/ (project) or inside a plugin.

### Do Claude Code skills remember things between sessions?

Not by themselves. A skill is fixed instructions you wrote. Facts Claude learns about your project are kept in CLAUDE.md files and auto memory. A memory plugin such as Hindsight can store and recall facts across sessions.

### Where do Claude Code skills go?

Personal skills go in ~/.claude/skills/<skill-name>/SKILL.md and apply to all your projects. Project skills go in .claude/skills/<skill-name>/SKILL.md and apply to that repository. Plugin skills live in the plugin folder and are called as /plugin-name:skill-name.

