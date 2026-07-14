# Document Pipeline

> This document describes every transformation a legal document goes through inside Mike — from the moment a user drags a file onto the screen to the moment a polished DOCX, PDF, PPTX, or XLSX lands back in their hands. Think of it as the complete assembly line for legal documents.

---

## 1. What This Document Is

When a lawyer uploads a contract, a court judgment, or a stack of discovery files into Mike, the system does far more than just store the file. It reads the document, figures out what kind of document it is, pulls out every useful piece of information, breaks it into searchable chunks, and makes it available for AI-powered analysis. Later, when the lawyer asks Mike to draft a new agreement or redline an existing one, the system generates a brand-new document from scratch — complete with proper formatting, tracked changes, and professional styling.

This document covers every step of that journey. It explains:

- How files get received and validated when they arrive
- How the system reads text out of images and scanned PDFs
- How DOCX files are parsed while preserving all their formatting and track changes
- How the system figures out whether a document is a contract, a court order, or a memo
- How metadata like author names, dates, and legal citations are pulled out
- How documents get broken into pieces for AI search (RAG)
- How new documents are generated from templates or from AI instructions
- How documents get exported in multiple formats
- How two versions of a document get compared side by side
- How tracked changes flow in and out of the system

Every step has been designed with legal work in mind. Court filings have strict formatting rules. Contracts need precise language. Confidentiality is non-negotiable. The pipeline respects all of these constraints.

---

## 2. Document Lifecycle Overview

Here is the high-level path every document follows through the system:

```
Upload → Parse → Classify → Extract → Chunk → Embed → Index → Store → Edit → Version → Export
```

Breaking that down into the detailed steps:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DOCUMENT LIFECYCLE                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. UPLOAD         User drops file → Multer validates → R2 storage     │
│       ↓                                                                 │
│  2. PARSE          PDF/image → OCR → text extraction                   │
│                    DOCX → SuperDoc/mammoth → structured content         │
│       ↓                                                                 │
│  3. CLASSIFY       Keyword analysis → document type detection          │
│       ↓                                                                 │
│  4. EXTRACT        Metadata, dates, citations, financial amounts       │
│       ↓                                                                 │
│  5. CHUNK          Split by paragraph/section → overlap handling        │
│       ↓                                                                 │
│  6. EMBED          Text → vector embeddings (semantic search)          │
│       ↓                                                                 │
│  7. INDEX          Vectors stored in RAG collection                    │
│       ↓                                                                 │
│  8. STORE          Database record + Cloudflare R2 backup              │
│       ↓                                                                 │
│  9. EDIT           AI edits, track changes, comments                   │
│       ↓                                                                 │
│  10. VERSION       Version snapshots, diff generation                  │
│       ↓                                                                 │
│  11. EXPORT        DOCX / PDF / PPTX / XLSX / HTML output             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

Each step can happen independently. A document might be uploaded and never edited. Another might be generated from scratch without ever being uploaded. The pipeline is flexible — not every document goes through every step.

---

## 3. File Upload Pipeline

The upload pipeline is the front door. Everything starts here.

### 3.1 Multer Receives the File

When a user drags a file onto the Mike interface or clicks the upload button, the frontend sends the file to a backend endpoint. On the server side, **multer** — a Node.js middleware for handling multipart form data — catches the incoming file.

Multer sits between the raw HTTP request and the application code. It reads the file stream, buffers it into memory (or to a temporary disk file for very large uploads), and attaches the file metadata to the request object so the rest of the code can access it.

### 3.2 File Type Validation

Not every file type is welcome. Mike checks the file in two ways:

1. **Extension check**: The file extension (`.docx`, `.pdf`, `.png`, etc.) is checked against an allowlist of supported types.
2. **Magic byte check**: The first few bytes of the file are inspected to verify the file actually matches its claimed type. A `.docx` file should start with the PK zip header. A PDF should start with `%PDF-`. This prevents users from disguising executable files as documents.

If the file fails validation, the upload is rejected immediately with a clear error message.

### 3.3 Size Limits

Every file is checked against size limits. The system enforces a maximum file size (configurable, typically around 50-100 MB for most document types). This prevents:

- Memory exhaustion from enormous files
- Storage abuse
- Unreasonably long processing times for giant scanned PDFs

If a file exceeds the limit, the upload is rejected before any processing begins.

### 3.4 Virus Scanning

Before a file gets stored permanently, it passes through a virus scanning check. The system scans the file content for known malware signatures. If a threat is detected, the file is discarded and the user is notified.

### 3.5 Storage to Cloudflare R2

Once validated, the file gets stored in **Cloudflare R2**, an S3-compatible object storage service. R2 provides:

- **Durability**: Files are replicated across multiple data centers
- **Encryption at rest**: All stored files are encrypted
- **CDN delivery**: Files can be served quickly to users worldwide
- **Cost efficiency**: R2 has no egress fees, which matters for large document libraries

The storage key (the path where the file lives in R2) is recorded in the database so the file can be retrieved later.

### 3.6 Metadata Extraction

Even before full parsing begins, basic metadata is pulled from the file:

- Original filename
- File size in bytes
- MIME type (e.g., `application/vnd.openxmlformats-officedocument.wordprocessingml.document`)
- Upload timestamp
- User who uploaded the file

### 3.7 Database Record Creation

A record is created in the **documents** table (`backend/src/db/schema.ts:422`) capturing all the information gathered so far:

- Document ID (auto-generated UUID)
- Project association
- User who owns the document
- Filename and file type
- Storage path in R2
- Size in bytes
- Status set to `"processing"` (the document is not ready yet)
- Lifecycle status set to `"DRAFT"`

