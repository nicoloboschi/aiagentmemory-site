# Claude Dreaming: Managed Agents Memory Stores and Dreams

> What dreaming means in Claude's agent memory: Managed Agents memory stores, the Dreams API, Anthropic's CL-Bench, Pokémon and wiki results, vs Letta and Hindsight.

Source: https://aiagentmemory.org/articles/claude-dreaming-agent-memory/ · Updated 2026-11-07 · aiagentmemory.org


**Claude dreaming** is an offline memory pass: Claude rereads past session transcripts and its own memory notes, corrects notes that turned out wrong, merges duplicates, and builds new structures such as a wiki or an index that later sessions can navigate. Anthropic described it in [The components of agent memory](https://www.anthropic.com/research/components-of-agent-memory) (November 6, 2026) and ships it in Claude Managed Agents as **Dreams**, a research-preview API.

This page explains what Anthropic means by agent memory, how memory stores and Dreams work, what the published tests showed (CL-Bench, Pokémon Red, a 420M-token wiki), and how the approach compares with Letta's dreaming and Hindsight's background consolidation. Every number comes from Anthropic's post, its docs or its cookbook, and none of it was reproduced here.

## What is Claude dreaming?

**Dreaming, in Anthropic's usage, is background work that an agent does between sessions to organize its memory: it tests notes against the original transcripts, fixes mistakes, merges and prunes, and writes representations (wiki pages, indexes, promoted top-level notes) that make memory easier to use next time.** It is separate from writing memory during a session, which happens in the moment and sees only that session's context.

Anthropic's post credits the idea to **sleep-time compute**, proposed by researchers at Letta and UC Berkeley ([Lin et al., 2025](https://arxiv.org/abs/2504.13171)): use idle time between interactions to reorganize context. Letta Code calls its version dreaming too, and Google's SLEEP framework uses the same word. The general technique is covered in [memory consolidation in AI agents](/articles/memory-consolidation-ai-agents/).

The post also defines what counts as agent memory: knowledge Claude gains from experience and retains between sessions, saved as context it can read later. It excludes weight changes, in-context learning within one conversation, and **compaction**, the summaries that replace older turns so a session can keep going.

## Memory stores in Claude Managed Agents

Dreams operate on **memory stores**, the persistence layer of [Claude Managed Agents](https://platform.claude.com/docs/en/managed-agents/memory). The docs describe them as follows (read November 7, 2026):

| Property | Managed Agents memory store |
|---|---|
| What it is | A workspace-scoped collection of text documents, each addressed by a path |
| How the agent sees it | Mounted as a directory under `/mnt/memory/<store-slug>/` in the session sandbox; read and written with ordinary file tools |
| Stores per session | Up to 8, attached only when the session is created |
| Access | `read_write` (default) or `read_only`, enforced at the filesystem level |
| Per-store guidance | An optional `instructions` string shown to the agent, capped at 4,096 characters |
| Size limit | 10,000 memories per store; new writes fail past that, existing ones stay readable |
| History | Every change creates an immutable memory version; versions are kept 30 days, with the recent versions of a live memory always kept; versions can be redacted |
| Status | Beta on every model (header `agent-memory-2026-07-22`) |

Two related features sit outside Managed Agents. The client-side [memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) (`memory_20250818`, Claude 4 and later) lets Claude request file operations under `/memories` that your own code executes, so the data stays in your storage. And [Claude Code's auto memory](/articles/claude-code-memory/) keeps a `MEMORY.md` index of which only the first 200 lines or 25 KB load at session start, the same "small top layer" idea the dreaming post argues for.

## How Dreams work

A **dream** is an asynchronous job, per the [Dreams docs](https://platform.claude.com/docs/en/managed-agents/dreams) (read November 7, 2026):

1. **Input:** one existing memory store, plus 1 to 100 past sessions whose transcripts Claude mines for patterns and insights. An optional `instructions` field steers the synthesis (focus areas, what to preserve, output conventions). It is not a line editor: "change sentence X to Y" produces no change.
2. **Run:** the dream moves from `pending` to `running`. Its `session_id` points at the session doing the work, so you can stream its events and watch what it reads and writes. Runs take "minutes to a few hours", scaling with the number of input transcripts.
3. **Output:** a new, separate memory store with duplicates merged, stale or contradicted entries replaced with the latest value, and new insights surfaced. The input store is never modified.
4. **Decide:** attach the output store to future sessions, in place of or alongside the original, or delete it. On `failed` or `canceled`, the partial output store is kept for inspection.

Dreams are billed at standard API token rates for the model you pick, with usage reported on the resource. They are in research preview on select models and need the `dreaming-2026-04-21` beta header; the general Managed Agents header does not grant access.

Anthropic's [cookbook](https://platform.claude.com/cookbook/managed-agents-cma-dreaming-with-a-dynamic-workflow) builds the same idea by hand as a workflow (Sonnet 5.5 leads, Haiku 5.5 analyzes each transcript). Its cost figures are the most concrete published so far: dreaming over fifty long transcripts, about 12 MB of text, cost about $8 at list price and took about ten minutes; eleven example transcripts (283 KB) cost $0.36 and took about two and a half minutes.

## What Anthropic's tests showed

The research post reports six experiments. The table collects the numbers; the sections below give the setup for each.

| Test | Setup | Result (Anthropic, November 2026) |
|---|---|---|
| Learning loop, CL-Bench 1.0 | Fresh session per question, shared memory directory; after each answer the model is told the correct one | Sonnet 4.6 left scattered untested guesses; Opus 5 was the first to write validated rules with their evidence and trap warnings; Opus 5.5 and Sonnet 5.5 did the same |
| Salience, Pokémon Red (~10,000 steps) | Memory folder the model reads and writes while playing | Sonnet 3.7: 413 files in 225 folders, 1 badge. Fable 5: nine files in two folders plus `global.md`, 4 badges. Opus 5.5 beat the game with a small folder |
| Steering rules, CL-Bench | One fixed rule per run vs plain "a memory folder is available" | "Record only failures" cost Opus 5 and Opus 4.7 much of memory's boost; telling Sonnet 4.6 to read memory and test its theories nearly doubled its score |
| Dream correction, Pokémon Red | Dream run on a folder holding a wrong PokéCenter location | The dream caught and rewrote the note; the uncorrected run fell off the same ledge three times and never reached Cerulean City |
| Wiki, internal corpus | ~350,000 files, ~420M tokens; 25 questions; 2 to 128 dreaming sessions | Wiki alone: 13% to 48% of graded criteria. Search plus wiki: 54% to 61%, vs 53% for search alone, saving ~20,000 tokens per question |
| Wiki, Harvey LAB merger questions | 24 questions, more than half needing several documents | 87% with the wiki vs 73% with search alone on the cross-document items; on one other LAB task the wiki grew larger than the documents and gave no gain |

### Memory as a prediction, and the learning loop

Writing a rule down is a bet that the pattern will matter again. On a [Continual Learning Bench 1.0](https://continual-learning-bench.com/news/cl-bench-1-0/) task, Claude answered database questions one per session, learned the correct answer after each, and could write to a persisted memory directory. Claude Sonnet 4.6 produced files full of untested guesses. Claude Opus 4.7 consolidated them into one reference document and started marking what was checked. Claude Opus 5 closed the loop: most entries held a rule, how it knew the rule was true, earlier answers, and warnings about traps. The newer Opus 5.5 and Sonnet 5.5 did the same, scoring about as Opus 5 did with memory from a slightly higher baseline.

CL-Bench itself measures whether a system improves from prior experience: tasks are sequential, the system is expected to change during evaluation, and a "gain" metric compares each system with a stateless copy of itself to separate learning from raw capability.

### Judging what is worth saving

At about 10,000 steps into Pokémon Red, Claude Sonnet 3.7 had written 413 memory files across 225 folders, often one per battle, item or character, and had earned one badge. Around that point it spent 17 steps creating six files and deleting nine without touching the game. Claude Fable 5 had nine files in two folders plus a `global.md` dashboard holding the current goal, a gym plan and its own rules for using memory, and had four badges. The difference: Sonnet 3.7 stored what the screen already showed every turn (money, position); Fable 5 stored only what the game would not tell it, such as a tested rule for catching a Magikarp after seven failed throws.

### Tell Claude what memory is for, not how to use it

Anthropic now gives frontier models simple guidance: write a note when asked or when something is clearly worth remembering. Fixed rules hurt stronger models; "record only failures" removed much of the gain that memory gave Opus 5 and Opus 4.7. Steering helped older models most: prompting Sonnet 4.6 to read its memory files before answering and to test its theories nearly doubled its score. Sonnet 5 read its memory before answering 9 out of 10 times without being told, Sonnet 4.6 less than half the time. This is the same lesson [procedural memory in AI agents](/articles/ai-agent-procedural-memory/) draws from prompt and skill learning: the agent's own validated rules beat rules imposed in advance.

### Dreams correct, build, promote and fill gaps

Four jobs a dream can do, each with an example in the post:

- **Correct.** A note placed the Mt. Moon PokéCenter on the wrong side, so Claude kept walking off a one-way ledge. A dream checked the note against the transcript, rewrote it, and the next run reached Cerulean City.
- **Build representations.** The wiki pattern, popularized by [Andrej Karpathy's LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) and covered in [LLM memory with Obsidian](/articles/llm-memory-obsidian/), is pages Claude writes and links so it can navigate what it knows. Over 128 dreaming sessions on the 420M-token corpus, wiki-only accuracy rose from 13% to 48% of graded criteria, and search plus wiki reached 61%.
- **Promote.** Dreaming can float a few notes to the top of the hierarchy for preemptive loading at session start, and keep that layer small by pushing detail down and leaving links. The post cites Claude Code's warning that longer instruction files reduce adherence.
- **Fill gaps.** A second dreaming pass over one LAB wiki found nine links to a page that did not exist, repointed them to the right page, and added three pages including an index.

A goal-focused dream matters as much as a generic one. On a LAB acquisition task, a first wiki whose agents shared one contract-terms table captured 41 of 57 graded facts (72%); after Claude rewrote the wiki-building instructions so each contract got its own page, quoted verbatim by a single agent, the brief met 100% of the criteria.

### Where it did not help

Anthropic reports the failures too. On a LAB task whose criteria could be answered from single documents by text search, the wiki showed no clear gain. On another, the wiki grew larger than the documents themselves, a map as big as the territory. And the post lists open questions: when to write memory (in-session writing distracts from the task, dreaming delays availability), how to keep notes current without re-checking everything, and whether transcript search should be the foundation with representations layered on top.

## Claude Dreams vs Letta dreaming vs Hindsight consolidation

| | Claude Managed Agents Dreams | Letta Code dreaming | Hindsight consolidation |
|---|---|---|---|
| What runs | An async job you start over a store plus 1 to 100 sessions | Background subagents that review recent conversations and update memory | Background job as facts are retained; merges related facts into observations |
| Trigger | On demand, via the API | After a set number of agent steps, or when context is compacted | New facts; timing not documented beyond "in the background" |
| Writes to | A new output store; the input is never modified | The agent's memory in place, with an optional review pass before updates apply | New observations with quotes and a proof count; raw facts kept and still searchable |
| Standing answers | Promoted notes in the store's top layer | Memory blocks the agent edits | Mental models, refreshed in the background; reading one is a database read with no LLM call |
| Open source | No; research preview on select models, standard token billing | Letta Code is Apache 2.0; you pay your model provider | Yes, MIT, in the self-hosted server |
| Main limit | Output can lag; cost scales with transcripts; no reproduction of Anthropic's results yet | Memory can lag the conversation; extra tokens for the review pass | Every retain and consolidation calls an LLM, so writes cost more |

Letta details are in the [Letta guide](/articles/letta-ai-guide/). Hindsight's column comes from its README and release notes; a November 3, 2026 change (v1.2.0) stops consolidation from letting an older fact overwrite a newer same-day amendment, the same "latest value wins" rule Dreams apply to contradicted entries.

## Where Hindsight fits

[Hindsight](https://github.com/vectorize-io/hindsight) is the closest open-source match to what Dreams do, for teams that run their own agent loop instead of Managed Agents. It consolidates in the background as memories arrive, turning related facts into **observations** that keep their supporting quotes and a proof count and are refined rather than overwritten. **Mental models** are standing answers to questions you define, kept current in the background, which is the "promoted top layer" idea without a separate job. Its `reflect` operation reasons over memory to answer questions that need thinking, not lookup. All of this ships in the MIT server, with Claude Code, Codex and Cursor plugins among 60+ integrations.

The honest trade-offs: Hindsight pays an LLM call on every retain and again on consolidation, so writes cost more than a plain file store; it returns more context per retrieval than Mem0 by default (set `max_tokens` lower); and like Dreams, when consolidation runs is not spelled out. Its benchmark numbers (LongMemEval-S 94.6, BEAM 10M 64.1, vendor-run) measure recall over long histories, not the wiki-building task Anthropic tested, so don't compare the two. More in [Hindsight AI memory](/articles/ai-memory-hindsight/).

## How to apply this in your own agent

1. **Keep transcripts.** Dreams need the original sessions as ground truth; a note can only be corrected against what actually happened.
2. **Let the model judge salience.** Give strong models the purpose of memory, not a rulebook. Reserve explicit "read your memory first" prompts for weaker models.
3. **Dream into a copy.** Write the reorganized memory to a new store or branch, review it, then switch. Never let the pass overwrite its own input.
4. **Keep the injected layer small.** Whatever loads at session start should be an index with links, not the archive.
5. **Give the dream a goal.** State what matters (contract terms, user preferences, commands that worked) and where exact quotes beat summaries.
6. **Measure against search alone.** Anthropic's wiki won on cross-document questions and lost on single-document lookups. Test both before paying for the representation.
7. **Budget it.** Cost scales with transcript count; start with a small batch, as the docs advise, and check the usage figures before scaling to nightly runs.

For the wider picture of memory types and write paths, see [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What is dreaming in Claude's agent memory?

Dreaming is an offline pass in which Claude rereads past session transcripts and its memory notes, then fixes wrong notes, merges duplicates, and builds new structures such as a wiki or an index. Anthropic describes it in its November 6, 2026 research post 'The components of agent memory'. In Claude Managed Agents it ships as Dreams, a research-preview API that reads a memory store plus 1 to 100 sessions and writes a reorganized copy to a new store.

### How do Dreams work in Claude Managed Agents?

You create a dream with an input memory store and up to 100 past sessions, optionally with instructions. It runs as an asynchronous job, from minutes to a few hours, and produces a separate output store with duplicates merged and stale entries replaced. The input store is never modified, so you can review the result and discard it. Dreams are billed at standard token rates and need the dreaming-2026-04-21 beta header.

### Does dreaming actually improve agent memory?

In Anthropic's own tests, yes, with limits. A wiki built by dreaming over 128 sessions on a 420M-token corpus took Claude from 13% to 48% of graded criteria with the wiki alone, and search plus wiki reached 61% against 53% for search alone. A dream also caught a wrong map note that had made Claude fall off the same ledge three times. But on one legal-brief task the wiki grew larger than the source documents and gave no gain over search.

