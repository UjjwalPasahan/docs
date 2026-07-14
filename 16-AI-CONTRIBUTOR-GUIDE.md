# AI Contributor Guide

## 1. What This Document Is

This guide is for anyone who wants to contribute to the Mike codebase—whether you're a human developer or an autonomous coding agent like Claude Code, Cursor, or Codex. It explains how the project is structured, what conventions to follow, and how to make changes safely.

If you're an AI agent, treat this document as your primary reference. It tells you exactly where things go, how to name files, what patterns to follow, and what mistakes to avoid. If you're a human developer, this guide will help you onboard quickly and write code that fits the existing codebase.

Mike is a legal AI platform. It combines a React frontend, an Express backend, PostgreSQL database, Redis job queues, and AI-powered document analysis. The codebase is written entirely in TypeScript, with a few Go packages for low-level operations.

Read this guide before making any changes. It will save you time and prevent common errors.

---

## 2. Project Overview

### What Mike Is

Mike is a legal AI platform that helps lawyers and legal professionals analyze, draft, and manage legal documents. It uses AI to extract insights from documents, generate summaries, and automate routine legal tasks.

The platform supports real-time collaboration, document versioning, and AI-assisted editing. Users can upload documents, ask questions about them, and receive AI-generated analysis. The system handles complex document formats and maintains a history of all changes.

### Tech Stack

**Frontend:**
- React 19 with Vite
- TypeScript
- Redux Toolkit for state management
- RTK Query for API calls
- TipTap for rich text editing
- SuperDoc for document rendering
- React Router for navigation
- Vitest for testing

**Backend:**
- Express.js
- TypeScript
- Drizzle ORM for database access
- PostgreSQL 16
- Redis for caching and job queues
- BullMQ for job processing
- Server-Sent Events (SSE) for streaming
- Zod for validation
- Vitest for testing

**Packages:**
- `packages/shared-types/` - TypeScript types used across frontend and backend
- `packages/editor-schema/` - TipTap editor schema definitions
- `packages/container-agent/` - Go container agent for sandbox management
- `packages/fuse-driver/` - Go FUSE driver for file system operations
- `packages/db/` - Database migrations and schema

**Services:**
- `services/sandbox-orchestrator/` - Bun-based sandbox orchestration service
- `sandbox/skills/` - Document generation scripts

**Infrastructure:**
- Docker for containerization
- Docker Compose for local development

### Architecture

The system follows a modular architecture:

1. **Frontend** communicates with the backend via REST API and SSE streams.
2. **Backend** handles HTTP requests, processes jobs via queues, and orchestrates AI operations.
3. **Database** stores documents, users, sessions, and application state.
4. **Redis** manages job queues, caching, and real-time pub/sub.
5. **Sandbox** environment runs document generation and AI skills in isolation.

### Key Directories

```
mike/
├── frontend/           # React Vite frontend
├── backend/            # Express backend
├── packages/           # Shared packages
│   ├── shared-types/   # TypeScript types
│   ├── editor-schema/  # TipTap schema
│   ├── container-agent/# Go container agent
│   ├── fuse-driver/    # Go FUSE driver
│   └── db/             # Database migrations
├── services/           # Microservices
│   └── sandbox-orchestrator/  # Bun sandbox service
├── sandbox/            # Sandbox environment
│   └── skills/         # Document generation scripts
├── docker/             # Dockerfiles
└── docs/               # Documentation
```

---

## 3. Getting Started

### Prerequisites

Before you begin, install these tools:

- **Node.js 20+** - JavaScript runtime
- **PostgreSQL 16** - Primary database
- **Redis** - Caching and job queues
- **Bun** - For sandbox services (faster than Node.js for some tasks)
- **Git** - Version control
- **Docker** (optional) - For containerized development

