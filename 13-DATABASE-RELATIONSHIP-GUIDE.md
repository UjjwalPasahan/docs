# Database Relationship Guide

## 1. What This Document Is

This document explains every table in the Mike legal AI platform database and how they all connect to each other. Think of it as a complete map of all the data the system stores. If you are a developer, this tells you exactly what information goes where. If you are a product person, this tells you what data the system can hold and how different features relate to each other.

The database has **87 tables** organized into 15 functional groups. Each group handles a specific part of the platform — from user accounts and documents to AI workflows and compliance checks. Every table has a unique ID, timestamps for when records are created and updated, and foreign keys that link it to other tables.

---

## 2. Database Overview

| Property | Value |
|---|---|
| **Database Engine** | PostgreSQL 16 |
| **ORM** | Drizzle ORM (schema-first approach) |
| **Total Tables** | 87 |
| **Total Enums** | 32 |
| **Primary Key Type** | UUID (auto-generated) |
| **Schema Source** | `backend/src/db/schema.ts` (2525 lines) |
| **Connection** | `backend/src/db/index.ts` |
| **Drizzle Config** | `backend/drizzle.config.ts` |
| **Migrations** | `backend/migrations/` and `packages/db/migrations/` |

### Schema-First Approach

The schema is defined in TypeScript using Drizzle ORM. The `schema.ts` file is the single source of truth. Drizzle generates SQL migrations from this file. This means you edit the TypeScript, run a migration command, and the database gets updated automatically.

### Connection Pool

The database connection uses a standard PostgreSQL connection pool with these defaults:
- Connection timeout: 5 seconds
- Idle timeout: 30 seconds
- Query timeout: 10 seconds
- Statement timeout: 10 seconds

---

## 3. Entity Relationship Diagram

Below is a text-based ERD showing the major relationships between table groups. Each arrow shows a foreign key relationship.

```
                          ┌─────────────────┐
                          │      users       │
                          │  (central hub)   │
                          └────────┬────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
          ▼                        ▼                        ▼
  ┌───────────────┐    ┌──────────────────┐    ┌──────────────────┐
  │  auth_accounts │    │  user_profiles   │    │  user_api_keys   │
  │ auth_sessions  │    │  user_memories   │    │  user_activity   │
  │    auth_otp    │    │ learning_events  │    │  user_activity   │
  └───────────────┘    └──────────────────┘    └──────────────────┘
          │                        │
          ▼                        ▼
  ┌───────────────────────────────────────────────────────────────┐
  │                        PROJECTS                               │
  │  projects ──► project_members, project_subfolders             │
  │  project_invitations (via share_invitations)                  │
  └───────────────────────────┬───────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
  │  DOCUMENTS   │  │   CHATS      │  │   WORKFLOWS      │
  │              │  │              │  │                  │
  │ versions     │  │ sessions     │  │ workflow_builder │
  │ comments     │  │ messages     │  │ templates        │
  │ edits        │  │ interview    │  │                  │
  │ shares       │  │ extractions  │  │                  │
  │ members      │  │              │  │                  │
  │ activity     │  │              │  │                  │
  └──────┬───────┘  └──────┬───────┘  └──────────────────┘
         │                 │
         ▼                 ▼
  ┌─────────────────────────────────────┐
  │           WORKSPACES                │
  │  workspaces ──► members             │
  │  task_lists, task_statuses          │
  │  tasks, task_assignees              │
  │  task_comments, task_activity       │
  │  folders, files, file_versions      │
  └─────────────────────────────────────┘
          │
          ├──► RAG (rag_collections, rag_source_index_entries)
          ├──► EVIDENCE (evidence_source_refs, generated_citation_evidence)
          ├──► APPROVALS (approval_approvers, approval_requests, approval_request_items)
          ├──► COMPLIANCE (compliance_reviews, compliance_review_rules, etc.)
          ├──► EMAIL (email_connector_accounts, email_drafts, etc.)
          ├──► SHARING (document_shares, share_invitations)
          ├──► NOTIFICATIONS (notifications)
          ├──► AGENT (agent_jobs, agent_sub_tasks, agent_artifacts)
          └──► TEMPLATES (templates)
```

---

## 4. Users & Authentication Tables

### 4.1 users

**Purpose**: The central table. Every person who uses Mike has exactly one row here. Everything else in the system links back to this table.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key, auto-generated |
| `email` | varchar(255) | Unique, not null |
| `full_name` | varchar(255) | Not null |
| `country` | varchar(100) | Not null |
| `jurisdiction` | varchar(255) | Nullable |
| `organization` | varchar(255) | Not null |
| `role` | varchar(20) | Default: "viewer" |
| `email_verified` | boolean | Default: false |
| `storage_limit_bytes` | bigint | Default: 16106127360 (15 GB) |
| `storage_used_bytes` | bigint | Default: 0 |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Relationships**:
- One user has many projects, documents, chats, workspaces, etc.
- One user has many sessions, API keys, profiles

**Indexes**: None explicitly (primary key on `id`, unique on `email`)

---

### 4.2 auth_accounts

**Purpose**: Stores OAuth provider accounts linked to a user. When someone signs in with Google or Microsoft, the provider details go here.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `provider` | varchar(50) | e.g. "google", "microsoft" |
| `provider_account_id` | varchar(255) | The provider's unique ID |
| `legality_auth` | text | Access token (column name is legacy) |
| `refresh_token` | text | Nullable |
| `expires_at` | timestamp | Nullable |
| `created_at` | timestamp | Default: now() |

**Indexes**:
- `auth_accounts_provider_idx` — unique on (provider, provider_account_id)
- `auth_accounts_user_idx` — on (user_id)

---

### 4.3 auth_sessions

**Purpose**: Active login sessions. When a user logs in, a session token is created here. The token is checked on every request.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `token` | varchar(500) | Unique, not null |
| `expires_at` | timestamp | Not null |
| `created_at` | timestamp | Default: now() |

**Indexes**:
- `auth_sessions_user_idx` — on (user_id)
- `auth_sessions_token_idx` — on (token)

---

### 4.4 auth_otp

**Purpose**: One-time passwords for email-based login. Stores the code sent to a user's email for verification.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `email` | varchar(255) | Not null |
| `code` | varchar(6) | The OTP code |
| `expires_at` | timestamp | Not null |
| `verified` | boolean | Default: false |
| `created_at` | timestamp | Default: now() |

**Indexes**:
- `auth_otp_email_idx` — on (email)

---

### 4.5 user_profiles

**Purpose**: Extended profile information for a user. Separate from the users table to keep the main table lean.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, unique, cascade delete |
| `display_name` | varchar(255) | Nullable |
| `organisation` | varchar(255) | Nullable |
| `message_credits_used` | integer | Default: 0 |
| `credits_reset_date` | timestamp | Nullable |
| `tier` | varchar(50) | Default: "Free" |
| `tabular_model` | varchar(100) | Nullable |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Indexes**: None explicitly (unique constraint on user_id)

---

### 4.6 user_api_keys

**Purpose**: Stores encrypted API keys that users provide for third-party services (like OpenAI, Anthropic, etc.).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `provider` | varchar(50) | e.g. "openai", "anthropic" |
| `encrypted_key` | text | The encrypted API key |
| `iv` | text | Initialization vector for decryption |
| `auth_tag` | text | Authentication tag for verification |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Indexes**:
- `user_api_keys_user_provider_idx` — unique on (user_id, provider)

