# AI Memory Architecture

## 1. What This Document Is

This document explains how the Mike legal AI platform remembers things. Memory is one of the most important parts of the system because it determines how well the AI can help you. Without memory, the AI would be like a person with amnesia - it would forget everything the moment you stop talking. With memory, the AI remembers your preferences, your past conversations, your legal documents, and the patterns it has learned from working with you.

This document covers every aspect of the memory system: what types of memory exist, how they are stored, how they are used, how they expire, and how they are protected. Whether you are a developer working on the codebase, a product manager understanding capabilities, or a security reviewer evaluating data handling, this document gives you the full picture.

The memory system is built across several key files in the codebase:

| File | Lines | Purpose |
|------|-------|---------|
| `backend/src/lib/orchestrator/memory/memoryLayer.ts` | 250 | Central memory API combining session and persistent storage |
| `backend/src/lib/orchestrator/memory/sessionMemory.ts` | 24 | In-memory per-chat storage for active conversations |
| `backend/src/lib/orchestrator/memory/persistentMemory.ts` | 5 | Delegates to MemoryLayer for database-backed storage |
| `backend/src/db/schema.ts` | 2525 | Database tables including `user_memories`, `conversation_context`, `learning_events` |
| `backend/src/prompts/promptBuilder.ts` | 547 | Injects memory context into LLM prompts |
| `backend/src/lib/chatOrchestrator.ts` | 4145 | Manages chat context and memory during conversations |
| `backend/src/lib/agentic/contextBuilder.ts` | 341 | Builds structured context blocks for agent execution |
| `backend/src/lib/orchestrator/index.ts` | 488 | Orchestrator pipeline that coordinates memory usage |
| `backend/src/lib/orchestrator/learning/learningSystem.ts` | 110 | Extracts learnings from agent runs and user feedback |

---

## 2. Why Memory Matters

Memory is the difference between a useful AI assistant and a useless one. Here is why it matters in plain terms:

### Without Memory
- The AI forgets everything between messages. You tell it "I prefer concise answers" and the next message it gives you a five-paragraph essay.
- The AI cannot reference past conversations. You discussed a contract last week and this week it has no idea what you are talking about.
- The AI cannot learn from experience. It makes the same mistakes over and over.
- The AI cannot maintain context. You are working on a complex legal matter and it loses track of the key parties, jurisdictions, and deadlines.

### With Memory
- The AI remembers your preferences. If you prefer concise answers, it gives you concise answers every time.
- The AI references past conversations. It knows you discussed that NDA last Tuesday and can pick up where you left off.
- The AI learns from experience. It remembers that research tasks in California contract law work best with a particular approach.
- The AI maintains context. It tracks the key parties, jurisdictions, deadlines, and document relationships across your entire matter.

### The Business Impact
Memory directly impacts the quality of legal work product. A memoryless AI requires you to re-explain everything every time. A memoryful AI builds on previous work, learns your firm's conventions, and produces increasingly relevant output over time. This translates to faster turnaround, fewer errors, and more consistent output across your team.

---

## 3. Memory Types Overview

The Mike platform uses six distinct memory types, each serving a different purpose:

| Memory Type | Purpose | Lifespan | Storage | Speed | Scope |
|-------------|---------|----------|---------|-------|-------|
| **Session Memory** | Current conversation state | Conversation lifetime | In-memory (RAM) | Very fast | Per chat |
| **Persistent Memory** | Cross-session learnings | Permanent (with optional expiry) | PostgreSQL `user_memories` table | Moderate | Per user |
| **User Memory** | User preferences and settings | Permanent | PostgreSQL `user_memories` table | Moderate | Per user |
| **Document Memory** | Document context and relationships | Varies by document lifecycle | PostgreSQL + in-memory | Moderate | Per document/project |
| **Agent Memory** | Agent learning and optimization | Permanent (with decay) | PostgreSQL `learning_events` + `user_memories` | Moderate | Per user |
| **Conversation Context** | Chat-specific structured data | Conversation lifetime (persisted) | PostgreSQL `conversation_context` table | Moderate | Per chat |

### How They Relate
Session memory and conversation context are closely related - session memory is the fast in-memory store while conversation context is the database-backed version that survives server restarts. Persistent memory and user memory both use the `user_memories` table but differ in their `type` field - user memories have type "preference" while persistent memories can be "legal_pattern", "preference", or other types. Document memory draws from the document index and RAG (retrieval-augmented generation) system. Agent memory combines the learning events table with user memories to track what the system has learned about how to best help each user.

---

## 4. Session Memory

### What It Is

Session memory is the fastest type of memory in the system. It lives entirely in the server's RAM (memory) and is used only during an active conversation. Think of it like a scratchpad you use during a phone call - you jot down notes while talking, but once you hang up, the scratchpad is gone.

Session memory is implemented as a simple key-value store where each chat has its own isolated namespace. Two different conversations cannot see each other's session memory.

### How It Works

Session memory is managed by the `SessionMemory` class in `backend/src/lib/orchestrator/memory/sessionMemory.ts`. The implementation is straightforward:

```typescript
// From sessionMemory.ts
export class SessionMemory {
  private sessions = new Map<string, Map<string, unknown>>();

  set(chatId: string, key: string, value: unknown): void {
    if (!this.sessions.has(chatId)) this.sessions.set(chatId, new Map());
    this.sessions.get(chatId)!.set(key, value);
  }

  get(chatId: string, key: string): unknown | null {
    return this.sessions.get(chatId)?.get(key) ?? null;
  }

  getAll(chatId: string): Record<string, unknown> {
    const session = this.sessions.get(chatId);
    return session ? Object.fromEntries(session.entries()) : {};
  }

  clear(chatId: string): void {
    this.sessions.delete(chatId);
  }
}
```

The data structure is a nested Map: the outer Map is keyed by `chatId` (a unique conversation identifier), and the inner Map holds arbitrary key-value pairs for that conversation. This means each conversation gets its own isolated memory space.

### What It Stores

Session memory stores temporary data that is only relevant to the current conversation:

- **Tool call results**: Intermediate results from tools the AI has called during the conversation
- **Context flags**: Boolean flags indicating what has been loaded or processed
- **Working data**: Temporary calculations or transformations
- **State transitions**: Whether certain processes have been initiated or completed

### Lifecycle

1. **Creation**: When a new message arrives and no session exists for the `chatId`, a new Map is created automatically on the first `set()` call.
2. **Updates**: Each `set()` call either adds a new key-value pair or updates an existing one for the given chat.
3. **Reads**: The `get()` and `getAll()` methods retrieve stored values without modifying them.
4. **Destruction**: When the conversation ends or the server restarts, all session memory is lost. The `clear()` method can also explicitly remove a chat's session.

### Configuration

