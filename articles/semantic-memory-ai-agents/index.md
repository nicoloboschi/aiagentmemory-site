# Semantic Memory in AI Agents: Facts, Profiles, Graphs

> What semantic memory is in AI agents: profile vs collection storage, triples and knowledge graphs, extracting facts with LangMem, and keeping facts current.

Source: https://aiagentmemory.org/articles/semantic-memory-ai-agents/ · Updated 2026-10-12 · aiagentmemory.org


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

