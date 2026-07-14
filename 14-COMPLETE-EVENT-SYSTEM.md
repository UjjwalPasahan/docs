# Complete Event System

## 1. What This Document Is

This document describes every event that exists in the Mike legal AI platform. An event is a small piece of data that one part of the system sends to another part to say "something happened." Events are the nervous system of Mike. They carry information from the backend AI engine to the frontend user interface, from one internal service to another, and from long-running background jobs back to the browser. This file is the complete reference for what every event is, when it fires, what data it carries, and how the system reacts to it. If you need to understand how Mike communicates with itself and with the user, this is the document to read.

## 2. What Is an Event

An event is like a notification. Think of it as a small message that says "something just happened." When the AI model starts thinking, an event is created. When a tool finishes running, an event is created. When a document gets created, an event is created. Something happens, an event is created, and something else reacts to it. This is how different parts of Mike talk to each other without being tightly connected.

Events help parts of the system communicate without knowing about each other directly. The backend does not need to know which frontend component is listening. The frontend does not need to know which backend service produced the event. They only need to agree on the event format. This loose coupling makes the system flexible and easy to extend.

There are three categories of events in Mike. Server-Sent Events (SSE) flow from the backend to the frontend over an HTTP connection. Internal backend events flow between services inside the server using an event bus. Frontend events flow within the browser using Redux state updates and React hooks. Together, these three layers form a complete event-driven architecture that powers the entire platform.

## 3. Event System Architecture

The event system follows a unidirectional flow from backend to frontend, with internal event buses connecting services on each side:

```
+-----------------------------------------------------------------------------------+
|                                BACKEND SERVICES                                   |
|                                                                                   |
|  +------------------+    +------------------+    +------------------+             |
|  |   LLM Streaming  |    |   Agent Planner  |    |  Sub-Agent Exec  |             |
|  |   (Mixer/AI)     |    |   (Planning)     |    |  (SubTasks)      |             |
|  +--------+---------+    +--------+---------+    +--------+---------+             |
|           |                      |                      |                          |
|           v                      v                      v                          |
|  +---------------------------------------------------------------------------------+
|  |                         BACKEND EVENT BUSES                                    |
|  |                                                                                |
|  |  +---------------------------+    +---------------------------+                |
|  |  |   agentEventBus.ts        |    |   agentJobEventBus.ts     |                |
|  |  |   (AgentStreamEvent)      |    |   (AgentJobEvent)         |                |
|  |  |   EventEmitter-based      |    |   EventEmitter-based      |                |
|  |  |   Channel: agent-job:ID   |    |   Channel: agent-job:ID   |                |
|  |  +---------------------------+    +---------------------------+                |
|  +---------------------------------------------------------------------------------+
|           |                                                                      |
|           v                                                                      |
|  +------------------+    +------------------+    +------------------+             |
|  |   SSE Formatter  |    |   Heartbeat      |    |  Stream          |             |
|  |   (events.ts)    |    |   (10s keepalive)|    |  Sanitizer       |             |
|  +--------+---------+    +--------+---------+    +--------+---------+             |
+-----------|------------------------------------------------------------------------+
            |
            v
+-----------------------------------------------------------------------------------+
|                         SSE TRANSPORT LAYER                                       |
|                                                                                   |
|   POST /chat  --->  Content-Type: text/event-stream  --->  data: JSON\n\n        |
|                                                                 |
|   Heartbeat: ": heartbeat\n\n"  (every 10 seconds)             |
|   Sanitizer: strips leaked tool-call XML from LLM output       |
+-------------------------------------------------------------------|---------------+
            |
            v
+-----------------------------------------------------------------------------------+
|                           FRONTEND EVENT HANDLING                                 |
|                                                                                   |
|  +---------------------------+    +---------------------------+                    |
|  |  useChatStreamEvents.ts   |    |  useAgentJobStream.ts     |                    |
|  |  (499 lines)              |    |  (204 lines)              |                    |
|  |  Handles all SSE types    |    |  Handles agent job SSE    |                    |
|  |  Updates ChatStreamState  |    |  Dispatches to Redux      |                    |
|  +---------------------------+    +---------------------------+                    |
|                                                                                   |
|  +---------------------------+    +---------------------------+                    |
|  |  backgroundStreams.ts     |    |  backgroundChatSlice.ts   |                    |
|  |  (293 lines)              |    |  (89 lines)               |                    |
|  |  Global stream manager    |    |  Background chat state    |                    |
|  |  Survives navigation      |    |  Messages + status        |                    |
|  +---------------------------+    +---------------------------+                    |
|                                                                                   |
|  +---------------------------+    +---------------------------+    +------------+  |
|  |  pendingProcessesSlice.ts |    |  agentSlice.ts            |    |  uiSlice   |  |
|  |  (115 lines)              |    |  (46 lines)               |    |  (143 ln)  |  |
|  |  Process tracking         |    |  Agent job state          |    |  Toasts    |  |
|  |  Running/completed/error  |    |  Plan + stream events     |    |  Modals    |  |
|  +---------------------------+    +---------------------------+    +------------+  |
+-----------------------------------------------------------------------------------+
            |
            v
+-----------------------------------------------------------------------------------+
|                            UI RENDERING                                           |
|                                                                                   |
|  Chat bubbles, tool cards, document links, compliance badges,                    |
|  progress indicators, toast notifications, modal dialogs                         |
+-----------------------------------------------------------------------------------+
```

## 4. SSE Event Types (Complete Reference)

Server-Sent Events (SSE) are the primary way the backend sends real-time information to the frontend. When the user sends a message, the backend opens an SSE connection and streams events as they happen. Each event is a JSON object sent on a line prefixed with `data: `. The frontend reads these events and updates the UI accordingly.

The SSE event types are defined in `backend/src/lib/agentic/events.ts` as the `AgenticSSEEvent` union type. There are 13 distinct agentic SSE event types. The chat streaming system in `chatApi.ts` defines additional event types for the conversational chat flow.

---

### thinking_delta

**When it fires:** The AI model is producing extended thinking output. This happens when the model uses chain-of-thought reasoning before answering. Each chunk of thinking text triggers this event.

**Payload structure:**
```json
{
  "type": "thinking_delta",
  "payload": {
    "text": "Let me analyze the contract clauses..."
  }
}
```

**Frontend handler:** The `useChatStreamEvents` hook listens for `reasoning_delta` and `thinking_delta` events. The handler appends the `text` (or `delta`) to the `reasoning` field in `ChatStreamState`.

**UI behavior:** A collapsible reasoning panel appears in the chat bubble, showing the AI's internal thought process. Text appears incrementally, similar to a typewriter effect.

---

### thinking_done

**When it fires:** The AI model has finished producing its extended thinking output. This marks the end of the thinking phase.

**Payload structure:**
```json
{
  "type": "thinking_done",
  "payload": {
    "durationMs": 3500,
    "tokenCount": 256
  }
}
```

**Frontend handler:** The frontend uses this event to finalize the reasoning panel state. The duration and token count can be displayed for transparency.

**UI behavior:** The reasoning panel stops updating and shows a final state. A small badge may appear showing how long the AI spent thinking.

---

### plan_ready

**When it fires:** The agent planner has created a multi-phase execution plan for a complex task. This happens after the agent analyzes the user's request and decides how to break it into steps.