Session memory has no configurable parameters. It is a simple in-memory store with no size limits, no expiry, and no persistence. The only constraint is available server RAM.

### When It Is Used

The `MemoryLayer` class wraps `SessionMemory` and exposes it through the `setSession()`, `getSession()`, `getAllSession()`, and `clearSession()` methods. The orchestrator and chat orchestrator use these methods to store and retrieve session-scoped data during active processing.

---

## 5. Persistent Memory

### What It Is

Persistent memory is memory that survives across sessions. Unlike session memory which is lost when a conversation ends, persistent memory is stored in the PostgreSQL database and can be retrieved in any future conversation. Think of it like a notebook you keep - you can write in it during one conversation and read from it in a later conversation.

Persistent memory is implemented through the `MemoryLayer` class and stored in the `user_memories` database table.

### How It Works

The persistent memory system uses a key-value model where each piece of memory has:
- A **user ID** (who it belongs to)
- A **key** (what it is about)
- A **value** (what was learned)
- A **type** (what category it falls into)
- A **confidence score** (how certain the system is about this memory)
- An **expiry date** (when it should be forgotten, if ever)

### Data Structure

The `user_memories` table in `backend/src/db/schema.ts` defines the persistent memory structure:

```typescript
export const userMemories = pgTable("user_memories", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").notNull()
    .references(() => users.id, { onDelete: "cascade" }),
  type: varchar("type", { length: 80 }).notNull(),
  key: text("key").notNull(),
  value: jsonb("value").$type<unknown>().notNull(),
  confidence: real("confidence").default(1.0).notNull(),
  sourceSessionId: uuid("source_session_id"),
  accessCount: integer("access_count").default(0).notNull(),
  lastAccessedAt: timestamp("last_accessed_at"),
  expiresAt: timestamp("expires_at"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
}, (table) => ({
  userIdTypeIdx: index("user_memories_user_id_type_idx")
    .on(table.userId, table.type),
  userIdKeyIdx: uniqueIndex("user_memories_user_id_key_idx")
    .on(table.userId, table.key),
}));
```

Key fields:
- **`type`**: Categorizes the memory. Common types include `"preference"` and `"legal_pattern"`.
- **`key`**: The identifier for this memory. For preferences, keys follow the pattern `"preference:response_style"` or `"preference:jurisdiction"`. For legal patterns, keys follow `"legal_pattern:domain:contract_law"`.
- **`value`**: JSON blob containing the actual memory content. Can be any valid JSON value.
- **`confidence`**: A number between 0 and 1 indicating how confident the system is. Higher confidence means the system is more certain this memory is correct.
- **`sourceSessionId`**: Links the memory back to the conversation where it was learned.
- **`accessCount`**: Tracks how many times this memory has been read, useful for identifying popular or important memories.
- **`expiresAt`**: Optional timestamp after which the memory should be ignored.

### Lifecycle

1. **Creation**: Persistent memories are created by the `setLongTerm()` method when the learning system detects a pattern. For example, when a user asks for "a brief summary", the system creates a preference memory with key `"preference:response_style"` and value `"concise"`.

2. **Updates**: The `setLongTerm()` method uses `onConflictDoUpdate` - if a memory with the same user ID and key already exists, it is updated rather than duplicated. This means the system tracks the latest version of each preference.

3. **Reads**: The `getLongTerm()` method retrieves a memory by user ID and key. It also increments the `accessCount` and updates `lastAccessedAt` to track usage patterns. Expired memories are automatically filtered out using the condition `or(isNull(userMemories.expiresAt), gt(userMemories.expiresAt, new Date()))`.

4. **Retrieval of All Preferences**: The `getUserPreferences()` method retrieves all memories of type `"preference"` for a user, returning them as a key-value map.

5. **Retrieval of Legal Patterns**: The `getLegalPatterns()` method retrieves memories of type `"legal_pattern"`, ordered by confidence (highest first), limited to 10 results.

### Storage

Persistent memory is stored in PostgreSQL with two important indexes:
- `user_memories_user_id_type_idx`: Speeds up queries that filter by user ID and type
- `user_memories_user_id_key_idx`: A unique index ensuring each user can only have one memory per key

The `onDelete: "cascade"` on the user ID foreign key means that when a user is deleted, all their memories are automatically deleted too.

---

## 6. User Memory

### What It Is

User memory is a subset of persistent memory that specifically stores user preferences and settings. It is the system's way of learning how each individual user likes to work. Think of it like a profile that the system builds about you over time.

### What It Stores

User memory captures several categories of preferences:

#### Response Style Preferences
- Whether the user prefers concise or detailed answers
- Whether the user prefers formal or casual language
- Whether the user prefers bullet points or prose

#### Legal Domain Preferences
- Which legal areas the user works in most frequently
- Which jurisdictions the user operates in
- Which types of documents the user typically needs

#### Tool Usage Patterns
- Which tools the user uses most frequently
- Which tools the user never uses
- Preferred workflows for common tasks

#### Document Preferences
- Preferred document formats
- Preferred clause libraries
- Preferred template styles

### How User Preferences Are Learned

User preferences are learned in three ways:

#### Explicit Settings
Users can directly set preferences through the UI. These are stored immediately as preference memories with high confidence.

#### Implicit Learning
The `LearningSystem` class in `backend/src/lib/orchestrator/learning/learningSystem.ts` automatically detects patterns from user behavior:

```typescript
// Detecting concise response preference
if (/\b(brief|quick|short|summary|tldr)\b/i.test(params.userQuery)) {
  await memoryLayer.setLongTerm({
    userId: params.userId,
    key: "preference:response_style",
    value: "concise",
    type: "preference",
    confidence: 0.8,
    sourceSessionId: params.chatId,
  });
}

// Detecting jurisdiction preference
const jurisdictionMatch = params.userQuery.match(
  /\b(california|new york|texas|uk|eu|federal|delaware|england|india)\b/i
);
if (jurisdictionMatch) {
  await memoryLayer.setLongTerm({
    userId: params.userId,
    key: "preference:jurisdiction",
    value: jurisdictionMatch[1].toLowerCase(),
    type: "preference",
    confidence: 0.7,
    sourceSessionId: params.chatId,
  });
}
```

#### Feedback-Based Learning
When users give positive or negative feedback on responses, the system adjusts the confidence of related memories:

```typescript
async recordFeedback(params: {
  userId: string;
  chatId: string;
  messageId: string;
  feedback: "positive" | "negative";
  messageContent: string;
  userQuery?: string;
}): Promise<void> {
  // Record the feedback event
  await memoryLayer.recordLearningEvent({...});

  // If positive, increase confidence of memories from this session
  if (params.feedback === "positive") {
    await db.update(userMemories)
      .set({ confidence: sql`LEAST(${userMemories.confidence} + 0.05, 1.0)` })
      .where(and(
        eq(userMemories.userId, params.userId),
        eq(userMemories.sourceSessionId, params.chatId)
      ));
  }
}
```

