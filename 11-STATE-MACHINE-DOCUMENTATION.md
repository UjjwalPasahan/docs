# State Machine Documentation

## 1. What This Document Is

This document describes every state the Mike legal AI platform can be in and how it moves between those states. Think of it as a map that shows all the possible conditions the system can be in at any given moment, and what causes it to change from one condition to another.

The platform is a large system with many moving parts: a chat assistant, document editors, background agents, file uploads, email drafting, compliance reviews, and more. Each of these parts has its own set of states. When you send a message to the AI, the system goes through a series of states. When a background agent processes a document, it goes through its own series of states. This document catalogs all of them.

We cover both the frontend (what you see in your browser) and the backend (the server that does the work). The frontend uses React Context and Redux to manage state. The backend uses databases, job queues, and stream processing.

## 2. What Is a State Machine

A state machine is a way of organizing behavior so that a system can only be in one condition at a time, and it moves between conditions based on specific rules.

Think of a traffic light. A traffic light has three states: red, yellow, and green. At any given moment, it is in exactly one of those states. It does not combine them. It cannot be both red and green at the same time. When a timer expires, it transitions to the next state. Red becomes green. Green becomes yellow. Yellow becomes red.

In software, a state machine works the same way. A chat window can be in one of these states: waiting for input, waiting for a response, receiving a response, or showing an error. It is never in two of these states at once. When you click "Send", it moves from "waiting for input" to "waiting for a response". When the server starts sending data back, it moves to "receiving a response". When the data finishes, it moves back to "waiting for input".

State machines are useful because they make it easy to reason about complex systems. If you know the current state and the rules, you can predict exactly what will happen next. There are no surprises. The system is always in exactly one defined state, and transitions happen only when specific conditions are met.

In Mike, we use state machines in many places:

- **Redux slices** store the current state of authentication, UI settings, background chats, pending processes, and the agent.
- **React Context** manages chat messages, input fields, artifacts, connectors, and UI panels.
- **Backend job queues** track whether agent jobs are planning, running, paused, completed, or failed.
- **Server-Sent Events (SSE)** streams transition through connection states as they connect, receive data, reconnect on failure, and disconnect when done.

Each state machine documented below lists its states, what triggers each transition, and what side effects happen along the way.

## 3. Conversation State Machine

The conversation state machine governs the lifecycle of a single chat interaction. It lives in the ChatContext and determines what the user sees in the chat interface.

### States

| State | What It Means |
|-------|---------------|
| `idle` | No conversation activity. The user can type a message. |
| `thinking` | The user sent a message. The system is preparing to process it. |
| `planning` | The orchestrator is deciding which agents to run. |
| `tool_running` | An agent is executing a tool (searching documents, running compliance checks, etc.). |
| `streaming` | The AI is sending back its response word by word. |
| `done` | The response is complete. The conversation is finished. |
| `cancelled` | The user or system stopped the conversation before completion. |
| `error` | Something went wrong. The system cannot continue. |

### Transitions

```
idle ──────────────────> thinking ─────────> planning ─────────> tool_running ─────────> streaming ─────────> done
                            │                    │                    │                    │
                            │                    │                    │                    │
                            v                    v                    v                    v
                        cancelled             cancelled             cancelled             cancelled
                            │                    │                    │                    │
                            v                    v                    v                    v
                          error               error               error               error


error ───────────────────> idle (retry)
```

### Transition Triggers

- `idle → thinking`: User sends a message.
- `thinking → planning`: The orchestrator starts classifying the intent and creating a task plan.
- `planning → tool_running`: A tool is invoked (document search, compliance review, web search, etc.).
- `tool_running → tool_running`: Another tool is invoked (tools can chain).
- `tool_running → streaming`: The system starts generating the final text response.
- `streaming → done`: The server sends the `[DONE]` event.
- `streaming → ready`: The user clicks "Stop" to cancel the stream.
- `any → cancelled`: User explicitly cancels the conversation or navigates away.
- `any → error`: An error occurs at any point (network failure, server error, timeout).
- `error → idle`: User clicks retry or sends a new message.

### Code Reference

- `ChatContext.tsx:42` — Status type: `'ready' | 'submitted' | 'streaming' | 'error'`
- `ChatContext.tsx:127` — Status state initialization
- `ChatContext.tsx:163` — Status set to `'submitted'` on message send
- `ChatContext.tsx:199` — Status set to `'streaming'` on content delta
- `ChatContext.tsx:241-244` — Status set to `'error'` or `'ready'` on completion

## 4. Chat Message State Machine

Each individual message in a conversation has its own lifecycle. A message can be a user message or an assistant message. This state machine tracks the delivery and display status of each message.

### States

| State | What It Means |
|-------|---------------|
| `sending` | The message has been typed and the send button was clicked. It is on its way to the server. |
| `sent` | The server has received the message. |
| `delivering` | The server is processing the message and preparing a response. |
| `delivered` | The response has been fully sent to the client. |
| `read` | The user has scrolled to see the message or it is visible on screen. |
| `failed` | The message could not be sent (network error, server error). |
| `retrying` | The system is attempting to resend a failed message. |
| `streaming` | The assistant response is being streamed in word by word. |
| `complete` | The assistant response is fully received and displayed. |

### Transitions

```
sending ──────────────> sent ──────────────> delivering ──────────────> delivered ──────────────> read
    │                                                                                              
    │                                                                                              
    v                                                                                              
  failed ──────────────> retrying ──────────────> sent
  


delivered ──────────────> streaming ──────────────> complete
```

### Transition Triggers

