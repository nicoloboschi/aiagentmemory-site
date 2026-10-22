# Voice Agent Memory: How Voice AI Remembers Callers

> How to give a voice agent long-term memory: the latency budget, recall per turn vs per call, and the options (VoiceMem, Mem0, Zep, Hindsight) for Pipecat, LiveKit and Vapi.

Source: https://aiagentmemory.org/articles/voice-agent-memory/ · Updated 2026-10-22 · aiagentmemory.org


## How does voice agent memory work?

**Voice agent memory** keeps facts about a caller between calls and puts the relevant ones in front of the model before it speaks. A memory store keyed to the caller saves each call's content in the background. On the next call, the agent recalls only what matters to this turn, inside a latency budget of a few hundred milliseconds.

A stock voice pipeline (speech-to-text, LLM, text-to-speech) starts every call with an empty context. When the caller hangs up, everything they said is gone. Pasting old transcripts into the prompt doesn't scale: the prompt grows with every call, and a longer prompt delays the first spoken word.

## What makes memory for voice agents different?

Three things separate voice from text chat.

- **Latency is audible.** A text user tolerates a second of "thinking". A caller hears dead air. Any memory lookup that runs before the LLM adds directly to time-to-first-word.
- **Turns are short and noisy.** Speech transcripts are full of fillers, false starts and back-channel ("mm-hm"). Storing raw transcripts stores the noise too, so the memory layer has to extract facts, not just save text.
- **Identity comes from the line.** A phone number is available before the agent answers, which makes it a natural memory key. One person may call from several numbers, so an account ID is better when you have one.

Emotion matters more too. The VoiceMem paper (arXiv 2608.26005, August 2026) argues that a voice agent should remember how a caller felt and how they relate to others, not only facts. Its design splits memory into a factual "left brain" and an emotional "right brain".

## Recall per turn or per call?

There are two places to put recall, and the voice platform often decides for you.

| | Per turn | Per call |
|---|---|---|
| When recall runs | Before every LLM reply | Once, when the call starts |
| Latency cost | Paid on every turn | Paid once, before the greeting |
| Follows topic changes | Yes | No |
| Typical platform | Pipecat, LiveKit Agents (you own the pipeline) | Vapi and other hosted platforms with webhooks |
| When writes happen | After each completed turn | After the call ends (full transcript) |

Per-turn recall follows the conversation: if the caller switches from billing to a delivery problem, the next lookup finds delivery history. Per-call recall is cheaper and simpler, but it only knows what the opening query found. Many teams combine them: load a short caller profile at call start, then do light per-turn recall.

## Memory options for voice agents

| | VoiceMem | Mem0 (Pipecat service) | Zep (LiveKit) | Hindsight |
|---|---|---|---|---|
| What it is | Memory system built for duplex speech models | General memory layer, integrated by Pipecat | Temporal context graph, Zep Cloud | Open-source memory server (MIT) |
| Voice integration | Own streaming pipeline | `Mem0MemoryService` in Pipecat | `zep-livekit` (`ZepUserAgent`, `ZepGraphAgent`) | `hindsight-pipecat`, `hindsight-vapi` |
| Recall timing | Streaming, writes while the user talks | Per turn, before the LLM | Per turn, save and context in one call | Per turn (Pipecat) or per call (Vapi) |
| Writes block the reply? | No (streaming) | No (background threads) | Run with the turn | No (fire-and-forget) |
| Emotion / persona memory | Yes, separate "right brain" | No dedicated store | No dedicated store | No dedicated store |
| Self-hosting | Yes, Apache 2.0 | Yes (open-source Mem0) | Graphiti only; Zep's self-hosted edition is deprecated | Yes, full server |
| Published latency | 134 ms retrieval (own paper) | Not stated in Pipecat docs | Not stated | Not published |