---

### 4.7 auth_email_events

**Purpose**: Tracks every email sent for authentication purposes (OTP codes, verification emails, etc.). Helps debug delivery issues.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `recipient` | varchar(255) | Email address |
| `template` | varchar(120) | Which email template was used |
| `trigger_type` | varchar(120) | What triggered the email |
| `status` | auth_email_status enum | pending/sent/failed/dev_fallback |
| `resend_message_id` | varchar(255) | ID from the email provider |
| `error` | text | Error message if failed |
| `retry_count` | integer | Default: 0 |
| `metadata` | jsonb | Additional data |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Indexes**:
- `auth_email_events_recipient_idx` — on (recipient)
- `auth_email_events_status_idx` — on (status)
- `auth_email_events_created_idx` — on (created_at)

---

## 5. Project Tables

### 5.1 projects

**Purpose**: A project is a container for organizing documents and work. Each user can have multiple projects.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `name` | varchar(255) | Not null |
| `cm_number` | varchar(100) | Nullable, custom metadata |
| `shared_with` | jsonb | Default: [] |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Indexes**:
- `projects_user_idx` — on (user_id)

---

### 5.2 project_members

**Purpose**: Tracks who has access to a project and what role they have (admin, editor, viewer).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `project_id` | uuid | FK → projects.id, cascade delete |
| `user_id` | uuid | FK → users.id, cascade delete, nullable |
| `email` | varchar(255) | Not null |
| `role` | share_role enum | admin/editor/viewer |
| `invited_by_user_id` | uuid | FK → users.id, set null on delete |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Indexes**:
- `project_members_project_idx` — on (project_id)
- `project_members_user_idx` — on (user_id)
- `project_members_email_idx` — on (email)
- `project_members_project_user_idx` — unique on (project_id, user_id) where user_id is not null
- `project_members_project_email_idx` — unique on (project_id, email)

---

### 5.3 project_subfolders

**Purpose**: Subfolders within a project for organizing documents.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `project_id` | uuid | FK → projects.id, cascade delete |
| `user_id` | uuid | FK → users.id, cascade delete |
| `name` | varchar(255) | Not null |
| `parent_folder_id` | uuid | Self-referential, nullable |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Indexes**:
- `project_subfolders_project_idx` — on (project_id)

---

## 6. Document Tables

### 6.1 documents

**Purpose**: The main document table. Every uploaded document gets a row here. Documents can belong to a project, a workspace, or both.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `project_id` | uuid | FK → projects.id, cascade delete, nullable |
| `workspace_id` | uuid | Nullable (no FK in schema, referenced by workspace) |
| `user_id` | uuid | FK → users.id, cascade delete |
| `folder_id` | uuid | FK → project_subfolders.id, set null on delete |
| `filename` | varchar(500) | Not null |
| `file_type` | varchar(20) | Nullable |
| `size_bytes` | integer | Nullable |
| `page_count` | integer | Nullable |
| `structure_tree` | jsonb | Document structure outline |
| `status` | varchar(50) | Default: "processing" |
| `lifecycle_status` | document_lifecycle_status enum | DRAFT/IN_REVIEW/PENDING_APPROVAL/APPROVED/FINALIZED |
| `current_version_id` | uuid | Points to current document_versions row |
| `attached` | boolean | Default: false |
| `is_primary` | boolean | Default: true |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Indexes**:
- `documents_project_idx` — on (project_id)
- `documents_workspace_idx` — on (workspace_id)
- `documents_user_idx` — on (user_id)
- `documents_attached_idx` — on (attached)
- `documents_is_primary_idx` — on (is_primary)

---

### 6.2 document_versions

**Purpose**: Every time a document is updated, a new version is created. This table stores all versions.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `document_id` | uuid | FK → documents.id, cascade delete |
| `storage_path` | varchar(1000) | Where the file is stored |
| `pdf_storage_path` | varchar(1000) | PDF version path, nullable |
| `source` | varchar(50) | Where the version came from |
| `version_number` | integer | Sequential number |
| `display_name` | varchar(500) | Human-readable name |
| `created_at` | timestamp | Default: now() |

**Indexes**:
- `document_versions_document_idx` — on (document_id)

---

### 6.3 document_comments

**Purpose**: Comments on documents. Supports threaded replies (parent_comment_id).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `document_id` | uuid | FK → documents.id, cascade delete |
| `version_id` | uuid | FK → document_versions.id, set null on delete |
| `user_id` | uuid | FK → users.id, set null on delete |
| `parent_comment_id` | uuid | FK → self (threaded), cascade delete |
| `body` | text | Not null |
| `anchor_text` | text | Text the comment is attached to |
| `anchor_start` | integer | Start position |
| `anchor_end` | integer | End position |
| `metadata` | jsonb | Default: {} |
| `resolved` | boolean | Default: false |
| `resolved_by_user_id` | uuid | FK → users.id, set null |
| `resolved_at` | timestamp | When resolved |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Indexes**:
- `document_comments_document_idx` — on (document_id)
- `document_comments_version_idx` — on (version_id)
- `document_comments_parent_idx` — on (parent_comment_id)
- `document_comments_user_idx` — on (user_id)

---

### 6.4 document_placeholder_values

**Purpose**: Stores values for placeholder fields in document templates. When a document has fields like "{{client_name}}", the values are stored here.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `document_id` | uuid | FK → documents.id, cascade delete |
| `field_key` | varchar(120) | The placeholder name |
| `value` | text | The value to substitute |
| `created_by_user_id` | uuid | FK → users.id, set null |
| `updated_by_user_id` | uuid | FK → users.id, set null |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Indexes**:
- `document_placeholder_values_document_idx` — on (document_id)
- `document_placeholder_values_field_idx` — on (field_key)
- `document_placeholder_values_document_field_idx` — unique on (document_id, field_key)

---

### 6.5 Additional Document Tables

- **document_context_files** — Links documents to context documents for AI analysis
- **document_edits** — Tracks individual text edits (insertions/deletions) on documents
- **document_activity** — Audit trail of all actions on documents
- **document_members** — Who has access to a specific document and their role (DRAFTER/REVIEWER/APPROVER)
- **document_chat_messages** — Chat messages tied to a specific document
- **document_state_transitions** — Tracks lifecycle status changes (DRAFT → IN_REVIEW → APPROVED, etc.)
- **document_email_events** — Emails sent related to a document
- **document_shares** — Sharing permissions for documents
- **document_change_requests** — Formal change requests on documents
- **analysis_results** — AI analysis results stored per scope (document/workspace/project)

---

## 7. Workspace Tables

### 7.1 workspaces

**Purpose**: A workspace is a collaborative space that can contain files, tasks, and team members. Think of it as a shared drive.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `owner_id` | uuid | FK → users.id, cascade delete |
| `name` | varchar(255) | Not null |
| `description` | text | Nullable |
| `storage_allocated_bytes` | bigint | Default: 16106127360 (15 GB) |
| `storage_used_bytes` | bigint | Default: 0 |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Indexes**:
- `workspaces_owner_idx` — on (owner_id)

---

### 7.2 workspace_members

