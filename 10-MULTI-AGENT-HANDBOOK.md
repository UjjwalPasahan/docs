# Multi-Agent Handbook

## 1. What This Document Is

This is the complete reference for every agent in the Mike legal AI platform. It explains what each agent does, what it receives, what it produces, what tools it can use, how it handles failures, and how agents work together. If you need to understand, modify, debug, or extend any part of the multi-agent system, this is the document to read.

The multi-agent system is the core intelligence of Mike. When a user asks a complex legal question, the system does not rely on a single AI model to do everything. Instead, it breaks the request into tasks and assigns each task to a specialist agent. A coordinator manages the flow, and a final synthesizer combines all results into one coherent response.

---

## 2. Multi-Agent Architecture Overview

### Why Multiple Agents?

Imagine a law firm with one lawyer who handles everything: research, drafting, compliance, document review, client communication. That lawyer would be overwhelmed and would make mistakes. Now imagine a firm with specialists: a researcher, a drafter, a compliance officer, a document clerk. Each person is excellent at their job. A senior partner coordinates them.

Mike works the same way.

**Single Agent Approach:**
- One AI model handles research, drafting, compliance, and document review
- The model must switch context constantly
- Quality degrades as tasks become more complex
- No specialization or deep expertise

**Multi-Agent Approach:**
- Each agent is a specialist focused on one type of legal work
- Agents use only the tools they need
- Agents can run in parallel when tasks are independent
- A coordinator manages dependencies and combines results
- Quality is higher because each agent has a focused system prompt and toolset

### The Team

Mike has these specialist agents:

| Agent | Role | Analogy |
|-------|------|---------|
| Research Agent | Finds legal information, cases, statutes | Law clerk / researcher |
| Document Agent | Reads, compares, and edits documents | Document clerk / paralegal |
| Compliance Agent | Checks rules and regulations | Compliance officer |
| Drafting Agent | Creates legal documents | Junior associate / drafter |
| Self-Critique Agent | Reviews AI output for errors | Senior partner review |
| Generic Agent | Handles simple single-step requests | General assistant |

### The Coordinator

The Coordinator (implemented in `agentCoordinator.ts`) is the project manager. It:

1. Receives a plan with multiple tasks
2. Determines which tasks can run in parallel and which must run sequentially
3. Executes tasks in the correct order
4. Passes results between agents via shared context
5. Handles failures and retries
6. Returns a combined result

### The Pipeline

The full orchestration pipeline works in this order:

1. **Intent Classification** - Determines if the request needs multi-agent orchestration
2. **Task Planning** - Breaks the request into subtasks with dependencies
3. **Agent Execution** - Runs agents according to the plan
4. **Response Synthesis** - Combines agent outputs into one response
5. **Output Validation** - Checks the response for quality issues
6. **Self-Critique** - Reviews and improves the response if validation found issues
7. **Final Delivery** - Sends the response to the user

---

## 3. Base Agent Contract

Every agent in Mike inherits from `BaseAgent` (defined in `backend/src/lib/orchestrator/agents/baseAgent.ts`). This base class defines the standard interface that all agents must follow.

### What It Receives (Input)

Each agent gets an `AgentExecutionParams` object:

```typescript
interface AgentExecutionParams {
  task: Task;                    // The specific task to execute
  sharedContext: Record<string, unknown>;  // Results from previously completed tasks
  originalQuery: string;         // The user's original question
  userId: string;                // Who is asking
  chatId: string;                // Which conversation this is in
  projectId?: string | null;     // Project scope (if any)
  workspaceId?: string | null;   // Workspace scope (if any)
  assembledContext?: AssistantRuntimeContext;  // Full platform context
  contextSummary?: string;       // Summary of available context
  write: SseWrite;               // Function to stream events to the user
  apiKeys?: UserApiKeys;         // API keys for external services
}
```

The `Task` object tells the agent:
- What to do (`description`)
- What tools it can use (`toolsNeeded`)
- How urgent it is (`priority`: critical, high, medium, low)
- What other tasks it depends on (`dependencies`)
- How many tokens to budget (`estimatedTokens`)
- Where to store its output (`outputKey`)

### What It Returns (Output)

Each agent returns an `AgentResult`:

```typescript
interface AgentResult {
  taskId: string;           // Which task was completed
  agentType: AgentType;     // Which agent type handled it
  success: boolean;         // Did it succeed
  output: unknown;          // The actual result (text, JSON, etc.)
  outputContextKey: string; // Key for storing in shared context
  error?: string;           // Error message if failed
  toolsUsed: string[];      // Which tools were called
  tokenCount: number;       // How many tokens were consumed
  durationMs: number;       // How long it took
}
```

### Tools It Can Use

Each agent declares its allowed tools:

```typescript
abstract readonly allowedToolNames: string[];
```

The base agent filters tools based on:
- What the agent is allowed to use
- Whether the user has write permissions
- Whether a project/workspace is in scope
- Whether the tool requires external connections

### Memory It Has Access To

Every agent gets a `MemoryLayer` that provides:
- **Session memory**: Short-term data for the current conversation
- **Long-term memory**: User preferences and legal patterns stored across sessions
- **Conversation context**: Entities, document references, task history, active jurisdiction

### How It Tracks Progress

Agents emit Server-Sent Events (SSE) to keep the user informed:
- `agent_start`: Agent has begun working
- `agent_progress`: Agent is making progress
- `agent_tool_call`: Agent is calling a specific tool
- `agent_complete`: Agent has finished (success or failure)

---

## 4. Research Agent

### Responsibilities

The Research Agent is the platform's information gatherer. It:

- Finds information from uploaded documents
- Searches the web for legal references and case law
- Queries RAG (Retrieval-Augmented Generation) collections
- Extracts key facts from sources
- Compiles research findings with citations
- Distinguishes binding authority from persuasive authority
- Flags jurisdictional limits on cited authorities

### What It Receives

| Input | Description |
|-------|-------------|
| Research query | The specific legal question to investigate |
| Document context | Documents already available in the workspace |
| Search scope | Where to search (web, documents, clause library) |
| Constraints | Time limits, jurisdiction preferences, depth requirements |

### What It Produces

