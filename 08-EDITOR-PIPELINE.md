# Editor Pipeline

## 1. What This Document Is

This document explains how documents flow through the Mike legal AI platform — from the moment a user opens a document to the moment it is saved, exported, or shared. It covers the complete lifecycle of document editing: loading content, rendering it in the correct editor, tracking changes, handling comments, saving edits, and exporting to various formats.

The editor pipeline is the heart of the Mike platform. Every legal document — whether a contract, brief, or memo — passes through this system. Understanding how it works helps developers debug issues, add features, and maintain the codebase.

The system supports two distinct editors, each optimized for different document formats. This dual-editor approach ensures that users get the best experience regardless of whether they are working with simple HTML content or complex DOCX files with tracked changes, comments, and precise formatting requirements.

---

## 2. The Two Editors

The Mike platform uses two separate editors, each designed for a specific use case:

### TipTap Editor

TipTap is a web-native HTML editor. It is fast, lightweight, and ideal for simple documents. Think of it as a rich text editor similar to what you see in Google Docs or Notion. It renders content directly in the browser using HTML and CSS.

**When to use TipTap:**
- Simple documents without complex formatting
- Markdown content
- Documents that do not need DOCX fidelity
- Quick edits where speed matters
- Documents without tracked changes in the original format

**Key characteristics:**
- Renders HTML directly in the browser
- Uses a schema-based system with extensions
- Supports collaborative editing via WebSocket connections
- Fast load times and responsive editing
- Limited DOCX format preservation

### SuperDoc Editor

SuperDoc is a DOCX-fidelity editor. It preserves the exact formatting, styles, and structure of Word documents. Think of it as an embedded version of Microsoft Word that runs inside your browser. It is slower to load but provides pixel-perfect rendering of DOCX files.

**When to use SuperDoc:**
- Complex legal documents with precise formatting
- Documents with tracked changes that need to be preserved
- Documents with comments, headers, footers, and page breaks
- Any document where DOCX fidelity is required
- Collaborative editing on DOCX files

**Key characteristics:**
- Renders DOCX files with full fidelity
- Preserves all formatting, styles, and structure
- Supports tracked changes and comments natively
- Requires a backend session for rendering
- Larger payload size and slower load times

### Why Two Editors?

Legal documents come in many formats. Some are simple text-based documents that do not need complex formatting. Others are precisely formatted contracts where every tab, indent, and font size matters. Using a single editor for both cases would mean either sacrificing speed for fidelity or vice versa. The dual-editor approach gives users the best of both worlds.

---

## 3. Editor Selection Logic

When a user opens a document, the system must decide which editor to use. This decision is based on the document's file format.

```
Document format?
├→ DOCX → SuperDoc
├→ HTML → TipTap
├→ Markdown → TipTap
└→ Unknown → SuperDoc (default)
```

### Decision Flow

1. **Check the file extension.** The system looks at the original file extension stored in the database.
2. **If the file is a DOCX**, use SuperDoc. DOCX files contain complex XML structures that cannot be faithfully reproduced in a simple HTML editor. SuperDoc handles these structures natively.
3. **If the file is HTML**, use TipTap. HTML is the native format for TipTap, so no conversion is needed.
4. **If the file is Markdown**, use TipTap. Markdown can be converted to HTML and rendered in TipTap with full fidelity.
5. **If the format is unknown**, default to SuperDoc. This is the safer choice because SuperDoc can handle more formats and preserves more information.

### Format Detection

