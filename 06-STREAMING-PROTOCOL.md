# Streaming Protocol

## 1. What This Document Is

This document describes every event that flows between the Mike backend and frontend during a streaming response. When you ask Mike a question, the backend does not wait until it has a complete answer. Instead, it sends a series of small events as they happen — a thought forms, a tool gets called, a citation appears, a document is created — and the frontend displays each piece in real time. This file is the complete reference for what those events are, what they contain, and how the frontend reacts to them.

## 2. Why Streaming Matters

A legal AI assistant can take anywhere from two seconds to two minutes to produce a full response. Without streaming, the user stares at a blank screen or spinner the entire time. With streaming, the user sees the AI thinking within a fraction of a second, watches it call tools, sees citations appear as they are found, and watches a document materialize paragraph by paragraph. This transforms the user experience from "waiting for a black box" to "watching an assistant work." It builds trust, reduces perceived latency, and lets the user interrupt early if the AI is going in the wrong direction.

## 3. The SSE Architecture

Mike uses Server-Sent Events (SSE) for all real-time communication between backend and frontend. Here is how it works:

**Backend opens SSE connection.** The frontend sends a regular HTTP POST request to the backend with the user's message. The backend responds with `Content-Type: text/event-stream` instead of a normal JSON response. This keeps the connection open.

**Frontend listens for events.** The browser's `EventSource` API (or a `fetch`-based reader) listens on the open connection. Each event arrives as a line prefixed with `data: ` followed by a JSON payload.

**Events flow one-way (server to client).** SSE is unidirectional. The server pushes events to the client. If the user wants to send another message, they open a new stream or abort the current one.

**Heartbeat keeps the connection alive.** Proxies, load balancers, and browsers will close idle HTTP connections. The backend sends a heartbeat comment (`: heartbeat\n\n`) every 10 seconds (configurable via `ASSISTANT_HEARTBEAT_INTERVAL_MS`) to keep the connection open.

**Abort controller for cancellation.** Both sides can end the stream. The frontend creates an `AbortController` and passes its signal. When the user navigates away, clicks cancel, or the component unmounts, the abort signal fires, and the backend stops processing.

Key backend files:

| File | Purpose |
|------|---------|
| `backend/src/lib/sseStream.ts` | Creates and manages SSE sessions — headers, heartbeat, write, abort |
| `backend/src/lib/assistant-runtime/sse-heartbeat.ts` | Standalone heartbeat class used by the mixer |
| `backend/src/lib/assistant-runtime/stream-sanitizer.ts` | Strips leaked tool-call XML from LLM output |
| `backend/src/lib/assistant-runtime/mixer.ts` | Orchestrates LLM streaming with heartbeat and sanitization |
| `backend/src/lib/chatOrchestrator.ts` | Main orchestrator that writes SSE events to the response |

## 4. Connection Lifecycle

Every streaming interaction follows this lifecycle:

```
1. Client sends POST request
   POST /api/chat/stream
   Body: { messages: [...], chatId: "...", model: "..." }

2. Server creates SSE session
   - Sets headers: Content-Type: text/event-stream, Cache-Control: no-cache,
     Connection: keep-alive, X-Accel-Buffering: no
   - Creates AbortController
   - Registers cleanup listeners on req.aborted, res.close, res.error

3. Server sends heartbeat every 10 seconds
   ": heartbeat\n\n" (comment line, not a data event)

4. Server sends events as they occur
   data: {"type":"content_delta","text":"Hello"}\n\n
   data: {"type":"tool_start","toolCallId":"tc-1","toolName":"search"}\n\n
   data: {"type":"tool_complete","toolCallId":"tc-1","output":{...}}\n\n

5. Server sends "done" or "stream_end" event
   data: {"type":"done"}\n\n

6. Connection closes
   - Server calls res.end()
   - AbortController fires cleanup
   - Heartbeat interval is cleared
   - Frontend sets isStreaming: false
```

The backend also sends a `chat_id` event early in the stream so the frontend can associate the response with a conversation:

```
data: {"type":"chat_id","chatId":"abc-123"}\n\n
```

## 5. Event Types — Complete Reference

### content_delta / text_delta

**When sent:** The LLM is generating text. Sent for every chunk of the response.

**Payload:**
```json
{ "type": "content_delta", "text": "The court held that" }
```
or
```json
{ "type": "text_delta", "delta": "the contract is enforceable" }
```

**Frontend handling:** Appends `text` (or `delta`) to the current message content. The UI re-renders the message with the new text, creating a typewriter effect.

**UI behavior:** Text appears word by word in the chat bubble.

---

### reasoning_delta / reasoning_start / reasoning_chunk / reasoning_complete

**When sent:** The model is using extended thinking (chain-of-thought). These events trace the reasoning process.