**Payload structure:**
```json
{
  "type": "plan_ready",
  "payload": {
    "planId": "plan-abc-123",
    "title": "Due Diligence Review",
    "summary": "3-phase review across corporate, IP, and commercial practice areas",
    "phases": [
      {
        "id": "phase-0",
        "title": "Corporate Review",
        "description": "Review corporate structure and governance documents"
      },
      {
        "id": "phase-1",
        "title": "IP Review",
        "description": "Analyze intellectual property portfolio"
      },
      {
        "id": "phase-2",
        "title": "Commercial Review",
        "description": "Review commercial contracts and obligations"
      }
    ]
  }
}
```

**Frontend handler:** The `useAgentJobStream` hook dispatches `addStreamEvent` to Redux. The agent slice stores the plan. The `useAgentJobStream` normalizer converts backend plan format to frontend `AgentPlan` format with steps.

**UI behavior:** A plan card appears showing all phases with their titles and descriptions. Each phase has a status indicator (pending, running, completed, failed). The user can review the plan before approving execution.

---

### approval_required

**When it fires:** The agent needs user approval before proceeding with execution. This is a gating event that pauses the workflow until the user responds.

**Payload structure:**
```json
{
  "type": "approval_required",
  "payload": {
    "planId": "plan-abc-123",
    "prompt": "The agent wants to modify 3 documents. Do you approve?",
    "options": ["approve", "reject", "modify"]
  }
}
```

**Frontend handler:** The frontend displays a modal or inline approval card. The user's response is sent back to the backend via a REST endpoint.

**UI behavior:** An approval dialog appears with the plan details and action buttons. The workflow is paused until the user approves, rejects, or requests modifications.

---

### question_required

**When it fires:** The agent needs clarification from the user before it can proceed. This happens when the request is ambiguous or when the agent needs additional information.

**Payload structure:**
```json
{
  "type": "question_required",
  "payload": {
    "questionId": "q-001",
    "prompt": "Which jurisdiction should govern the contract review?",
    "choices": ["England & Wales", "New York", "Singapore"],
    "allowFreeText": true
  }
}
```

**Frontend handler:** The frontend displays a question card with the prompt and available choices. The user's answer is sent back to the backend.

**UI behavior:** A clarification card appears in the chat with the question and selectable options. The user can either pick a choice or type a free-text answer.

---

### phase_start

**When it fires:** A new phase of the agent workflow is beginning. This happens when the agent moves from one practice area or step to the next.

**Payload structure:**
```json
{
  "type": "phase_start",
  "payload": {
    "phase": "execution",
    "workstreamId": "ws-corporate",
    "sectionId": "section-governance"
  }
}
```

**Frontend handler:** The frontend updates the plan progress indicator to show which phase is currently active.

**UI behavior:** The corresponding phase in the plan card highlights as "running." A progress bar may update to show overall completion percentage.

---

### subagent_start

**When it fires:** A sub-agent (specialized worker) has been spawned to handle a specific task. This happens when the main agent delegates work to a focused specialist.

**Payload structure:**
```json
{
  "type": "subagent_start",
  "payload": {
    "subagentId": "sa-ip-review",
    "title": "IP Portfolio Analysis",
    "workstreamId": "ws-intellectual-property"
  }
}
```

**Frontend handler:** The frontend creates a new entry in the activity trace showing the sub-agent has started.

**UI behavior:** A new card or row appears in the activity panel showing the sub-agent's title and a running indicator.

---

### subagent_done

**When it fires:** A sub-agent has completed its work. The result may be a success or failure.

**Payload structure:**
```json
{
  "type": "subagent_done",
  "payload": {
    "subagentId": "sa-ip-review",
    "workstreamId": "ws-intellectual-property",
    "success": true,
    "summary": "Reviewed 12 IP-related documents. Found 3 amber and 1 red issues."
  }
}
```

**Frontend handler:** The frontend updates the activity trace to show the sub-agent completed. The summary is displayed in the UI.

**UI behavior:** The sub-agent card updates from "running" to "completed" (green checkmark) or "failed" (red X). The summary text appears below the title.

---

### tool_start

**When it fires:** The AI is calling a tool (function). This happens when the model decides it needs to use an external capability like searching documents, reading a file, or running a compliance check.

**Payload structure:**
```json
{
  "type": "tool_start",
  "payload": {
    "toolId": "tc-001",
    "toolName": "searchDocuments",
    "workstreamId": "ws-corporate"
  }
}
```

**Frontend handler:** The `useChatStreamEvents` hook handles `tool_start` events by adding a new entry to the `tools` array in `ChatStreamState` with status `"running"`.

**UI behavior:** A tool card appears in the chat showing the tool name and a spinning indicator. The card shows "Running: searchDocuments" with a loading animation.

---

### tool_done

**When it fires:** The AI has finished using a tool. The result is available.

**Payload structure:**
```json
{
  "type": "tool_done",
  "payload": {
    "toolId": "tc-001",
    "toolName": "searchDocuments",
    "success": true,
    "outputSummary": "Found 5 relevant documents matching the query"
  }
}
```

**Frontend handler:** The `useChatStreamEvents` hook updates the tool entry in the `tools` array, setting its status to `"complete"` and attaching the output.

**UI behavior:** The tool card updates to show a green checkmark. The output summary appears in a collapsible section. The user can expand it to see full details.

---

### artifact_ready

**When it fires:** The agent has produced a deliverable (document, table, diagram, etc.). This is a significant output that the user will want to review or download.

**Payload structure:**
```json
{
  "type": "artifact_ready",
  "payload": {
    "artifactId": "art-001",
    "title": "Due Diligence Report",
    "kind": "document",
    "referenceId": "doc-abc-123"
  }
}
```

**Frontend handler:** The frontend creates a clickable link or card that lets the user open or download the artifact.

**UI behavior:** An artifact card appears with the title and type icon (document, table, diagram). Clicking it opens the artifact in a new tab or downloads it.

---

### draft_ready

**When it fires:** A document draft has been generated or updated. This is specifically for drafting operations where the AI produces a document revision.

**Payload structure:**
```json
{
  "type": "draft_ready",
  "payload": {
    "revision": 3,
    "totalTokens": 4500,
    "gapCount": 2,
    "coverageCount": 15,
    "attachmentCount": 3
  }
}
```

**Frontend handler:** The frontend updates the document editor to show the new revision and its statistics.

**UI behavior:** A notification appears showing the draft is ready. The document editor may auto-refresh to show the updated content. Gap and coverage counts appear in a status bar.

---

### agent_error

**When it fires:** An error occurred during agent execution. This is a catch-all error event that reports failures in any phase.

**Payload structure:**
```json
{
  "type": "agent_error",
  "payload": {
    "phase": "execution",
    "attempt": 2,
    "message": "Failed to complete IP review: model returned invalid JSON"
  }
}
```

**Frontend handler:** The frontend displays the error message in the chat or activity panel. The user can retry or report the issue.

**UI behavior:** A red error banner appears with the error message. The failing phase is highlighted in the plan. A retry button may be shown.

---

### heartbeat

**When it fires:** The backend sends a heartbeat every 10 seconds (configurable via `ASSISTANT_HEARTBEAT_INTERVAL_MS` environment variable). This keeps the SSE connection alive through proxies, load balancers, and browsers that would otherwise close idle connections.

**Payload structure:**
```
: heartbeat\n\n
```
Note: Heartbeats are SSE comments (prefixed with `:`), not data events. They do not have a JSON payload.

**Frontend handler:** The browser's `EventSource` API automatically ignores comment lines. No explicit frontend handling is needed.

