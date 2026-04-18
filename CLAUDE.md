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

Set `ANTHROPIC_API_KEY` in `.env` to use real Claude. If left empty, the app falls back to `MockLanguageModel` (static code generation) defined in `src/lib/provider.ts`. The mock simulates a 4-step generation sequence using a state machine keyed on tool message count.

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in chat; Claude generates code via tool calls; results render instantly in a sandboxed preview.

### Request Flow

```
Chat input → POST /api/chat → Vercel AI SDK (streamText)
  → Claude (Haiku-4.5, 10k token budget, 40 tool steps max) with tools: str_replace_editor, file_manager
  → VirtualFileSystem (in-memory)
  → Babel JSX transform → PreviewFrame renders component
```

Each request body includes `fileSystem.serialize()` (VFS state as plain objects) and optional `projectId`. On stream finish, the project is saved to Prisma only if both `projectId` and a valid session are present.

### Key Layers

**AI Integration** (`src/app/api/chat/route.ts`, `src/lib/tools/`, `src/lib/prompts/`)
- Streaming via Vercel AI SDK. Two tools:
  - `str_replace_editor` — commands: `view`, `create`, `str_replace`, `insert`. Returns formatted strings. Auto-creates parent directories on `create`. `str_replace` does literal (non-regex) replacement of all occurrences.
  - `file_manager` — commands: `rename`, `delete`. Returns result objects. `rename` acts as move, creating intermediate directories.
- System prompt in `src/lib/prompts/generation.tsx` defines how Claude generates components.
- Anthropic prompt caching (`ephemeral`) is enabled on the system prompt.

**Virtual File System** (`src/lib/file-system.ts`, `src/lib/contexts/file-system-context.tsx`)
- All generated files live in an in-memory tree (no disk I/O for component code).
- Paths are normalized to `/path/format` (leading slash, no trailing slash).
- `serialize()` / `deserializeFromNodes()` convert to/from plain JSON-safe objects for transport.
- `FileSystemContext` is the global state; components read/write through it. It processes tool calls from the AI stream and increments a `refreshTrigger` counter to force UI updates.

**JSX Transform** (`src/lib/transform/jsx-transformer.ts`)
- Babel standalone transpiles JSX → executable JS in the browser.
- Generates an import map: `@/` aliases resolve to root, relative imports resolve to blob URLs, third-party packages resolve to `https://esm.sh/<package>`.
- CSS imports are stripped and collected as inline `<style>` blocks.
- The `PreviewFrame` (`src/components/preview/`) renders output in an `<iframe>` sandboxed with `allow-scripts allow-same-origin allow-forms`.

**Auth & Persistence** (`src/lib/auth.ts`, `src/lib/prisma.ts`, `src/actions/`)
- JWT sessions via `jose` (HS256, 7-day expiry, `auth-token` HttpOnly cookie).
- Passwords hashed with bcrypt (cost 10).
- Prisma schema: `User` (id, email, password) and `Project` (id, name, optional `userId`, `messages` as JSON string, `data` as JSON string).
- Anonymous users can use the app but cannot save projects. Anonymous work is tracked in `sessionStorage` (`uigen_has_anon_work`, `uigen_anon_data`) and converted to a named project on sign-in.
- Middleware in `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes.

**State Management**
- React Context only (no Redux/Zustand): `FileSystemContext` for files, `ChatContext` for messages and project tracking.
- `ChatContext` uses Vercel AI SDK's `useChat`; the `onToolCall` hook delegates to `FileSystemContext.handleToolCall`.

### Tech Stack

Next.js 15 (App Router) · React 19 · TypeScript 5 (strict) · Tailwind CSS v4 · Radix UI · Monaco Editor · Vercel AI SDK · Prisma + SQLite · Vitest (jsdom environment)