**Payload:**
```json
{ "type": "reasoning_delta", "delta": "First, I need to consider..." }
```
```json
{ "type": "reasoning_start" }
```
```json
{ "type": "reasoning_chunk", "content": "The precedent suggests..." }
```
```json
{ "type": "reasoning_complete" }
```

**Frontend handling:** Appends reasoning text to `state.reasoning`. The `reasoning_start` and `reasoning_complete` events bracket the reasoning phase.

**UI behavior:** A collapsible "Reasoning" section appears above the response, showing the AI's thought process.

---

### thinking_delta / thinking_done

**When sent:** Agentic runtime signals it is processing. Used in the assistant runtime mixer path.

**Payload:**
```json
{ "type": "thinking_delta", "payload": { "text": "Analyzing document structure..." } }
```
```json
{ "type": "thinking_done", "payload": { "durationMs": 1200, "tokenCount": 340 } }
```

**Frontend handling:** Shows a thinking animation or indicator.

**UI behavior:** A pulsing dot or "Thinking..." label appears while the AI processes.

---

### plan_ready

**When sent:** The task planner has created an execution plan for a multi-step agentic task.

**Payload:**
```json
{
  "type": "plan_ready",
  "payload": {
    "planId": "plan-abc",
    "title": "Due Diligence Review",
    "summary": "Review 5 documents for compliance issues",
    "phases": [
      { "id": "phase-1", "title": "Document Classification", "description": "Classify each document type" },
      { "id": "phase-2", "title": "Compliance Analysis", "description": "Run compliance checks" }
    ]
  }
}
```

**Frontend handling:** Displays a planning card showing the steps the AI will take.

**UI behavior:** An expandable card appears with the plan title and numbered phases.

---

### approval_required

**When sent:** The agentic runtime needs user approval before executing a plan.

**Payload:**
```json
{
  "type": "approval_required",
  "payload": {
    "planId": "plan-abc",
    "prompt": "I plan to review 5 documents. May I proceed?",
    "options": ["Approve", "Modify Plan", "Cancel"]
  }
}
```

**Frontend handling:** Renders an approval card with action buttons.

**UI behavior:** A card with the plan summary and Approve/Modify/Cancel buttons appears. The stream pauses until the user responds.

---

### question_required

**When sent:** The AI needs clarification from the user before continuing.

**Payload:**
```json
{
  "type": "question_required",
  "payload": {
    "questionId": "q-1",
    "prompt": "Which jurisdiction should I apply?",
    "choices": ["California", "New York", "Federal"],
    "allowFreeText": true
  }
}
```

**Frontend handling:** Shows a clarification card with selectable options or a text input.

**UI behavior:** A card with the question and clickable choices (or a free-text field) appears inline in the conversation.

---

### clarification

**When sent:** Similar to `question_required`, used in the chat orchestrator path.

**Payload:**
```json
{
  "type": "clarification",
  "question": "Should I draft the notice for postal service or email delivery?",
  "options": [
    { "label": "Postal Service", "value": "postal" },
    { "label": "Email", "value": "email" }
  ]
}
```

**Frontend handling:** Renders an interactive clarification card.

**UI behavior:** Buttons or a dropdown appear for the user to select an option.

---

### tool_start / tool_call / tool_call_start

**When sent:** The AI has decided to call a tool (search, document creation, compliance check, etc.).

**Payload:**
```json
{
  "type": "tool_start",
  "toolCallId": "tc-001",
  "toolName": "search_documents",
  "input": { "query": "force majeure clauses", "scope": "project" }
}
```

Legacy format (deprecated, still handled):
```json
{
  "type": "tool_call",
  "id": "tc-001",
  "tool": "search_documents",
  "input": { "query": "force majeure clauses" }
}
```

**Frontend handling:** Adds a tool card to `state.tools` with `status: 'running'`. The card shows the tool name and a spinner.

**UI behavior:** A collapsible card appears showing "Searching documents..." with an animated spinner.

---

### tool_streaming

**When sent:** A tool is producing incremental output (e.g., streaming a document generation).

**Payload:**
```json
{
  "type": "tool_streaming",
  "toolCallId": "tc-001",
  "chunk": "Generating section 1 of 5..."
}
```

**Frontend handling:** Updates the tool card's output with the new chunk and sets status to `'streaming'`.

**UI behavior:** The tool card's content updates in real time.

---

### tool_complete / tool_result / tool_call_complete

**When sent:** A tool has finished executing.

**Payload:**
```json
{
  "type": "tool_complete",
  "toolCallId": "tc-001",
  "toolName": "search_documents",
  "output": { "results": [...], "count": 12 }
}
```