### Setup Steps

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd mike
   ```

2. **Install dependencies:**
   ```bash
   npm install
   ```

3. **Set up the database:**
   ```bash
   # Create PostgreSQL database
   createdb mike_dev
   
   # Run migrations
   npm run db:migrate
   
   # Seed with sample data (optional)
   npm run db:seed
   ```

4. **Configure environment:**
   ```bash
   # Copy example env file
   cp .env.example .env
   
   # Edit .env with your database credentials
   # Required: DATABASE_URL, REDIS_URL, JWT_SECRET
   ```

5. **Start development:**
   ```bash
   # Start all services
   npm run dev
   
   # Or start individually:
   npm run dev:frontend  # Frontend on port 5173
   npm run dev:backend   # Backend on port 3000
   ```

### Development Commands

| Command | Description |
|---------|-------------|
| `npm run dev` | Start all services in development mode |
| `npm run build` | Build all packages for production |
| `npm run test` | Run all tests |
| `npm run test:watch` | Run tests in watch mode |
| `npm run lint` | Lint all code with ESLint |
| `npm run lint:fix` | Fix auto-fixable lint errors |
| `npm run typecheck` | Run TypeScript type checking |
| `npm run db:migrate` | Run database migrations |
| `npm run db:seed` | Seed database with sample data |
| `npm run db:studio` | Open Drizzle Studio |

---

## 4. Coding Standards

### TypeScript

- **Strict mode** is enabled in `tsconfig.json`. Never disable it.
- **No `any` types**. Use `unknown` if the type is truly unknown, then narrow it.
- **Use interfaces** for object shapes that might be extended.
- **Use type aliases** for unions, intersections, and simpler types.
- **Validate external data** with Zod schemas. Never trust API responses or user input.

```typescript
// Good
interface User {
  id: string;
  email: string;
  name: string;
}

// Bad
const user: any = await fetchUser();
```

### Naming Conventions

| Item | Convention | Example |
|------|-----------|---------|
| Files | kebab-case | `user-service.ts`, `auth-middleware.ts` |
| Components | PascalCase | `UserCard.tsx`, `DocumentList.tsx` |
| Functions | camelCase | `getUserById()`, `createDocument()` |
| Types/Interfaces | PascalCase | `User`, `DocumentMetadata` |
| Constants | UPPER_SNAKE_CASE | `MAX_FILE_SIZE`, `API_VERSION` |
| Database tables | snake_case | `users`, `documents`, `document_versions` |
| Database columns | snake_case | `created_at`, `user_id` |

### Code Style

- **Indentation:** 2 spaces (no tabs)
- **Quotes:** Single quotes for strings
- **Semicolons:** Always use semicolons
- **Trailing commas:** Use trailing commas in multi-line arrays/objects
- **Max line length:** 100 characters
- **Imports:** Group imports: external libraries first, then internal modules, then relative imports

```typescript
// Import order
import express from 'express';
import { z } from 'zod';

import { UserService } from '../modules/user/user-service';
import { db } from '../db';
import { logger } from '../lib/logger';

import type { Request, Response } from 'express';
```

---

## 5. Folder Conventions

### Backend Structure

```
backend/src/
├── routes/              # API route definitions
│   ├── auth.ts          # Authentication routes
│   ├── documents.ts     # Document routes
│   └── index.ts         # Route aggregator
├── modules/             # Feature modules
│   ├── user/            # User module
│   │   ├── user-service.ts
│   │   ├── user-repository.ts
│   │   └── user-types.ts
│   └── document/        # Document module
├── lib/                 # Business logic utilities
│   ├── ai/              # AI integration
│   ├── auth/            # Authentication
│   ├── email/           # Email services
│   └── storage/         # File storage
├── db/                  # Database
│   ├── index.ts         # Database connection
│   ├── schema.ts        # Drizzle schema
│   └── migrations/      # Generated migrations
├── middleware/           # Express middleware
│   ├── auth.ts          # Authentication middleware
│   ├── validation.ts    # Request validation
│   └── error-handler.ts # Error handling
├── prompts/             # AI prompt templates
├── tools/               # AI tool definitions
├── skills/              # Legal skill implementations
├── agent/               # Background agent logic
├── queue/               # BullMQ job queues
├── events/              # Event bus
├── scripts/             # Utility scripts
└── shared/              # Shared utilities
    ├── constants.ts
    ├── errors.ts
    └── logger.ts