| Output | Description |
|--------|-------------|
| Findings list | Structured research results |
| Source citations | Specific references to cases, statutes, regulations |
| Confidence scores | How reliable each finding is |
| Gaps identified | What could not be found or verified |

### Tools Used

- `search_web` / `webSearch` - Internet search for legal references
- `httpGet` - Fetch specific URLs for legal content
- `search_sources` - Search the platform's source database
- `workspaceRagRetrieve` - Query workspace-specific RAG collections
- `searchDocuments` - Search across uploaded documents
- `read_document` - Read a specific document in full
- `fetch_documents` - Retrieve multiple documents
- `list_documents` - List available documents
- `search_clause_library` - Search the clause library for standard provisions

### Failure Modes

| Failure | What Happens | Fallback |
|---------|--------------|----------|
| No results found | Search returns empty | Try alternative search terms or different search scope |
| Ambiguous query | Too many results or unclear intent | Narrow the search or ask for clarification |
| Timeout | Search takes too long | Return partial results with a note |
| Source unavailable | Linked source cannot be reached | Flag the gap and continue with available sources |
| Network error | External search fails | Fall back to internal document search |

### Examples

**Example 1: Simple Research Request**

Input:
```
Query: "What are the grounds for termination under Indian employment law?"
Scope: Web + clause library
Jurisdiction: India
```

Output:
```json
{
  "findings": [
    {
      "topic": "Termination under Indian employment law",
      "summary": "Indian employment law permits termination primarily under: (1) Contract terms, (2) Industrial Disputes Act 1947 Section 25F for workmen, (3) Shops and Establishments Acts for state-specific rules, (4) Standing Orders under the Industrial Employment (Standing Orders) Act 1946.",
      "source": "Industrial Disputes Act 1947, Section 25F",
      "confidence": "high",
      "jurisdiction": "India"
    }
  ],
  "gaps": ["State-specific amendments not fully covered", "Recent case law not verified"],
  "sources_cited": 3
}
```

**Example 2: Document-Based Research**

Input:
```
Query: "Find all change of control clauses in the uploaded agreements"
Scope: Workspace documents
Document IDs: ["doc-1", "doc-2", "doc-3"]
```

Output:
```json
{
  "findings": [
    {
      "document": "MSA-v2.docx",
      "clause": "Section 8.3 - Change of Control",
      "text": "Either party may terminate this Agreement upon 30 days written notice if there is a change of control of the other party...",
      "analysis": "Standard change of control termination right with 30-day notice period. No consent requirement, only notification.",
      "confidence": "high"
    }
  ],
  "gaps": ["One document was unreadable"],
  "sources_cited": 1
}
```

**Example 3: Case Law Research**

Input:
```
Query: "Find recent cases on non-compete enforceability in technology sector"
Scope: Web + legal databases
Jurisdiction: India + US (California)
```

Output:
```json
{
  "findings": [
    {
      "case": "Niranjan Shankar Golikari v. Century Spinning",
      "citation": "AIR 1967 SC 1098",
      "holding": "Negative covenants during employment are enforceable if reasonable and necessary to protect trade secrets",
      "distinction": "Post-employment restraints remain void under Section 27 of the Indian Contract Act",
      "jurisdiction": "India",
      "confidence": "high"
    },
    {
      "case": "Edwards v. Arthur Andersen LLP",
      "citation": "44 Cal.4th 937 (2008)",
      "holding": "Non-competes are void in California except in connection with sale of a business",
      "jurisdiction": "California, US",
      "confidence": "high"
    }
  ],
  "gaps": ["No recent 2024-2025 cases found"],
  "sources_cited": 2
}
```

---

## 5. Document Agent

### Responsibilities

The Document Agent handles all document-related operations. It:

- Reads and parses documents (PDF, DOCX, TXT)
- Compares two or more documents
- Edits existing documents
- Generates new documents from templates
- Tracks changes between versions
- Extracts specific clauses or sections
- Validates document consistency

### What It Receives

| Input | Description |
|-------|-------------|
| Document references | Which documents to work with |
| Action type | What to do (read, compare, edit, generate) |
| Parameters | Action-specific options (template, format, sections) |

### What It Produces

| Output | Description |
|--------|-------------|
| Document content | Extracted text or structured content |
| Comparison results | Differences between documents |
| Generated documents | New documents created from templates |
| Change summaries | What was modified and why |

### Tools Used

- `list_documents` - List available documents
- `fetch_documents` - Retrieve multiple documents at once
- `read_document` - Read a specific document in full
- `find_in_document` - Search within a specific document
- `search_sources` - Search across source materials
- `workspaceRagRetrieve` - Query workspace RAG collections
- `searchDocuments` - Search across all documents
- `compare_documents` - Compare two documents side by side
- `extract_clauses` - Pull out specific clauses or sections
- `legal_draft_consistency_check` - Check for internal inconsistencies
- `suggest_edit` - Suggest changes to a document
- `extract_placeholders` - Find fill-in-the-blank fields
- `get_placeholder_values` - Read current placeholder values
- `edit_document` - Make changes to a document
- `insert_content` - Add content to a document
- `replicate_document` - Copy a document
- `generate_docx` - Create a new DOCX document

### Failure Modes

| Failure | What Happens | Fallback |
|---------|--------------|----------|
| Document not found | Requested document does not exist | Report the error and suggest alternatives |
| Format unsupported | File type cannot be parsed | Suggest converting to a supported format |
| Generation failure | Cannot create the document | Use a simpler template or generate as text |
| Edit failure | Changes could not be applied | Report what failed and suggest manual changes |
| Extraction failure | Text cannot be extracted from PDF | Request a text-searchable version |

### Examples

**Example 1: Document Comparison**

Input:
```
Action: compare
Documents: ["NDA-v1.docx", "NDA-v2.docx"]
Focus: "Changes to termination and non-compete clauses"
```

Output:
```json
{
  "comparison": {
    "document_a": "NDA-v1.docx",
    "document_b": "NDA-v2.docx",
    "differences": [
      {
        "section": "Termination",
        "v1_text": "Either party may terminate with 30 days notice",
        "v2_text": "Either party may terminate with 60 days notice",
        "change_type": "modified",
        "significance": "Notice period doubled from 30 to 60 days"
      },
      {
        "section": "Non-Compete",
        "v1_text": "Not present",
        "v2_text": "12-month non-compete within 50km radius",
        "change_type": "added",
        "significance": "New restrictive covenant added"
      }
    ],
    "summary": "2 material changes identified"
  }
}
```