Legacy format:
```json
{
  "type": "tool_result",
  "tool_call_id": "tc-001",
  "tool": "search_documents",
  "output": { "results": [...] },
  "status": "complete"
}
```

**Frontend handling:** Updates the tool card with final output and sets status to `'complete'`. The spinner stops.

**UI behavior:** The tool card shows a checkmark and the result summary.

---

### tool_error / tool_call_error

**When sent:** A tool call failed.

**Payload:**
```json
{
  "type": "tool_error",
  "toolCallId": "tc-001",
  "toolName": "search_documents",
  "error": "Index temporarily unavailable"
}
```

**Frontend handling:** Updates the tool card status to `'error'` and stores the error message.

**UI behavior:** The tool card shows a red error icon and the error message.

---

### phase_start

**When sent:** The agentic runtime begins a new execution phase.

**Payload:**
```json
{
  "type": "phase_start",
  "payload": {
    "phase": "execution",
    "workstreamId": "ws-1",
    "sectionId": "sec-3"
  }
}
```

**Frontend handling:** Highlights the current phase in the plan card.

**UI behavior:** The active phase indicator moves to the next step.

---

### subagent_start / subagent_done

**When sent:** A sub-agent is spawned for parallel work, and when it completes.

**Payload:**
```json
{
  "type": "subagent_start",
  "payload": {
    "subagentId": "sa-1",
    "title": "Clause Analysis",
    "workstreamId": "ws-1"
  }
}
```
```json
{
  "type": "subagent_done",
  "payload": {
    "subagentId": "sa-1",
    "workstreamId": "ws-1",
    "success": true,
    "summary": "Found 3 risk clauses"
  }
}
```

**Frontend handling:** Shows a sub-agent card with a spinner, then updates with the result.

**UI behavior:** A nested card appears under the plan showing the sub-agent's work.

---

### citations

**When sent:** The AI references legal sources, cases, or document sections.

**Payload:**
```json
{
  "type": "citations",
  "citations": [
    {
      "text": "See Section 4.2 of the Agreement",
      "source_type": "document",
      "document_id": "doc-123",
      "filename": "Master_Agreement.pdf",
      "page": 12,
      "deep_link_url": "/documents/doc-123#page=12"
    }
  ]
}
```

**Frontend handling:** Appends citations to `state.citations`.

**UI behavior:** Inline citation links appear in the response text. Clicking opens the referenced document.

---

### source_result / source_results

**When sent:** Document or web search results are returned.

**Payload:**
```json
{
  "type": "source_result",
  "rank": 1,
  "filename": "NDA_Template.docx",
  "source_type": "document",
  "text": "Non-disclosure agreement between parties...",
  "document_url": "/download/doc-456",
  "deep_link_url": "/documents/doc-456"
}
```

**Frontend handling:** Appends sources to `state.sources`.

**UI behavior:** A "Sources" section appears below the response showing matching documents with links.

---

### web_search_results

**When sent:** A web search completes.

**Payload:**
```json
{
  "type": "web_search_results",
  "query": "force majeure clause enforceability 2024",
  "results": [
    {
      "title": "Force Majeure in Modern Contracts",
      "url": "https://example.com/article",
      "snippet": "Recent case law suggests..."
    }
  ]
}
```

**Frontend handling:** Adds to `state.webSearch` and converts results to source artifacts.

**UI behavior:** Web search results appear in the Sources section.

---

### doc_created_start / doc_created / doc_edited / doc_replicated

**When sent:** A document is being created or has been created.

**Payload:**
```json
{
  "type": "doc_created_start",
  "filename": "Compliance_Report.docx"
}
```
```json
{
  "type": "doc_created",
  "id": "doc-789",
  "filename": "Compliance_Report.docx",
  "download_url": "/download/doc-789",
  "deep_link_url": "/documents/doc-789",
  "indexing_status": "queued",
  "createdAt": "2026-07-14T10:30:00Z"
}
```

**Frontend handling:** `doc_created_start` shows a loading state. `doc_created` adds a document artifact to `state.documents`.

**UI behavior:** A document card appears showing the filename, a download link, and creation status.

---

### doc_read / doc_read_start

**When sent:** The AI is reading a document for context.

**Payload:**
```json
{ "type": "doc_read_start", "filename": "Contract_v3.pdf" }
```
```json
{ "type": "doc_read", "filename": "Contract_v3.pdf" }
```

**Frontend handling:** Shows a brief loading indicator that the AI is reading a document.

**UI behavior:** A subtle "Reading Contract_v3.pdf..." message appears.

---

### compliance_review_start / compliance_review_complete

**When sent:** A compliance review is running or has finished.