```

### Frontend Structure

```
frontend/src/client/
├── app/                 # App shell and providers
│   ├── App.tsx          # Main app component
│   ├── providers.tsx    # Context providers
│   └── routes.tsx       # Route definitions
├── components/          # Reusable UI components
│   ├── ui/              # Basic UI components (Button, Input, etc.)
│   ├── layout/          # Layout components (Header, Sidebar)
│   └── shared/          # Shared business components
├── pages/               # Page components
│   ├── Dashboard.tsx
│   ├── Documents.tsx
│   └── Settings.tsx
├── store/               # Redux store
│   ├── index.ts         # Store configuration
│   ├── api.ts           # RTK Query API
│   └── slices/          # Feature slices
├── hooks/               # Custom React hooks
│   ├── useAuth.ts
│   └── useDocuments.ts
├── contexts/            # React contexts
│   ├── AuthContext.tsx
│   └── ThemeContext.tsx
├── services/            # API service functions
│   ├── document-service.ts
│   └── user-service.ts
├── features/            # Feature-specific code
│   ├── editor/          # Document editor feature
│   └── ai-chat/         # AI chat feature
├── shared/              # Shared utilities
│   ├── utils.ts
│   └── constants.ts
├── types/               # TypeScript type definitions
└── config/              # Configuration constants
```

---

## 6. Adding a New API Route

### Step-by-Step Guide

1. **Create the route file** in `backend/src/routes/`:
   ```typescript
   // backend/src/routes/documents.ts
   import { Router } from 'express';
   import { z } from 'zod';
   import { validateRequest } from '../middleware/validation';
   import { DocumentService } from '../modules/document/document-service';
   
   const router = Router();
   
   // Define Zod schema for request validation
   const createDocumentSchema = z.object({
     title: z.string().min(1).max(255),
     content: z.string().optional(),
     folderId: z.string().uuid().optional(),
   });
   
   // POST /api/documents
   router.post('/', validateRequest(createDocumentSchema), async (req, res) => {
     try {
       const document = await DocumentService.create(req.body);
       res.status(201).json(document);
     } catch (error) {
       // Error handling is done by middleware
       throw error;
     }
   });
   
   export default router;
   ```

2. **Add the route to the app** in `backend/src/routes/index.ts`:
   ```typescript
   import documentsRouter from './documents';
   
   export function registerRoutes(app: Express) {
     app.use('/api/documents', documentsRouter);
     // ... other routes
   }
   ```

3. **Create the service** if it doesn't exist:
   ```typescript
   // backend/src/modules/document/document-service.ts
   import { db } from '../../db';
   import { documents } from '../../db/schema';
   
   export class DocumentService {
     static async create(data: CreateDocumentInput) {
       // Implementation
     }
   }
   ```

4. **Add to Swagger documentation** if using API docs.

5. **Write tests**:
   ```typescript
   // backend/src/routes/__tests__/documents.test.ts
   import { describe, it, expect } from 'vitest';
   import request from 'supertest';
   import { app } from '../../app';
   
   describe('POST /api/documents', () => {
     it('should create a document', async () => {
       const response = await request(app)
         .post('/api/documents')
         .send({ title: 'Test Document' });
       
       expect(response.status).toBe(201);
       expect(response.body.title).toBe('Test Document');
     });
   });
   ```

---

## 7. Adding a New Frontend Page

### Step-by-Step Guide

1. **Create the page component** in `frontend/src/client/pages/`:
   ```typescript
   // frontend/src/client/pages/Settings.tsx
   import React from 'react';
   import { useGetSettingsQuery } from '../store/api';
   
   export function Settings() {
     const { data: settings, isLoading } = useGetSettingsQuery();
   
     if (isLoading) return <div>Loading...</div>;
   
     return (
       <div className="settings-page">
         <h1>Settings</h1>
         {/* Page content */}
       </div>
     );
   }
   ```

2. **Add the route** in `frontend/src/client/app/routes.tsx`:
   ```typescript
   import { Settings } from '../pages/Settings';
   
   export const routes = [
     // ... other routes
     {
       path: '/settings',
       element: <Settings />,
     },
   ];
   ```

3. **Create API service** using RTK Query if needed:
   ```typescript
   // frontend/src/client/store/api.ts
   import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';
   
   export const api = createApi({
     baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
     endpoints: (builder) => ({
       getSettings: builder.query<Settings, void>({
         query: () => '/settings',
       }),
     }),
   });
   ```

4. **Add Redux slice** if complex state management is needed:
   ```typescript
   // frontend/src/client/store/slices/settings-slice.ts
   import { createSlice } from '@reduxjs/toolkit';
   
   export const settingsSlice = createSlice({
     name: 'settings',
     initialState: {},
     reducers: {
       // Reducers
     },
   });
   ```

5. **Write tests**:
   ```typescript
   // frontend/src/client/pages/__tests__/Settings.test.tsx
   import { render, screen } from '@testing-library/react';
   import { Settings } from '../Settings';
   
   describe('Settings', () => {
     it('renders settings page', () => {
       render(<Settings />);
       expect(screen.getByText('Settings')).toBeInTheDocument();
     });
   });
   ```

---

## 8. Adding a New AI Tool

### Step-by-Step Guide

1. **Define the tool schema** in `backend/src/tools/`:
   ```typescript
   // backend/src/tools/summarize-document.ts
   import { z } from 'zod';
   
   export const summarizeDocumentSchema = z.object({
     documentId: z.string().uuid().describe('The document to summarize'),
     maxLength: z.number().optional().describe('Maximum summary length'),
   });
   
   export type SummarizeDocumentInput = z.infer<typeof summarizeDocumentSchema>;
   ```

2. **Implement the executor**:
   ```typescript
   // backend/src/tools/summarize-document.ts
   export async function executeSummarizeDocument(
     input: SummarizeDocumentInput,
     context: ToolContext
   ) {
     const document = await DocumentService.getById(input.documentId);
     const summary = await AIService.summarize(document.content, {
       maxLength: input.maxLength,
     });
     
     return {
       summary,
       documentId: input.documentId,
     };
   }
   ```

3. **Register the tool** in the tool registry:
   ```typescript
   // backend/src/tools/index.ts
   import { summarizeDocumentSchema, executeSummarizeDocument } from './summarize-document';
   
   export const tools = {
     summarize_document: {
       schema: summarizeDocumentSchema,
       execute: executeSummarizeDocument,
       description: 'Summarizes a legal document',
     },
   };
   ```

4. **Create frontend renderer** if the tool returns rich content:
   ```typescript
   // frontend/src/client/features/ai-chat/components/ToolRenderers.tsx
   export function SummaryRenderer({ result }: { result: SummaryResult }) {
     return (
       <div className="summary-tool">
         <h4>Document Summary</h4>
         <p>{result.summary}</p>
       </div>
     );
   }
   ```

5. **Write tests**:
   ```typescript
   describe('summarize_document tool', () => {
     it('should summarize a document', async () => {
       const result = await executeSummarizeDocument(
         { documentId: 'test-id' },
         mockContext
       );
       
       expect(result.summary).toBeDefined();
     });
   });
   ```

---

## 9. Adding a New Agent

### Step-by-Step Guide

1. **Create the agent class** in `backend/src/agent/`:
   ```typescript
   // backend/src/agent/document-reviewer.ts
   export class DocumentReviewerAgent {
     private name = 'document-reviewer';
     private description = 'Reviews documents for legal compliance';
   
     async execute(task: AgentTask): Promise<AgentResult> {
       // Agent logic here
       return {
         status: 'completed',
         result: { /* output */ },
       };
     }
   }
   ```

2. **Define responsibilities** clearly:
   - What the agent does
   - What inputs it expects
   - What outputs it produces
   - What tools it can use

3. **Implement required tools** or use existing ones:
   ```typescript
   private tools = ['read_document', 'analyze_clauses', 'check_compliance'];
   ```

4. **Register with the coordinator**:
   ```typescript
   // backend/src/agent/coordinator.ts
   import { DocumentReviewerAgent } from './document-reviewer';
   
   export const agents = {
     'document-reviewer': new DocumentReviewerAgent(),
   };
   ```

5. **Write tests**:
   ```typescript
   describe('DocumentReviewerAgent', () => {
     it('should review a document', async () => {
       const agent = new DocumentReviewerAgent();
       const result = await agent.execute({
         type: 'review',
         documentId: 'test-id',
       });
       
       expect(result.status).toBe('completed');
     });
   });
   ```

---

## 10. Database Changes

### Step-by-Step Guide

1. **Update the schema** in `packages/db/src/schema.ts`:
   ```typescript
   // packages/db/src/schema.ts
   import { pgTable, uuid, varchar, timestamp } from 'drizzle-orm/pg-core';
   
   export const documents = pgTable('documents', {
     id: uuid('id').defaultRandom().primaryKey(),
     title: varchar('title', { length: 255 }).notNull(),
     createdAt: timestamp('created_at').defaultNow().notNull(),
     updatedAt: timestamp('updated_at').defaultNow().notNull(),
   });
   
   // Add new table or column
   export const documentTags = pgTable('document_tags', {
     id: uuid('id').defaultRandom().primaryKey(),
     documentId: uuid('document_id').references(() => documents.id),
     tag: varchar('tag', { length: 50 }).notNull(),
   });
   ```

2. **Generate migration**:
   ```bash
   npm run db:generate
   ```

3. **Review the generated migration** in `packages/db/migrations/`.

4. **Test the migration**:
   ```bash
   npm run db:migrate
   ```

5. **Update repositories** to use new schema:
   ```typescript
   // backend/src/modules/document/document-repository.ts
   import { documentTags } from '@mike/db/schema';
   
   export class DocumentRepository {
     async addTag(documentId: string, tag: string) {
       return db.insert(documentTags).values({ documentId, tag });
     }
   }
   ```

---

## 11. Testing Requirements

### Unit Tests

- **Test individual functions** in isolation.
- **Mock external dependencies** (databases, APIs, file system).
- **Aim for 80% coverage** on new code.
- **Place tests** in `__tests__/` directories next to the code.

```typescript
// Good unit test
describe('calculateTotal', () => {
  it('should sum line items', () => {
    const items = [{ price: 10 }, { price: 20 }];
    expect(calculateTotal(items)).toBe(30);
  });

  it('should handle empty array', () => {
    expect(calculateTotal([])).toBe(0);
  });
});
```

### Integration Tests

- **Test API endpoints** end-to-end.
- **Test database operations** with a test database.
- **Test AI tool execution** with mocked AI services.
- **Place tests** in `__tests__/` directories.

```typescript
// Good integration test
describe('POST /api/documents', () => {
  it('should create document in database', async () => {
    const response = await request(app)
      .post('/api/documents')
      .send({ title: 'Test' });
    
    expect(response.status).toBe(201);
    
    const doc = await db.query.documents.findFirst({
      where: eq(documents.title, 'Test'),
    });
    expect(doc).toBeDefined();
  });
});
```

### E2E Tests

- **Test critical user flows** with Playwright.
- **Test login, document creation, AI features**.
- **Place tests** in `e2e/` directory.

```typescript
// Good E2E test
test('user can create document', async ({ page }) => {
  await page.goto('/documents');
  await page.click('button:has-text("New Document")');
  await page.fill('input[name="title"]', 'My Document');
  await page.click('button:has-text("Save")');
  
  await expect(page.locator('h1')).toContainText('My Document');
});
```

---

## 12. Security Rules

### Never Do

- **Never log secrets** (passwords, API keys, tokens).
- **Never commit secrets** to version control.
- **Never trust user input** without validation.
- **Never skip validation** on API endpoints.
- **Never expose internal errors** to the client.
- **Never use `eval()`** or similar dynamic code execution.
- **Never store passwords** in plain text.

### Always Do

- **Always validate input** with Zod schemas.
- **Always use parameterized queries** (Drizzle handles this).
- **Always hash passwords** with bcrypt.
- **Always use HTTPS** in production.
- **Always log security events** (failed logins, permission denied).
- **Sanitize output** to prevent XSS.
- **Use rate limiting** on authentication endpoints.

```typescript
// Good: Input validation
const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

