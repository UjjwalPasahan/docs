# UI Behavior Guide for Mike Legal AI Platform

## 1. What This Document Is

This document is your complete visual handbook for the Mike legal AI platform. Think of it as a detailed map showing you exactly what happens on your screen when you click, type, or interact with any part of the application. If you’ve ever wondered, “What should appear when I press this button?” or “Why does this screen look like this?”—this guide has the answers. It covers everything from the moment you log in to the final export of a document, explaining every animation, loading spinner, error message, and visual cue in plain, straightforward language. Whether you’re a new user, a support agent, or a developer trying to understand the front-end behavior, this guide will walk you through each interaction step by step.

## 2. The Complete User Journey

A typical session in Mike follows this path:

1. **Login**: You open the app and see the login screen. After entering your credentials, you’re taken to the dashboard.
2. **Dashboard**: The dashboard shows your recent projects, documents, and quick actions. You can see summaries of ongoing work and jump into any project.
3. **Open Assistant**: You click the assistant icon or “New Chat” button. A chat panel opens, ready for your first message.
4. **Send Message**: You type a question or instruction in the chat input and press Enter or click the send button. Your message appears in the conversation thread.
5. **See Response**: The AI starts thinking. You see a “Thinking…” indicator, then a planning card if the request is complex. Tool calls appear as cards, citations stream in, and the response text appears word by word. Finally, artifact cards (like documents or tables) show up at the end.
6. **Open Document**: If the AI generated a document, you click the “Open” button on the artifact card. The document opens in the built‑in editor.
7. **Edit**: You make changes directly in the editor. Track changes are on by default, so you can see suggestions and accept or reject them.
8. **Export**: When you’re satisfied, you click the export button and choose a format (DOCX, PDF, or plain text). The file downloads to your computer.

Each step has its own set of visual cues, loading states, and feedback mechanisms, all detailed in the following sections.

## 3. Chat Interaction Flow

When you send a message in the chat, the following sequence of visual events occurs:

1. **User types in ChatInput**: The chat input box is at the bottom of the chat panel. As you type, the placeholder text disappears and your text appears in the input field. The send button (usually an arrow icon) becomes active once you have text.
2. **Send button activates**: The send button changes color or opacity to indicate it’s clickable. You can also press Enter to send.
3. **Message appears in conversation**: Your message bubbles up into the chat thread, aligned to the right if it’s your message, with a timestamp and your avatar.
4. **“Thinking” animation appears**: Immediately after sending, a small animation (three pulsing dots) appears on the left side, indicating the AI is processing.
5. **“Planning” card shows (if complex)**: For requests that require multiple steps, a card labeled “Planning your request…” appears. It may list the steps the AI plans to take (e.g., “Search for case law”, “Analyze contract”, “Generate summary”).
6. **Tool calls appear as cards**: Each tool the AI uses (like a document search, web search, or compliance check) is displayed as a card. The card shows the tool name, a spinner while it’s running, and then the result when it completes.
7. **Citations stream in**: As the AI finds relevant sources, citation links appear in the response text. Hovering over a citation shows a tooltip with the source title and snippet.
8. **Response text streams word‑by‑word**: The AI’s answer appears gradually, as if someone is typing it in real time. This gives you immediate feedback that the system is working.
9. **Artifact cards appear**: At the end of the response, any generated artifacts (documents, tables, compliance reports, etc.) appear as expandable cards. You can click to open, download, or preview them.
10. **Completion indicator**: A small checkmark or “Done” label appears, and the chat input re‑activates for your next message.

## 4. AI Thinking Indicators

The app uses several visual cues to show you that the AI is working:

- **Typing dots animation**: Three dots that pulse in sequence, usually below the AI’s avatar. This appears the moment you send a message and stays until the first text starts streaming.
- **“Thinking…” label**: A text label that may appear next to the dots, especially for longer‑running requests.
- **“Planning your request…” card**: A card that outlines the steps the AI will take. It may show a list of actions with checkmarks as each step begins or completes.
- **Tool call spinners**: Each tool card has a circular spinner that rotates while the tool is executing. When the tool finishes, the spinner is replaced by a result icon (a checkmark for success, an exclamation for error).
- **Tool result cards**: After a tool runs, its result is displayed in a card. For example, a web search tool shows a list of results; a document tool shows a preview or diff.
- **Progress bars for long operations**: For tasks that take a while (like generating a long document), a progress bar fills from left to right, giving you an idea of how much time is left.