**Purpose**: Tracks who belongs to a workspace and their role.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `workspace_id` | uuid | FK → workspaces.id, cascade delete |
| `user_id` | uuid | FK → users.id, cascade delete |
| `role` | varchar(20) | Default: "viewer" |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Indexes**:
- `workspace_members_workspace_idx` — on (workspace_id)
- `workspace_members_user_idx` — on (user_id)
- `workspace_members_unique_idx` — unique on (workspace_id, user_id)

---

### 7.3 workspace_task_lists

**Purpose**: Task lists within a workspace. Organizes tasks into logical groups.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `workspace_id` | uuid | FK → workspaces.id, cascade delete |
| `name` | varchar(255) | Not null |
| `description` | text | Nullable |
| `position` | integer | Default: 0, for ordering |
| `is_archived` | boolean | Default: false |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 7.4 workspace_task_statuses

**Purpose**: Custom status columns for tasks (like "To Do", "In Progress", "Done").

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `workspace_id` | uuid | FK → workspaces.id, cascade delete |
| `name` | varchar(80) | Not null |
| `color` | varchar(32) | Nullable, for UI |
| `position` | integer | Default: 0 |
| `is_default` | boolean | Default: false |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 7.5 workspace_tasks

**Purpose**: Individual tasks within a workspace. Can be nested (subtasks via parent_task_id).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `workspace_id` | uuid | FK → workspaces.id, cascade delete |
| `list_id` | uuid | FK → workspace_task_lists.id, cascade delete |
| `status_id` | uuid | FK → workspace_task_statuses.id, restrict delete |
| `parent_task_id` | uuid | FK → self (subtasks), cascade delete |
| `created_by_user_id` | uuid | FK → users.id, set null |
| `title` | varchar(500) | Not null |
| `description` | text | Nullable |
| `priority` | workspace_task_priority enum | urgent/high/medium/low/none |
| `start_date` | timestamp | Nullable |
| `due_date` | timestamp | Nullable |
| `position` | integer | Default: 0 |
| `is_archived` | boolean | Default: false |
| `completed_at` | timestamp | Nullable |
| `attachments` | jsonb | Nullable |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

**Indexes**:
- `workspace_tasks_workspace_idx` — on (workspace_id)
- `workspace_tasks_list_idx` — on (list_id)
- `workspace_tasks_status_idx` — on (status_id)
- `workspace_tasks_due_date_idx` — on (workspace_id, due_date)
- `workspace_tasks_priority_idx` — on (workspace_id, priority)
- `workspace_tasks_active_idx` — on (workspace_id, is_archived)
- `workspace_tasks_parent_idx` — on (parent_task_id)

---

### 7.6 Additional Workspace Tables

- **workspace_task_assignees** — Who is assigned to each task (many-to-many)
- **workspace_task_comments** — Comments on tasks (threaded)
- **workspace_task_activity** — Audit trail of task changes
- **workspace_activity** — General workspace activity log
- **folders** (driveFolders) — Folders in the workspace drive
- **files** (driveFiles) — Files in the workspace drive
- **file_versions** — Version history for drive files
- **file_activity** — Audit trail for file actions

---

## 8. Chat Tables

### 8.1 chat_sessions

**Purpose**: Groups multiple chats together. Created when a user clicks "New Chat". A session can be tied to a project or workspace.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `project_id` | uuid | FK → projects.id, cascade delete, nullable |
| `workspace_id` | uuid | FK → workspaces.id, cascade delete, nullable |
| `title` | varchar(500) | Nullable |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 8.2 chats

**Purpose**: Individual chat conversations. Each chat belongs to a session.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `session_id` | uuid | FK → chat_sessions.id, cascade delete, nullable |
| `user_id` | uuid | FK → users.id, cascade delete |
| `project_id` | uuid | FK → projects.id, cascade delete, nullable |
| `workspace_id` | uuid | FK → workspaces.id, cascade delete, nullable |
| `title` | varchar(500) | Nullable |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 8.3 chat_messages

**Purpose**: Every message in a chat conversation. Messages can be from the user or the AI.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `chat_id` | uuid | FK → chats.id, cascade delete |
| `role` | varchar(50) | "user", "assistant", "system" |
| `content` | text | Nullable |
| `files` | jsonb | Attached files |
| `workflow` | jsonb | Workflow data if triggered |
| `annotations` | jsonb | AI annotations |
| `created_at` | timestamp | Default: now() |

**Indexes**:
- `chat_messages_chat_idx` — on (chat_id)

---

### 8.4 Additional Chat Tables

- **chat_document_extractions** — Structured data extracted from documents during chat (invoices, etc.)
- **chat_interview_state** — State for guided interview-style conversations
- **conversation_context** — AI conversation memory (short-term memory, legal entities, document references)
- **user_memories** — Long-term user preferences and facts
- **learning_events** — Events used to improve AI responses over time

---

## 9. Workflow Tables

### 9.1 workflows

**Purpose**: Reusable workflow templates. A workflow defines a sequence of steps for document processing.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `title` | varchar(500) | Not null |
| `type` | varchar(50) | Not null |
| `prompt_md` | text | Markdown prompt |
| `columns_config` | jsonb | Column configuration |
| `practice` | varchar(255) | Practice area |
| `is_system` | boolean | Default: false |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 9.2 workflow_shares

**Purpose**: Sharing workflows with other users via email.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `workflow_id` | uuid | FK → workflows.id, cascade delete |
| `shared_by_user_id` | uuid | FK → users.id, cascade delete |
| `shared_with_email` | varchar(255) | Not null |
| `allow_edit` | boolean | Default: false |
| `created_at` | timestamp | Default: now() |

---

### 9.3 hidden_workflows

**Purpose**: Tracks which workflows a user has chosen to hide from their list.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `workflow_id` | uuid | FK → workflows.id, cascade delete |
| `created_at` | timestamp | Default: now() |

---

### 9.4 Workflow Builder Tables

The workflow builder is a visual flow editor. These tables support it:

- **workflow_builder_flows** — Visual workflow definitions with nodes and edges (like a flowchart)
- **workflow_builder_runs** — Execution instances of a flow
- **workflow_builder_node_runs** — Individual node executions within a run
- **workflow_builder_templates** — Pre-built workflow templates users can start from

---

## 10. RAG Tables

RAG stands for Retrieval-Augmented Generation. These tables power the AI's ability to search through documents.

### 10.1 rag_collections

**Purpose**: A collection of indexed documents that the AI can search through. Scoped to personal, project, workspace, or document level.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `scope_type` | rag_scope_type enum | personal/project/workspace/document |
| `scope_id` | uuid | The ID of the scope entity |
| `owner_user_id` | uuid | FK → users.id, set null |
| `collection_name` | varchar(255) | Unique, not null |
| `display_name` | varchar(255) | Nullable |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 10.2 rag_source_index_entries

**Purpose**: Tracks which documents have been indexed into a RAG collection and their status.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `collection_id` | uuid | FK → rag_collections.id, cascade delete |
| `scope_type` | rag_scope_type enum | Not null |
| `scope_id` | uuid | Not null |
| `source_type` | rag_source_type enum | document/drive_file |
| `source_id` | uuid | The document or file ID |
| `version_id` | uuid | Which version was indexed |
| `user_id` | uuid | FK → users.id, set null |
| `project_id` | uuid | FK → projects.id, cascade delete |
| `workspace_id` | uuid | FK → workspaces.id, cascade delete |
| `filename` | varchar(500) | Not null |
| `mime_type` | varchar(255) | Not null |
| `storage_path` | varchar(1000) | Not null |
| `checksum` | varchar(128) | Nullable |
| `status` | rag_source_status enum | pending/indexed/failed/skipped_unsupported |
| `last_error` | text | Nullable |
| `retryable` | boolean | Default: false |
| `indexed_at` | timestamp | Nullable |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