- `sending → sent`: The HTTP request is acknowledged by the server.
- `sent → delivering`: The server begins processing and generating a response.
- `delivering → delivered`: The server sends the full response payload.
- `delivered → read`: The client marks the message as visible.
- `sending → failed`: The HTTP request fails (timeout, network error, 5xx).
- `failed → retrying`: The user clicks retry or the system auto-retries.
- `retrying → sent`: The retry request is acknowledged.
- `delivered → streaming`: An SSE stream begins sending content deltas.
- `streaming → complete`: The server sends the `[DONE]` event.

### Code Reference

- `ChatContext.tsx:162-163` — User message created and status set to `'submitted'`
- `useChatStreamEvents.ts:448-452` — Stream end event sets `isStreaming: false`
- `backgroundStreams.ts:92,223-226` — Stream completion dispatches `'completed'` to Redux

## 5. Tool Execution State Machine

When the AI calls a tool (search documents, check compliance, look up case law, create a document), each tool invocation goes through its own state machine.

### States

| State | What It Means |
|-------|---------------|
| `pending` | The tool has been registered but execution has not started. |
| `running` | The tool is actively executing. |
| `streaming` | The tool is producing output incrementally (e.g., a document being generated). |
| `success` | The tool completed successfully and produced a result. |
| `complete` | The tool finished and the result has been processed. |
| `failed` | The tool encountered an error and could not produce a result. |
| `timeout` | The tool ran longer than the allowed time limit. |
| `retrying` | The system is attempting to re-execute a failed tool. |

### Transitions

```
pending ──────────────> running ──────────────> streaming ──────────────> complete
                            │                         
                            │                         
                            v                         
                          success
                            │
                            v
                          failed ──────────────> retrying ──────────────> running
                            │
                            v
                         timeout
```

### Transition Triggers

- `pending → running`: The tool is invoked by the agent coordinator.
- `running → streaming`: The tool produces incremental output (used for long-running tools).
- `running → success`: The tool returns a result without error.
- `running → complete`: The tool result is processed and stored.
- `running → failed`: The tool throws an error or returns a failure response.
- `running → timeout`: The tool exceeds the configured timeout (e.g., 45 seconds for planning, 120 seconds for agent execution).
- `failed → retrying`: The system decides to retry based on retry logic.
- `retrying → running`: The tool is re-invoked.

### Code Reference

- `useChatStreamEvents.ts:158-176` — `tool_start` event creates tool with status `'running'`
- `useChatStreamEvents.ts:178-189` — `tool_streaming` event updates status to `'streaming'`
- `useChatStreamEvents.ts:192-210` — `tool_complete` event sets status to `'complete'`
- `useChatStreamEvents.ts:214-235` — `tool_error` event sets status to `'error'`
- `orchestrator/index.ts:21-24` — Timeout constants (PLAN: 45s, AGENTS: 120s, SYNTHESIS: 45s)

## 6. Document Editor State Machine

The document editor allows users to view, edit, and save documents within the platform. It manages loading content, editing, saving changes, and exporting.

### States

| State | What It Means |
|-------|---------------|
| `loading` | The document is being fetched from the server. |
| `ready` | The document is loaded and displayed. No changes have been made. |
| `editing` | The user is actively making changes to the document. |
| `unsaved` | Changes have been made but not yet saved. |
| `saving` | The document is being saved to the server. |
| `saved` | All changes have been successfully saved. |
| `error` | Something went wrong during loading, saving, or editing. |
| `exporting` | The document is being prepared for download or export. |
| `exported` | The document has been successfully exported. |

### Transitions

```
loading ──────────────> ready ──────────────> editing ──────────────> saving ──────────────> saved
    │                     │                     │                                               │
    │                     │                     │                                               │
    v                     v                     v                                               v
  error                 error                 unsaved ──────────────> saving ──────────────> saved
                                                                           
saved ──────────────> exporting ──────────────> exported
```

### Transition Triggers

- `loading → ready`: Document content is fetched and rendered.
- `loading → error`: The document fetch fails (404, network error, permission denied).
- `ready → editing`: The user clicks into the editor and starts typing.
- `editing → unsaved`: The user makes a change that has not been auto-saved.
- `unsaved → saving`: The auto-save timer fires or the user clicks save.
- `saving → saved`: The server confirms the save was successful.
- `saving → error`: The save request fails.
- `saved → exporting`: The user clicks download or export.
- `exporting → exported`: The file download begins.
- `any → error`: Any operation can fail and transition to error state.

### Code Reference

- `ArtifactContext.tsx:94-102` — `openDocument` sets loading then opens artifact
- `ArtifactContext.tsx:112-120` — Streaming document operations (open, update, finalize)
- `ArtifactContext.tsx:148-151` — `updateDocument` updates content in state

## 7. Background Agent State Machine

Background agents are long-running tasks that process documents, perform compliance reviews, or generate pleadings matrices. They run in a job queue on the backend and report status via Server-Sent Events.

### States

| State | What It Means |
|-------|---------------|
| `submitted` | The job has been submitted to the queue but has not started processing. |
| `planning` | The agent is analyzing the task and creating an execution plan. |
| `running` | The agent is actively executing its plan. |
| `paused` | The agent has been temporarily paused (e.g., waiting for user input). |
| `paused_for_input` | The agent needs clarification from the user before continuing. |
| `completed` | The agent finished successfully and produced an artifact. |
| `failed` | The agent encountered an error and could not complete. |
| `cancelled` | The user or system cancelled the job. |

### Transitions

```
submitted ──────────────> planning ──────────────> running ──────────────> completed
                               │                      │
                               │                      │
                               v                      v
                            failed                 paused ──────────────> running
                                                       │
                                                       v
                                              paused_for_input ──────────────> running
                               │                      │
                               v                      v
                            failed                 failed
                                                       │
                                                       v
                                                   cancelled
```

### Transition Triggers

