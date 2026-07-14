# Agent Decision Tree

## 1. What This Document Is

This document is the complete decision tree showing how every agent in the Mike legal AI platform thinks and acts. It maps out every decision point, from the moment a user types a message to the final response they receive. Think of it as a flowchart for the system's brain—showing all the paths it can take, all the choices it makes, and all the backup plans it has when things go wrong.

The system is built like a team of specialized lawyers, each with their own expertise. Some are researchers, some are document reviewers, some are compliance experts, and some are drafters. There's also a coordinator who manages the team when a task needs multiple specialists. This document shows exactly how the system decides which lawyer to call, what they should do, and how their work gets combined into a final answer.

---

## 2. The Complete User Request Flow

This is the high-level flow every user request follows:

```
User Request
    ↓
Intent Classifier
    ↓ (simple / complex / multi_agent)
    ├→ Simple: Direct LLM response
    │       ↓
    │   Return answer to user
    │
    ├→ Complex: Single agent pipeline
    │       ↓
    │   Select one specialist agent
    │       ↓
    │   Agent processes request
    │       ↓
    │   Self-critique review
    │       ↓
    │   Output validation
    │       ↓
    │   Return answer to user
    │
    └→ Multi-Agent: Coordinator pipeline
            ↓
        Coordinator selects multiple agents
            ↓
        Build dependency graph
            ↓
        Topological sort (order by dependencies)
            ↓
        Execute agents in order
        ├→ Parallel where possible
        └→ Sequential where dependent
            ↓
        Collect all results
            ↓
        Synthesize combined response
            ↓
        Self-critique review
            ↓
        Output validation
            ↓
        Return answer to user
```

**Key decision point**: The system only uses the multi-agent pipeline when the classification confidence is above 0.65 AND the request is marked as complex or multi-agent. Otherwise, it falls back to the simpler single-agent or direct LLM path.

---

## 3. Intent Classification Decision Tree

The intent classifier is the system's first brain. It reads the user's message and decides what kind of request it is.

```
User Message
    ↓
Is it a greeting?
    ├→ Yes: Friendly response
    │       Example: "Hello!", "Hi there", "Good morning"
    │       Action: Return a friendly greeting without using any agents
    │
    └→ No: Continue
        ↓
    Is it a question about the platform?
        ├→ Yes: Help response
        │       Example: "How do I upload a document?", "What can you do?"
        │       Action: Return platform help information
        │
        └→ No: Continue
            ↓
        Does it reference documents?
            ├→ Yes: Document context loaded
            │       Example: "Review this contract", "Summarize clause 5"
            │       Action: Load document context, then continue classification
            │
            └→ No: Continue
                ↓
            Is it a simple request?
                ├→ Yes: Direct tool call
                │       Example: "What's the capital of France?", "Convert 5 USD to EUR"
                │       Action: Use simple tool directly, no agent needed
                │
                └→ No: Continue
                    ↓
                Is it a complex multi-step request?
                    ├→ Yes: Task planner
                    │       Example: "Review my contract and suggest improvements"
                    │       Action: Break into sub-tasks, assign to multiple agents
                    │
                    └→ No: Single agent
                            Example: "Summarize this document"
                            Action: Assign to one specialist agent
```

**Legal domain detection** happens during classification. The system identifies:
- `contract_drafting` - Creating or reviewing contracts
- `compliance_check` - Checking rules and regulations
- `legal_research` - Finding legal information
- `document_review` - Analyzing existing documents
- `template_fill` - Filling in legal templates
- `general_legal` - General legal questions
- `non_legal` - Not legal-related at all

---

## 4. Agent Selection Decision Tree

Once the system knows what kind of request it is, it picks the right agent(s).

```
Classified Intent
    ↓
Domain Detection
    ↓
What expertise is needed?
    │
    ├→ Research needed?
    │   (Finding information, reading documents, web search)
    │   → Research Agent
    │
    ├→ Document work?
    │   (Reading, writing, comparing, editing documents)
    │   → Document Agent
    │
    ├→ Compliance check?
    │   (Checking rules, regulations, policies)
    │   → Compliance Agent
    │
    ├→ Content creation?
    │   (Drafting contracts, memos, briefs)
    │   → Drafting Agent
    │
    └→ Multiple needs?
        (e.g., "Research this topic, then draft a memo about it")
        → Coordinator
            ↓
        Identify all needed agents
            ↓
        Build task dependency graph
            ↓
        Execute in proper order
```

