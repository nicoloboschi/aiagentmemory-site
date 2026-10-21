# Hermes Agent Skills: How Procedural Memory Works

> How Hermes Agent skills work: how the agent writes SKILL.md files from experience, where they live, how they load, the Skills Hub, the curator, and limits.

Source: https://aiagentmemory.org/articles/hermes-agent-skills/ · Updated 2026-10-21 · aiagentmemory.org


**Hermes Agent skills** are the agent's procedural memory: markdown files that record how to do a task. Hermes writes them itself after working out a multi-step job, patches them when it hits a new pitfall, and loads one only when a task needs it. Each skill is a `SKILL.md` folder under `~/.hermes/skills/`, in the open Agent Skills format.

[Hermes Agent](https://github.com/NousResearch/hermes-agent) is the MIT-licensed agent from Nous Research, with about 256,000 GitHub stars on 21 October 2026. This page covers how its skills are made, stored, loaded, shared and cleaned up. Facts come from the official [Skills System docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills), the [Curator docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/curator) and the v0.22 release notes, checked 21 October 2026. For the fact files and session search, see [Hermes Agent memory](/articles/hermes-agent-memory/).

## What are Hermes Agent skills?

**A Hermes skill is a folder with a `SKILL.md` file: YAML front matter (name, description, version, tags) and a body with when to use it, the steps, the pitfalls and how to check the result. The agent sees every skill's name and description, and reads the full file only when a task matches.**

The docs put it plainly: when the agent "figures out a non-trivial workflow, it saves the approach as a skill for future reuse." That's what separates skills from memory. Memory says what is true ("the user lives in PST"). A skill says how to do something ("how to deploy to Kubernetes"). The Hermes guide's rule of thumb: "If you'd put it in a reference document, it's a skill. If you'd put it on a sticky note, it's memory."

Skills are not unique to Hermes. The format follows the [Agent Skills specification](https://agentskills.io/specification), which Claude Code, Codex and others also read, so a skill folder can move between tools. What Hermes adds is the loop around it: the agent writes and edits its own skills, and a background job prunes the ones nobody uses. The wider idea is covered in [procedural memory in AI agents](/articles/ai-agent-procedural-memory/).

## How Hermes creates and improves skills

Skills get written in four ways:

1. **During a task.** The system prompt asks the agent to save a non-trivial workflow with the `skill_manage` tool. The docs list three triggers: it worked out a multi-step workflow worth repeating, it hit dead ends and found the working path, or you corrected its approach.
2. **On a nudge.** The default `skills.creation_nudge_interval` is 15, so Hermes reminds the model to consider saving a skill every 15 tool-calling iterations (per `cli-config.yaml.example` in the repo, October 2026). Set it to 0 to turn the nudge off.
3. **In the background review.** After a turn, a forked copy of the agent can save memory or create and patch skills. Since v0.22.0 (14 October 2026), the release notes say "background review can improve any skill," and corrections go into the skill for that task.
4. **On request with `/learn`.** Point it at a folder, a URL, a pasted procedure or "how I just deployed the staging server," and the agent writes a skill from it. A large source such as a book becomes a short `SKILL.md` plus one file per topic under `references/`.

`skill_manage` has six actions: `create`, `patch` (a targeted find-and-replace, the preferred way to update), `patch` with full `content` (a rewrite), `delete`, `write_file` and `remove_file`. The docs say a targeted patch "is more token-efficient than a full rewrite."

Hermes also tells the agent what a good skill looks like: "lessons, not logs." A pitfall should be a general rule plus a short reason, not the story of the incident. An advisory linter runs on each write and warns on three shapes: a body full of PR or issue numbers, more than 60 reference files, and a `SKILL.md` body over about 24,000 characters. It warns; it never blocks.

## Where skills live and the SKILL.md format

Everything sits in `~/.hermes/skills/`, which the docs call "the primary directory and source of truth." Bundled skills are copied there on install, and hub installs and agent-made skills land there too. Skills are grouped in category folders:

```text
~/.hermes/skills/
├── devops/
│   └── deploy-k8s/
│       ├── SKILL.md        # required
│       ├── references/     # extra docs, loaded on demand
│       ├── templates/
│       ├── scripts/
│       └── assets/
├── .hub/                   # lock.json, quarantine/, audit.log
├── .usage.json             # view, use and patch counts per skill
└── .bundled_manifest       # hashes of seeded bundled skills
```

A minimal `SKILL.md`, in the shape the docs give:

```markdown
---
name: deploy-k8s
description: Deploy a service to our Kubernetes cluster
version: 1.0.0
platforms: [macos, linux]
metadata:
  hermes:
    tags: [kubernetes, deploy]
    category: devops
    requires_toolsets: [terminal]
---

# Deploy to Kubernetes

## When to Use
## Procedure
## Pitfalls
## Verification
```

Hermes reads skills from more than one place. When names clash, the docs give the order as `project → local (~/.hermes/skills/) → skills.create_dir → external_dirs`:

- **Project skills** in `<repo>/.hermes/skills/` or `<repo>/.agents/skills/` load only after you run `hermes skills trust` in that repo, and each one is scanned first. A skill the scanner rates dangerous is quarantined.
- **`skills.create_dir`** sends new agent-made skills somewhere else, such as a git-tracked folder.
- **`skills.external_dirs`** adds shared folders like `~/.agents/skills`. The docs warn these are "not a write-protection boundary": if Hermes can write there, the agent can edit them.

Each profile has its own skills folder and manifest, so `hermes -p coder` has a separate library.

## How skills get into context

Skills load by **progressive disclosure**, in three levels:

| Level | Call | What the agent gets | Cost |
|---|---|---|---|
| 0 | `skills_list()` | Name, description and category of every skill | About 3k tokens, at session start |
| 1 | `skill_view(name)` | The full `SKILL.md` | Varies |
| 2 | `skill_view(name, path)` | One reference file | Varies |

The Hermes docs estimate the level-0 skill list at about 3,000 tokens (Skills System docs, checked October 2026). The model reads that list and decides which skill to open; Hermes doesn't document any embedding or ranking step for that choice, so treat selection as the model's own judgment.

You can also load skills yourself:

- **Slash commands.** Every installed skill is a command: `/github-pr-workflow create a PR for the auth refactor`. Up to 5 leading `/skill` tokens stack in one message.
- **Bundles.** A YAML file in `~/.hermes/skill-bundles/` groups skills under one command, such as `/backend-dev`.
- **Auto-load.** `skills.auto_load` in `config.yaml` loads named skills in full at the start of every session.
- **Conditional activation.** Front matter fields like `fallback_for_toolsets: [web]` hide a skill when a tool is present, so a free search skill appears only when the paid web tool is missing.

The skill list is part of the system prompt, which stays fixed so the provider's prompt cache keeps working. A skill added mid-chat is announced in a short note on your next message, and the system prompt itself picks it up at the next compaction.

## The Skills Hub: installing and sharing skills

The **Skills Hub** is Hermes's skill installer. It searches several registries and installs from them with a security scan. In the repo on 21 October 2026, Hermes shipped 58 bundled `SKILL.md` files under `skills/` and 153 optional ones under `optional-skills/` (counted from the repo tree).

| Source | Example identifier | Trust level |
|---|---|---|
| `official` (optional skills in the Hermes repo) | `official/security/1password` | Built-in trust |
| `github`, default taps | `openai/skills/k8s` (also `anthropics/skills`, `huggingface/skills`, `NVIDIA/skills`) | Trusted |
| `skills-sh` (Vercel's directory) | `skills-sh/vercel-labs/json-render/json-render-react` | Community |
| `well-known` (a site's `/.well-known/skills/index.json`) | `well-known:https://mintlify.com/docs/.well-known/skills/mintlify` | Community |
| `url` (any direct `SKILL.md` link) | `https://example.com/SKILL.md` | Community |
| `clawhub`, `lobehub`, `browse-sh` | Source-specific | Community |

To install and share a skill:

1. Search with `hermes skills search kubernetes` (or `/skills search` in chat).
2. Preview with `hermes skills inspect openai/skills/k8s`.
3. Install with `hermes skills install openai/skills/k8s`. The scan checks for data exfiltration, prompt injection, destructive commands and supply-chain signals. `--force` overrides warnings but never a `dangerous` verdict.
4. Start a new session (or `/reset`) so it's in the skill list.
5. Run `hermes skills check` and `hermes skills update` later. Skills you edited locally are skipped unless you pass `--force`.
6. To share your own, push a GitHub repo with a `skills/<name>/SKILL.md` layout. Others run `hermes skills tap add owner/repo`, or install one skill with `hermes skills install owner/repo/skills/my-workflow`.

Hub operations use the GitHub API, which allows 60 unauthenticated requests per hour, so set `GITHUB_TOKEN` if installs hit rate limits (Skills System docs, October 2026). Plugins can also ship namespaced skills (`plugin:skill`), which stay out of the skill list until loaded by name; see [Hermes Agent plugins](/articles/hermes-agent-plugins/).

## Keeping skills healthy: curator, approval and limits

A self-writing skill library grows. The curator docs name the risk: "dozens of narrow near-duplicates that pollute the catalog and waste tokens." Hermes has three controls.

**The curator** is a background pass that runs when at least 7 days have passed since the last run and the agent has been idle for 2 hours (Curator docs, October 2026). By default it only prunes: an agent-created skill unused for 14 days becomes stale, and one unused for 30 days moves to `~/.hermes/skills/.archive/`. The v0.22.0 release notes (14 October 2026) say unused skills now archive at 30 days, down from 90. An LLM pass that merges overlapping skills into umbrella skills exists but is off by default (`curator.consolidate: false`); the docs say a full sweep "typically takes 50–100 API calls." The curator never deletes outright, keeps a per-change ledger, and `hermes curator rollback` undoes a run or a single edit.

**Write approval.** With `skills.write_approval: true`, every skill create, patch or delete is staged under `~/.hermes/pending/skills/` and reviewed with `/skills pending`, `/skills diff <id>` and `/skills approve <id>`. It's off by default. The docs suggest it for "small models that misjudge what they learned." A separate content scanner, `skills.guard_agent_created`, is also off by default.

**Pinning.** `hermes curator pin <skill>` stops the curator and the agent from deleting a skill; patches still go through.

The limits worth knowing:

- **Context cost.** The skill list is about 3k tokens every session, and a loaded `SKILL.md` stays in context for the rest of the session. The linter flags bodies past about 24,000 characters for that reason.
- **Quality drift.** The agent edits skills, including ones you wrote or installed. A bad lesson from one session can land in the skill that governs every later run of that task. Turn on write approval if that matters.
- **Review cost.** The background review replays the conversation. Unless you set `max_input_tokens`, its input budget is 75% of the review model's context window, capped at 600,000 tokens (memory docs, October 2026). You can route it to a cheaper model or disable it.
- **The curator ignores most skills.** It only manages skills marked agent-created, which today means skills the background review made. Skills you or the foreground agent created (including `/learn`) are left alone unless you run `hermes curator adopt`.
- **Docs disagree on one number.** The curator page gives 14 and 30 days in its config, and "30d unused → stale → 90d unused → archived" further down. The config defaults and the v0.22.0 notes both say 30 days to archive.

Because skills are instructions the agent follows, a poisoned skill is an attack path. Hermes's install scan and project-trust step exist for that reason; the general risk is covered in [AI memory injection](/articles/ai-memory-injection/).

## Skills and the other memory layers

Hermes splits memory by job. Skills are one layer of four:

| Layer | Holds | Loaded | Written by |
|---|---|---|---|
| `MEMORY.md` / `USER.md` | Short facts about the environment and you (2,200 and 1,375 characters) | Every session, frozen snapshot | Agent, via the `memory` tool |
| Session search | Full past sessions in SQLite | When the agent calls `session_search` | Automatic |
| **Skills** | Procedures: steps, commands, pitfalls | List at start, full file on demand | Agent, background review, `/learn`, you, the hub |
| External provider | Long-term memory from one plugin (Honcho, Mem0, Hindsight and others) | Prefetched before each turn | The provider |

The same background review writes to both memory and skills, which the docs call the "self-improvement loop": "memory stores small durable facts that should always be in context, while skills store longer procedures that should load only when relevant." Whether an external memory provider ever sees skill content is not documented; assume it doesn't.

Other systems learn procedures differently. LangMem's prompt optimizer rewrites the system prompt itself, to "update system prompts based on conversation logs or agent trajectories" ([LangMem docs](https://langchain-ai.github.io/langmem/guides/optimize_memory_prompt/)). That changes every call, where a Hermes skill only loads when relevant. Claude Code and other Agent Skills clients read the same `SKILL.md` format but leave writing skills to you; see [Claude Code memory](/articles/claude-code-memory/).

**[Hindsight](https://github.com/vectorize-io/hindsight)** sits in the external-provider layer, not the skills layer. It installs with `hermes plugins install hindsight` and doesn't write `SKILL.md` files. What it adds next to skills is learned knowledge that nobody has to write as a procedure: it merges related facts into **observations** (evidence-backed beliefs with quotes and a proof count), keeps **mental models** (standing answers it refreshes in the background, read without an LLM call), and answers reasoning questions with `reflect`. That fits lessons that don't belong in one skill, such as how a user wants pull requests reviewed across every project. It's MIT-licensed with every feature in the self-hosted server. The limit: each `retain` calls an LLM, and recall returns more context per turn than Mem0 does by default (about 4x, by Mem0's own measurement), and Hermes prefetches it every turn, so set a low `max_tokens`. More in [Hindsight explained](/articles/ai-memory-hindsight/), and the provider comparison is on [OpenClaw vs Hermes Agent memory](/articles/openclaw-vs-hermes-agent-memory/).


## FAQ

### How does Hermes Agent create skills?

Hermes saves skills with its skill_manage tool. Its system prompt asks it to record non-trivial workflows, a nudge reminds it every 15 tool-calling iterations by default, and a background review after a turn can create or patch skills. You can also ask for one with /learn. New skills are SKILL.md files under ~/.hermes/skills/.

### Where are Hermes Agent skills stored?

In ~/.hermes/skills/, one folder per skill with a required SKILL.md plus optional references/, templates/, scripts/, examples/ and assets/. Hermes also reads project skills from .hermes/skills/ or .agents/skills/ in a trusted git repo, and any folders listed under skills.external_dirs in config.yaml.

### What is the difference between Hermes skills and Hermes memory?

Memory (MEMORY.md and USER.md) holds a few short facts that load into every session. Skills hold longer procedures, how to do a task, and only the skill list loads at start; the full SKILL.md loads when a task needs it. The Hermes docs call skills the agent's procedural memory.

