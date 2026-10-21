# Gemini AI Long-Term Memory: Past Chats and Settings

> How Gemini's long-term memory works in 2026: memory from past chats, asking Gemini to remember, importing memory from ChatGPT or Claude, how to delete it, and context windows.

Source: https://aiagentmemory.org/articles/gemini-ai-long-term-memory/ · Updated 2026-10-20 · aiagentmemory.org


**Gemini AI long-term memory** is the setting in the Gemini app that lets it use details from your **past chats** to personalize answers, plus the option to ask it to **remember specific facts**. It needs you to be 18 or older, signed in with a personal Google Account, with **Keep Activity** on. You switch it in Gemini's personalization settings and erase it by deleting chats.

There's no list of saved memories to scroll through and prune, unlike ChatGPT or Claude. That's the biggest practical difference. Everything here comes from Google's Gemini Apps help pages and Google blog posts, checked on 8 October 2026; Google's own pages differ slightly by device and region, which is noted where it matters.

## Does Gemini have long-term memory?

**Yes. Gemini can remember across conversations in two ways. Memory of past chats lets it pull relevant details from your earlier Gemini conversations. Saved details are facts you ask it to remember, like "I'm vegetarian." Both are tied to your Google Account activity, and you control them through personalization settings and Gemini Apps Activity.**