**UI behavior:** No visible UI change. The connection stays open and streaming continues.

---

### clarification

**When it fires:** The agent is asking the user a question during execution. Similar to `question_required` but specifically for the agentic workflow context.

**Payload structure:**
```json
{
  "type": "clarification",
  "payload": {
    "questionId": "clar-001",
    "prompt": "Should I include the subsidiary's data in the analysis?",
    "choices": ["Yes, include all subsidiaries", "No, only the parent entity"],
    "allowFreeText": false
  }
}
```

**Frontend handler:** The frontend pauses the streaming display and shows a clarification card. The user's response is sent back to continue execution.

**UI behavior:** A clarification dialog appears with the question and response options. The streaming animation pauses until the user responds.

---

## 5. Internal Backend Events

Internal backend events flow between services inside the server using Node.js `EventEmitter`-based event buses. These events never reach the frontend directly. Instead, they coordinate work between backend components and are eventually converted to SSE events for frontend delivery.

### Agent Events

Agent events are emitted by the `emitAgentEvent` function from `backend/src/events/agentEventBus.ts`. They use the `AgentStreamEvent` type and are published on per-job channels (`agent-job:{jobId}`).

#### agent.started

**Producer:** The agent job executor when a new job begins execution.

**Consumer:** The SSE stream endpoint that forwards events to the frontend. The audit logging system.

**Payload:**
```json
{
  "type": "sub_task_progress",
  "index": 0,
  "message": "Starting Corporate Review"
}
```

**Purpose:** Notifies the system that a new agent job has started. The frontend displays a "job started" indicator.

#### agent.completed

**Producer:** The agent job executor when all sub-tasks finish successfully.

**Consumer:** The SSE stream endpoint. The audit logging system. The artifact assembler.

**Payload:**
```json
{
  "type": "sub_task_completed",
  "index": 2,
  "result": {
    "findings": [...],
    "documents_reviewed": ["contract.pdf", "schedule.pdf"],
    "outstanding_rfi": ["Need board resolution"]
  }
}
```

**Purpose:** Signals that the agent job is complete. The frontend shows a completion indicator and displays the final artifact.

#### agent.failed

**Producer:** The agent job executor when a fatal error occurs.

**Consumer:** The SSE stream endpoint. The audit logging system. The error tracking system.

**Payload:**
```json
{
  "type": "job_failed",
  "error": "Model returned invalid JSON after 3 retries"
}
```

**Purpose:** Notifies the system that the agent job has failed. The frontend shows an error message and offers retry options.

#### agent.progress

**Producer:** The sub-agent executor (`subAgentExecutor.ts`) at various points during execution.

**Consumer:** The SSE stream endpoint for real-time progress updates.

**Payload:**
```json
{
  "type": "sub_task_progress",
  "index": 1,
  "message": "Model response received; validating JSON output"
}
```

**Purpose:** Provides granular progress updates during long-running tasks. The frontend updates progress indicators and status text.

---

### Tool Events

Tool events track when the AI uses external tools (functions) during its work.

#### tool.started

**Producer:** The chat orchestrator when the AI invokes a tool.

**Consumer:** The SSE stream endpoint. The activity trace system.

**Payload:**
```json
{
  "type": "tool_start",
  "toolCallId": "tc-001",
  "toolName": "searchDocuments",
  "input": { "query": "force majeure clauses" }
}
```

**Purpose:** Tells the frontend that the AI is about to use a tool. The UI shows a tool card with a running indicator.

#### tool.completed

**Producer:** The chat orchestrator when a tool finishes successfully.

**Consumer:** The SSE stream endpoint. The activity trace system.

**Payload:**
```json
{
  "type": "tool_complete",
  "toolCallId": "tc-001",
  "toolName": "searchDocuments",
  "output": { "results": [...], "count": 5 }
}
```

**Purpose:** Tells the frontend that a tool finished. The UI updates the tool card with a success indicator and output.

#### tool.failed

**Producer:** The chat orchestrator when a tool encounters an error.

**Consumer:** The SSE stream endpoint. The error tracking system.

**Payload:**
```json
{
  "type": "tool_error",
  "toolCallId": "tc-001",
  "toolName": "searchDocuments",
  "error": "Document index not available",
  "output": { "success": false, "error": "..." }
}
```

**Purpose:** Tells the frontend that a tool failed. The UI shows the tool card with an error indicator and error details.

---

### Document Events

Document events track the lifecycle of documents created or modified by the AI.

#### document.created

**Producer:** The document creation tool when a new document is generated.

**Consumer:** The SSE stream endpoint. The document library update system.

**Payload:**
```json
{
  "type": "doc_created",
  "document_id": "doc-abc-123",
  "filename": "DD_Report_Acme_Corp.docx",
  "version_id": "v-001",
  "version_number": 1,
  "download_url": "/download/doc-abc-123",
  "deep_link_url": "/documents/doc-abc-123"
}
```

**Purpose:** Notifies the frontend that a new document was created. The UI shows a document card with a download link.

#### document.updated

**Producer:** The document editing tool when an existing document is modified.

**Consumer:** The SSE stream endpoint. The document versioning system.

**Payload:**
```json
{
  "type": "doc_edited",
  "document_id": "doc-abc-123",
  "filename": "DD_Report_Acme_Corp.docx",
  "edit_id": "edit-001",
  "version_id": "v-002",
  "version_number": 2,
  "download_url": "/download/doc-abc-123?v=2",
  "annotations": [...]
}
```

**Purpose:** Notifies the frontend that a document was edited. The UI shows an edit confirmation and updated version.

#### document.deleted

**Producer:** The document deletion tool when a document is removed.

**Consumer:** The SSE stream endpoint. The document library update system.

**Payload:**
```json
{
  "type": "doc_deleted",
  "document_id": "doc-abc-123",
  "filename": "Draft_Notes.txt"
}
```

**Purpose:** Notifies the frontend that a document was deleted. The UI removes the document from the list.

#### document.version_created

**Producer:** The document versioning system when a new version is created.

**Consumer:** The SSE stream endpoint. The version history system.

**Payload:**
```json
{
  "type": "doc_version_created",
  "document_id": "doc-abc-123",
  "version_id": "v-003",
  "version_number": 3,
  "created_by": "agent"
}
```

**Purpose:** Tracks document version history. The UI shows version history in the document panel.

---

### Approval Events

Approval events track the approval workflow for agent actions.

#### approval.requested

**Producer:** The agent executor when it needs user approval for a plan or action.

**Consumer:** The SSE stream endpoint. The approval queue system.

**Payload:**
```json
{
  "type": "approval_required",
  "planId": "plan-abc-123",
  "prompt": "The agent wants to modify 3 documents. Do you approve?",
  "options": ["approve", "reject"]
}
```

**Purpose:** Pauses the workflow and asks the user for approval. The frontend shows an approval dialog.

#### approval.approved

**Producer:** The approval endpoint when the user approves an action.

**Consumer:** The agent executor (resumes execution). The audit logging system.

**Payload:**
```json
{
  "type": "plan_approved",
  "planId": "plan-abc-123",
  "approvedBy": "user-123"
}
```

**Purpose:** Resumes the paused workflow. The frontend hides the approval dialog and shows execution resuming.

#### approval.rejected

**Producer:** The approval endpoint when the user rejects an action.

**Consumer:** The agent executor (cancels execution). The audit logging system.