The document is now in the system. Next, the heavy processing begins.

---

## 4. OCR Pipeline

OCR (Optical Character Recognition) is how Mike reads text out of images and scanned PDFs. Not every document arrives as a clean DOCX. Lawyers deal with scanned contracts, photographed court orders, and image-heavy filings every day.

### 4.1 PDF Parsing

For PDF files, the system uses **pdfjs-dist** (the Mozilla PDF.js library). This library:

1. Opens the PDF binary
2. Iterates through every page
3. Extracts the text content from each page
4. Preserves the reading order (top-to-bottom, left-to-right)

PDFs can contain text in two forms:
- **Native text**: Text that was created digitally (e.g., a Word document saved as PDF). This text is directly extractable.
- **Scanned images**: Pages that are just images of paper documents. These require OCR.

The system detects which type each page is and routes it accordingly.

### 4.2 Image Text Extraction

For pure image files (PNG, JPEG, TIFF) or scanned PDF pages, the OCR engine kicks in. The system:

1. Preprocesses the image (contrast adjustment, noise reduction, deskewing)
2. Applies OCR to recognize characters and words
3. Outputs the recognized text with confidence scores

### 4.3 Layout Detection

Legal documents have complex layouts. The system doesn't just dump all the text into one blob. It detects:

- **Columns**: Many court filings use two-column layouts
- **Text blocks**: Paragraphs, headings, and sidebars are identified separately
- **Reading order**: The system determines the correct reading sequence even in multi-column layouts

### 4.4 Table Extraction

Tables are critical in legal documents — financial schedules, comparison charts, pleadings matrices. The system:

1. Detects grid lines and cell boundaries
2. Maps text to cells
3. Preserves row and column structure
4. Outputs tables as structured data (arrays of arrays)

### 4.5 Header/Footer Detection

Legal documents often have headers (case numbers, firm names) and footers (page numbers, confidentiality notices). The system:

1. Identifies repeated content at the top and bottom of pages
2. Separates headers/footers from the main body content
3. Stores them as metadata rather than mixing them into the document flow

### 4.6 Footnote Extraction

Footnotes are common in legal briefs and academic-style legal writing. The system:

1. Detects footnote markers in the body text (superscript numbers)
2. Finds the corresponding footnote text at the bottom of the page
3. Links them together so the relationship is preserved

### 4.7 Quality Scoring

After OCR, the system assigns a quality score to the extracted text. This score reflects:

- **Confidence levels** from the OCR engine
- **Text coherence**: Does the extracted text make sense as sentences?
- **Structure preservation**: Were tables, lists, and headings detected correctly?

Low-quality OCR results are flagged so downstream processes (like AI analysis) know to be cautious with the text.

---

## 5. DOCX Processing Pipeline

DOCX files are the bread and butter of legal work. Mike has a dedicated pipeline for handling them, built around the **SuperDoc SDK** and supplemented by **mammoth.js** for fallback parsing.

### 5.1 DOCX Parsing (SuperDoc SDK)

A DOCX file is actually a ZIP archive containing XML files. The SuperDoc SDK handles the complexity:

1. Unzips the DOCX archive
2. Reads the main document XML (`word/document.xml`)
3. Parses the XML into an internal document model
4. Extracts text, formatting, and structure

The SDK preserves:
- **Paragraphs and runs**: Each piece of text with its formatting
- **Bold, italic, underline**: Inline formatting
- **Headings**: Document structure (H1, H2, H3, etc.)
- **Lists**: Bullet and numbered lists
- **Tables**: Table structure and cell content
- **Images**: Embedded images (stored separately)

### 5.2 Content Extraction

Once parsed, the content is extracted in multiple forms:

- **Plain text**: For search indexing and AI analysis
- **Structured content**: Preserving paragraphs, headings, and lists
- **HTML representation**: For display in the web editor (TipTap)
- **OOXML fragments**: For operations that need to manipulate the raw XML (track changes, comments)

### 5.3 Style Preservation

When Mike edits a DOCX file, it tries to preserve the original styling:

- Font families and sizes are maintained
- Colors are preserved
- Paragraph spacing and indentation are kept
- Page margins and orientation are respected
- Headers and footers are carried forward

This is important because legal documents have strict formatting rules (court filing requirements, firm style guides).

### 5.4 Track Changes Import

If the DOCX already contains tracked changes (revisions from Word), the system imports them:

1. Opens the DOCX and navigates to `word/document.xml`
2. Finds all `<w:ins>` (insertion) and `<w:del>` (deletion) elements
3. Extracts the author, date, and text for each change
4. Converts them to HTML with data attributes so they display in the editor

This is handled by `backend/src/lib/docxTrackChangesImport.ts`.

### 5.5 Comment Extraction

Comments (the yellow sticky-note-style annotations in Word) are also extracted:

1. Reads `word/comments.xml` from the DOCX
2. Maps each comment to its anchor text in the document
3. Extracts author, date, comment body, and resolution status
4. Stores them as structured data

This is handled by `backend/src/lib/docxComments.ts`.

### 5.6 Metadata Extraction

DOCX files contain rich metadata in `docProps/core.xml` and `docProps/app.xml`:

- **Title, subject, keywords**: Document identification
- **Author and last modified by**: Who worked on the document
- **Creation date and modification date**: Timeline information
- **Word count and page count**: Document statistics
- **Revision number**: How many times the document was edited
- **Application version**: Which version of Word created the file

### 5.7 Template Detection

The system also checks whether a DOCX is a template:

1. Scans for placeholder patterns like `{{field_name}}` or `[ENTER DATE HERE]`
2. Identifies the types of placeholders (text, date, number, email, phone)
3. Counts how many times each placeholder appears
4. Generates a preview HTML showing the template with blank fields

