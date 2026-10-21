# Persistent Memory in AI: How It Works and How to Add It

> What persistent memory in AI is, how apps and agents keep it across sessions and restarts, which tools offer it (AnythingLLM, ChatGPT, Mem0), and what breaks.

Source: https://aiagentmemory.org/articles/persistent-memory-ai/ · Updated 2026-10-12 · aiagentmemory.org


**Persistent memory in AI is information that survives the end of a session and a restart of the app, so an assistant or agent can use it next time.** The model doesn't provide it; every LLM call is stateless. Persistence comes from a durable store (a database or files) that the app writes to, scopes per user, and reads back into the prompt.

## What is persistent memory in AI?

**Persistent memory is any memory an AI system keeps on durable storage across sessions, restarts and deployments, and retrieves into the model's context when it's relevant.** It's the opposite of the context window and of in-RAM chat buffers, which vanish when the conversation or process ends.

Three tests separate persistent memory from the short-term kind:

1. **New session:** open a new chat. Does it still know the fact?
2. **Restart:** kill the process or redeploy. Is the fact still there?
3. **Another device or instance:** does a second server or client see it?

A chat history list in RAM passes none. A Postgres table passes all three. Many "memory" demos fail test two because they use in-memory stores. LangGraph's [persistence docs](https://docs.langchain.com/oss/python/langgraph/persistence) warn about exactly this for its in-memory checkpointer: "When the process restarts, all checkpoints are lost."

Persistent memory overlaps with **long-term memory**, and the terms are often used as synonyms. Long-term describes what's kept (facts, events, skills); persistent describes how it's kept (durably). For the types and retrieval side, see [AI agent long-term memory](/articles/ai-agent-long-term-memory/) and the [AI agent memory guide](/articles/ai-agent-memory-explained/).

## Can an LLM have permanent memory?

People searching for "permanent memory LLM" usually want the model itself to remember. There's only one place that could happen: the **weights**. They change through training or fine-tuning, never through chatting.

