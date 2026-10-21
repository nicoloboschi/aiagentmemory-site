# GitHub Copilot Memory: What It Stores and Where

> How GitHub Copilot Memory works: repository facts, user preferences, citation checking, the 28-day expiry, which Copilot surfaces use it, and how to manage or replace it.

Source: https://aiagentmemory.org/articles/github-copilot-memory/ · Updated 2026-10-17 · aiagentmemory.org


**Copilot Memory** is GitHub's automatic memory for Copilot's agentic features. It saves two things as Copilot works: **repository facts** (conventions, architecture decisions, build commands) and **user preferences**. Each fact stores citations to the code that supports it, Copilot revalidates those citations before using the fact, and anything unused for 28 days is deleted.

It is in **public preview** and available on paid Copilot plans. Every fact below comes from GitHub's [Copilot Memory docs](https://docs.github.com/en/copilot/concepts/agents/copilot-memory) and the [public preview changelog](https://github.blog/changelog/2026-01-15-agentic-memory-for-github-copilot-is-in-public-preview/), checked on 17 October 2026. For how other agents handle the same problem, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

## What is Copilot Memory?

**Copilot Memory is a managed store of short, specific notes that Copilot writes about your repositories and about how you work. Copilot captures them while it does normal work, attaches citations to supporting code or to your own words, checks those citations against the current branch before reusing a note, and expires anything that stops being used. You never write the notes by hand.**

That design answers the usual complaint about agent memory: notes that were true once and quietly rot. GitHub's answer is to make every repository fact point at the code that justifies it, then recheck it.

## The two kinds of memory

| | Repository facts | User preferences |
|---|---|---|
| Example | "Build runs through `make ci`", a layering rule, an architecture decision | How you like Copilot to work with you, stated or implied |
| Who can use it | Everyone with Copilot Memory access to that repository, in that repository only | Only you, across all your repositories |
| Who can create it | Only users with **write access** to the repository | You |
| Citations | Links to supporting code, revalidated against the current branch | May include direct quotes of what you said |
| Used in code review? | Yes | **No** |
| Ownership on Business/Enterprise | The repository | Your billing entity (the org or enterprise that licenses you) |

Two details matter in practice. Repository facts can also come out of **pull requests closed without merging**, so a rejected approach can still teach Copilot something, which is why validation against the live codebase exists. And if you hold Copilot licences from several organizations, you must pick a **default billing entity** in account settings before any preferences are generated at all.

## Where Copilot Memory works, and where it doesn't

GitHub's docs list four surfaces:

- **Copilot coding agent** (the cloud agent that works on an issue and opens a pull request)
- **Copilot code review**
- **Copilot CLI**
- **Agentic autofix**

A memory captured by one can be used by another: something the coding agent learns can inform code review later. Scope narrows per surface. Copilot CLI uses repository facts plus the initiating user's preferences. Code review uses **repository facts only**, never personal preferences.

The IDE is not in that list. GitHub's Copilot Memory pages do not mention VS Code or any other editor, and the "manage for yourself" page says nothing about IDE behaviour. So if you ask "does Copilot remember what I told it in VS Code yesterday," the honest answer today is that persistence in the editor comes from instruction files, not from Copilot Memory.

## Instruction files: the part that works in your editor

