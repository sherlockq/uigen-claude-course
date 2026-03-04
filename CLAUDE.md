# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # Initial setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack (localhost:3000)
npm run build        # Build for production
npm run lint         # Run ESLint
npm run test         # Run Vitest test suite
npm run db:reset     # Reset database to initial state
```

Run a single test file:
```bash
npx vitest run src/components/chat/__tests__/MessageList.test.tsx
```

## Architecture Overview

**UIGen** is an AI-powered React component generator. Users describe components in natural language; Claude generates code that appears in a code editor with live preview.

### Core Data Flow

1. User types in **ChatInterface** → sends to `/api/chat` route
2. API route streams responses from Claude (Anthropic API or mock) via Vercel AI SDK
3. Claude uses two structured tools to manipulate the **Virtual File System (VFS)**:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`): create/view/edit files
   - `file_manager` (`src/lib/tools/file-manager.ts`): rename/delete files
4. Tool call results update `FileSystemProvider` state → re-renders **CodeEditor** and **PreviewFrame**
5. **PreviewFrame** (`src/components/preview/PreviewFrame.tsx`) transforms JSX via `@babel/standalone` in an iframe and renders the component live

### Virtual File System

`src/lib/file-system.ts` — A fully in-memory file system (no disk writes). Paths always start with `/`. The VFS serializes to `Record<string, FileNode>` for database storage (JSON string in the `data` column).

Entry point detection order: `/App.jsx`, `/App.tsx`, `/index.jsx`, `/index.tsx`.

### State Management

Two main React contexts in `src/lib/contexts/`:
- **`file-system-context.tsx`** — VFS state and file operations (create, read, update, delete, rename)
- **`chat-context.tsx`** — Chat messages and AI integration via `useChat` from Vercel AI SDK

### AI Integration

- Model: Claude Haiku 4.5 (`src/lib/provider.ts`); falls back to mock provider if `ANTHROPIC_API_KEY` is unset
- System prompt: `src/lib/prompts/generation.tsx` — requires `/App.jsx` as root, Tailwind CSS only (no hardcoded styles), `@/` import alias for project files
- Max 40 tool call steps; prompt caching enabled on system message

### Authentication & Persistence

- JWT sessions via `jose` + bcrypt passwords (`src/lib/auth.ts`)
- Protected routes via `src/middleware.ts`
- Server actions in `src/actions/` handle auth and project CRUD
- Database: SQLite via Prisma (`prisma/schema.prisma`) with `User` and `Project` models
- Anonymous users: VFS is session-only; work tracked in localStorage (`src/lib/anon-work-tracker.ts`)

### Key Conventions

- All VFS paths normalized: start with `/`, no trailing slashes
- Generated components must use Tailwind CSS classes; Babel standalone handles JSX→JS in browser
- UI primitives are Radix UI components in `src/components/ui/`
- Tests use Vitest + React Testing Library + jsdom; test files live in `__tests__/` directories next to source
