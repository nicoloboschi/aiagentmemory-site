# GitHub Copilot Memory: What It Stores and Where

> How GitHub Copilot Memory works after its October 2026 GA: repository facts, user preferences, voting and the 90-day expiry, the eight surfaces including VS Code, Copilot dreaming, bulk import, and how to manage or replace it.

Source: https://aiagentmemory.org/articles/github-copilot-memory/ · Updated 2026-11-09 · aiagentmemory.org


**Copilot Memory** is GitHub's automatic memory for Copilot's agentic features. It saves two things as Copilot works: **repository facts** (conventions, architecture decisions, build commands) and **user preferences**. Each fact stores citations to the code that supports it, Copilot revalidates those citations before using the fact, votes on whether it still holds, and anything that goes 90 days without an upvote expires.

It became **generally available on paid Copilot plans on 28 October 2026**, per GitHub's [changelog](https://github.blog/changelog/2026-10-28-github-intelligence-puts-your-code-context-and-history-to-work/), which also added VS Code, the Copilot app, GitHub Mobile, Slack and Teams as surfaces and introduced [Copilot dreaming](#copilot-dreaming-memory-curation-in-the-background) in public preview. Every fact below comes from GitHub's [Copilot Memory docs](https://docs.github.com/en/copilot/concepts/prompting-and-context/copilot-memory), its how-to pages and that changelog, read on 9 November 2026. Two numbers changed since this page was first written in October: the expiry went from 28 days to 90, and the surface list went from four to eight. For how other agents handle the same problem, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

## What is Copilot Memory?

**Copilot Memory is a managed store of short, specific notes that Copilot writes about your repositories and about how you work. Copilot captures them while it does normal work, attaches citations to supporting code or to your own words, checks those citations against the current branch before reusing a note, upvotes or downvotes it on the result, and expires anything that goes 90 days without an upvote. You don't edit the notes by hand, though you can bulk-import prepared ones from the CLI.**

That design answers the usual complaint about agent memory: notes that were true once and quietly rot. GitHub's answer is to make every repository fact point at the code that justifies it, then recheck it. The docs are careful about the limit: the citation check is "a judgment Copilot makes, rather than a guarantee enforced by the system", so a rule you need followed every time belongs in custom instructions, not in memory.

## The two kinds of memory

| | Repository facts | User preferences |
|---|---|---|
| Example | "Build runs through `make ci`", a layering rule, an architecture decision | How you like Copilot to work with you, stated or implied |
| Who can use it | Everyone with read access to the repository who can use Copilot Memory, in that repository only | Only you, across all your repositories (and anyone who views a shared agent session where it appears) |
| Who can create it | Only users with **write access** to the repository | You |
| Who can delete it | Users with write access and user preferences enabled on their account | You; on Business and Enterprise, also an org or enterprise admin |
| Citations | Links to supporting code, revalidated against the current branch | May include direct quotes of what you said |
| Used in code review and autofix? | Yes | **No** |
| Expiry | 90 days without an upvote; admins can set 7 to 180 days | 90 days without an upvote, not configurable |
| Ownership on Business/Enterprise | The repository | Your billing entity (the org or enterprise that licenses you) |

Three details matter in practice. Repository facts can also come out of **pull requests closed without merging**, so a rejected approach can still teach Copilot something, which is why validation against the live codebase exists. Facts are **not pooled across an organization's repositories**, and forking a repository does not copy its memories to the fork. And if you hold Copilot licences from several organizations, you must pick a **default billing entity** in account settings before any memory is created at all; preferences stay owned by the entity that was active when they were made, and Copilot stops retrieving them if you move to a different licence.

Copilot also refuses to store some things. Per the docs, memories are scanned for secrets (tokens, keys, credentials) and rejected if one is found, memories containing recognized model-control tokens can be rejected as a prompt-injection defence, and Copilot is instructed not to store sensitive personal information or short-lived task instructions. How that compares with other systems is on [memory poisoning in AI agents](/articles/ai-memory-injection/).

## Where Copilot Memory works, and where it doesn't

As of the 28 October 2026 GA, GitHub's docs list eight surfaces: the **GitHub Copilot app**, **Copilot CLI**, **Copilot code review**, **Copilot cloud agent** (the agent that works on an issue and opens a pull request, including sessions started from Slack or Teams), **agentic autofix**, **Copilot in GitHub Mobile**, **Copilot in Slack** and **Copilot in Microsoft Teams**. Visual Studio Code is covered through its own local setting. Until October the list was four items long and the IDE was not on it.

A memory captured by one surface can be used by another: something the cloud agent learns about database connections can inform code review later. Scope narrows per surface. All surfaces read repository facts; code review and agentic autofix **never apply user preferences**, so a personal preference won't change how your pull requests are reviewed. Some surfaces add a local switch that can only restrict, never override, your GitHub settings and policies:

| Surface | Local default | How to change it |
|---|---|---|
| Copilot CLI, interactive | On, follows your GitHub settings | `/memory off`, `/memory on`, or `"memory": false` in CLI user settings |
| Copilot CLI, programmatic (`copilot -p`) | **Off** | `--enable-memory` on each run |
| Visual Studio Code (Local Agent Host sessions) | **Off** | `chat.copilot.memory.enabled` in VS Code settings |
| GitHub Copilot app | Follows your GitHub settings, where enabled for your account | No in-app toggle |
| Cloud agent, code review, autofix | Follow your GitHub and repository settings | Personal, repository and org settings |

Two things about the VS Code entry. The setting applies to Copilot sessions that run in VS Code's Local Agent Host, and it is off by default even when your GitHub settings allow memory, so a developer who never touches it still gets no Copilot Memory in the editor. And it is separate from VS Code's own memory tool for local memory files, which keeps working either way. The Copilot app's project **Memory** setting is also a different feature: it stores per-project preferences across tasks, not the shared facts described here.

In the Copilot app and in interactive CLI sessions, Copilot can ask you to approve storing a memory. The cloud agent and code review don't ask. In the CLI only the root agent can call `store_memory` or `vote_memory`; subagents can read memories but not change them. Applications on the Copilot SDK can turn memory on when they create or resume a session, but the SDK has no API to list, edit or delete memories.

## Instruction files: the part that works in your editor

Copilot in VS Code reads persistent context from files in the repository and in your home directory. From the [VS Code custom instructions docs](https://code.visualstudio.com/docs/copilot/customization/custom-instructions):

- `AGENTS.md` at the repository root, enabled by `chat.useAgentsMdFile`. Nested `AGENTS.md` files are experimental and off by default (`chat.useNestedAgentsMdFiles`).
- `.github/copilot-instructions.md`, read by the Local harness when `github.copilot.chat.codeGeneration.useInstructionFiles` is on.
- `*.instructions.md` files under `.github/instructions/`, searched recursively, each able to declare an `applyTo` glob or a `description` so it loads only when relevant.
- `CLAUDE.md` at the root or in `.claude/`, plus `.claude/rules`, enabled by `chat.useClaudeMdFile`. `CLAUDE.local.md` stays local.
- Personal files: `~/.copilot/copilot-instructions.md` and `~/.copilot/instructions`.
- Organization-wide instructions from GitHub org settings, discovered when `github.copilot.chat.organizationInstructions.enabled` is set.

The docs are explicit that these sources are **additive** and that you should not rely on file order or precedence, because it differs by harness. If two files disagree, fix the files. This is procedural memory you own and review in a pull request, which is the opposite trade-off from the automatic store. Those two models are compared in [agent memory vs documentation](/articles/agent-memory-vs-documentation/). GitHub's own guidance, in the Copilot Memory docs, is to put rules that must always hold (a security requirement, a mandatory review step) in custom instructions and let memory carry "the accumulated detail that would otherwise make an instructions file long". Since October, [dreaming](#copilot-dreaming-memory-curation-in-the-background) can also propose edits to these files as pull requests.

## How memories are reinforced and expire

**Every memory carries an internal vote count and an expiry date. Copilot upvotes a memory it can validate and downvotes one it finds wrong or outdated; upvoted memories are more likely to be included in a prompt, and an upvote resets the memory's 90-day expiry clock.** Retrieval alone resets nothing. Votes are not displayed, but the settings pages list memories in vote order, so a memory sliding down the list is being downvoted.

| Rule | Repository facts | User preferences |
|---|---|---|
| Default expiry | 90 days from creation or last upvote | 90 days from creation or last upvote |
| Configurable | Yes: 7 to 180 days, set by an enterprise or organization owner | No, always the default |
| Org and enterprise both set a value | The shorter one applies | n/a |
| Changing the period | Doesn't re-date stored facts; each adopts the new period at its next upvote | n/a |
| Downvoted past a threshold | Hidden from the list, no longer used, deleted after a period of disuse | Same |
| "Forget this" in a prompt | Downvotes, does not delete; delete it in settings | Same |

Per the docs read on 9 November 2026, the default expiry period for Copilot Memory is 90 days; the public-preview docs and the March 2026 changelog said 28 days. For a team, the practical reading is that a build command Copilot keeps confirming lives for as long as the project does, while a decision nobody has acted on in three months quietly disappears, which is also how [memory decay](/articles/ai-memory-decay/) is meant to work. If your compliance rules need facts gone sooner, an organization owner can set repository retention down to 7 days under Copilot settings > Memory retention; that lever does not exist for personal preferences.

## How to turn it on, off and clean it up

1. **Check your own switch.** Profile picture > **Copilot settings** > Features > **Copilot Memory**, then Enabled or Disabled. It controls user preferences only. It is on by default on Pro, Pro+ and Max.
2. **Business and Enterprise plans need a policy first.** An admin enables user preferences; individual users can then opt out. Repository facts are a separate control: an organization's **Repository access** setting defaults to **All repositories**, so facts can be on while preferences are off, and vice versa. With licences from several orgs, the most restrictive policy wins.
3. **Review your preferences** at [github.com/settings/copilot/memory](https://github.com/settings/copilot/memory) and delete any you don't want. Memories Copilot has learned not to use are hidden from the list but may still be stored until they expire.
4. **Review repository facts** under Repository Settings > Copilot > Memory. Anyone with write access and preferences enabled on their account can read and delete them.
5. **Switch it off per repository.** A repository admin can disable repository facts in the repository's Copilot settings. After that, facts are neither saved nor read; existing facts are kept rather than deleted, and user preferences are untouched.
6. **In Copilot CLI**, use `/memory show`, `/memory on` and `/memory off`. The opt-out is saved per machine and survives changes to your GitHub setting. Non-interactive runs (`copilot -p "..."`) have memory off unless you pass `--enable-memory`. Turning memory off mid-conversation doesn't remove memories already in that conversation's context; start a new session.
7. **In VS Code**, enable `chat.copilot.memory.enabled`. It is off by default.
8. **Import memories in bulk** with `copilot memories import FILE --dry-run`, then without the flag. The file is JSONL, one object per line with `type`, `version`, `scope` (user, or a repository by host/owner/name), `subject`, `fact`, `reason` and `citations`. Conflicts fail per record unless you pass `--on-conflict skip`; rerunning a whole file after a partial failure can create duplicates. The import counts as approval, so no per-memory prompt, but secret scanning and permission checks still run.
9. **Admins on Business and Enterprise** can export preferences as JSONL or delete them, in bulk or per user. Exports are reusable for two hours and capped at 10 new exports per hour per enterprise.

Since the [May 2026 changelog](https://github.blog/changelog/2026-05-26-copilot-memory-has-more-controls-for-deletion-scope-and-the-copilot-cli/), the `store_memory` permission prompt also says which kind of entry it is about to write: a user preference visible only to you, or a repository fact visible to every contributor. Read that prompt before approving, because the second one is a shared write. Memories that end up in a model's context count toward token usage and so toward AI credits.

## Copilot dreaming: memory curation in the background

**Copilot dreaming is a scheduled background job, in public preview since 28 October 2026, that reads your synced Copilot sessions and pull request review feedback and then curates memory: it adds, updates and deletes memories without asking, and proposes changes to checked-in instruction and skill files as pull requests.** GitHub's framing is that it "updates context, not the underlying AI model". You don't need a Copilot conversation open for it to run, and you can't start a run or edit its schedule, prompt or model.

| Dream type | What it reads | What it changes |
|---|---|---|
| Personal memory curation | Your eligible synced sessions and personal memories | Your personal memories; no instruction or skill files |
| Repository customization | Eligible repository sessions, code, memories and guidance | Repository memories directly; `.github/copilot-instructions.md`, `.github/instructions/**`, `.github/agents/**`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `REVIEW.md` and `.github/skills/**` via pull requests |
| Code review customization | Feedback on merged pull requests and current code | Internal guidance for future Copilot code reviews, no pull request |

Four constraints from the [dreaming docs](https://docs.github.com/en/copilot/concepts/prompting-and-context/copilot-dreaming):

- **Opt-in at two levels.** Dreaming is off by default. Individuals enable personal dreaming under Copilot settings (hidden unless Copilot Memory is on) and repository dreaming per personally owned repository. In organizations an admin sets a policy (**Let organizations decide**, **Enabled everywhere**, **Disabled everywhere**), and members must still enable personal dreaming themselves. Removing a repository from Copilot Memory's repository selection turns off both memory and dreaming for it.
- **Model and plan.** The initial preview requires GPT-6.1 Sol to be allowed by policy, is not available on Copilot Pro, and does not support GitHub Enterprise Cloud with data residency. Code review customization doesn't yet support self-hosted runners.
- **Session syncing.** Personal and repository dreams need your CLI and Copilot app sessions synced to GitHub, which is on by default but gated by the org policy "Store local sessions in the Cloud" (at least **View from cloud**) and by `remoteExport` in `~/.copilot/settings.json`.
- **Cost.** Every run bills AI credits, including runs that change nothing and runs over inactive repositories. GitHub tells admins to set spending budgets before widening coverage, and notes there is no separate budget for dreaming itself.

Turning dreaming off stops future runs but does not undo its edits, close its pull requests or restore memories it deleted. Run results are visible only in the Copilot app's **Dreaming** tab, and that view is "not a complete activity history". Compared with the schemes on [memory consolidation for AI agents](/articles/memory-consolidation-ai-agents/), dreaming is unusual in that its output includes edits to the human-owned instruction files, not only to the machine-written store; Anthropic's take on the same idea is on [Claude dreaming](/articles/claude-dreaming-agent-memory/).

## Copilot Memory next to other coding agents

| Agent | Automatic memory | Default | Scope | Expiry |
|---|---|---|---|---|
| **GitHub Copilot** | Copilot Memory: repository facts + user preferences, citation-checked and voted | On for Pro, Pro+ and Max; admin policy for orgs; VS Code off until enabled | Server-side, shared per repository | 90 days without an upvote (7 to 180 for repository facts) |
| **Claude Code** | Auto memory in `~/.claude/projects/<project>/memory/` | On for local sessions | Machine-local | None; you prune it |
| **Codex** | Memories in `~/.codex/memories/` | Off | Machine-local | Unused 30 days: dropped from consolidation |
| **Cursor** | Removed in 2.1.x, export to Rules | n/a | n/a | n/a |

Copilot is the only one of the four whose automatic memory is **shared across a team by default**, **validated against the code** and, with dreaming on, curated by a scheduled job. Its limits are different ones: the editor needs an opt-in setting, user preferences follow your billing entity rather than you, and the notes are short facts, not a searchable history of how a feature was built. Details per tool are in [Claude Code memory](/articles/claude-code-memory/) and [Codex memory](/articles/codex-memory/).

## When to add an external memory server

Copilot Memory covers the "what is true about this repo right now" layer. Teams usually reach for something else for two other jobs: **long history** (why a decision was made eight months ago) and **memory shared across tools**, when some developers use Copilot, others Claude Code or Codex.

[Hindsight](https://github.com/vectorize-io/hindsight) is one option, and it fits this reader for a specific reason: it ships a native Copilot plugin alongside Claude Code, Codex and Cursor plugins plus an MCP endpoint per memory bank, so one store serves every agent on the team. Three things separate it here:

- **It consolidates instead of appending.** Related lessons merge into **observations**: beliefs that keep their supporting quotes and a proof count, refined as new sessions add evidence. `reflect` reasons over many memories to answer a question like "why did we drop the Redis cache?" Mental models are standing answers kept current in the background, so reading one is a database read. That background upkeep is the closest thing to Copilot dreaming on the self-hosted side, with the difference that Hindsight never edits your instruction files and bills no credits for a run that changes nothing.
- **Long histories are its strongest result.** On the BEAM benchmark it scores **73.9 at 1M tokens and 64.1 at 10M**, against 64.1 and 48.6 for Mem0's Platform. Those figures come from Mem0's own [comparison page](https://mem0.ai/compare/mem0-vs-hindsight), so they are not Hindsight's marketing, but both are vendor-run, not an independent audit.
- **It can run on your Copilot subscription.** The MIT-licensed server includes every feature, including graph and temporal recall, and works with 25+ LLM providers, local Ollama included, or an existing Copilot, Claude or ChatGPT subscription instead of an API key.

Two honest limits before you add it. It returns more context per retrieval than Mem0 by default, roughly 4x by Mem0's measurement, so set a lower `max_tokens` if session prompts matter to you. And it publishes **no compliance certifications**, which is a real gap next to a GitHub-managed feature if your code is covered by an audit. More in [Hindsight AI memory](/articles/ai-memory-hindsight/).


## FAQ

### Does GitHub Copilot remember previous sessions?

Yes, on paid plans. Copilot Memory, generally available since 28 October 2026, saves repository facts and personal preferences automatically and reuses them in the Copilot app, Copilot CLI, VS Code, Copilot cloud agent, code review, agentic autofix, GitHub Mobile, Slack and Teams. In VS Code it is off until you turn on the chat.copilot.memory.enabled setting; until then your persistent context there comes from instruction files like AGENTS.md.

### How long does Copilot Memory keep a memory?

90 days by default, counted from when the memory was created or last upvoted by Copilot. Merely retrieving a memory does not reset the clock. Enterprise and organization owners can set the retention of repository facts anywhere from 7 to 180 days; user preferences always use the default. Downvoted memories stop being used and are deleted after a period of disuse.

### Where do I delete Copilot memories?

Your own preferences are at github.com/settings/copilot/memory. Repository facts are under Repository Settings > Copilot > Memory, where a user with write access can delete them and an admin can switch Memory off for the repository. In Copilot CLI, /memory show, /memory on and /memory off control it per machine; non-interactive runs need --enable-memory. Asking Copilot to forget something only downvotes the memory, it does not delete it.

### What is Copilot dreaming?

A public-preview background job, announced 28 October 2026, that reviews your synced Copilot sessions and pull request feedback, then adds, updates and deletes memories on its own and proposes changes to checked-in instruction and skill files as pull requests. It is off by default, needs GPT-6.1 Sol allowed by policy, is not available on Copilot Pro, and every run spends AI credits, even runs that change nothing.