**Agent specialization breakdown:**

| Agent Type | What It Does | Tools It Uses |
|------------|--------------|---------------|
| Research Agent | Finds information from documents, web, databases | RAG search, web search, legal databases |
| Document Agent | Reads, writes, compares, edits documents | Document parsing, comparison, editing |
| Compliance Agent | Checks rules, regulations, policies | Rule lookup, clause analysis, policy check |
| Drafting Agent | Creates legal documents, contracts, memos | Template filling, content generation |
| Coordinator | Orchestrates other agents | Task planning, dependency management |

---

## 5. Research Agent Decision Tree

The Research Agent is the system's detective. It finds information from multiple sources.

```
Research Request
    ↓
What sources are needed?
    │
    ├→ Documents in project?
    │   (User uploaded files, project documents)
    │   → RAG search
    │       ↓
    │   Search vector database for relevant chunks
    │       ↓
    │   Results found?
    │       ├→ Yes: Return findings
    │       └→ No: Try alternative search terms
    │
    ├→ External documents?
    │   (Files user wants to upload)
    │   → File upload + OCR
    │       ↓
    │   Process uploaded files
    │       ↓
    │   Extract text and index
    │       ↓
    │   Search extracted content
    │
    ├→ Web information?
    │   (Current events, general knowledge)
    │   → Web search
    │       ↓
    │   Search the web
    │       ↓
    │   Filter and summarize results
    │
    ├→ Legal databases?
    │   (Case law, statutes, regulations)
    │   → Legal research
    │       ↓
    │   Search legal databases
    │       ↓
    │   Extract relevant cases/statutes
    │
    └→ All of the above?
        (Comprehensive research request)
        → Multi-source research
            ↓
        Execute all searches in parallel
            ↓
        Combine and deduplicate results
            ↓
        Rank by relevance
    ↓
Results found?
    ├→ Yes: Synthesize findings
    │       ↓
    │   Create coherent summary
    │       ↓
    │   Include citations
    │       ↓
    │   Return to user
    │
    └→ No: Try alternative sources
            ↓
        Use different search terms
            ↓
        Try different databases
            ↓
        Still no results?
            ├→ Yes: Report gap to user
            │       "I couldn't find information about X"
            └→ No: Continue with what was found
```

---

## 6. Document Agent Decision Tree

The Document Agent is the system's reader and writer. It handles all document operations.

```
Document Request
    ↓
What action is needed?
    │
    ├→ Read?
    │   (Extract content from a document)
    │   → Extract content
    │       ↓
    │   Parse document structure
    │       ↓
    │   Extract text, tables, images
    │       ↓
    │   Return structured content
    │
    ├→ Compare?
    │   (Side-by-side analysis of documents)
    │   → Side-by-side analysis
    │       ↓
    │   Align document sections
    │       ↓
    │   Identify differences
    │       ↓
    │   Generate comparison report
    │
    ├→ Edit?
    │   (Make changes to a document)
    │   → Track changes
    │       ↓
    │   Identify edit locations
    │       ↓
    │   Apply changes with tracking
    │       ↓
    │   Generate redlined version
    │
    ├→ Generate?
    │   (Create new document from template)
    │   → Template + fill
    │       ↓
    │   Select appropriate template
    │       ↓
    │   Fill in required fields
    │       ↓
    │   Generate document
    │
    ├→ Review?
    │   (Analyze document for issues)
    │   → Clause analysis
    │       ↓
    │   Identify key clauses
    │       ↓
    │   Check for problematic language
    │       ↓
    │   Generate review summary
    │
    └→ Export?
        (Convert to different format)
        → Format conversion
            ↓
        Identify target format
            ↓
        Convert content
            ↓
        Preserve formatting
```

---

## 7. Compliance Agent Decision Tree

The Compliance Agent is the system's rule checker. It ensures everything follows the rules.