// Bad: No validation
app.post('/login', (req, res) => {
  const { email, password } = req.body; // Could be anything!
});
```

---

## 13. Performance Rules

### Backend

- **Use connection pooling** for database connections.
- **Cache frequent queries** with Redis.
- **Add indexes** for frequently queried columns.
- **Paginate results** to avoid large responses.
- **Use streaming** for large data transfers.
- **Batch database operations** when possible.

```typescript
// Good: Paginated query
const documents = await db.query.documents.findMany({
  limit: 20,
  offset: (page - 1) * 20,
  orderBy: desc(documents.createdAt),
});

// Good: Cached query
async function getDocument(id: string) {
  const cached = await redis.get(`document:${id}`);
  if (cached) return JSON.parse(cached);
  
  const doc = await db.query.documents.findFirst({
    where: eq(documents.id, id),
  });
  
  await redis.set(`document:${id}`, JSON.stringify(doc), 'EX', 3600);
  return doc;
}
```

### Frontend

- **Lazy load components** with React.lazy.
- **Memoize expensive calculations** with useMemo.
- **Use React.memo** for pure components.
- **Debounce user input** to reduce API calls.
- **Virtual scroll** for long lists (react-window).
- **Code split** by route.

```typescript
// Good: Lazy loading
const Dashboard = React.lazy(() => import('./pages/Dashboard'));

