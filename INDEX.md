# Mike (Prism) - Comprehensive Documentation Index

> **AI-Powered Legal Document Assistant Platform**
> Complete architecture, reasoning, and implementation guide.

---

## Documentation Overview

This directory contains **16 comprehensive documents** covering every aspect of the Mike platform. Together, they form the complete knowledge base for understanding, maintaining, and extending the system.

**Total Documentation:** ~15,000+ lines across 16 documents.

---

## Documents

### Core AI & Orchestration

| # | Document | Lines | Purpose |
|---|----------|-------|---------|
| 1 | [AI Orchestration Bible](./01-AI-ORCHESTRATION-BIBLE.md) | ~900 | How the AI thinks, decides, and acts |
| 2 | [Tool Calling Manual](./02-TOOL-CALLING-MANUAL.md) | ~2300 | Encyclopedia of every tool the AI can use |
| 3 | [Prompt Library](./03-PROMPT-LIBRARY.md) | ~850 | Every prompt used in the system |
| 4 | [Agent Decision Tree](./04-AGENT-DECISION-TREE.md) | ~1160 | Complete decision trees for every agent |

### User Interface & Interaction

| # | Document | Lines | Purpose |
|---|----------|-------|---------|
| 5 | [UI Behavior Guide](./05-UI-BEHAVIOR-GUIDE.md) | ~800 | Every visual interaction in the app |
| 6 | [Streaming Protocol](./06-STREAMING-PROTOCOL.md) | ~1160 | Every event between backend and frontend |

### Data Pipelines

| # | Document | Lines | Purpose |
|---|----------|-------|---------|
| 7 | [Document Pipeline](./07-DOCUMENT-PIPELINE.md) | ~1300 | Every transformation a document goes through |
| 8 | [Editor Pipeline](./08-EDITOR-PIPELINE.md) | ~770 | How documents are loaded, edited, and saved |
| 9 | [RAG Pipeline](./09-RAG-PIPELINE.md) | ~1220 | Retrieval-Augmented Generation system |

### Agent Systems

| # | Document | Lines | Purpose |
|---|----------|-------|---------|
| 10 | [Multi-Agent Handbook](./10-MULTI-AGENT-HANDBOOK.md) | ~1530 | Complete reference for every agent |
| 11 | [State Machine Documentation](./11-STATE-MACHINE-DOCUMENTATION.md) | ~1150 | Every state and transition in the system |
| 12 | [AI Memory Architecture](./12-AI-MEMORY-ARCHITECTURE.md) | ~1390 | How the AI remembers things |

### Infrastructure & Data

| # | Document | Lines | Purpose |
|---|----------|-------|---------|
| 13 | [Database Relationship Guide](./13-DATABASE-RELATIONSHIP-GUIDE.md) | ~1740 | Every table and how they relate |
| 14 | [Complete Event System](./14-COMPLETE-EVENT-SYSTEM.md) | ~1540 | Every event in the system |

### Roadmap & Contribution

| # | Document | Lines | Purpose |
|---|----------|-------|---------|
| 15 | [AI Improvement Roadmap](./15-AI-IMPROVEMENT-ROADMAP.md) | ~2450 | Path to competing with top legal AI products |
| 16 | [AI Contributor Guide](./16-AI-CONTRIBUTOR-GUIDE.md) | ~1190 | Guide for developers and AI coding agents |

---

## How to Use This Documentation

### For New Engineers
Start with these documents in order:
1. **AI Orchestration Bible** - Understand the big picture
2. **Agent Decision Tree** - Learn how decisions are made
3. **Tool Calling Manual** - Know what tools are available
4. **Database Relationship Guide** - Understand the data model
5. **AI Contributor Guide** - Learn how to contribute

### For AI Architects
Focus on:
1. **AI Orchestration Bible** - Core reasoning architecture
2. **Prompt Library** - All prompts used
3. **Multi-Agent Handbook** - Agent system design
4. **RAG Pipeline** - Retrieval system
5. **AI Memory Architecture** - Memory systems
6. **AI Improvement Roadmap** - Future direction

### For Frontend Developers
Focus on:
1. **UI Behavior Guide** - All visual interactions
2. **Streaming Protocol** - Real-time events
3. **Editor Pipeline** - Document editing
4. **AI Contributor Guide** - Coding standards

### For Backend Developers
Focus on:
1. **Document Pipeline** - Data processing
2. **Complete Event System** - Event architecture
3. **Database Relationship Guide** - Data model
4. **AI Contributor Guide** - Coding standards

### For AI Coding Agents
Start with:
1. **AI Contributor Guide** - Rules and conventions
2. **AI Orchestration Bible** - System understanding
3. **Tool Calling Manual** - Available tools
4. **Database Relationship Guide** - Data model

---

## Key Architecture Decisions

### Dual Orchestration System
- **Legacy Chat Orchestrator** (`chatOrchestrator.ts`) - Real-time chat
- **New Multi-Agent Orchestrator** (`orchestrator/`) - Complex tasks

### Dual Editor System
- **TipTap** - HTML/markdown content
- **SuperDoc** - DOCX fidelity editing

### Multi-Provider LLM
- **OpenAI** (GPT-4, GPT-4o)
- **Google Gemini**
- **Anthropic Claude**
- **Kimi** (alternative)

### Event-Driven Architecture
- **SSE** for real-time streaming
- **BullMQ** for background jobs
- **Event Bus** for internal communication

---

## Technology Stack Summary

| Layer | Technology |
|-------|-----------|
| Frontend | React 19, Vite 8, TypeScript 6, Redux Toolkit, TipTap, SuperDoc |
| Backend | Node.js 20+, Express 4, TypeScript 5.8, Drizzle ORM |
| Database | PostgreSQL 16, Redis + BullMQ |
| AI/LLM | OpenAI, Gemini, Claude, Kimi |
| Storage | Cloudflare R2 |
| Sandbox | Docker, Go, Bun, LibreOffice |
| Testing | Vitest, Playwright, MSW |

---

## Document Statistics

| Metric | Value |
|--------|-------|
| Total Documents | 16 |
| Total Lines | ~15,000+ |
| Total Size | ~500KB+ |
| Coverage | Architecture, AI, Tools, Prompts, Agents, UI, Streaming, Documents, Editor, RAG, Memory, Database, Events, Roadmap, Contribution |

---

## Maintenance

These documents should be updated when:
- New agents are added
- New tools are added
- Prompts are modified
- Database schema changes
- API endpoints change
- UI components change
- Event types change
- Architecture changes

---

*Last updated: July 2026*
*Generated from codebase analysis of Mike (Prism) v1.0*