## 11. Evidence & Citation Tables

### 11.1 evidence_source_refs

**Purpose**: Stores evidence references that back up AI claims. When the AI makes a statement, it can cite specific passages from documents.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `evidence_id` | varchar(120) | Unique, not null |
| `user_id` | uuid | FK → users.id, set null |
| `chat_id` | uuid | FK → chats.id, cascade delete |
| `tool_call_id` | varchar(120) | Nullable |
| `query` | text | The search query that found this |
| `source_type` | evidence_source_type enum | document/drive_file/web |
| `source_id` | uuid | Nullable |
| `version_id` | uuid | Nullable |
| `chunk_id` | varchar(160) | Not null |
| `quote` | text | The actual text passage |
| `quote_hash` | varchar(64) | Not null |
| `page_number` | integer | Nullable |
| `start_offset` | integer | Nullable |
| `end_offset` | integer | Nullable |
| `section_title` | text | Nullable |
| `filename` | varchar(500) | Nullable |
| `deleted_at` | timestamp | Soft delete |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 11.2 generated_citation_evidence

**Purpose**: Links evidence to specific citations in generated documents. When the AI writes a document and cites a source, this table tracks which evidence supports which citation.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `evidence_ref_id` | uuid | FK → evidence_source_refs.id, cascade delete |
| `evidence_id` | varchar(120) | Not null |
| `user_id` | uuid | FK → users.id, set null |
| `chat_id` | uuid | FK → chats.id, cascade delete |
| `chat_message_id` | uuid | FK → chat_messages.id, cascade delete |
| `generated_document_id` | uuid | FK → documents.id, cascade delete |
| `generated_document_version_id` | uuid | FK → document_versions.id, cascade delete |
| `citation_ref` | varchar(80) | Not null |
| `section_title` | text | Nullable |
| `paragraph_index` | integer | Nullable |
| `claim_text` | text | Nullable |
| `created_at` | timestamp | Default: now() |

---

### 11.3 evidence_navigation_events

**Purpose**: Tracks when users click on evidence citations to view the source. Helps understand which evidence is most useful.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `event_name` | varchar(80) | Not null |
| `evidence_id` | varchar(120) | Nullable |
| `user_id` | uuid | FK → users.id, set null |
| `chat_id` | uuid | FK → chats.id, cascade delete |
| `generated_document_id` | uuid | FK → documents.id, cascade delete |
| `metadata` | jsonb | Nullable |
| `created_at` | timestamp | Default: now() |

---

## 12. Approval Tables

### 12.1 approval_approvers

**Purpose**: Defines who can approve documents or workspace items. Each approver has a specific role (Legal Head, Brand Head, etc.).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `subject_type` | approval_subject_type enum | document/drive_file/workspace |
| `subject_id` | uuid | The item being approved |
| `role` | approval_role enum | bd_head/legal_head/brand_head/etc. |
| `approver_name` | varchar(255) | Not null |
| `approver_email` | varchar(255) | Not null |
| `is_required` | boolean | Default: false |
| `created_by_user_id` | uuid | FK → users.id, set null |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 12.2 approval_requests

**Purpose**: An actual approval request sent to an approver. Contains a unique token for the approval link.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `subject_type` | approval_subject_type enum | Not null |
| `subject_id` | uuid | Not null |
| `approver_id` | uuid | FK → approval_approvers.id, set null |
| `role` | approval_role enum | Not null |
| `approver_name` | varchar(255) | Not null |
| `approver_email` | varchar(255) | Not null |
| `status` | approval_status enum | pending/approved/rejected/cancelled |
| `token` | varchar(128) | Unique, for approval link |
| `decision_note` | text | Nullable |
| `requested_by_user_id` | uuid | FK → users.id, set null |
| `decided_at` | timestamp | Nullable |
| `expires_at` | timestamp | Not null |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 12.3 approval_request_items

**Purpose**: Individual items within an approval request. A single request can cover multiple changes.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `request_id` | uuid | FK → approval_requests.id, cascade delete |
| `subject_type` | approval_subject_type enum | Not null |
| `subject_id` | uuid | Not null |
| `role` | approval_role enum | Not null |
| `item_type` | varchar(50) | Not null |
| `item_id` | uuid | Nullable |
| `title` | varchar(500) | Not null |
| `original_text` | text | What was there before |
| `new_text` | text | What it should be changed to |
| `reason` | text | Why the change is needed |
| `metadata` | jsonb | Nullable |
| `created_at` | timestamp | Default: now() |

---

## 13. Compliance Tables

### 13.1 compliance_reviews

**Purpose**: A compliance review checks documents against rules and regulations. Each review produces a score and detailed results.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `project_id` | uuid | FK → projects.id, cascade delete, nullable |
| `workspace_id` | uuid | FK → workspaces.id, cascade delete, nullable |
| `primary_document_id` | uuid | FK → documents.id, cascade delete, nullable |
| `title` | varchar(255) | Nullable |
| `status` | compliance_review_status enum | pending/running/completed/failed |
| `compliance_score` | integer | Nullable |
| `results` | jsonb | Detailed results |
| `ai_insights` | jsonb | AI-generated insights |
| `rag_collection_name` | varchar(255) | Nullable |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 13.2 compliance_review_rules

**Purpose**: Individual rules checked during a compliance review.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `review_id` | uuid | FK → compliance_reviews.id, cascade delete |
| `content` | text | The rule text |
| `status` | compliance_rule_status enum | pending/compliant/non_compliant/partial/error |
| `result` | jsonb | Detailed result |
| `sort_order` | integer | Default: 0 |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 13.3 compliance_review_questions

**Purpose**: Questions asked during a compliance review (similar to rules but phrased as questions).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `review_id` | uuid | FK → compliance_reviews.id, cascade delete |
| `content` | text | The question |
| `status` | compliance_rule_status enum | Same as rules |
| `result` | jsonb | Nullable |
| `sort_order` | integer | Default: 0 |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 13.4 compliance_review_supporting_docs

**Purpose**: Links supporting documents to a compliance review.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `review_id` | uuid | FK → compliance_reviews.id, cascade delete |
| `document_id` | uuid | FK → documents.id, cascade delete |
| `created_at` | timestamp | Default: now() |

---

### 13.5 rulebooks

**Purpose**: Collections of rules that can be reused across compliance reviews.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete, nullable |
| `workspace_id` | uuid | FK → workspaces.id, cascade delete, nullable |
| `name` | varchar(255) | Not null |
| `description` | text | Nullable |
| `category` | varchar(100) | Nullable |
| `is_system` | boolean | Default: false |
| `is_active` | boolean | Default: true |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 13.6 rulebook_rules

**Purpose**: Individual rules within a rulebook.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `rulebook_id` | uuid | FK → rulebooks.id, cascade delete |
| `text` | text | Not null |
| `guidance` | text | Nullable, how to comply |
| `severity` | rulebook_rule_severity enum | required/recommended/optional |
| `category` | varchar(100) | Nullable |
| `order_index` | integer | Default: 0 |
| `created_at` | timestamp | Default: now() |