This is handled by `backend/src/lib/docxTemplateAnalyzer.ts`.

---

## 6. Document Classification

Once the text is extracted, the system figures out what kind of document it is. This matters because different document types need different analysis approaches.

### 6.1 Type Detection

The classifier (`backend/src/lib/legal/document-classifier.ts`) examines the title and first paragraph of the document. It uses keyword matching to identify the document type:

| Keywords Found | Document Type |
|---|---|
| "non-disclosure", "confidentiality agreement" | NDA |
| "shareholders", "shareholder agreement" | Shareholders Agreement |
| "share purchase" | Share Purchase Agreement |
| "memorandum of understanding" | MOU |
| "legal notice", "notice is hereby given" | Legal Notice |
| "sale deed", "conveyance deed" | Deed |
| "power of attorney" | Power of Attorney |
| "employment agreement", "appointment letter" | Employment |
| "lease agreement", "rent agreement" | Lease |
| "board resolution", "special resolution" | Resolution |
| "plaint", "cause of action" | Plaint |
| "written statement", "order viii" | Written Statement |
| "judgment", "court order", "ratio decidendi" | Judgment |
| "last will", "testament" | Will |
| "privacy policy", "terms of use" | Policy |
| "whereas", "now therefore", "in witness whereof" | Contract |
| "whereas", "now therefore" | Agreement |

### 6.2 Practice Area Detection

Beyond the document type, the system can detect the practice area:

- **Corporate/M&A**: Share purchase agreements, shareholder agreements, board resolutions
- **Litigation**: Plaints, written statements, judgments
- **Real Estate**: Lease agreements, sale deeds, conveyance deeds
- **Employment**: Employment agreements, appointment letters
- **Intellectual Property**: Licensing agreements, patent filings
- **Commercial**: NDAs, MOUs, supply agreements

### 6.3 Jurisdiction Detection

The system looks for jurisdiction indicators:

- Court names (Supreme Court, High Court, District Court)
- State or country references
- Specific legal citations (e.g., "AIR 2020 SC 1234" indicates Indian Supreme Court)
- References to specific statutes (Indian Contract Act, Companies Act, etc.)

### 6.4 Confidence Scoring

The classification result includes a confidence score:

- **High confidence**: Multiple strong keyword matches
- **Medium confidence**: Some keywords found, but ambiguous
- **Low confidence**: Few or conflicting signals

Low-confidence results are flagged for manual review.

### 6.5 Manual Override

Users can override the automatic classification. If the system detects a document as a "contract" but the user knows it is a "memorandum," they can change it. The system will re-run relevant analysis based on the corrected type.

---

## 7. Metadata Extraction

Metadata is the hidden information that makes documents searchable and organized. Mike extracts several categories of metadata.

### 7.1 Basic Document Info

- **Title**: From the document's title property or first heading
- **Author**: From DOCX metadata or first-person references in the text
- **Creation date**: When the document was originally created
- **Modification date**: When it was last changed
- **Word count**: Total words in the document
- **Page count**: Number of pages (from PDF metadata or calculated from text length)

### 7.2 Key Terms and Names

The system identifies important terms and names in the document:

- **Party names**: Companies, individuals, and organizations mentioned
- **Defined terms**: Terms that are explicitly defined in the document (e.g., "the Company" means XYZ Corp)
- **Legal terms of art**: Words like "indemnification", "force majeure", "arbitration"

### 7.3 Dates and Deadlines

Legal documents are full of dates. The system extracts:

- **Effective dates**: When the agreement starts
- **Expiration dates**: When it ends
- **Notice periods**: How much advance notice is required
- **Filing deadlines**: Court filing deadlines
- **Compliance dates**: Dates by which certain actions must be taken

### 7.4 Financial Amounts

Contracts and court orders often contain financial terms:

- **Payment amounts**: Dollar figures, percentages
- **Penalty clauses**: Late fees, liquidated damages
- **Interest rates**: Applied to overdue amounts
- **Cap amounts**: Maximum liability limits
- **Threshold amounts**: Minimum values for certain triggers

### 7.5 Legal Citations

For court filings and legal research documents, citations are critical:

- **Case citations**: References to court decisions (e.g., "(2022) 7 SCC 508")
- **Statute citations**: References to laws and regulations
- **Regulation citations**: References to administrative rules

The system uses pattern matching (`backend/src/lib/legal/citationPatterns.ts`) to detect Indian legal citation formats:

- SCC citations: `(2022) 7 SCC 508`
- AIR citations: `AIR 2020 SC 1234`
- SCC OnLine citations: `2023 SCC OnLine SC 961`
- MANU citations: `MANU/SC/1234/2023`
- writ petition numbers: `W.P.(C) No. 123/2023`

---

## 8. Chunking Pipeline

Documents need to be broken into smaller pieces for AI search (RAG — Retrieval Augmented Generation). You cannot feed an entire 100-page contract into a search query. The chunking pipeline handles this.

### 8.1 Splitting Strategies

The system supports multiple ways to split a document:

- **By paragraph**: Each paragraph becomes a chunk. Good for contracts where each clause is self-contained.
- **By section**: Headings define section boundaries. Good for structured documents with clear divisions.
- **By page**: Each page becomes a chunk. Good for court filings where page numbers matter.
- **By fixed size**: A fixed number of tokens per chunk with overlap. Good for unstructured text.

### 8.2 Overlap Handling

When splitting by size, adjacent chunks overlap by a configurable number of tokens (typically 100-200 tokens). This ensures that a sentence split across two chunks is still searchable from either side.

### 8.3 Metadata Preservation

