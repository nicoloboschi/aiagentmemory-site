# Claude AI Long-Term Memory: How It Works and Settings

> Does Claude have long-term memory? How Claude's memory and chat search work, plans, how to turn memory on, import from ChatGPT, context window sizes, and the API.

Source: https://aiagentmemory.org/articles/claude-ai-long-term-memory/ · Updated 2026-10-12 · aiagentmemory.org


**Claude AI long-term memory** is a feature of the Claude apps that saves what you tell it as **individual memory topics** and uses them in later chats. Paid plans can also **search past chats** on demand. You control both in **Settings > Memory**, where you can view and edit each topic, pause memory or reset it. Memory is on by default for Free, Pro and Max.

That covers the chat apps. Developers building on the Claude API get a different tool, and Claude Code has its own memory files. All three are below. Facts come from Anthropic's help center and docs, checked on 8 October 2026.

## Does Claude have long-term memory?

**Yes. Claude has two ways to remember across conversations. Memory saves facts and context as topics you can read and edit, and Claude uses them in new chats. Chat search lets Claude look up relevant past conversations with a retrieval tool call when you refer to earlier work. Both are optional and adjustable in Settings.**

The details, per Anthropic's [chat search and memory article](https://support.claude.com/en/articles/11817273-use-claude-s-chat-search-and-memory-to-build-on-previous-context):

- **Memory** "saves memory as a set of individual topics as you chat, rather than summarizing conversations after they end." You can also ask Claude directly to remember something.
- **Chat search** works across non-project chats, and within a project for that project's chats. It uses RAG and shows up as a tool call in the conversation. It's on Pro, Max, Team and Enterprise, not Free.
- **Projects** get their own memory space and summary, separate from other projects and from regular chats.

### When memory came to each plan