---

## 14. Email Tables

### 14.1 email_connector_accounts

**Purpose**: Connected email accounts (Gmail, Outlook). Stores encrypted OAuth tokens for sending emails on behalf of the user.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `provider` | email_connector_provider enum | gmail/outlook |
| `provider_account_id` | varchar(255) | Not null |
| `email` | varchar(255) | Not null |
| `display_name` | varchar(255) | Nullable |
| `encrypted_access_token` | text | Not null |
| `access_token_iv` | text | Not null |
| `access_token_auth_tag` | text | Not null |
| `encrypted_refresh_token` | text | Nullable |
| `refresh_token_iv` | text | Nullable |
| `refresh_token_auth_tag` | text | Nullable |
| `expires_at` | timestamp | Nullable |
| `scopes` | jsonb | Default: [] |
| `metadata` | jsonb | Nullable |
| `is_active` | boolean | Default: true |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 14.2 email_connector_preferences

**Purpose**: Per-user email preferences, like which account is the default.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete, unique |
| `default_account_id` | uuid | FK → email_connector_accounts.id, set null |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 14.3 email_drafts

**Purpose**: Email drafts created by the user or generated by AI. Supports reply/forward chains.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `conversation_id` | uuid | FK → chats.id, set null |
| `account_id` | uuid | FK → email_connector_accounts.id, cascade delete |
| `type` | email_draft_type enum | NEW/REPLY/FORWARD |
| `to` | jsonb | Array of email addresses |
| `cc` | jsonb | Array of email addresses |
| `bcc` | jsonb | Array of email addresses |
| `subject` | varchar(998) | Not null |
| `body` | text | Not null |
| `is_html` | boolean | Default: false |
| `reply_to_message_id` | text | Nullable |
| `reply_to_thread_id` | text | Nullable |
| `reply_all` | boolean | Default: false |
| `thread_id` | uuid | Default: gen_random_uuid() |
| `version` | integer | Default: 1 |
| `parent_draft_id` | uuid | FK → self, set null |
| `provider_draft_id` | text | Nullable |
| `provider_message_id` | text | Nullable |
| `status` | email_draft_status enum | DRAFT/SENDING/SENT/FAILED/CANCELLED |
| `sent_at` | timestamp | Nullable |
| `error` | text | Nullable |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 14.4 email_draft_attachments

**Purpose**: Files attached to email drafts. Links to drive files.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `draft_id` | uuid | FK → email_drafts.id, cascade delete |
| `file_id` | uuid | FK → files.id, cascade delete |
| `type` | email_draft_attachment_type enum | ATTACHMENT/INLINE |
| `content_id` | varchar(255) | For inline images |
| `created_at` | timestamp | Default: now() |

---

### 14.5 email_draft_sendRequests

**Purpose**: Tracks send requests for email drafts. Each send attempt is recorded here.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `draft_id` | uuid | FK → email_drafts.id, cascade delete |
| `user_id` | uuid | FK → users.id, cascade delete |
| `status` | email_draft_send_status enum | pending/sending/sent/failed/cancelled |
| `provider_message_id` | text | Nullable |
| `payload_snapshot` | jsonb | What was sent |
| `error` | text | Nullable |
| `requested_at` | timestamp | Default: now() |
| `sent_at` | timestamp | Nullable |
| `updated_at` | timestamp | Default: now() |

---

## 15. Sharing Tables

### 15.1 document_shares

**Purpose**: Sharing permissions for documents with specific users or emails.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `document_id` | uuid | FK → documents.id, cascade delete |
| `user_id` | uuid | FK → users.id, cascade delete, nullable |
| `email` | varchar(255) | Not null |
| `role` | share_role enum | admin/editor/viewer |
| `invited_by_user_id` | uuid | FK → users.id, set null |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 15.2 share_invitations

**Purpose**: Pending sharing invitations. Supports sharing documents, projects, and workspaces via email links.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `token_hash` | varchar(128) | Unique, for invitation link |
| `resource_type` | share_resource_type enum | document/project/workspace |
| `document_id` | uuid | FK → documents.id, cascade delete, nullable |
| `project_id` | uuid | FK → projects.id, cascade delete, nullable |
| `workspace_id` | uuid | FK → workspaces.id, cascade delete, nullable |
| `email` | varchar(255) | Not null |
| `role` | share_role enum | admin/editor/viewer |
| `status` | share_invitation_status enum | pending/accepted/revoked/expired |
| `invited_by_user_id` | uuid | FK → users.id, cascade delete |
| `accepted_by_user_id` | uuid | FK → users.id, set null |
| `accepted_at` | timestamp | Nullable |
| `expires_at` | timestamp | Not null |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

## 16. System Tables

### 16.1 notifications

**Purpose**: In-app notifications for users. Tracks read/unread status and whether an email was sent.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `icon` | notification_icon_type enum | document/compliance/comment/approval/etc. |
| `title` | varchar(255) | Not null |
| `description` | text | Nullable |
| `read` | boolean | Default: false |
| `on_email` | boolean | Default: false |
| `email_sent_at` | timestamp | Nullable |
| `link` | text | Deep link |
| `resource_type` | varchar(50) | Nullable |
| `resource_id` | uuid | Nullable |
| `actor_user_id` | uuid | FK → users.id, set null |
| `metadata` | jsonb | Nullable |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 16.2 user_activity

**Purpose**: General audit trail of all user actions across the platform.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `action` | varchar(80) | What was done |
| `resource_type` | varchar(50) | Nullable |
| `resource_id` | uuid | Nullable |
| `resource_name` | varchar(500) | Nullable |
| `actor_user_id` | uuid | FK → users.id, set null |
| `actor_name` | varchar(255) | Nullable |
| `details` | jsonb | Nullable |
| `created_at` | timestamp | Default: now() |

---

### 16.3 attention_items

**Purpose**: Items that need the user's attention. Things like pending approvals, compliance issues, invitations, etc.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete |
| `source_type` | attention_item_source_type enum | document_risk/compliance_issue/approval_request/etc. |
| `source_id` | uuid | Nullable |
| `secondary_source_id` | uuid | Nullable |
| `severity` | varchar(20) | Not null |
| `title` | varchar(500) | Not null |
| `description` | text | Nullable |
| `metadata` | jsonb | Nullable |
| `status` | attention_item_status enum | pending/viewed/resolved/dismissed |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |
| `resolved_at` | timestamp | Nullable |

---

### 16.4 service_healthChecks

**Purpose**: Health check results for external services. Tracks uptime and response times.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `service_name` | varchar(100) | Not null |
| `status` | health_check_status enum | operational/degraded/down |
| `response_time_ms` | integer | Nullable |
| `error_message` | text | Nullable |
| `checked_at` | timestamp | Default: now() |

---

## 17. Agent Tables

These tables are defined in `packages/db/migrations/` and support the AI agent orchestration system.

### 17.1 agent_jobs

**Purpose**: A high-level AI job. When the AI is asked to do something complex (like "review this contract"), it creates a job that gets broken into sub-tasks.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `project_id` | uuid | FK → projects.id, nullable |
| `user_id` | uuid | FK → users.id, nullable |
| `workflow_type` | text | Not null |
| `status` | text | pending/planning/running/paused_for_input/completed/failed |
| `title` | text | Nullable |
| `plan` | jsonb | The execution plan |
| `context` | jsonb | Context data |
| `error` | text | Error message if failed |
| `started_at` | timestamptz | Nullable |
| `completed_at` | timestamptz | Nullable |
| `created_at` | timestamptz | Default: now() |
| `updated_at` | timestamptz | Default: now() |