**Payload:**
```json
{
  "type": "plan_rejected",
  "planId": "plan-abc-123",
  "reason": "Too many documents to modify at once"
}
```

**Purpose:** Cancels the workflow. The frontend shows a cancellation message.

#### approval.cancelled

**Producer:** The agent executor when a pending approval is cancelled (e.g., due to timeout or user navigating away).

**Consumer:** The approval queue system. The audit logging system.

**Payload:**
```json
{
  "type": "approval_cancelled",
  "planId": "plan-abc-123",
  "reason": "User navigated away"
}
```

**Purpose:** Cleans up pending approvals. The frontend removes the approval dialog.

---

### Workflow Events

Workflow events track the lifecycle of multi-step workflows.

#### workflow.started

**Producer:** The workflow orchestrator when a new workflow begins.

**Consumer:** The SSE stream endpoint. The workflow tracking system.

**Payload:**
```json
{
  "type": "workflow_event",
  "event": {
    "type": "started",
    "workflowId": "wf-001",
    "workflowType": "due_diligence"
  }
}
```

**Purpose:** Notifies the system that a workflow has started. The frontend shows a workflow progress indicator.

#### workflow.step_completed

**Producer:** The workflow orchestrator when a step finishes.

**Consumer:** The SSE stream endpoint. The workflow tracking system.

**Payload:**
```json
{
  "type": "workflow_event",
  "event": {
    "type": "step_completed",
    "workflowId": "wf-001",
    "stepIndex": 1,
    "stepName": "Corporate Review",
    "result": { "findings": [...] }
  }
}
```

**Purpose:** Updates workflow progress. The frontend updates the step indicator from "running" to "completed."

#### workflow.completed

**Producer:** The workflow orchestrator when all steps finish.

**Consumer:** The SSE stream endpoint. The artifact assembler. The audit logging system.

**Payload:**
```json
{
  "type": "workflow_completed",
  "result": {
    "totalSteps": 3,
    "completedSteps": 3,
    "failedSteps": 0,
    "artifacts": [...]
  }
}
```

**Purpose:** Signals workflow completion. The frontend shows a completion indicator and displays the final artifact.

#### workflow.failed

**Producer:** The workflow orchestrator when a step fails fatally.

**Consumer:** The SSE stream endpoint. The error tracking system.

**Payload:**
```json
{
  "type": "workflow_event",
  "event": {
    "type": "failed",
    "workflowId": "wf-001",
    "failedStep": "IP Review",
    "error": "Model returned invalid JSON"
  }
}
```

**Purpose:** Signals workflow failure. The frontend shows an error message and offers retry options.

---

## 6. Frontend Events

Frontend events flow within the browser using Redux state updates and React hooks. They are triggered by SSE events and user interactions.

### Chat Events

#### chat.message_received

**Producer:** The `useChatStreamEvents` hook when a `text_delta` or `content_delta` SSE event arrives.

**Consumer:** The chat UI components.

**Payload:** The accumulated content string from all text deltas.

**Purpose:** Updates the chat bubble with new text as it streams in.

#### chat.streaming_started

**Producer:** The `useChatStreamEvents` hook when the stream begins and `isStreaming` is set to `true`.

**Consumer:** The chat UI components.

**Payload:** The initial `ChatStreamState` with `isStreaming: true`.

**Purpose:** Shows a streaming indicator in the chat bubble.

#### chat.streaming_completed

**Producer:** The `useChatStreamEvents` hook when a `done` or `stream_end` SSE event arrives.

**Consumer:** The chat UI components. The `backgroundChatSlice`.

**Payload:** The final `ChatStreamState` with `isStreaming: false`.

**Purpose:** Hides the streaming indicator and finalizes the message.

---

### Agent Events

#### agent.plan_received

**Producer:** The `useAgentJobStream` hook when a `plan_ready` SSE event arrives.

**Consumer:** The agent UI components. The `agentSlice` Redux store.

**Payload:** The `AgentPlan` object with steps and summary.

**Purpose:** Displays the execution plan in the UI.

#### agent.progress_updated

**Producer:** The `useAgentJobStream` hook when `sub_task_started`, `sub_task_progress`, or `sub_task_completed` events arrive.

**Consumer:** The agent UI components. The `agentSlice` Redux store.

**Payload:** The step ID, step name, progress percentage, and status message.

**Purpose:** Updates progress indicators in the agent panel.

#### agent.completed

**Producer:** The `useAgentJobStream` hook when a `job_completed` or `job_failed` SSE event arrives.

**Consumer:** The agent UI components. The `agentSlice` Redux store.

**Payload:** The final agent result or error message.

**Purpose:** Shows the completion or failure state of the agent job.

---

### UI Events

#### ui.notification

**Producer:** Various components when they need to show a persistent notification.

**Consumer:** The notification component.

**Payload:**
```typescript
{
  message: string
  type: 'success' | 'error' | 'warning' | 'info'
}
```

**Purpose:** Shows a notification banner at the top of the screen.

#### ui.toast

**Producer:** The `showToast` action from `uiSlice`.

**Consumer:** The toast component.

**Payload:**
```typescript
{
  show: true
  message: string
  type: 'success' | 'error' | 'warning' | 'info'
}
```

**Purpose:** Shows a temporary toast message that auto-dismisses.

#### ui.modal

**Producer:** The `openModal` action from `uiSlice`.

**Consumer:** The modal component.

**Payload:** The modal ID string (e.g., `"approval"`, `"clarification"`, `"settings"`).

**Purpose:** Opens a modal dialog overlay.

---

## 7. Event Bus Architecture

The backend uses two event bus implementations, both built on Node.js `EventEmitter`.

### How Events Are Published

Events are published using the `emit` method on the event emitter. Each event bus has its own publishing function:

**`backend/src/events/agentEventBus.ts`:**
```typescript
export function emitAgentEvent(jobId: string, event: AgentStreamEvent): void {
  agentEventBus.emit(jobEventName(jobId), event);
}
```

**`backend/src/agent/agentEventBus.ts`:**
```typescript
export function emitAgentJobEvent(jobId: string, event: AgentJobEvent): void {
  const envelope: AgentJobEventEnvelope = {
    jobId,
    event,
    emittedAt: new Date().toISOString(),
  };
  emitter.emit(allJobsChannel, envelope);
  emitter.emit(jobChannel(jobId), envelope);
}
```

### How Events Are Subscribed

Subscribers register handlers for specific event channels. Each subscription returns an unsubscribe function:

```typescript
// Subscribe to all events for a specific job
const unsubscribe = subscribeToJob(jobId, (event) => {
  console.log('Received event:', event);
});

// Later, to stop listening:
unsubscribe();
```

### Event Routing

Events are routed by job ID. Each job gets its own channel (`agent-job:{jobId}`), so subscribers only receive events for the jobs they care about. The `agentJobEventBus` also has a global channel (`agent-job-event`) that receives all events from all jobs.

### Event Filtering

There is no built-in event filtering in the event bus. Subscribers receive all events on their channel and must filter by event type in their handler code. For example:

```typescript
subscribeToJob(jobId, (event) => {
  if (event.type === 'job_completed') {
    // Handle completion
  }
});
```

### Event History

The event bus does not maintain event history. Events are fire-and-forget. If a subscriber is not connected when an event is emitted, it will miss that event. This is by design for real-time streaming. For historical data, events are persisted separately in the audit log.

---

## 8. Event Ordering

### Events Within a Stream Are Ordered

