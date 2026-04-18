# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Code Style

Use comments sparingly. Only comment complex code.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server (Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (unit/component tests)
npm run db:reset     # Force reset SQLite database (dev only)
```

Run a single test file:
```bash
npx vitest run src/path/to/file.test.ts
```

## Environment

Set `ANTHROPIC_API_KEY` in `.env` to use real Claude. If left empty, the app falls back to `MockLanguageModel` (static code generation) defined in `src/lib/provider.ts`.

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in chat; Claude generates code via tool calls; results render instantly in a sandboxed preview.

### Request Flow

```
Chat input → POST /api/chat → Vercel AI SDK (streamText)
  → Claude (Haiku-4.5) with tools: str_replace_editor, file_manager
  → VirtualFileSystem (in-memory)
  → Babel JSX transform → PreviewFrame renders component
```

### Key Layers

**AI Integration** (`src/app/api/chat/route.ts`, `src/lib/tools/`, `src/lib/prompts/`)
- Streaming via Vercel AI SDK. Two tools: `str_replace_editor` (edit file contents) and `file_manager` (create/delete files).
- System prompt in `src/lib/prompts/generation.tsx` defines how Claude generates components.
- Anthropic prompt caching (`ephemeral`) is enabled on the system prompt.

**Virtual File System** (`src/lib/file-system.ts`, `src/lib/contexts/file-system-context.tsx`)
- All generated files live in an in-memory tree (no disk I/O for component code).
- `FileSystemContext` is the global state; components read/write through it.

**JSX Transform** (`src/lib/transform/jsx-transformer.ts`)
- Babel standalone transpiles JSX → executable JS in the browser.
- `PreviewFrame` (`src/components/preview/`) runs the transpiled output.

**Auth & Persistence** (`src/lib/auth.ts`, `src/lib/prisma.ts`, `src/actions/`)
- JWT sessions via `jose`. Passwords hashed with bcrypt.
- Projects stored in SQLite (Prisma). Anonymous users can use the app but cannot save projects.
- Middleware in `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes.

**State Management**
- React Context only (no Redux/Zustand): `FileSystemContext` for files, `ChatContext` for messages and project tracking.

### Tech Stack

Next.js 15 (App Router) · React 19 · TypeScript 5 (strict) · Tailwind CSS v4 · Radix UI · Monaco Editor · Vercel AI SDK · Prisma + SQLite · Vitest