```
Compliance Request
    ↓
What rules apply?
    │
    ├→ Regulatory?
    │   (Government laws and regulations)
    │   → Rule lookup
    │       ↓
    │   Search regulatory database
    │       ↓
    │   Find applicable regulations
    │       ↓
    │   Check compliance
    │
    ├→ Contractual?
    │   (Terms within contracts)
    │   → Clause analysis
    │       ↓
    │   Extract contract clauses
    │       ↓
    │   Check against requirements
    │       ↓
    │   Identify violations
    │
    ├→ Internal policy?
    │   (Company rules and guidelines)
    │   → Policy check
    │       ↓
    │   Load company policies
    │       ↓
    │   Compare against document
    │       ↓
    │   Flag non-compliance
    │
    └→ Multiple rule types?
        (Need to check several sources)
        → Parallel checks
            ↓
        Run all checks simultaneously
            ↓
        Combine results
            ↓
        Prioritize by severity
    ↓
Violations found?
    ├→ Yes: Severity assessment
    │       ↓
    │   What level is the violation?
    │       │
    │       ├→ Critical: Immediate alert
    │       │   "This violates federal law and could result in penalties"
    │       │   Action: Stop processing, alert user immediately
    │       │
    │       ├→ High: Warning + fix suggestion
    │       │   "This clause may be unenforceable"
    │       │   Action: Warn user, suggest specific fix
    │       │
    │       ├→ Medium: Note + recommendation
    │       │   "This language could be clearer"
    │       │   Action: Note issue, recommend improvement
    │       │
    │       └→ Low: Informational only
    │           "This is non-standard but acceptable"
    │           Action: Mention for awareness
    │
    └→ No: Pass confirmation
            "This document complies with all checked rules"
            Action: Return positive confirmation
```

---

## 8. Drafting Agent Decision Tree

The Drafting Agent is the system's writer. It creates new legal documents.

```
Drafting Request
    ↓
What type of document?
    │
    ├→ Contract?
    │   → Contract template
    │       ↓
    │   Select contract type (NDA, Service Agreement, etc.)
    │       ↓
    │   Load template with standard clauses
    │
    ├→ Memo?
    │   → Memo template
    │       ↓
    │   Select memo format (internal, external, legal)
    │       ↓
    │   Load memo structure
    │
    ├→ Brief?
    │   → Brief template
    │       ↓
    │   Select brief type (court brief, research brief)
    │       ↓
    │   Load brief format
    │
    ├→ Letter?
    │   → Letter template
    │       ↓
    │   Select letter type (demand, cease & desist, etc.)
    │       ↓
    │   Load letter format
    │
    └→ Custom?
        → Blank template
            ↓
        Start from scratch
            ↓
        Define document structure
    ↓
Template available?
    ├→ Yes: Fill template
    │       ↓
    │   Insert user-provided information
    │       ↓
    │   Apply legal language
    │       ↓
    │   Ensure consistency
    │
    └→ No: Generate from scratch
            ↓
        Use LLM to create content
            ↓
        Apply legal formatting
            ↓
        Structure appropriately
    ↓
User review required?
    ├→ Yes: Show preview
    │       ↓
    │   Display draft to user
    │       ↓
    │   Wait for feedback
    │       ↓
    │   Make revisions if needed
    │
    └→ No: Auto-generate
            ↓
        Create final version
            ↓
        Ready for use
```

---

## 9. Coordinator Decision Tree

The Coordinator is the team manager. It handles requests that need multiple agents working together.

```
Multi-Agent Request
    ↓
Identify all needed agents
    ↓
Example: "Research this case law, then draft a memo about it"
    ↓
Needed agents: Research Agent → Drafting Agent
    ↓
Build dependency graph
    ↓
What depends on what?
    ├→ Research must happen first (no dependencies)
    └→ Drafting depends on Research output
    ↓
Topological sort
    ↓
Execution order:
    1. Research Agent (no dependencies, can start immediately)
    2. Drafting Agent (depends on Research output)
    ↓
Execute in order
    ↓
Batch 1: Research Agent
    ├→ Execute task
    ├→ Collect result
    └→ Store in shared context
    ↓
Batch 2: Drafting Agent
    ├→ Get Research output from shared context
    ├→ Execute task with research findings
    └→ Collect result
    ↓
Collect all results
    ↓
Synthesize combined response
    ├→ Combine outputs from all agents
    ├→ Ensure consistency
    └→ Create coherent final answer
    ↓
Self-critique
    ├→ Quality score
    ├→ Legal accuracy check
    ├→ Citation verification
    └─── Consistency check
        ↓
    Score acceptable?
        ├→ Yes: Proceed
        └→ No: Regenerate
            ↓
        Max regenerations?
            ├→ Yes: Use best attempt
            └─── No: Try again
    ↓
Validate output
    ├→ Format correct?
    ├→ Complete?
    ├→ Safe?
    └─── Useful?
        ↓
    All pass?
        ├─→ Yes: Return to user
        └→ No: Fix and retry
```

