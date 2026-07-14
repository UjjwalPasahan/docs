# RAG Pipeline

## 1. What This Document Is

This document explains the complete RAG (Retrieval-Augmented Generation) pipeline in Mike — the system that lets the AI answer questions based on your actual documents rather than just its training data. Think of it as the bridge between your files and the AI's answers.

Every time you upload a document, ask a question about your files, or get an answer that references specific clauses or pages, this pipeline is working behind the scenes. It handles everything from breaking your documents into searchable pieces, to finding the most relevant passages, to feeding them to the AI so it can give you accurate, source-backed answers.

The RAG pipeline touches nearly every part of the platform: document uploads, chat conversations, search, citations, and even the prompts that guide the AI. This document covers all of it in plain language.

---

## 2. RAG in Simple Terms

RAG stands for **Retrieval-Augmented Generation**. Here's how it works using a librarian analogy:

**The Library Analogy:**

Imagine you walk into a massive law library with thousands of books, contracts, and legal documents. You ask the librarian a complex legal question. Here's what happens:

1. **You ask the question** — "What are the termination clauses in our vendor agreements?"

2. **The librarian searches the library** — Instead of reading every single book, the librarian quickly finds the pages that are most relevant to your question. They use two methods:
   - **Looking up keywords** — searching the index for "termination" and "vendor agreement"
   - **Understanding meaning** — recognizing that "ending the contract" is related to "termination"

3. **The librarian pulls the relevant pages** — They grab the specific pages from the specific books that contain the answer.

4. **The librarian hands those pages to you** — You now have the actual text from the actual documents.

5. **You read and answer** — Using those pages as proof, you give a well-informed answer.

That's exactly what Mike's RAG system does, but automatically:

- Your documents are the "books in the library"
- When you ask a question, the system finds the most relevant passages
- Those passages are given to the AI as context
- The AI uses those passages to give you an accurate, cited answer

**Why This Matters:**

Without RAG, the AI would only know what it learned during training — it couldn't reference your specific contracts, your specific clauses, or your specific documents. RAG makes the AI aware of YOUR data, YOUR files, and YOUR legal matters.

---

## 3. The Complete RAG Flow

Here's the entire pipeline from document upload to answer generation:

```
DOCUMENT INGESTION PIPELINE (when you upload a file):
═══════════════════════════════════════════════════════

  User Uploads Document
         │
         ▼
  ┌──────────────┐
  │  File Stored  │──── R2 Cloud Storage
  │  in R2       │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  DB Record   │──── documents / driveFiles tables
  │  Created     │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  OCR / Text  │──── Extract text from PDFs, images
  │  Extraction  │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Chunking    │──── Split into searchable pieces
  │              │     with context overlap
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Embedding   │──── Convert text to vector numbers
  │              │     (1024 dimensions)
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Indexing    │──── Store in RAG service
  │              │     + BM25 keyword index
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Collection  │──── Organized by scope
  │  Updated     │     (personal/project/workspace)
  └──────────────┘


SEARCH PIPELINE (when you ask a question):
════════════════════════════════════════════

  User Question
         │
         ▼
  ┌──────────────┐
  │  Query       │──── Clean and prepare the question
  │  Processing  │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Hybrid      │──── Vector search (meaning)
  │  Search      │     + BM25 search (keywords)
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Re-ranking  │──── Cross-encoder re-scores results
  │              │     for better relevance
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Context     │──── Select best chunks
  │  Assembly    │     Extract citations
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Prompt      │──── Add context to AI prompt
  │  Assembly    │     with instructions to cite
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  AI Answer   │──── AI generates response
  │  + Citations │     using retrieved context
  └──────────────┘
```

---

## 4. Document Ingestion Pipeline

### Step 1: Upload

When a user uploads a document, here's what happens:

1. **File arrives** — The user uploads a file through the UI (drag-and-drop, file picker, etc.)

2. **File stored in R2** — The file is saved to Cloudflare R2 object storage at a unique path like `documents/{userId}/{documentId}/{versionId}/{filename}`

3. **Database records created** — Two records are created:
   - `documents` table — The main document record with metadata (filename, type, size, owner)
   - `documentVersions` table — The specific version with its storage path and checksum