- `submitted → planning`: The worker picks up the job and starts the planning phase.
- `planning → running`: The plan is complete and execution begins.
- `planning → failed`: The planning step fails (invalid input, LLM error).
- `running → completed`: All sub-tasks are complete and the artifact is generated.
- `running → failed`: A sub-task fails or an unrecoverable error occurs.
- `running → paused`: The user pauses the job (if supported).
- `running → paused_for_input`: The agent needs clarification from the user.
- `paused_for_input → running`: The user provides the requested input.
- `running → cancelled`: The user cancels the job.

### Code Reference

- `agentWorker.ts:456-461` — Job transitions from planning to running with plan and start time
- `agentWorker.ts:237-238` — `updateJobStatus(jobId, "planning")`
- `agentWorker.ts:242-245` — `updateJobStatus(jobId, "running", { plan, started_at })`
- `agentWorker.ts:281-283` — `updateJobStatus(jobId, "completed", { completed_at })`
- `agentWorker.ts:493-499` — `updateJobStatus(jobId, "failed", { error })`
- `agentWorker.ts:324` — `updateJobStatus(jobId, "paused_for_input")`

## 8. Approval Workflow State Machine

Documents and drafts in Mike can go through an approval workflow. This is used for legal documents that need review before they are finalized.

### States

| State | What It Means |
|-------|---------------|
| `draft` | The document is being written or revised. It is not yet submitted for review. |
| `pending_approval` | The document has been submitted and is waiting for an approver to review it. |
| `approved` | An approver has reviewed and approved the document. |
| `rejected` | An approver has reviewed the document and rejected it. Revise and resubmit. |
| `cancelled` | The approval request was cancelled before a decision was made. |

### Transitions

```
draft ──────────────> pending_approval ──────────────> approved
                          │
                          │
                          v
                      rejected ──────────────> draft (revise and resubmit)
                          │
                          v
                      cancelled
```

### Transition Triggers

- `draft → pending_approval`: The author submits the document for review.
- `pending_approval → approved`: The approver clicks approve.
- `pending_approval → rejected`: The approver clicks reject with comments.
- `pending_approval → cancelled`: The author or an admin cancels the request.
- `rejected → draft`: The author revises the document and resubmits.

## 9. Document Lifecycle State Machine

Documents in the platform go through a lifecycle from creation to finalization. This is broader than the approval workflow and covers the entire life of a document.

### States

| State | What It Means |
|-------|---------------|
| `DRAFT` | The document is being actively written or edited. |
| `IN_REVIEW` | The document has been submitted for peer or supervisor review. |
| `PENDING_APPROVAL` | The document is waiting for formal approval from an authorized person. |
| `APPROVED` | The document has been approved and is ready for use. |
| `FINALIZED` | The document is locked and cannot be edited further. |
| `ARCHIVED` | The document has been moved to long-term storage. It is read-only. |

### Transitions

```
DRAFT ──────────────> IN_REVIEW ──────────────> PENDING_APPROVAL ──────────────> APPROVED ──────────────> FINALIZED
    │                     │
    │                     │
    v                     v
 ARCHIVED             DRAFT (revision needed)


Any State ──────────────> ARCHIVED
```

### Transition Triggers

- `DRAFT → IN_REVIEW`: The author submits the document for review.
- `IN_REVIEW → PENDING_APPROVAL`: The reviewer passes it to the approver.
- `IN_REVIEW → DRAFT`: The reviewer sends it back for revision.
- `PENDING_APPROVAL → APPROVED`: The approver approves it.
- `PENDING_APPROVAL → DRAFT`: The approver sends it back for revision.
- `APPROVED → FINALIZED`: The document is finalized (locked, versioned).
- `Any → ARCHIVED`: An admin or the system archives the document.

## 10. RAG Collection State Machine

RAG (Retrieval-Augmented Generation) collections are indexes of documents that the AI can search through. Each collection goes through a lifecycle as it is built, updated, and queried.

### States

| State | What It Means |
|-------|---------------|
| `empty` | The collection exists but contains no documents. |
| `indexing` | Documents are being processed and added to the search index. |
| `ready` | The collection is fully indexed and available for queries. |
| `updating` | New documents are being added or existing documents are being re-indexed. |
| `error` | An error occurred during indexing or querying. |
| `reindexing` | The collection is being rebuilt from scratch after an error. |

### Transitions

```
empty ──────────────> indexing ──────────────> ready
                          │
                          v
                       updating ──────────────> ready
                          │
                          v
                        error ──────────────> reindexing ──────────────> ready
```

### Transition Triggers

- `empty → indexing`: Documents are uploaded or added to the collection.
- `indexing → ready`: All documents have been processed and indexed.
- `ready → updating`: New documents are added or existing documents change.
- `updating → ready`: The update is complete.
- `ready → error`: An error occurs during indexing or querying.
- `error → reindexing`: The system starts a full re-index.
- `reindexing → ready`: The re-index completes successfully.

## 11. SSE Connection State Machine

Server-Sent Events (SSE) provide real-time streaming from the backend to the frontend. The connection goes through several states as it is established, maintained, and recovered.

### States

| State | What It Means |
|-------|---------------|
| `disconnected` | No active SSE connection. |
| `connecting` | The client is attempting to establish a connection. |
| `connected` | The connection is active and receiving events. |
| `reconnecting` | The connection was lost and the client is trying to re-establish it. |
| `timeout` | The connection timed out (no data received within the expected interval). |
| `aborted` | The connection was intentionally closed (user navigated away, user clicked stop). |

### Transitions

```
disconnected ──────────────> connecting ──────────────> connected
                                   │                       │
                                   │                       │
                                   v                       v
                               aborted               reconnecting ──────────────> connected
                                                       │
                                                       v
                                                     timeout ──────────────> disconnected
```

### Transition Triggers