**Payload:**
```json
{
  "type": "compliance_review_start",
  "compliance_review_id": "cr-001",
  "document_id": "doc-123",
  "filename": "Service_Agreement.docx"
}
```
```json
{
  "type": "compliance_review_complete",
  "compliance_review_id": "cr-001",
  "score": 87,
  "bottlenecks": [
    { "clause": "Termination", "severity": "high", "message": "Missing notice period" }
  ]
}
```

**Frontend handling:** Adds a compliance artifact to `state.compliance` with status updating from `'running'` to `'complete'`.

**UI behavior:** A compliance card appears with a progress indicator, then shows the score and issues.

---

### tabular_review_start / tabular_review_complete

**When sent:** A tabular data extraction is running or finished.

**Payload:**
```json
{
  "type": "tabular_review_start",
  "review_id": "tr-001",
  "document_id": "doc-456",
  "filename": "Payment_Schedule.xlsx"
}
```
```json
{
  "type": "tabular_review_complete",
  "review_id": "tr-001",
  "row_count": 24,
  "column_count": 6
}
```

**Frontend handling:** Adds a tabular artifact to `state.tabular`.

**UI behavior:** A table card appears showing the extracted data dimensions.

---

### template_wizard_start / template_wizard_lifecycle

**When sent:** A document template wizard starts or changes state.

**Payload:**
```json
{
  "type": "template_wizard_start",
  "wizard_id": "wiz-001",
  "wizard_status": "ACTIVE",
  "template_name": "NDA Template",
  "fields": [...]
}
```
```json
{
  "type": "template_wizard_lifecycle",
  "wizard_id": "wiz-001",
  "wizard_status": "COMPLETED"
}
```

**Frontend handling:** Stores wizard data in `state.wizardData`. Lifecycle events update the status.

**UI behavior:** A wizard card appears showing the template fields to fill. On completion, it shows the generated document.

---

### legal_case_collection

**When sent:** The AI has gathered related legal cases.

**Payload:**
```json
{
  "type": "legal_case_collection",
  "cases": [
    { "title": "Smith v. Jones", "citation": "123 F.3d 456", "relevance": "high" }
  ]
}
```

**Frontend handling:** Stores the collection in `state.legalCaseCollection`.

**UI behavior:** A case collection card appears listing the found cases.

---

### workflow_applied / workflow_event / workflow_phase / workflow_completed

**When sent:** The execution engine runs a workflow.

**Payload:**
```json
{ "type": "workflow_applied", "workflow_id": "wf-1", "title": "Due Diligence" }
```
```json
{ "type": "workflow_phase", "phase": "executing" }
```
```json
{ "type": "workflow_completed", "result": { "status": "success" } }
```

**Frontend handling:** Updates the workflow status indicator.

**UI behavior:** A workflow progress bar or step indicator updates.

---

### artifact_ready

**When sent:** An agentic runtime produces an artifact (document, table, diagram).

**Payload:**
```json
{
  "type": "artifact_ready",
  "payload": {
    "artifactId": "art-001",
    "title": "Risk Heatmap",
    "kind": "diagram",
    "referenceId": "doc-123"
  }
}
```

**Frontend handling:** Shows an artifact card.

**UI behavior:** A card with the artifact title and a preview or download link appears.

---

### draft_ready

**When sent:** A document draft has been generated.

**Payload:**
```json
{
  "type": "draft_ready",
  "payload": {
    "revision": 3,
    "totalTokens": 4200,
    "gapCount": 2,
    "coverageCount": 15,
    "attachmentCount": 1
  }
}
```

**Frontend handling:** Updates the draft status in the UI.

**UI behavior:** A draft card shows revision number, coverage stats, and gaps found.

---

### agent_error

**When sent:** The agentic runtime encounters an error.

**Payload:**
```json
{
  "type": "agent_error",
  "payload": {
    "phase": "execution",
    "attempt": 2,
    "message": "Tool execution timed out"
  }
}
```

**Frontend handling:** Shows an error card with retry option.

**UI behavior:** An error card appears with the phase, attempt count, and error message.

---

### status

**When sent:** System status changes during processing.

**Payload:**
```json
{ "type": "status", "status": "processing", "message": "Analyzing 5 documents..." }
```

**Frontend handling:** Updates a status indicator.

**UI behavior:** A status bar or label shows the current operation.

---

### progress

**When sent:** A long operation reports progress.

**Payload:**
```json
{ "type": "progress", "current": 3, "total": 10, "label": "Reviewing clauses" }
```

**Frontend handling:** Updates a progress bar.

**UI behavior:** A progress bar fills from 30% to completion.

---

### warning

**When sent:** A non-critical issue occurs.

**Payload:**
```json
{ "type": "warning", "message": "Document indexing is delayed", "code": "INDEX_DELAY" }
```

