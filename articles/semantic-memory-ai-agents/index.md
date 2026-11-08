# Semantic Memory in AI Agents: Facts, Profiles, Graphs

> What semantic memory is in AI agents: profiles vs collections, triples and graphs, extracting facts with LangMem, how Mem0, CrewAI, Graphiti and Hindsight handle a changed fact, and LongMemEval knowledge-update scores by system.

Source: https://aiagentmemory.org/articles/semantic-memory-ai-agents/ · Updated 2026-11-08 · aiagentmemory.org


**Semantic memory in AI agents** stores facts: who the user is, what they prefer, what's true about the domain, and how entities relate. Unlike episodic memory, a semantic fact isn't tied to the moment it was learned. The agent saves these facts outside the model and pulls the relevant ones into the prompt to ground and personalize its answers.

It's the most common kind of agent memory in production, because most "the assistant remembers me" features are semantic memory. The hard part isn't storing facts. It's deciding what counts as a fact, and keeping facts correct when they change. The pillar guide, [AI agent memory explained](/articles/ai-agent-memory-explained/), shows where semantic memory sits among the other types.

## What is semantic memory in AI agents?

**Semantic memory in an AI agent is a long-term store of facts and concepts, detached from when they were learned: user attributes and preferences, domain knowledge, and relationships between entities. The agent writes facts into it from conversations or documents and retrieves them to ground its responses.**