4. **RAG indexing queued** — The system calls `queueDocumentVersionIndex()` which starts the RAG ingestion process in the background

**Key Code Flow:**
```
User Upload → R2 Storage → DB Record → queueDocumentVersionIndex()
                                              │
                                              ▼
                                       queueRagIndex()
                                              │
                                              ▼
                                       indexRagSource() [background]
```

**Supported File Types:**
- PDF documents (`application/pdf`)
- Word documents (`application/msword`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`)
- Excel spreadsheets (`application/vnd.ms-excel`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`)
- PowerPoint presentations (`application/vnd.ms-powerpoint`, `application/vnd.openxmlformats-officedocument.presentationml.presentation`)
- Text files (`text/plain`, `text/csv`, `text/markdown`)
- RTF files (`application/rtf`)
- XML and JSON files

**Unsupported File Types (skipped):**
- HTML files (`text/html`)
- Image files (`image/*`) — These get marked as `skipped_unsupported` immediately

### Step 2: Processing

Once queued, the `indexRagSource()` function handles processing:

1. **Scope determination** — The system figures out which collection this document belongs to:
   - `personal` — User's personal documents
   - `project` — Documents within a specific project
   - `workspace` — Documents within a workspace

2. **MIME type detection** — The system determines the file type from the filename extension

3. **Unsupported format check** — If the file type isn't supported, it's marked as `skipped_unsupported` and processing stops

4. **Collection creation** — If the scope doesn't have a collection yet, one is created via the RAG service API

5. **Signed URL generation** — A temporary signed URL is created so the RAG service can access the file from R2

6. **Ingestion request** — The file URL is sent to the RAG service for processing

### Step 3: Chunking

The RAG service (external) handles chunking. Here's what happens:

1. **Document parsing** — The RAG service extracts text from the document:
   - PDF: Uses OCR and text extraction
   - DOCX: Parses the XML structure
   - Text files: Direct text reading

2. **Paragraph/section splitting** — The text is split into chunks, typically:
   - By paragraph boundaries
   - By section headings
   - By page boundaries (for PDFs)

3. **Context overlap** — Each chunk overlaps slightly with the previous one to maintain context across chunk boundaries

4. **Metadata extraction** — For each chunk, metadata is preserved:
   - Page number
   - Section title
   - Start/end character offsets
   - Document context (surrounding text)

5. **Citation mapping** — Each chunk gets a unique identifier that can be traced back to the source document

### Step 4: Embedding

After chunking, each text chunk is converted to a vector embedding:

1. **Text preprocessing** — The chunk text is cleaned and normalized

2. **Vector generation** — The text is converted to a 1024-dimensional vector using an embedding model. This vector captures the semantic meaning of the text

3. **Quality validation** — The system checks that the embedding was generated successfully

4. **Dimension handling** — Vectors are normalized to exactly 1024 dimensions

**What is a vector?**
Think of a vector as a list of 1024 numbers that represent the "meaning" of the text. Similar texts will have similar numbers. This allows the system to find semantically related content even if the exact words don't match.

### Step 5: Indexing

The final step stores everything in the RAG service:

1. **Vector storage** — The 1024-dimensional vectors are stored in a vector database for similarity search

2. **BM25 indexing** — A keyword-based index is created for exact text matching (BM25 is a classic information retrieval algorithm)

3. **Metadata indexing** — Document metadata (filename, page numbers, section titles) is indexed for filtering

4. **Collection update** — The collection is updated with the new document's chunks

5. **Status update** — The database record is updated to `indexed` status with a timestamp

---

## 5. Collection Management

Collections are how Mike organizes your documents for RAG search. Think of a collection as a "searchable library" within the system.

### Creating Collections

Collections are created automatically when documents are indexed:

```typescript
// From rag.ts:ensureRagCollection()
async function ensureRagCollection(scope: RagScope) {
  // Check if collection already exists
  const [existing] = await db.select().from(ragCollections)
    .where(and(
      eq(ragCollections.scopeType, scope.type),
      eq(ragCollections.scopeId, scope.id),
    ));
  if (existing) return existing;

  // Create new collection via RAG service
  const collectionName = await ragService.createCollection(scope);
  
  // Store in database
  const [created] = await db.insert(ragCollections).values({
    scopeType: scope.type,
    scopeId: scope.id,
    ownerUserId: scope.ownerUserId,
    collectionName,
    displayName: scope.name,
  });
  return created;
}
```