**Indexes**:
- `agent_jobs_project_id_idx` — on (project_id)
- `agent_jobs_status_idx` — on (status)

---

### 17.2 agent_sub_tasks

**Purpose**: Individual steps within an AI job. Each sub-task is a specific piece of work.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `job_id` | uuid | FK → agent_jobs.id, cascade delete |
| `index` | int | Execution order |
| `practice_area` | text | Nullable |
| `status` | text | pending/running/completed/failed |
| `prompt` | text | What to do |
| `result` | jsonb | Output |
| `started_at` | timestamptz | Nullable |
| `completed_at` | timestamptz | Nullable |

---

### 17.3 agent_clarifications

**Purpose**: Questions the AI asks the user when it needs more information to complete a job.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `job_id` | uuid | FK → agent_jobs.id, nullable |
| `question` | text | Not null |
| `options` | jsonb | Multiple choice options |
| `answer` | text | Nullable, user's response |
| `answered_at` | timestamptz | Nullable |
| `created_at` | timestamptz | Default: now() |

---

### 17.4 agent_artifacts

**Purpose**: Output files produced by an AI job (documents, reports, etc.).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `job_id` | uuid | FK → agent_jobs.id, nullable |
| `type` | text | Not null |
| `file_id` | uuid | FK → files.id, nullable |
| `content` | jsonb | Nullable |
| `created_at` | timestamptz | Default: now() |

---

### 17.5 agent_audit_log

**Purpose**: Detailed audit trail for every action within an AI job.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `job_id` | uuid | FK → agent_jobs.id, cascade delete |
| `event_type` | text | Not null |
| `actor` | text | Not null |
| `payload` | jsonb | Nullable |
| `created_at` | timestamptz | Default: now() |

---

## 18. Template Tables

### 18.1 templates

**Purpose**: Document templates that users can create or upload. Templates have placeholder fields that get filled in.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | FK → users.id, cascade delete, nullable |
| `name` | varchar(255) | Not null |
| `category` | varchar(100) | Not null |
| `description` | text | Nullable |
| `content_html` | text | Not null, the template HTML |
| `fields` | jsonb | Placeholder field definitions |
| `source_filename` | varchar(500) | Original upload filename |
| `source_storage_path` | varchar(1000) | Where the source file is stored |
| `source_mime_type` | varchar(255) | Nullable |
| `source_checksum` | varchar(128) | Nullable |
| `source_metadata` | jsonb | Nullable |
| `is_created_by_user` | boolean | Default: true |
| `created_at` | timestamp | Default: now() |
| `updated_at` | timestamp | Default: now() |

---

### 18.2 template_variables

**Purpose**: Dynamic variables used within templates. Defines what values can be substituted.

Note: This table is referenced in the requirements but the schema uses `fields` jsonb column in templates instead of a separate table. The template variables are stored as part of the `fields` jsonb column in the templates table.

---

## 19. Key Enums

All enum types defined in the schema:

| Enum Name | Values |
|---|---|
| `approval_subject_type` | document, drive_file, workspace |
| `approval_role` | bd_head, legal_head, brand_head, brand_ceo, finance_head, it_head, ceo_yum |
| `approval_status` | pending, approved, rejected, cancelled |
| `document_lifecycle_status` | DRAFT, IN_REVIEW, PENDING_APPROVAL, APPROVED, FINALIZED |
| `document_role` | DRAFTER, REVIEWER, APPROVER |
| `document_chat_role_badge` | DRAFTER, REVIEWER, APPROVER, OWNER_ADMIN, AI |
| `document_ai_label` | AI_MIKE, AI_MIKE_PRISM |
| `document_email_status` | pending, sent, failed, dev_fallback |
| `share_resource_type` | document, project, workspace |
| `share_role` | admin, editor, viewer |
| `share_invitation_status` | pending, accepted, revoked, expired |
| `document_change_request_status` | pending, approved, rejected, cancelled |
| `access_request_status` | pending, approved, rejected |
| `auth_email_status` | pending, sent, failed, dev_fallback |
| `rag_scope_type` | personal, project, workspace, document |
| `rag_source_type` | document, drive_file |
| `rag_source_status` | pending, indexed, failed, skipped_unsupported |
| `evidence_source_type` | document, drive_file, web |
| `email_connector_provider` | gmail, outlook |
| `email_draft_type` | NEW, REPLY, FORWARD |
| `email_draft_status` | DRAFT, SENDING, SENT, FAILED, CANCELLED |
| `email_draft_attachment_type` | ATTACHMENT, INLINE |
| `email_draft_send_status` | pending, sending, sent, failed, cancelled |
| `workspace_task_priority` | urgent, high, medium, low, none |
| `workspace_task_activity_action` | task_list_created, task_list_updated, task_list_archived, task_list_deleted, task_status_created, task_status_updated, task_status_deleted, task_created, task_updated, task_moved, task_assigned, task_unassigned, task_deleted, task_completed, task_comment_created, task_comment_updated, task_comment_deleted |
| `compliance_review_status` | pending, running, completed, failed |
| `compliance_rule_status` | pending, compliant, non_compliant, partial, error |
| `workflow_builder_run_status` | pending, running, paused, completed, failed, cancelled |
| `workflow_builder_node_run_status` | pending, running, completed, failed, skipped |
| `health_check_status` | operational, degraded, down |
| `attention_item_source_type` | document_risk, compliance_issue, compliance_question, project_invitation, document_invitation, approval_request, workspace_invitation, document_change_request, access_request |
| `attention_item_status` | pending, viewed, resolved, dismissed |
| `notification_icon_type` | document, compliance, comment, approval, workspace, alert, share, mention, system |
| `rulebook_rule_severity` | required, recommended, optional |

---

## 20. Relationships Map

Below is a comprehensive list of all foreign key relationships:

```
users
├── auth_accounts.user_id → users.id (CASCADE)
├── auth_sessions.user_id → users.id (CASCADE)
├── user_profiles.user_id → users.id (CASCADE)
├── user_api_keys.user_id → users.id (CASCADE)
├── projects.user_id → users.id (CASCADE)
├── documents.user_id → users.id (CASCADE)
├── workspaces.owner_id → users.id (CASCADE)
├── chats.user_id → users.id (CASCADE)
├── chat_sessions.user_id → users.id (CASCADE)
├── workflows.user_id → users.id (CASCADE)
├── templates.user_id → users.id (CASCADE)
├── notifications.user_id → users.id (CASCADE)
├── user_activity.user_id → users.id (CASCADE)
├── attention_items.user_id → users.id (CASCADE)
├── compliance_reviews.user_id → users.id (CASCADE)
├── email_connector_accounts.user_id → users.id (CASCADE)
├── email_drafts.user_id → users.id (CASCADE)
├── rulebooks.user_id → users.id (CASCADE)
├── rag_collections.owner_user_id → users.id (SET NULL)
├── evidence_source_refs.user_id → users.id (SET NULL)
├── learning_events.user_id → users.id (CASCADE)
├── conversation_context.user_id → users.id (CASCADE)
└── user_memories.user_id → users.id (CASCADE)

projects
├── documents.project_id → projects.id (CASCADE)
├── project_members.project_id → projects.id (CASCADE)
├── project_subfolders.project_id → projects.id (CASCADE)
├── chats.project_id → projects.id (CASCADE)
├── chat_sessions.project_id → projects.id (CASCADE)
├── share_invitations.project_id → projects.id (CASCADE)
├── tabular_reviews.project_id → projects.id (CASCADE)
├── compliance_reviews.project_id → projects.id (CASCADE)
├── workflow_builder_flows.project_id → projects.id (SET NULL)
├── agent_jobs.project_id → projects.id (no cascade)
├── rag_source_index_entries.project_id → projects.id (CASCADE)
└── analysis_results (via scope_id)

workspaces
├── workspace_members.workspace_id → workspaces.id (CASCADE)
├── workspace_task_lists.workspace_id → workspaces.id (CASCADE)
├── workspace_task_statuses.workspace_id → workspaces.id (CASCADE)
├── workspace_tasks.workspace_id → workspaces.id (CASCADE)
├── workspace_task_activity.workspace_id → workspaces.id (CASCADE)
├── workspace_activity.workspace_id → workspaces.id (CASCADE)
├── driveFolders.workspace_id → workspaces.id (CASCADE)
├── driveFiles.workspace_id → workspaces.id (CASCADE)
├── share_invitations.workspace_id → workspaces.id (CASCADE)
├── tabular_reviews.workspace_id → workspaces.id (CASCADE)
├── compliance_reviews.workspace_id → workspaces.id (CASCADE)
├── access_requests.workspace_id → workspaces.id (CASCADE)
├── rulebooks.workspace_id → workspaces.id (CASCADE)
├── chats.workspace_id → workspaces.id (CASCADE)
├── chat_sessions.workspace_id → workspaces.id (CASCADE)
└── rag_source_index_entries.workspace_id → workspaces.id (CASCADE)

documents
├── document_versions.document_id → documents.id (CASCADE)
├── document_comments.document_id → documents.id (CASCADE)
├── document_edits.document_id → documents.id (CASCADE)
├── document_activity.document_id → documents.id (CASCADE)
├── document_members.document_id → documents.id (CASCADE)
├── document_chat_messages.document_id → documents.id (CASCADE)
├── document_state_transitions.document_id → documents.id (CASCADE)
├── document_email_events.document_id → documents.id (CASCADE)
├── document_shares.document_id → documents.id (CASCADE)
├── document_context_files.document_id → documents.id (CASCADE)
├── document_placeholder_values.document_id → documents.id (CASCADE)
├── document_change_requests.document_id → documents.id (CASCADE)
├── share_invitations.document_id → documents.id (CASCADE)
├── compliance_review_supporting_docs.document_id → documents.id (CASCADE)
├── generated_citation_evidence.generated_document_id → documents.id (CASCADE)
├── evidence_navigation_events.generated_document_id → documents.id (CASCADE)
├── documents.folder_id → project_subfolders.id (SET NULL)
└── documents.project_id → projects.id (CASCADE)

chats
├── chat_messages.chat_id → chats.id (CASCADE)
├── chat_document_extractions.chat_id → chats.id (CASCADE)
├── chat_interview_state.chat_id → chats.id (CASCADE)
├── conversation_context.chat_id → chats.id (CASCADE)
├── learning_events.chat_id → chats.id (CASCADE)
├── evidence_source_refs.chat_id → chats.id (CASCADE)
├── generated_citation_evidence.chat_id → chats.id (CASCADE)
├── evidence_navigation_events.chat_id → chats.id (CASCADE)
├── email_drafts.conversation_id → chats.id (SET NULL)
└── chats.session_id → chat_sessions.id (CASCADE)

workflows
├── workflow_shares.workflow_id → workflows.id (CASCADE)
├── hidden_workflows.workflow_id → workflows.id (CASCADE)
└── workflows.user_id → users.id (CASCADE)

workflow_builder
├── workflow_builder_runs.flow_id → workflow_builder_flows.id (CASCADE)
├── workflow_builder_node_runs.run_id → workflow_builder_runs.id (CASCADE)
└── workflow_builder_flows.user_id → users.id (CASCADE)

email
├── email_connector_preferences.user_id → users.id (CASCADE)
├── email_connector_preferences.default_account_id → email_connector_accounts.id (SET NULL)
├── email_drafts.account_id → email_connector_accounts.id (CASCADE)
├── email_draft_attachments.draft_id → email_drafts.id (CASCADE)
├── email_draft_attachments.file_id → driveFiles.id (CASCADE)
├── email_draft_send_requests.draft_id → email_drafts.id (CASCADE)
└── email_drafts.parent_draft_id → email_drafts.id (SET NULL)

compliance
├── compliance_review_rules.review_id → compliance_reviews.id (CASCADE)
├── compliance_review_questions.review_id → compliance_reviews.id (CASCADE)
├── compliance_review_supporting_docs.review_id → compliance_reviews.id (CASCADE)
└── rulebook_rules.rulebook_id → rulebooks.id (CASCADE)

approval
├── approval_requests.approver_id → approval_approvers.id (SET NULL)
└── approval_request_items.request_id → approval_requests.id (CASCADE)

RAG
├── rag_source_index_entries.collection_id → rag_collections.id (CASCADE)
└── generated_citation_evidence.evidence_ref_id → evidence_source_refs.id (CASCADE)

agent
├── agent_sub_tasks.job_id → agent_jobs.id (CASCADE)
├── agent_clarifications.job_id → agent_jobs.id (no cascade)
├── agent_artifacts.job_id → agent_jobs.id (no cascade)
└── agent_audit_log.job_id → agent_jobs.id (CASCADE)

workspace tasks
├── workspace_tasks.list_id → workspace_task_lists.id (CASCADE)
├── workspace_tasks.status_id → workspace_task_statuses.id (RESTRICT)
├── workspace_tasks.parent_task_id → workspace_tasks.id (CASCADE)
├── workspace_task_assignees.task_id → workspace_tasks.id (CASCADE)
├── workspace_task_comments.task_id → workspace_tasks.id (CASCADE)
└── workspace_task_activity.task_id → workspace_tasks.id (CASCADE)

drive
├── driveFiles.folder_id → driveFolders.id (SET NULL)
├── driveFolders.parent_folder_id → driveFolders.id (CASCADE)
├── fileVersions.file_id → driveFiles.id (CASCADE)
└── fileActivity.file_id → driveFiles.id (CASCADE)
```

---

## 21. Common Query Patterns

### Get all documents in a project with their current version

```sql
SELECT d.*, dv.storage_path, dv.version_number
FROM documents d
LEFT JOIN document_versions dv ON d.current_version_id = dv.id
WHERE d.project_id = ?
ORDER BY d.created_at DESC;
```

### Get a user's workspaces with member count

```sql
SELECT w.*, COUNT(wm.id) as member_count
FROM workspaces w
LEFT JOIN workspace_members wm ON w.id = wm.workspace_id
WHERE w.owner_id = ? OR wm.user_id = ?
GROUP BY w.id;
```

### Get chat messages with user info

```sql
SELECT cm.*, u.full_name, u.email
FROM chat_messages cm
LEFT JOIN chats c ON cm.chat_id = c.id
LEFT JOIN users u ON c.user_id = u.id
WHERE cm.chat_id = ?
ORDER BY cm.created_at ASC;
```