Fine-tuning on your facts sounds like permanent memory, but it works poorly for that job. [Ovadia et al. (2023)](https://arxiv.org/abs/2312.05934) compared fine-tuning with retrieval for adding knowledge and found RAG "consistently outperforms it, both for existing knowledge encountered during training and entirely new knowledge." Fine-tuned facts are also hard to update or delete: you can't remove one user's data from weights without retraining.

So in practice, permanent memory for an LLM means an external store kept as long as you need it. How the model side works (weights, context, KV cache) is covered in [how LLM memory works](/articles/how-llm-memory-works/).

## The parts of a persistence layer

Whatever tool you pick, persistent memory needs the same five parts. Skipping any one causes a known failure.

| Part | Question it answers | What goes wrong without it |
|---|---|---|
| Durable store | Where does it live? | Lost on restart |
| Scope key | Whose memory is this? | One user's facts leak into another's chat |
| Update rule | What happens when a fact changes? | "Lives in Paris" and "lives in Berlin" both come back |
| Retrieval | What goes into the prompt? | Prompt fills with irrelevant memories |
| Deletion | How do we forget? | Can't honor "delete my data" |

### Minimal example in Python

This uses only the standard library. SQLite writes to a file, so the memory survives restarts. The primary key makes each fact one row per user, so an update replaces the old value instead of adding a conflicting one.

```python
import sqlite3, time

db = sqlite3.connect("memory.db")  # a file on disk, so it survives restarts
db.execute("""CREATE TABLE IF NOT EXISTS memories (
    user_id TEXT, key TEXT, value TEXT, updated_at REAL,
    PRIMARY KEY (user_id, key))""")

def remember(user_id: str, key: str, value: str) -> None:
    # Upsert: a new value for the same key replaces the old one
    db.execute("INSERT INTO memories VALUES (?, ?, ?, ?) "
               "ON CONFLICT(user_id, key) DO UPDATE SET value=excluded.value, "
               "updated_at=excluded.updated_at", (user_id, key, value, time.time()))
    db.commit()  # durable once committed

def recall(user_id: str) -> str:
    rows = db.execute("SELECT key, value FROM memories WHERE user_id = ?", (user_id,))
    return "\n".join(f"- {k}: {v}" for k, v in rows)

def forget_user(user_id: str) -> None:
    db.execute("DELETE FROM memories WHERE user_id = ?", (user_id,))
    db.commit()

remember("u42", "city", "Paris")
remember("u42", "city", "Berlin")   # update, not a second fact
remember("u42", "diet", "vegetarian")
print(recall("u42"))                # goes into the system prompt
```

Output: `- city: Berlin` and `- diet: vegetarian`. You'd put that string in the system prompt before each model call.

This covers a handful of structured facts. It doesn't cover search by meaning, extracting facts from free text, or thousands of memories per user. That's what memory frameworks add. A full walkthrough is in [how to give AI agents memory](/articles/how-to-give-ai-agents-memory/).

## Persistent memory in AnythingLLM

AnythingLLM, the open-source desktop and Docker app for chatting with local or hosted models, has a built-in persistent memory feature called **Memories & Personalization**. Per its [documentation](https://docs.anythingllm.com/features/memories):

- **Off by default.** An admin turns on Enable Personalization from the chat settings menu.
- **Two scopes.** Workspace memories (up to 20 per workspace) and global memories (up to 5, shared across workspaces). In multi-user mode, each user's memories are private.
- **Manual or automatic.** You can add and edit memories in the sidebar. Automatic extraction runs on a schedule (every 3 hours by default) with an "Observer/Reflector pipeline": the Observer proposes up to 3 facts per run, and the Reflector removes duplicates, revises existing memories and drops low-confidence ones.
- **Injection.** Memories go into the system prompt under `## Things I Remember About You`: all global memories plus the 5 most relevant workspace memories.

Two cautions from the same page: don't store passwords or API keys as memories, and "Memory content is sent to your LLM provider as part of the system prompt." If privacy is the reason you run AnythingLLM, pair it with a local model.

Before this feature, "AnythingLLM persistent memory" meant its document workspaces: files you embed stay searchable across chats. That's persistent retrieval over documents, not memory about you.

## Which AI has persistent memory?

Consumer apps and developer tools both offer it, in different forms.

**Consumer chat apps.** ChatGPT, Claude, Gemini and Copilot all keep saved facts or search past chats across sessions, with different controls and plan limits. The sourced comparison is in [best chatbot for memory](/articles/best-chatbot-for-memory/). For one-off chats that shouldn't persist, ChatGPT has Temporary Chat and Claude has incognito chats.

**Developer tools.** If you're building your own app or agent, these add a persistence layer:

| Tool | How it persists | License |
|---|---|---|
| [LangGraph](https://docs.langchain.com/oss/python/langgraph/persistence) stores and checkpointers | Postgres or other backends; in-memory versions are lost on restart | MIT |
| [Mem0](https://github.com/mem0ai/mem0) | Extracted facts in a vector store, per user, agent or session | Apache-2.0 |
| [Letta](https://github.com/letta-ai/letta-code) | Git-backed memory files (MemFS) and memory blocks, managed by the agent runtime | Apache-2.0 |
| [Graphiti](https://github.com/getzep/graphiti) (Zep) | Temporal knowledge graph that marks old facts as superseded | Apache-2.0 |
| [Hindsight](https://github.com/vectorize-io/hindsight) | Facts, consolidated observations and mental models on Postgres with pgvector (embedded option), per memory bank; every feature in the open-source server | MIT |

Which one fits depends on how long memory has to last and how much it has to hold. LangGraph stores suit agents already built on LangGraph. Mem0 is the lightest layer and returns the fewest tokens per search. Graphiti fits when facts change and you need to know what was true when. Letta suits agents that manage their own memory inside its runtime.

**Hindsight** is the one built for memory that piles up over months. On the long-history BEAM test it scores 73.9 at 1M tokens and 64.1 at 10M, against Mem0's 64.1 and 48.6, by Mem0's own comparison page. Its update rule is consolidation: related facts merge into evidence-backed observations that get refined, not overwritten. The trade-off is cost: every write calls an LLM, and recall returns more context than Mem0 unless you lower `max_tokens`. See [Hindsight explained](/articles/ai-memory-hindsight/).


## Shared and selective persistent memory

Not everything should persist. A July 2026 paper from Apple researchers, [Shared Selective Persistent Memory for Agentic LLM Systems](https://arxiv.org/abs/2607.09493) (Pedada, Dhavala and Patil), tests this directly. Their system keeps four kinds of reusable context, task specifications, data schemas, tool configurations and output constraints, and throws away session-specific reasoning traces.

In their controlled test on four public datasets, the authors report:

- **Selective memory:** 12 of 12 trials completed, about 3.9K input tokens.
- **No memory:** 0 of 12, about 3.8K tokens.
- **Full conversation history:** 8 of 12, about 7.7K tokens.

Their summary: "What is kept matters more than how much is kept." Old tool-use traces bias the agent toward paths it already tried. The "shared" part means memory is packaged into workspaces that can be passed between users under role-based access control. These are the authors' own results on small samples; in a 24-task enterprise test, they note the differences weren't statistically significant.

The practical lesson holds even without the paper: persist decisions, preferences and settings, not raw transcripts.

## What goes wrong with persistent memory

Persistence turns short-lived mistakes into lasting ones.

- **Stale facts.** Without an update rule, old and new facts both come back. Test a changed fact on any system you adopt.
- **Bad extraction.** An automatic extractor can save something the user never meant. Keep a link from each memory to its source so you can audit it.
- **Cross-user leaks.** Scope every read and write by user or tenant ID, and test it.
- **Memory poisoning.** A malicious message can plant an instruction that fires in later sessions. The [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) lists memory and context poisoning as ASI06.
- **Deletion gaps.** Claude's help pages note that deleting a conversation doesn't remove memories made from it. Your own system should make clear which delete removes what.


## FAQ

### What is persistent memory in AI?

Persistent memory is information an AI app or agent keeps after a session ends and after the process restarts, then uses in later sessions. The model itself doesn't keep it. The app stores it in a database or files, scoped to a user or agent, and adds the relevant parts to the prompt.

### Does AnythingLLM have persistent memory?

Yes. AnythingLLM's Memories & Personalization feature stores short facts per user: up to 20 per workspace and 5 global. It's off by default. Once enabled, it can extract facts from chats on a schedule and adds them to the system prompt under 'Things I Remember About You.'

### Can an LLM have permanent memory?

Only through its weights, which change by training or fine-tuning, not by chatting. Fine-tuning is a poor way to add new facts; a 2023 study found retrieval consistently beat it. In practice, permanent memory means an external store the app keeps for as long as it needs.