**Frontend handling:** Shows a warning toast notification.

**UI behavior:** A yellow toast appears briefly at the top of the screen.

---

### done / stream_end

**When sent:** The response is complete.

**Payload:**
```json
{ "type": "done" }
```
or
```json
{ "type": "stream_end" }
```

**Frontend handling:** Sets `isStreaming: false`. Triggers the `onComplete` callback. Final message is persisted.

**UI behavior:** The typing indicator disappears. The message is finalized and saved to the conversation.

---

### error

**When sent:** A critical error occurs that stops the stream.

**Payload:**
```json
{ "type": "error", "message": "Model rate limit exceeded", "code": "RATE_LIMIT", "retryable": true }
```

**Frontend handling:** Sets `isStreaming: false`. Calls `onError` with the error. May show a retry button.

**UI behavior:** An error message appears in the chat with a retry option if retryable.

---

### heartbeat

**When sent:** Every 10 seconds as a keep-alive.

**Payload:**
```
: heartbeat\n\n
```
This is an SSE comment (starts with `:`), not a data event. The frontend's EventSource ignores it automatically.

**Frontend handling:** No action needed. The connection stays alive.

**UI behavior:** None. Invisible to the user.

---

### chat_id

**When sent:** Early in the stream, provides the conversation ID.

**Payload:**
```json
{ "type": "chat_id", "chatId": "chat-abc-123" }
```

**Frontend handling:** Stores the chat ID for associating the response.

**UI behavior:** None directly. Used internally for conversation tracking.

---

## 6. Event Ordering

Events arrive in a logical order, though not strictly sequential. Here is the typical flow:

```
heartbeat (repeating every 10s)
  → chat_id
  → thinking_delta / thinking_done
  → reasoning_start → reasoning_chunk (repeating) → reasoning_complete
  → plan_ready (if agentic)
  → approval_required (if approval needed)
  → phase_start
  → tool_start → tool_streaming (optional) → tool_complete
  → tool_start → tool_complete (multiple tools in sequence)
  → content_delta (repeating, interleaved with tool calls)
  → citations
  → source_result / web_search_results
  → doc_created_start → doc_created
  → artifact_ready
  → draft_ready
  → done / stream_end
```

Key rules:
- `chat_id` always arrives first (before any content).
- Heartbeats repeat throughout the entire stream.
- `content_delta` events are interleaved with tool calls — the AI may write text, call a tool, write more text, call another tool.
- `done` or `stream_end` is always the last event.
- `error` can appear at any point and ends the stream.

## 7. Event Aggregation

The frontend does not render each event individually. Instead, it aggregates events into a single message object:

```typescript
interface ChatStreamState {
  content: string        // accumulated from content_delta events
  reasoning: string      // accumulated from reasoning_delta events
  tools: ToolArtifact[]  // built from tool_start/tool_complete events
  sources: SourceArtifact[] // built from source_result events
  citations: CitationArtifact[] // built from citations events
  documents: DocumentArtifact[] // built from doc_created events
  compliance: ComplianceArtifact[] // built from compliance_review events
  tabular: TabularArtifact[] // built from tabular_review events
  webSearch: WebSearchArtifact[] // built from web_search_results events
  isStreaming: boolean   // false when done/error arrives
}
```

Each event type updates one or more fields. The UI re-renders the message component whenever state changes. This means a single chat bubble may show text, tool cards, citations, and document links — all assembled from dozens of individual events.

## 8. Background Streaming

Some streams continue even when the user navigates away from the page that started them. This is managed by the background streams service:

**Compliance streams.** A compliance review may take minutes. The stream is registered in a global `activeStreams` map. Status updates dispatch directly to Redux. When the user navigates back, the UI reads the current status from Redux.

**Chat streams.** Long-running chat responses (document generation, multi-step analysis) use the same pattern. The stream is registered globally, and completion/error callbacks dispatch to Redux.

**Agent job streams.** The `useAgentJobStream` hook connects to `/agent/jobs/{jobId}/stream` using `EventSource`. It handles reconnection with exponential backoff (up to 3 retries). Events dispatch to the `agentSlice` in Redux.

**Navigation survival.** The key insight: callbacks are registered at the service level, not the component level. When a component unmounts (user navigates), the stream keeps running. The `onComplete` callback writes to Redux, and when the user returns to the relevant page, the component reads the updated state.

```typescript
// backgroundStreams.ts — the stream survives component unmount
const abortController = new AbortController()
activeStreams.set(processId, { abortController, processId, type: 'chat' })

streamChat({
  signal: abortController.signal,
  onComplete: () => {
    store.dispatch(updateProcessStatus({ id: processId, status: 'completed' }))
    // This runs even if the originating component is gone
  }
})
```