### Get pending approval requests for a document

```sql
SELECT ar.*, aa.approver_name, aa.role
FROM approval_requests ar
LEFT JOIN approval_approvers aa ON ar.approver_id = aa.id
WHERE ar.subject_type = 'document'
  AND ar.subject_id = ?
  AND ar.status = 'pending';
```

### Get tasks in a workspace with assignees

```sql
SELECT wt.*, wts.name as status_name, wts.color,
       json_agg(json_build_object('user_id', wta.user_id, 'name', u.full_name)) as assignees
FROM workspace_tasks wt
JOIN workspace_task_statuses wts ON wt.status_id = wts.id
LEFT JOIN workspace_task_assignees wta ON wt.id = wta.task_id
LEFT JOIN users u ON wta.user_id = u.id
WHERE wt.workspace_id = ?
  AND wt.is_archived = false
GROUP BY wt.id, wts.name, wts.color
ORDER BY wt.position;
```

### Get unread notifications for a user

```sql
SELECT *
FROM notifications
WHERE user_id = ? AND read = false
ORDER BY created_at DESC
LIMIT 50;
```

### Get compliance review with rules

```sql
SELECT cr.*, 
       json_agg(json_build_object('content', crr.content, 'status', crr.status, 'result', crr.result)) as rules
FROM compliance_reviews cr
LEFT JOIN compliance_review_rules crr ON cr.id = crr.review_id
WHERE cr.id = ?
GROUP BY cr.id;
```

---

## 22. Database Migrations

### Migration History

| Migration | Description | File |
|---|---|---|
| Initial | Core tables (users, projects, documents, chats, etc.) | Schema file |
| 0001 | Workflow builder tables | `backend/migrations/0001_workflow_builder.sql` |
| 0002 | Email connectors, workspace tasks | `backend/migrations/0002_assistant_tools_tasks_email.sql` |
| 20240601 | Agent orchestration (jobs, subtasks, artifacts) | `packages/db/migrations/20240601_agent_orchestration.sql` |
| 20240602 | Agent audit log | `packages/db/migrations/20240602_agent_audit_log.sql` |

### Migration Conventions

1. **Idempotent creation**: All migrations use `CREATE TABLE IF NOT EXISTS` and `CREATE INDEX IF NOT EXISTS`
2. **Enum creation**: Enums are wrapped in `DO $$ BEGIN ... EXCEPTION WHEN duplicate_object THEN null; END $$;` blocks
3. **Foreign keys**: Explicitly defined with ON DELETE behavior
4. **Timestamps**: Most tables have `created_at` and `updated_at` with `DEFAULT now()`
5. **Primary keys**: All use `uuid` with `gen_random_uuid()` or `uuid_generate_v4()`

### Adding New Migrations

1. Edit `backend/src/db/schema.ts` to add/modify tables
2. Run `npx drizzle-kit generate` to create the migration SQL
3. Review the generated SQL in `backend/drizzle/`
4. Apply with `npx drizzle-kit migrate`

---

## 23. Performance Considerations

### Indexing Strategy

The database uses indexes strategically:

1. **Foreign key indexes**: Every FK column has an index for join performance
2. **Composite indexes**: For common query patterns (e.g., `workspace_tasks_workspace_idx` on (workspace_id, is_archived))
3. **Unique indexes**: For data integrity (e.g., `user_api_keys_user_provider_idx`)
4. **Partial indexes**: For filtered queries (e.g., `access_requests_workspace_requester_pending_idx` where status = 'pending')
5. **覆盖索引**: Some indexes cover multiple columns to avoid table lookups

### Query Optimization Tips

1. **Always filter by indexed columns**: Use `project_id`, `workspace_id`, `user_id` in WHERE clauses
2. **Use LIMIT**: Always limit result sets, especially for chat messages and activity logs
3. **Avoid N+1 queries**: Use JOINs instead of multiple queries
4. **Use jsonb wisely**: JSONB columns are great for flexible data but avoid querying deep into them frequently
5. **Pagination**: Use cursor-based pagination (created_at, id) instead of OFFSET for large datasets

### Storage Considerations

- Default storage limit: 15 GB per user/workspace
- File storage paths are tracked in `storage_path` columns
- Soft deletes used for some tables (evidence_source_refs.deleted_at)

---

## 24. Data Integrity

### Cascading Deletes

The database uses several cascade strategies:

| Strategy | When Used | Example |
|---|---|---|
| **CASCADE** | When child data is meaningless without parent | chat_messages when chat is deleted |
| **SET NULL** | When the reference is optional | document.folder_id when folder is deleted |
| **RESTRICT** | When deletion should be prevented | workspace_tasks.status_id (can't delete a status that's in use) |

### Unique Constraints

Key unique constraints enforce data integrity:

- `users.email` — One account per email
- `user_api_keys.(user_id, provider)` — One key per provider per user
- `workspace_members.(workspace_id, user_id)` — No duplicate memberships
- `project_members.(project_id, email)` — No duplicate invitations
- `approval_requests.token` — One token per approval request
- `rag_collections.collection_name` — Unique collection names

### Validation Rules

1. **NOT NULL**: Critical fields like email, name, status are required
2. **Enum types**: Status fields use PostgreSQL enums to enforce valid values
3. **Default values**: Most boolean fields default to false, most timestamps default to now()
4. **Check constraints**: Agent status uses CHECK constraints for valid values

---

## 25. Key Files Reference

| File | Purpose | Lines |
|---|---|---|
| `backend/src/db/schema.ts` | Complete database schema definition | 2525 |
| `backend/src/db/index.ts` | Database connection and pool setup | 23 |
| `backend/drizzle.config.ts` | Drizzle ORM configuration | 10 |
| `backend/migrations/0001_workflow_builder.sql` | Workflow builder migration | 95 |
| `backend/migrations/0002_assistant_tools_tasks_email.sql` | Email and tasks migration | 251 |
| `packages/db/migrations/20240601_agent_orchestration.sql` | Agent tables migration | 81 |
| `packages/db/migrations/20240602_agent_audit_log.sql` | Agent audit log migration | 13 |

### Schema Organization

The schema file is organized into clear sections:

1. **Enums** (lines 1-200) — All enum type definitions
2. **Auth Tables** (lines 200-345) — User authentication
3. **User Profile Tables** (lines 280-345) — Extended user data
4. **Project Tables** (lines 345-415) — Project management
5. **Document Tables** (lines 415-780) — Document management
6. **RAG Tables** (lines 815-900) — AI search indexing
7. **Evidence Tables** (lines 900-1000) — Citation tracking
8. **Approval Tables** (lines 1000-1100) — Governance workflows
9. **Workspace Tables** (lines 1100-1615) — Collaborative workspaces
10. **Chat Tables** (lines 1615-1805) — AI conversations
11. **Tabular Review Tables** (lines 1805-1895) — Spreadsheet-like reviews
12. **Workflow Tables** (lines 1895-1960) — Process automation
13. **Template Tables** (lines 1960-1990) — Document templates
14. **Compliance Tables** (lines 1990-2105) — Regulatory compliance
15. **Workflow Builder Tables** (lines 2105-2230) — Visual flow editor
16. **System Tables** (lines 2230-2480) — Health, attention, notifications
17. **Rulebook Tables** (lines 2480-2525) — Compliance rule collections