**Example 2: Clause Extraction**

Input:
```
Action: extract_clauses
Document: "MSA-v3.docx"
Clauses: ["limitation of liability", "indemnification", "termination"]
```

Output:
```json
{
  "clauses": [
    {
      "name": "Limitation of Liability",
      "section": "Section 12",
      "text": "In no event shall either party's total aggregate liability exceed the fees paid during the 12 months preceding the claim...",
      "analysis": "Standard cap at 12-month fees. No carve-out for IP infringement or confidentiality breach."
    }
  ],
  "extraction_confidence": "high"
}
```

**Example 3: Document Generation**

Input:
```
Action: generate
Template: "NDA-mutual.docx"
Parameters: {
  "party_a": "Acme Corp",
  "party_b": "Beta Inc",
  "governing_law": "India",
  "term": "2 years",
  "purpose": "Business evaluation for potential acquisition"
}
```

Output:
```json
{
  "document_created": true,
  "file_id": "doc-new-001",
  "filename": "NDA_Acme_Beta_2024.docx",
  "sections_generated": 14,
  "placeholders_filled": 8,
  "warnings": ["Non-compete clause not included - standard template does not have one"]
}
```

---

## 6. Compliance Agent

### Responsibilities

The Compliance Agent checks documents and business practices against regulatory requirements. It:

- Checks documents against specific rules and regulations
- Identifies violations or gaps
- Assesses severity of each issue
- Suggests fixes or remediation steps
- Generates compliance reports
- Flags jurisdiction-specific requirements
- Monitors for high-velocity regulatory changes

### What It Receives

| Input | Description |
|-------|-------------|
| Document content | The document to check |
| Rule set | Which regulations or standards to check against |
| Jurisdiction | Which jurisdiction's rules apply |
| Compliance type | What kind of compliance (DPDP, GDPR, Companies Act, etc.) |

### What It Produces

| Output | Description |
|--------|-------------|
| Violations list | Specific issues found |
| Severity ratings | How serious each issue is (HIGH/MEDIUM/LOW) |
| Fix suggestions | How to address each issue |
| Compliance score | Overall assessment |

### Tools Used

- `search_web` / `webSearch` - Look up current regulations
- `httpGet` - Fetch specific regulatory texts
- `search_sources` - Search the platform's regulatory database
- `workspaceRagRetrieve` - Query workspace-specific compliance data
- `searchDocuments` - Search across documents for compliance evidence
- `list_documents` - List available documents
- `fetch_documents` - Retrieve documents for review
- `read_document` - Read a specific document
- `find_in_document` - Search within a document for specific content
- `compare_documents` - Compare documents for consistency
- `extract_clauses` - Extract specific clauses for review

### Failure Modes

| Failure | What Happens | Fallback |
|---------|--------------|----------|
| Rules not available | Cannot find the relevant regulation | Flag the gap and suggest manual research |
| Ambiguous violations | Issue is unclear or borderline | Flag as "needs review" with context |
| Timeout | Compliance check takes too long | Return partial results |
| Jurisdiction mismatch | Rules for wrong jurisdiction applied | Warn the user and request clarification |

### Examples

**Example 1: Data Protection Compliance**

Input:
```
Document: "Privacy-Policy-v2.pdf"
Regulation: "DPDP Act 2023 (India)"
Jurisdiction: "India"
```

Output:
```json
{
  "overall_status": "GAPS IDENTIFIED",
  "confidence": "high",
  "findings": [
    {
      "requirement": "Section 6 - Notice requirement",
      "status": "Partially met",
      "risk_level": "MEDIUM",
      "detail": "Privacy policy does not clearly state the purpose of data collection in plain language as required",
      "remediation": "Rewrite Section 3 to state purpose in clear, plain language per Section 6(1)"
    },
    {
      "requirement": "Section 8 - Data fiduciary obligations",
      "status": "Not met",
      "risk_level": "HIGH",
      "detail": "No mechanism for data principal to exercise right of withdrawal of consent",
      "remediation": "Add a consent withdrawal mechanism with clear instructions and timeline"
    }
  ],
  "compliance_score": "60%",
  "currentness_warning": "DPDP rules are high-velocity - verify against latest MCA notifications"
}
```

**Example 2: Contract Compliance Check**

Input:
```
Document: "Vendor-Agreement.docx"
Regulation: "Companies Act 2013 - Section 188 Related Party Transactions"
Jurisdiction: "India"
Context: "Company is entering into agreement with vendor where director has interest"
```

Output:
```json
{
  "overall_status": "NON-COMPLIANT",
  "confidence": "medium",
  "findings": [
    {
      "requirement": "Section 188 - Board resolution required",
      "status": "Not met",
      "risk_level": "HIGH",
      "detail": "No evidence of board resolution authorizing related party transaction",
      "remediation": "Obtain board resolution before executing agreement. Ensure interested director abstains from voting."
    },
    {
      "requirement": "Section 188 - Shareholder approval if exceeding thresholds",
      "status": "Cannot verify",
      "risk_level": "MEDIUM",
      "detail": "Transaction value not specified in document. If value exceeds prescribed thresholds, shareholder approval required.",
      "remediation": "Determine transaction value and check against Section 188(1) thresholds"
    }
  ]
}
```

**Example 3: Regulatory Gap Analysis**

Input:
```
Document: "Employment-Policy.docx"
Regulation: "Multiple: Shops Act, PF Act, ESI Act, Maternity Benefit Act"
Jurisdiction: "Maharashtra, India"
```

Output:
```json
{
  "overall_status": "GAPS IDENTIFIED",
  "findings": [
    {
      "requirement": "Maternity Benefit Act - 26 weeks leave",
      "status": "Partially met",
      "risk_level": "HIGH",
      "detail": "Policy provides 18 weeks. Act requires 26 weeks for first two children.",
      "remediation": "Amend policy to provide 26 weeks maternity leave as per Section 5(1)(a)"
    },
    {
      "requirement": "PF Act - Contribution on basic + DA",
      "status": "Met",
      "risk_level": "LOW",
      "detail": "Policy correctly references PF contribution on basic + DA",
      "remediation": "No action required"
    }
  ]
}
```