Every chunk carries metadata from its parent document:

- Document ID
- Document type
- Page number (if available)
- Section heading
- Position within the document (start character, end character)
- Any tags or labels applied to the parent document

### 8.4 Citation Mapping

If a chunk contains legal citations, those citations are tracked. This allows users to search by citation and find the relevant chunk of text.

### 8.5 Context Window Management

Different AI models have different context window sizes. The chunking pipeline respects these limits:

- Chunks are sized to fit within the model's context window
- When combining chunks for analysis, the system respects token limits
- Metadata is compressed to minimize overhead

---

## 9. Embedding Pipeline

Embeddings turn text into numbers that computers can compare semantically. The embedding pipeline converts each text chunk into a vector (a list of numbers) that represents its meaning.

### 9.1 Text Preprocessing

Before embedding, text is cleaned up:

- **Normalization**: Lowercasing, removing extra whitespace
- **Token splitting**: Breaking text into tokens the embedding model understands
- **Special character handling**: Removing or escaping characters that confuse the model
- **Length truncation**: Trimming text that exceeds the model's maximum input length

### 9.2 Embedding Model Selection

The system uses a vector embedding model (configured externally, typically a model like OpenAI's text-embedding-3-small or a similar service). The model:

- Takes text as input
- Outputs a fixed-size vector (e.g., 1536 dimensions)
- Captures semantic meaning (similar texts produce similar vectors)

### 9.3 Vector Generation

Each chunk is passed through the embedding model to produce a vector. The system:

- Processes chunks in batches for efficiency
- Handles rate limits and retries
- Stores the resulting vector alongside the chunk text

### 9.4 Dimension Handling

Different embedding models produce vectors of different sizes. The system tracks the dimension of each vector and ensures compatibility when searching. If the embedding model changes, previously indexed chunks may need re-embedding.

### 9.5 Quality Validation

After embedding, basic quality checks are performed:

- **Null vector check**: Ensures the vector is not all zeros
- **Dimension check**: Verifies the vector has the expected number of dimensions
- **Magnitude check**: Ensures the vector has a reasonable magnitude (not degenerate)

---

## 10. RAG Indexing

RAG (Retrieval Augmented Generation) is how Mike makes documents searchable by AI. Once chunks are embedded, they need to be stored in a searchable index.

### 10.1 Collection Creation

For each scope (personal, project, workspace, or document), the system creates a RAG collection. The collection is the container for all the vectors in that scope.

### 10.2 Vector Storage

Vectors are stored in an external RAG service (configured via environment variables). The service provides:

- **Vector similarity search**: Find the most similar chunks to a query
- **Metadata filtering**: Filter results by document type, date, tags, etc.
- **Hybrid search**: Combine vector similarity with keyword matching

### 10.3 Metadata Indexing

Alongside each vector, the system stores:

- Chunk text
- Document ID
- Document type
- Source information (which document, which page, which section)
- Timestamps
- Any user-applied tags

### 10.4 Citation Mapping

Citations found in the chunk are stored as searchable metadata. This enables queries like "find all text citing (2022) 7 SCC 508."

### 10.5 Update Handling

When a document is edited or a new version is uploaded:

1. Old chunks for that document are removed from the index
2. New chunks are generated from the updated content
3. New vectors are computed and indexed
4. The index is updated atomically (all changes applied at once)

This ensures the search index always reflects the current state of the document.

---

## 11. Document Versioning

Legal documents go through many revisions. Mike tracks every version so you can always go back.

### 11.1 Version Creation

Every time a significant edit is made, a new version is created. Versions are stored in the **document_versions** table (`backend/src/db/schema.ts:482`) with:

- Version ID (UUID)
- Document ID (parent document)
- Storage path in R2
- Source (user upload, AI edit, assistant edit)
- Version number (sequential)
- Display name
- Creation timestamp

### 11.2 Diff Generation

When comparing two versions, the system generates a diff:

1. Extracts text from both versions
2. Runs a diff algorithm (typically patience diff or Myers diff)
3. Identifies insertions, deletions, and modifications
4. Groups changes by paragraph

### 11.3 Version History

Users can view the complete version history of any document:

- List of all versions in chronological order
- Who created each version
- When it was created
- What changed (summary)

### 11.4 Rollback Capability

If a bad edit is made, users can roll back to a previous version:

1. Select the version to restore
2. The system copies that version's content to a new version
3. The document is now at the restored state
4. The version history is preserved (no versions are deleted)

### 11.5 Version Comparison

Side-by-side comparison lets users:

- See both versions simultaneously
- Highlight differences
- Accept or reject individual changes
- Generate a redline document showing all changes

---

## 12. Document Generation

Mike can create entirely new documents from scratch using AI. This is one of its most powerful features.

### 12.1 Template Selection

The user can choose from:

- **Blank document**: Start from scratch
- **Existing template**: A DOCX template with placeholders
- **Custom instructions**: Describe what you want in plain language

### 12.2 Content Generation (LLM)

For custom instructions, the system:

1. Sends the user's instructions to a large language model (LLM)
2. The LLM generates the document content in a structured format
3. The system validates the output structure
4. If the output is malformed, it retries with error feedback

The LLM is prompted to generate content that follows the artifact schema (`backend/src/lib/sandbox/artifactSchema.ts`), which defines:

- Sections with headings and content
- Tables with headers and rows
- Charts with labels and data series
- Defined terms and definitions
- Signature blocks
- And more

### 12.3 DOCX Assembly (docx library)

Once the content is generated, the system assembles a DOCX file:

1. Creates a new DOCX using the `docx` library (or template engine for template-based documents)
2. Adds sections, paragraphs, and runs
3. Applies formatting (fonts, colors, spacing)
4. Inserts tables with proper styling
5. Adds headers and footers
6. Sets page margins and orientation

### 12.4 Style Application

The system applies consistent styling:

- **Classic theme**: Traditional legal document look (Times New Roman, formal spacing)
- **Modern theme**: Clean, contemporary styling
- **Legal theme**: Court-filing-ready formatting
- **Minimal theme**: Simple, distraction-free layout

### 12.5 Metadata Injection

Generated documents include metadata:

- Title
- Author
- Creation date
- Document type
- Jurisdiction
- Confidentiality notice (if applicable)

### 12.6 Quality Check

Before delivering the document, a quality check is run:

- **ZIP integrity**: Verifies the DOCX is a valid ZIP archive
- **XML validation**: Checks the internal XML is well-formed
- **Content verification**: Ensures the document is not empty
- **Size check**: Verifies the file is a reasonable size

---

## 13. Document Export

Mike supports exporting documents in multiple formats, each suited for different use cases.

### 13.1 DOCX Export

The primary export format for legal documents. Two approaches:

**Direct DOCX generation** (using the `docx` library):
- Creates a new DOCX from structured content
- Full control over formatting
- Used for newly generated documents

**SuperDoc-based export**:
- Opens the document in a SuperDoc session
- Applies edits as tracked changes
- Exports the modified document
- Used for editing existing documents

The system also supports exporting with tracked changes preserved (`backend/src/lib/docxTrackChangesExport.ts`). This parses HTML content from the TipTap editor and converts `<span data-track-insert>` and `<span data-track-delete>` elements into proper OOXML `<w:ins>` and `<w:del>` elements.

### 13.2 PDF Export

PDFs are generated for:

- Court filings (many courts require PDF)
- Sharing with parties who may not have Word
- Archival purposes

The system converts DOCX to PDF using either:

- **LibreOffice**: Headless conversion via the sandbox service
- **Puppeteer**: HTML-to-PDF rendering for simpler documents

### 13.3 PPTX Export

For creating legal presentations:

- Uses `pptxgenjs` library
- Generates slides with proper layouts
- Supports charts, timelines, and comparison slides
- Includes firm branding and logos

The sandbox service (`sandbox/skills/pptx/`) provides Python scripts for advanced PPTX operations like thumbnail generation and slide manipulation.

### 13.4 XLSX Export

For spreadsheets and data tables:

- Uses the `xlsx` library
- Generates properly formatted spreadsheets
- Supports formulas, formatting, and multiple sheets
- Useful for financial schedules, damage calculations, and data analysis

The sandbox service (`sandbox/skills/xlsx/`) provides Python scripts for advanced operations like recalculation and validation.

### 13.5 HTML Export

For web display and sharing:

- Converts document content to clean HTML
- Preserves structure (headings, lists, tables)
- Includes inline styling for standalone viewing
- Used for email bodies and web previews

---

## 14. Document Comparison

Comparing two versions of a document is a core legal task. Mike's comparison engine goes beyond simple text diff.

### 14.1 Text Extraction

Both documents are parsed to extract their text content:

- DOCX files are parsed via SuperDoc or mammoth
- PDF files are parsed via pdfjs-dist
- Plain text files are read directly

### 14.2 Diff Algorithm

The system uses **fast-diff** for efficient text comparison:

1. Tokenizes both texts
2. Computes the longest common subsequence
3. Identifies insertions (text in document B but not A)
4. Identifies deletions (text in document A but not B)
5. Identifies modifications (text that changed)

### 14.3 Change Categorization

Each change is categorized:

- **Insertion**: New text was added
- **Deletion**: Existing text was removed
- **Modification**: Text was altered (a deletion followed by an insertion at the same position)
- **Hidden edit**: A change that was made without track changes enabled

### 14.4 Visual Diff Generation

The comparison produces a visual representation:

- Insertions highlighted in green
- Deletions highlighted in red (with strikethrough)
- Modifications shown with both the old and new text
- Context around each change for understanding

This is handled by `backend/src/lib/docxCompare.ts`.

### 14.5 Summary Generation

After comparing, the system generates a summary:

- Total number of changes
- Breakdown by type (insertions, deletions, modifications)
- Word count changes
- Key differences in plain language

---

## 15. Track Changes Pipeline

Track changes are the lifeblood of legal document collaboration. Mike's track changes system works both inbound (importing from Word) and outbound (exporting to Word).

### 15.1 Change Detection

When a document with track changes is opened:

1. The system parses `word/document.xml`
2. It finds all `<w:ins>` elements (insertions) and `<w:del>` elements (deletions)
3. For each change, it extracts:
   - The `w:id` attribute (unique change identifier)
   - The `w:author` attribute (who made the change)
   - The `w:date` attribute (when the change was made)
   - The text content

### 15.2 Change Formatting

Changes are formatted for display in the editor:

- Insertions appear with green highlighting
- Deletions appear with red strikethrough
- The author's name and timestamp are shown as tooltips
- Changes can be filtered by author

### 15.3 Change Approval

Users can accept or reject individual changes:

- **Accept**: The inserted text becomes permanent, or the deleted text stays removed
- **Reject**: The inserted text is removed, or the deleted text is restored
- **Accept all**: All changes are accepted at once
- **Reject all**: All changes are rejected at once

When a change is resolved, it is recorded in the **document_edits** table (`backend/src/db/schema.ts:501`) with:

- Change ID
- Deleted text and inserted text
- Context before and after
- Status (pending, accepted, rejected)
- Resolution timestamp

### 15.4 Change Export

When exporting to DOCX, changes are preserved as proper OOXML revision marks:

1. The editor HTML is parsed
2. `<span data-track-insert>` elements become `<w:ins>` elements
3. `<span data-track-delete>` elements become `<w:del>` elements
4. Author and timestamp attributes are added
5. The resulting DOCX can be opened in Microsoft Word with track changes visible

This is handled by `backend/src/lib/docxTrackChangesExport.ts`.

### 15.5 Change Import

When importing a DOCX with track changes:

1. The DOCX is parsed and all `<w:ins>` and `<w:del>` elements are found
2. Changes are converted to HTML with data attributes
3. The HTML is rendered in the editor with track changes visible
4. Users can then accept or reject changes within Mike

This is handled by `backend/src/lib/docxTrackChangesImport.ts`.

---

## 16. Template System

Templates let users create documents quickly by filling in pre-defined fields. Mike's template system is sophisticated.

### 16.1 Template Storage

Templates are stored as DOCX files in the system. They contain placeholder text that will be replaced with actual values. The system recognizes two types of placeholders:

- **Mustache-style**: `{{field_name}}` — double curly braces around the field name
- **Bracket-style**: `[ENTER DATE HERE]` — square brackets around instruction text

### 16.2 Placeholder Detection

The template analyzer (`backend/src/lib/docxTemplateAnalyzer.ts`) scans the template and identifies:

- All unique placeholder keys
- How many times each placeholder appears
- The type of each placeholder (text, date, number, email, phone, address)
- Whether the placeholder appears in the main document, headers, or footers
- Occurrences that span across XML elements

### 16.3 Variable Filling

The template engine (`backend/src/docx/templateEngine.ts`) fills in the placeholders:

1. Loads the DOCX template using PizZip
2. Processes each word content part (document.xml, headers, footers)
3. Replaces `{{key}}` patterns with the provided values
4. Optionally applies tracked changes (showing what was replaced)
5. Generates the final DOCX

### 16.4 Conditional Sections

Templates can include conditional sections that appear or disappear based on input values:

```
{{#if include_indemnification}}
INDEMNIFICATION: The Company shall indemnify...
{{/if}}
```

The engine evaluates these conditions and includes or excludes the content accordingly.

### 16.5 Loop Sections

Templates can repeat sections for lists of items:

```
{{#each parties}}
Party: {{name}}
Role: {{role}}
{{/each}}
```

The engine iterates over the provided array and generates the section for each item.

### 16.6 Output Generation

The final output is a fully formatted DOCX with:

- All placeholders replaced
- Conditional sections included/excluded
- Loop sections repeated
- Original formatting preserved
- Tracked changes showing what was filled in (optional)

---

## 17. Document Security

Legal documents contain highly sensitive information. Mike's security measures are designed to protect that information.

### 17.1 Access Control

Every document is associated with a user and optionally a project. Access is controlled through:

- **User ownership**: Only the owner can access their documents by default
- **Project sharing**: Documents in shared projects are accessible to project members
- **Workspace access**: Documents in workspaces are accessible to workspace members
- **Role-based permissions**: Admin, editor, and viewer roles with different capabilities

### 17.2 Encryption at Rest

All stored documents are encrypted:

- Files in Cloudflare R2 are encrypted at rest by default
- Database records are in an encrypted database
- API keys are encrypted with AES-256 before storage

### 17.3 Encryption in Transit

All communication between the client and server uses TLS encryption:

- HTTPS for all API calls
- WebSocket connections are encrypted
- File uploads and downloads use encrypted channels

### 17.4 Watermarking

Documents can be watermarked to deter unauthorized sharing:

- User name and email as a watermark
- Date and time of access
- "Confidential" or "Privileged" markings

### 17.5 Audit Logging

Every significant action on a document is logged:

- Document created
- Document viewed
- Document edited
- Document exported
- Document shared
- Version created
- Changes accepted/rejected

These logs are stored in the **document_activity** table (`backend/src/db/schema.ts:594`).

### 17.6 Metadata Scrubbing

Before sharing or exporting documents, sensitive metadata can be removed (`backend/src/lib/docxMetadataScrub.ts`):

- **Track changes**: Accept all, reject all, or keep them
- **Comments**: Remove all comments
- **Author information**: Strip author name and revision history
- **Hidden text**: Remove hidden text and field codes
- **Revision history**: Clear the revision log

This ensures that when a document is shared externally, it does not leak internal information.

---

## 18. Performance Optimization

Processing legal documents can be slow. Mike uses several techniques to keep things fast.

### 18.1 Lazy Loading

Documents are not fully processed until they need to be:

- Metadata is extracted on upload, but full text parsing may be deferred
- Embeddings are computed only when the document is first searched
- PDF rendering is done on demand, not eagerly

### 18.2 Caching

Frequently accessed data is cached:

- Parsed document content is cached after the first parse
- Embeddings are cached to avoid recomputation
- Template analysis results are cached
- RAG search results are cached for repeated queries

### 18.3 Parallel Processing

Multiple operations run simultaneously:

- Multiple chunks are embedded in parallel
- Multiple documents in a batch are processed concurrently
- Upload and parsing can overlap
- Export and upload can overlap

### 18.4 Background Processing

Heavy operations run in the background:

- OCR processing happens asynchronously
- RAG indexing happens after the document is saved
- PDF generation happens in a background worker
- Large batch operations are queued

### 18.5 Progressive Rendering

Users see results as they become available:

- Document text appears as soon as parsing is done
- Search results appear as chunks are found
- Exports start downloading before the full file is ready
- Version diffs appear incrementally

---

## 19. Error Handling

Things go wrong. Files get corrupted. Networks fail. Mike handles these situations gracefully.

### 19.1 Upload Failures

- **File too large**: Clear error message with size limit
- **Invalid file type**: Suggests correct file types
- **Network interruption**: Retry with resume support
- **Storage full**: Notification with instructions to free space

### 19.2 Parse Errors

- **Corrupted DOCX**: Falls back to mammoth.js for basic text extraction
- **Unsupported PDF features**: Extracts what it can, flags what it could not
- **Malformed XML**: Tries to repair, falls back to plain text
- **Password-protected files**: Prompts for password or skips with notification

### 19.3 OCR Failures

- **Low quality image**: Reports low confidence score
- **Unsupported language**: Attempts best-effort recognition
- **Image too large**: Resizes before OCR
- **OCR engine timeout**: Retries with simpler settings

### 19.4 Generation Errors

- **LLM output malformed**: Retries with error feedback to the model
- **Template missing fields**: Generates with available fields, flags missing ones
- **DOCX assembly failure**: Falls back to simpler generation method
- **Content too long**: Splits into multiple documents

### 19.5 Export Failures

- **PDF conversion fails**: Falls back to alternative conversion method
- **PPTX generation fails**: Offers DOCX export instead
- **XLSX generation fails**: Offers CSV export instead
- **Network timeout during download**: Allows resume from last checkpoint

### 19.6 Recovery Strategies

The system implements several recovery mechanisms:

- **Automatic retry**: Transient errors are retried up to 3 times
- **Fallback methods**: If the primary method fails, alternatives are tried
- **Partial results**: If processing fails midway, partial results are saved
- **User notification**: Users are informed of failures with actionable messages
- **Graceful degradation**: The system continues to work even if some features fail

---

## 20. Complete Pipeline Diagram

Here is the complete end-to-end pipeline showing every step:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        COMPLETE DOCUMENT PIPELINE                            │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐                                                             │
│  │   UPLOAD    │                                                             │
│  └──────┬──────┘                                                             │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────┐    ┌──────────────────┐                                │
│  │ Multer receives  │───▶│ File validation   │                               │
│  │ the file         │    │ (type + magic)    │                               │
│  └──────────────────┘    └────────┬─────────┘                                │
│                                   │                                          │
│                                   ▼                                          │
│                          ┌──────────────────┐                                │
│                          │ Size limit check  │                               │
│                          └────────┬─────────┘                                │
│                                   │                                          │
│                                   ▼                                          │
│                          ┌──────────────────┐                                │
│                          │ Virus scanning    │                               │
│                          └────────┬─────────┘                                │
│                                   │                                          │
│                                   ▼                                          │
│                          ┌──────────────────┐                                │
│                          │ Store in R2       │                               │
│                          └────────┬─────────┘                                │
│                                   │                                          │
│                                   ▼                                          │
│                          ┌──────────────────┐                                │
│                          │ DB record created │                               │
│                          │ (status:processing)                               │
│                          └────────┬─────────┘                                │
│                                   │                                          │
│         ┌─────────────────────────┼─────────────────────────┐                │
│         │                         │                         │                │
│         ▼                         ▼                         ▼                │
│  ┌─────────────┐          ┌─────────────┐          ┌─────────────┐          │
│  │   PDF/IMG   │          │    DOCX     │          │ Plain text  │          │
│  └──────┬──────┘          └──────┬──────┘          └──────┬──────┘          │
│         │                        │                        │                 │
│         ▼                        ▼                        │                 │
│  ┌─────────────┐          ┌─────────────┐                 │                 │
│  │   OCR +     │          │  SuperDoc   │                 │                 │
│  │  pdfjs-dist │          │  SDK parse  │                 │                 │
│  └──────┬──────┘          └──────┬──────┘                 │                 │
│         │                        │                        │                 │
│         │                        ▼                        │                 │
│         │                 ┌─────────────┐                 │                 │
│         │                 │ Extract     │                 │                 │
│         │                 │ - text      │                 │                 │
│         │                 │ - styles    │                 │                 │
│         │                 │ - track chg │                 │                 │
│         │                 │ - comments  │                 │                 │
│         │                 │ - metadata  │                 │                 │
│         │                 └──────┬──────┘                 │                 │
│         │                        │                        │                 │
│         └────────────┬───────────┴────────────┬───────────┘                 │
│                      │                        │                              │
│                      ▼                        ▼                              │
│              ┌──────────────┐        ┌──────────────┐                       │
│              │  CLASSIFY    │        │  EXTRACT     │                       │
│              │  (type, area │        │  metadata    │                       │
│              │  jurisdiction)│       │  dates, $    │                       │
│              └──────┬───────┘        │  citations   │                       │
│                     │                └──────┬───────┘                       │
│                     │                       │                               │
│                     └───────────┬───────────┘                               │
│                                 │                                           │
│                                 ▼                                           │
│                        ┌──────────────┐                                     │
│                        │   CHUNK      │                                     │
│                        │  (paragraph/ │                                     │
│                        │   section)   │                                     │
│                        └──────┬───────┘                                     │
│                               │                                             │
│                               ▼                                             │
│                        ┌──────────────┐                                     │
│                        │   EMBED      │                                     │
│                        │  (vectors)   │                                     │
│                        └──────┬───────┘                                     │
│                               │                                             │
│                               ▼                                             │
│                        ┌──────────────┐                                     │
│                        │   INDEX      │                                     │
│                        │  (RAG store) │                                     │
│                        └──────┬───────┘                                     │
│                               │                                             │
│                               ▼                                             │
│                        ┌──────────────┐                                     │
│                        │   STORE      │                                     │
│                        │  (DB + R2)   │                                     │
│                        └──────┬───────┘                                     │
│                               │                                             │
│         ┌─────────────────────┼─────────────────────┐                       │
│         │                     │                     │                       │
│         ▼                     ▼                     ▼                       │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐                 │
│  │   EDIT      │      │  GENERATE   │      │  COMPARE    │                 │
│  │ - track chg │      │ - template  │      │ - diff      │                 │
│  │ - comments  │      │ - AI draft  │      │ - changes   │                 │
│  │ - AI edit   │      │ - DOCX asm  │      │ - summary   │                 │
│  └──────┬──────┘      └──────┬──────┘      └──────┬──────┘                 │
│         │                     │                     │                       │
│         └─────────────────────┼─────────────────────┘                       │
│                               │                                             │
│                               ▼                                             │
│                        ┌──────────────┐                                     │
│                        │   VERSION    │                                     │
│                        │ - snapshot   │                                     │
│                        │ - diff       │                                     │
│                        │ - rollback   │                                     │
│                        └──────┬───────┘                                     │
│                               │                                             │
│                               ▼                                             │
│                        ┌──────────────┐                                     │
│                        │   EXPORT     │                                     │
│                        │ - DOCX       │                                     │
│                        │ - PDF        │                                     │
│                        │ - PPTX       │                                     │
│                        │ - XLSX       │                                     │
│                        │ - HTML       │                                     │
│                        └──────┬───────┘                                     │
│                               │                                             │
│                               ▼                                             │
│                        ┌──────────────┐                                     │
│                        │   DELIVER    │                                     │
│                        │ (to user)    │                                     │
│                        └──────────────┘                                     │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 21. Key Files Reference

The following table lists all the key files involved in the document pipeline, along with their purpose.

| File | Purpose |
|---|---|
| `backend/src/lib/docxTrackedChanges.ts` | Apply tracked edits to DOCX files, resolve changes by ID |
| `backend/src/lib/docxTrackChangesExport.ts` | Export HTML with tracked changes to DOCX format |
| `backend/src/lib/docxTrackChangesImport.ts` | Import DOCX tracked changes to HTML for editor display |
| `backend/src/lib/docxCompare.ts` | Compare two DOCX documents and generate detailed diffs |
| `backend/src/lib/docxComments.ts` | Extract and insert comments in DOCX files |
| `backend/src/lib/docxMetadataScrub.ts` | Remove sensitive metadata (track changes, comments, author info) |
| `backend/src/lib/docxTemplateAnalyzer.ts` | Analyze DOCX templates for placeholders and structure |
| `backend/src/docx/templateEngine.ts` | Fill DOCX templates with variable data |
| `backend/src/docx/tableWriter.ts` | Generate tables within DOCX documents |
| `backend/src/docx/pleadingsMatrixWriter.ts` | Generate pleadings matrix documents |
| `backend/src/lib/legal/document-classifier.ts` | Classify documents by type (contract, brief, memo, etc.) |
| `backend/src/lib/legal/documentSummary.ts` | Generate AI-powered document summaries |
| `backend/src/lib/legal/dynamicSummary.ts` | Generate dynamic case-type-specific summaries |
| `backend/src/lib/legal/consistencyCheck.ts` | Check documents for internal consistency issues |
| `backend/src/lib/legal/intelligentEdit.ts` | AI-powered document editing with legal reasoning |
| `backend/src/lib/legal/legalCaseExtractor.ts` | Extract legal cases from web search results |
| `backend/src/lib/legal/citationPatterns.ts` | Regex patterns for Indian legal citations |
| `backend/src/lib/legal/disclosures.ts` | AI disclosure and disclaimer management |
| `backend/src/lib/sandbox/service.ts` | Sandbox service for executing document generation scripts |
| `backend/src/lib/sandbox/orchestrator-client.ts` | Client for sandbox container orchestration |
| `backend/src/lib/sandbox/artifactGenerators.ts` | Generate DOCX, PPTX, XLSX artifacts |
| `backend/src/lib/sandbox/artifactSchema.ts` | Schema definitions for document artifacts |
| `backend/src/lib/superdoc/superdocService.ts` | SuperDoc SDK integration for DOCX editing |
| `backend/src/lib/superdoc/superdocTools.ts` | SuperDoc tool dispatch and management |
| `backend/src/lib/superdoc/sessionManager.ts` | SuperDoc session lifecycle management |
| `packages/shared-types/src/documents.ts` | Shared document type definitions |
| `backend/src/db/schema.ts` | Database schema including all document tables |
| `sandbox/skills/docx/` | DOCX generation and manipulation scripts |
| `sandbox/skills/pptx/` | PPTX generation and manipulation scripts |
| `sandbox/skills/xlsx/` | XLSX generation and manipulation scripts |

---

## Database Schema Overview

The document-related database tables (defined in `backend/src/db/schema.ts`) include:

| Table | Purpose |
|---|---|
| `documents` | Main document records (ID, project, user, filename, status, lifecycle) |
| `document_versions` | Version snapshots (storage path, source, version number) |
| `document_edits` | Tracked change records (change ID, deleted/inserted text, status) |
| `document_comments` | Comments with anchoring (body, anchor text, resolution status) |
| `document_placeholder_values` | Template placeholder values per document |
| `document_context_files` | Links between documents and their context files |
| `document_activity` | Audit log of document actions |
| `rag_sources` | RAG indexing status per document |
| `rag_chunks` | Individual text chunks for RAG search |
| `document_change_requests` | Change request workflow (pending, approved, rejected) |

---

*This document covers the complete document processing pipeline in Mike. Every legal document — whether uploaded, generated, or edited — follows some subset of these steps. The pipeline is designed to preserve the integrity and formatting that legal work demands while leveraging AI to make the process faster and more accurate.*
