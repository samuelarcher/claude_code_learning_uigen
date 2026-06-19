# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # First-time setup: install deps, generate Prisma client, run migrations
npm run dev            # Start dev server with Turbopack at http://localhost:3000
npm run dev:daemon     # Run dev server in background, logs to logs.txt
npm run build          # Production build
npm run lint           # ESLint
npm test               # Run all tests (Vitest)
npm run db:reset       # Reset and re-run all migrations (destructive)
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

**Do not run `npm audit fix`** — dependencies are pinned to specific compatible versions. Update pinned versions directly if security issues arise.

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY`. If unset or left as `your-api-key-here`, the app falls back to `MockLanguageModel` in `src/lib/provider.ts`, which returns canned components without calling the API. The model used is `claude-haiku-4-5`.

`JWT_SECRET` in `.env` protects session tokens; defaults to a dev-only string in `src/lib/auth.ts`.

## Architecture

**UIGen** is a Next.js 15 app (App Router) that lets users describe React components in a chat interface and see them rendered live in an iframe preview — no files are written to disk.

### Core data flow

1. User types a message → `ChatInterface` → POST `/api/chat`
2. The API route (`src/app/api/chat/route.ts`) calls Claude via Vercel AI SDK (`streamText`) with two tools: `str_replace_editor` and `file_manager`
3. Claude calls those tools to create/edit files in a server-side `VirtualFileSystem` instance
4. Tool results stream back; the client updates its own `VirtualFileSystem` (held in `FileSystemContext`)
5. `PreviewFrame` watches `refreshTrigger`, transforms all `.jsx`/`.tsx` files with Babel standalone, creates blob URLs, and builds an import map injected into an `<iframe srcdoc>`

### Virtual file system

`src/lib/file-system.ts` — `VirtualFileSystem` is a pure in-memory tree (no disk I/O). It is instantiated on both server (per-request, in the API route) and client (in `FileSystemContext`). The client state is the source of truth for the preview; project data is serialized as JSON in the `data` column of the `Project` table.

### Preview rendering

`src/lib/transform/jsx-transformer.ts` runs entirely in the browser:
- Transforms JSX/TSX with `@babel/standalone`
- Creates blob URLs for each file
- Builds an ES module import map (with `@/` alias support and automatic `esm.sh` fallback for third-party packages)
- Injects the map and a `ReactDOM.createRoot` bootstrap into the iframe

The iframe uses `allow-scripts allow-same-origin allow-forms` sandbox flags (same-origin is required for blob URL imports).

### AI tools

- `str_replace_editor` (`src/lib/tools/str-replace.ts`) — Claude's primary editing tool; supports `view`, `create`, `str_replace`, and `insert` commands
- `file_manager` (`src/lib/tools/file-manager.ts`) — supports `delete`, `rename`, and `list` commands

### Auth & persistence

- JWT sessions via `jose`, stored in an `httpOnly` cookie (`src/lib/auth.ts`). Auth is server-only (marked with `import "server-only"`).
- Middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem` routes.
- Anonymous users can use the app; only authenticated users can persist projects.
- Prisma with SQLite (`prisma/dev.db`). Schema has two models: `User` and `Project`. The `Project.messages` and `Project.data` columns store JSON strings.
- Generated Prisma client lives in `src/generated/prisma/` (not `node_modules`).

### State management

- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — wraps `VirtualFileSystem` for the client, exposes a `refreshTrigger` counter that the preview subscribes to
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK's `useChat`, wires tool call results back into `FileSystemContext`
- Server actions in `src/actions/` handle project CRUD

### Testing

Tests use Vitest + jsdom + React Testing Library. Test files live alongside source in `__tests__/` subdirectories.