---

## 7. Drafting Agent

### Responsibilities

The Drafting Agent creates legal documents. It:

- Creates legal documents from scratch
- Fills templates with user-provided information
- Generates contracts, memos, and correspondence
- Uses sandbox tools for complex document generation
- Applies formatting and structure
- Ensures internal consistency
- Creates charts, tables, and visual elements when needed

### What It Receives

| Input | Description |
|-------|-------------|
| Document type | What kind of document to create |
| Parameters | Content to include (parties, terms, dates) |
| Template reference | Which template to use (if any) |
| Style preferences | Formatting requirements |

### What It Produces

| Output | Description |
|--------|-------------|
| Generated document | The created document (DOCX, PDF, etc.) |
| Draft content | Text content for review |
| Formatting applied | Structure and layout details |
| Quality score | Self-assessed quality rating |

### Tools Used

- `list_documents` - List available documents and templates
- `fetch_documents` - Retrieve reference documents
- `read_document` - Read reference materials
- `search_sources` - Search for legal references
- `search_templates` - Find appropriate templates
- `fill_and_create` - Fill a template and create a new document
- `generate_docx` - Create a new DOCX document
- `generate_xlsx` - Create an Excel spreadsheet
- `generate_pptx` - Create a PowerPoint presentation
- `sandboxExec` - Execute sandbox commands for complex generation
- `pythonInterpreter` - Run Python for calculations and data processing
- `sandboxReadFile` / `sandboxWriteFile` / `sandboxEditFile` - File operations in sandbox
- `sandboxListDir` - List files in sandbox
- `edit_document` - Edit an existing document
- `insert_content` - Add content to a document
- `replicate_document` - Copy a document as a starting point
- `extract_placeholders` - Find template fields
- `legal_draft_consistency_check` - Check for internal consistency
- `set_placeholder_value` / `fill_placeholders` - Fill template fields

### Special Rules

The Drafting Agent has a strict "ONE FINAL DOCUMENT ONLY" rule:
- Create one document per request unless the user explicitly asks for multiple
- If the user asks for "summary AND checklist", create ONE document with sections
- Document structure must include: Executive Summary, Research Findings, Requested Content, Risk Flags, Authorities Cited

### Failure Modes

| Failure | What Happens | Fallback |
|---------|--------------|----------|
| Template not found | Requested template does not exist | Use a generic template or draft from scratch |
| Generation failure | Cannot create the document | Generate as plain text instead |
| Quality too low | Self-check fails | Revise and regenerate |
| Sandbox unavailable | Cannot use sandbox tools | Fall back to basic document generation |

### Examples

**Example 1: NDA Drafting**

Input:
```
Document type: "Mutual NDA"
Parameters: {
  "party_a": "Acme Technologies Pvt Ltd",
  "party_b": "Beta Solutions Inc",
  "governing_law": "India",
  "jurisdiction": "Mumbai",
  "term": "3 years",
  "purpose": "Evaluation of potential joint venture"
}
```

Output:
```json
{
  "document_created": true,
  "file_id": "doc-nda-001",
  "filename": "Mutual_NDA_Acme_Beta_2024.docx",
  "sections": [
    "Definition of Confidential Information",
    "Obligations of Receiving Party",
    "Exclusions from Confidential Information",
    "Term and Termination",
    "Return of Materials",
    "Remedies",
    "Governing Law and Jurisdiction"
  ],
  "word_count": 2847,
  "placeholders_filled": 12,
  "notes": [
    "Non-compete clause not included per standard mutual NDA template",
    "Governing law set to India as requested"
  ]
}
```

**Example 2: Legal Memo**

Input:
```
Document type: "Legal Research Memo"
Topic: "Enforceability of non-compete agreements in India"
Jurisdiction: "India"
Research: [Agent outputs from Research Agent]
```

Output:
```json
{
  "document_created": true,
  "file_id": "doc-memo-001",
  "filename": "Memo_NonCompete_Enforceability_India.docx",
  "sections": [
    "Question Presented",
    "Brief Answer",
    "Applicable Legal Framework",
    "Analysis",
    "Conclusion",
    "Authorities Cited"
  ],
  "word_count": 3200,
  "confidence": "medium",
  "caveats": [
    "Analysis based on training knowledge - verify recent case law",
    "State-specific variations not fully covered"
  ]
}
```

**Example 3: Compliance Checklist Generation**

Input:
```
Document type: "DPDP Compliance Checklist"
Company: "TechStart Solutions Pvt Ltd"
Documents: ["Privacy Policy", "Vendor Agreements", "Employee Handbook"]
```

Output:
```json
{
  "document_created": true,
  "file_id": "doc-checklist-001",
  "filename": "DPDP_Compliance_Checklist_TechStart.docx",
  "structure": "Single document with sections for each requirement area",
  "sections": [
    "Consent Management",
    "Data Principal Rights",
    "Data Fiduciary Obligations",
    "Cross-Border Transfer",
    "Breach Notification",
    "Children's Data Protection"
  ],
  "total_items": 47,
  "status_summary": {
    "compliant": 28,
    "gaps": 12,
    "needs_review": 7
  }
}
```

---

## 8. Self-Critique Agent

### Responsibilities

The Self-Critique Agent is the quality reviewer. It:

- Reviews AI-generated outputs for errors
- Checks that cited authorities are real and accurate
- Verifies citations match what was actually found
- Assesses overall quality and completeness
- Suggests improvements
- Adds legal caveats where needed
- Ensures the response actually answers the original question

### What It Receives

| Input | Description |
|-------|-------------|
| AI output | The draft response to review |
| Original request | What the user actually asked for |
| Context | Available information and agent outputs |
| Quality criteria | What to check for |

### What It Produces

| Output | Description |
|--------|-------------|
| Quality score | Rating from 0 to 1 |
| Issues found | Specific problems identified |
| Improvement suggestions | How to fix issues |
| Pass/fail decision | Whether the output needs revision |
| Revised response | An improved version of the output |
| Legal caveats added | Disclaimers and warnings inserted |

### Tools Used

- `readDocument` - Verify cited document content
- `searchSources` - Check if cited authorities exist

### How It Works

The Self-Critique Agent receives a checklist of things to verify:

1. Does the response answer the actual question asked?
2. Are cited authorities verified against agent outputs?
3. Are jurisdiction caveats present where needed?
4. Is precise legal language used?
5. Is legal information distinguished from legal advice?
6. Are clear next steps included?

### Failure Modes

| Failure | What Happens | Fallback |
|---------|--------------|----------|
| Cannot verify | Citation cannot be checked | Flag as unverified |
| Timeout | Review takes too long | Skip review and return original |
| Parse error | Response cannot be parsed | Return original with no changes |

### Behavior

The Self-Critique Agent only runs when:
- The output validator found errors, OR
- The output validator found 2 or more warnings

It has an 8-second timeout. If it cannot complete in time, it returns the original response unchanged with `skipRevision: true`.

---

## 9. Output Validator

### Responsibilities

The Output Validator is the first line of quality defense. It:

- Validates the format of tool outputs
- Checks for completeness
- Verifies safety rules are followed
- Ensures compliance with legal AI standards
- Detects common failure patterns

### What It Receives

| Input | Description |
|-------|-------------|
| Output content | The response to validate |
| Validation rules | What to check against |
| Safety rules | Legal AI safety requirements |

### What It Produces

| Output | Description |
|--------|-------------|
| Validation result | Pass, warning, or error |
| Issues found | Specific problems with codes |
| Confidence score | How confident the validator is |

### What It Checks

The validator looks for these specific issues:

**Tool Output Validation:**
- `EMPTY_DOCUMENT_CONTENT` - Document read returned nothing
- `EMPTY_SEARCH_RESULTS` - Search returned no results
- `MISSING_GENERATED_DOCUMENT_ID` - Document generation did not return an ID
- `DOCUMENT_EDIT_FAILED` - Document edit reported failure

**Legal Response Validation:**
- `UNHELPFUL_REFUSAL` - Response refuses without providing useful information
- `DOCUMENT_CLAIM_WITHOUT_TOOL_OUTPUT` - Claims a document was created but no tool output confirms it
- `MISSING_JURISDICTION_CAVEAT` - Absolute legal statement without jurisdiction context
- `UNVERIFIED_CITATION` - Citation appears to come from training data, not agent outputs
- `TEMPLATE_PLACEHOLDER_LEAKED` - Response contains unfilled template fields
- `RESPONSE_TOO_SHORT` - Complex legal response is too brief

### Failure Modes

| Failure | What Happens | Fallback |
|---------|--------------|----------|
| Cannot parse output | Response format is unexpected | Flag as warning |
| Validation timeout | Check takes too long | Skip validation |

---

## 10. Response Generator

### Responsibilities

The Response Generator synthesizes all agent results into one coherent response. It:

- Combines outputs from multiple agents
- Formats for the user's needs
- Adds citations and references
- Creates artifacts (documents, spreadsheets)
- Ensures the response actually answers the original question
- Merges overlapping content into one deliverable
- Validates document type matches what was requested

### What It Receives

| Input | Description |
|-------|-------------|
| Agent results | All completed task outputs |
| Original request | What the user asked for |
| Memory context | User preferences and patterns |
| Requested document types | What the user wanted created |

### What It Produces

| Output | Description |
|--------|-------------|
| Final response | The synthesized answer |
| Citations | Source references preserved from agent outputs |
| Artifacts | Generated documents and files |
| Metadata | Response statistics and quality indicators |

### Key Rules

1. **Address the original request** - The response must directly deliver what the user asked for
2. **Consolidate into one response** - If multiple agents produced overlapping content, merge them
3. **Default to plain prose** - Use Markdown only when the content will become a document
4. **Quality of citations** - Include actual legal reasoning, not just case names
5. **Source preservation** - Preserve citation markers and evidence IDs from agent outputs
6. **Document type validation** - Verify created documents match what was requested

---

## 11. Coordinator

### Responsibilities

The Coordinator (implemented in `agentCoordinator.ts`) is the orchestration engine. It:

- Receives a plan with multiple tasks
- Identifies which agents are needed
- Manages task dependencies
- Executes tasks in parallel or sequentially
- Handles failures and retries
- Synthesizes results into a unified response

### How It Works

**Step 1: Receive Plan**

The coordinator receives a `TaskPlan` from the task planner. This plan contains:
- A list of tasks with descriptions
- Dependencies between tasks
- Which agent type handles each task
- Priority levels for each task
- Maximum retry counts

**Step 2: Build Execution Order**

The coordinator uses `getReadyTasks()` to find tasks whose dependencies are all completed. Tasks with no dependencies can run immediately.

**Step 3: Execute in Batches**

- If the execution strategy is `sequential`, only one task runs at a time
- If the execution strategy is `parallel`, all ready tasks run simultaneously
- Each task gets the current shared context (results from completed tasks)

**Step 4: Handle Results**

- Successful results are stored in shared context under their `outputKey`
- Failed tasks are retried up to their `maxRetries` count
- If a critical task fails, the entire orchestration stops
- Non-critical task failures are logged but do not stop the pipeline

**Step 5: Return Combined Result**

The coordinator returns a `CoordinatorResult` with:
- All completed task results
- All failed tasks with error messages
- The combined shared context
- Total token usage
- Total duration

### Dependency Management

Tasks declare dependencies with a type:
- `sequential` - Must wait for the dependency to complete
- `parallel` - Can run at the same time as the dependency
- `optional` - The dependency is nice-to-have but not required

The coordinator only runs tasks whose `sequential` dependencies are all completed.

### Failure Handling

| Scenario | Response |
|----------|----------|
| Task fails, retries left | Increment retry count and re-queue |
| Task fails, no retries left | Mark as failed, continue with other tasks |
| Critical task fails | Stop entire orchestration, report failure |
| All tasks fail | Return failure result with error details |
| Timeout (90 seconds per agent) | Agent is killed, task marked as failed |

---

## 12. Background Agent System

The background agent system handles long-running jobs that do not need real-time responses. It uses a job queue (BullMQ) to process tasks asynchronously.

### Agent Planner

**File:** `backend/src/agent/agentPlanner.ts` (247 lines)

The Agent Planner creates execution plans for background jobs. It:

- Receives a job with documents and instructions
- Detects which practice areas are involved
- Creates phased execution plans
- Identifies dependencies between phases
- Requests clarifications when needed