### Collection Scope Types

| Scope Type | Description | Example |
|------------|-------------|---------|
| `personal` | User's personal documents | User ID as scope ID |
| `project` | Documents within a project | Project UUID as scope ID |
| `workspace` | Documents within a workspace | Workspace UUID as scope ID |
| `document` | Single document scope | Document UUID as scope ID |

### Collection Configuration

When a collection is created, it's configured with:
- **Vector dimension**: 1024 (standard for modern embedding models)
- **BM25 language**: English
- **BM25 stemming**: Enabled (matches "running" with "run")
- **BM25 bigrams**: Enabled (considers word pairs)
- **BM25 k1**: 1.5 (term frequency saturation)
- **BM25 b**: 0.75 (length normalization)

### Adding Documents to Collections

Documents are added via the `ingestDocument()` API call:

```typescript
// From rag.ts:ingestDocument()
async ingestDocument(collectionName, documentId, url) {
  const response = await fetch(`${this.apiUrl}/ingest`, {
    method: "POST",
    body: JSON.stringify({
      document_urls: [url],
      collection_name: collectionName,
      document_ids: [documentId],
    }),
  });
  return await response.json();
}
```

### Removing Documents

Documents can be removed from collections:

```typescript
// From rag.ts:deleteDocument()
async deleteDocument(collectionName, documentId) {
  await fetch(`${this.apiUrl}/document/delete`, {
    method: "POST",
    body: JSON.stringify({
      collection_name: collectionName,
      document_id: documentId,
    }),
  });
}
```

### Collection Permissions

Access to collections is controlled by scope:
- **Personal scope**: Only the owner can search
- **Project scope**: Users with project access can search
- **Workspace scope**: Workspace members can search

---

## 6. Search Pipeline

When you ask a question, the search pipeline finds the most relevant passages from your documents.

### Step 1: Query Processing

Before searching, the query is processed:

1. **Trimming** — Whitespace is removed from the query
2. **Scope resolution** — The system determines which collections to search:
   - Workspace scope (if in a workspace)
   - Project scope (if in a project)
   - Personal scope (fallback)

**Search priority order:**
```
1. Workspace scope (if user has access)
2. Project scope (if user has access)
3. Personal scope (always as fallback)
```

### Step 2: Hybrid Search

The system uses **hybrid search** — combining two search methods for better results:

**Vector Search (Semantic):**
- Converts the query to a vector embedding
- Finds chunks with similar vector representations
- Good for: "What are the termination conditions?" (finds "ending the contract")

**BM25 Search (Keyword):**
- Uses classic information retrieval
- Finds chunks containing exact keywords
- Good for: "Section 4.2" (finds that exact section)

**Result Merging:**
```typescript
// From rag.ts:queryCollection()
body: JSON.stringify({
  query,
  collection_name: collectionName,
  top_k: topK,
  vector_weight: 0.5,  // 50% weight to semantic search
  bm25_weight: 0.5,    // 50% weight to keyword search
  use_reranking: true,  // Enable re-ranking
  rerank_top_k: topK,   // Re-rank top K results
})
```

### Step 3: Re-ranking

After hybrid search, results are re-ranked using a cross-encoder:

1. **Cross-encoder model** — A more sophisticated model that reads both the query AND each result together
2. **Score recalculation** — Each result gets a new relevance score
3. **Threshold filtering** — Results below a minimum score are removed
4. **Result limiting** — Only the top K results are returned

### Step 4: Context Assembly

The final step assembles the context for the AI:

1. **Chunk selection** — The best chunks are selected based on scores
2. **Citation extraction** — Each chunk's citation information is extracted:
   - Document name
   - Page number
   - Section title
   - Character offsets
3. **Deep link generation** — Links to the source document are created
4. **Deduplication** — Duplicate chunks from the same source are removed
5. **Score normalization** — Scores are normalized for consistent ranking

---

## 7. Citation Pipeline

Citations are how Mike proves its answers come from your actual documents.

### Citation Detection in Chunks

When a chunk is returned from search, the system extracts citation information:

```typescript
// From rag.ts:searchSingleScope()
const citation = {
  evidence_id: stableEvidenceId({...}),  // Unique identifier
  source_id: sourceId,                    // Document ID
  source_type: sourceType,                // "document" or "drive_file"
  version_id: result.document_id,         // Version ID
  chunk_id: chunkId,                      // Chunk identifier
  quote_hash: resultQuoteHash,            // Hash of the quoted text
  text_hash: textHash,                    // Hash of the full text
  page_number: pageNum,                   // Page number
  filename: filename,                     // Original filename
  document_type: citationDocumentType(filename, mimeType),
  location: {
    pageNumber,
    snippet: snippetText.slice(0, 500),
    snippetContext: context.slice(0, 1000),
    charLength: snippetText.length,
    startOffset,
    endOffset,
    sectionTitle,
  },
};
```

### Citation Mapping to Source

Each citation is mapped back to its source document:

1. **Version lookup** — The system looks up the document version by ID
2. **Entry matching** — Finds the `ragSourceIndexEntries` record
3. **Source resolution** — Maps back to the original document or drive file
4. **URL generation** — Creates deep links to the document:
   - Documents: `/documents/{sourceId}?page={pageNum}&highlight={snippet}`
   - Drive files: `/drive/{sourceId}?page={pageNum}&highlight={snippet}`

### Citation Formatting

Citations are formatted for the AI to use:

```typescript
// From chatTools.ts
When a search_sources result includes evidence_id, cite that exact 
evidence handle inline, for example [ev_123abc]. Never invent evidence IDs.

In the <CITATIONS> block, copy:
- evidence_id
- source_id
- source_type
- version_id
- chunk_id
- quote_hash
- page
- filename
- quote
```

### Citation Verification

Before storing, citations are verified:

1. **Evidence ID validation** — Every `evidence_id` is checked against the database
2. **Quote hash matching** — The quoted text must match the original
3. **Source existence** — The referenced document must exist
4. **Permission check** — The user must have access to the source

### Citation Streaming to UI

Citations are streamed to the UI in real-time:

```typescript
// From chatTools.ts:search_sources handler
write(`data: ${JSON.stringify({
  type: "source_results",
  tool_call_id: tc.id,
  query,
  scope: output.scope,
  results: output.results,
  count: output.count,
})}\n\n`);
```

---

## 8. Hybrid Search

Hybrid search combines two search approaches for the best results.

### Vector Search (Semantic)

**How it works:**
1. The query is converted to a 1024-dimensional vector
2. The system finds chunks whose vectors are closest to the query vector
3. "Closest" means most semantically similar

**Example:**
- Query: "What happens if we breach the contract?"
- Vector search finds: "Termination for cause" section (semantically related)
- Even though "breach" and "termination" are different words, the meaning is similar

**Strengths:**
- Understands meaning and context
- Handles synonyms and paraphrases
- Works across languages (with multilingual models)

**Weaknesses:**
- May miss exact matches
- Can be less precise for specific terms

### BM25 Search (Keyword)

**How it works:**
1. The query is tokenized into keywords
2. Each keyword is looked up in the inverted index
3. Results are scored based on term frequency and document frequency

**Example:**
- Query: "Section 4.2 payment terms"
- BM25 finds: Documents containing "Section 4.2" and "payment terms"
- Exact keyword matching

**Strengths:**
- Excellent for exact matches
- Fast and efficient
- Works well for technical terms

**Weaknesses:**
- No understanding of meaning
- Misses synonyms
- Can't handle paraphrases

### Combined Scoring

The final score combines both approaches:

```
Combined Score = (Vector Score × 0.5) + (BM25 Score × 0.5)
```

**Why 50/50?**
- Equal weight ensures both approaches contribute
- Vector search handles semantic queries
- BM25 handles exact matches
- The cross-encoder re-ranking then refines the results

---

## 9. Context Compression

When too many chunks are retrieved, context compression ensures only the most relevant information is used.

### Extractive Summarization

Instead of using entire chunks, the system extracts the most relevant sentences:

1. **Relevance scoring** — Each sentence is scored against the query
2. **Top-N selection** — The top N sentences are selected
3. **Context preservation** — Surrounding context is included for coherence

### Key Point Extraction

The system identifies the key points in each chunk:

1. **Topic detection** — Identifies the main topics in the chunk
2. **Key sentence selection** — Selects sentences that represent key points
3. **Redundancy removal** — Removes duplicate information across chunks

### Redundancy Removal

When multiple chunks contain similar information:

1. **Overlap detection** — Finds overlapping content between chunks
2. **Deduplication** — Removes duplicate passages
3. **Consolidation** — Merges similar information

### Token Budget Management

The system manages the total token count sent to the AI:

1. **Token counting** — Estimates tokens for each chunk
2. **Budget allocation** — Allocates tokens across chunks based on relevance
3. **Truncation** — If needed, chunks are truncated to fit the budget

---

## 10. Prompt Assembly for RAG

The RAG context is carefully assembled into the AI's prompt.

### System Prompt with Context

The prompt builder adds RAG information to the system prompt:

```typescript
// From promptBuilder.ts:buildUnifiedContextBlock()
const lines = [
  "CONTEXT:",
  `Scope: ${req.scope.type} - ${scopeRef}`,
  `RAG: ${ragStatus.indexedSourceCount} indexed source(s)${
    ragCollection
      ? `, ready to search (${ragCollection.displayName || ragCollection.collectionName})`
      : ", no collection available"
  }`,
  // ... document manifest ...
];
```

### Retrieved Chunks as Context

When the AI calls `search_sources`, the results are added to the conversation:

```typescript
// From chatTools.ts:search_sources handler
const output = await searchRagSources({
  userId,
  userEmail,
  projectId,
  workspaceId,
  query,
  topK,
});
```

The results include:
- **Rank** — Position in the result list
- **Document name** — Source filename
- **Page number** — Where in the document
- **Text snippet** — The relevant passage
- **Score** — Relevance score
- **Deep link URL** — Link to the source

### Citation Formatting

Citations are formatted for the AI to reference:

```typescript
// From chatTools.ts
EVIDENCE NAVIGATION CITATIONS (Required for search_sources results):
When a search_sources result includes evidence_id, cite that exact 
evidence handle inline, for example [ev_123abc].
```

### Instructions to Use Provided Context

The prompt instructs the AI to use the retrieved context:

```typescript
// From promptBuilder.ts
"TURN ROUTING: The user is asking a source-backed search question. 
Use search_sources first with a focused query before answering, 
then cite document content when making factual claims."
```

---

## 11. RAG Quality Metrics

The RAG system tracks several quality metrics to ensure accurate retrieval.

### Recall@k

**What it measures:** Of all relevant documents, how many did we retrieve in the top k results?

**Formula:** `Recall@k = (relevant documents in top k) / (total relevant documents)`

**Example:**
- 10 documents contain the answer
- Top 8 results include 7 of them
- Recall@8 = 7/10 = 0.7

### Precision@k

**What it measures:** Of the top k results, how many are actually relevant?

**Formula:** `Precision@k = (relevant documents in top k) / k`

**Example:**
- Top 8 results
- 6 are actually relevant
- Precision@8 = 6/8 = 0.75

### MRR (Mean Reciprocal Rank)

**What it measures:** How high up is the first relevant result?

**Formula:** `MRR = 1 / rank of first relevant result`

**Example:**
- First relevant result is at rank 1
- MRR = 1/1 = 1.0 (perfect)
- First relevant result is at rank 3
- MRR = 1/3 ≈ 0.33

### NDCG (Normalized Discounted Cumulative Gain)

**What it measures:** How well are the results ranked?

**Key idea:** Relevant results at the top are worth more than relevant results at the bottom.

**Example:**
- Rank 1: Very relevant (high weight)
- Rank 2: Somewhat relevant (medium weight)
- Rank 3: Less relevant (low weight)

### Citation Accuracy

**What it measures:** When the AI cites a source, is the citation correct?

**Checks:**
- Does the cited text actually support the claim?
- Is the citation pointing to the right document?
- Is the page number correct?

---

## 12. RAG Configuration

The RAG system has several configurable parameters:

### Chunk Size

- **Default:** Determined by the RAG service
- **Typical:** 500-2000 characters per chunk
- **Trade-off:** Smaller chunks = more precise retrieval; larger chunks = more context

### Chunk Overlap

- **Purpose:** Maintains context across chunk boundaries
- **Typical:** 10-20% of chunk size
- **Example:** 1000-character chunks with 200-character overlap