### How User Preferences Are Used

User preferences are loaded into the prompt context via the `buildMemoryContext()` method:

```typescript
async buildMemoryContext(userId: string, chatId: string): Promise<string> {
  const [preferences, legalPatterns, context] = await Promise.all([
    this.getUserPreferences(userId),
    this.getLegalPatterns(userId),
    this.getConversationContext(chatId),
  ]);

  const lines = ["## User Context & Memory"];
  const preferenceEntries = Object.entries(preferences);
  if (preferenceEntries.length) {
    lines.push(`Preferences: ${preferenceEntries
      .map(([key, value]) => `${key}=${JSON.stringify(value)}`)
      .join("; ")}`);
  }
  if (legalPatterns.length) {
    lines.push(`Legal patterns: ${legalPatterns
      .map((pattern) => pattern.key)
      .join(", ")}`);
  }
  // ... conversation context ...

  const text = lines.join("\n");
  return text.split(/\s+/).slice(0, 400).join(" ");
}
```

The `400-word limit` ensures that user preferences do not consume too much of the token budget allocated to the LLM.

---

## 7. Document Memory

### What It Is

Document memory is the system's understanding of the documents it is working with. Unlike user memory which is about the user, document memory is about the content, structure, and relationships of legal documents. Think of it like a librarian who knows not just where every book is, but also how they relate to each other.

### What It Stores

Document memory captures several types of information:

#### Document Summaries
Brief descriptions of what each document contains, enabling the AI to quickly determine relevance without reading entire documents.

#### Key Terms and Concepts
Important legal terms, party names, dates, monetary amounts, and other critical data points extracted from documents.

#### Citations
References between documents, cross-references to clauses, and links to external legal authorities.

#### Relationships
How documents relate to each other - which documents belong to the same matter, which documents reference each other, which documents are versions of the same document.

#### Version History
Tracking how documents have changed over time, what edits were made, and who made them.

### How Document Memory Is Built

Document memory is assembled from multiple sources:

#### The Document Index (`DocIndex`)
The `DocIndex` type represents the complete set of documents available in the current scope. It is built by the `buildDocContext()`, `buildProjectDocContext()`, and `buildWorkspaceDocContext()` functions in `chatTools.ts`.

Each entry in the index contains:
- `document_id`: The unique identifier
- `filename`: The human-readable name
- `context_role`: Whether it is a "primary" or "supporting" document
- `source_type`: Whether it is a "document" or "drive_file"
- `lifecycle_status`: The current status (DRAFT, IN_REVIEW, PENDING_APPROVAL, APPROVED, FINALIZED)
- `size_bytes`: The file size

#### RAG (Retrieval-Augmented Generation)
The RAG system indexes document content and enables semantic search across all documents. When the AI needs to answer a question about documents, it uses RAG to find the most relevant passages.

#### Eager Loading
For small documents (under 6000 characters), the system eagerly loads their full content into the prompt context:

```typescript
const EAGER_CHARS_PER_DOC = 6000;
const EAGER_CHARS_TOTAL = 18000;
```

This means the AI has immediate access to the content of small documents without needing to make additional tool calls.

### How Document Memory Is Used

Document memory is injected into the system prompt through the `buildUnifiedContextBlock()` function in `promptBuilder.ts`:

```
CONTEXT:
Scope: project - project abc123
RAG: 42 indexed source(s), ready to search

PROJECT CONTEXT SIZE POLICY:
The list below is a manifest, not full text.
Primary documents: 5
Supporting documents: 12

DOCUMENTS IN SCOPE:
- doc-001: contracts/nda-v1.docx [primary, document, DRAFT, 45000 bytes]
- doc-002: contracts/nda-v2.docx [primary, document, IN_REVIEW, 48000 bytes]
- doc-003: research/precedent-analysis.docx [supporting, document, FINALIZED, 23000 bytes]
```

This tells the AI exactly what documents are available, their status, and their relevance, without overwhelming the token budget with full document text.

### Document Memory and the Context Builder

The `contextBuilder.ts` file provides structured context blocks for more complex use cases:

- **`MatterContext`**: Full matter context including documents, knowledge items, accepted answers, lists, and monitors
- **`JudgmentContext`**: Context for judgment analysis including sources and metadata
- **`WorkspaceGraphContext`**: Relationship graph between workspace documents

These structured blocks enable the AI to reason about complex document relationships and matter-level context.

---

## 8. Agent Memory

### What It Is

Agent memory is what the system learns about how to best execute tasks using its various AI agents. Each agent (research, drafting, analysis, etc.) has its own set of experiences, and agent memory captures what works, what fails, and what to try differently next time. Think of it like a team of specialists who keep notes about their best practices.

### What It Stores

Agent memory captures several categories of learnings:

#### Successful Tool Combinations
When a particular sequence of tool calls produces a good result, that sequence is recorded. For example, if calling `search_sources` then `read_document` then `generate_docx` consistently produces good contract drafts, that pattern is remembered.

#### Failed Approaches
When an approach fails, it is recorded so the system can avoid it in the future. This includes:
- Tool calls that returned errors
- Agent combinations that produced low-quality output
- Approaches that were rejected by the user

#### Timing Information
How long different tasks take, helping the system make better decisions about parallelization and resource allocation.

#### Quality Scores
Metrics about the quality of agent output, including validation results, user feedback scores, and self-critique assessments.

### How Agent Memory Is Captured

Agent memory is captured through the learning system:

```typescript
// From learningSystem.ts
async extractFromRun(params: {
  userId: string;
  chatId: string;
  userQuery: string;
  finalResponse: string;
  agentTypes: AgentType[];
  legalDomain: string;
  coordinatorResult: CoordinatorResult;
}): Promise<void> {
  // Learn from successful research tasks
  if (params.agentTypes.includes("research") && params.coordinatorResult.success) {
    await memoryLayer.setLongTerm({
      userId: params.userId,
      key: `legal_pattern:domain:${params.legalDomain}`,
      value: {
        domain: params.legalDomain,
        lastUsed: new Date().toISOString(),
        useCount: 1,
      },
      type: "legal_pattern",
      confidence: 0.9,
      sourceSessionId: params.chatId,
    });
  }

  // Record task history for the conversation
  await memoryLayer.saveConversationContext({
    chatId: params.chatId,
    userId: params.userId,
    updates: {
      taskHistory: params.coordinatorResult.completedTasks.map((task) => ({
        agentType: task.agentType,
        taskId: task.taskId,
        success: task.success,
        timestamp: new Date().toISOString(),
      })),
    },
  });
}
```