| Date | Change | Source |
|---|---|---|
| 11 Sep 2025 | Memory for Team and Enterprise; incognito chat for all users | [Anthropic](https://claude.com/blog/memory) |
| 23 Oct 2025 | Memory for Pro and Max | Anthropic |
| 2 Mar 2026 | Memory for the Free plan, plus easier import from other chatbots | [Engadget](https://engadget.com/ai/anthropic-brings-memory-to-claudes-free-plan-220729070.html) |

On Team and Enterprise, owners decide whether memory is available, and it starts off for each member until they turn it on.

## What Claude remembers, and what it won't

Claude's memory leans toward work. Anthropic's import guide says Claude "may not retain imported personal details unrelated to work."

**Sensitive topics are off by default.** Claude doesn't store personal or sensitive subjects unless you turn on "Include sensitive topics in memory" in Settings > Memory. Then it shows a notice each time it saves one. Some data is never saved even if you ask: government ID numbers, criminal history, financial account numbers and immigration status.

**Deleted chats don't delete memories.** When a conversation expires or you delete it, memory entries generated from it stay. Delete those entries yourself in the memory panel. This is a change from the older "legacy" memory, which removed deleted conversations from its synthesis and updated every 24 hours.

## How to turn Claude memory on, off or reset it

Steps for the current memory experience on web and desktop:

1. Open **Settings > Memory**.
2. Toggle **Generate memory from chats** to turn memory on or off.
3. Toggle **Search and reference chats** to control chat search (paid plans).
4. Use the memory panel to read topics, edit them, or tell Claude what to change or remove.
5. To stop memory without deleting it, **pause** it. Existing memory stays, but nothing new is added and nothing is used.
6. To wipe everything, **reset** memory. This "permanently deletes all memories including project memories" and can't be undone.

For a single conversation, two options:

- **Incognito chat** (the ghost icon) isn't saved to memory or history, and Claude won't find it in search later. It's on every plan.
- **Memory off for one chat** (the "+" menu, before your first message) keeps the chat in history and searchable but leaves it out of memory.

A few Team and Enterprise organizations still use legacy memory, which lives under Settings > Capabilities instead.

## How to import memory into Claude from ChatGPT or Gemini

Anthropic's [import and export guide](https://support.claude.com/en/articles/12123587-import-and-export-your-memory-from-claude) lists four steps, available on Free, Pro, Max and Team plans on the web and Claude Desktop:

1. **Export from the other assistant.** Use Anthropic's suggested prompt, which starts "I'm moving to another service and need to export my data" and asks for one code block of dated entries. Edit it to leave out anything sensitive.
2. **Start the import.** In Claude, open Settings > Memory and select **Start import**.
3. **Paste and add.** Paste the text and click **Add to memory**. Claude stores it as individual entries.
4. **Review.** Check the entries in the memory panel, then click **See what Claude learned about you** for a summary chat.

Limits to know: only memory transfers, not chat history. The feature is experimental, so Claude may not keep every item. To **export** Claude's memory, ask in a chat: "Write out your memories of me verbatim, exactly as they appear in your memory," and save the result. Memory data is also included in a full account data export.

If you're coming from ChatGPT, the guide on [how to find ChatGPT memory](/articles/how-to-find-chatgpt-memory/) shows where to view what it saved before you export.

## Claude's context window in the apps

Memory and the context window are different things. The **context window** is how much text Claude can read in one conversation; memory is what carries across conversations. The [context window guide](/articles/context-window-of-an-llm/) explains the difference.

Anthropic's [context window article](https://support.claude.com/en/articles/8606394-how-large-is-the-context-window-on-paid-claude-plans) for paid plans (Pro, Max, Team, Enterprise) lists sizes by model in chat:

| Context window | Models (Claude chat, October 2026) |
|---|---|
| 1M tokens | Fable 5.1, Opus 5.5, Opus 5, Sonnet 5.5, Sonnet 5, Haiku 5.5 |
| 500K tokens | Fable 5, Opus 4.8, Opus 4.7, Opus 4.6, Sonnet 4.6 |
| 200K tokens | Other models |

Claude Code and Cowork have their own per-model limits. On paid plans with code execution enabled, Claude **automatically summarizes earlier messages** when a chat nears its limit, which "allows conversations to continue indefinitely in most cases." The full history stays available to reference, but these long chats use more of your usage limit.

## Memory for developers: the Claude API and Claude Code

The app memory above isn't exposed through the API. Developers have two separate mechanisms.

### The memory tool in the Claude API

The [memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) lets Claude create, read, edit and delete files in a `/memories` directory across conversations. It runs **client-side**: Claude asks for an operation (`view`, `create`, `str_replace`, `insert`, `delete`, `rename`) and your code executes it against storage you control. It's available on Claude 4 and later models, with tool type `memory_20250818`. The Python SDK ships a local-filesystem implementation:

```python
import anthropic
from anthropic.tools import BetaLocalFilesystemMemoryTool

client = anthropic.Anthropic()
memory = BetaLocalFilesystemMemoryTool(base_path="./memory")

runner = client.beta.messages.tool_runner(
    model="claude-opus-5-5",
    max_tokens=1024,
    messages=[{"role": "user",
               "content": "Remember that Acme Corp prefers email follow-ups."}],
    tools=[memory],
)
print(runner.until_done().content)
```

If you write your own handler, Anthropic's docs require rejecting any path outside `/memories` to block directory traversal. The tool also pairs with compaction for long-running agents: compaction shrinks the conversation, memory keeps what must survive it.

### Claude Code memory

[Claude Code](https://code.claude.com/docs/en/memory) uses **CLAUDE.md** files that you write (project, user or organization rules) and **auto memory**, notes Claude writes itself about your preferences, corrections, project context and references. Auto memory lives per repository in `~/.claude/projects/<project>/memory/`, and the first 200 lines or 25KB load at the start of each session. For memory that searches past sessions instead of a capped file, plugins such as [Hindsight](https://github.com/vectorize-io/hindsight)'s build a per-repo memory bank from git history and earlier sessions and inject the relevant parts as Claude Code starts work; see [AI coding agent memory](/articles/ai-coding-agent-memory/).

## How Claude's memory compares

Against other chatbots, Claude's strengths are editable topics, project-level separation and clear per-chat controls. ChatGPT pulls from more sources, and Gemini's saved-facts feature is narrower; see [best chatbot for memory](/articles/best-chatbot-for-memory/) for the side-by-side and [Gemini AI long-term memory](/articles/gemini-ai-long-term-memory/) for Google's version.

If you're building your own assistant on Claude and need memory per end user, the memory tool gives you files, not search or fact extraction. Teams often add a memory layer such as Mem0, Zep, Letta or Hindsight for that. Hindsight's `wrap_anthropic()` wraps the Anthropic SDK client so memories are recalled before each call and retained after, though each write makes an extra LLM call; the [AI agent memory guide](/articles/ai-agent-memory-explained/) explains the options.


## FAQ

### Does Claude AI have long-term memory?

Yes. Claude can save memory as individual topics while you chat and use them in later conversations, and paid plans can also search past chats. Memory is on by default for Free, Pro and Max. On Team and Enterprise, the owner enables it and each member turns it on. You can view, edit, pause or reset it in Settings > Memory.

### How do I import my ChatGPT memory into Claude?

Ask your old assistant to list everything it remembers about you (Anthropic provides a prompt), then in Claude go to Settings > Memory, select Start import, paste the text and click Add to memory. It works on Free, Pro, Max and Team plans on web and desktop. Only memory transfers, not chat history, and Anthropic calls the feature experimental.

### How large is Claude's context window?

In the Claude apps on paid plans it depends on the model: per Anthropic's help center (October 2026), the newest models such as Opus 5.5, Sonnet 5.5 and Haiku 5.5 have 1M tokens, several earlier models have 500K, and other models have 200K. With code execution on, Claude summarizes older messages in long chats so they can continue.