**Practice Areas:**
- Corporate
- Intellectual Property (IP)
- Commercial
- Employment
- Real Estate
- Data Protection
- Litigation
- Tax

**Plan Structure:**
```json
{
  "phases": [
    {
      "index": 0,
      "title": "Phase title",
      "practice_area": "corporate",
      "description": "What this phase does",
      "depends_on": []
    }
  ],
  "estimated_duration_mins": 120,
  "practice_areas": ["corporate", "ip"]
}
```

### Sub-Agent Executor

**File:** `backend/src/agent/subAgentExecutor.ts` (256 lines)

The Sub-Agent Executor runs individual tasks within a background job. It:

- Selects the appropriate skill for the task
- Builds a system prompt with skill instructions
- Calls the LLM to analyze documents
- Validates the JSON output
- Returns structured findings

**Output Structure:**
```json
{
  "findings": [
    {
      "severity": "red|amber|green|incomplete",
      "issue": "Description of the issue",
      "recommendation": "How to fix it",
      "vdr_reference": "Document reference",
      "document_name": "Source document"
    }
  ],
  "documents_reviewed": ["List of documents"],
  "outstanding_rfi": ["Questions that need answers"]
}
```

### Preliminary Assessor

**File:** `backend/src/agent/preliminaryAssessor.ts` (213 lines)

The Preliminary Assessor evaluates Virtual Data Room (VDR) structure for due diligence. It:

- Analyzes file metadata (names, paths, sizes)
- Infers folder structure from file paths
- Assigns files to practice areas
- Identifies missing expected documents
- Produces a structured assessment

**Assessment Structure:**
```json
{
  "file_count": 150,
  "folder_structure": [...],
  "practice_area_assignments": {
    "corporate": ["file-1", "file-2"],
    "ip": ["file-3"]
  },
  "missing_expected_documents": ["Certificate of Incorporation", "MOA"],
  "assessment_notes": "Key observations about the VDR"
}
```

### Report Assembler

**File:** `backend/src/agent/reportAssembler.ts` (297 lines)

The Report Assembler creates the final deliverable document. It:

- Takes all sub-task results
- Generates a DOCX report (using templates if available)
- Creates finding tables with risk ratings
- Uploads the report to storage
- Creates a file record in the database
- Returns an artifact reference

**Report Types:**
- Due Diligence Report
- Pleadings Matrix

### Artifact Reducer

**File:** `backend/src/agent/artifactReducer.ts` (196 lines)

The Artifact Reducer synthesizes all sub-task results into a final report structure. It:

- Groups findings by severity (red, amber, green, incomplete)
- Identifies cross-cutting issues across practice areas
- Generates an executive summary
- Flags items requiring partner attention
- Produces the final `ReportData` structure

### Topological Sort

**File:** `backend/src/agent/topologicalSort.ts` (49 lines)

The Topological Sort determines the execution order for tasks with dependencies. It:

- Takes a list of phases with dependency references
- Validates no circular dependencies exist
- Groups phases into batches that can run in parallel
- Returns batches in correct execution order

### Agent Worker

**File:** `backend/src/queue/agentWorker.ts` (534 lines)

The Agent Worker is the BullMQ worker that processes background jobs. It:

- Listens for jobs on the `agent-jobs` queue
- Loads the job from the database
- Routes to the appropriate processor (due diligence, pleadings, etc.)
- Manages the full lifecycle: planning, clarification, execution, assembly
- Handles clarification requests with 24-hour timeout
- Emits events for real-time status updates
- Generates and uploads final artifacts

**Job Lifecycle:**
1. `queued` - Job submitted
2. `planning` - Creating execution plan
3. `running` - Executing sub-tasks
4. `paused_for_input` - Waiting for clarification
5. `completed` - Done successfully
6. `failed` - Error occurred

---

## 13. Agent Communication

### Shared Context

Agents communicate primarily through shared context. When one agent completes a task, its output is stored in a shared context map keyed by `outputContextKey`. Subsequent agents receive this map and can read outputs from earlier tasks.

```typescript
// In agentCoordinator.ts
const sharedContext = new Map<string, unknown>();

// After a task completes:
sharedContext.set(result.outputContextKey, result.output);

// When executing the next task:
sharedContext: Object.fromEntries(sharedContext)
```

### Message Passing

Agents do not communicate directly with each other. Instead, they communicate through the coordinator:
1. Agent A completes and writes to shared context
2. Coordinator reads Agent A's output
3. Coordinator passes it to Agent B as part of the shared context
4. Agent B reads and uses Agent A's output

### Event Bus

The system uses Server-Sent Events (SSE) to stream progress to the user:
- `orchestration_thinking` - Planning in progress
- `plan_ready` - Plan created
- `orchestration_start` - Execution beginning
- `agent_start` - Individual agent starting
- `agent_progress` - Agent making progress
- `agent_tool_call` - Agent calling a tool
- `agent_complete` - Agent finished
- `orchestration_complete` - All agents finished
- `content_delta` - Response text streaming
- `citations` - Source references

### Memory Sharing

The `MemoryLayer` provides persistent memory across conversations:
- **Session memory**: Data for the current chat (cleared when chat ends)
- **Long-term memory**: User preferences and legal patterns (persists across sessions)
- **Conversation context**: Entities, document references, task history

---

## 14. Agent Error Handling

### Retry with Backoff

Tasks can be retried up to `maxRetries` (typically 2). The coordinator increments `retryCount` and re-queues the task.

```typescript
const nextRetryCount = task.retryCount + 1;
if (nextRetryCount <= task.maxRetries) {
  task.retryCount = nextRetryCount;
  continue; // Re-queue for retry
}
```

### Alternative Agent

If the primary agent fails, the system can fall back to the `genericAgent` which has no specialized tools but can handle general requests.

### Graceful Degradation

When errors occur:
1. The specific task is marked as failed
2. Non-critical failures are logged but do not stop the pipeline
3. Critical task failures stop the entire orchestration
4. The system falls back to the legacy single-agent path if orchestration fails

### User Notification

Users are notified through:
- SSE events showing agent status
- Error messages in the response
- Fallback notices when orchestration falls back to legacy mode

### Timeout Management

