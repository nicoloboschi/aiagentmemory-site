# AI Dungeon Memory System: Memory Bank and Story Summary

> How AI Dungeon's Memory System works and why it forgets: memories every six actions, Auto Summarization, Memory Bank embedding search, slots per tier, and how context is built.

Source: https://aiagentmemory.org/articles/ai-dungeon-memory-system/ · Updated 2026-10-23 · aiagentmemory.org


The **AI Dungeon memory system** is the feature that keeps long adventures coherent after the story outgrows the AI's context window. It has two parts: **Auto Summarization**, which maintains a running **Story Summary**, and the **Memory Bank**, which stores short AI-written summaries of past actions and pulls the most relevant ones back into context using embeddings.

Both are built by Latitude, AI Dungeon's developer, and adapted from its other game, Voyage. Everything below comes from the [AI Dungeon help center](https://help.aidungeon.com/), checked on 23 October 2026.

## What is the AI Dungeon memory system?

**AI Dungeon's Memory System automatically compresses, stores and retrieves story details so the AI can follow long adventures. A summarization model turns every six actions into a Memory. Auto Summarization keeps a high-level Story Summary of the plot, and the Memory Bank retrieves specific Memories by embedding similarity when they're relevant to your latest action.**

The problem it solves is plain context length. AI Dungeon builds each prompt from your story text plus AI Instructions, Plot Essentials, Author's Note and triggered Story Cards. Once the adventure is longer than the model's window, the oldest text gets cut, and the AI seems to forget. Latitude's [Memory System article](https://help.aidungeon.com/faq/the-memory-system) frames it as two brain-like strategies: **compression** (summaries) and **retrieval** (bringing back details when they matter).

## How AI Dungeon creates memories

A **Memory** is an AI-generated summary of a small set of story actions. It keeps plot facts and drops the descriptive prose, so it's denser than the original text.

The schedule is fixed:

1. Nothing is summarized for the first 12 actions.
2. At action 12, actions 1-6 become your first Memory. The six most recent actions stay as raw text.
3. At action 18, the next six actions become a second Memory.
4. This repeats every six actions, forever.

Because the latest six actions are never summarized yet, you can edit or undo them without affecting memories.

## Auto Summarization and the Story Summary

**Auto Summarization** writes and updates the **Story Summary**, a Plot Component that holds an overview of the whole plot. It re-summarizes every **15 actions**, and Memories fill the gap in between, so every part of the story is covered.

- When the summary gets too long, it's compressed automatically.
- Turning it on in an existing adventure makes it summarize from the beginning, updating each action or retry until it catches up.
- You can edit the summary by hand. Auto updates overwrite it, but your edits are sent to the summarization model, so they should carry forward.
- If you change something early in the story, update the summary manually. Latitude says re-summarizing whole histories on every edit would exceed the summarizer's own context and cost too much.

The Story Summary can also replace raw text it has already covered, so your context viewer may show less story text than expected. That's intended.

## How the Memory Bank retrieves memories

The **Memory Bank** works like automatic Story Cards. Each Memory is stored with an **embedding** vector. When the full adventure no longer fits in context, AI Dungeon embeds your **most recent action**, scores every stored Memory by similarity, and inserts the top ones that fit. Those are "Used Memories." Return to the town of Castlebrook, and memories about Castlebrook come back.

When the bank is full, the **least-used memories are removed** ("Forgotten Memories"). Old memories that keep getting used can stay indefinitely. You can browse them with **Explore Memories** in the Context Viewer, in Timeline or Relevance view.

| Membership tier | Price (USD/month) | Memory Bank slots | Context length |
|---|---|---|---|
| Wanderer | Free | 25 memories | Up to 4K tokens |
| Journey | $14.99 | 100 memories | Up to 8K |
| Legend | $29.99 | 200 memories | Up to 16K |
| Mythic | $49.99 | 400 memories | Up to 32K |
| Ultimate | $99.99 | 800 memories | Up to 32K |