All these indicators are designed to keep you informed and prevent you from thinking the app has frozen.

## 5. Tool Call Renderers

The assistant can call many different tools, each with its own visual renderer. Here are the main categories:

### Document Tools
- **Preview cards**: When the AI creates or retrieves a document, a card appears with a thumbnail preview, title, and an “Open” button.
- **Diff views**: If the AI suggests changes to an existing document, a side‑by‑side diff view shows the original text on the left and the proposed changes on the right, with additions highlighted in green and deletions in red.
- **Generation progress**: For large documents, a progress bar shows how much of the document has been generated.

### Legal Tools
- **Analysis tables**: Results from legal research or analysis are displayed in neat tables with columns for key points, citations, and relevance scores.
- **Compliance cards**: Compliance checks produce a card with a pass/fail indicator, a list of issues found, and links to relevant regulations.

### Search Tools
- **Result lists**: Web searches and internal searches return a list of results, each with a title, snippet, and link.
- **Source cards**: For each source, a card shows the document type, a preview, and options to open or cite.

### Generation Tools
- **Progress indicators**: When generating a document or summary, a spinner or progress bar shows the status.
- **Preview**: A small preview of the generated content appears as soon as it’s ready, even if the full generation isn’t complete.

### Email Tools
- **Draft preview**: Before sending an email, the AI shows a draft with subject, body, and recipient fields. You can edit it directly.
- **Send confirmation**: After you approve, a confirmation card appears with a “Sent” checkmark and a link to view the email.

### Workspace Tools
- **Project/workspace cards**: Creating or switching projects shows a card with the project name, description, and recent activity.

### Sandbox Tools
- **Code execution results**: If the AI runs code in a sandbox, the output is displayed in a code block with syntax highlighting and an option to copy.

## 6. Artifact Display

Artifacts are the final outputs of an AI conversation. They appear in different formats:

- **Citations**: Inline links in the response text. Hovering over a citation shows a tooltip with the full source details. Clicking opens the source in a side panel or new tab.
- **Documents**: A card with a title, preview image, and buttons to open, download, or share. Opening the document loads it in the built‑in editor.
- **Tables**: Formatted tables with headers, sortable columns, and the ability to export to CSV or Excel.
- **Images**: Inline images (like charts or diagrams) appear directly in the chat. Clicking an image opens a larger view.
- **Code blocks**: Code snippets are displayed in a box with syntax highlighting and a copy button. Long code blocks have a “Show more” toggle.

## 7. Document Editor Interactions

When you open a document from a chat or from the documents page, the editor loads with these steps:

1. **Opening a document**: You click the “Open” button on an artifact card or a document list item. A new tab or panel opens, showing a loading spinner.
2. **Loading SuperDoc**: The SuperDoc editor initializes. You see a skeleton layout (gray placeholders) until the content loads.
3. **Editing with track changes**: By default, track changes are on. Any text you add or remove is marked with colored highlights and can be accepted or rejected later.
4. **Suggestion cards**: The AI may suggest changes while you edit. These appear as small cards near the relevant text, with “Accept” and “Reject” buttons.
5. **Diff view**: You can toggle a diff view that shows the original document side‑by‑side with your edited version.
6. **Export options**: The editor toolbar includes an export button that opens a dropdown with format choices (DOCX, PDF, plain text). Selecting one starts the download.

## 8. Agent Background Tasks

Some AI tasks run in the background, especially long‑running analyses. The UI for these tasks includes:

- **Task submission**: You click “Run in background” or the AI automatically decides to run a task in the background. A small notification appears saying “Task started.”
- **Progress panel**: A panel slides out from the right, showing a list of active background tasks. Each task has a progress bar, status text, and an option to cancel.
- **Plan panel**: For complex tasks, a plan panel shows the steps the agent is taking, with checkmarks for completed steps and a spinner for the current step.
- **Completion summary**: When a background task finishes, a card appears in the chat with a summary of the results and a button to open the full output.
- **Error handling**: If a task fails, an error card appears with a description of what went wrong and a retry button.
- **Clarification prompts**: Sometimes the agent needs more information. A clarification card appears with questions and input fields for you to provide answers.

## 9. Workflow Builder Interactions

The workflow builder is a visual tool for creating automated processes. Its interactions include:

- **Drag and drop nodes**: You drag tool nodes from a palette onto a canvas. Each node represents a step in the workflow.
- **Connect nodes**: You draw lines between nodes to define the order of execution. Lines snap to connection points on the nodes.
- **Configure nodes**: Clicking a node opens a configuration panel where you set parameters (e.g., which document to analyze, what search terms to use).
- **Run workflow**: A “Run” button starts the workflow. The canvas shows the progress, with each node lighting up as it executes.
- **View execution history**: A history panel lists past workflow runs, with timestamps, status, and links to view detailed logs.

## 10. Source Canvas

The source canvas is where you view documents and web pages referenced by the AI. Its features include:

- **Opening sources**: Clicking a citation or source card opens the source in a side panel or a new tab, depending on your settings.
- **PDF viewer**: PDFs are displayed in an embedded viewer with page navigation, zoom, and search.
- **HTML viewer**: Web pages are rendered in an iframe with basic navigation controls.
- **Drive file viewer**: Files from Google Drive or OneDrive open in their respective viewers embedded in the app.
- **Web page viewer**: External web pages open in a new browser tab.

## 11. Notification System

Notifications keep you informed about events in the app:

- **Toast notifications**: Small pop‑up messages that appear at the bottom or top of the screen. They auto‑dismiss after a few seconds but can be clicked for more details.
- **In‑app notifications**: A bell icon in the header shows a badge with the number of unread notifications. Clicking it opens a dropdown list of recent notifications.
- **Email notifications**: For important events (like a background task completing), the app can send an email summary. This is optional and can be turned off in settings.
- **Agent completion notifications**: When a background agent finishes, a toast notification appears and a card is added to the chat thread.

## 12. Loading States

Every part of the app has a loading state to show that data is on its way:

- **Page loading**: When you navigate to a new page, a skeleton screen (gray placeholders) appears until the page content loads.
- **Data fetching**: Tables and lists show a loading spinner above the rows while data is being fetched.
- **Tool execution**: Each tool card has its own spinner while the tool runs.
- **Document generation**: A progress bar shows how much of the document has been generated.
- **Export processing**: When exporting a document, a modal with a progress bar appears, and the download starts automatically when the export is ready.

## 13. Error States

Errors are displayed in clear, non‑technical language:

- **Network errors**: A banner at the top of the screen says “Connection lost. Trying to reconnect…” with a retry button.
- **Tool failures**: If a tool fails, its card shows a red exclamation icon and a message like “Search timed out. Please try again.” A retry button is provided.
- **Permission denied**: If you don’t have access to a resource, a modal appears saying “You don’t have permission to view this document.” with a link to request access.
- **Rate limiting**: If you send too many messages too quickly, a small warning appears: “Please wait a moment before sending another message.”
- **Validation errors**: When you fill out a form, any validation errors appear in red text below the relevant field.

## 14. Empty States

When there’s no data to show, the app displays friendly empty states:

- **No projects**: A message like “You don’t have any projects yet. Create your first project to get started.” with a “Create Project” button.
- **No documents**: “No documents here yet. Ask the assistant to generate one, or upload a file.” with an upload button.
- **No conversations**: “Start a new conversation with the assistant.” with a “New Chat” button.
- **No search results**: “No results found. Try different keywords or check your filters.”
- **No workflow runs**: “This workflow hasn’t been run yet. Click ‘Run’ to start it.”

## 15. Responsive Behavior

The app adapts to different screen sizes:

- **Desktop layout**: Full sidebar with navigation links, main content area, and optional right panel for sources or details.
- **Tablet layout**: Sidebar collapses to icons only, panels stack vertically, and modals become full‑screen.
- **Mobile layout**: Sidebar becomes a hamburger menu, chat takes full width, and editors are simplified for touch.
- **Sidebar behavior**: The sidebar can be toggled open/closed. On small screens, it overlays the content.
- **Panel resizing**: Some panels (like the source canvas) can be resized by dragging their edges.

## 16. Keyboard Shortcuts

Keyboard shortcuts speed up common actions:

- `Ctrl + Enter`: Send a message in the chat.
- `Ctrl + N`: Start a new conversation.
- `Ctrl + S`: Save the current document.
- `Ctrl + E`: Export the current document.
- `Ctrl + /`: Open the keyboard shortcut help modal.
- `Escape`: Close any open modal or panel.
- `Tab`: Move focus to the next interactive element.
- `Shift + Tab`: Move focus to the previous element.
- `Ctrl + Z`: Undo the last edit in the document.
- `Ctrl + Y`: Redo the last undone edit.

## 17. Animation and Transitions

Smooth animations make the app feel responsive:

- **Page transitions**: Pages fade in and slide slightly from the right when you navigate.
- **Card expansions**: Cards expand smoothly when you click to see more details, with a slight scale and opacity change.
- **Modal open/close**: Modals fade in and scale up from 95% to 100% size when opening, and reverse when closing.
- **Sidebar toggle**: The sidebar slides in/out with a 200ms ease‑in‑out transition.
- **Tool call appearance**: Tool cards slide in from the bottom as they appear in the chat.

## 18. Dark/Light Mode

The app supports both dark and light themes:

- **Toggle switch**: A sun/moon icon in the header switches between themes.
- **Automatic switching**: The app can follow your system’s theme setting.
- **Persistence**: Your theme choice is saved in your profile and remembered across sessions.
- **Color adjustments**: All colors, backgrounds, and text contrast are adjusted for readability in each mode.

## 19. Accessibility

The app is built with accessibility in mind:

- **Screen reader support**: All interactive elements have proper ARIA labels and roles.
- **Keyboard navigation**: Every feature can be accessed using only the keyboard.
- **High contrast**: The app meets WCAG AA contrast ratios.
- **Focus indicators**: Visible focus outlines appear when you tab through elements.
- **Alt text**: Images and icons have descriptive alt text.
- **Resizable text**: You can increase or decrease text size in your browser, and the layout adapts.

## 20. Complete Interaction Map

Below is a table mapping common user actions to the visual responses you’ll see:

| User Action | Visual Response |
|-------------|-----------------|
| Click “New Chat” | Chat panel clears, input focus moves to chat box |
| Type in chat input | Placeholder text disappears, send button becomes active |
| Press Enter to send | Message bubbles up, typing dots appear, send button disables |
| Click send button | Same as pressing Enter |
| Click a tool card | Card expands to show full results |
| Click a citation | Tooltip appears, or source opens in side panel |
| Click “Open” on a document | New tab/panel opens with loading spinner, then editor loads |
| Click export button | Dropdown menu appears with format options |
| Select export format | Modal with progress bar appears, then download starts |
| Click sidebar icon | Sidebar toggles open/closed |
| Click a notification bell | Dropdown list of notifications appears |
| Click “Run” on a workflow | Canvas shows progress, nodes light up as they execute |
| Click “Cancel” on a background task | Task stops, progress bar disappears, status changes to “Cancelled” |
| Click “Accept” on a suggestion | Suggestion card disappears, change is applied to document |
| Click “Reject” on a suggestion | Suggestion card disappears, change is discarded |
| Click theme toggle | Entire app colors transition smoothly to new theme |
| Press `Ctrl + /` | Keyboard shortcut modal appears |
| Press `Escape` | Any open modal or panel closes |
| Tab through elements | Focus outline moves to each interactive element in order |

This table is a quick reference for the most common interactions. Each behavior is explained in more detail in the sections above.

---

*This guide covers all visual interactions in the Mike legal AI platform. If you encounter a behavior not listed here, please check the component‑specific documentation or contact the development team.*