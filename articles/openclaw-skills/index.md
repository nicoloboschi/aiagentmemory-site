# OpenClaw Skills (2026): Install from ClawHub, Write SKILL.md, Safety

> How to find, install, update and write OpenClaw skills: the openclaw skills commands, the SKILL.md format and its metadata fields, and what to check before you install a third-party skill. Sourced from the OpenClaw ClawHub and skill format docs, read 2026-11-11.

Source: https://aiagentmemory.org/articles/openclaw-skills/ · Updated 2026-11-11 · aiagentmemory.org


**OpenClaw skills** are folders of instructions that teach the agent a task, such as a calendar, a
Todoist client or a database backup. You install them from ClawHub, the public registry, or write
your own. A skill is not memory. It does not store what the agent learns. For that, see
[OpenClaw memory](/articles/openclaw-ai-agent-memory/) and the [OpenClaw guide](/articles/openclaw-guide/).

Sources: the OpenClaw ClawHub docs (documentation.openclaw.ai/clawhub) and the skill format page
(documentation.openclaw.ai/clawhub/skill-format). Both read 2026-11-11.

## The commands

| What you want | Command | Notes from the docs |
|---|---|---|
| Find a skill | `openclaw skills search "calendar"` | Searches ClawHub |
| Install one | `openclaw skills install @openclaw/demo` | Records the source so updates stay on ClawHub |
| Update all | `openclaw skills update --all` | Native OpenClaw command |
| Publish or delete | `clawhub` CLI (`npm i -g clawhub`) | Needs `clawhub login`. Used for publishing, delete and undelete |
| Install a plugin, not a skill | `openclaw plugins install clawhub:<package>` | Plugins are a different thing from skills |

The docs say the native `openclaw` commands install into the active OpenClaw workspace. The separate
`clawhub` CLI installs into `./skills` under the folder you run it from, and writes `.clawhub/lock.json`.
Use one of the two for each skill, so you know where it lives.

## What a skill folder holds

| Part | Required | What it does |
|---|---|---|
| `SKILL.md` | Yes | Markdown with optional YAML frontmatter. The instructions the agent reads |
| Other files | No | Any regular files in the folder, for example scripts or templates. Total bundle size limit: 50 MB |
| `.clawhubignore` or `.gitignore` | No | Files left out when you publish |

The docs also accept `skill.md`; the legacy `skills.md` still works. For portable Agent Skills, the
`name` should match the folder name and use 1 to 64 lowercase letters, numbers or hyphens.

## The frontmatter fields that matter

```yaml
---
name: todoist
description: Manage tasks via the Todoist API.
version: 1.0.0
metadata:
  openclaw:
    requires:
      env:
        - TODOIST_API_KEY
      bins:
        - curl
    primaryEnv: TODOIST_API_KEY
    emoji: "✅"
    homepage: https://github.com/example/todoist-cli
---
```

- `description` is the summary in the registry and search.
- `requires.env` lists variables the skill cannot run without. Put optional ones under `envVars` with
  `required: false`, not in `requires.env`.
- `requires.bins` lists CLI tools that must all exist. `requires.anyBins` means at least one must exist.
- `install` can install dependencies with `brew`, `node`, `go` or `uv`.
- `os` limits a skill to systems such as `macos` or `linux`.

The docs say ClawHub's security analysis checks that what the skill declares matches what its code uses.
If the code reads an API key that the frontmatter does not declare, the analysis flags it.

## Before you install a skill

1. Read its `SKILL.md`. Look for shell commands, network calls and any variable it asks you to set.
2. Check the public scan summary on its ClawHub page. ClawHub runs automated checks on published skills.
   A held or blocked release can disappear from the catalog. Third-party guides warn that a "Pass" is not
   a guarantee.
3. Give it only the keys it needs. Use a test key first.

ClawHub is open by default: anyone with a GitHub account old enough to pass the upload gate can publish.
Treat a skill like a script you download from the internet.

## Write your own

1. Make a folder named after the skill, with a `SKILL.md`.
2. Add `name` and `description`. Add `metadata.openclaw.requires` for every variable and binary.
3. Test it in a chat session before you publish.
4. Publish with `clawhub skill publish <path> --slug <slug> --name "<name>" --version 1.0.0`.

Each publish makes a new semver version. Skills are MIT-0 licensed. ClawHub does not support paid skills.

## Where Hindsight fits

A skill can call a memory tool, but it does not bring memory with it. Hindsight is a separate plugin,
`@vectorize-io/hindsight-openclaw`. Since v1.2.0 (released 2026-11-03) it gives OpenClaw the
`hindsight_retain`, `hindsight_recall` and `hindsight_reflect` tools, with a `memoryMode` setting:
`hybrid` (default), `context` or `tools`.

- **Its best-sourced number:** on the BEAM benchmark at 10M tokens, Hindsight scored 64.1 and Mem0 48.6.
  Mem0 ran that comparison on its own site, so it is vendor-run.
- **Its open-source part:** the full server is MIT licensed, including graph recall and reflect.
- **Its limits:** a retrieval returns more context than Mem0's by default (set `max_tokens` lower to change
  it). Each save calls an LLM, so writes cost more. Failed saves are held in memory only, so a crash
  while the server is down loses them. (The source is the Hermes plugin's v1.2.0 notes; check the OpenClaw plugin before you rely on it.)
- **A skill's risk is separate:** a third-party skill that reads your memory is one more thing to audit.

Install Hindsight only if the file memory stops being enough. Read the [OpenClaw memory page](/articles/openclaw-ai-agent-memory/)
first.

## FAQ

**How do I install an OpenClaw skill?** Search with `openclaw skills search`, then run
`openclaw skills install @owner/<slug>`. Update all with `openclaw skills update --all`.

**What is a SKILL.md file?** The instruction file at the top of a skill folder. Its frontmatter names the skill,
its summary and the variables it needs.

**Is it safe to install skills from ClawHub?** Read the file and the scan result first. Treat third-party
skills as untrusted code.

**Do skills give the agent memory?** No. Use the workspace memory files or a memory plugin such as Hindsight.

Checked 2026-11-11 against the OpenClaw ClawHub and skill format docs. Commands may change between
OpenClaw releases; run `openclaw skills --help` to check yours.


## FAQ

### How do I install an OpenClaw skill?

Run openclaw skills search "<topic>" to find one, then openclaw skills install @owner/<slug>. Run openclaw skills update --all to update them. The commands come from the OpenClaw ClawHub docs, read 2026-11-11.

### What is a SKILL.md file in OpenClaw?

A skill is a folder with a SKILL.md file: Markdown with optional YAML frontmatter. The description field is the summary shown in the registry and in search. The frontmatter can also list the environment variables, CLI binaries and config files the skill needs.

### Is it safe to install skills from ClawHub?

Treat third-party skills as untrusted. ClawHub is open by default, so anyone can publish. Read the SKILL.md and the public scan result before you install. Third-party guides warn that a "Pass" scan is not a guarantee. Check the scan details yourself.

### Do OpenClaw skills give the agent memory?

Not by themselves. A skill is a set of instructions and tools. OpenClaw's memory is the Markdown files in the workspace, or a memory plugin such as Hindsight, which is a separate install.