Events within a single SSE connection arrive in the order they were emitted. The backend writes events to the HTTP response sequentially, and the browser receives them in the same order. This guarantee is critical for text streaming, where `text_delta` events must arrive in order to produce coherent text.

### Events Across Streams May Be Unordered

Events from different SSE connections (e.g., a chat stream and an agent job stream) have no ordering guarantee. They run on separate connections and may arrive in any order relative to each other.

### Ordering Guarantees

- Text deltas within a single stream are strictly ordered.
- Tool start/complete pairs within a single stream are ordered (start always comes before complete).
- Phase start events come before their corresponding sub-agent events.
- The `done` event is always the last event in a stream.

### Race Condition Handling

The frontend handles race conditions through several mechanisms:

1. **State immutability:** Redux updates are atomic. Each event produces a new state object, preventing partial updates.
2. **Event normalization:** The `normalizeAgentStreamEvent` function in `useAgentJobStream.ts` normalizes event data regardless of arrival order.
3. **Terminal event detection:** The `TERMINAL_EVENT_TYPES` array (`['job_completed', 'job_failed']`) ensures the stream connection is closed immediately when a terminal event arrives, preventing stale events from being processed.
4. **Abort controller:** The `AbortController` pattern ensures cleanup happens synchronously when the user navigates away, preventing events from being processed after the component unmounts.

---

## 9. Event Persistence

### Audit Logging

All agent job events are logged to the audit system. The `AuditLogEntry` type tracks:
- `id`: Unique log entry ID
- `job_id`: The agent job that produced the event
- `event_type`: The type of event
- `actor`: Who or what produced the event
- `payload`: The full event payload
- `created_at`: Timestamp

### Event History

The `agentSlice` Redux store maintains a `streamEvents` array that accumulates all events received during an agent job session. This array is available for inspection and debugging but is cleared when the job is cleared.

### Replay Capability

There is no built-in event replay mechanism. Events are real-time and fire-and-forget. However, the agent job's final state is persisted in the database, and the `getAgentJobDebugExport` endpoint returns the full job state including all sub-tasks, clarifications, and audit log entries.

### Debugging Support

The `useAgentJobStream` hook exposes:
- `events`: The full array of received events
- `latestEvent`: The most recent event
- `isConnected`: Whether the SSE connection is active
- `retryCount`: How many reconnection attempts have been made
- `hasExhaustedRetries`: Whether all retry attempts have been used

The `VITE_AGENT_DEBUG_ENABLED` environment variable enables additional debug logging to an external endpoint.

---

## 10. Event Error Handling

### Event Delivery Failure

When an SSE connection fails, the `useAgentJobStream` hook implements exponential backoff reconnection:

- Base delay: 1000ms
- Maximum retries: 3
- Backoff formula: `BASE_RETRY_DELAY * 2^retryCount`
- Delays: 1s, 2s, 4s

After exhausting retries, the `onError` callback is invoked with an error message.

### Handler Failure

If an event handler throws an exception, the error is caught at the stream level and logged. The stream continues processing subsequent events. Invalid JSON in SSE data is silently skipped.

### Timeout

The SSE connection has no explicit timeout. The heartbeat mechanism (every 10 seconds) keeps the connection alive. If the backend stops sending heartbeats, the browser will eventually close the connection, triggering the reconnection logic.

### Retry Logic

The retry logic is implemented in `useAgentJobStream.ts`:

```typescript
if (shouldReconnectRef.current && retryCountRef.current < MAX_RETRIES) {
  const delay = BASE_RETRY_DELAY * Math.pow(2, retryCountRef.current);
  retryCountRef.current += 1;
  setRetryCount(retryCountRef.current);
  retryTimeoutRef.current = setTimeout(() => {
    connect();
  }, delay);
}
```

### Dead Letter Queue

There is no dead letter queue. Failed events are logged and discarded. The system is designed for real-time streaming where missed events are not recoverable. The final state is always available through the REST API.

---

## 11. Event Performance

### Event Batching

The `SmoothTextBuffer` in `chatApi.ts` batches text deltas to reduce re-renders. Text chunks are held for 20ms (`STREAM_TEXT_RELEASE_DELAY_MS`) before being flushed to the UI. This prevents the React render loop from being overwhelmed by individual character updates.

### Debouncing

The `useChatStreamEvents` hook uses `useCallback` to memoize event handlers, preventing unnecessary re-renders. The `stateRef` pattern avoids triggering state updates for every intermediate event.

### Throttling

There is no explicit throttling on event processing. The SSE protocol and the browser's event loop naturally throttle event delivery. The `SmoothTextBuffer` provides implicit throttling for text content.

### Memory Management

- The `streamEvents` array in `agentSlice` grows unboundedly during a session but is cleared when the job completes or is cleared.
- The `activeStreams` Map in `backgroundStreams.ts` is cleaned up when streams complete or error.
- The `SseHeartbeat` interval is explicitly cleared when the stream ends.
- The `StreamSanitizer` buffer is flushed when the stream ends.

### Connection Pooling

Each SSE connection is a dedicated HTTP connection. There is no connection pooling. The `backgroundStreams.ts` service manages a global registry of active streams to prevent duplicate connections for the same process.

---

## 12. Event Security

### Event Sanitization

The `StreamSanitizer` class in `stream-sanitizer.ts` strips leaked tool-call XML from LLM output before it reaches the frontend. This prevents sensitive tool invocation data from being exposed in the chat UI. The sanitizer uses pattern matching to detect and remove:
- `<function=...>...</function>` blocks
- `<tool_call>...</tool_call>` blocks
- `<|tool_call|>...</|tool_call|>` blocks
- DSML tool call blocks

### Access Control

SSE connections require authentication. The `useAgentJobStream` hook passes the access token as a query parameter. The backend validates the token before establishing the SSE connection. The `backgroundStreams.ts` service retrieves the access token from localStorage.

### Rate Limiting

There is no explicit rate limiting on SSE event delivery. The backend controls event frequency through the heartbeat interval and the natural speed of LLM generation. The `SmoothTextBuffer` on the frontend provides implicit rate limiting for text rendering.

### Audit Logging

All agent job events are logged with the actor (user or system) and timestamp. The audit log is immutable and provides a complete history of all actions taken during an agent job.

---

## 13. Event Testing

### Unit Testing Events

The `parseSSELine` and `formatSSEEvent` functions in `events.ts` are pure functions that can be unit tested:

```typescript
// Parse valid event
const event = parseSSELine('data: {"type":"thinking_delta","payload":{"text":"hello"}}');
expect(event).toEqual({ type: 'thinking_delta', payload: { text: 'hello' } });

// Parse invalid event
const invalid = parseSSELine('data: invalid json');
expect(invalid).toBeNull();

// Parse unknown event type
const unknown = parseSSELine('data: {"type":"unknown_type"}');
expect(unknown).toBeNull();
```

### Integration Testing

The `useChatStreamEvents` hook can be tested by mocking the `onStateChange` callback and calling `handleEvent` directly:

```typescript
const { handleEvent, getState } = useChatStreamEvents({
  onStateChange: (updater) => { state = updater(state); },
  onComplete: jest.fn(),
  onError: jest.fn(),
});

handleEvent('text_delta', { text: 'Hello' });
expect(getState().content).toBe('Hello');
```

### Load Testing

Load testing should focus on:
- SSE connection stability under high event throughput
- Memory usage with many concurrent streams
- Reconnection behavior under network instability