- `disconnected → connecting`: The client initiates a new SSE connection.
- `connecting → connected`: The server accepts the connection and starts sending events.
- `connecting → aborted`: The connection attempt is cancelled.
- `connected → reconnecting`: The connection drops unexpectedly.
- `reconnecting → connected`: The reconnection succeeds.
- `reconnecting → timeout`: Reconnection attempts exhaust the timeout.
- `timeout → disconnected`: The connection is closed after timeout.
- `any → aborted`: The user clicks stop, navigates away, or the component unmounts.

### Code Reference

- `useChatStreamEvents.ts:96` — `resetState` sets `isStreaming: true`
- `useChatStreamEvents.ts:448-452` — `done` event sets `isStreaming: false`
- `backgroundStreams.ts:97-101` — AbortError handling keeps status as running
- `ChatContext.tsx:144-152` — `stop()` calls `abortRef.current?.abort()`

## 12. Authentication State Machine

The authentication system manages user login, token refresh, and session expiry. It persists state to localStorage so users stay logged in across browser sessions.

### States

| State | What It Means |
|-------|---------------|
| `unauthenticated` | The user is not logged in. They need to sign in. |
| `authenticating` | The system is verifying credentials (calling the API). |
| `authenticated` | The user is logged in and has a valid access token. |
| `refreshing` | The access token has expired and the system is getting a new one. |
| `expired` | The session has expired and the user needs to log in again. |
| `error` | An authentication error occurred (wrong password, server error). |
| `isLoading` | A general loading flag used during any auth operation. |

### Transitions

```
unauthenticated ──────────────> authenticating ──────────────> authenticated
                                    │                               │
                                    │                               │
                                    v                               v
                                  error                          refreshing ──────────────> authenticated
                                                                    │
                                                                    v
                                                                  expired ──────────────> unauthenticated
```

### Transition Triggers

- `unauthenticated → authenticating`: User submits login form or OAuth redirect completes.
- `authenticating → authenticated`: Login succeeds. Token and user data are saved.
- `authenticating → error`: Login fails (invalid credentials, server error).
- `authenticated → refreshing`: The access token expires and a refresh is attempted.
- `refreshing → authenticated`: Token refresh succeeds.
- `refreshing → expired`: Token refresh fails. User must log in again.
- `any → error`: Any auth operation can fail.

### Code Reference

- `authSlice.ts:60-67` — Initial auth state loaded from localStorage
- `authSlice.ts:99-106` — `setCredentials` sets authenticated state
- `authSlice.ts:130-138` — `logout` clears all auth state
- `authSlice.ts:188-206` — RTK Query matchers handle pending/rejected states

## 13. UI State Machine

The UI state machine manages the visual layout of the application: sidebar visibility, theme, modals, and panels. These are toggle-based states that the user controls directly.

### States

| State | What It Means |
|-------|---------------|
| `sidebar_collapsed` | The sidebar is hidden to give more screen space to content. |
| `sidebar_open` | The sidebar is visible showing navigation and conversation list. |
| `ai_panel_collapsed` | The AI assistant panel is hidden. |
| `ai_panel_open` | The AI assistant panel is visible. |
| `theme_light` | The application uses a light color scheme. |
| `theme_dark` | The application uses a dark color scheme. |
| `theme_system` | The application follows the operating system preference. |
| `modal_closed` | No modal dialog is open. |
| `modal_open` | A modal dialog is open (identified by `activeModal` string). |
| `toast_visible` | A notification toast is being displayed. |
| `toast_hidden` | No toast is being displayed. |

### Transitions

```
sidebar_open ──────────────> sidebar_collapsed
sidebar_collapsed ──────────────> sidebar_open

ai_panel_open ──────────────> ai_panel_collapsed
ai_panel_collapsed ──────────────> ai_panel_open

theme_light ──────────────> theme_dark
theme_dark ──────────────> theme_light
theme_light ──────────────> theme_system
theme_system ──────────────> theme_light

modal_closed ──────────────> modal_open ──────────────> modal_closed

toast_hidden ──────────────> toast_visible ──────────────> toast_hidden
```

### Transition Triggers

- `toggleSidebar`: User clicks the sidebar toggle button.
- `toggleAiPanel`: User clicks the AI panel toggle.
- `setTheme`: User selects a theme from settings.
- `openModal`: User triggers an action that opens a modal (create project, share, etc.).
- `closeModal`: User clicks close, clicks outside, or presses Escape.
- `showToast`: System displays a notification (success, error, warning, info).
- `hideToast`: Toast auto-dismisses after a timeout or user clicks dismiss.

### Code Reference

- `uiSlice.ts:49-58` — Initial UI state loaded from localStorage
- `uiSlice.ts:64-66` — `toggleSidebar` with localStorage persistence
- `uiSlice.ts:78-80` — `toggleAiPanel` with localStorage persistence
- `uiSlice.ts:99-101` — `setTheme` with localStorage persistence
- `uiSlice.ts:103-108` — `openModal` / `closeModal` for modal management
- `uiSlice.ts:109-121` — `showToast` / `hideToast` for notifications

## 14. Workflow Execution State Machine

Workflows are multi-step automated processes. They can be triggered by users or by the system. A workflow coordinates multiple agents, tools, and approvals.

### States

| State | What It Means |
|-------|---------------|
| `idle` | The workflow exists but has not been started. |
| `running` | The workflow is actively executing its steps. |
| `paused` | The workflow has been temporarily halted. |
| `completed` | The workflow finished all steps successfully. |
| `failed` | The workflow encountered an error and could not continue. |
| `cancelled` | The user or system cancelled the workflow. |

### Transitions

```
idle ──────────────> running ──────────────> completed
                         │
                         │
                         v
                       paused ──────────────> running
                         │
                         v
                       failed
                         │
                         v
                     cancelled
```

### Transition Triggers

