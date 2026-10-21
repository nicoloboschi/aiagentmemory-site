# How to Add Memory to a Chatbot (with Python)

> How to add memory to a chatbot: chat history buffers, trimming, rolling summaries and long-term facts, with tested Python code for SQLite and LangChain.

Source: https://aiagentmemory.org/articles/how-to-add-memory-to-chatbot/ · Updated 2026-10-12 · aiagentmemory.org


To **add memory to a chatbot**, store each conversation's messages and resend them with every request, then trim or summarize old turns so the history fits the model's window. For memory across conversations, also save key facts about the user to a database and add the relevant ones to the prompt when a new chat starts.

## What is chatbot memory?

**Chatbot memory is everything an app sends to the model so it can respond as if it remembers: the recent message history of the current chat (short-term memory) and stored facts from past chats (long-term memory).** The model itself keeps nothing. Every API request starts blank unless your code puts the past back in.

That's why a raw API call "forgets" your name one message later. Chat apps hide this by resending the history. You have three places to keep it:

- **In your app.** You store messages and send them each time. Full control, works with any model.
- **With the provider.** OpenAI's Responses API can chain turns with `previous_response_id` or keep a durable Conversation object. The [OpenAI conversation state guide](https://developers.openai.com/api/docs/guides/conversation-state) notes that "all previous input tokens for responses in the chain are billed as input tokens," so this saves code, not money.
- **In a framework.** LangGraph checkpointers save the message state per thread.

Long-term memory is a different job. It needs a store keyed by user, not by conversation. The broader picture is in [AI agent memory explained](/articles/ai-agent-memory-explained/).

## Chatbot memory strategies compared

Resending the full history works for short chats. Long chats need a strategy, because cost grows with every turn and the history eventually overflows the [context window](/articles/context-window-of-an-llm/).

| Strategy | What gets sent | Pros | Cons |
|---|---|---|---|
| Full buffer | Every message so far | Nothing is lost | Cost grows each turn; overflows the window |
| Window buffer | Last N messages | Fixed cost, simple | Forgets anything older than N |
| Token buffer | As many recent messages as fit a budget | Uses the window well | Same hard cutoff, just measured in tokens |
| Rolling summary | Summary of old turns + recent messages | Keeps key facts from long chats | Summary can drop details; one extra LLM call when compacting |
| Long-term facts | Retrieved user facts + current chat | Remembers across sessions | Needs extraction, updates and deletion |

Longer prompts aren't free in quality either. The [Lost in the Middle paper](https://arxiv.org/abs/2307.03172) (Liu et al., 2023) found that model performance "significantly degrades when models must access relevant information in the middle of long contexts." A short summary plus recent turns often beats pasting the whole transcript.

Most production chatbots combine the last three: a token budget for recent turns, a summary of older ones, and a small block of long-term facts.

## How to add memory to a chatbot in 6 steps

1. **Give every conversation an ID.** A `session_id` or `thread_id`. Memory without an ID mixes up users.
2. **Persist messages.** Write each user and assistant message to a database as it happens. In-process lists vanish on restart.
3. **Load history per request.** Read the session's messages and send them after the system prompt.
4. **Cap the history.** Keep recent turns within a fixed count or token budget.
5. **Summarize what you drop.** Fold old turns into a running summary and put it in the system prompt.
6. **Add long-term facts if you need them.** Save stable facts per user and retrieve them at the start of each chat.

## Example: persistent chat memory with SQLite and a rolling summary

This is the whole pattern in plain Python with the OpenAI SDK and SQLite: history survives restarts, the last six messages go verbatim, and older turns get folded into a summary.

Tested with `openai` 3.26:

```python
import sqlite3
from openai import OpenAI

client = OpenAI()
MODEL = "gpt-5-mini"
KEEP_TURNS = 6  # recent messages sent verbatim

db = sqlite3.connect("chat.db")
db.executescript("""
CREATE TABLE IF NOT EXISTS messages (session_id TEXT, role TEXT, content TEXT, id INTEGER PRIMARY KEY);
CREATE TABLE IF NOT EXISTS summaries (session_id TEXT PRIMARY KEY, summary TEXT);
""")

def load(session_id: str) -> tuple[str, list[dict]]:
    row = db.execute("SELECT summary FROM summaries WHERE session_id = ?", (session_id,)).fetchone()
    rows = db.execute("SELECT role, content FROM messages WHERE session_id = ? ORDER BY id", (session_id,))
    return (row[0] if row else ""), [{"role": r, "content": c} for r, c in rows]

def save(session_id: str, role: str, content: str) -> None:
    db.execute("INSERT INTO messages (session_id, role, content) VALUES (?, ?, ?)", (session_id, role, content))
    db.commit()

def compact(session_id: str, summary: str, history: list[dict]) -> None:
    """Fold everything except the last KEEP_TURNS messages into a running summary."""
    old = history[:-KEEP_TURNS]
    if not old:
        return
    transcript = "\n".join(f"{m['role']}: {m['content']}" for m in old)
    resp = client.chat.completions.create(model=MODEL, messages=[{
        "role": "user",
        "content": f"Update this summary of a chat. Keep names, facts, decisions and open questions.\n\n"
                   f"Current summary:\n{summary or '(none)'}\n\nNew messages:\n{transcript}",
    }])
    db.execute("INSERT OR REPLACE INTO summaries VALUES (?, ?)", (session_id, resp.choices[0].message.content))
    db.execute("DELETE FROM messages WHERE session_id = ? AND id NOT IN "
               "(SELECT id FROM messages WHERE session_id = ? ORDER BY id DESC LIMIT ?)",
               (session_id, session_id, KEEP_TURNS))
    db.commit()

def chat(session_id: str, user_message: str) -> str:
    save(session_id, "user", user_message)
    summary, history = load(session_id)
    system = "You are a helpful assistant."
    if summary:
        system += f"\n\nSummary of the earlier conversation:\n{summary}"
    resp = client.chat.completions.create(model=MODEL, messages=[{"role": "system", "content": system}, *history])
    answer = resp.choices[0].message.content
    save(session_id, "assistant", answer)
    if len(history) + 1 > 2 * KEEP_TURNS:
        compact(session_id, summary, history + [{"role": "assistant", "content": answer}])
    return answer
```