// Good: Memoized computation
const sortedDocuments = useMemo(() => {
  return documents.sort((a, b) => 
    new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime()
  );
}, [documents]);

// Good: Debounced search
const debouncedSearch = useMemo(
  () => debounce((query: string) => search(query), 300),
  []
);
```

---

## 14. Error Handling Patterns

### Try/Catch Blocks

Wrap potentially failing operations:

```typescript
async function createDocument(data: CreateDocumentInput) {
  try {
    const document = await db.insert(documents).values(data).returning();
    return document[0];
  } catch (error) {
    logger.error('Failed to create document', { error, data });
    throw new ApplicationError('Failed to create document', {
      cause: error,
    });
  }
}
```

### Error Boundaries

Use React error boundaries for UI errors:

```typescript
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  
  static getDerivedStateFromError() {
    return { hasError: true };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    logger.error('UI Error', { error, info });
  }
  
  render() {
    if (this.state.hasError) {
      return <ErrorFallback />;
    }
    return this.props.children;
  }
}
```

### Error Responses

Use consistent error format:

```typescript
// Application error class
export class ApplicationError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode: number = 500,
    public details?: Record<string, unknown>
  ) {
    super(message);
    this.name = 'ApplicationError';
  }
}

// Error handler middleware
function errorHandler(err, req, res, next) {
  if (err instanceof ApplicationError) {
    return res.status(err.statusCode).json({
      error: {
        code: err.code,
        message: err.message,
        details: err.details,
      },
    });
  }
  
  // Unknown error
  logger.error('Unhandled error', { error: err });
  return res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
    },
  });
}
```

---

## 15. Streaming Conventions

### SSE Format

Use Server-Sent Events for real-time updates:

```typescript
// Backend: SSE endpoint
app.get('/api/stream/:id', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  
  const sendEvent = (event: string, data: unknown) => {
    res.write(`event: ${event}\n`);
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };
  
  // Send heartbeat every 30 seconds
  const heartbeat = setInterval(() => {
    sendEvent('heartbeat', { timestamp: Date.now() });
  }, 30000);
  
  // Cleanup on disconnect
  req.on('close', () => {
    clearInterval(heartbeat);
  });
});
```

### Event Naming

- Use kebab-case for event names: `document-updated`, `ai-thinking`
- Use descriptive names that indicate what happened

### Error Handling in Streams

```typescript
try {
  sendEvent('ai-thinking', { status: 'started' });
  const result = await processWithAI(data);
  sendEvent('ai-result', { result });
} catch (error) {
  sendEvent('error', { 
    message: 'AI processing failed',
    code: 'AI_ERROR',
  });
}
```

---

## 16. Review Checklist

Before submitting code, verify:

### Code Quality
- [ ] Tests pass (`npm run test`)
- [ ] Types are correct (`npm run typecheck`)
- [ ] No lint errors (`npm run lint`)
- [ ] Code follows naming conventions
- [ ] No `any` types used

### Security
- [ ] Input is validated
- [ ] No secrets logged or committed
- [ ] SQL injection prevented (using Drizzle)
- [ ] XSS prevented (sanitized output)
- [ ] Authentication checked where needed

### Performance
- [ ] Database queries optimized
- [ ] Proper indexes added
- [ ] Pagination implemented for lists
- [ ] Caching considered
- [ ] Large payloads avoided

### Documentation
- [ ] README updated (if needed)
- [ ] API documentation updated
- [ ] Complex logic commented
- [ ] Types are self-documenting

### Error Handling
- [ ] Errors are caught and handled
- [ ] User-friendly error messages
- [ ] Errors are logged
- [ ] Loading states handled
- [ ] Edge cases considered

---

## 17. Common Pitfalls

### 1. Forgetting to Validate Input

```typescript
// Bad: Direct access without validation
app.post('/api/documents', (req, res) => {
  const { title } = req.body; // Could be undefined or wrong type
});