- `idle → running`: The user or system starts the workflow.
- `running → paused`: The user pauses the workflow or a step requires manual approval.
- `paused → running`: The user resumes the workflow.
- `running → completed`: All steps complete successfully.
- `running → failed`: A step fails and cannot be retried.
- `running → cancelled`: The user cancels the workflow.

## 15. Email Draft State Machine

The email drafting system allows users to compose, schedule, and send emails through connected email accounts.

### States

| State | What It Means |
|-------|---------------|
| `draft` | The email is being composed. |
| `sending` | The email is being sent through the email provider. |
| `sent` | The email has been successfully sent. |
| `scheduled` | The email is queued to be sent at a later time. |
| `failed` | Sending the email failed. |
| `retrying` | The system is attempting to resend a failed email. |

### Transitions

```
draft ──────────────> sending ──────────────> sent
    │                     
    │                     
    v                     
  scheduled ──────────────> sending ──────────────> sent
                            
                            
  sending ──────────────> failed ──────────────> retrying ──────────────> sending
```

### Transition Triggers

- `draft → sending`: The user clicks send.
- `sending → sent`: The email provider confirms delivery.
- `sending → failed`: The email provider returns an error.
- `failed → retrying`: The system or user initiates a retry.
- `retrying → sending`: The retry attempt begins.
- `draft → scheduled`: The user sets a future send time.

### Code Reference

- `ArtifactContext.tsx:89-92` — `openEmailDraft` creates initial draft artifact
- `ArtifactContext.tsx:128` — `sendEmailDraft` returns success result
- `ArtifactContext.tsx:138-140` — `sendEmailDraftById` marks draft as sent in local state
- `ArtifactContext.tsx:142-144` — `deleteEmailDraftById` marks draft as deleted

## 16. File Upload State Machine

File uploads follow a state machine that handles selection, upload progress, completion, and failure.

### States

| State | What It Means |
|-------|---------------|
| `selected` | The user has selected files but upload has not started. |
| `uploading` | The file is being transferred to the server. |
| `uploaded` | The file has been fully uploaded and processed. |
| `failed` | The upload failed (network error, file too large, etc.). |
| `paused` | The upload has been temporarily halted. |
| `retrying` | The system is attempting to retry a failed upload. |

### Transitions

```
selected ──────────────> uploading ──────────────> uploaded
                            │
                            │
                            v
                          paused ──────────────> uploading
                            │
                            v
                         failed ──────────────> retrying ──────────────> uploading
```

### Transition Triggers

- `selected → uploading`: The upload begins (either auto-start or user clicks upload).
- `uploading → uploaded`: The server confirms receipt and processing.
- `uploading → failed`: The upload fails (network drop, server error, validation error).
- `uploading → paused`: The user pauses the upload.
- `paused → uploading`: The user resumes the upload.
- `failed → retrying`: The user clicks retry.
- `retrying → uploading`: The upload restarts.

### Code Reference

- `InputContext.tsx:83-99` — `handleFileUpload` manages upload state with `isUploading` flag
- `InputContext.tsx:71` — `isUploading` state tracks upload progress

## 17. State Transition Rules

State transitions follow strict rules. Here is a summary of the principles that govern them:

### What Triggers Each Transition

Every transition is caused by one of these events:

1. **User action**: Clicking a button, typing text, selecting an option.
2. **Server response**: An API call returns success, failure, or timeout.
3. **System timer**: Auto-save timers, session expiry timers, retry delays.
4. **Stream event**: An SSE event arrives from the backend (content_delta, tool_start, done, error).
5. **State cascade**: One state change triggers another (e.g., stream completion triggers process status update).

### What Actions Happen During Transition

When a transition occurs, the system may:

1. **Update Redux state**: Dispatch an action to change the stored state.
2. **Update React state**: Call a setState function to re-render the UI.
3. **Persist to localStorage**: Save state for recovery after page reload.
4. **Send an API request**: Make an HTTP call to the backend.
5. **Start or stop a stream**: Open or close an SSE connection.
6. **Dispatch a notification**: Show a toast or alert to the user.
7. **Log an event**: Write to the console or a logging service.

### What Conditions Must Be Met

Some transitions have guards:

- **Authentication required**: You cannot start a workflow without being logged in.
- **Validation passed**: You cannot save a document that fails validation.
- **Rate limit not exceeded**: You cannot exceed the orchestration rate limit.
- **Token budget not exceeded**: The tool loop stops when the input token budget is hit.
- **No repeated tool calls**: The system stops if the same tool is called 3 times in a row with the same input.
- **No stalled loops**: The system stops if a tool produces no progress for 2 consecutive steps.

### What Side Effects Occur

Side effects that happen as a result of transitions:

1. **Redux persistence**: Auth and UI state are saved to localStorage on every change.
2. **Stream registration**: Active streams are registered in a global Map for lifecycle management.
3. **Process tracking**: Running processes are added to the pending processes list.
4. **Worker updates**: Backend job status is written to the database.
5. **Notifications**: Users are notified when jobs complete or fail.
6. **Audit logging**: All significant state changes are logged for compliance.

## 18. State Persistence

The system uses multiple persistence mechanisms to ensure state is not lost.

### Redux Persist (localStorage)

The following state slices are persisted to localStorage:

| Slice | Storage Key | What Is Saved |
|-------|-------------|---------------|
| `auth` | `legality_auth` | User, accessToken, refreshToken, onboardingCompleted |
| `ui` | `legality_ui` | sidebarCollapsed, aiPanelCollapsed, theme |
| `sourceCanvas` | `sourceCanvasPanelWidth` | Panel width for the source canvas |

### Session Storage

Session storage is not used in the current codebase. All persistence goes through localStorage.

### Server State

The following state is persisted on the server:

- **Chat messages**: Stored in the database per chatId.
- **Agent jobs**: Status, plan, results, and artifacts are stored in the database.
- **Documents**: File content and metadata are stored in the database and object storage.
- **Compliance reviews**: Results are stored in the database.
- **User settings**: API keys and preferences are stored in the database.

### How Persistence Works

On page load, the system reads from localStorage and reconstructs the initial state:

1. `authSlice.ts:15-32` — `loadAuthFromStorage()` reads the auth data.
2. `uiSlice.ts:20-30` — `loadUIFromStorage()` reads the UI data.
3. `sourceCanvasSlice.ts:9-22` — `loadPanelWidth()` reads the panel width.

On every state change, the system writes back to localStorage:

1. `authSlice.ts:34-48` — `saveAuthToStorage()` writes auth data.
2. `uiSlice.ts:32-45` — `saveUIToStorage()` writes UI data.

## 19. State Synchronization

The system synchronizes state across multiple channels.

### Client-Server Sync

- **Redux Toolkit Query (RTK Query)**: Used for auth API calls. The cache invalidation and refetching logic keeps the client in sync with the server.
- **Server-Sent Events (SSE)**: Used for real-time streaming of chat responses, tool outputs, and agent progress.
- **Background streams**: The `backgroundStreams.ts` service manages long-running streams that continue even when the user navigates away.

### Multi-Tab Sync

Multi-tab synchronization is not currently implemented. Each browser tab maintains its own independent state. If a user opens the application in two tabs, changes in one tab will not automatically appear in the other. However, since auth state is in localStorage, both tabs will share the same authentication session.

### Real-Time Updates

Real-time updates are delivered via SSE:

- **Chat streaming**: Content deltas, tool events, and completion signals arrive in real time.
- **Agent progress**: Sub-task started, progress, and completion events arrive in real time.
- **Compliance reviews**: Start and complete events arrive in real time.
- **Background process status**: Redux is updated globally when background processes complete.

### Conflict Resolution

Since multi-tab sync is not implemented, there are no conflict resolution mechanisms. If a user edits a document in two tabs simultaneously, the last save wins.

## 20. State Error Handling

The system handles state errors through several strategies.

### Invalid Transitions

If an invalid transition is attempted (e.g., sending a message while already streaming), the system simply ignores the action. For example:

- `InputContext.tsx:107` — The `submit` function checks `if (!text || isLoading) return` and does nothing if the chat is already loading.
- `ChatContext.tsx:144-152` — The `stop` function aborts the controller and sets status to `'ready'`, regardless of the current status.

### State Corruption

If state becomes corrupted (e.g., a stream event arrives after the component has unmounted), the system handles it gracefully:

- `backgroundStreams.ts:97-101` — AbortError is silently ignored. The stream is removed from the active map but the process status is not changed.
- `useChatStreamEvents.ts:93-98` — `resetState` replaces the entire state with a fresh initial state, preventing stale data from accumulating.

### Recovery Strategies

1. **Auto-retry**: Failed uploads and API calls can be retried by the user.
2. **Stream reconnection**: SSE connections can be re-established if they drop.
3. **localStorage fallback**: If server state is lost, the client can recover from localStorage.
4. **Graceful degradation**: If the orchestrator fails, the system falls back to the legacy chat handler (`orchestrator/index.ts:288-312`).
5. **Process removal**: Completed or errored processes are automatically removed after a 5-second delay (`backgroundStreams.ts:289-293`).

### Rollback Mechanisms

There are no explicit rollback mechanisms. The system does not maintain undo history for state changes. However, since most state is derived from the server, a page reload will restore the last known good state from the server.

## 21. Complete State Diagram

The following is a comprehensive text diagram showing all major states and transitions across the entire Mike platform.

