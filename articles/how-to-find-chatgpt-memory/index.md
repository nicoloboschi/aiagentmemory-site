# How to Find ChatGPT Memory: View, Turn Off, Delete

> How to find ChatGPT memory in Settings, see what it remembers, turn memory on or off, and fully delete saved memories and chat history, on web and phone.

Source: https://aiagentmemory.org/articles/how-to-find-chatgpt-memory/ · Updated 2026-10-12 · aiagentmemory.org


To **find ChatGPT memory**, open **Settings > Personalization > Memory**. There you'll see a memory summary or a list of saved memories, plus switches to turn memory and chat history reference on or off. You can also just ask ChatGPT "What do you remember about me?" or tap **Sources** under a reply.

## What is ChatGPT memory?

**ChatGPT memory is the feature that lets ChatGPT reuse details from earlier chats to personalize new ones.** Depending on plan and region, OpenAI says it can draw on past chats, saved memories, custom instructions, files in your Library, and connected apps such as Gmail ([OpenAI Memory FAQ](https://help.openai.com/en/articles/8590148-memory-faq)).

It isn't a full transcript. OpenAI states that memory "does not retain every detail from every conversation" and that ChatGPT picks what seems relevant to each answer. It's also separate from your **chat history** in the sidebar, which is just the list of past conversations.

Memory is available on Free as well as paid plans. Free users got saved memories first and, from June 2025, references to recent conversations ([ChatGPT release notes](https://help.openai.com/en/articles/6825453-chatgpt-release-notes)).

## How to see what ChatGPT remembers

1. Open **Settings**.
2. Select **Personalization**.
3. Select **Memory**.
4. Read the **Memory summary** if your account has it, or select **Manage** (or **Saved memories**) to see the list of individual memories.
5. In any chat, tap **Sources** under a personalized reply to see which saved memory, past chat, custom instruction or file shaped it.
6. Ask ChatGPT directly: "What do you remember about me?"

The labels differ by account, because OpenAI is moving users between two memory systems (see below). If a setting is missing, OpenAI says it may not be available for your account yet.

The summary isn't everything. OpenAI's FAQ says it's "a high-level view" that "may omit details," so Sources and asking ChatGPT can surface things the summary doesn't show.

## Improved memory vs legacy saved memories

In June 2026 OpenAI started moving users to **improved memory**, starting with Plus and Pro in the US, with Free and Go to follow. It replaced a flat list with a running summary.

| | Improved memory | Legacy saved memories |
|---|---|---|
| What you see | A **memory summary** you can edit in place | A list of individual memory items |
| How it updates | Updated automatically as your context changes | Saved when you ask, or when ChatGPT judges a detail useful |
| How to correct | Type a correction, or highlight text and correct it | Edit or delete a single item |
| Extra options | "Don't mention this again" | Search, sorting, automatic prioritization, version history (plan dependent) |

Where both are offered, you switch in **Settings > Personalization > Memory**: choose **Saved memories** for the legacy list, or **Try improved memory** to go back. Switching doesn't delete your chat history.

## How to turn ChatGPT memory on or off

Go to **Settings > Personalization > Memory** and use the toggles. Depending on your account, you'll see one **Memory** switch, or two: **Reference saved memories** and **Reference chat history**.

How they interact, per OpenAI:

- Turning off **Reference saved memories** also turns off **Reference chat history**.
- Turning off **Reference chat history** keeps your saved memories, and schedules what ChatGPT learned from past chats for deletion within 30 days.
- Turning off **Memory** doesn't delete past chats. If you turn it back on, ChatGPT may build new memories from chats still in your history.

For a single conversation without memory, start a **Temporary Chat**: open a new chat, select **Temporary**, and choose **Unpersonalized** before your first message. Temporary chats never create or update memories, and they stay out of your history unless you save them ([Temporary chat help](https://help.openai.com/en/articles/8914046)).

## How to delete or wipe ChatGPT memory

Deleting memory takes more than one click because the same fact can live in several places. OpenAI lists what to clear:

| To remove | Do this |
|---|---|
| One memory | Delete it in **Memory > Manage**, edit the summary, or tell ChatGPT to forget it |
| All memory at once | In the memory summary, open the **•••** menu and choose **Delete and turn off memory** |
| The original source | Delete the chat where you shared it, including archived chats |
| Files | Delete the file from **Library** |
| Connected data | Disconnect the app (for example Gmail) |

Three points catch people out:

- **Deleting a chat doesn't delete a memory** made from it. They're stored separately.
- **Deleting a memory doesn't edit old chats.** Mentions stay in past conversations until you delete those too.
- **"Don't mention this again" isn't deletion.** It reduces future references but keeps the source.

OpenAI may keep logs of deleted saved memories for up to 30 days for safety and debugging. On a phone, the steps are the same Settings > Personalization > Memory path. OpenAI noted in June 2026 that "Delete and turn off memory" reached the web first, with mobile following.

## ChatGPT memory full: how to free up space

The "memory full" message applies to the legacy saved memories list, which has a capacity limit. To free space, open **Memory > Manage** and delete entries you don't need, starting with outdated or duplicate ones.

OpenAI has reduced the problem twice. In October 2025 it began managing saved memories automatically for Plus and Pro on the web, "moving less important ones to the background" to help "avoid 'memory full' state." The June 2026 improved memory gave Plus and Pro "twice as much memory capacity." OpenAI also says there's no separate storage limit for what ChatGPT can reference through chat history.

## Memory in ChatGPT projects

Projects can have **project-only memory**. With it on, chats in the project can use other chats in that project, but nothing from outside, and nothing from the project leaks into other chats. You can change an existing project's setting from the project's three-dot menu under **Project settings > Memory**. Shared projects always use project-only memory.

This is the cleanest way to keep, say, a work project from coloring personal chats. Claude does something similar with per-project memory; see [how chatbot memory compares across apps](/articles/best-chatbot-for-memory/).

## ChatGPT memory vs memory you build yourself

ChatGPT's memory is a feature of the ChatGPT app. If you build on OpenAI's developer tools, you design memory yourself; [memory in the OpenAI Agents SDK](/articles/openai-agents-sdk-memory/) covers that side. Open-source memory servers such as [Hindsight](https://github.com/vectorize-io/hindsight) give your own agent a store you can inspect and delete from, with an OpenAI Agents SDK integration; each write calls an LLM, so it costs more than a plain database. For why every chatbot still forgets things, even with memory on, see [why AI memory is so bad](/articles/why-is-ai-memory-so-bad/), and for Anthropic's version, [Claude's long-term memory](/articles/claude-ai-long-term-memory/).


## FAQ

### Where can I see my ChatGPT memory?

Open Settings, select Personalization, then Memory. Depending on your account you'll see a memory summary or a Saved memories list under Manage. You can also ask ChatGPT what it remembers about you, or tap Sources under a reply to see which memory or past chat shaped it.

### Does deleting a chat delete ChatGPT's memory of it?

Not necessarily. OpenAI says saved memories are stored separately from chat history, so deleting a chat doesn't automatically delete a memory created from it. To remove something fully, delete the saved memory and the chat, plus any archived chats, Library files or connected apps that hold it.

### How do I fix 'memory full' in ChatGPT?

Delete old or unneeded entries in Settings > Personalization > Memory > Manage. Since October 2025, Plus and Pro accounts also manage saved memories automatically, moving less important ones to the background to avoid a full memory, and the newer improved memory doubled capacity for Plus and Pro.