### Embedding Model

- **Dimensions:** 1024
- **Type:** Dense vector embeddings
- **Purpose:** Captures semantic meaning of text

### Search Parameters

```typescript
// From rag.ts:queryCollection()
{
  top_k: 8,              // Number of results to return
  vector_weight: 0.5,    // Weight for semantic search
  bm25_weight: 0.5,      // Weight for keyword search
  use_reranking: true,   // Enable cross-encoder re-ranking
  rerank_top_k: 8,       // Number of results to re-rank
}
```

### BM25 Parameters

```typescript
// From rag.ts:createCollection()
{
  bm25_language: "english",
  bm25_use_stemming: true,   // Match "running" with "run"
  bm25_use_bigrams: true,    // Consider word pairs
  bm25_k1: 1.5,              // Term frequency saturation
  bm25_b: 0.75,              // Length normalization
}
```

### Collection Settings

- **Vector dimension:** 1024 (fixed)
- **Scope types:** personal, project, workspace, document
- **Source types:** document, drive_file

---

## 13. RAG Performance

The RAG system includes several performance optimizations.

### Embedding Caching

- **Purpose:** Avoid re-embedding the same text
- **Method:** Vector embeddings are cached in the RAG service
- **Benefit:** Faster repeated searches

### Search Caching

- **Purpose:** Cache frequent search queries
- **Method:** Search results are cached at the RAG service level
- **Benefit:** Faster response for common queries

### Index Optimization

- **BM25 indexing:** Uses inverted index for fast keyword search
- **Vector indexing:** Uses approximate nearest neighbor (ANN) for fast similarity search
- **Metadata indexing:** Uses database indexes for fast filtering

### Query Optimization

- **Scope-based search:** Only searches relevant collections
- **Parallel search:** Searches multiple scopes simultaneously
- **Early termination:** Stops searching once enough results are found

### Batch Processing

- **Ingestion:** Documents are processed in the background
- **Queue system:** `queueRagIndex()` adds to a background queue
- **Non-blocking:** Upload doesn't wait for indexing to complete

---

## 14. RAG Error Handling

The RAG system handles various error scenarios gracefully.

### Embedding Failures

**Error:** RAG service fails to generate embeddings

**Response:**
```typescript
// From rag.ts:normalizeRagError()
{
  summary: "RAG ingest failed (500): Service temporarily unavailable",
  code: "500",
  category: "provider_bug",
  retryable: true,
  retryAfterSeconds: 30,
}
```

**Recovery:** Mark as `failed` with `retryable: true`, allow manual retry

### Search Failures

**Error:** RAG service fails to execute search

**Response:**
```typescript
{
  ok: false,
  message: "No indexed source collections found for this chat scope.",
  results: [],
  count: 0,
}
```

**Recovery:** Return empty results, log the error

### Index Corruption

**Error:** Vector index becomes corrupted

**Response:**
```typescript
{
  status: "failed",
  lastError: "Index corruption detected",
  lastErrorCode: "index_corruption",
  retryable: false,
}
```

**Recovery:** Require manual re-indexing

### Recovery Strategies

1. **Automatic retry:** For transient errors (network issues, timeouts)
2. **Manual retry:** Users can retry failed sources via API
3. **Backfill:** `backfillRagSources()` re-indexes all documents
4. **Graceful degradation:** Search continues with available data

---

## 15. RAG Security

The RAG system includes several security measures.

### Collection Access Control

```typescript
// From rag.ts:searchRagSources()
// Build list of scopes to search in priority order:
// 1. Workspace scope (if provided and accessible)
// 2. Project scope (if provided and accessible)
// 3. Personal scope (always as fallback)

if (input.workspaceId) {
  const workspaceIds = await listAccessibleWorkspaceIds(input.userId);
  if (workspaceIds.includes(input.workspaceId)) {
    // User has access, add to search scopes
  }
}
```

### Document Permissions

- **Workspace scope:** Only workspace members can search
- **Project scope:** Only users with project access can search
- **Personal scope:** Only the owner can search

### Query Sanitization

- **Input trimming:** Whitespace is removed
- **Query length limits:** Maximum query length enforced
- **Special character handling:** Queries are properly escaped