```
============================================================================================
                           MIKE PLATFORM - COMPLETE STATE DIAGRAM
============================================================================================

AUTHENTICATION
===============
unauthenticated ──[login]──> authenticating ──[success]──> authenticated
                                    │                           │
                                    │                           │
                                    v                           v
                                 error                      refreshing
                                                              │      │
                                                              │      │
                                                              v      v
                                                          unauthenticated  authenticated
                                                             
                                                              v
                                                           expired ──> unauthenticated


CHAT CONVERSATION
=================
idle ──[send message]──> thinking ──[orchestrator starts]──> planning
                                                            │
                                                            │
                                                            v
                                                     tool_running ──> tool_running (loop)
                                                            │
                                                            v
                                                        streaming ──[done]──> done
                                                            │
                                                            v
                                                         error ──[retry]──> idle
                                                            │
                                                            v
                                                        cancelled


CHAT STATUS (ChatContext)
==========================
ready ──[sendMessage]──> submitted ──[content_delta]──> streaming ──[done]──> ready
                              │                             │
                              │                             │
                              v                             v
                           error                         error
                              │
                              v
                           ready (onComplete)


TOOL EXECUTION
===============
pending ──[invoke]──> running ──[output]──> streaming ──[complete]──> complete
                          │
                          │
                          v
                        success
                          │
                          v
                       failed ──[retry]──> retrying ──[invoke]──> running
                          │
                          v
                       timeout


DOCUMENT EDITOR
===============
loading ──[fetch]──> ready ──[edit]──> editing ──[change]──> unsaved
    │                  │                                          │
    │                  │                                          │
    v                  v                                          v
  error              error                                     saving ──[confirm]──> saved
                                                                     │
                                                                     v
                                                                   error
                                                                     │
                                                                     v
                                                                   saved ──[export]──> exporting ──[done]──> exported


BACKGROUND AGENT
================
submitted ──[pick up]──> planning ──[plan done]──> running ──[all done]──> completed
                              │                      │
                              │                      │
                              v                      v
                           failed                 paused ──[resume]──> running
                                                      │
                                                      v
                                           paused_for_input ──[answer]──> running
                              │                      │
                              v                      v
                           failed                cancelled


APPROVAL WORKFLOW
=================
draft ──[submit]──> pending_approval ──[approve]──> approved
                         │
                         │
                         v
                      rejected ──[revise]──> draft
                         │
                         v
                      cancelled


DOCUMENT LIFECYCLE
==================
DRAFT ──[submit]──> IN_REVIEW ──[pass]──> PENDING_APPROVAL ──[approve]──> APPROVED ──[finalize]──> FINALIZED
    │                    │
    │                    │
    v                    v
 ARCHIVED            DRAFT (revision)
    
Any ──[archive]──> ARCHIVED


RAG COLLECTION
===============
empty ──[add docs]──> indexing ──[done]──> ready ──[new docs]──> updating ──[done]──> ready
                              │                                      │
                              │                                      v
                              v                                    error
                           error ──[reindex]──> reindexing ──[done]──> ready


SSE CONNECTION
===============
disconnected ──[init]──> connecting ──[accept]──> connected
                              │                      │
                              │                      │
                              v                      v
                           aborted             reconnecting ──[success]──> connected
                                                  │
                                                  v
                                                timeout ──> disconnected
                           aborted <── [stop/navigate away] ── any state


BACKGROUND CHAT
================
inactive ──[start]──> active+streaming ──[done]──> active+complete
                          │
                          v
                      update message (streaming)


PENDING PROCESSES
==================
(none) ──[add]──> running ──[complete]──> completed
                     │
                     v
                   error
                     │
                     v
                 (removed after 5s)


SOURCE CANVAS
==============
closed ──[open]──> open+loading ──[loaded]──> open+ready
                         │
                         v
                      error
                         │
                         v
                     open+error


INPUT STATE
============
idle ──[type]──> has input ──[submit]──> sending ──[clear]──> idle
                      │
                      v
                 has attachments ──[upload]──> uploading ──[done]──> uploaded


EMAIL DRAFT
============
draft ──[send]──> sending ──[confirm]──> sent
    │
    │
    v
 scheduled ──[time]──> sending
    │
    v
  failed ──[retry]──> retrying ──> sending


FILE UPLOAD
============
selected ──[start]──> uploading ──[done]──> uploaded
                          │
                          v
                        paused ──[resume]──> uploading
                          │
                          v
                       failed ──[retry]──> retrying ──> uploading


WORKFLOW
=========
idle ──[start]──> running ──[complete]──> completed
                     │
                     v
                   paused ──[resume]──> running
                     │
                     v
                   failed
                     │
                     v
                 cancelled


ORCHESTRATOR PIPELINE
======================
entry ──[classify]──> simple ──> legacy handler
         │
         v
      complex ──[rate limit check]──> rate limited ──> legacy handler
         │
         v
      orchestrate ──[plan]──> planning ──[agents]──> executing ──[synthesize]──> synthesis
         │                      │                       │                           │
         │                      │                       │                           │
         v                      v                       v                           v
      fallback             timeout                  timeout                     timeout
         │
         v
    legacy fallback


AGENT STOP CONDITIONS
======================
running ──[token budget exceeded]──> stopped
running ──[repeated tool call x3]──> stopped
running ──[same tool loop x8]──> stopped
running ──[stalled x4 steps]──> stopped
```

## 22. State Reference Table

Master table of all states across the Mike platform.