// Good: Validate first
const schema = z.object({
  title: z.string().min(1),
});

app.post('/api/documents', validateRequest(schema), (req, res) => {
  const { title } = req.body; // Definitely a string
});
```

### 2. Not Handling Async Errors

```typescript
// Bad: Unhandled promise rejection
app.get('/api/documents', async (req, res) => {
  const docs = await getDocuments(); // If this fails, server crashes
  res.json(docs);
});

// Good: Try/catch or error middleware
app.get('/api/documents', async (req, res, next) => {
  try {
    const docs = await getDocuments();
    res.json(docs);
  } catch (error) {
    next(error); // Pass to error handler
  }
});
```

### 3. Creating N+1 Query Problems

```typescript
// Bad: N+1 queries
const documents = await db.query.documents.findMany();
for (const doc of documents) {
  doc.author = await db.query.users.findFirst({
    where: eq(users.id, doc.authorId),
  });
}

// Good: Join or batch
const documents = await db.query.documents.findMany({
  with: { author: true },
});
```

### 4. Forgetting to Clean Up

```typescript
// Bad: Memory leak
useEffect(() => {
  const interval = setInterval(fetchData, 5000);
  // Never cleared!
}, []);

// Good: Clean up
useEffect(() => {
  const interval = setInterval(fetchData, 5000);
  return () => clearInterval(interval);
}, []);
```

### 5. Hardcoding Values

```typescript
// Bad: Hardcoded value
const maxRetries = 3;

