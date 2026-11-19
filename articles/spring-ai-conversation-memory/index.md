# Spring AI Chat Memory: ChatMemory, Advisors, JDBC

> How conversational memory works in Spring AI 2.0: ChatMemory, MessageWindowChatMemory, advisors, JDBC/Redis/Neo4j repositories, 2.0 breaking changes, and the 2.1 move to session memory.

Source: https://aiagentmemory.org/articles/spring-ai-conversational-memory/ · Updated 2026-11-05 · aiagentmemory.org


**Spring AI conversational memory** is handled by the `ChatMemory` interface and a memory **advisor** on the `ChatClient`. `MessageWindowChatMemory` keeps the last 20 messages per conversation by default. A `ChatMemoryRepository` stores them in memory, JDBC, Cassandra, Neo4j, MongoDB or Redis. `MessageChatMemoryAdvisor` adds that history to every prompt.

This page follows the [Spring AI 2.0.x chat memory reference](https://docs.spring.io/spring-ai/reference/api/chat-memory.html). Spring AI 2.0.0 shipped in June 2026 and changed memory behavior in ways that break 1.x code; those changes are covered below. Spring AI is a Java framework, so the examples are Java.

**Coming in 2.1:** Spring AI 2.1.0-RC1 (23 October 2026, RC2 on 5 November) deprecates this whole `ChatMemory` API in favor of a new **session memory**, with removal planned for 3.0.0. Existing code still runs. See [what changes in Spring AI 2.1](#spring-ai-21-chatmemory-is-deprecated-for-session-memory) before you start new code on `ChatMemory`.

## What is conversational memory in Spring AI?

**Conversational memory in Spring AI is the set of recent messages a ChatClient sends back to the model so it keeps context across calls. ChatMemory decides which messages to keep, a ChatMemoryRepository stores them per conversation ID, and an advisor injects them into each prompt.**

The docs separate two ideas. **Chat memory** is what the model needs to keep context. **Chat history** is the full record of every message. `ChatMemory` is built for the first and, in the docs' words, "not the best fit for storing the chat history." For a complete audit record, the docs recommend Spring Data.

## Adding memory to a ChatClient

Spring Boot auto-configures a `ChatMemory` bean: an `InMemoryChatMemoryRepository` behind a `MessageWindowChatMemory`. Register an advisor and pass a conversation ID on each call:

```java
ChatMemory chatMemory = MessageWindowChatMemory.builder()
        .maxMessages(20)
        .build();

ChatClient chatClient = ChatClient.builder(chatModel)
        .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
        .build();

// Derive the ID on the server, per user and per conversation
String conversationId = currentUser.getId() + ":" + httpSession.getId();

String answer = chatClient.prompt()
        .user("My name is Dana. What's a good first Spring AI project?")
        .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
        .call()
        .content();
```

The conversation ID is mandatory in 2.0. Calls without `ChatMemory.CONVERSATION_ID` throw `IllegalArgumentException`; there's no default ID anymore. The docs recommend building it on the server from the authenticated user and session, and checking ownership before listing or deleting conversations.

### Advisors in Spring AI 2.0

| Advisor | Backed by | How history reaches the model |
|---|---|---|
| `MessageChatMemoryAdvisor` | `ChatMemory` | As a list of chat messages (recommended) |
| `VectorStoreChatMemoryAdvisor` | `VectorStore` | Retrieved by similarity, appended to the system message as text |
| `PromptChatMemoryAdvisor` | `ChatMemory` | Removed in 2.0 (deprecated in 1.1.3) |

`VectorStoreChatMemoryAdvisor` is the one closest to long-term memory: it retrieves relevant past messages rather than the most recent ones. Its template needs two placeholders, `instructions` (the original system message) and `long_term_memory` (the retrieved text).

## MessageWindowChatMemory: how the window works

`MessageWindowChatMemory` keeps a sliding window of up to `maxMessages` (default 20). System messages are always kept.

The 2.0 reference describes eviction by **whole turns** (the 1.1 docs only said older messages are removed). A turn starts at a `UserMessage` and includes the assistant reply, tool calls and tool responses up to the next `UserMessage`. So `maxMessages` is an upper bound, and the stored count may be lower. If `maxMessages` is smaller than one full turn, non-system messages can all be evicted until the next user message arrives. Keeping turns whole avoids sending a tool result without the call that produced it, which many model APIs reject.

A window of 20 messages is roughly 10 exchanges. That's plenty for a support chat and too little for anything that should remember a user next month. See [short-term memory in AI agents](/articles/short-term-memory-ai-agents/) for why a window is short-term by design.

## Chat memory repositories

| Repository | Starter artifact | Tool calls stored |
|---|---|---|
| `InMemoryChatMemoryRepository` | Auto-configured | n/a (in-process `ConcurrentHashMap`) |
| `JdbcChatMemoryRepository` | `spring-ai-starter-model-chat-memory-repository-jdbc` | No |
| `CassandraChatMemoryRepository` | `spring-ai-starter-model-chat-memory-repository-cassandra` | No |
| `Neo4jChatMemoryRepository` | `spring-ai-starter-model-chat-memory-repository-neo4j` | Yes, as nodes |
| `MongoChatMemoryRepository` | `spring-ai-starter-model-chat-memory-repository-mongodb` | No |
| `RedisChatMemoryRepository` | `spring-ai-starter-model-chat-memory-repository-redis` | No mention; needs Redis Stack 7.0+ |

Azure Cosmos DB support exists as an external module maintained by the Cosmos DB team.

### JDBC setup

Add the JDBC starter and Spring Boot wires a `JdbcChatMemoryRepository` for your `DataSource`:

```java
@Autowired
JdbcChatMemoryRepository chatMemoryRepository;

ChatMemory chatMemory = MessageWindowChatMemory.builder()
        .chatMemoryRepository(chatMemoryRepository)
        .maxMessages(20)
        .build();
```

Supported databases are PostgreSQL, MySQL/MariaDB, SQL Server, HSQLDB and Oracle; the dialect is auto-detected. Schema creation is controlled by `spring.ai.chat.memory.repository.jdbc.initialize-schema`: `embedded` (default, embedded databases only), `always` or `never` (use with Flyway or Liquibase). Data goes in the `SPRING_AI_CHAT_MEMORY` table.

One limit matters for agents. The JDBC repository **silently drops** assistant messages with tool calls and tool response messages on save. If you need those, the docs point to the community [Spring AI Session](https://spring-ai-community.github.io/spring-ai-session/latest/) project and its JDBC session store.

### Redis and Cassandra TTLs

Redis and Cassandra support expiry, which is useful when chat data has a retention policy. Redis takes `spring.ai.chat.memory.repository.redis.time-to-live` (for example `24h` or `30d`). Cassandra takes `spring.ai.chat.memory.cassandra.time-to-live`, and the docs suggest a long TTL such as three years for audit use. MongoDB has `spring.ai.chat.memory.repository.mongo.ttl` in seconds.

### Redis fixes in 2.0.1: upgrade before you scale out

If you run `RedisChatMemoryRepository` on 2.0.0, upgrade. The [2.0.1 release](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.1) (21 August 2026) fixes three Redis memory bugs that lose or leak data without any error:

- **Concurrent writes overwrote messages.** `add()` reserved one timestamp slot in Redis per call and numbered the rest of the batch locally, so two calls for the same conversation could pick the same slots and one silently overwrote the other. 2.0.1 reserves a block sized to the batch in one atomic Lua script ([#6784](https://github.com/spring-projects/spring-ai/pull/6784)). This bites as soon as two app instances, or two requests, write to one conversation.
- **`clear()` left messages behind.** The search behind it returned 10 results by default, so a conversation with more than 10 messages was only partly deleted. 2.0.1 deletes in a loop until nothing matches ([#6783](https://github.com/spring-projects/spring-ai/pull/6783)). If you clear conversations for privacy or retention, older data on 2.0.0 may still be there.
- **Metadata values are now escaped** in Redis memory queries ([#6765](https://github.com/spring-projects/spring-ai/issues/6765)).

The same release renames the Redis auto-configuration artifact to match the others: `spring-ai-autoconfigure-model-chat-memory-redis` becomes `spring-ai-autoconfigure-model-chat-memory-repository-redis` ([#6416](https://github.com/spring-projects/spring-ai/issues/6416)). The starter name doesn't change, so this only matters if you depend on the auto-configuration module directly.

### Neo4j index fix in 2.0.2

Before [2.0.2](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.2) (23 October 2026), `Neo4jChatMemoryRepository` created its two indexes on properties it never writes (`Session.conversationId` and `Message.index`, while the queries use `Session.id` and `Message.idx`). The indexes stayed empty, so every load, save and delete scanned all `Session` nodes, and got slower as conversations piled up ([#6967](https://github.com/spring-projects/spring-ai/pull/6967)). 2.0.2 creates `session_id_index` and `message_idx_index` on the right properties at startup. The old empty indexes are harmless; the upgrade notes say to drop them with `DROP INDEX session_conversation_id_index IF EXISTS` and `DROP INDEX message_index_index IF EXISTS`.

## Spring AI 2.1: ChatMemory is deprecated for session memory

[Spring AI 2.1.0-RC1](https://github.com/spring-projects/spring-ai/releases/tag/v2.1.0-RC1) adds **session memory** ([#7161](https://github.com/spring-projects/spring-ai/pull/7161)) and deprecates the chat memory API for removal in 3.0.0 ([#7186](https://github.com/spring-projects/spring-ai/pull/7186)). The deprecation itself changes nothing at runtime. [2.1.0-RC2](https://github.com/spring-projects/spring-ai/releases/tag/v2.1.0-RC2) followed on 5 November 2026; its only session change is the safe batch write described [below](#writing-to-a-session-from-several-places-21-rc2). 2.1.0 is still a release candidate, not GA.

A session stores the conversation as an append-only log of events, and the prompt is built from a view of that log. When the history grows, **compaction** moves older turns out of the view (optionally replacing them with an LLM summary) but never deletes them from the log, and never splits a turn, so the model never sees a tool call without its result. That fixes the agent problem above, where the JDBC chat memory repository drops tool messages.

| What | Status in 2.1.0-RC1 and RC2 |
|---|---|
| `ChatMemory`, `MessageWindowChatMemory`, `ChatMemoryRepository`, `InMemoryChatMemoryRepository` | Deprecated, removal in 3.0.0 |
| `MessageChatMemoryAdvisor` | Deprecated; use `SessionAdvisors.builder(sessionService)` |
| JDBC, Cassandra, Neo4j, MongoDB, Redis chat memory repositories | Deprecated |
| `VectorStoreChatMemoryAdvisor` | **Not** deprecated (no session version of semantic recall yet) |
| Session JDBC store | PostgreSQL, MySQL, MariaDB, H2 (`spring-ai-starter-model-session-jdbc`) |
| Session stores for Cassandra, Neo4j, MongoDB, Redis; dialects for HSQLDB, Oracle, SQL Server, SQLite | Not yet; planned before 3.0.0 |
| Conversation key | Moves to `MemoryAdvisor.CONVERSATION_ID`, same value as `ChatMemory.CONVERSATION_ID` |

The [session memory reference](https://github.com/spring-projects/spring-ai/blob/v2.1.0-RC1/spring-ai-docs/src/main/antora/modules/ROOT/pages/api/session.adoc) gives this setup:

```java
@Bean
ChatClient chatClient(ChatModel chatModel, SessionService sessionService) {
    return ChatClient.builder(chatModel)
        .defaultAdvisors(SessionAdvisors.builder(sessionService)
            .compaction(new TurnCountTrigger(20),
                WindowCompactionStrategy.builder().lastTurns(10).build())
            .build())
        .build();
}

String answer = chatClient.prompt()
    .user("Hello!")
    .advisors(SessionAdvisors.session("session-abc"))
    .call()
    .content();
```

Compaction can trigger on turn count (`TurnCountTrigger`) or estimated tokens (`TokenCountTrigger`), and the window can keep the last N events (20 by default), the last N turns, or what fits a token budget.

What to watch when you move:

- **No data migration.** Sessions use their own tables and don't read `SPRING_AI_CHAT_MEMORY`, so old conversations don't carry over on their own.
- **Fewer databases, for now.** If you store chat memory in Redis, Cassandra, Neo4j, MongoDB, Oracle, SQL Server, HSQLDB or SQLite, there is no session store for it yet. Stay on chat memory until one ships.
- **Still one conversation.** A session is still scoped to one session ID. It doesn't carry facts about a user into a new conversation; that still needs a separate memory service (see below).

### Writing to a session from several places (2.1-RC2)

In RC1, only compaction could write to the session log safely when another writer was active. Any other code that read the log and wrote events based on it (your own summarizer, a note-taker, a second request on the same session) could write on top of a log that had already moved. RC2 puts the same safe write on `SessionService` ([#7314](https://github.com/spring-projects/spring-ai/pull/7314)):

| Method | What it does |
|---|---|
| `getEventVersion(sessionId)` | Returns a counter that goes up on every new event and once per successful batch; `0` for an empty or unknown session |
| `appendEvents(sessionId, events, expectedVersion)` | Writes all the events in one step only if the counter still equals `expectedVersion`; otherwise writes nothing and returns `false` |

The pattern from the [session concepts docs](https://github.com/spring-projects/spring-ai/blob/v2.1.0-RC2/spring-ai-docs/src/main/antora/modules/ROOT/pages/api/session/concepts.adoc): read the version, read the events your write depends on, build the new events, call `appendEvents`, and on `false` read again and rebuild. A system message in the batch throws, as with `appendEvent`, unless `allowSystemMessages(true)` is set, and that check runs before the version check.

## Upgrading chat memory from Spring AI 1.x to 2.0

The [2.0 upgrade notes](https://docs.spring.io/spring-ai/reference/upgrade-notes.html) list four memory changes:

1. **`PromptChatMemoryAdvisor` is removed.** Replace it with `MessageChatMemoryAdvisor`; the builder API is the same.
2. **Conversation ID is required.** There's no default conversation ID. Pass `ChatMemory.CONVERSATION_ID` on every call.
3. **JDBC table gains a `sequence_id` column.** Messages are now ordered by it, not by timestamp, because timestamp precision varies across databases. Tables created by 1.x need the column added and backfilled before they work; the notes include a PostgreSQL migration script.
4. **Messages carry a timestamp in metadata.** Read it with `JdbcChatMemoryRepository.CONVERSATION_TS`. Because metadata differs, a retrieved message no longer `equals()` an otherwise identical new one.

## When you need more than chat memory

Spring AI's chat memory is short-term: a bounded window of recent messages per conversation. It doesn't extract facts, merge duplicates, or carry a user's preferences into a new conversation ID. `VectorStoreChatMemoryAdvisor` gets you similarity search over old messages, which helps, but it's still retrieving raw messages.

For assistants that should remember users across conversations, teams usually add a separate memory service called over HTTP or exposed as a tool. [Hindsight](https://github.com/vectorize-io/hindsight) is one that fits this pattern: an MIT-licensed server with a REST API and an MCP server, so a Spring service can call it like any other HTTP backend. It extracts facts with their time and entities, consolidates them into observations, and searches by meaning, keyword, entity graph and time range at once, which is where it's strongest on long histories (64.1 on BEAM 10M, vendor-reported). Two limits for a Java team: there's no Java client (its clients are Python, Node/TypeScript, Go and a CLI), and every write calls an LLM, so it costs more per message than a JDBC window. The [AI agent memory guide](/articles/ai-agent-memory-explained/) explains the options, and [how to add memory to a chatbot](/articles/how-to-add-memory-to-chatbot/) walks through the patterns in Python that translate directly to a Spring service.


## FAQ

### Is ChatMemory deprecated in Spring AI?

Yes, from Spring AI 2.1.0 (RC1 released 23 October 2026). ChatMemory, MessageWindowChatMemory, MessageChatMemoryAdvisor and the JDBC, Cassandra, Neo4j, MongoDB and Redis repositories are deprecated for removal in 3.0.0, replaced by session memory (SessionAdvisors, SessionService, spring-ai-starter-model-session-jdbc). Code keeps working. VectorStoreChatMemoryAdvisor is not deprecated, and sessions don't read old SPRING_AI_CHAT_MEMORY data.

### How do I add conversation memory in Spring AI?

Build a ChatMemory, usually MessageWindowChatMemory, and register MessageChatMemoryAdvisor on the ChatClient. On each call, pass the conversation ID with .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, id)). The advisor loads earlier messages into the prompt and saves the new exchange.

### What is the default window size of MessageWindowChatMemory?

20 messages. System messages are always kept, and the 2.0 reference says eviction removes whole turns, from a user message through the assistant reply and any tool messages, so the stored count can be lower than the limit.

### Was PromptChatMemoryAdvisor removed in Spring AI 2.0?

Yes. It was deprecated in 1.1.3 and removed in 2.0, so code that references it no longer compiles. The upgrade notes say to replace it with MessageChatMemoryAdvisor, which has the same builder API but sends history as chat messages instead of system-prompt text.

### Is Redis chat memory safe to use in Spring AI 2.0.0?

Upgrade to 2.0.1 first. In 2.0.0, two concurrent writes to one conversation could silently overwrite each other's messages, and clear() left messages behind in conversations longer than 10 messages. Spring AI 2.0.1 (August 2026) fixes both (PRs #6784 and #6783).