In psychology, semantic memory is general knowledge, distinct from episodic memory of specific events. The CoALA framework ([Sumers et al., 2023](https://arxiv.org/abs/2309.02427)) carries the term over to agents: semantic memory "stores an agent's knowledge about the world and itself."

### Semantic memory is not semantic search

LangGraph's [memory guide](https://docs.langchain.com/oss/python/langgraph/memory) makes this point directly. **Semantic memory** is a term from psychology for stored facts. **Semantic search** is a retrieval method that matches by meaning, usually with embeddings. You can implement semantic memory as a JSON profile you look up by user ID, with no vectors involved.

## Where semantic memory comes from

CoALA describes two sources.

- **Read-only corpora.** Classic retrieval-augmented generation reads from human-written text such as Wikipedia, product docs or a catalog. CoALA calls this retrieving from "a semantic memory of unstructured text." The agent never writes to it.
- **Agent-written knowledge.** Agents can also write new knowledge into semantic memory as a form of learning. A chat assistant extracts "the user is vegetarian" from a conversation. Reflexion ([Shinn et al., 2023](https://arxiv.org/abs/2303.11366)) reflects on a failed attempt and stores the lesson ("there is no dishwasher in kitchen") for the next try.

The second source is what turns RAG into memory: the store changes with every interaction. The distinction is covered in more depth in [RAG vs agent memory](/articles/rag-vs-agent-memory/).

Facts often come from episodes. Generative Agents ([Park et al., 2023](https://arxiv.org/abs/2304.03442)) periodically reflects over raw events and writes conclusions like "I like to ski now," which CoALA classifies as semantic memory. See [episodic memory in AI agents](/articles/episodic-memory-in-ai-agents/) for the event side.

## Profile vs collection

LangGraph and LangMem name two ways to hold semantic memory. The choice affects extraction, updates and retrieval.

| | Profile | Collection |
|---|---|---|
| Shape | One JSON document per user or entity, with a fixed schema | Many small records, one fact each |
| Update | Rewrite or patch the document with each new fact | Insert new records; update or delete conflicting ones |
| Retrieval | Load the whole profile, no search needed | Search per query (vector, keyword, filter) |
| Strength | Fast, predictable, easy for users to view and edit | Open-ended; LangGraph says it "tends to lead to higher recall downstream" |
| Weakness | Gets error-prone as the profile grows; only holds fields you defined | Over-inserting creates duplicates; facts lose context; updates are harder |
| Good for | Name, language, timezone, plan tier, style preferences | Preferences, relationships, project facts, domain notes |

LangMem's [conceptual guide](https://langchain-ai.github.io/langmem/concepts/conceptual_guide/) describes profiles as "a single document that represents the current state" and says collections "are what most people think of when they imagine agent long-term memory." Many systems use both: a small profile always in the prompt, and a searchable collection for everything else.

## How semantic facts are represented

| Representation | Example | Used by | Trade-off |
|---|---|---|---|
| Free-text fact | "User prefers window seats on long flights" | Mem0, most memory APIs | Easy to extract and read; fuzzy to dedupe |
| Triple | (Alice, manages, ML team) | LangMem `Triple` schema, knowledge graphs | Precise, linkable; loses nuance without a context field |
| Knowledge graph | Entities as nodes, facts as edges with metadata | Graphiti/Zep, Cognee | Multi-hop questions and change tracking; more setup |
| Structured profile | `{"name": "Alice", "timezone": "America/Los_Angeles"}` | LangMem profiles, Mastra working memory | Fast lookup; limited to the schema |
| Facts plus consolidated beliefs | World facts with entities and time, merged into observations that cite their evidence | Hindsight | Keeps history and a current view; each write and consolidation calls an LLM |

Graphs help when questions span entities ("who on Bob's team worked on the NLP project?"). Graphiti, for instance, stores facts as edges with validity windows so a fact can be superseded without being deleted. The trade-offs between stores are covered in [knowledge graphs for AI memory](/articles/ai-memory-knowledge-graph/).

## Extracting semantic memories with LangMem

LangMem's `create_memory_manager` prompts an LLM to extract facts into a schema, and with `enable_deletes=True` it can also remove facts that new information contradicts. This is adapted from LangMem's [semantic extraction guide](https://langchain-ai.github.io/langmem/guides/extract_semantic_memories/):

```python
from langmem import create_memory_manager
from pydantic import BaseModel

class Triple(BaseModel):
    """Store all new facts, preferences, and relationships as triples."""
    subject: str
    predicate: str
    object: str
    context: str | None = None

manager = create_memory_manager(
    "anthropic:claude-sonnet-4-5",
    schemas=[Triple],
    instructions="Extract user preferences and any other useful information",
    enable_inserts=True,
    enable_deletes=True,
)

first = [{"role": "user", "content": "Alice manages the ML team and mentors Bob, who is also on the team."}]
memories = manager.invoke({"messages": first})
# -> Triples like (Alice, manages, ML_team), (Alice, mentors, Bob), (Bob, is_member_of, ML_team)

second = [{"role": "user", "content": "Bob now leads the ML team and the NLP project."}]
update = manager.invoke({"messages": second, "existing": memories})
# -> RemoveDoc for (Alice, manages, ML_team), plus new triples (Bob, leads, ML_team), (Bob, leads, NLP_project)
```

This functional API doesn't touch a database. You decide what a `RemoveDoc` means: a hard delete, a soft delete, or a lower ranking. To have LangMem write to a LangGraph store directly, use `create_memory_store_manager` with a namespace.

The guide makes a point worth copying: add a `context` field. A memory like `{"content": "User said yes"}` is useless later. `(user, response, yes, "when asked about attending team meeting")` is not.

## Keeping semantic memory current

Facts change. "I live in NYC" in January becomes "I just moved to SF" in June. If both sit in the store, retrieval may return either. Systems handle this in a few ways:

1. **Overwrite.** Profiles replace the old value. Simple, but history is gone.
2. **Delete on contradiction.** LangMem's manager emits `RemoveDoc` for facts the new message contradicts.
3. **Invalidate with time.** Graphiti marks the old edge invalid from the moment the new fact became true, so "where did Alice live in March?" still works.
4. **Keep both, rank by recency.** Cheap, but pushes conflict resolution onto the model at read time.
5. **Consolidate into beliefs.** Some systems merge related facts into a summary that is refined as evidence comes in. [Hindsight](https://github.com/vectorize-io/hindsight) calls these **observations**: each one keeps quotes from its source facts and a proof count, and gets refined rather than overwritten when a new fact arrives.

Test this case with real data before you pick a system. LongMemEval ([Wu et al., 2024](https://arxiv.org/abs/2410.10813)) has a dedicated "knowledge update" category for it, because it's where many setups fail quietly.

### How each system handles a changed fact

The five strategies above map onto real systems like this. "Decides" is who settles the conflict; "old fact" is what happens to the record that was true before.

| System | When it checks | Who decides | Old fact | Source |
|---|---|---|---|---|
| Profile (LangMem profile, Mastra working memory) | Every write | The extraction LLM rewrites the document | Overwritten, no history | [LangMem guide](https://langchain-ai.github.io/langmem/concepts/conceptual_guide/) |
| LangMem collection, `enable_deletes=True` | Each extraction, against the `existing` memories you pass in | LLM emits `RemoveDoc` | Removed, if your code acts on the `RemoveDoc` | [LangMem extraction guide](https://langchain-ai.github.io/langmem/guides/extract_semantic_memories/) |
| Mem0, as in the 2025 paper | Each new fact against its 10 most similar memories | LLM picks ADD, UPDATE, DELETE or NOOP | Extended or deleted | [Mem0 paper](/articles/mem0-building-production-ready-ai-agents/) |
| Mem0, since April 2026 | Never during extraction | Nobody: single-pass, ADD-only | Kept next to the new one; time-aware retrieval is meant to rank the current one; you call `update` or `delete` | [Mem0 README](https://github.com/mem0ai/mem0) |
| CrewAI `Memory` (1.10+) | On save, when similarity is at or above 0.85, against up to 5 records | LLM picks keep, update, delete or insert_new | Merged or deleted | [CrewAI docs](https://docs.crewai.com/en/concepts/memory) |
| Graphiti / Zep | On ingest, when a new edge contradicts an existing one | LLM-based resolver | Invalidated with a validity window, not deleted | [Graphiti README](https://github.com/getzep/graphiti) |
| [Hindsight](https://github.com/vectorize-io/hindsight) | Background consolidation after each `retain` | LLM refines the observation; since v1.2.0 the code, not the model, decides which fact is newer | Kept as evidence under the observation, with its quote | README, [v1.2.0 notes](https://github.com/vectorize-io/hindsight/releases/tag/v1.2.0) |

Two things stand out. Mem0 moved from "the LLM decides" to "nothing is decided at write time", and pushed the conflict to retrieval; that keeps writes to one LLM call, but it also means both facts stay in the store. Every other row still spends an LLM call per conflict, and Hindsight spends one on consolidation as well as on `retain`, which is its main cost.

Letting a model judge which fact is newer has a failure mode Hindsight documented when it fixed it. In [PR #5966](https://github.com/vectorize-io/hindsight/pull/5966) (merged 22 October 2026, shipped in v1.2.0), when a rule and an amendment to it were stated on the same day and the amendment was retained first, consolidation let the older rule overwrite it every time. The PR gives two causes from the model's own reasoning: the two timestamps reached it in different formats, so it misread which was later; and wording like "updated policy" outweighed the times. The fix moved the comparison into code: one time format in the prompt, and a `newer_than_input_facts` flag on every observation that was last true after every fact in the batch, which the model only follows. The lesson holds for any row in the table where an LLM decides: hand it the comparison already made, not two timestamps to compare.

## How memory systems score on facts that change

Two LongMemEval question types test semantic memory directly. **Knowledge-update** (78 questions) asks for a fact the user later changed. **Single-session-preference** (30 questions) asks the assistant to use a preference the user stated once. Table 3 of the Hindsight paper ([Latimer et al., arXiv 2512.12818](https://arxiv.org/abs/2512.12818), Dec 2025) breaks results down by type, on the S setting (about 115,000 tokens over roughly 50 sessions):

| System (answer model) | Knowledge-update | Preference | Overall |
|---|---|---|---|
| Full context (GPT-4o) | 78.2 | 20.0 | 60.2 |
| Full context (OSS-20B) | 60.3 | 20.0 | 39.0 |
| Zep (GPT-4o) | 83.3 | 56.7 | 71.2 |
| Supermemory (GPT-4o) | 88.5 | 70.0 | 81.6 |
| Supermemory (GPT-5) | 87.2 | 76.7 | 84.6 |
| Supermemory (Gemini-3) | 89.7 | 70.0 | 85.2 |
| Hindsight (OSS-20B) | 84.6 | 66.7 | 83.6 |
| Hindsight (OSS-120B) | 92.3 | 86.7 | 89.0 |
| Hindsight (Gemini-3) | **94.9** | 80.0 | **91.4** |

What the table shows:

- **Putting the whole history in the prompt is worst at preferences.** Full context got 20% on both models: the stated preference is in there, but the model doesn't apply it. Every memory system did at least 2.8x better, which is the case for storing preferences as separate facts.
- **Updates are where systems differ most at the top.** Hindsight with Gemini-3 answered 94.9% of knowledge-update questions and Supermemory with Gemini-3 89.7%. On preferences, Hindsight's best was 86.7% with the open OSS-120B model.
- **The answer model matters as much as the memory.** Hindsight goes from 84.6 to 94.9 on updates just by changing the model that reads the memories.

Read it with three limits. The paper is by Hindsight's authors at Vectorize, so this is a vendor-run comparison, not an independent one. The question counts are small: on preferences one question is 3.3 points. And LongMemEval's judge counts an answer as correct if it gives the new value, even when it also repeats the old one, so the test doesn't punish a system for keeping stale facts around. Hindsight's [benchmarks site](https://benchmarks.hindsight.vectorize.io/) now reports 94.6 overall on LongMemEval S, newer than this table. The [LLM memory comparison](/articles/llm-memory-comparison/) collects other published results.

## Retrieving semantic memory

How facts reach the prompt matters as much as how they're stored:

- **Always-on profile.** Load a small profile into the system prompt on every turn. No search, no misses, fixed token cost.
- **Query-time search.** Search the collection with the user's message. Combine embeddings with keyword search, since names, product codes and IDs often fail pure vector search.
- **Entity lookup.** Detect entities in the message and fetch their facts or graph neighborhood.
- **Agent-driven tools.** Give the agent a search tool (LangMem's `create_search_memory_tool`) and let it decide when to look things up.
- **Several searches at once.** Hindsight's recall runs semantic, BM25 keyword, graph and time-range search in parallel, fuses the results, reranks them with a cross-encoder and trims them to a `max_tokens` budget. It's strong on long histories (73.9 on BEAM 1M, vendor-reported) but returns more context than Mem0 by default, so set that budget on purpose.

Keep the memory block small. A few precise facts beat twenty loose matches, and every irrelevant fact is a chance for the model to get distracted.


## FAQ

### What is semantic memory in AI agents?

Semantic memory is the part of an agent's memory that stores facts and concepts without the event they came from: user preferences, profile details, domain knowledge and relationships between entities. The agent retrieves these facts into the prompt to ground and personalize its answers.

### Is semantic memory the same as semantic search?

No. Semantic memory is a psychology term for stored facts and knowledge. Semantic search is a retrieval technique that finds content by meaning, usually with embeddings. A semantic memory can be searched by keyword, by entity or by profile lookup, with no embeddings at all.

### Should agent semantic memory be a profile or a collection?

Use a profile (one structured document per user) when you need fast access to the current state of a known set of fields. Use a collection (many small fact records) when facts are open-ended and you want to recall them contextually. LangGraph's docs note collections tend to give higher recall but are harder to update.

### Which memory systems delete a changed fact, and which keep it?

LangMem (with enable_deletes) and CrewAI's Memory let an LLM delete or merge the old record on write. Mem0 has been ADD-only since April 2026: both facts stay, and retrieval is meant to rank the newer one. Graphiti marks the old edge invalid with a validity window. Hindsight keeps the old fact as evidence under a refined observation; since v1.2.0 its code, not the model, decides which fact is newer. A profile just overwrites the field.

### Which memory system handles changed facts best?

On LongMemEval's knowledge-update questions (78 questions, S setting), the Hindsight paper reports Hindsight at 94.9% with Gemini-3 and 92.3% with the open OSS-120B model, Supermemory at 87.2-89.7% and Zep at 83.3% with GPT-4o, and full context at 78.2% with GPT-4o. The paper is by Hindsight's authors, so treat it as vendor-run, and test on your own data.