## 9. Stream Sanitization

The `StreamSanitizer` class cleans LLM output before it reaches the frontend:

**Tool call leaking.** LLMs sometimes output raw tool-call XML (`<function=...>`, `<tool_call>...</tool_call>`). The sanitizer detects these patterns and strips them from the text stream:

```typescript
const LEAK_COMPLETE_PATTERNS = [
  /<function=[^>]*>[\s\S]*?<\/tool_call>/g,
  /<tool_call>[\s\S]*?<\/tool_call>/g,
  /<\|tool_call\|>[\s\S]*?<\|\/tool_call\|>/g,
];
```

**Buffering.** If a tool-call tag starts in one chunk but ends in another, the sanitizer buffers the partial content (up to 2000 characters) until the closing tag arrives, then strips the entire block.

**Configuration.** Sanitization can be disabled by setting `ASSISTANT_STREAM_SANITIZE=false`.

**Message preparation.** Before the LLM is called, `prepareMessagesForModel` cleans the message history:
- Filters empty messages
- Strips reasoning-only parts
- Normalizes JSON-like tool arguments
- Compacts old messages to stay within the model's context budget (default 320,000 characters)

## 10. Heartbeat Mechanism

The heartbeat prevents the connection from being closed by intermediate infrastructure:

**Interval.** Default is 10 seconds, configurable via `ASSISTANT_HEARTBEAT_INTERVAL_MS`. Minimum is 1 second.

**Format.** Heartbeats are SSE comments: `: heartbeat\n\n`. The `:` prefix means the browser's EventSource ignores them — they do not trigger `onmessage`.

**Implementation.** Two heartbeat implementations exist:
- `sseStream.ts`: `setInterval` that writes `": heartbeat\n\n"` to the response
- `sse-heartbeat.ts`: Standalone `SseHeartbeat` class used by the `AssistantRuntimeMixer`

**Lifecycle.** Heartbeat starts when the stream begins and stops when the stream ends (in a `finally` block to guarantee cleanup).

**Timeout detection.** If the client stops receiving heartbeats (network issue), it knows the connection is dead. The `useAgentJobStream` hook triggers reconnection on error.

## 11. Error Handling in Streams

**Connection drops.** The backend listens for `req.aborted`, `res.close`, and `res.error` events. When any fires, the `AbortController` aborts, the heartbeat stops, and the stream is marked closed.

**Server errors.** If an unhandled error occurs during streaming, the backend sends an error event:
```json
{ "type": "error", "message": "Internal server error", "code": "INTERNAL" }
```

**Timeout.** If the LLM call hangs, the `AbortController` signal eventually fires (if the client disconnected) or the backend logs a timeout error.

**Client-side reconnection.** The `useAgentJobStream` hook implements exponential backoff:
- Base delay: 1 second
- Max retries: 3
- Delay formula: `1000 * 2^retryCount`
- After 3 failures, `hasExhaustedRetries` becomes true

**Abort handling.** When a user navigates away, the frontend aborts the stream. The backend detects the aborted request and stops processing. The `AbortError` is not treated as a real error — the stream status is kept as "running" in Redux until the user returns.

## 12. Frontend State Management

**Redux slices.** Stream state lives in several Redux slices:
- `chatSlice`: Current message content, streaming status
- `agentSlice`: Agent job events, plan state
- `pendingProcessesSlice`: Background process status
- `backgroundChatSlice`: Background chat completion status

**React contexts.** The `useChatStreamEvents` hook manages per-message state via `useRef`. It does not use Redux directly for streaming state — it uses a callback-based approach:

```typescript
const { resetState, handleEvent, createMessageFromState } = useChatStreamEvents({
  onStateChange: (updater) => setMessages(prev => /* update with updater */),
  onComplete: (finalState) => { /* persist message */ },
  onError: (error) => { /* show error */ },
})
```

**Hook management.** Each streaming component creates its own `useChatStreamEvents` instance. The hook maintains a `stateRef` (not React state) for performance — it avoids re-renders on every chunk. State updates are batched via the `onStateChange` callback.

**Cleanup on unmount.** When a component unmounts:
1. The `AbortController` fires, aborting the fetch
2. Event listeners are removed
3. Heartbeat stops on the backend
4. Redux state is preserved for re-mount

## 13. Stream Cancellation

**User abort.** The user clicks "Stop" or "Cancel." The frontend calls `abortController.abort()`. The backend detects the aborted request and stops processing immediately.

**Timeout abort.** If a stream takes too long (configurable per endpoint), the backend aborts the LLM call via the `AbortSignal`.

**Navigation abort.** When the user navigates to a different page, the component unmounts and aborts the stream. For background streams, the abort is registered but the stream continues.