### How Agent Memory Improves Over Time

Agent memory improves through several mechanisms:

#### Reinforcement from Success
When a user gives positive feedback after an agent run, the confidence of all memories created during that session is increased by 5%:

```typescript
if (params.feedback === "positive") {
  await db.update(userMemories)
    .set({ confidence: sql`LEAST(${userMemories.confidence} + 0.05, 1.0)` })
    .where(and(
      eq(userMemories.userId, params.userId),
      eq(userMemories.sourceSessionId, params.chatId)
    ));
}
```

#### Learning from Failure
Failed tasks are recorded in the conversation context's `taskHistory`, providing data for future task planning to avoid similar failures.

#### Pattern Recognition
The system identifies patterns across multiple runs - for example, noticing that a particular user always needs California contract law analysis, and proactively preparing the right tools and context.

#### Strategy Optimization
Over time, the system builds a library of legal patterns ranked by confidence, enabling it to choose the most effective approach for each type of task.

### Learning Events Table

The `learning_events` table captures detailed information about what happened during each agent run:

```typescript
export const learningEvents = pgTable("learning_events", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").notNull()
    .references(() => users.id, { onDelete: "cascade" }),
  chatId: uuid("chat_id").notNull()
    .references(() => chats.id, { onDelete: "cascade" }),
  messageId: uuid("message_id"),
  eventType: varchar("event_type", { length: 80 }).notNull(),
  messageContent: text("message_content"),
  userQuery: text("user_query"),
  context: jsonb("context").$type<Record<string, unknown>>()
    .default({}).notNull(),
  processed: boolean("processed").default(false).notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

The `processed` flag enables batch processing of learning events, allowing the system to analyze patterns asynchronously without slowing down the main conversation flow.

---

## 9. Conversation Context

### What It Is

Conversation context is the structured data that accompanies each conversation. It is the bridge between session memory (fast, in-memory) and persistent memory (permanent, in-database). Think of it like a case file that travels with the conversation, accumulating notes, references, and history.

### How It Works

Conversation context is stored in the `conversation_context` table and managed by the `MemoryLayer` class:

```typescript
export const conversationContext = pgTable("conversation_context", {
  id: uuid("id").primaryKey().defaultRandom(),
  chatId: uuid("chat_id").notNull().unique()
    .references(() => chats.id, { onDelete: "cascade" }),
  userId: uuid("user_id").notNull()
    .references(() => users.id, { onDelete: "cascade" }),
  shortTermMemory: jsonb("short_term_memory")
    .$type<Record<string, unknown>>().default({}).notNull(),
  legalEntities: jsonb("legal_entities")
    .$type<unknown[]>().default([]).notNull(),
  documentReferences: jsonb("document_references")
    .$type<unknown[]>().default([]).notNull(),
  taskHistory: jsonb("task_history")
    .$type<unknown[]>().default([]).notNull(),
  activeJurisdiction: text("active_jurisdiction"),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

### Components

#### Short-Term Memory
A JSON object storing arbitrary key-value pairs relevant to the current conversation. This is the structured equivalent of session memory but persisted to the database.

#### Legal Entities
An array tracking all legal entities (companies, people, organizations) mentioned in the conversation. This enables the AI to maintain a consistent understanding of who is involved in the matter.

#### Document References
An array tracking all documents that have been referenced or discussed in the conversation. This provides a running list of the documents the AI knows about.

#### Task History
An array recording all tasks that have been executed during the conversation, including which agent type was used, whether it succeeded, and when it ran.

#### Active Jurisdiction
The current legal jurisdiction being discussed. This is updated when the conversation shifts to a different jurisdiction.

### Lifecycle

1. **Creation**: Conversation context is created when the first message in a conversation is processed.

2. **Updates**: Each message can trigger updates to the conversation context through the `saveConversationContext()` method. The method merges new data with existing data:

```typescript
async saveConversationContext(params: {
  chatId: string;
  userId: string;
  updates: Partial<{
    shortTermMemory: Record<string, unknown>;
    legalEntities: unknown[];
    documentReferences: unknown[];
    taskHistory: unknown[];
    activeJurisdiction: string;
  }>;
}): Promise<void> {
  const existing = await this.getConversationContext(params.chatId);
  const merged: ConversationContextValue = {
    shortTermMemory: {
      ...(existing?.shortTermMemory ?? {}),
      ...(params.updates.shortTermMemory ?? {}),
    },
    legalEntities: params.updates.legalEntities
      ?? existing?.legalEntities ?? [],
    documentReferences: params.updates.documentReferences
      ?? existing?.documentReferences ?? [],
    taskHistory: [
      ...(existing?.taskHistory ?? []),
      ...(params.updates.taskHistory ?? []),
    ],
    activeJurisdiction: params.updates.activeJurisdiction
      ?? existing?.activeJurisdiction ?? null,
  };
  // ... upsert to database ...
}
```

Note that `taskHistory` is append-only - new tasks are added to the existing list. Other fields are replaced wholesale when updated.

3. **Reads**: The `getConversationContext()` method retrieves the full context for a conversation, used by the prompt builder and the memory context builder.

4. **Destruction**: Conversation context is deleted when the chat is deleted, enforced by the `onDelete: "cascade"` foreign key constraint.

---

## 10. Memory Injection into Prompts

### What It Is

Memory injection is the process of adding memory context to the prompts sent to the LLM. This is how the AI "remembers" things - the relevant memory is included in the prompt so the AI can reference it. Think of it like giving someone a briefing document before they walk into a meeting.

### How It Works

Memory is injected at multiple levels:

#### System Prompt Injection
The system prompt is built by `buildBaseSystemPromptExtra()` in `promptBuilder.ts`. This includes:

1. **Scope context**: Whether the conversation is in a project, workspace, or personal scope
2. **Document manifest**: The list of documents available in the current scope
3. **Template list**: Available templates for document generation
4. **Workflow state**: Active workflows and their status
5. **Interview state**: Template interview progress
6. **Permissions**: What the assistant is allowed to read and write

#### User Memory Injection
User preferences are injected through the `buildMemoryContext()` method:

```typescript
async buildMemoryContext(userId: string, chatId: string): Promise<string> {
  const [preferences, legalPatterns, context] = await Promise.all([
    this.getUserPreferences(userId),
    this.getLegalPatterns(userId),
    this.getConversationContext(chatId),
  ]);

  const lines = ["## User Context & Memory"];
  if (preferenceEntries.length) {
    lines.push(`Preferences: ${preferenceEntries
      .map(([key, value]) => `${key}=${JSON.stringify(value)}`)
      .join("; ")}`);
  }
  if (legalPatterns.length) {
    lines.push(`Legal patterns: ${legalPatterns
      .map((pattern) => pattern.key).join(", ")}`);
  }
  if (context) {
    lines.push(`This conversation: ${JSON.stringify({
      legalEntities: context.legalEntities.slice(0, 8),
      documentReferences: context.documentReferences.slice(0, 8),
      activeJurisdiction: context.activeJurisdiction,
    })}`);
  }

  const text = lines.join("\n");
  return text.split(/\s+/).slice(0, 400).join(" ");
}
```

The 400-word limit ensures memory context does not consume too much of the token budget.

#### RAG Results Injection
When the AI searches for relevant documents, the results are injected into the conversation as tool results. These are not part of the system prompt but are included in the message history.

#### Document Content Injection
For small documents (under 6000 characters), the full content is eagerly loaded and injected into the prompt. For larger documents, only the manifest entry is included, and the AI must use `read_document` to fetch the full content.

### Token Budget Management

The system carefully manages token usage to stay within LLM context window limits:

- **System prompt**: Includes document manifest, templates, workflows, and instructions (variable size)
- **User memory**: Capped at 400 words
- **Document content**: Eagerly loaded up to 6000 chars per doc, 18000 chars total
- **Message history**: Maintained by the chat orchestrator with pruning
- **Tool results**: Included in the message history

The `selectManifestEntries()` function in `promptBuilder.ts` compresses the document manifest when there are more than 40 documents in scope:

```typescript
export const SCOPED_MANIFEST_MAX_ITEMS = 40;

export function selectManifestEntries(req: PromptRequest, docIndex: DocIndex) {
  const entries = Object.entries(docIndex);
  if (entries.length <= SCOPED_MANIFEST_MAX_ITEMS) return entries;
  // Prioritize: focused docs > primary docs > supporting docs
  const focusedLabels = focusedDocLabels(req, docIndex);
  const focused = entries.filter(([label]) => focusedLabels.has(label));
  const primary = entries.filter(
    ([label, info]) => !focusedLabels.has(label) && info.context_role !== "supporting",
  );
  const rest = entries.filter(
    ([label, info]) => !focusedLabels.has(label) && info.context_role === "supporting",
  );
  return [...focused, ...primary, ...rest].slice(0, SCOPED_MANIFEST_MAX_ITEMS);
}
```

---

## 11. Memory Read/Write Operations

### Read Operations

#### Get User Preferences
```typescript
// From memoryLayer.ts
async getUserPreferences(userId: string): Promise<Record<string, unknown>> {
  const rows = await db.select().from(userMemories)
    .where(and(
      eq(userMemories.userId, userId),
      eq(userMemories.type, "preference"),
      or(
        isNull(userMemories.expiresAt),
        gt(userMemories.expiresAt, new Date())
      ),
    ));
  return Object.fromEntries(rows.map((row) => [row.key, row.value]));
}
```
Returns all non-expired preference memories for a user as a key-value map.

#### Get Legal Patterns
```typescript
async getLegalPatterns(userId: string): Promise<Array<{
  key: string; value: unknown; confidence: number;
}>> {
  const rows = await db.select().from(userMemories)
    .where(and(
      eq(userMemories.userId, userId),
      eq(userMemories.type, "legal_pattern"),
      or(
        isNull(userMemories.expiresAt),
        gt(userMemories.expiresAt, new Date())
      ),
    ))
    .orderBy(desc(userMemories.confidence))
    .limit(10);
  return rows.map((row) => ({
    key: row.key, value: row.value, confidence: row.confidence,
  }));
}
```
Returns up to 10 legal patterns for a user, ordered by confidence (highest first).

#### Get Long-Term Memory
```typescript
async getLongTerm(userId: string, key: string): Promise<{
  value: unknown; confidence: number; type: string;
} | null> {
  const [memory] = await db.select().from(userMemories)
    .where(and(
      eq(userMemories.userId, userId),
      eq(userMemories.key, key),
      or(
        isNull(userMemories.expiresAt),
        gt(userMemories.expiresAt, new Date())
      ),
    )).limit(1);
  if (!memory) return null;
  // Update access stats
  await db.update(userMemories)
    .set({
      accessCount: sql`${userMemories.accessCount} + 1`,
      lastAccessedAt: new Date(),
    }).where(eq(userMemories.id, memory.id));
  return { value: memory.value, confidence: memory.confidence, type: memory.type };
}
```
Retrieves a specific memory by key, increments its access count, and returns the value with metadata.

#### Get Conversation Context
```typescript
async getConversationContext(chatId: string): Promise<ConversationContextValue | null> {
  const [row] = await db.select().from(conversationContext)
    .where(eq(conversationContext.chatId, chatId))
    .limit(1);
  if (!row) return null;
  return {
    shortTermMemory: row.shortTermMemory,
    legalEntities: row.legalEntities,
    documentReferences: row.documentReferences,
    taskHistory: row.taskHistory,
    activeJurisdiction: row.activeJurisdiction,
  };
}
```
Retrieves the full conversation context for a chat.

#### Build Memory Context
```typescript
async buildMemoryContext(userId: string, chatId: string): Promise<string> {
  const [preferences, legalPatterns, context] = await Promise.all([
    this.getUserPreferences(userId),
    this.getLegalPatterns(userId),
    this.getConversationContext(chatId),
  ]);
  // ... format and return ...
}
```
Builds a formatted string summarizing all relevant memory for a user and conversation.

### Write Operations

#### Set Long-Term Memory
```typescript
async setLongTerm(params: {
  userId: string; key: string; value: unknown;
  type?: string; confidence?: number;
  expiresInDays?: number; sourceSessionId?: string;
}): Promise<void> {
  const now = new Date();
  const expiresAt = params.expiresInDays
    ? new Date(now.getTime() + params.expiresInDays * 24 * 60 * 60 * 1000)
    : null;
  await db.insert(userMemories).values({
    userId: params.userId, key: params.key, value: params.value,
    type: params.type ?? "preference",
    confidence: params.confidence ?? 1,
    expiresAt, sourceSessionId: params.sourceSessionId ?? null,
    updatedAt: now,
  }).onConflictDoUpdate({
    target: [userMemories.userId, userMemories.key],
    set: { /* update all fields */ },
  });
}
```
Creates or updates a persistent memory. Uses upsert to avoid duplicates.

#### Save Conversation Context
```typescript
async saveConversationContext(params: {
  chatId: string; userId: string;
  updates: Partial<{
    shortTermMemory: Record<string, unknown>;
    legalEntities: unknown[];
    documentReferences: unknown[];
    taskHistory: unknown[];
    activeJurisdiction: string;
  }>;
}): Promise<void> {
  // Merge with existing context and upsert to database
}
```
Updates conversation context by merging new data with existing data.

#### Record Learning Event
```typescript
async recordLearningEvent(params: {
  userId: string; chatId: string;
  messageId?: string; eventType: string;
  messageContent?: string; userQuery?: string;
  context?: Record<string, unknown>;
}): Promise<void> {
  await db.insert(learningEvents).values({
    userId: params.userId, chatId: params.chatId,
    messageId: params.messageId ?? null,
    eventType: params.eventType,
    messageContent: params.messageContent ?? null,
    userQuery: params.userQuery ?? null,
    context: params.context ?? {},
  });
}
```
Records an event for later processing by the learning system.

#### Set Session Memory
```typescript
setSession(chatId: string, key: string, value: unknown): void {
  this.sessionMemory.set(chatId, key, value);
}
```
Stores a value in the in-memory session store.

---

## 12. Memory Expiration

### Session Memory
Session memory expires immediately when the conversation ends or the server restarts. There is no gradual decay - it is either there or it is not. The `clear(chatId)` method can also explicitly remove a session.

### Persistent Memory
Persistent memory can have an optional expiration date set via the `expiresInDays` parameter:

```typescript
const expiresAt = params.expiresInDays
  ? new Date(now.getTime() + params.expiresInDays * 24 * 60 * 60 * 1000)
  : null;
```

When `expiresInDays` is not provided, the memory never expires. When it is provided, the memory is ignored after the expiration date but not physically deleted from the database.

All read operations filter out expired memories:
```typescript
or(isNull(userMemories.expiresAt), gt(userMemories.expiresAt, new Date()))
```

### User Memory
User memories (type "preference") generally do not expire. They persist until explicitly overwritten or until the user is deleted. The system prefers to update existing preferences rather than creating new ones, so stale preferences are typically replaced rather than expired.

### Agent Memory
Agent memories (type "legal_pattern") can decay over time through several mechanisms:
- **Access-based decay**: Memories that are never accessed become less relevant
- **Time-based decay**: The `lastAccessedAt` field tracks when a memory was last used
- **Confidence decay**: While confidence can increase through positive feedback, there is no automatic decay mechanism currently implemented

The learning system focuses more on accumulating new patterns than on pruning old ones, keeping the memory store rich and comprehensive.

### Conversation Context
Conversation context is deleted when the associated chat is deleted, enforced by the `onDelete: "cascade"` foreign key constraint. There is no automatic expiration of conversation context.

### Learning Events
Learning events have a `processed` flag. Once processed, they are not deleted but are marked as processed to prevent duplicate processing. The events accumulate in the database as an audit trail.

---

## 13. Memory Privacy

### User Data Isolation
Every memory is scoped to a specific user via the `userId` foreign key. The database schema enforces this with foreign key constraints and cascading deletes. A user cannot access another user's memories because all queries filter by `userId`.

```typescript
// Every query filters by user ID
.eq(userMemories.userId, userId)
```

### Encryption
Memory values are stored as JSONB in PostgreSQL. While PostgreSQL JSONB is not encrypted at rest by default, the database connection uses SSL/TLS encryption in transit. For production deployments, PostgreSQL Transparent Data Encryption (TDE) or application-level encryption should be considered for sensitive memory values.

### Access Control
Memory operations are only available through authenticated API endpoints. The `userId` is extracted from the authenticated session, not from user input, preventing IDOR (Insecure Direct Object Reference) attacks.

### Right to Deletion
When a user is deleted, all their memories are automatically deleted through cascading foreign key constraints:

```typescript
userId: uuid("user_id").notNull()
  .references(() => users.id, { onDelete: "cascade" }),
```

This applies to `user_memories`, `conversation_context`, and `learning_events` tables.

### Audit Logging
The `learning_events` table provides an audit trail of what the system has learned about a user. Each event records:
- What happened (`eventType`)
- What the user said (`userQuery`)
- What the AI responded (`messageContent`)
- When it happened (`createdAt`)
- Additional context (`context`)

### Data Minimization
The system follows data minimization principles:
- Session memory is not persisted beyond the conversation
- Memory context in prompts is capped at 400 words
- Document manifests are capped at 40 items
- Eager document content is capped at 18000 characters total
- Learning events have a `processed` flag to enable cleanup

---

## 14. Memory Performance

### Caching Strategies
- **Session memory** is entirely in-memory (RAM) with no database calls, providing sub-millisecond access
- **User preferences** are loaded once per orchestration run and shared across all agents
- **Legal patterns** are limited to 10 results to keep queries fast
- **Document index** is built once per request and reused across all prompt building operations

### Lazy Loading
- Full document content is loaded lazily - only the manifest is included in the prompt by default
- RAG searches are performed on demand rather than pre-computed
- Learning events are processed asynchronously (fire-and-forget pattern)

```typescript
// From orchestrator/index.ts - learning is fire-and-forget
void learningSystem
  .extractFromRun({...})
  .catch((error) => console.error("[orchestrator/learning] failed:", error));
```

### Batch Operations
- Multiple memory reads are parallelized using `Promise.all()`:

```typescript
const [preferences, legalPatterns, context] = await Promise.all([
  this.getUserPreferences(userId),
  this.getLegalPatterns(userId),
  this.getConversationContext(chatId),
]);
```

- Learning events can be batch processed using the `processed` flag

### Index Optimization
The database has two key indexes for memory performance:

```typescript
userIdTypeIdx: index("user_memories_user_id_type_idx")
  .on(table.userId, table.type),
userIdKeyIdx: uniqueIndex("user_memories_user_id_key_idx")
  .on(table.userId, table.key),
```

- The composite index on `(userId, type)` speeds up queries that filter by user and memory type
- The unique index on `(userId, key)` enables fast lookups and prevents duplicate memories

### Query Optimization
- Memory queries use `limit(1)` for single-record lookups
- Legal patterns are limited to 10 results with `limit(10)`
- Expired memories are filtered at the database level using `gt(userMemories.expiresAt, new Date())`
- The `buildMemoryContext()` method caps output at 400 words to minimize prompt token usage

---

## 15. Memory Error Handling

### Database Failures
All database operations in the memory layer use Drizzle ORM, which provides error propagation. Database failures will throw errors that bubble up to the calling code.

Key failure scenarios and handling:

#### Connection Loss
If the database connection is lost during a write operation, the write will fail and the error will be logged. The conversation will continue with session memory only, and the persistent memory update will be lost.

#### Constraint Violations
The unique index on `(userId, key)` prevents duplicate memories. The `onConflictDoUpdate` pattern ensures that writes are idempotent - writing the same memory twice simply updates it.

#### Timeout
Database queries have implicit timeouts set by the database driver. Long-running queries will be terminated by the database.

### Memory Corruption
Memory values are stored as JSONB, which PostgreSQL validates on write. Corrupted JSON will be rejected by the database. For application-level corruption (e.g., storing invalid data structures), the system relies on TypeScript type checking at compile time.

### Recovery Strategies
- **Session memory**: Cannot be recovered after loss. The system continues without it.
- **Persistent memory**: If a write fails, the memory is simply not saved. The system continues without it.
- **Conversation context**: If a read fails, the system treats it as if no context exists and builds fresh context.
- **Learning events**: If recording fails, the learning is lost but the conversation is unaffected.

### Fallback Behavior
The system is designed to be resilient to memory failures:

```typescript
// From orchestrator/index.ts - memory failure does not break the pipeline
void learningSystem
  .extractFromRun({...})
  .catch((error) => console.error("[orchestrator/learning] failed:", error));
```

Learning extraction is fire-and-forget, so a failure in learning does not affect the main response generation.

The orchestrator also falls back to the legacy chat handler if orchestration fails:

```typescript
} catch (error) {
  console.error("[orchestrator/stream] error, falling back to legacy:", error);
  emitSse(write, {
    type: "orchestration_fallback",
    reason: error instanceof Error ? error.message : "Unexpected orchestration error",
  });
  const legacyResult = await this.legacy.handle({...});
}
```

---

## 16. Memory Testing

### Unit Tests
Unit tests for memory should cover:

#### Session Memory
- Setting and getting values
- Getting all values for a chat
- Clearing a chat's session
- Isolation between different chat IDs

#### Persistent Memory
- Creating new memories
- Updating existing memories (upsert behavior)
- Reading memories with and without expiry
- Access count increment on read
- User preference retrieval
- Legal pattern retrieval with confidence ordering

#### Conversation Context
- Creating new context
- Merging updates with existing context
- Task history append behavior
- Short-term memory merge behavior

### Integration Tests
Integration tests should cover:

#### Memory End-to-End
- Full lifecycle: create user, store memory, retrieve memory, verify content
- Cross-session persistence: store in one request, retrieve in another
- Expiry behavior: create memory with short expiry, verify it is not returned after expiry

#### Memory Injection
- Verify that user preferences appear in the prompt context
- Verify that document context appears in the system prompt
- Verify that memory context is capped at 400 words

#### Learning System
- Verify that positive feedback increases memory confidence
- Verify that concise requests create response style preference
- Verify that jurisdiction mentions create jurisdiction preference
- Verify that task history is appended correctly

### Performance Tests
Performance tests should cover:

- Concurrent memory reads and writes for the same user
- Memory performance under high conversation volume
- Database query performance with large numbers of memories
- Memory context building time with many preferences

### Privacy Tests
Privacy tests should cover:

- User A cannot read User B's memories
- Deleting a user cascades to all memory tables
- Expired memories are not returned in any query
- Memory values are not logged in application logs

---

## 17. Complete Memory Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           USER SENDS MESSAGE                                │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CHAT ORCHESTRATOR receives message                      │
│                                                                              │
│  1. Load chat history from database                                         │
│  2. Load conversation context from conversation_context table               │
│  3. Build document index (DocIndex)                                         │
│  4. Check RAG collection status                                             │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PROMPT BUILDER assembles prompt                       │
│                                                                              │
│  System Prompt:                                                              │
│  ├── Scope instructions (project/workspace/personal)                        │
│  ├── Document manifest (up to 40 items)                                     │
│  ├── Template list                                                          │
│  ├── Workflow state                                                         │
│  ├── Interview state                                                        │
│  ├── Permissions                                                            │
│  └── Search mode instructions                                               │
│                                                                              │
│  User Memory:                                                                │
│  ├── User preferences (from user_memories WHERE type="preference")          │
│  ├── Legal patterns (from user_memories WHERE type="legal_pattern")         │
│  └── Conversation context (legal entities, doc refs, jurisdiction)          │
│                                                                              │
│  Document Content:                                                           │
│  ├── Eagerly loaded small docs (up to 6000 chars each, 18000 total)        │
│  └── Attached/displayed document content                                    │
│                                                                              │
│  RAG Results:                                                                │
│  └── Search results from vector store (on demand)                           │
│                                                                              │
│  Message History:                                                            │
│  └── Previous messages in conversation                                      │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     ORCHESTRATOR PIPELINE processes request                   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ 1. INTENT CLASSIFICATION                                          │       │
│  │    - Analyzes user message                                        │       │
│  │    - Determines if orchestration is needed                        │       │
│  │    - Falls back to legacy if not needed                           │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                    │                                         │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ 2. MEMORY CONTEXT LOADING                                        │       │
│  │    memoryLayer.buildMemoryContext(userId, chatId)                 │       │
│  │    ├── getUserPreferences(userId) ───► user_memories table        │       │
│  │    ├── getLegalPatterns(userId) ────► user_memories table         │       │
│  │    └── getConversationContext(chatId) ► conversation_context tbl  │       │
│  │    Returns: formatted string (capped at 400 words)               │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                    │                                         │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ 3. TASK PLANNING                                                 │       │
│  │    createTaskPlan() with:                                         │       │
│  │    - User message                                                 │       │
│  │    - Conversation history                                         │       │
│  │    - Intent classification                                        │       │
│  │    - Assembled context                                            │       │
│  │    - Available tools                                              │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                    │                                         │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ 4. AGENT EXECUTION                                               │       │
│  │    agentCoordinator.execute() with:                               │       │
│  │    - Task plan                                                    │       │
│  │    - Shared context (including memory context)                    │       │
│  │    - User ID, Chat ID, Project/Workspace ID                       │       │
│  │                                                                   │       │
│  │    Each agent:                                                    │       │
│  │    ├── Reads shared context                                       │       │
│  │    ├── Calls tools (which may access documents, RAG, etc.)       │       │
│  │    ├── Produces output                                            │       │
│  │    └── Results stored in shared context                           │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                    │                                         │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ 5. RESPONSE SYNTHESIS                                             │       │
│  │    generateFinalResponse() with:                                  │       │
│  │    - Original query                                               │       │
│  │    - Agent outputs                                                │       │
│  │    - Memory context                                               │       │
│  │    - Requested document types                                     │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                    │                                         │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ 6. SELF-CRITIQUE (if validation issues found)                    │       │
│  │    selfCritiqueAgent.critique() with:                             │       │
│  │    - Draft response                                               │       │
│  │    - Validation issues                                            │       │
│  │    - Memory context                                               │       │
│  │    May revise response if issues found                            │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                    │                                         │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ 7. RESPONSE DELIVERY                                              │       │
│  │    - Stream response to user                                      │       │
│  │    - Persist assistant message to chat_messages table             │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                    │                                         │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ 8. LEARNING (async, fire-and-forget)                              │       │
│  │    learningSystem.extractFromRun() with:                          │       │
│  │    ├── Detect response style preference                           │       │
│  │    │   (brief/quick/short/summary/tldr)                           │       │
│  │    │   ──► user_memories WHERE type="preference"                  │       │
│  │    ├── Detect jurisdiction preference                             │       │
│  │    │   (california/new york/texas/uk/eu/federal)                  │       │
│  │    │   ──► user_memories WHERE type="preference"                  │       │
│  │    ├── Record legal pattern if research succeeded                 │       │
│  │    │   ──► user_memories WHERE type="legal_pattern"               │       │
│  │    └── Update conversation task history                           │       │
│  │        ──► conversation_context.taskHistory                       │       │
│  │                                                                    │       │
│  │    learningSystem.recordFeedback() (on user feedback):            │       │
│  │    ├── Record feedback event                                      │       │
│  │    │   ──► learning_events table                                   │       │
│  │    └── If positive, boost confidence of session memories          │       │
│  │        ──► user_memories.confidence += 0.05                       │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           USER RECEIVES RESPONSE                             │
│                                                                              │
│  Response includes:                                                          │
│  - AI-generated content (informed by all memory types)                       │
│  - Citations (if source-backed)                                              │
│  - Orchestration summary (internal metadata)                                │
│                                                                              │
│  Memory state after this turn:                                               │
│  - Session memory: updated with current turn data                           │
│  - Conversation context: task history appended                               │
│  - User memories: potentially new preferences/patterns                       │
│  - Learning events: new events recorded                                      │
│  - User feedback: if provided, adjusts memory confidence                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 18. Memory Configuration Reference

| Parameter | Location | Default | Description |
|-----------|----------|---------|-------------|
| `SCOPED_MANIFEST_MAX_ITEMS` | `promptBuilder.ts` | 40 | Maximum number of documents in the context manifest |
| `EAGER_CHARS_PER_DOC` | `chatOrchestrator.ts` | 6000 | Maximum characters eagerly loaded per document |
| `EAGER_CHARS_TOTAL` | `chatOrchestrator.ts` | 18000 | Maximum total characters eagerly loaded across all documents |
| `ORCHESTRATION_TIMEOUT_MS` | `orchestrator/index.ts` | 180,000 | Timeout for the full orchestration pipeline (3 minutes) |
| `PLAN_TIMEOUT_MS` | `orchestrator/index.ts` | 45,000 | Timeout for task planning phase (45 seconds) |
| `AGENTS_TIMEOUT_MS` | `orchestrator/index.ts` | 120,000 | Timeout for agent execution phase (2 minutes) |
| `SYNTHESIS_TIMEOUT_MS` | `orchestrator/index.ts` | 45,000 | Timeout for response synthesis phase (45 seconds) |
| `MAX_REPLANS` | `chatOrchestrator.ts` | 2 | Maximum number of replanning attempts |
| Memory context limit | `memoryLayer.ts` | 400 words | Maximum words in the memory context string |
| Legal patterns limit | `memoryLayer.ts` | 10 | Maximum legal patterns returned per query |
| Confidence boost | `learningSystem.ts` | +0.05 | Confidence increase per positive feedback event |
| Max confidence | `learningSystem.ts` | 1.0 | Maximum confidence score for any memory |
| Default confidence | `memoryLayer.ts` | 1.0 | Default confidence for new memories |
| Memory type | `memoryLayer.ts` | "preference" | Default type for new long-term memories |

---

## 19. Key Files Reference

| File | Lines | Purpose | Key Exports |
|------|-------|---------|-------------|
| `backend/src/lib/orchestrator/memory/memoryLayer.ts` | 250 | Central memory API | `memoryLayer`, `MemoryLayer` |
| `backend/src/lib/orchestrator/memory/sessionMemory.ts` | 24 | In-memory per-chat storage | `SessionMemory` |
| `backend/src/lib/orchestrator/memory/persistentMemory.ts` | 5 | Delegates to MemoryLayer | (empty module) |
| `backend/src/db/schema.ts` | 2525 | Database schema | `userMemories`, `conversationContext`, `learningEvents` |
| `backend/src/prompts/promptBuilder.ts` | 547 | Prompt assembly | `buildBaseSystemPromptExtra`, `buildUnifiedContextBlock`, `buildApiMessageInputs`, `buildAssistantRuntimeContextSummary` |
| `backend/src/lib/chatOrchestrator.ts` | 4145 | Chat orchestration | `ChatOrchestrator`, `detectIntent`, `buildDocContext`, `buildProjectDocContext`, `buildWorkspaceDocContext` |
| `backend/src/lib/agentic/contextBuilder.ts` | 341 | Context building | `buildMatterContextBlock`, `buildJudgmentContextBlock`, `buildWorkspaceGraphContextBlock` |
| `backend/src/lib/orchestrator/index.ts` | 488 | Orchestrator pipeline | `orchestratorPipeline`, `OrchestratorPipeline` |
| `backend/src/lib/orchestrator/learning/learningSystem.ts` | 110 | Learning extraction | `learningSystem`, `LearningSystem` |
| `backend/src/lib/chatTools.ts` | (varies) | Document context building | `buildDocContext`, `buildProjectDocContext`, `buildWorkspaceDocContext`, `buildWorkflowStore`, `readDocumentContent` |
| `backend/src/lib/rag.ts` | (varies) | RAG scope management | `scopeForRecord`, `RagScope` |
| `backend/src/lib/legal/skills.ts` | (varies) | Legal skill detection | `detectSkill`, `LegalSkill` |
| `backend/src/lib/legal/document-classifier.ts` | (varies) | Document type classification | `classifyLegalDocument`, `LegalDocumentType` |

### Database Tables

| Table | Schema Location | Purpose |
|-------|----------------|---------|
| `user_memories` | `schema.ts:1736-1758` | Persistent user preferences and patterns |
| `conversation_context` | `schema.ts:1760-1782` | Per-conversation structured data |
| `learning_events` | `schema.ts:1784-1805` | Audit trail of learning events |
| `chat_messages` | `schema.ts` | Message history (used by session memory) |
| `chats` | `schema.ts` | Chat metadata (referenced by conversation context) |

### Memory-Related Enums

| Enum | Values | Purpose |
|------|--------|---------|
| Document lifecycle status | DRAFT, IN_REVIEW, PENDING_APPROVAL, APPROVED, FINALIZED | Tracks document state for memory |
| RAG scope type | personal, project, workspace, document | Scopes RAG searches for document memory |
| RAG source type | document, drive_file | Identifies source of document memory |
| RAG source status | pending, indexed, failed, skipped_unsupported | Tracks document indexing for RAG |

---

*Document Version: 1.0*
*Last Updated: 2026-07-14*
*Codebase Reference: Mike Legal AI Platform*