Each layer has its own timeout:
- **Orchestration timeout**: 180 seconds total
- **Planning timeout**: 45 seconds
- **Agent execution timeout**: 120 seconds (90 seconds per individual agent)
- **Synthesis timeout**: 45 seconds
- **Self-critique timeout**: 8 seconds
- **Clarification timeout**: 24 hours (for background jobs)

---

## 15. Agent Performance

### Parallel Execution

When the execution strategy is `parallel`, all ready tasks run simultaneously:

```typescript
const batch = params.plan.executionStrategy === "sequential" 
  ? readyTasks.slice(0, 1) 
  : readyTasks;

const results = await Promise.allSettled(
  batch.map((task) => this.executeTask(task, params))
);
```

### Caching

The agent tool executor maintains caches:
- **findCache**: Caches search results to avoid duplicate searches
- **readDocumentContentByDoc**: Caches document reads
- **executedToolCache**: Caches tool execution results for idempotency

### Timeout Management

Timeouts are enforced at multiple levels:
- Individual agent timeout (90 seconds)
- Planning phase timeout (45 seconds)
- Synthesis phase timeout (45 seconds)
- Overall orchestration timeout (180 seconds)

### Resource Limits

- Maximum 6 tasks per plan
- Maximum 3 iterations per agent tool call
- Maximum 3 concurrent background jobs
- Token budgets per agent: research=1500, drafting=3000, compliance=2000

---

## 16. Agent Security

### Tool Permissions

Each agent has a whitelist of allowed tools. The base agent filters tools based on:
- Agent's declared `allowedToolNames`
- User's write permissions
- Project/workspace scope availability

```typescript
protected getAllowedToolNames(params: AgentExecutionParams): string[] {
  return this.allowedToolNames.filter((toolName) => {
    if (!canWrite && MUTATING_TOOL_NAMES.has(toolName)) return false;
    if (!params.projectId && toolName.startsWith("project")) return false;
    if (!params.workspaceId && toolName.startsWith("workspace")) return false;
    return true;
  });
}
```

### Mutating Tool Protection

Tools that modify data are tracked separately:
```typescript
const MUTATING_TOOL_NAMES = new Set([
  "create_project", "create_workspace", "generate_docx",
  "edit_document", "fill_and_create", "email", "sandboxExec",
  // ... more tools
]);
```

### Output Validation

Every agent output is validated by the `LegalOutputValidator` which checks for:
- Empty results
- Missing document IDs
- Unverified citations
- Template placeholder leaks
- Insufficient response length
- Claims without tool evidence

### Audit Logging

The system logs:
- Every orchestration run with timing and success metrics
- Every agent execution with tools used and duration
- Every tool call with capability tracking
- Every background job lifecycle event

### Rate Limiting

Orchestration requests are rate-limited per user:
```typescript
const rateLimit = await checkOrchestrationRateLimit(params.userId);
if (!rateLimit.allowed) {
  return this.legacy.handle(params);
}
```

---

## 17. Complete Agent Interaction Diagram

```
USER REQUEST
    |
    v
+------------------+
| Intent           |
| Classifier       |
| (classifyIntent) |
+------------------+
    |
    | (simple) --> Legacy Single Agent Path
    |
    | (complex/multi_agent)
    v
+------------------+
| Task Planner     |
| (createTaskPlan) |
+------------------+
    |
    v
+------------------+
| Agent            |     +-------------------+
| Coordinator      |---->| Shared Context    |
| (agentCoordinator|     | (Map of results)  |
+------------------+     +-------------------+
    |
    |--- Task 1: Research Agent
    |       |--- search_web
    |       |--- search_sources
    |       |--- read_document
    |       |--- search_clause_library
    |       |--> Output stored in shared context
    |
    |--- Task 2: Document Agent (waits for Task 1)
    |       |--- read_document
    |       |--- compare_documents
    |       |--- extract_clauses
    |       |--> Output stored in shared context
    |
    |--- Task 3: Compliance Agent (waits for Task 2)
    |       |--- search_web
    |       |--- read_document
    |       |--- find_in_document
    |       |--> Output stored in shared context
    |
    |--- Task 4: Drafting Agent (waits for Task 1)
    |       |--- generate_docx
    |       |--- fill_placeholders
    |       |--- sandboxExec
    |       |--> Output stored in shared context
    |
    v
+------------------+
| Response         |
| Generator        |
| (generateFinal   |
|  Response)       |
+------------------+
    |
    v
+------------------+
| Output           |
| Validator        |
| (legalOutput     |
|  Validator)      |
+------------------+
    |
    | (errors found)
    v
+------------------+
| Self-Critique    |
| Agent            |
| (selfCritique    |
|  Agent)          |
+------------------+
    |
    v
FINAL RESPONSE --> USER
```

---

## 18. Agent Reference Table

