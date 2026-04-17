# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack
npm run dev:daemon   # Start dev server in background, logs to logs.txt
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run tests with Vitest
npm run db:reset     # Reset database (destructive)
```

Run a single test file: `npx vitest run src/path/to/file.test.ts`

## Environment

Requires `ANTHROPIC_API_KEY` for AI features. Without it, the app falls back to a `MockLanguageModel` in `src/lib/provider.ts` that returns static example components. The real model used is `claude-haiku-4-5` (configurable in `src/lib/provider.ts`).

## Architecture

**UIGen** is an AI-powered React component generator with live preview, built on Next.js 15 App Router.

### Core Flow

1. User sends a chat message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. AI streams a response using Vercel AI SDK, calling tools to write files
3. Tools operate on the **virtual file system** (in-memory, no disk writes)
4. The virtual file system state triggers re-rendering of the preview iframe
5. The iframe transforms JSX at runtime via Babel (`src/lib/transform/jsx-transformer.ts`), importing third-party packages from `esm.sh`

### Virtual File System

`src/lib/file-system.ts` — `VirtualFileSystem` class that lives entirely in memory. It is serializable to JSON for database persistence. The `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) wraps it in React state and distributes it to components.

### AI Tools

Two tools are registered in `src/app/api/chat/route.ts`:
- `str_replace_editor` (`src/lib/tools/str-replace.ts`) — view, create, str_replace, insert operations on virtual files
- `file_manager` (`src/lib/tools/file-manager.ts`) — rename and delete files

The system prompt in `src/lib/prompts/generation.tsx` instructs the model to generate Tailwind-styled React components.

### Preview

`src/components/preview/PreviewFrame.tsx` renders a sandboxed iframe. `src/lib/transform/jsx-transformer.ts` uses Babel to transpile JSX→JS at runtime, resolves imports from the virtual file system into Blob URLs, and fetches third-party packages from `https://esm.sh`. The preview injects Tailwind CSS via CDN (`cdn.tailwindcss.com`), so generated components can use Tailwind classes without any config.

Entry point detection order: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx` → `/src/App.tsx` → first `.jsx`/`.tsx` file found. The `refreshTrigger` counter in `FileSystemContext` drives re-renders of the preview.

### Authentication

JWT tokens stored in httpOnly cookies (7-day expiry). `src/lib/auth.ts` handles session CRUD. `src/actions/index.ts` has `signUp`/`signIn`/`signOut`/`getUser` server actions. `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes.

Anonymous users can work without logging in; `src/lib/anon-work-tracker.ts` tracks their progress in localStorage and prompts sign-up at a threshold.

### AI-Generated File Conventions

The system prompt (`src/lib/prompts/generation.tsx`) enforces these rules for AI-generated code:
- Every project must have `/App.jsx` as its root entry point with a default export
- All local imports use the `@/` alias (e.g. `import Foo from '@/components/Foo'`)
- Style with Tailwind only — no hardcoded styles, no HTML files

### Database

Prisma + SQLite. Schema in `prisma/schema.prisma`: `User` has many `Project`s. Project stores messages and file system state as JSON strings. The Prisma client is generated to `src/generated/prisma` (non-standard path set in `prisma/schema.prisma`).

### State Management

- `FileSystemContext` — virtual file system state and operations
- `ChatContext` — wraps `useChat` from `@ai-sdk/react` for message streaming

### UI Layout

`src/app/main-content.tsx` — resizable three-panel layout: chat | preview | code editor. The editor uses Monaco (`src/components/editor/CodeEditor.tsx`) with a `FileTree` sidebar.