### Chaos Testing

Chaos testing should simulate:
- Backend crashes during active SSE streams
- Network partitions during agent execution
- Rapid page navigation during streaming
- Multiple concurrent agent jobs

---

## 14. Event Monitoring

### Event Metrics

Key metrics to monitor:
- SSE connection count (active streams)
- Event throughput (events per second)
- Average event latency (time from emission to frontend receipt)
- Reconnection rate
- Error rate per event type

### Latency Tracking

The `agentDebugLog` function in `chatApi.ts` logs timestamps at key points:
- Client request start
- Response headers received
- First content delta
- Stream done
- Stream errors

These logs can be used to measure end-to-end latency.

### Error Rates

Error rates should be tracked per event type:
- `agent_error` events
- `tool_error` events
- `error` events
- Stream connection failures
- Reconnection attempts

### Throughput

Throughput monitoring should track:
- Events per second per stream
- Total events per session
- Peak event rates during complex operations
- Heartbeat frequency compliance

---

## 15. Complete Event Flow Diagram

This diagram shows the complete flow of events through the entire system, from user input to UI rendering.

```
USER ACTION: "Review this contract for force majeure clauses"
    |
    v
+-----------------------------------------------------------------------+
| FRONTEND: Chat Input                                                   |
|   - User types message and clicks Send                                 |
|   - startChatStream() called from backgroundStreams.ts                 |
|   - HTTP POST to /api/chat with messages array                         |
|   - AbortController created for cancellation                           |
+-----------------------------------------------------------------------+
    |
    v
+-----------------------------------------------------------------------+
| BACKEND: Chat Orchestrator                                             |
|   - Receives POST request                                              |
|   - Creates SSE response (Content-Type: text/event-stream)            |
|   - Starts SseHeartbeat (every 10 seconds)                            |
|   - Creates StreamSanitizer                                            |
|   - Prepares messages for model                                        |
|   - Calls AssistantRuntimeMixer.run()                                  |
+-----------------------------------------------------------------------+
    |
    v
+-----------------------------------------------------------------------+
| BACKEND: LLM Streaming (via Mixer)                                    |
|   - Model begins generating response                                  |
|   - Heartbeat starts: ": heartbeat\n\n"                               |
|                                                                       |
|   EVENT 1: thinking_delta                                              |
|   data: {"type":"thinking_delta","payload":{"text":"Analyzing..."}}    |
|   -> Frontend: reasoning panel shows thinking text                    |
|                                                                       |
|   EVENT 2: thinking_done                                               |
|   data: {"type":"thinking_done","payload":{"durationMs":2500}}        |
|   -> Frontend: reasoning panel finalized                              |
|                                                                       |
|   EVENT 3: content_delta (text streaming begins)                      |
|   data: {"type":"content_delta","text":"Based on my review..."}       |
|   -> Frontend: text appears in chat bubble (typewriter effect)        |
|                                                                       |
|   EVENT 4: tool_start (AI decides to search documents)               |
|   data: {"type":"tool_start","toolCallId":"tc-1",                     |
|          "toolName":"searchDocuments"}                                 |
|   -> Frontend: tool card appears with "Running" indicator             |
|                                                                       |
|   EVENT 5: tool_complete (search finishes)                            |
|   data: {"type":"tool_complete","toolCallId":"tc-1",                  |
|          "output":{"results":[...],"count":5}}                        |
|   -> Frontend: tool card shows success, results listed                |
|                                                                       |
|   EVENT 6: content_delta (more text)                                  |
|   data: {"type":"content_delta","text":"The contract contains..."}    |
|   -> Frontend: more text appears in chat bubble                       |
|                                                                       |
|   EVENT 7: source_results (citations found)                           |
|   data: {"type":"source_results","results":[...]}                     |
|   -> Frontend: source cards appear below the message                  |
|                                                                       |
|   EVENT 8: citations (legal citations)                                |
|   data: {"type":"citations","citations":[...]}                        |
|   -> Frontend: citation badges appear inline                          |
|                                                                       |
|   EVENT 9: doc_created (AI generates a summary document)              |
|   data: {"type":"doc_created","document_id":"doc-123",                |
|          "filename":"FM_Summary.docx","download_url":"..."}           |
|   -> Frontend: document card with download link appears               |
|                                                                       |
|   EVENT 10: content_delta (final text)                                |
|   data: {"type":"content_delta","text":"In conclusion..."}            |
|   -> Frontend: final text appears in chat bubble                      |
|                                                                       |
|   EVENT 11: done (stream complete)                                    |
|   data: {"type":"done"}                                               |
|   -> Frontend: isStreaming = false, message finalized                 |
|   -> Heartbeat stopped                                                |
|   -> Connection closed                                                |
+-----------------------------------------------------------------------+
    |
    v
+-----------------------------------------------------------------------+
| FOR AGENT JOBS (separate flow):                                       |
|                                                                       |
| POST /agent/jobs  ->  Job created                                     |
|    |                                                                  |
|    v                                                                  |
| SSE /agent/jobs/{id}/stream  ->  Events stream:                       |
|    |                                                                  |
|    +-> plan_ready         ->  Plan card displayed                     |
|    +-> approval_required  ->  User approves                           |
|    +-> phase_start        ->  Phase indicator updates                 |
|    +-> subagent_start     ->  Sub-agent card appears                 |
|    +-> sub_task_progress  ->  Progress bar updates                    |
|    +-> sub_task_completed ->  Step marked complete                    |
|    +-> subagent_done      ->  Sub-agent card finalized               |
|    +-> tool_start         ->  Tool card appears                      |
|    +-> tool_done          ->  Tool card completed                    |
|    +-> artifact_ready     ->  Download link appears                  |
|    +-> job_completed      ->  Stream closed, final state saved       |
|                                                                       |
| Backend event bus flow:                                               |
|    emitAgentJobEvent() -> EventEmitter -> onAgentJobEvent()          |
|    Channel: "agent-job-event:{jobId}"                                 |
|    Envelope: { jobId, event, emittedAt }                              |
+-----------------------------------------------------------------------+
    |
    v
+-----------------------------------------------------------------------+
| BACKGROUND STREAMS (survives navigation):                             |
|                                                                       |
| startComplianceStream()  ->  Global registry                          |
| startChatStream()        ->  Global registry                          |
|    |                                                                  |
|    +-> On complete: dispatch(updateProcessStatus('completed'))        |
|    +-> On error:    dispatch(updateProcessStatus('error'))            |
|    +-> On abort:    removed from registry (no status change)          |
|                                                                       |
| backgroundChatSlice tracks:                                           |
|    - isActive, isStreaming, isComplete                                |
|    - messages array                                                   |
|    - chatId, projectId, workspaceId                                   |
|    - lastUserMessage                                                  |
+-----------------------------------------------------------------------+
    |
    v
+-----------------------------------------------------------------------+
| REDUX STATE UPDATES:                                                  |
|                                                                       |
| agentSlice:                                                           |
|    - activeJobId, planApproved, streamEvents[]                        |
|                                                                       |
| pendingProcessesSlice:                                                |
|    - processes[] (id, type, title, status, progress)                  |
|    - dismissedIds[]                                                   |
|                                                                       |
| backgroundChatSlice:                                                  |
|    - isActive, isStreaming, isComplete, messages[]                    |
|                                                                       |
| uiSlice:                                                              |
|    - toast, activeModal, sidebarCollapsed, theme                      |
+-----------------------------------------------------------------------+
    |
    v
+-----------------------------------------------------------------------+
| UI RENDERING:                                                         |
|                                                                       |
| Chat bubble:          Content from text_delta events                  |
| Reasoning panel:      Collapsible, from thinking_delta events         |
| Tool cards:           Running/complete/error indicators               |
| Source cards:         From source_results events                      |
| Citation badges:      Inline from citations events                   |
| Document cards:       Download links from doc_created events          |
| Plan card:            Phase progress from plan_ready events           |
| Activity trace:       Sub-agent status from subagent events           |
| Progress bar:         From sub_task_progress events                   |
| Approval dialog:      Modal from approval_required events             |
| Clarification card:   Question + options from question_required       |
| Toast notifications:  From ui.showToast() dispatches                 |
| Error banners:        From agent_error and error events               |
+-----------------------------------------------------------------------+
```