Source: the [Memberships & Benefits page](https://help.aidungeon.com/memberships-benefits). Subscribers can also spend 1 credit per action for extra context on some models. All players get Auto Summarization; the Memory Bank size is what scales with tier.

**To turn it on**, open the game settings sidebar and go to **Gameplay > AI Models > Memory System**, then toggle Memory Bank and Auto Summarization. The setting carries to every adventure until you turn it off. It takes a few turns for the Story Summary to appear and longer for memories to be used.

## Why AI Dungeon forgets things (and what to check)

Most "the AI forgot" cases trace back to one of these documented behaviors:

| What you see | Why, per the help center | What to do |
|---|---|---|
| No memories show up in a new adventure | The Memory Bank stays empty until the story is 12 actions deep, and no memories are *used* until the full adventure no longer fits in context | Keep playing; check Explore Memories in the Context Viewer |
| An old event is gone | The bank was full and the least-used memories were removed ("Forgotten Memories") | Put must-keep facts in Plot Essentials or a Story Card; a higher tier holds more memories (25 free, up to 800) |
| A relevant memory exists but isn't used | Retrieval matches your **latest action** only, and the Memory Bank gets about 25% of the leftover context, so only the top matches that fit go in | Name the person or place in your action so the search finds it |
| The summary contradicts an edit you made early on | Story Summary isn't re-written when you change old actions | Edit the Story Summary by hand |
| Less story text in context than expected | The Story Summary replaces raw text it already covers | Intended; view it in the Context Viewer |
| Details drop fast on the free tier | Free context is up to 4K tokens, so little raw history fits | Keep Plot Essentials and Author's Note short; spend a credit per action for extra context on supported models (subscribers) |

You can't edit or delete single Memories, per the [Memory System article](https://help.aidungeon.com/faq/the-memory-system); only the Story Summary is editable.

## What goes into AI Dungeon's context

AI Dungeon splits context into **required** and **dynamic** parts, per its [context article](https://help.aidungeon.com/faq/what-goes-into-the-context-sent-to-the-ai).

**Required elements:** AI Instructions, Plot Essentials, Story Summary, Author's Note, Front Memory (for scripts) and the last action. If they add up to more than **70%** of the context, the last action and Front Memory stay whole, then Author's Note, Plot Essentials, AI Instructions and Story Summary are added in that priority order, trimmed to fit.

**Dynamic elements** share what's left:

| Dynamic element | Share of remaining tokens |
|---|---|
| Story Cards (triggered by keywords) | About 25% |
| History (recent actions) | About 50%, or 75% if Memory Bank is off |
| Memory Bank | About 25% |

Story Cards are ranked by how recently and often their triggers appeared. The system checks at least the last 4 actions for triggers; with more than 500 tokens available for cards, it checks tokens / 100 actions (900 tokens means 9 actions).

The assembled order sent to the model is: Instructions (system prompt), Plot Essentials, Story Cards, Story Summary, Memory Bank, History, Author's Note, Last Action, Front Memory.

## Plot Components vs the Memory System

The Memory System doesn't replace manual tools. They do different jobs:

| Component | Use | In context |
|---|---|---|
| **Plot Essentials** (formerly "Memory") | Facts the AI must always know | Always |
| **Story Summary** | Overall plot; updated by Auto Summarization | Always |
| **Author's Note** | Short style and direction guidance, placed just before your last action | Always |
| **Story Cards** | World details tied to trigger words | When triggered |
| **Memory Bank** | Specific past events, retrieved by relevance | When context overflows |

AI Dungeon's [context management guide](https://help.aidungeon.com/how-do-i-manage-context) says most of your context should be story text. Keep Plot Essentials minimal, keep Author's Note short, and avoid broad Story Card triggers: a trigger like "Em" also fires on "Empire," "Them" and "Stem," because triggers match inside words and ignore case.

## How it compares with other AI memory designs

AI Dungeon's design is a small, well-documented version of what agent memory systems do: periodic **summarization** for the big picture plus **vector retrieval** for details, with eviction of least-used items. The [AI agent memory guide](/articles/ai-agent-memory-explained/) covers the same ideas for agents, and [memory consolidation in AI agents](/articles/memory-consolidation-ai-agents/) goes deeper on summarize-and-merge approaches.

Other roleplay apps are simpler. [Janitor AI](/articles/janitor-llm-memory/) relies on a Chat Memory note you maintain, and [Character.AI](/articles/what-is-character-ai-memory-limit/) uses pins and a short memory field. All of them are bounded by the model's [context window](/articles/context-window-of-an-llm/).

If you're building your own story or roleplay app, open-source memory servers do the same jobs with fewer fixed limits. [Hindsight](https://github.com/vectorize-io/hindsight), for example, extracts facts and timelines from each scene, searches them four ways (semantic, keyword, graph and time), and keeps consolidated character facts current, at the cost of an LLM call per write.


## FAQ

### How does AI Dungeon's memory system work?

Every six actions, AI Dungeon summarizes six older actions into a short Memory. Auto Summarization keeps a running Story Summary updated every 15 actions, and the Memory Bank stores Memories with embeddings, then retrieves the ones most relevant to your latest action once the story no longer fits in context. Both are toggled under Gameplay > AI Models > Memory System.

### How many memories can the AI Dungeon Memory Bank store?

Per Latitude's membership page (October 2026): 25 memories on the free Wanderer tier, 100 on Journey, 200 on Legend, 400 on Mythic and 800 on Ultimate. When the bank is full, the least-used memories are removed to make room for new ones.

### Why does AI Dungeon forget things?

Usually for one of four reasons: the story still fits in context so no memories are used yet, the Memory Bank is full and dropped its least-used memories, retrieval only matches your latest action, or the small free-tier context (up to 4K tokens) leaves little room for history. Put facts the AI must always know in Plot Essentials or Story Cards.

### What context length does AI Dungeon give each tier?

Free gets up to 4K tokens of context, Journey up to 8K, Legend up to 16K, and Mythic and Ultimate up to 32K, according to AI Dungeon's help center. Subscribers can also spend credits per action for more context on some models.