---

## 10. Tool Selection Decision Tree

Each agent uses tools to do its work. Here's how the system picks the right tool.

```
Tool Needed
    ↓
Categorize request
    │
    ├→ Search?
    │   (Finding information)
    │   → Search tools
    │       ├→ Document search (RAG)
    │       ├→ Web search
    │       ├→ Legal database search
    │       └→ Internal knowledge search
    │
    ├→ Document?
    │   (Working with files)
    │   → Document tools
    │       ├→ Read document
    │       ├→ Write document
    │       ├→ Compare documents
    │       ├→ Extract from document
    │       └→ Convert document format
    │
    ├→ Legal?
    │   (Legal-specific operations)
    │   → Legal tools
    │       ├→ Case law research
    │       ├→ Statute lookup
    │       ├→ Regulation check
    │       ├→ Citation verification
    │       └─→ Precedent analysis
    │
    ├→ Generate?
    │   (Creating new content)
    │   → Generation tools
    │       ├→ Text generation
    │       ├→ Document generation
    │       ├→ Summary creation
    │       └→ Report building
    │
    ├→ Email?
    │   (Communication)
    │   → Email tools
    │       ├→ Send email
    │       ├→ Read email
    │       └→ Draft email
    │
    ├→ Workspace?
    │   (Project management)
    │   → Workspace tools
    │       ├→ List files
    │       ├→ Upload files
    │       ├─→ Organize files
    │       └→ Manage permissions
    │
    └→ Other?
        (Miscellaneous operations)
        → Utility tools
            ├→ Math calculations
            ├→ Date operations
            ├─→ Text formatting
            └→ Data conversion
    ↓
Specific tool selection
    ├→ Check risk level
    │       Is this tool safe to use automatically?
    │
    ├→ Check dependencies
    │       Does this tool need data from another tool?
    │
    ├→ Check permissions
    │       Is the user allowed to use this tool?
    │
    └→ Execute
            Run the selected tool
```

---

## 11. Error Recovery Decision Tree

Things don't always go as planned. Here's how the system handles errors.

```
Error Occurs
    ↓
What type of error?
    │
    ├→ LLM timeout?
    │   (Language model took too long)
    │   → Retry with backoff
    │       ↓
    │   Wait 1 second, retry
    │       ↓
    │   Wait 2 seconds, retry
    │       ↓
    │   Wait 4 seconds, retry
    │       ↓
    │   Still failing? → Use fallback model
    │
    ├→ Tool failure?
    │   (A tool didn't work)
    │   → Try alternative tool
    │       ↓
    │   Example: RAG search failed → try web search
    │       ↓
    │   Still failing? → Skip tool, proceed without
    │
    ├→ Rate limit?
    │   (Too many requests)
    │   → Queue and wait
    │       ↓
    │   Place request in queue
    │       ↓
    │   Wait for rate limit to reset
    │       ↓
    │   Resume processing
    │
    ├→ Permission denied?
    │   (User not authorized)
    │   → Request approval
    │       ↓
    │   Ask user for permission
    │       ↓
    │   Wait for response
    │       ↓
    │   Proceed if approved
    │
    ├→ Invalid output?
    │   (Agent produced bad output)
    │   → Rephrase and retry
    │       ↓
    │   Send feedback to agent
    │       ↓
    │   Ask agent to try again
    │       ↓
    │   Still invalid? → Use template response
    │
    └→ Unknown error?
        (Something unexpected)
        → Log and report
            ↓
        Log error details
            ↓
        Report to monitoring system
            ↓
        Provide generic error to user
    ↓
Max retries exceeded?
    ├→ Yes: Graceful degradation
    │       ↓
    │   Return best available result
    │       ↓
    │   Explain what went wrong
    │       ↓
    │   Suggest alternatives
    │
    └→ No: Retry
            ↓
        Go back to error type handling
```

