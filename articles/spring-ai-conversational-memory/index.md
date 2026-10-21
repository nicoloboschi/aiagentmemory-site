# Spring AI Chat Memory: ChatMemory, Advisors, JDBC

> How conversational memory works in Spring AI 2.0: ChatMemory, MessageWindowChatMemory, memory advisors, JDBC/Redis/Neo4j repositories, and 2.0 breaking changes.

Source: https://aiagentmemory.org/articles/spring-ai-conversational-memory/ · Updated 2026-10-20 · aiagentmemory.org


**Spring AI conversational memory** is handled by the `ChatMemory` interface and a memory **advisor** on the `ChatClient`. `MessageWindowChatMemory` keeps the last 20 messages per conversation by default. A `ChatMemoryRepository` stores them in memory, JDBC, Cassandra, Neo4j, MongoDB or Redis. `MessageChatMemoryAdvisor` adds that history to every prompt.

This page follows the [Spring AI 2.0.1 chat memory reference](https://docs.spring.io/spring-ai/reference/api/chat-memory.html). Spring AI 2.0.0 shipped in June 2026 and changed memory behavior in ways that break 1.x code; those changes are covered below. Spring AI is a Java framework, so the examples are Java.

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

### How do I add conversation memory in Spring AI?

Build a ChatMemory, usually MessageWindowChatMemory, and register MessageChatMemoryAdvisor on the ChatClient. On each call, pass the conversation ID with .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, id)). The advisor loads earlier messages into the prompt and saves the new exchange.

### What is the default window size of MessageWindowChatMemory?

20 messages. System messages are always kept, and the 2.0 reference says eviction removes whole turns, from a user message through the assistant reply and any tool messages, so the stored count can be lower than the limit.

### Was PromptChatMemoryAdvisor removed in Spring AI 2.0?

Yes. It was deprecated in 1.1.3 and removed in 2.0, so code that references it no longer compiles. The upgrade notes say to replace it with MessageChatMemoryAdvisor, which has the same builder API but sends history as chat messages instead of system-prompt text.

### Is Redis chat memory safe to use in Spring AI 2.0.0?

Upgrade to 2.0.1 first. In 2.0.0, two concurrent writes to one conversation could silently overwrite each other's messages, and clear() left messages behind in conversations longer than 10 messages. Spring AI 2.0.1 (August 2026) fixes both (PRs #6784 and #6783).