Copilot in VS Code reads persistent context from files in the repository and in your home directory. From the [VS Code custom instructions docs](https://code.visualstudio.com/docs/copilot/customization/custom-instructions):

- `AGENTS.md` at the repository root, enabled by `chat.useAgentsMdFile`. Nested `AGENTS.md` files are experimental and off by default (`chat.useNestedAgentsMdFiles`).
- `.github/copilot-instructions.md`, read by the Local harness when `github.copilot.chat.codeGeneration.useInstructionFiles` is on.
- `*.instructions.md` files under `.github/instructions/`, searched recursively, each able to declare an `applyTo` glob or a `description` so it loads only when relevant.
- `CLAUDE.md` at the root or in `.claude/`, plus `.claude/rules`, enabled by `chat.useClaudeMdFile`. `CLAUDE.local.md` stays local.
- Personal files: `~/.copilot/copilot-instructions.md` and `~/.copilot/instructions`.
- Organization-wide instructions from GitHub org settings, discovered when `github.copilot.chat.organizationInstructions.enabled` is set.

The docs are explicit that these sources are **additive** and that you should not rely on file order or precedence, because it differs by harness. If two files disagree, fix the files. This is procedural memory you own and review in a pull request, which is the opposite trade-off from the automatic store. Those two models are compared in [agent memory vs documentation](/articles/agent-memory-vs-documentation/).

## How to turn it on, off and clean it up

1. **Check your own switch.** Profile picture > **Copilot settings** > Features > **Copilot Memory**, then Enabled or Disabled. It is on by default on paid individual plans, including Pro and Pro+ since the [March 2026 changelog](https://github.blog/changelog/2026-03-04-copilot-memory-now-on-by-default-for-pro-and-pro-users-in-public-preview/).
2. **Enterprise and organization plans need a policy first.** An admin enables it; individual users can then opt out.
3. **Review your preferences** at [github.com/settings/copilot/memory](https://github.com/settings/copilot/memory) and delete any you don't want.
4. **Review repository facts** under Repository Settings > Copilot > Memory. Repository owners can read and delete them.
5. **Switch it off per repository.** A repository admin can disable Memory in the repository's Copilot feature controls. After that, repository facts are neither saved nor read, existing facts are kept rather than deleted, and user preferences are untouched.
6. **In Copilot CLI**, use `/memory show`, `/memory on` and `/memory off`. The choice carries across sessions.
7. **Admins on Business and Enterprise** can export or delete preferences in bulk or per user.

Since the [May 2026 changelog](https://github.blog/changelog/2026-05-26-copilot-memory-has-more-controls-for-deletion-scope-and-the-copilot-cli/), the `store_memory` permission prompt also says which kind of entry it is about to write: a user preference visible only to you, or a repository fact visible to every contributor. Read that prompt before approving, because the second one is a shared write.

## Copilot Memory next to other coding agents

| Agent | Automatic memory | Default | Scope | Expiry |
|---|---|---|---|---|
| **GitHub Copilot** | Copilot Memory: repository facts + user preferences, citation-checked | On for paid individual plans; admin policy for orgs | Server-side, shared per repository | Unused entries deleted after 28 days |
| **Claude Code** | Auto memory in `~/.claude/projects/<project>/memory/` | On for local sessions | Machine-local | None; you prune it |
| **Codex** | Memories in `~/.codex/memories/` | Off | Machine-local | None |
| **Cursor** | Removed in 2.1.x, export to Rules | n/a | n/a | n/a |

Copilot is the only one of the four whose automatic memory is **shared across a team by default** and **validated against the code**. It is also the most limited in reach: it does not follow you into your editor, it keeps nothing older than 28 unused days, and the notes are short facts, not a searchable history of how a feature was built. Details per tool are in [Claude Code memory](/articles/claude-code-memory/) and [Codex memory](/articles/codex-memory/).

## When to add an external memory server

Copilot Memory covers the "what is true about this repo right now" layer. Teams usually reach for something else for two other jobs: **long history** (why a decision was made eight months ago) and **memory shared across tools**, when some developers use Copilot, others Claude Code or Codex.

[Hindsight](https://github.com/vectorize-io/hindsight) is one option, and it fits this reader for a specific reason: it ships a native Copilot plugin alongside Claude Code, Codex and Cursor plugins plus an MCP endpoint per memory bank, so one store serves every agent on the team. Three things separate it here:

- **It consolidates instead of appending.** Related lessons merge into **observations**: beliefs that keep their supporting quotes and a proof count, refined as new sessions add evidence. `reflect` reasons over many memories to answer a question like "why did we drop the Redis cache?" Mental models are standing answers kept current in the background, so reading one is a database read.
- **Long histories are its strongest result.** On the BEAM benchmark it scores **73.9 at 1M tokens and 64.1 at 10M**, against 64.1 and 48.6 for Mem0's Platform. Those figures come from Mem0's own [comparison page](https://mem0.ai/compare/mem0-vs-hindsight), so they are not Hindsight's marketing, but both are vendor-run, not an independent audit.
- **It can run on your Copilot subscription.** The MIT-licensed server includes every feature, including graph and temporal recall, and works with 25+ LLM providers, local Ollama included, or an existing Copilot, Claude or ChatGPT subscription instead of an API key.

Two honest limits before you add it. It returns more context per retrieval than Mem0 by default, roughly 4x by Mem0's measurement, so set a lower `max_tokens` if session prompts matter to you. And it publishes **no compliance certifications**, which is a real gap next to a GitHub-managed feature if your code is covered by an audit. More in [Hindsight AI memory](/articles/ai-memory-hindsight/).


## FAQ

### Does GitHub Copilot remember previous sessions?

Partly. Copilot Memory, in public preview, saves repository facts and personal preferences automatically and reuses them in Copilot coding agent, code review, Copilot CLI and agentic autofix. GitHub's docs do not list the IDE among the surfaces that use it, so inside VS Code your persistent context still comes from instruction files like AGENTS.md.

### How long does Copilot Memory keep a memory?

Any fact or preference that goes unused for 28 days is deleted automatically. Successful validation or use resets that 28-day timer, so entries that keep proving useful stay and the rest expire.

### Where do I delete Copilot memories?

Your own preferences are at github.com/settings/copilot/memory. Repository facts are under Repository Settings > Copilot > Memory, where a repository admin can also delete them or switch Memory off for the repository. In Copilot CLI, /memory show, /memory on and /memory off control it per session and the choice persists.