---

## 12. Human-in-the-Loop Decision Tree

Sometimes the system needs to ask a human for help.

```
Action Required
    ↓
What's the risk level?
    │
    ├→ Critical: Always ask
    │   (Could cause legal or financial harm)
    │   Example: "Should I sign this contract?"
    │   Action: STOP, wait for human approval
    │
    ├→ High: Ask if uncertain
    │   (Might cause problems)
    │   Example: "Should I delete this clause?"
    │   Action: Ask if confidence < 80%
    │
    ├→ Medium: Ask if first time
    │   (Unfamiliar situation)
    │   Example: "Should I use this template?"
    │   Action: Ask if user hasn't done this before
    │
    └→ Low: Auto-execute
        (Safe, routine operations)
        Example: "Summarize this document"
        Action: Proceed automatically
    ↓
User available?
    ├→ Yes: Wait for response
    │       ↓
    │   Present question clearly
    │       ↓
    │   Wait for user input
    │       ↓
    │   Proceed based on response
    │
    └→ No: Queue for later
            ↓
        Save action to queue
            ↓
        Notify user when available
            ↓
        Resume when user responds
```

---

## 13. Background Agent Decision Tree

Some work happens in the background while the user waits or does other things.

```
Background Job Submitted
    ↓
Agent Planner
    ↓
What needs to be done?
    ↓
Practice area detection
    ├→ Corporate law
    ├→ Litigation
    ├→ Real estate
    ├→ Intellectual property
    └→ General practice
    ↓
Sub-tasks generated
    ↓
Example: "Review all contracts in VDR"
    ├→ Task 1: List all documents
    ├→ Task 2: Read each document
    ├→ Task 3: Analyze each document
    ├→ Task 4: Generate summary report
    └→ Task 5: Compile findings
    ↓
Dependency ordering (topological sort)
    ↓
What must happen first?
    ├→ Task 1 (no dependencies)
    ├→ Task 2 (depends on Task 1)
    ├→ Task 3 (depends on Task 2)
    ├→ Task 4 (depends on Task 3)
    └→ Task 5 (depends on Task 4)
    ↓
Execute sub-tasks
    ├→ Each sub-task gets its own agent
    ├→ Results collected
    └─── Errors handled
        ↓
    Artifact reduction
        ↓
    Combine outputs from all sub-tasks
        ↓
    Remove duplicates
        ↓
    Prioritize by importance
        ↓
    Report assembly
        ↓
    Create final report
        ↓
    Format for readability
        ↓
    Include executive summary
        ↓
    Notification sent
        ↓
    User notified that job is complete
        ↓
    Results available for review
```

---

## 14. Critique and Validation Decision Tree

The system checks its own work before showing it to the user.

```
Output Generated
    ↓
Self-critique
    ├→ Quality score
    │       Is this response well-written?
    │       Score 1-10
    │
    ├→ Legal accuracy
    │       Are the legal statements correct?
    │       Check against known law
    │
    ├→ Citation verification
    │       Are all citations real and accurate?
    │       Verify each citation exists
    │
    └─── Consistency check
            Is this response consistent with earlier parts?
            Check for contradictions
        ↓
    Score acceptable?
        ├→ Yes: Proceed
        │       ↓
        │   Move to output validation
        │
        └→ No: Regenerate
                ↓
            Send feedback to agent
                ↓
            Agent tries again
                ↓
            Max regenerations?
                ├→ Yes: Use best attempt
                │       ↓
                │   Use highest-scoring version
                │       ↓
                │   Add disclaimer if needed
                │
                └─── No: Try again
                        ↓
                    Go back to self-critique
    ↓
Output validation
    ├→ Format correct?
    │       Is the response properly formatted?
    │       Check structure and layout
    │
    ├→ Complete?
    │       Does it answer the full question?
    │       Check for missing parts
    │
    ├→ Safe?
    │       Is this response safe to share?
    │       Check for sensitive information
    │
    └─── Useful?
            Will this help the user?
            Check practical value
        ↓
    All pass?
        ├─→ Yes: Return to user
        │       Response is ready
        │
        └→ No: Fix and retry
                ↓
            Identify what failed
                ↓
            Fix the specific issue
                ↓
            Re-validate
```

---

## 15. Complete System Flow Diagram