**Cleanup.** The `SseStreamSession.dispose()` method:
1. Clears the heartbeat interval
2. Removes event listeners from `req` and `res`
3. Fires the `AbortController` abort

The session's `end()` method additionally calls `res.end()` to close the HTTP connection.

## 14. Performance Optimization

**Chunked transfer.** SSE uses HTTP chunked transfer encoding. Each event is a separate chunk, so the browser can process it immediately without waiting for the full response.

**Gzip compression.** The `X-Accel-Buffering: no` header disables nginx buffering for SSE connections, ensuring events flow through immediately.

**Event batching.** Some events are batched on the backend. For example, the `citations` event contains an array of citations rather than one event per citation.

**Debouncing.** The frontend does not re-render on every `content_delta` chunk. The `onStateChange` callback can batch updates. The `stateRef` pattern avoids React state overhead during high-frequency streaming.

**Content budget.** The backend enforces an input character budget (default 320,000 chars). Messages exceeding this are compacted — older messages are summarized into a single placeholder to stay within the model's context window.

**Tool loop stopping.** The backend has four stop conditions to prevent infinite tool loops:
1. Token budget exceeded (model-specific)
2. Repeated identical tool calls (3+ consecutive)
3. Same tool loop detected (8+ consecutive identical calls)
4. Stalled tool loop (no progress for 4+ steps)

## 15. Testing Streams

**Mock SSE.** For unit tests, mock the `write` function:
```typescript
const chunks: string[] = []
const write = (chunk: string) => { chunks.push(chunk) }
// Then assert on chunks after the stream completes
```

**Unit tests.** Test individual components:
- `StreamSanitizer.process()` — verify tool-call stripping
- `SseHeartbeat` — verify start/stop lifecycle
- `prepareMessagesForModel` — verify compaction and normalization
- `shouldStop` — verify stop conditions trigger correctly

**Integration tests.** Test the full stream by:
1. Creating an Express request/response pair
2. Calling `createSseStreamSession`
3. Writing events through the session
4. Parsing the response chunks
5. Asserting on event types and payloads

**Frontend tests.** Test `useChatStreamEvents` by:
1. Calling `handleEvent` with mock data
2. Asserting on the state changes via `getState()`
3. Verifying `createMessageFromState` produces correct message objects

## 16. Complete Event Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER ACTION                                 │
│                     Types message, clicks Send                       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      FRONTEND                                        │
│  1. POST /api/chat/stream with messages                             │
│  2. Create AbortController                                          │
│  3. Register useChatStreamEvents hook                               │
│  4. Start reading SSE response                                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      BACKEND                                         │
│  1. Validate request, create SSE session                            │
│  2. Set response headers (text/event-stream)                        │
│  3. Start heartbeat (every 10s)                                     │
│  4. Build context (messages, tools, system prompt)                  │
│  5. Call LLM via OpenAI/Gemini streaming API                        │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │ heartbeat│    │thinking  │    │content   │
        │ (10s)    │    │_delta    │    │_delta    │
        └──────────┘    └──────────┘    └──────────┘
              │                │                │
              │                ▼                │
              │         ┌──────────┐           │
              │         │reasoning │           │
              │         │_chunk    │           │
              │         └──────────┘           │
              │                │                │
              │                ▼                │
              │         ┌──────────┐           │
              │         │plan_ready│           │
              │         └──────────┘           │
              │                │                │
              │                ▼                │
              │         ┌──────────┐           │
              │         │tool_start│           │
              │         └──────────┘           │
              │                │                │
              │                ▼                │
              │         ┌──────────┐           │
              │         │tool_     │           │
              │         │complete  │           │
              │         └──────────┘           │
              │                │                │
              │                ▼                │
              │         ┌──────────┐           │
              │         │citations │           │
              │         └──────────┘           │
              │                │                │
              │                ▼                │
              │         ┌──────────┐           │
              │         │doc_created│          │
              │         └──────────┘           │
              │                │                │
              │                ▼                │
              │         ┌──────────┐           │
              │         │ done     │◄──────────┘
              │         └──────────┘
              │                │
              ▼                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      FRONTEND                                        │