| Agent | Purpose | Tools | Inputs | Outputs | Failure Modes |
|-------|---------|-------|--------|---------|---------------|
| **Research Agent** | Find legal information, cases, statutes | search_web, webSearch, httpGet, search_sources, workspaceRagRetrieve, searchDocuments, read_document, fetch_documents, list_documents, search_clause_library | Research query, document context, search scope, constraints | Findings list, source citations, confidence scores, gaps identified | No results, ambiguous query, timeout, network error |
| **Document Agent** | Read, compare, edit, generate documents | list_documents, fetch_documents, read_document, find_in_document, search_sources, workspaceRagRetrieve, searchDocuments, compare_documents, extract_clauses, legal_draft_consistency_check, suggest_edit, extract_placeholders, get_placeholder_values, edit_document, insert_content, replicate_document, generate_docx | Document references, action type, parameters | Document content, comparison results, generated documents, change summaries | Document not found, format unsupported, generation failure, extraction failure |
| **Compliance Agent** | Check documents against regulations | search_web, webSearch, httpGet, search_sources, workspaceRagRetrieve, searchDocuments, list_documents, fetch_documents, read_document, find_in_document, compare_documents, extract_clauses | Document content, rule set, jurisdiction, compliance type | Violations list, severity ratings, fix suggestions, compliance score | Rules not available, ambiguous violations, timeout, jurisdiction mismatch |
| **Drafting Agent** | Create legal documents from templates or scratch | list_documents, fetch_documents, read_document, search_sources, search_templates, fill_and_create, generate_docx, generate_xlsx, generate_pptx, sandboxExec, pythonInterpreter, sandboxReadFile, sandboxWriteFile, sandboxEditFile, sandboxListDir, edit_document, insert_content, replicate_document, extract_placeholders, legal_draft_consistency_check, set_placeholder_value, fill_placeholders | Document type, parameters, template reference, style preferences | Generated document, draft content, formatting applied, quality score | Template not found, generation failure, quality too low, sandbox unavailable |
| **Self-Critique Agent** | Review and improve AI outputs | readDocument (for verification), searchSources (for citation check) | AI output, original request, context, quality criteria | Quality score, issues found, improvement suggestions, pass/fail, revised response, legal caveats | Cannot verify, timeout (8s), parse error |
| **Output Validator** | Validate format, completeness, safety | N/A (static analysis) | Output content, validation rules, safety rules | Validation result (pass/warning/error), issues with codes | Cannot parse, validation timeout |
| **Response Generator** | Synthesize agent results into final response | completeText (LLM call) | Agent results, original request, memory context, requested document types | Final response, citations, artifacts, metadata | LLM failure, timeout (45s) |
| **Generic Agent** | Handle simple single-step requests | None (no specialized tools) | General query | General response | LLM failure, timeout |
| **Agent Coordinator** | Orchestrate multiple agents, manage dependencies | N/A (manages other agents) | Task plan, shared context, user parameters | CoordinatorResult with completed/failed tasks | Agent timeout, critical task failure, all tasks fail |
| **Agent Planner** | Plan background job execution phases | completeText (LLM call) | AgentJob with documents and instructions | AgentPlan with phases, practice areas, dependencies | Parse error, validation error, timeout |
| **Sub-Agent Executor** | Execute individual background tasks | completeText (LLM call) | AgentSubTask with prompt and practice area | SubTaskResult with findings, documents reviewed, RFIs | LLM failure, parse error, validation error |
| **Preliminary Assessor** | Assess VDR structure for due diligence | completeText (LLM call) | AgentJob with file metadata | VDRAssessment with folder structure, practice area assignments, missing documents | LLM failure, parse error, validation error |
| **Report Assembler** | Generate final DOCX reports | docx generation, template filling, storage upload | ReportData or PleadingsMatrixData | AgentArtifact with file reference | Template not found, upload failure, generation error |
| **Artifact Reducer** | Synthesize sub-task results into report data | completeText (LLM call) | AgentJob and SubTaskResult[] | ReportData with sections, findings, executive summary | LLM failure, parse error, validation error |
| **Topological Sort** | Order tasks by dependencies | N/A (pure algorithm) | AgentPlanPhase[] | Batched execution order | Circular dependency, missing dependency, duplicate index |

---

## Appendix A: Key Source Files

| File | Lines | Purpose |
|------|-------|---------|
| `backend/src/lib/orchestrator/index.ts` | 488 | Main orchestrator pipeline |
| `backend/src/lib/orchestrator/types.ts` | 117 | Agent type definitions |
| `backend/src/lib/orchestrator/agents/baseAgent.ts` | 252 | Base agent class |
| `backend/src/lib/orchestrator/agents/researchAgent.ts` | 32 | Research agent |
| `backend/src/lib/orchestrator/agents/documentAgent.ts` | 30 | Document agent |
| `backend/src/lib/orchestrator/agents/complianceAgent.ts` | 25 | Compliance agent |
| `backend/src/lib/orchestrator/agents/draftingAgent.ts` | 67 | Drafting agent |
| `backend/src/lib/orchestrator/agents/agentCoordinator.ts` | 187 | Agent coordination |
| `backend/src/lib/orchestrator/agents/agentToolExecutor.ts` | 245 | Tool execution |
| `backend/src/lib/orchestrator/critique/selfCritiqueAgent.ts` | 72 | Self-critique |
| `backend/src/lib/orchestrator/validator/outputValidator.ts` | 99 | Output validation |
| `backend/src/lib/orchestrator/response/responseGenerator.ts` | 62 | Response synthesis |
| `backend/src/lib/orchestrator/memory/memoryLayer.ts` | 250 | Memory management |
| `backend/src/lib/orchestrator/planner/taskPlanner.ts` | 218 | Task planning |
| `backend/src/lib/orchestrator/planner/executionPlan.ts` | 50 | Execution plans |
| `backend/src/agent/agentPlanner.ts` | 247 | Background job planning |
| `backend/src/agent/subAgentExecutor.ts` | 256 | Sub-task execution |
| `backend/src/agent/preliminaryAssessor.ts` | 213 | VDR assessment |
| `backend/src/agent/reportAssembler.ts` | 297 | Report generation |
| `backend/src/agent/artifactReducer.ts` | 196 | Artifact reduction |
| `backend/src/agent/topologicalSort.ts` | 49 | Task ordering |
| `backend/src/queue/agentWorker.ts` | 534 | BullMQ worker |
| `backend/src/lib/agentic/prompts.ts` | 972 | Legal Safety Contract |
| `backend/src/lib/agentic/humanLoop.ts` | 111 | Human-in-the-loop |

## Appendix B: Glossary

| Term | Definition |
|------|-----------|
| **Agent** | A specialized AI component that handles one type of legal work |
| **Coordinator** | The component that manages multiple agents and their dependencies |
| **Shared Context** | A map of results passed between agents during execution |
| **Task** | A single unit of work assigned to an agent |
| **Plan** | An ordered list of tasks with dependencies |
| **Legal Domain** | The area of law a request relates to (contract_drafting, compliance_check, etc.) |
| **SSE** | Server-Sent Events - used to stream progress to the user |
| **VDR** | Virtual Data Room - a repository of documents for due diligence |
| **RAG** | Retrieval-Augmented Generation - searching documents to ground AI responses |
| **Legal Safety Contract** | The set of rules every agent must follow to ensure safe legal AI output |
| **BullMQ** | The job queue library used for background agent processing |
| **Practice Area** | A legal specialty (corporate, IP, commercial, employment, etc.) |
| **Finding** | A specific issue identified by an agent during analysis |
| **Artifact** | A generated document or file produced by an agent |
| **Confidence Score** | A rating of how reliable an agent's output is |
| **Idempotency** | The property that running the same operation twice produces the same result |