This is the massive diagram showing the ENTIRE flow from user input to final response.

```
USER TYPES MESSAGE
        ↓
┌─────────────────────────────────────┐
│         INTENT CLASSIFIER           │
│                                     │
│  Is it a greeting? ──Yes──→ Return  │
│         │                           │
│         No                          │
│         ↓                           │
│  Platform question? ──Yes──→ Help   │
│         │                           │
│         No                          │
│         ↓                           │
│  References documents? ──Yes──→     │
│  Load document context              │
│         │                           │
│         No                          │
│         ↓                           │
│  Simple request? ──Yes──→ Direct    │
│  tool call ──→ Return answer        │
│         │                           │
│         No                          │
│         ↓                           │
│  Complex request? ──Yes──→          │
│  Mark as complex                    │
│         │                           │
│         No                          │
│         ↓                           │
│  Single agent                       │
│                                     │
│  Confidence > 0.65?                 │
│  AND (complex OR multi_agent)?      │
│         │                           │
│    Yes  │  No                       │
│    ↓    │  ↓                       │
│  ORCHESTRATE  LEGACY PATH           │
│         │       ↓                   │
│         │   Direct LLM response     │
│         │       ↓                   │
│         │   Return to user          │
│         │                           │
└─────────┼───────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│         RATE LIMIT CHECK            │
│                                     │
│  User under limit?                  │
│         │                           │
│    Yes  │  No                       │
│    ↓    │  ↓                       │
│  Continue  Fall back to legacy      │
│         │       ↓                   │
│         │   Direct LLM response     │
│         │       ↓                   │
│         │   Return to user          │
│         │                           │
└─────────┼───────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│         CONTEXT PREPARATION         │
│                                     │
│  Prepare assistant runtime context  │
│  Build context summary              │
│  Load user API keys                 │
│  Load project/workspace data        │
│                                     │
└─────────────────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│         TASK PLANNING               │
│                                     │
│  Create task plan from classification│
│  Assign agent types to tasks        │
│  Set execution strategy             │
│  (sequential or parallel)           │
│                                     │
└─────────────────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│         AGENT COORDINATION          │
│                                     │
│  Get ready tasks (no dependencies)  │
│                                     │
│  Execute batch:                     │
│  ├→ Research Agent (if needed)      │
│  ├→ Document Agent (if needed)      │
│  ├→ Compliance Agent (if needed)    │
│  ├→ Drafting Agent (if needed)      │
│  └→ Generic Agent (if needed)       │
│                                     │
│  Collect results                    │
│  Store in shared context            │
│  Move to next batch                 │
│                                     │
│  All tasks complete?                │
│         │                           │
│    Yes  │  No                       │
│    ↓    │  ↓                       │
│  Continue  Handle failures          │
│         │   (retry or skip)         │
│         │       ↓                   │
│         │   Continue                │
│         │                           │
└─────────┼───────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│         CITATION COLLECTION         │
│                                     │
│  Extract citations from all outputs │
│  Deduplicate citations              │
│  Verify citation format             │
│                                     │
└─────────────────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│         RESPONSE SYNTHESIS          │
│                                     │
│  Combine outputs from all agents    │
│  Create coherent final response     │
│  Apply legal caveats                │
│  Include citations                  │
│                                     │
└─────────────────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│         SELF-CRITIQUE               │
│                                     │
│  Quality score                      │
│  Legal accuracy check               │
│  Citation verification              │
│  Consistency check                  │
│                                     │
│  Score acceptable?                  │
│         │                           │
│    Yes  │  No                       │
│    ↓    │  ↓                       │
│  Continue  Regenerate               │
│         │   (up to 3 times)         │
│         │       ↓                   │
│         │   Use best attempt        │
│         │                           │
└─────────┼───────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│         OUTPUT VALIDATION           │
│                                     │
│  Format correct?                    │
│  Complete?                          │
│  Safe?                              │
│  Useful?                            │
│                                     │
│  All pass?                          │
│         │                           │
│    Yes  │  No                       │
│    ↓    │  ↓                       │
│  Continue  Fix and retry            │
│         │       ↓                   │
│         │   Re-validate             │
│         │                           │
└─────────┼───────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│         LEARNING SYSTEM             │
│                                     │
│  Log response quality               │
│  Update patterns                    │
│  Improve future responses           │
│                                     │
└─────────────────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│         RETURN TO USER              │
│                                     │
│  Stream response to user            │
│  Include citations                  │
│  Add legal disclaimers              │
│  Provide actionable next steps      │
│                                     │
└─────────────────────────────────────┘
          ↓
USER RECEIVES RESPONSE
```