**VoiceMem** is the only option here designed for speech first. Its paper reports 134 ms retrieval, which the authors say fits inside normal voice activity detection delay. Its README reports 91.2% on LoCoMo with the top 5 memories, against 61.68% for Mem0, and 430 memory tokens per use against 6,956 for Mem0. These are the VoiceMem authors' own numbers, and the Mem0 figures are their runs, not Mem0's. The repo ([xzf-thu/VoiceMem](https://github.com/xzf-thu/VoiceMem)) was created in August 2026 and had about 2,600 GitHub stars on October 22, 2026; its last push was September 14.

**Mem0** is the default memory service in Pipecat. According to the [Pipecat Mem0 docs](https://docs.pipecat.ai/server/services/memory/mem0), `Mem0MemoryService` sits before the LLM, retrieves up to 10 memories per turn by default, and runs every Mem0 call in background threads. Since Mem0 2.0, writes are processed asynchronously on the server, so a fact said in this turn may not be recallable in the next one.

**Zep** published `zep-livekit` 1.0.0 on PyPI on October 14, 2026. Its `ZepUserAgent` saves messages and gets context back in one round trip per turn. Zep also ingests asynchronously, so new messages aren't searchable right away. More on Zep's model in [what Zep memory is](/articles/what-is-zep-memory/).

## Where Hindsight fits

[Hindsight](https://github.com/vectorize-io/hindsight) covers both recall patterns with official packages. `hindsight-pipecat` adds one processor between Pipecat's user aggregator and the LLM: it recalls on the latest user message before each reply and retains each completed user and assistant turn without blocking. `hindsight-vapi` is a webhook handler that recalls at call start (using the caller's phone number as the query) and retains the full transcript when the call ends, with a helper for outbound calls.

```python
from pipecat.pipeline.pipeline import Pipeline
from hindsight_pipecat import HindsightMemoryService

memory = HindsightMemoryService(
    bank_id=f"caller-{caller_number}",  # one bank per caller, not per call
    hindsight_api_url="http://localhost:8888",  # self-hosted; or Hindsight Cloud
    recall_budget="low",        # favor latency on live calls
    recall_max_tokens=2048,     # cap what gets injected
)

pipeline = Pipeline([
    transport.input(), stt_service, user_aggregator,
    memory,  # recall before the LLM, retain after the turn
    llm_service, assistant_aggregator, tts_service, transport.output(),
])
```

Its strengths for voice are the ones it has everywhere. Retain extracts facts, times and entities with an LLM, so the bank holds facts rather than raw transcript. Recall runs semantic, keyword, graph and time-range search in parallel, which helps with "what did I order last month" questions. Hindsight scored 94.6% on LongMemEval-S and 73.9% on BEAM 1M in benchmarks Vectorize runs and reports, October 2026. Its lead is largest on very long histories, which matters for callers who come back for months. The whole server is MIT-licensed and self-hostable, including graph and temporal recall.

The limits matter more in voice than in text:

- **No published latency figures.** Vectorize hasn't published recall latency for voice. Measure it on your own setup before going live.
- **Bigger context by default.** Mem0's own comparison measured Hindsight returning about 4x more tokens per retrieval than Mem0. The Pipecat default is 4,096 tokens; Hindsight's voice guide suggests `recall_budget="low"` and a lower `recall_max_tokens` for live calls.
- **Reflect is too slow for a live turn.** Use plain recall on the hot path. Mental models (standing summaries Hindsight keeps up to date) are a database read with no LLM call, which suits a caller profile loaded at call start.
- **No emotion store.** Hindsight records what was said, not a separate model of the caller's mood, which VoiceMem does.

More detail on the system is in the [Hindsight guide](/articles/ai-memory-hindsight/) and the [Mem0 vs Hindsight comparison](/articles/mem0-vs-hindsight/).

## How to add memory to a voice agent

1. **Pick the memory key.** Use the caller's account ID if you authenticate, otherwise a normalized phone number. Never the call ID, or every call is a first call.
2. **Choose per-turn or per-call recall.** Own the pipeline (Pipecat, LiveKit): per turn. Hosted platform with webhooks (Vapi): per call.
3. **Make writes non-blocking.** Store after the turn or after the call, in the background. None of the options above should hold up the reply.
4. **Cap what recall returns.** A caller needs two or three facts to feel known, not a page of history. Set a token limit.
5. **Measure time-to-first-word with memory on and off.** That difference is your memory cost per turn.
6. **Test across two calls.** State a fact on call one, hang up, call again from the same identity and ask a question that depends on it.
7. **Plan for deletion.** Callers will ask what you store about them. Per-caller banks make "delete everything about me" one operation.

## Which memory should a voice agent use?

- **Research or duplex speech models, emotion matters:** VoiceMem. It's the only speech-first design, but it's young and its benchmarks are self-reported.
- **Pipecat, simplest path:** the built-in Mem0 service, or `hindsight-pipecat` if you want a full open-source server with graph and temporal recall.
- **LiveKit Agents on Zep Cloud:** `zep-livekit`.
- **Vapi, callers who return for months:** `hindsight-vapi`, with recall at call start and the long-history lead noted above.

For the general picture, see [long-term memory for AI agents](/articles/ai-agent-long-term-memory/).


## FAQ

### How do voice agents remember callers between calls?

They store facts from each call in a memory bank keyed to the caller (phone number or account ID), then recall the relevant ones before the model replies on the next call. Storing runs in the background; only recall sits on the response path.

### Does memory add latency to a voice agent?

Recall does, because it runs before the LLM generates. Writes can be fire-and-forget. VoiceMem reports 134 ms retrieval in its own paper; Hindsight's voice guide recommends a low recall budget and a token cap for live calls.

### Which memory tools work with Pipecat, LiveKit and Vapi?

Pipecat ships a Mem0 memory service and Hindsight publishes hindsight-pipecat. Zep publishes zep-livekit for LiveKit Agents. Hindsight publishes hindsight-vapi, which recalls at call start and stores the transcript at call end.