In our test, we told the bot "I'm Ana, planning a 5-day trip to Japan in April," then sent six more messages. The first turns were compacted into the summary, and a later "What's my name and where am I going?" was still answered correctly: "Your name is Ana and you're going to Japan."

Two choices to tune: `KEEP_TURNS` (higher means more exact recall, more tokens) and the summary prompt (tell it what your domain must never lose, such as order numbers or dates).

## Example: the same with LangChain and LangGraph

If you use LangChain, the old memory classes are gone from the main package. `ConversationBufferMemory` now lives in `langchain-classic` and warns that it "was deprecated in LangChain 0.3.1 and will be removed in 2.0.0. Use `langchain.agents.create_agent` instead." The replacement is an agent with a **checkpointer** (saves each thread's messages) and **SummarizationMiddleware** (compacts long threads).

Tested with `langchain` 1.4 and `langgraph-checkpoint-sqlite`:

```python
import sqlite3
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware
from langgraph.checkpoint.sqlite import SqliteSaver

checkpointer = SqliteSaver(sqlite3.connect("checkpoints.db", check_same_thread=False))

agent = create_agent(
    model="openai:gpt-5-mini",
    tools=[],
    checkpointer=checkpointer,
    middleware=[SummarizationMiddleware(model="openai:gpt-5-mini", trigger=("tokens", 4000), keep=("messages", 20))],
)

config = {"configurable": {"thread_id": "ana-session-1"}}
agent.invoke({"messages": [{"role": "user", "content": "Hi, I'm Ana."}]}, config)
result = agent.invoke({"messages": [{"role": "user", "content": "What's my name?"}]}, config)
print(result["messages"][-1].content)  # "Your name is Ana. ..."
```

The [LangChain short-term memory docs](https://docs.langchain.com/oss/python/langchain/short-term-memory) recommend a database-backed checkpointer such as `PostgresSaver` in production. A fuller walkthrough is in [building a chatbot with memory in LangGraph](/articles/chatbot-with-memory-langgraph/).

## Adding long-term memory across conversations

History and summaries are per conversation. When the user opens a new chat, they're gone. To remember a user across chats, add a second store keyed by user ID:

- **After each conversation (or turn),** extract stable facts: name, preferences, ongoing projects. An LLM call with "list lasting facts about the user" works; memory libraries like Mem0 do this for you.
- **At the start of each chat,** retrieve the facts that match the first message and add them to the system prompt.
- **When facts change,** update or replace them so "lives in NYC" doesn't survive the move to SF.

Tested code for both the do-it-yourself and library versions is in [how to give AI agents memory](/articles/how-to-give-ai-agents-memory/). If you'd rather not build it, memory libraries handle extraction and updates for you:

- **Mem0:** the lightest option. One LLM pass per write, under 7K tokens per retrieval by its own measurement.
- **Zep:** a managed service that tracks when facts change, so "lives in NYC" gets a validity window instead of lingering.
- **LangMem:** memory tools inside LangGraph, if your chatbot already runs there.
- **[Hindsight](https://github.com/vectorize-io/hindsight):** an MIT-licensed server built for long histories. It scores 73.9 on BEAM at 1M tokens and 64.1 at 10M, against 64.1 and 48.6 for Mem0 (both vendor-reported, from [Mem0's comparison page](https://mem0.ai/compare/mem0-vs-hindsight)). It merges related facts into observations that get refined as new ones arrive. The catch: writes and consolidation call an LLM, and it returns more context per retrieval than Mem0, so set a token budget. For a simple FAQ bot it's more than you need.

## How to clean up chatbot memory

Memory you can't delete becomes a privacy problem and a quality problem. Plan cleanup in three layers:

- **Session history.** Delete the session's rows (`DELETE FROM messages WHERE session_id = ?`) and its summary. With LangGraph checkpointers, delete the thread.
- **Long-term facts.** Delete by user ID. Mem0 has `delete_all(user_id=...)`; most memory APIs have an equivalent.
- **Bad memories.** When a stored fact is wrong, fix or delete that one item rather than letting it be retrieved forever. Keep the source message ID with each fact so you can trace where it came from.

If you're a user rather than a builder: in ChatGPT you can view and delete saved memories, or turn memory off. Steps are in [how to find ChatGPT memory](/articles/how-to-find-chatgpt-memory/).


## FAQ

### How does chatbot memory work?

The model behind a chatbot is stateless, so the app resends earlier messages with every request. That resent history is the chatbot's short-term memory. Long-term memory is separate: the app saves facts to a database and adds the relevant ones to the prompt in later conversations.

### What is an LLM memory buffer?

A memory buffer is the list of recent messages an app keeps and sends with each request. A plain buffer keeps everything; a window buffer keeps the last N messages; a token buffer keeps as many recent messages as fit a token budget. LangChain's old ConversationBufferMemory class is now deprecated in favor of agents with checkpointers.

### How do you clean up a chatbot's memory?

Delete the stored conversation for that session or user, and delete any long-term facts saved under their ID. In your own app that means removing rows from the history and memory tables. In ChatGPT, you manage memories under Settings > Personalization > Memory.