### Audit Logging

```typescript
// From rag.ts:queueRagIndex()
logger.info("RAG background index completed", {
  filename: input.filename,
  sourceType: input.sourceType,
  sourceId: input.sourceId,
  versionId: input.versionId,
  status: entry?.status ?? "unknown",
});
```

### Data Isolation

- **Scope-based isolation:** Each scope has its own collection
- **No cross-scope search:** Users can only search their accessible scopes
- **Version tracking:** Each document version is tracked separately

---

## 16. Complete RAG Flow Diagram

Here's the complete flow from document upload to answer generation, showing every step:

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                        COMPLETE RAG FLOW DIAGRAM                           ║
╚══════════════════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────────────────┐
│                         DOCUMENT UPLOAD FLOW                                │
└─────────────────────────────────────────────────────────────────────────────┘

  User (Browser)
       │
       │  1. Upload file (drag-drop / file picker)
       ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                         Frontend (React)                                │
  │  - File selection UI                                                    │
  │  - Upload progress indicator                                            │
  │  - POST /api/documents/upload                                          │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                         Backend (Express)                               │
  │  2. Validate file type and size                                         │
  │  3. Generate unique document ID                                         │
  │  4. Create document record in DB                                        │
  │  5. Create version record in DB                                         │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                         Cloudflare R2                                   │
  │  6. Store file at path:                                                 │
  │     documents/{userId}/{docId}/{versionId}/{filename}                   │
  │  7. Return storage path                                                 │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    RAG Index Queue (Background)                         │
  │  8. queueDocumentVersionIndex() called                                  │
  │  9. queueRagIndex() adds to background queue                            │
  │  10. Returns immediately (non-blocking)                                 │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    RAG Index Worker (Background)                        │
  │  11. indexRagSource() processes the document                            │
  │  12. Determine scope (personal/project/workspace)                       │
  │  13. Ensure collection exists (create if needed)                        │
  │  14. Check file type support                                            │
  │  15. Generate signed URL for RAG service                                │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    External RAG Service                                  │
  │  16. Receive ingestion request                                          │
  │  17. Download file from signed URL                                      │
  │  18. Parse document (OCR for PDFs, XML for DOCX)                        │
  │  19. Extract text content                                               │
  │  20. Split into chunks (by paragraph/section/page)                      │
  │  21. Generate 1024-dim vector embeddings                                │
  │  22. Build BM25 keyword index                                           │
  │  23. Store vectors and metadata                                         │
  │  24. Return success status                                              │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    Update Database Status                                │
  │  25. Update ragSourceIndexEntries:                                       │
  │      - status: "indexed"                                                │
  │      - indexedAt: now                                                   │
  │      - lastError: null                                                  │
  │  26. Log completion                                                     │
  └─────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                         SEARCH FLOW                                         │