## 16. Event Reference Table

Master table of ALL events in the Mike platform.

| Event Name | Type | Source File | Producer | Consumer | Payload Summary | Frequency |
|---|---|---|---|---|---|---|
| thinking_delta | SSE | events.ts | LLM Mixer | useChatStreamEvents | `{ text: string }` | Per text chunk during thinking |
| thinking_done | SSE | events.ts | LLM Mixer | useChatStreamEvents | `{ durationMs, tokenCount }` | Once per thinking block |
| plan_ready | SSE | events.ts | Agent Planner | useAgentJobStream | `{ planId, title, summary, phases[] }` | Once per agent job |
| approval_required | SSE | events.ts | Agent Executor | UI Dialog | `{ planId, prompt, options[] }` | When approval needed |
| question_required | SSE | events.ts | Agent Executor | UI Dialog | `{ questionId, prompt, choices[], allowFreeText }` | When clarification needed |
| phase_start | SSE | events.ts | Agent Executor | Progress UI | `{ phase, workstreamId?, sectionId? }` | Per workflow phase |
| subagent_start | SSE | events.ts | Sub-Agent Executor | Activity Trace | `{ subagentId, title, workstreamId }` | Per sub-agent |
| subagent_done | SSE | events.ts | Sub-Agent Executor | Activity Trace | `{ subagentId, workstreamId, success, summary? }` | Per sub-agent |
| tool_start | SSE | events.ts | Chat Orchestrator | useChatStreamEvents | `{ toolId, toolName, workstreamId? }` | Per tool call |
| tool_done | SSE | events.ts | Chat Orchestrator | useChatStreamEvents | `{ toolId, toolName, success, outputSummary? }` | Per tool call |
| artifact_ready | SSE | events.ts | Artifact Assembler | UI Link | `{ artifactId, title, kind, referenceId? }` | Per artifact |
| draft_ready | SSE | events.ts | Drafting Engine | Editor UI | `{ revision, totalTokens, gapCount, coverageCount, attachmentCount }` | Per draft revision |
| agent_error | SSE | events.ts | Agent Executor | Error UI | `{ phase, attempt, message }` | Per error |
| heartbeat | SSE | sse-heartbeat.ts | Heartbeat Class | Browser (ignored) | `: heartbeat` (comment) | Every 10 seconds |
| clarification | SSE | events.ts | Agent Executor | UI Dialog | `{ questionId, prompt, choices?, allowFreeText }` | When clarification needed |
| content_delta | SSE | chatApi.ts | LLM Stream | useChatStreamEvents | `{ text: string }` | Per text chunk |
| text_delta | SSE | chatApi.ts | LLM Stream | useChatStreamEvents | `{ delta: string }` | Per text chunk (legacy) |
| reasoning_delta | SSE | chatApi.ts | LLM Stream | useChatStreamEvents | `{ text: string }` | Per reasoning chunk |
| reasoning_start | SSE | chatApi.ts | LLM Stream | useChatStreamEvents | `{}` | Once per reasoning block |
| reasoning_chunk | SSE | chatApi.ts | LLM Stream | useChatStreamEvents | `{ delta: string }` | Per reasoning chunk |
| reasoning_complete | SSE | chatApi.ts | LLM Stream | useChatStreamEvents | `{}` | Once per reasoning block |
| tool_call | SSE | chatApi.ts | Chat Orchestrator | useChatStreamEvents | `{ id, tool, input }` | Per tool call (legacy) |
| tool_call_start | SSE | chatApi.ts | Chat Orchestrator | useChatStreamEvents | `{ id, tool, input }` | Per tool call (legacy) |
| tool_result | SSE | chatApi.ts | Chat Orchestrator | useChatStreamEvents | `{ id, tool, output }` | Per tool call (legacy) |
| tool_call_complete | SSE | chatApi.ts | Chat Orchestrator | useChatStreamEvents | `{ id, tool }` | Per tool call (legacy) |
| tool_call_error | SSE | chatApi.ts | Chat Orchestrator | useChatStreamEvents | `{ id, error }` | Per tool error (legacy) |
| tool_streaming | SSE | chatApi.ts | Chat Orchestrator | useChatStreamEvents | `{ toolCallId, chunk }` | Per tool output chunk |
| source_results | SSE | chatApi.ts | RAG Pipeline | useChatStreamEvents | `{ results[] }` | Per search query |
| source_result | SSE | chatApi.ts | RAG Pipeline | useChatStreamEvents | `{ rank, filename, text }` | Per source found |
| web_search_results | SSE | chatApi.ts | Web Search Tool | useChatStreamEvents | `{ query, results[] }` | Per web search |
| citations | SSE | chatApi.ts | Citation Engine | useChatStreamEvents | `{ citations[] }` | Per citation batch |
| doc_created | SSE | chatApi.ts | Document Tool | useChatStreamEvents | `{ document_id, filename, download_url }` | Per document created |
| doc_edited | SSE | chatApi.ts | Document Tool | useChatStreamEvents | `{ document_id, filename, version_id }` | Per document edit |
| doc_replicated | SSE | chatApi.ts | Document Tool | useChatStreamEvents | `{ document_id, filename, count }` | Per document copy |
| doc_read_start | SSE | chatApi.ts | Document Tool | Activity Trace | `{ filename, document_id? }` | Per document read |
| doc_read | SSE | chatApi.ts | Document Tool | Activity Trace | `{ filename, document_id? }` | Per document read |
| doc_find_start | SSE | chatApi.ts | Document Tool | Activity Trace | `{ filename, query? }` | Per document search |
| doc_find | SSE | chatApi.ts | Document Tool | Activity Trace | `{ filename, query? }` | Per document search |
| doc_created_start | SSE | chatApi.ts | Document Tool | Activity Trace | `{ filename }` | Per document creation |
| doc_edited_start | SSE | chatApi.ts | Document Tool | Activity Trace | `{ filename }` | Per document edit |
| doc_edit_preview | SSE | chatApi.ts | Document Tool | Editor UI | `{ filename, edits[] }` | Per edit preview |
| doc_replicate_start | SSE | chatApi.ts | Document Tool | Activity Trace | `{ filename, count? }` | Per document copy |
| compliance_review_start | SSE | chatApi.ts | Compliance Engine | useChatStreamEvents | `{ compliance_review_id, document_id }` | Per compliance review |
| compliance_review_complete | SSE | chatApi.ts | Compliance Engine | useChatStreamEvents | `{ compliance_review_id, score }` | Per compliance review |
| tabular_review_start | SSE | chatApi.ts | Tabular Engine | useChatStreamEvents | `{ review_id, document_id }` | Per tabular review |
| tabular_review_complete | SSE | chatApi.ts | Tabular Engine | useChatStreamEvents | `{ review_id, row_count }` | Per tabular review |
| legal_case_collection | SSE | chatApi.ts | Case Law Engine | useChatStreamEvents | `{ cases[] }` | Per case law search |
| template_wizard_start | SSE | chatApi.ts | Template Engine | useChatStreamEvents | `{ wizard_id, fields[] }` | Per template wizard |
| template_wizard_lifecycle | SSE | chatApi.ts | Template Engine | useChatStreamEvents | `{ wizard_id, wizard_status }` | Per wizard status change |
| cross_reference_result | SSE | chatApi.ts | Analysis Engine | UI Table | `{ summary, headers[], rows[][] }` | Per cross-reference |
| project_analysis | SSE | chatApi.ts | Analysis Engine | UI Panel | `{ summary, parties[], risks[] }` | Per project analysis |
| draft_operation | SSE | chatApi.ts | Drafting Engine | Editor UI | `{ documentId, operation }` | Per draft edit |
| draft_complete | SSE | chatApi.ts | Drafting Engine | Editor UI | `{ documentId, summary }` | Per draft completion |
| activity_trace | SSE | chatApi.ts | Orchestrator | Activity Panel | `{ activities[] }` | Per activity update |
| activity_trace_delta | SSE | chatApi.ts | Orchestrator | Activity Panel | `{ activities[] }` | Per activity change |
| workflow_event | SSE | chatApi.ts | Workflow Runtime | Progress UI | `{ event }` | Per workflow event |
| workflow_completed | SSE | chatApi.ts | Workflow Runtime | UI | `{ result }` | Per workflow completion |
| workflow_phase | SSE | chatApi.ts | Workflow Runtime | Progress UI | `{ phase }` | Per workflow phase |
| legal_disclosure | SSE | chatApi.ts | Safety System | UI Banner | `{ disclosure_type, text }` | Per disclosure trigger |
| work_contract | SSE | chatApi.ts | Contract Engine | UI Panel | `{ contract_id, completion_criteria[] }` | Per work contract |
| artifact_validation | SSE | chatApi.ts | Validation Engine | UI Badge | `{ artifact_id, validation_status }` | Per artifact validation |
| completion_accounting | SSE | chatApi.ts | Accounting System | UI Counter | `{ requested, completed, failed }` | Per completion check |
| orchestration_thinking | SSE | chatApi.ts | Orchestrator | Activity Panel | `{ message }` | Per orchestration step |
| orchestration_start | SSE | chatApi.ts | Orchestrator | Activity Panel | `{ message, taskCount }` | Per orchestration start |
| orchestration_fallback | SSE | chatApi.ts | Orchestrator | Activity Panel | `{ reason }` | Per fallback |
| orchestration_complete | SSE | chatApi.ts | Orchestrator | Activity Panel | `{ successCount, failedCount }` | Per orchestration end |
| plan_ready (chat) | SSE | chatApi.ts | Orchestrator | Activity Panel | `{ taskCount, titles[] }` | Per plan |
| agent_start | SSE | chatApi.ts | Agent System | Activity Panel | `{ agentType, taskTitle? }` | Per agent start |
| agent_progress | SSE | chatApi.ts | Agent System | Activity Panel | `{ agentType, message }` | Per agent progress |
| agent_tool_call | SSE | chatApi.ts | Agent System | Activity Panel | `{ agentType, toolName }` | Per agent tool call |
| agent_complete | SSE | chatApi.ts | Agent System | Activity Panel | `{ agentType, success, durationMs? }` | Per agent completion |
| compliance_deep_link | SSE | chatApi.ts | Compliance Engine | UI Link | `{ compliance_review_id, filename }` | Per compliance link |
| tabular_deep_link | SSE | chatApi.ts | Tabular Engine | UI Link | `{ review_id, filename }` | Per tabular link |
| workspace_created | SSE | chatApi.ts | Workspace System | UI Notification | `{ workspace_id, name }` | Per workspace creation |
| project_created | SSE | chatApi.ts | Project System | UI Notification | `{ project_id, name }` | Per project creation |
| chat_id | SSE | chatApi.ts | Chat Backend | Frontend State | `{ chatId }` | Once per chat |
| done | SSE | chatApi.ts | Chat Backend | useChatStreamEvents | `{}` | Once per stream end |
| stream_end | SSE | chatApi.ts | Chat Backend | useChatStreamEvents | `{}` | Once per stream end (legacy) |
| error | SSE | chatApi.ts | Chat Backend | useChatStreamEvents | `{ message }` | Per error |
| plan_ready (agent) | Agent Bus | agentEventBus.ts | Agent Planner | SSE Forwarder | `{ plan }` | Per agent job |
| clarification_needed | Agent Bus | agentEventBus.ts | Agent Executor | SSE Forwarder | `{ clarification }` | Per clarification |
| clarification_answered | Agent Bus | agentEventBus.ts | Approval Endpoint | Agent Executor | `{ clarification_id }` | Per answer |
| sub_task_started | Agent Bus | agentEventBus.ts | Sub-Agent Executor | SSE Forwarder | `{ index, practice_area }` | Per sub-task |
| sub_task_completed | Agent Bus | agentEventBus.ts | Sub-Agent Executor | SSE Forwarder | `{ index, result }` | Per sub-task |
| job_completed | Agent Bus | agentEventBus.ts | Agent Executor | SSE Forwarder | `{ artifact }` | Per job end |
| job_failed | Agent Bus | agentEventBus.ts | Agent Executor | SSE Forwarder | `{ error }` | Per job failure |
| sub_task_progress | Event Bus | agentEventBus.ts | Sub-Agent Executor | SSE Forwarder | `{ index, message }` | Per progress update |
| startBackgroundChat | Redux | backgroundChatSlice.ts | Background Stream | Chat UI | `{ messages, chatId }` | Per background chat |
| updateBackgroundChatMessage | Redux | backgroundChatSlice.ts | Stream Handler | Chat UI | `{ content }` | Per text delta |
| setBackgroundChatComplete | Redux | backgroundChatSlice.ts | Stream Handler | Chat UI | `{}` | Per completion |
| addProcess | Redux | pendingProcessesSlice.ts | Stream Start | Process UI | `{ id, type, title }` | Per process start |
| updateProcessStatus | Redux | pendingProcessesSlice.ts | Stream Handler | Process UI | `{ id, status, progress? }` | Per status change |
| removeProcess | Redux | pendingProcessesSlice.ts | Stream End | Process UI | `{ id }` | Per process end |
| setActiveJob | Redux | agentSlice.ts | Agent Hook | Agent UI | `{ jobId }` | Per job start |
| approvePlan | Redux | agentSlice.ts | User Action | Agent UI | `{ approved }` | Per approval |
| addStreamEvent | Redux | agentSlice.ts | Agent Hook | Agent UI | `{ event }` | Per agent event |
| clearJob | Redux | agentSlice.ts | User Action | Agent UI | `{}` | Per job clear |
| showToast | Redux | uiSlice.ts | Any Component | Toast UI | `{ message, type }` | Per notification |
| openModal | Redux | uiSlice.ts | Any Component | Modal UI | `{ modalId }` | Per modal open |
| closeModal | Redux | uiSlice.ts | Any Component | Modal UI | `{}` | Per modal close |