| State Name | Component/File | Current States | Transitions From | Transitions To | Triggers | Persistence |
|------------|----------------|----------------|------------------|----------------|----------|-------------|
| Auth: unauthenticated | authSlice.ts | - | authenticated, expired, error | authenticating | Page load, logout, token expiry | localStorage |
| Auth: authenticating | authSlice.ts | - | unauthenticated | authenticated, error | Login form submit, OAuth redirect | localStorage |
| Auth: authenticated | authSlice.ts | - | authenticating, refreshing | refreshing, expired, unauthenticated | Login success, token refresh | localStorage |
| Auth: refreshing | authSlice.ts | - | authenticated | authenticated, expired | Token expiry | localStorage |
| Auth: expired | authSlice.ts | - | refreshing | unauthenticated | Refresh failure | localStorage |
| Auth: error | authSlice.ts | - | authenticating, refreshing | unauthenticated | Login failure, server error | localStorage |
| Auth: isLoading | authSlice.ts | - | Any | Any | Any auth operation | No |
| Chat: ready | ChatContext.tsx | `status='ready'` | streaming, error, done | submitted | sendMessage | No |
| Chat: submitted | ChatContext.tsx | `status='submitted'` | ready | streaming, error | sendMessage | No |
| Chat: streaming | ChatContext.tsx | `status='streaming'` | submitted | ready, error | content_delta event | No |
| Chat: error | ChatContext.tsx | `status='error'` | submitted, streaming | ready | error event, network failure | No |
| Stream: idle | useChatStreamEvents.ts | `isStreaming=false` | streaming | streaming | resetState | No |
| Stream: streaming | useChatStreamEvents.ts | `isStreaming=true` | idle | idle | resetState, done event | No |
| Process: running | pendingProcessesSlice.ts | `status='running'` | (new) | completed, error | addProcess, updateProcessStatus | Redux |
| Process: completed | pendingProcessesSlice.ts | `status='completed'` | running | (removed) | updateProcessStatus | Redux |
| Process: error | pendingProcessesSlice.ts | `status='error'` | running | (removed) | updateProcessStatus | Redux |
| Agent: active | agentSlice.ts | `activeJobId != null` | (none) | (none) | setActiveJob | Redux |
| Agent: plan approved | agentSlice.ts | `planApproved=true` | (none) | (none) | approvePlan | Redux |
| Background Chat: inactive | backgroundChatSlice.ts | `isActive=false` | active | active | clearBackgroundChat, restoreBackgroundChat | Redux |
| Background Chat: active | backgroundChatSlice.ts | `isActive=true` | inactive | inactive | startBackgroundChat | Redux |
| Background Chat: streaming | backgroundChatSlice.ts | `isStreaming=true` | inactive | inactive | startBackgroundChat | Redux |
| Background Chat: complete | backgroundChatSlice.ts | `isComplete=true` | streaming | inactive | setBackgroundChatComplete | Redux |
| UI: sidebar_collapsed | uiSlice.ts | `sidebarCollapsed=true` | false | false | toggleSidebar | localStorage |
| UI: sidebar_open | uiSlice.ts | `sidebarCollapsed=false` | true | true | toggleSidebar | localStorage |
| UI: ai_panel_collapsed | uiSlice.ts | `aiPanelCollapsed=true` | false | false | toggleAiPanel | localStorage |
| UI: ai_panel_open | uiSlice.ts | `aiPanelCollapsed=false` | true | true | toggleAiPanel | localStorage |
| UI: theme | uiSlice.ts | `theme='light'\|'dark'\|'system'` | Any | Any | setTheme | localStorage |
| UI: activeModal | uiSlice.ts | `activeModal=string\|null` | null | null | openModal, closeModal | No |
| UI: toast | uiSlice.ts | `toast=object\|null` | null | null | showToast, hideToast | No |
| Source Canvas: open | sourceCanvasSlice.ts | `isOpen=true` | closed | closed | openSourceCanvas | No |
| Source Canvas: closed | sourceCanvasSlice.ts | `isOpen=false` | open | open | closeSourceCanvas | No |
| Source Canvas: loading | sourceCanvasSlice.ts | `isLoading=true` | open | open | openSourceCanvas | No |
| Source Canvas: error | sourceCanvasSlice.ts | `error=string\|null` | open | open | setSourceCanvasError | No |
| Artifact: open | ArtifactContext.tsx | `isOpen=true` | closed | closed | openDocument, openEmailDraft | No |
| Artifact: closed | ArtifactContext.tsx | `isOpen=false` | open | open | closeArtifact | No |
| Artifact: loading | ArtifactContext.tsx | `isLoading=true` | closed | open | openDocument | No |
| Artifact: saving | ArtifactContext.tsx | `isSaving=true` | open | open | updateDocument | No |
| Input: idle | InputContext.tsx | `input=''` | has input | has input | setInput | No |
| Input: has input | InputContext.tsx | `input != ''` | idle | idle | setInput | No |
| Input: dragging | InputContext.tsx | `isDragging=true` | not dragging | not dragging | handleDragOver | No |
| Input: uploading | InputContext.tsx | `isUploading=true` | not uploading | not uploading | handleFileUpload | No |
| Connector: loading | ConnectorContext.tsx | `isLoading=true` | loaded | loaded | refreshConnectors | No |
| Connector: refreshing | ConnectorContext.tsx | `isRefreshing=true` | loaded | loaded | refreshConnectors | No |
| Agent Worker: submitted | agentWorker.ts | DB status='submitted' | (none) | planning | Job enqueued | Database |
| Agent Worker: planning | agentWorker.ts | DB status='planning' | submitted | running, failed | planAgentJob | Database |
| Agent Worker: running | agentWorker.ts | DB status='running' | planning, paused_for_input | completed, failed, paused, paused_for_input, cancelled | executeSubTasks | Database |
| Agent Worker: paused | agentWorker.ts | DB status='paused' | running | running | User action | Database |
| Agent Worker: paused_for_input | agentWorker.ts | DB status='paused_for_input' | running | running | User provides clarification | Database |
| Agent Worker: completed | agentWorker.ts | DB status='completed' | running | (terminal) | All tasks done | Database |
| Agent Worker: failed | agentWorker.ts | DB status='failed' | planning, running | (terminal) | Error thrown | Database |
| Agent Worker: cancelled | agentWorker.ts | DB status='cancelled' | running, paused, paused_for_input | (terminal) | User cancel | Database |
| Tool: pending | useChatStreamEvents.ts | `status='pending'` | (none) | running | Tool registered | No |
| Tool: running | useChatStreamEvents.ts | `status='running'` | pending | streaming, complete, error | tool_start event | No |
| Tool: streaming | useChatStreamEvents.ts | `status='streaming'` | running | complete, error | tool_streaming event | No |
| Tool: complete | useChatStreamEvents.ts | `status='complete'` | running, streaming | (terminal) | tool_complete event | No |
| Tool: error | useChatStreamEvents.ts | `status='error'` | running, streaming | (terminal) | tool_error event | No |
| Orchestrator: entry | orchestrator/index.ts | - | (none) | legacy, orchestrate | Request arrives | No |
| Orchestrator: planning | orchestrator/index.ts | - | orchestrate | executing, timeout | classifyIntent, createTaskPlan | No |
| Orchestrator: executing | orchestrator/index.ts | - | planning | synthesis, timeout | agentCoordinator.execute | No |
| Orchestrator: synthesis | orchestrator/index.ts | - | executing | complete, timeout | generateFinalResponse | No |
| Orchestrator: fallback | orchestrator/index.ts | - | any | legacy | Error or timeout | No |
| Stop: token budget | stop-conditions.ts | - | running | stopped | inputTokens >= budget | No |
| Stop: repeated call | stop-conditions.ts | - | running | stopped | 3 identical consecutive calls | No |
| Stop: same loop | stop-conditions.ts | - | running | stopped | 8 identical consecutive loops | No |
| Stop: stalled | stop-conditions.ts | - | running | stopped | 4 steps with zero progress | No |

---

*Document generated from codebase analysis. Last updated: 2026-07-14*