│  1. Assemble final message from accumulated state                    │
│  2. Persist to conversation                                          │
│  3. Set isStreaming: false                                           │
│  4. Render final message with tools, citations, documents           │
└─────────────────────────────────────────────────────────────────────┘
```

## 17. Event Reference Table

| Event Name | Payload Key Fields | Frequency | Frontend Handler | UI Effect |
|---|---|---|---|---|
| `content_delta` / `text_delta` | `text` or `delta` | Every LLM chunk (10-50/sec) | Append to message content | Typewriter text effect |
| `reasoning_delta` / `reasoning_chunk` | `delta`, `text`, or `content` | During reasoning phase | Append to reasoning text | Collapsible reasoning section |
| `reasoning_start` | (empty) | Once per reasoning phase | Start reasoning section | Open reasoning panel |
| `reasoning_complete` | (empty) | Once per reasoning phase | Close reasoning section | Close reasoning panel |
| `thinking_delta` | `payload.text` | During thinking | Show thinking indicator | Pulsing "Thinking..." |
| `thinking_done` | `payload.durationMs`, `payload.tokenCount` | Once | Hide thinking indicator | Remove thinking animation |
| `plan_ready` | `payload.planId`, `payload.phases[]` | Once per plan | Render plan card | Expandable plan card |
| `approval_required` | `payload.planId`, `payload.prompt`, `payload.options` | Once if needed | Render approval buttons | Approve/Modify/Cancel UI |
| `question_required` | `payload.questionId`, `payload.prompt`, `payload.choices` | Once if needed | Render clarification card | Question with selectable options |
| `clarification` | `question`, `options[]` | Once if needed | Render clarification card | Interactive choice buttons |
| `tool_start` / `tool_call` | `toolCallId`, `toolName`, `input` | Per tool call | Add tool card (running) | Spinner + tool name card |
| `tool_streaming` | `toolCallId`, `chunk` | During tool execution | Update tool card output | Real-time tool output |
| `tool_complete` / `tool_result` | `toolCallId`, `output` | Per tool call | Update tool card (complete) | Checkmark + result summary |
| `tool_error` | `toolCallId`, `error` | On tool failure | Update tool card (error) | Red error icon + message |
| `phase_start` | `payload.phase`, `payload.workstreamId` | Per phase | Highlight active phase | Phase indicator update |
| `subagent_start` | `payload.subagentId`, `payload.title` | Per sub-agent | Show sub-agent card | Nested spinner card |
| `subagent_done` | `payload.subagentId`, `payload.success` | Per sub-agent | Update sub-agent card | Result summary |
| `citations` | `citations[]` | When sources found | Add citation artifacts | Inline citation links |
| `source_result` / `source_results` | `rank`, `filename`, `text` | Per search result | Add source artifacts | Sources section with links |
| `web_search_results` | `query`, `results[]` | Per web search | Add web search + sources | Web results in Sources |
| `doc_created_start` | `filename` | Before doc creation | Show creating state | "Creating document..." |
| `doc_created` / `doc_edited` | `id`, `filename`, `download_url` | Per document | Add document artifact | Document card with link |
| `doc_read_start` | `filename` | Before doc read | Show reading indicator | "Reading filename..." |
| `doc_read` | `filename` | After doc read | Hide reading indicator | Remove loading text |
| `compliance_review_start` | `compliance_review_id`, `filename` | Per review | Add compliance card (running) | Progress indicator |
| `compliance_review_complete` | `compliance_review_id`, `score`, `bottlenecks` | Per review | Update compliance card | Score + issue list |
| `tabular_review_start` | `review_id`, `filename` | Per review | Add tabular card (running) | Progress indicator |
| `tabular_review_complete` | `review_id`, `row_count`, `column_count` | Per review | Update tabular card | Table dimensions |
| `template_wizard_start` | `wizard_id`, `fields[]` | Once per wizard | Store wizard data | Wizard form card |
| `template_wizard_lifecycle` | `wizard_id`, `wizard_status` | On status change | Update wizard status | Status badge update |
| `legal_case_collection` | `cases[]` | Once per collection | Store case collection | Case list card |
| `workflow_applied` | `workflow_id`, `title` | Once per workflow | Show workflow indicator | Workflow badge |
| `workflow_phase` | `phase` | Per phase | Update workflow indicator | Phase label |
| `workflow_completed` | `result` | Once | Show completion | Checkmark badge |
| `artifact_ready` | `payload.artifactId`, `payload.kind` | Per artifact | Show artifact card | Artifact preview card |
| `draft_ready` | `payload.revision`, `payload.gapCount` | Once per draft | Update draft status | Draft stats card |
| `status` | `status`, `message` | As needed | Update status indicator | Status bar text |
| `progress` | `current`, `total`, `label` | During long ops | Update progress bar | Filling progress bar |
| `warning` | `message`, `code` | On non-critical issue | Show warning toast | Yellow toast notification |
| `chat_id` | `chatId` | Once, early in stream | Store chat ID | None (internal) |
| `done` / `stream_end` | (empty or `summary`) | Once, always last | Set isStreaming: false | Finalize message |
| `error` | `message`, `code`, `retryable` | On critical error | Set isStreaming: false, show error | Error message + retry button |
| `heartbeat` | (SSE comment, no data) | Every 10 seconds | Ignored | None (invisible) |