Google launched past-chat personalization on 13 August 2025, [per its blog post](https://blog.google/products-and-platforms/products/gemini/temporary-chats-privacy-controls/). It started with Gemini 2.5 Pro in select countries for users over 18, and the same announcement added Temporary Chats and renamed "Gemini Apps Activity" to "Keep Activity."

| Feature | What it does | Who gets it (Oct 2026) |
|---|---|---|
| **Memory of past chats** | Uses details from earlier chats in new answers | 18+, personal account, Keep Activity on |
| **"Remember that..."** | Stores specific facts you state | Google AI plan required; US only for now |
| **Instructions** | Rules applied to every chat, like "start with a short summary" | All users |
| **Temporary chat** | A chat that isn't used for memory | All users |

## How Gemini's memory from past chats works

When Memory is on, Gemini can reference relevant earlier conversations to tailor a reply. Google's [past-chats help page](https://support.google.com/gemini/answer/16598469?hl=en) lists the requirements:

- Be 18 or over.
- Sign in with a **personal** Google Account. It "isn't available when you sign in to a work, school, or supervised Google Account."
- Have **Keep Activity** on. Without it, the feature is off.

It works in the Gemini mobile app, the web app at gemini.google.com, Gemini in Chrome where available, and on smartwatches. It's **not available in Gems or Live chats**, though in a text chat you can ask Gemini to reference a past Live conversation.

To check whether a reply drew on your history, ask: **"Did you use any info from past chats?"** One version of the help page (iPhone) adds that the feature isn't offered in the EEA, Switzerland or the UK and needs Gemini set to English; check the page for your device and region.

## Asking Gemini to remember specific facts

You can also tell Gemini directly: "Remember that I am vegetarian." To fix a fact, correct it in chat: "Actually, my daughter is 8, not 7," and Gemini says it will update its memory.

This narrower feature has stricter rules. Google's [memory management page](https://support.google.com/gemini/answer/16598625?hl=en) says you need a **Google AI plan**, and that for now it's **only available in the US**. Instructions are open to everyone. Add them under **Personal context > Your instructions for Gemini**, where you can edit, delete or switch them off.

## Can you import memory from ChatGPT or Claude into Gemini?

**Yes. Since 26 March 2026, Gemini can import both your memories and your chat history from other AI apps, per [Google's announcement](https://blog.google/innovation-and-ai/products/gemini-app/switch-to-gemini-app/).** The same post renamed the "past chats" feature to "memory." Google doesn't name the source apps; it says "other AI apps."

There are two separate imports, described on Google's [import page](https://gemini.google/import-memory/):

- **Memories.** Copy Gemini's suggested prompt into your old AI app, let it write a summary of your preferences and context, then paste that summary into Gemini's import settings. It's a text summary, not a file transfer, so it carries only what the old app chooses to write.
- **Chat history.** Request a data export from your old provider, then upload the ZIP under **Chat history import** in Gemini's settings. Google's limit is 5 GB per ZIP. Only the text of chats comes across; images and files generated in the old app, and project files, are not included. Imported chats show up in the chat list with an import icon, and you can delete one chat or a whole ZIP's worth at once.

Who gets it: personal accounts of users 18 and over, with personalization on. Google's post says business, Enterprise and under-18 accounts aren't supported, and it wasn't yet available in the EEA, the UK or Switzerland at launch.

## How to turn Gemini memory on, off or delete it

Google's pages use two labels for the same area, **Personal context** and **Personal Intelligence**, depending on device and version. The steps:

1. **Mobile app:** tap Menu, then your profile picture, then **Personal context** (or Settings > Personal Intelligence).
2. **Web app:** open Menu, then **Settings & help**, then **Personal context**.
3. Switch **Memory** on or off.
4. To remove what Gemini knows, open [Gemini Apps Activity](https://myactivity.google.com/product/gemini) and **delete every chat** that contains the information.
5. For information that came from a connected Google app, delete those chats **and** disconnect the app. Google says doing only one isn't enough.

Expect a lag. Google notes "a short delay before Gemini stops using" a deleted chat for personalization. It also warns that asking Gemini to forget or avoid a topic "doesn't always work perfectly," which is why deleting chats is the reliable route.

### Temporary chats

A **Temporary Chat** doesn't appear in your recent chats or in Gemini Apps Activity, isn't used to personalize later answers, and isn't used to train Google's models. Google keeps it for up to **72 hours** to respond and handle any feedback you send. Use it when you don't want something to become memory.

## Gemini's context window by plan

**Memory** is what carries between chats. The **context window** is how much Gemini can read inside one chat, including uploaded files. Google's [limits page](https://support.google.com/gemini/answer/16275805?hl=en) lists:

| Plan | Context window in the Gemini app |
|---|---|
| No AI plan (free) | 32K tokens |
| Google AI Plus | 128K tokens |
| Google AI Pro and AI Ultra | 1 million tokens |

Ultra doesn't raise the window over Pro; it raises usage limits. When a chat or file goes past the window, Google says answers may miss content or connections. The Gemini API has its own per-model limits, and the [context window guide](/articles/context-window-of-an-llm/) explains why long windows don't replace memory.

## Google's long-term memory for developers

The app's memory isn't available through the Gemini API, which is stateless: each request only sees what you send. Developers who want an agent to remember users on Google's stack use:

- **Google ADK** session state (with `user:` and `app:` prefixes for cross-session values) and its memory services.
- **Agent Platform Memory Bank**, formerly [Vertex AI Agent Engine Memory Bank](/articles/vertex-ai-agent-engine-memory-bank/), a managed service that extracts and consolidates memories per user with Gemini.
- **An open-source memory server** if you want to self-host or switch models later. [Hindsight](https://github.com/vectorize-io/hindsight) (MIT) has a Google ADK integration and works with 25+ LLM providers; it adds an LLM call on every write, so it's more than a simple chatbot needs.

People searching "Google long-term memory AI" often mean Google Research's **Titans** architecture, which adds a learned memory module inside the model. That's research, not a product feature; the [Titans explainer](/articles/google-titans-give-ai-human-like-memory/) covers it.

## How Gemini's memory compares with other chatbots

Gemini's memory is tied to your account activity rather than a separate memory store. That makes it simple to wipe (delete chats) but hard to inspect (no list of saved facts). ChatGPT and Claude both show what they've saved and let you edit single items; see [Claude AI long-term memory](/articles/claude-ai-long-term-memory/) and the [best chatbot for memory](/articles/best-chatbot-for-memory/) comparison.


## FAQ

### Does Gemini remember past conversations?

Yes, if Memory is on. Gemini can use details from your past chats to personalize answers when you're 18 or older, signed in with a personal Google Account, and have Keep Activity on. It isn't available in Gems or Live chats. You can ask Gemini 'Did you use any info from past chats?' to check.

### How do I make Gemini forget something?

Delete every chat that contains the information in Gemini Apps Activity (myactivity.google.com/product/gemini). Google says asking Gemini to forget a topic 'doesn't always work perfectly,' and there may be a short delay before deleted chats stop being used. For details from a connected app, delete the chats and also disconnect the app.

### Can I import my ChatGPT or Claude memory into Gemini?

Yes, since March 2026. For memories, paste Gemini's suggested prompt into your old AI app and paste the summary it writes into Gemini's import settings. For chat history, upload your old provider's data-export ZIP (up to 5 GB) under Chat history import; only chat text comes across. Personal accounts, 18+; not available at launch in the EEA, UK or Switzerland.

### What is Gemini's context window in the app?

Per Google's Gemini Apps help page (October 2026), it's 32K tokens without an AI plan, 128K tokens with Google AI Plus, and 1 million tokens with Google AI Pro or Ultra.