The system detects the format through multiple signals:
- The file extension stored in the database (`docx`, `html`, `md`, etc.)
- The MIME type of the file stored in R2 (Cloudflare's object storage)
- The content signature (checking for XML headers, HTML doctypes, etc.)
- The document's metadata flags indicating whether it has tracked changes

If a document has tracked changes in the original DOCX, the system will always use SuperDoc regardless of the current content format, because tracked changes cannot be faithfully represented in TipTap.

---

## 4. TipTap Editor Pipeline

The TipTap editor handles HTML and Markdown documents. Here is the complete flow from loading to export:

### Step 1: Load Document Content

When the user opens a document, the frontend makes an API request to the backend. The backend fetches the document from R2 (Cloudflare's object storage) and returns the content. For HTML documents, the raw HTML is returned. For Markdown documents, the backend converts Markdown to HTML before returning.

```
Frontend → API Request → Backend → R2 Fetch → Content Extraction → HTML Response
```

### Step 2: Initialize TipTap

The frontend receives the HTML content and initializes the TipTap editor. This involves:
- Creating a new TipTap `Editor` instance
- Loading the schema extensions (custom nodes, marks, and extensions)
- Configuring the toolbar and menu items
- Setting up event handlers for user input

### Step 3: Apply Schema Extensions

TipTap uses a schema system that defines what content types are allowed. The Mike platform extends the base TipTap schema with custom extensions:

**Block ID Extension:**
Every block (paragraph, heading, list item, etc.) gets a unique block ID. This ID is used for track changes, comments, and collaborative editing. Without block IDs, the system cannot reliably identify which block a change or comment refers to.

**Custom Nodes:**
- `MentionNode` — For @mentions of users or clauses
- `AnnotationNode` — For legal annotations and highlights
- `FootnoteNode` — For footnotes and endnotes
- `PageBreakNode` — For page breaks in export

**Custom Marks:**
- `HighlightMark` — For highlighting text with colors
- `CommentMark` — For marking text with comments
- `TrackChangeMark` — For marking inserted/deleted text

**Extensions:**
- `CollaborationExtension` — For real-time collaborative editing
- `HistoryExtension` — For undo/redo functionality
- `PlaceholderExtension` — For showing placeholder text in empty blocks

### Step 4: Enable Collaborative Editing

If collaborative editing is enabled, the frontend establishes a WebSocket connection to the backend. The backend manages the real-time synchronization of edits between multiple users. Each user's cursor position and selection are broadcast to all other users.

### Step 5: Handle User Input

TipTap captures all user input through its event system. When the user types, deletes, formats text, or performs any other action, TipTap:
1. Captures the raw input event
2. Applies the change to the internal document model
3. Updates the rendered HTML in the browser
4. Emits change events that the application listens to

### Step 6: Track Changes

When track changes mode is enabled, every edit is marked as a suggestion rather than being applied directly. TipTap uses its `TrackChangeMark` to mark inserted and deleted text. The changes are displayed in different colors:
- Inserted text appears in green
- Deleted text appears in red with a strikethrough

### Step 7: Save Changes

The system uses debounced auto-save. When the user makes a change, the system waits a short period (typically 1-2 seconds) before sending the change to the backend. This prevents overwhelming the server with requests for every keystroke.

```
User types → Change captured → Debounce timer starts → Timer expires → Content serialized → Sent to backend
```

### Step 8: Export

When the user exports the document, the system:
1. Serializes the current TipTap document to HTML
2. Applies any export-specific styling
3. For DOCX export, converts HTML to DOCX using the backend
4. For PDF export, renders the HTML to PDF using Puppeteer
5. Stores the exported file in R2
6. Returns a download URL to the frontend

---

### TipTap Schema

The TipTap schema defines the structure of the document. Here are the key components:

#### Block ID Extension

Every block in the document gets a unique ID. This is critical for track changes and comments because it allows the system to identify exactly which block a change or comment refers to.

```typescript
// Simplified block ID extension
const BlockId = Extension.create({
  name: 'blockId',
  
  addAttributes() {
    return {
      blockId: {
        default: null,
        parseHTML: element => element.getAttribute('data-block-id'),
        renderHTML: attributes => ({
          'data-block-id': attributes.blockId,
        }),
      },
    }
  },
  
  // Auto-assign IDs to new blocks
  onCreate() {
    this.editor.state.doc.descendants(node => {
      if (node.type.isBlock && !node.attrs.blockId) {
        // Generate and assign unique ID
      }
    })
  },
})
```

#### Custom Nodes

Custom nodes extend the document structure beyond what TipTap provides by default:

- **MentionNode**: Allows @mentioning users or legal clauses. Renders as a styled inline element that can be clicked to navigate to the mentioned entity.
- **AnnotationNode**: Allows adding annotations to specific parts of the text. Annotations can be legal notes, warnings, or reference markers.
- **FootnoteNode**: Allows adding footnotes to the document. Footnotes are displayed at the bottom of the page or document.
- **PageBreakNode**: Inserts a page break for PDF/DOCX export. Not visible during editing but affects the exported output.

#### Custom Marks

Marks are inline formatting that can be applied to text:

- **HighlightMark**: Applies colored highlighting to text. Used for emphasizing important clauses or marking areas that need review.
- **CommentMark**: Associates a comment with specific text. The comment is stored separately and linked via the mark.
- **TrackChangeMark**: Marks text as inserted or deleted in track changes mode. Stores the author, timestamp, and change type.

#### Extensions

Extensions add behavior to the editor:

- **CollaborationExtension**: Manages real-time synchronization with other users via WebSocket. Handles conflict resolution and operational transformation.
- **HistoryExtension**: Provides undo/redo functionality by maintaining a history stack of document states.
- **PlaceholderExtension**: Shows placeholder text in empty blocks (e.g., "Start typing here...").

---

### TipTap Toolbar

The toolbar provides quick access to common formatting actions:

| Action | Description |
|--------|-------------|
| **Bold** | Makes selected text bold |
| **Italic** | Makes selected text italic |
| **Underline** | Underlines selected text |
| **Strikethrough** | Strikes through selected text |
| **Heading 1-6** | Changes block to a heading level |
| **Bullet List** | Creates a bulleted list |
| **Numbered List** | Creates a numbered list |
| **Table** | Inserts a table |
| **Image** | Inserts an image (uploaded or linked) |
| **Link** | Creates a hyperlink |
| **Code Block** | Inserts a code block |
| **Block Quote** | Creates a block quote |
| **Horizontal Rule** | Inserts a horizontal line |
| **Undo** | Undoes the last action |
| **Redo** | Redoes the last undone action |

The toolbar is context-sensitive. Some actions are only available when the cursor is in a specific context (e.g., table actions are only available when the cursor is inside a table).

---

## 5. SuperDoc Editor Pipeline

The SuperDoc editor handles DOCX documents with full fidelity. Here is the complete flow:

### Step 1: Load DOCX from R2

When the user opens a DOCX document, the frontend requests the document from the backend. The backend fetches the DOCX file from R2 and extracts the content. The DOCX file is not converted to HTML — it is passed to SuperDoc in its native format.

```
Frontend → API Request → Backend → R2 Fetch → DOCX Binary → SuperDoc Session
```

### Step 2: Initialize SuperDoc Session

SuperDoc runs as a service on the backend. When a user opens a DOCX document, the backend creates a new SuperDoc session:

1. The backend generates a unique session ID
2. The DOCX file is loaded into the session
3. A WebSocket connection is established between the frontend and the SuperDoc service
4. The session is registered in the session manager

### Step 3: Render Document with Fidelity

SuperDoc parses the DOCX XML structure and renders the document with full fidelity. This includes:
- Preserving all fonts, sizes, and colors
- Rendering tables with correct cell borders and spacing
- Displaying images in their correct positions
- Maintaining headers and footers
- Showing page breaks and margins
- Rendering tracked changes and comments

### Step 4: Enable Editing

Once the document is rendered, SuperDoc enables editing. Users can:
- Type and delete text
- Format text (bold, italic, underline, etc.)
- Insert and modify tables
- Add and resize images
- Modify headers and footers
- Add page breaks

All edits are tracked and synchronized through the session.

### Step 5: Track Changes

SuperDoc natively supports track changes. When track changes mode is enabled:
- Inserted text appears in a different color (typically blue or green)
- Deleted text appears with a strikethrough in a different color (typically red)
- Each change is attributed to the user who made it
- Changes include timestamps for audit purposes

### Step 6: Handle Comments

SuperDoc supports comments natively. Users can:
- Select text and add a comment
- Reply to existing comments
- Resolve (mark as done) comments
- Delete comments
- View all comments in a sidebar panel

Comments are stored in the DOCX file structure and preserved when the document is exported.

### Step 7: Save Changes

SuperDoc sessions auto-save changes periodically. The flow is:
1. User makes edits in the SuperDoc editor
2. Changes are captured by the SuperDoc service
3. Changes are synchronized to the backend via WebSocket
4. Backend updates the document in R2
5. Backend creates a new version entry in the database
6. Frontend receives confirmation of the save

### Step 8: Export DOCX

When the user exports the document, SuperDoc generates a new DOCX file from the current state of the document. This DOCX file includes:
- All current content
- All tracked changes (unless the user chooses to accept/reject them first)
- All comments
- All formatting and styles
- All images and embedded objects

The exported DOCX is stored in R2 and a download URL is returned to the frontend.

---

### SuperDoc Session Management

SuperDoc sessions are managed by a dedicated session manager on the backend.

#### Session Creation

When a user opens a DOCX document:
1. The backend checks if a session already exists for this document and user
2. If not, a new session is created with a unique ID
3. The DOCX file is loaded into the session
4. The session is registered in the session manager with metadata (user ID, document ID, creation time, last activity time)

#### Document Loading

The session manager loads the DOCX file into memory. The file is parsed into an internal representation that SuperDoc can render and modify. The original file is preserved in case the user wants to discard changes.

#### Token Management

Each session has an authentication token. The token is used to verify that the user has permission to access the document. Tokens expire after a period of inactivity, and the session is cleaned up.

#### Cleanup

Sessions are cleaned up in several scenarios:
- The user closes the document
- The user's token expires
- The session has been inactive for too long (timeout)
- The server needs to free memory

When a session is cleaned up, any unsaved changes are either saved or discarded based on the cleanup reason.

---

### SuperDoc Features

SuperDoc provides a rich set of features for DOCX editing:

#### Format Preservation

SuperDoc preserves all formatting from the original DOCX file. This includes:
- Font families, sizes, and colors
- Paragraph spacing and indentation
- Line spacing
- Text alignment
- Borders and shading

#### Track Changes

SuperDoc supports full track changes functionality:
- Insertions are marked with colored text and an underline
- Deletions are marked with colored text and a strikethrough
- Each change is attributed to the author
- Changes include timestamps
- Users can accept or reject individual changes or all changes

#### Comments

SuperDoc supports comments that are anchored to specific text ranges:
- Comments are displayed in a sidebar panel
- Users can reply to comments
- Comments can be resolved (marked as done)
- Comments are preserved in the DOCX export

#### Images

SuperDoc supports images in DOCX files:
- Images are rendered at their correct positions
- Users can resize and reposition images
- New images can be inserted
- Image captions are preserved

#### Tables

SuperDoc supports tables with full fidelity:
- Cell borders and spacing are preserved
- Cell merging and splitting works correctly
- Table styles are maintained
- Nested tables are supported

#### Headers and Footers

SuperDoc supports headers and footers:
- Different first page headers/footers
- Odd and even page headers/footers
- Page numbers in headers/footers
- Date and time fields

#### Page Breaks

SuperDoc preserves page breaks:
- Manual page breaks are respected
- Automatic page breaks are calculated correctly
- Section breaks are preserved

---

## 6. Document Loading Pipeline

The complete document loading flow, from user click to editor ready:

### Step 1: User Clicks Document

The user clicks on a document in the document list. This triggers a navigation event in the frontend.

### Step 2: Frontend Requests Document

The frontend sends an API request to the backend with the document ID. The request includes the user's authentication token.

### Step 3: Backend Fetches from R2

The backend looks up the document in the database to find the R2 storage path. It then fetches the file from R2 (Cloudflare's object storage).

### Step 4: Backend Extracts Content

The backend extracts the content from the file. For DOCX files, this means reading the binary data. For HTML files, this means reading the text content. For Markdown files, this means converting to HTML.

### Step 5: Backend Detects Format

The backend determines the document format based on:
- File extension
- MIME type
- Content signature
- Metadata flags (e.g., has tracked changes)

### Step 6: Frontend Initializes Correct Editor

Based on the format information returned by the backend, the frontend initializes either TipTap or SuperDoc:
- **TipTap**: The frontend creates a TipTap editor instance, loads the schema extensions, and sets the HTML content.
- **SuperDoc**: The frontend sends the DOCX data to the SuperDoc service, which creates a session and begins rendering.

### Step 7: Editor Renders Content

The editor renders the document content:
- **TipTap**: The HTML is rendered directly in the browser.
- **SuperDoc**: The DOCX is rendered by the SuperDoc service, which streams the rendered output to the frontend via WebSocket.

### Step 8: User Can Begin Editing

Once the document is fully rendered, the user can begin editing. The editor is in a ready state and will respond to user input.

---

## 7. Editing Pipeline

The editing flow, from user input to saved changes:

### Step 1: User Makes Change

The user performs an editing action — typing text, formatting text, inserting an element, or deleting content.

### Step 2: Editor Captures Change

The editor captures the change through its internal event system:
- **TipTap**: TipTap's ProseMirror foundation captures the change and updates the document model.
- **SuperDoc**: The SuperDoc service captures the change and updates its internal representation.

### Step 3: Change is Tracked

If track changes mode is enabled, the change is marked as a suggestion:
- Inserted text is marked with a "track change insert" attribute
- Deleted text is marked with a "track change delete" attribute
- The change is attributed to the current user with a timestamp

### Step 4: Change is Optionally Validated

Some changes may be validated before being applied:
- Content policy checks (e.g., no profanity, no sensitive data)
- Format validation (e.g., table structure is valid)
- Business rule validation (e.g., certain fields cannot be empty)

### Step 5: Change is Sent to Backend

The change is sent to the backend via API request or WebSocket message:
- **TipTap**: Changes are sent via API request (debounced)
- **SuperDoc**: Changes are sent via WebSocket (real-time)

### Step 6: Backend Updates Document

The backend receives the change and updates the document:
- For TipTap documents: The HTML content is updated
- For SuperDoc documents: The DOCX file is updated

### Step 7: Backend Creates New Version

The backend creates a new version entry in the database. Each version stores:
- The document content at that point in time
- The user who made the change
- The timestamp of the change
- A summary of what changed

### Step 8: Frontend Receives Confirmation

The backend sends a confirmation to the frontend. The frontend updates the UI to show that the document is saved (e.g., removing the "unsaved changes" indicator).

---

## 8. Track Changes Pipeline

The track changes system allows users to make edits that are visible as suggestions rather than direct changes. Other users can review these suggestions and accept or reject them.

### Step 1: Enable Track Changes Mode

The user enables track changes mode by clicking the track changes toggle in the toolbar. This sets a flag in the editor that causes all subsequent edits to be tracked.

### Step 2: User Makes Edit

The user performs an editing action — typing, deleting, formatting, etc.

### Step 3: Edit is Marked as Suggestion

Instead of applying the edit directly, the editor marks it as a suggestion:
- **Insertions**: The new text is displayed in a different color (typically green) with an underline
- **Deletions**: The deleted text is displayed in a different color (typically red) with a strikethrough
- **Formatting changes**: The change is noted in the track changes panel

### Step 4: Suggestion is Displayed

The suggestion is displayed in the editor and in the track changes panel. The track changes panel shows:
- A list of all pending suggestions
- The author of each suggestion
- The timestamp of each suggestion
- Accept and reject buttons for each suggestion

### Step 5: Other Users Can Accept/Reject

Other users (or the original author) can review the suggestions and decide to accept or reject them:
- **Accept**: The change is applied permanently. Inserted text becomes normal text. Deleted text is removed.
- **Reject**: The change is discarded. Inserted text is removed. Deleted text is restored.

### Step 6: Accepted Changes are Applied

When a suggestion is accepted:
- The track change mark is removed from the text
- The text becomes part of the normal document
- The change is recorded in the document's history

### Step 7: Rejected Changes are Removed

When a suggestion is rejected:
- The track change mark is removed from the text
- The text reverts to its original state
- The change is recorded as rejected in the document's history

### Step 8: Final Document is Exported

When the document is exported, the user can choose to:
- Include all track changes (for review purposes)
- Accept all changes and export a clean document
- Reject all changes and export the original document

---

### Track Changes Import

When a DOCX file with tracked changes is imported into SuperDoc, the system:

1. **Parse DOCX tracked changes**: The backend reads the DOCX XML structure and identifies all tracked changes (insertions, deletions, formatting changes).
2. **Convert to internal format**: Each tracked change is converted to an internal representation that includes the change type, author, timestamp, and affected text range.
3. **Display in editor**: The tracked changes are rendered in the SuperDoc editor with the appropriate colors and marks.
4. **Enable accept/reject**: Users can accept or reject individual changes or all changes at once.

### Track Changes Export

When a document with tracked changes is exported to DOCX, the system:

1. **Collect all changes**: The system collects all tracked changes from the editor's internal state.
2. **Format as DOCX tracked changes**: Each change is formatted as a DOCX tracked change element (w:ins, w:del, w:rPrChange, etc.).
3. **Inject into DOCX**: The tracked changes are injected into the DOCX XML structure at the correct positions.
4. **Save**: The modified DOCX file is saved to R2.

---

## 9. Comments Pipeline

The comments system allows users to add notes and feedback to specific parts of a document.

### Step 1: User Selects Text

The user selects the text they want to comment on by clicking and dragging.

### Step 2: User Adds Comment

The user clicks the "Add Comment" button in the toolbar or right-clicks and selects "Add Comment" from the context menu.

### Step 3: Comment is Stored

The comment is stored in the document's comment database. The comment includes:
- The author's user ID
- The text range the comment is anchored to
- The comment text
- A timestamp
- The block ID of the anchor block

### Step 4: Comment is Displayed

The commented text is highlighted in the editor. A comment marker (usually a small icon or colored highlight) indicates that a comment exists. The comment text is displayed in a sidebar panel.

### Step 5: Other Users Can Reply

Other users can reply to the comment. Replies are stored in the same comment thread and displayed nested under the original comment.

### Step 6: Comments Can be Resolved

When the issue raised in a comment is addressed, the comment can be marked as "resolved." Resolved comments are hidden from the main view but can be accessed in the comment history.

### Step 7: Comments are Exported to DOCX

When the document is exported to DOCX, comments are preserved in the DOCX structure. They appear as comment annotations in Microsoft Word and can be viewed, replied to, and resolved there.

---

## 10. Document Save Pipeline

The save flow, from user action to persistent storage:

### Step 1: User Clicks Save (or Auto-Save)

The save can be triggered by:
- The user clicking the "Save" button
- Auto-save after a period of inactivity
- Auto-save when the user navigates away from the document
- Auto-save on a periodic interval

### Step 2: Editor Serializes Content

The editor serializes the current document state to the appropriate format:
- **TipTap**: The document model is serialized to HTML
- **SuperDoc**: The document is serialized to DOCX binary

### Step 3: Content is Sent to Backend

The serialized content is sent to the backend via API request:
- **TipTap**: HTML content is sent as a JSON payload
- **SuperDoc**: DOCX binary is sent as a multipart form data payload

### Step 4: Backend Validates Content

The backend validates the content before saving:
- Checks that the content is well-formed
- Validates against any business rules
- Sanitizes the content to prevent security issues
- Checks file size limits

### Step 5: Backend Stores in R2

The validated content is stored in R2 (Cloudflare's object storage). The content is stored at the path determined by the document's ID and version number.

### Step 6: Backend Creates New Version

The backend creates a new version entry in the database. This version entry includes:
- The version number (incremented from the previous version)
- The R2 storage path for this version
- The user who created this version
- A timestamp
- A summary of changes (if available)

### Step 7: Backend Updates Database

The backend updates the document's database record to point to the new version. The old version is preserved in the version history.

### Step 8: Frontend Receives Confirmation

The backend sends a success response to the frontend. The frontend updates the UI to indicate that the document is saved (e.g., removing the "unsaved changes" indicator, showing a "Saved" message).

---

## 11. Document Export Pipeline

The export system generates downloadable files from the current document state.

### DOCX Export

1. **Serialize editor content**: The editor serializes the current document state to a DOCX-compatible format.
2. **Apply styles**: Any export-specific styles are applied (e.g., custom fonts, page layout).
3. **Generate DOCX**: The backend generates a DOCX file from the serialized content. For TipTap documents, this involves converting HTML to DOCX. For SuperDoc documents, this involves generating the DOCX from the internal representation.
4. **Store in R2**: The generated DOCX is stored in R2 with a temporary path.
5. **Return download URL**: A signed URL is generated and returned to the frontend. The URL expires after a short period (e.g., 1 hour).

### PDF Export

1. **Render document HTML**: The document content is rendered as HTML. For SuperDoc documents, this involves converting the DOCX to HTML.
2. **Use Puppeteer to render**: The HTML is loaded into a headless Chrome browser via Puppeteer. Puppeteer renders the HTML as it would appear in a browser.
3. **Generate PDF**: Puppeteer generates a PDF from the rendered HTML. This preserves all formatting, images, and layout.
4. **Store in R2**: The generated PDF is stored in R2 with a temporary path.
5. **Return download URL**: A signed URL is generated and returned to the frontend.

---

## 12. Document Comparison Pipeline

The document comparison system allows users to compare two versions of a document and see the differences.

### Step 1: Load Two Documents

The user selects two documents (or two versions of the same document) to compare. Both documents are loaded from R2.

### Step 2: Extract Text from Both

The text content is extracted from both documents. For DOCX files, this involves parsing the XML and extracting the text nodes. For HTML files, this involves parsing the HTML and extracting the text.

### Step 3: Run Diff Algorithm

A diff algorithm compares the two text extracts and identifies the differences. The algorithm used is typically a variant of the Longest Common Subsequence (LCS) algorithm, which identifies the minimum set of changes needed to transform one document into the other.

### Step 4: Categorize Changes

The differences are categorized into:
- **Insertions**: Text that exists in the second document but not in the first
- **Deletions**: Text that exists in the first document but not in the second
- **Modifications**: Text that exists in both but has been changed
- **Moves**: Text that has been moved to a different position

### Step 5: Generate Visual Diff

A visual diff is generated that shows the differences in a human-readable format:
- Insertions are shown in green
- Deletions are shown in red with a strikethrough
- Modifications are shown with both the old and new text highlighted

### Step 6: Display Side-by-Side

The visual diff is displayed to the user in a side-by-side view. The left panel shows the original document, and the right panel shows the modified document. Changes are highlighted in both panels.

### Step 7: Generate Summary

A summary is generated that includes:
- The total number of changes
- The number of insertions, deletions, and modifications
- A list of the most significant changes
- The overall similarity score between the two documents

---

## 13. Template Pipeline

The template system allows users to create documents from pre-defined templates.

### Step 1: User Selects Template

The user browses the template library and selects a template. Templates are organized by category (contracts, briefs, memos, etc.).

### Step 2: Template is Loaded

The selected template is loaded from the template storage. Templates are stored as DOCX files with placeholder variables.

### Step 3: Placeholders are Detected

The template engine scans the template for placeholders. Placeholders are typically marked with a specific syntax (e.g., `{{client_name}}`, `{{contract_date}}`).

### Step 4: User Fills Placeholders

The user fills in the placeholder values through a form. The form is generated dynamically based on the placeholders detected in the template.

### Step 5: Template is Populated

The template engine replaces each placeholder with the user-provided value. The engine handles different data types (text, dates, numbers, etc.) and formats them appropriately.

### Step 6: Document is Generated

The populated template is saved as a new document. The document is stored in R2 and a database entry is created.

### Step 7: Document is Opened in Editor

The newly generated document is opened in the appropriate editor (TipTap or SuperDoc) so the user can make any additional edits.

---

## 14. Collaborative Editing

The collaborative editing system allows multiple users to edit the same document simultaneously.

### Real-Time Sync

Changes made by one user are broadcast to all other users in real-time via WebSocket connections. The system uses Operational Transformation (OT) or Conflict-free Replicated Data Types (CRDTs) to ensure that all users see a consistent view of the document.

### Conflict Resolution

When two users make conflicting changes (e.g., both edit the same paragraph at the same time), the system resolves the conflict automatically. The resolution strategy depends on the type of change:
- **Text edits**: Last-write-wins with operational transformation
- **Formatting changes**: Last-write-wins
- **Structural changes** (e.g., inserting a table): Queued and applied sequentially

### User Presence

The system shows which users are currently viewing or editing the document. Each user's avatar or name is displayed in the document header or sidebar. Users can see when other users joined and left the document.

### Cursor Tracking

Each user's cursor position is broadcast to all other users. Other users' cursors are displayed in the document with the user's name and color. This helps users avoid editing the same section simultaneously.

### Selection Sharing

When a user selects text, the selection is broadcast to all other users. Other users' selections are displayed with the user's color. This helps users see what other users are focused on.

---

## 15. Editor Performance

The editor pipeline includes several performance optimizations:

### Lazy Loading

Documents are loaded on-demand. When the user opens a document, only the visible portion is loaded initially. The rest of the document is loaded as the user scrolls. This reduces initial load time for large documents.

### Virtual Scrolling

For very large documents, the editor uses virtual scrolling. Only the visible blocks are rendered in the DOM. As the user scrolls, blocks are added and removed from the DOM. This keeps the DOM small and responsive even for documents with thousands of blocks.

### Debounced Saves

Changes are not saved immediately. Instead, a debounce timer is used. When the user makes a change, the timer is reset. When the timer expires (e.g., after 2 seconds of inactivity), the changes are saved. This prevents overwhelming the server with requests for every keystroke.

### Optimistic Updates

When the user makes a change, the editor updates the UI immediately without waiting for server confirmation. This makes the editor feel responsive even on slow connections. If the save fails, the editor shows an error and allows the user to retry.

### Background Sync

Changes are synchronized in the background. The editor does not block user input while waiting for server responses. All network operations are asynchronous and non-blocking.

---

## 16. Editor Error Handling

The editor pipeline includes comprehensive error handling:

### Load Failures

If the document fails to load:
- The editor shows an error message with details about the failure
- The user can retry loading the document
- If the document is corrupted, the editor offers to load the last known good version

### Save Failures

If the document fails to save:
- The editor shows a warning that changes may be lost
- The changes are stored locally (in localStorage or IndexedDB) so they can be recovered
- The editor retries the save automatically
- If the retry fails, the user is prompted to download their changes as a local file

### Sync Conflicts

If a sync conflict occurs:
- The editor shows a notification that a conflict was detected
- The conflict is resolved automatically using the conflict resolution strategy
- If automatic resolution fails, the user is prompted to resolve the conflict manually

### Format Errors

If the document contains format errors:
- The editor shows a warning that the document has formatting issues
- The editor attempts to recover as much content as possible
- The user is offered to repair the document or continue with the current state

### Recovery Strategies

The editor uses several recovery strategies:
- **Local storage**: Unsaved changes are stored locally so they can be recovered after a crash
- **Version history**: Previous versions of the document can be restored
- **Auto-save**: Changes are saved automatically to minimize data loss
- **Conflict resolution**: Conflicts are resolved automatically to prevent data loss

---

## 17. Complete Editor Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              COMPLETE EDITOR PIPELINE                           │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐
  │  User Clicks │
  │   Document   │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ Frontend API │────▶│   Backend    │────▶│   R2 Fetch   │
  │   Request    │     │   Process    │     │   Document   │
  └──────────────┘     └──────────────┘     └──────┬───────┘
                                                    │
                                                    ▼
                                            ┌──────────────┐
                                            │    Format    │
                                            │   Detection  │
                                            └──────┬───────┘
                                                    │
                              ┌─────────────────────┼─────────────────────┐
                              │                     │                     │
                              ▼                     ▼                     ▼
                      ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
                      │   DOCX →     │     │   HTML →     │     │  Unknown →   │
                      │   SuperDoc   │     │    TipTap    │     │   SuperDoc   │
                      └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
                              │                     │                     │
                              ▼                     ▼                     ▼
                      ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
                      │   Session    │     │   TipTap     │     │   Session    │
                      │   Create     │     │  Initialize  │     │   Create     │
                      └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
                              │                     │                     │
                              ▼                     ▼                     ▼
                      ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
                      │  Load DOCX   │     │  Load HTML   │     │  Load DOCX   │
                      │  from R2     │     │  Content     │     │  from R2     │
                      └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
                              │                     │                     │
                              ▼                     ▼                     ▼
                      ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
                      │   Render     │     │  Apply Schema│     │   Render     │
                      │   with       │     │  Extensions  │     │   with       │
                      │   Fidelity   │     │              │     │   Fidelity   │
                      └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
                              │                     │                     │
                              └─────────────────────┼─────────────────────┘
                                                    │
                                                    ▼
                                            ┌──────────────┐
                                            │   User Can   │
                                            │    Begin     │
                                            │   Editing    │
                                            └──────┬───────┘
                                                    │
                                                    ▼
                    ┌───────────────────────────────────────────────────────────┐
                    │                    EDITING LOOP                           │
                    │                                                           │
                    │  ┌────────────┐   ┌────────────┐   ┌────────────┐       │
                    │  │   User     │   │   Editor   │   │   Change   │       │
                    │  │   Makes    │──▶│  Captures  │──▶│  is Tracked│       │
                    │  │   Change   │   │   Change   │   │            │       │
                    │  └────────────┘   └────────────┘   └─────┬──────┘       │
                    │                                          │               │
                    │                                          ▼               │
                    │  ┌────────────┐   ┌────────────┐   ┌────────────┐       │
                    │  │  Backend   │   │  Backend   │   │  Change is │       │
                    │  │  Updates   │◀──│  Validates │◀──│  Sent to   │       │
                    │  │  Document  │   │  Content   │   │  Backend   │       │
                    │  └─────┬──────┘   └────────────┘   └────────────┘       │
                    │        │                                                  │
                    │        ▼                                                  │
                    │  ┌────────────┐   ┌────────────┐                         │
                    │  │  Backend   │──▶│  Frontend  │                         │
                    │  │  Creates   │   │  Receives  │                         │
                    │  │  Version   │   │ Confirm.   │                         │
                    │  └────────────┘   └────────────┘                         │
                    │                                                           │
                    └───────────────────────────────────────────────────────────┘
                                                    │
                                                    ▼
                    ┌───────────────────────────────────────────────────────────┐
                    │                    TRACK CHANGES                          │
                    │                                                           │
                    │  ┌────────────┐   ┌────────────┐   ┌────────────┐       │
                    │  │  Enable    │──▶│   Edit is  │──▶│ Suggestion │       │
                    │  │  Track     │   │   Marked   │   │ Displayed  │       │
                    │  │  Changes   │   │            │   │            │       │
                    │  └────────────┘   └────────────┘   └─────┬──────┘       │
                    │                                          │               │
                    │                                          ▼               │
                    │  ┌────────────┐   ┌────────────┐   ┌────────────┐       │
                    │  │   Final    │◀──│  Rejected  │◀──│   User     │       │
                    │  │  Document  │   │  Changes   │   │  Accepts/  │       │
                    │  │  Exported  │   │  Removed   │   │  Rejects   │       │
                    │  └────────────┘   └────────────┘   └────────────┘       │
                    │                                                           │
                    └───────────────────────────────────────────────────────────┘
                                                    │
                                                    ▼
                    ┌───────────────────────────────────────────────────────────┐
                    │                    SAVE PIPELINE                          │
                    │                                                           │
                    │  ┌────────────┐   ┌────────────┐   ┌────────────┐       │
                    │  │  User      │──▶│  Editor    │──▶│  Content   │       │
                    │  │  Clicks    │   │ Serializes │   │  Sent to   │       │
                    │  │  Save      │   │  Content   │   │  Backend   │       │
                    │  └────────────┘   └────────────┘   └─────┬──────┘       │
                    │                                          │               │
                    │                                          ▼               │
                    │  ┌────────────┐   ┌────────────┐   ┌────────────┐       │
                    │  │  Frontend  │◀──│  Backend   │◀──│  Backend   │       │
                    │  │  Receives  │   │  Updates   │   │  Stores    │       │
                    │  │ Confirm.   │   │  Database  │   │  in R2     │       │
                    │  └────────────┘   └────────────┘   └────────────┘       │
                    │                                                           │
                    └───────────────────────────────────────────────────────────┘
                                                    │
                                                    ▼
                    ┌───────────────────────────────────────────────────────────┐
                    │                   EXPORT PIPELINE                         │
                    │                                                           │
                    │  ┌────────────┐   ┌────────────┐   ┌────────────┐       │
                    │  │ Serialize  │──▶│  Apply     │──▶│  Generate  │       │
                    │  │  Content   │   │  Styles    │   │  DOCX/PDF  │       │
                    │  └────────────┘   └────────────┘   └─────┬──────┘       │
                    │                                          │               │
                    │                                          ▼               │
                    │  ┌────────────┐   ┌────────────┐                         │
                    │  │  Return    │◀──│  Store in  │                         │
                    │  │ Download   │   │    R2      │                         │
                    │  │   URL      │   │            │                         │
                    │  └────────────┘   └────────────┘                         │
                    │                                                           │
                    └───────────────────────────────────────────────────────────┘
```

---

## 18. Key Files Reference

| File | Description |
|------|-------------|
| `frontend/src/client/components/pages/DocumentEditorPage.tsx` | Main editor page component (416KB). Orchestrates the entire editing experience, including editor initialization, toolbar, sidebars, and document management. |
| `frontend/src/client/components/TiptapEditor.tsx` | TipTap HTML editor component. Initializes TipTap with schema extensions, handles user input, and manages collaborative editing. |
| `frontend/src/client/components/superdoc/SuperDocDocumentEditor.tsx` | SuperDoc DOCX editor component. Manages the SuperDoc session, renders DOCX content, and handles user interaction. |
| `frontend/src/client/components/superdoc/SuperDocToolbar.tsx` | SuperDoc toolbar component. Provides formatting and editing actions for the SuperDoc editor. |
| `frontend/src/client/components/pages/document-editor/exportDocument.ts` | Export utilities. Handles DOCX and PDF export, including format conversion and R2 storage. |
| `frontend/src/client/components/trackChanges/` | Track changes UI components. Displays track changes panel, accept/reject buttons, and change summaries. |
| `packages/editor-schema/src/` | TipTap schema extensions. Custom nodes, marks, and extensions for the TipTap editor. |
| `packages/shared-types/src/documents.ts` | Document type definitions. Shared TypeScript types for documents, versions, and metadata. |
| `backend/src/lib/docxTrackedChanges.ts` | Track changes backend logic. Parses and manipulates tracked changes in DOCX files. |
| `backend/src/lib/docxTrackChangesExport.ts` | Track changes export. Generates DOCX tracked changes from the internal representation. |
| `backend/src/lib/docxTrackChangesImport.ts` | Track changes import. Parses DOCX tracked changes into the internal representation. |
| `backend/src/lib/docxCompare.ts` | Document comparison. Runs diff algorithms and generates visual diffs between two documents. |
| `backend/src/lib/docxComments.ts` | Comments backend logic. Parses and manipulates comments in DOCX files. |
| `backend/src/lib/docxTemplateAnalyzer.ts` | Template analysis. Detects placeholders and structure in DOCX templates. |
| `backend/src/docx/templateEngine.ts` | Template engine. Populates templates with user-provided values and generates documents. |
| `backend/src/lib/superdoc/` | SuperDoc service. Manages SuperDoc sessions, rendering, and editing operations. |
| `backend/src/lib/superdoc/session-manager.ts` | SuperDoc session manager. Creates, maintains, and cleans up SuperDoc sessions. |
| `backend/src/lib/sandbox/service.ts` | Sandbox service. Provides isolated execution environments for document processing. |
| `docs/superdoc-*.txt` | SuperDoc documentation files (10 files). Reference documentation for the SuperDoc service. |