└─────────────────────────────────────────────────────────────────────────────┘

  User (Browser)
       │
       │  1. Ask question: "What are the termination clauses?"
       ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                         Frontend (React)                                │
  │  2. User types question in chat                                         │
  │  3. Message sent to backend via SSE                                     │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    AI Orchestrator (Backend)                             │
  │  4. Receive user message                                                │
  │  5. Classify intent (source-backed question)                            │
  │  6. Add search_sources to available tools                               │
  │  7. Send to LLM with system prompt                                      │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    LLM (OpenAI/Gemini)                                  │
  │  8. Process user question                                               │
  │  9. Decide to call search_sources tool                                  │
  │  10. Generate tool call:                                                │
  │      search_sources({                                                   │
  │        query: "termination clauses vendor agreements",                   │
  │        top_k: 8                                                         │
  │      })                                                                 │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    Tool Executor (Backend)                               │
  │  11. Parse tool call arguments                                          │
  │  12. Validate query and top_k                                           │
  │  13. Call searchRagSources()                                            │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    RAG Search Service                                    │
  │  14. Resolve search scopes:                                              │
  │      - Check workspace access                                           │
  │      - Check project access                                             │
  │      - Add personal scope as fallback                                   │
  │  15. For each scope:                                                    │
  │      a. Look up collection in DB                                        │
  │      b. Call RAG service query API                                      │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    External RAG Service (Query)                          │
  │  16. Receive query request                                              │
  │  17. Convert query to vector embedding                                  │
  │  18. Vector similarity search (cosine distance)                         │
  │  19. BM25 keyword search                                                │
  │  20. Merge results (50/50 weighting)                                    │
  │  21. Cross-encoder re-ranking                                           │
  │  22. Return ranked results with scores                                  │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    Result Processing                                     │
  │  23. Map results to SearchResultItem format                             │
  │  24. Generate evidence IDs (stable, deterministic)                      │
  │  25. Build citation locations (page, snippet, offsets)                  │
  │  26. Generate deep link URLs                                            │
  │  27. Deduplicate by evidence ID                                         │
  │  28. Sort by score descending                                           │
  │  29. Take top K results                                                 │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    Evidence Persistence                                  │
  │  30. persistEvidenceFromSearchResults()                                 │
  │  31. Store in evidence_source_refs table:                               │
  │      - evidence_id, source_id, version_id                               │
  │      - chunk_id, quote_hash, page_number                                │
  │      - filename, location, metadata                                     │
  │  32. Return evidence IDs for citation                                   │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    Stream Results to Frontend                            │
  │  33. Send SSE event: source_results                                     │
  │      {                                                                  │
  │        type: "source_results",                                          │
  │        query, scope, results, count                                     │
  │      }                                                                  │
  │  34. Send SSE event: tool_result                                        │
  │      {                                                                  │
  │        type: "tool_result",                                             │
  │        tool: "search_sources",                                          │
  │        output, status: "complete"                                       │
  │      }                                                                  │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    Frontend (React)                                      │
  │  35. Receive SSE events                                                 │
  │  36. Display source results in chat                                     │
  │  37. Show document name, page, snippet                                  │
  │  38. Render clickable citation links                                    │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    LLM Continues Generation                             │
  │  39. Receive search results as tool response                            │
  │  40. Read retrieved chunks                                              │
  │  41. Generate answer based on context                                   │
  │  42. Include citations: [ev_abc123]                                     │
  │  43. Return final answer                                                │
  └───────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    Final Answer Displayed                                │
  │  44. AI answer shown in chat                                            │
  │  45. Citations are clickable                                            │
  │  46. User can navigate to source document                               │
  └─────────────────────────────────────────────────────────────────────────┘
```

---

## 17. Key Files Reference

| File | Lines | Purpose |
|------|-------|---------|
| `backend/src/lib/rag.ts` | 1350 | Core RAG logic: collections, search, ingestion, error handling |
| `backend/src/routes/rag.routes.ts` | 151 | API endpoints: backfill, sources, retry, status |
| `frontend/src/client/store/api/ragApi.ts` | 25 | RTK Query API: getRagStatus endpoint |
| `backend/src/prompts/promptBuilder.ts` | 547 | Prompt assembly with RAG context |
| `backend/src/lib/orchestrator/memory/memoryLayer.ts` | 250 | Memory layer with conversation context |
| `backend/src/lib/legal/retrieval/index.ts` | 36 | Legal retrieval types and interface |
| `backend/src/lib/legal/citations/index.ts` | 30 | Citation normalization and formatting |
| `backend/src/db/schema.ts` | 2525 | Database schema: ragCollections, ragSourceIndexEntries, evidenceSourceRefs |
| `backend/src/lib/chatTools.ts` | 10147 | search_sources tool definition and execution |
| `backend/src/lib/tools/executors.ts` | 1970 | Tool executors: searchDocuments, workspaceRagRetrieve |

### Key Database Tables

| Table | Purpose |
|-------|---------|
| `rag_collections` | Stores collection metadata (scope, name, owner) |
| `rag_source_index_entries` | Tracks document indexing status and errors |
| `evidence_source_refs` | Stores citation evidence for search results |
| `generated_citation_evidence` | Links citations to generated documents |

### Key API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/rag/backfill` | POST | Re-index all user documents |
| `/rag/sources` | GET | List indexed sources |
| `/rag/sources/:id/retry` | POST | Retry failed indexing |
| `/rag/status` | GET | Get indexing status counts |

### Key Tool Definitions

| Tool | Purpose |
|------|---------|
| `search_sources` | Search indexed documents using RAG |
| `search_documents` | Search personal documents |
| `workspaceRagRetrieve` | Search workspace documents |