// Good: Configuration
const maxRetries = config.get('maxRetries', 3);
```

### 6. Ignoring TypeScript Errors

```typescript
// Bad: @ts-ignore
// @ts-ignore
const result = someRiskyOperation();

// Good: Proper typing
const result = someRiskyOperation() as string | null;
if (result !== null) {
  // Use result
}
```

### 7. Not Using Indexes

```typescript
// Bad: Full table scan
const users = await db.query.users.findMany({
  where: eq(users.email, email),
});

// Add index in schema:
// email: varchar('email', { length: 255 }).notNull().unique()
```

---

## 18. Useful Resources

### Official Documentation

- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/)
- [React Documentation](https://react.dev/)
- [Redux Toolkit](https://redux-toolkit.js.org/)
- [Express.js Guide](https://expressjs.com/)
- [Drizzle ORM](https://orm.drizzle.team/)
- [Zod Documentation](https://zod.dev/)
- [Vitest Documentation](https://vitest.dev/)
- [BullMQ Documentation](https://docs.bullmq.io/)
- [Vite Documentation](https://vitejs.dev/)
- [TipTap Documentation](https://tiptap.dev/)

### Project-Specific

- `docs/ARCHITECTURE.md` - System architecture overview
- `docs/comprehensive/` - Detailed documentation series
- `README.md` - Project setup and overview
- `package.json` - Available scripts

### Tools

- [Drizzle Studio](https://orm.drizzle.team/studio) - Database GUI
- [React DevTools](https://chrome.google.com/webstore/detail/react-developer-tools/) - Browser extension
- [Redux DevTools](https://chrome.google.com/webstore/detail/redux-devtools/) - State inspection

---

## 19. Getting Help

### When You're Stuck

1. **Check the documentation** in `docs/` directory.
2. **Search existing issues** in the repository.
3. **Look at similar code** for patterns and examples.
4. **Run the linter** - it often catches common mistakes.
5. **Check types** - TypeScript errors usually explain the problem.

### Debugging Tips

- Use `console.log` sparingly - prefer the logger.
- Use breakpoints in your IDE for complex debugging.
- Check browser Network tab for API issues.
- Use Drizzle Studio to inspect database state.
- Check Redis with `redis-cli` for queue issues.

### Asking for Help

When asking for help, provide:

1. **What you're trying to do** - Clear description of the goal.
2. **What you've tried** - Steps you've already taken.
3. **Error messages** - Full error output, not just the last line.
4. **Relevant code** - Minimal code that reproduces the issue.
5. **Environment details** - OS, Node version, etc.

### Code Review

When reviewing code:

- Focus on correctness first, style second.
- Check for security issues.
- Verify error handling.
- Look for performance problems.
- Suggest improvements constructively.

---

*This guide is maintained by the Mike development team. Last updated: 2026*