---

## 16. Decision Tree Summary Table

| Decision Point | Options | Outcomes | Files Involved |
|----------------|---------|----------|----------------|
| **Intent Classification** | Simple, Complex, Multi-Agent | Determines pipeline path | `intentClassifier.ts` |
| **Legal Domain** | Contract, Compliance, Research, Document, Template, General, Non-Legal | Selects specialist agent | `types.ts` |
| **Agent Selection** | Research, Document, Compliance, Drafting, Generic | Assigns task to agent | `agentCoordinator.ts` |
| **Tool Selection** | Search, Document, Legal, Generate, Email, Workspace, Other | Picks specific tool | `agentToolExecutor.ts` |
| **Error Type** | Timeout, Tool failure, Rate limit, Permission, Invalid output, Unknown | Determines recovery strategy | `baseAgent.ts` |
| **Risk Level** | Critical, High, Medium, Low | Decides human-in-the-loop | `baseAgent.ts` |
| **Retry Strategy** | Backoff, Alternative tool, Queue, Approval, Rephrase, Log | Handles failures | `agentCoordinator.ts` |
| **Validation** | Format, Completeness, Safety, Usefulness | Ensures quality output | `outputValidator.ts` |
| **Critique** | Quality, Accuracy, Citations, Consistency | Checks own work | `selfCritiqueAgent.ts` |
| **Background Processing** | Planning, Sub-tasks, Dependencies, Execution, Assembly | Handles long-running jobs | `agentPlanner.ts`, `subAgentExecutor.ts` |
| **Execution Strategy** | Parallel, Sequential | Optimizes task execution | `agentCoordinator.ts` |
| **Response Synthesis** | Combine, Deduplicate, Prioritize, Format | Creates final answer | `responseGenerator.ts` |
| **Learning** | Log, Update patterns, Improve | Gets better over time | `learningSystem.ts` |
| **Fallback** | Legacy path, Direct LLM, Graceful degradation | Handles system failures | `index.ts` |
| **VDR Assessment** | Structure detection, Document listing, Priority sorting | Processes virtual data rooms | `preliminaryAssessor.ts` |
| **Report Generation** | Summary, Detailed, Executive, Technical | Creates final reports | `reportAssembler.ts` |
| **Artifact Reduction** | Combine, Deduplicate, Prioritize, Compress | Reduces sub-task outputs | `artifactReducer.ts` |
| **Task Ordering** | Topological sort, Dependency graph, Priority queue | Orders tasks properly | `topologicalSort.ts` |

---

## Quick Reference: Agent Decision Flow

For rapid reference, here's the simplified decision flow:

```
User Message
    ↓
Intent? ──→ Simple? ──→ Direct LLM ──→ Done
    │
    ↓
Complex? ──→ One agent? ──→ Select agent ──→ Execute ──→ Critique ──→ Validate ──→ Done
    │
    ↓
Multi-agent? ──→ Coordinator ──→ Plan tasks ──→ Execute in order ──→ Synthesize ──→ Critique ──→ Validate ──→ Done
    │
    ↓
Error? ──→ Retry ──→ Fallback ──→ Graceful degradation ──→ Done
```

---

## Appendix: Key Timeouts and Limits

| Component | Timeout | Purpose |
|-----------|---------|---------|
| Orchestration | 180 seconds | Total pipeline time |
| Planning | 45 seconds | Task planning phase |
| Agents | 120 seconds | Agent execution phase |
| Synthesis | 45 seconds | Response synthesis phase |
| Individual Agent | 90 seconds | Single agent execution |
| Retry Backoff | 1s, 2s, 4s, 8s | Error recovery delays |
| Max Retries | 3 attempts | Per-task retry limit |
| Confidence Threshold | 0.65 | Minimum for orchestration |
| Critique Regenerations | 3 attempts | Self-improvement limit |

---

*Document Version: 1.0*
*Last Updated: 2026-07-14*
*Part of the Mike Legal AI Platform Documentation*