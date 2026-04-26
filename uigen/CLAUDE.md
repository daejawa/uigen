# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps + generate Prisma client + migrate DB
npm run dev          # Start dev server (Turbopack) at http://localhost:3000
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run all Vitest tests
npx vitest run src/lib/__tests__/file-system.test.ts  # Run a single test file
npm run db:reset     # Reset and re-migrate the SQLite database
```

`NODE_OPTIONS='--require ./node-compat.cjs'` is automatically prepended by all npm scripts — this polyfill is required for Prisma to work with Next.js/Turbopack.

The Prisma client is generated to `src/generated/prisma/` (not the default location). After schema changes run `npx prisma generate && npx prisma migrate dev`.

## Architecture

### Core concept
UIGen is an AI-powered React component generator. Users describe a component in chat, the AI generates code using tool calls, and the result renders live in a sandboxed iframe — all without writing files to disk.

### Virtual file system
`src/lib/file-system.ts` — `VirtualFileSystem` is an in-memory store (path → content). All generated files live here. It serializes to/from plain JSON for persistence in the DB and transmission to the API route.

`src/lib/contexts/file-system-context.tsx` — React context that wraps `VirtualFileSystem` with state. The `refreshTrigger` counter is incremented on every mutation to force re-renders in consumers like `PreviewFrame`.

### AI generation pipeline
`src/app/api/chat/route.ts` — The POST handler receives messages + serialized file system, reconstructs `VirtualFileSystem`, then calls `streamText` (Vercel AI SDK) with two tools:
- `str_replace_editor` — create/str_replace/insert operations on files
- `file_manager` — rename/delete operations

Tool calls stream back to the client and are applied to the local `VirtualFileSystem` via `handleToolCall` in `FileSystemContext`. The server reconstructs the file system from scratch on each request using the files sent in the request body.

`src/lib/provider.ts` — Returns the real `anthropic("claude-haiku-4-5")` model when `ANTHROPIC_API_KEY` is set, otherwise falls back to `MockLanguageModel` which generates a hard-coded Counter/Form/Card component without making any API calls.

`src/lib/contexts/chat-context.tsx` — Wraps Vercel AI SDK's `useChat`, wires `onToolCall` to `handleToolCall`, and sends the current serialized file system in the request body on every submission.

### Live preview
`src/lib/transform/jsx-transformer.ts` — Babel standalone transforms JSX/TSX files into plain JS. `createImportMap` processes all files, creates blob URLs for each, and builds an ES module import map. Third-party packages are resolved via `https://esm.sh/`. Missing local imports get auto-generated placeholder modules.

`src/components/preview/PreviewFrame.tsx` — Renders a sandboxed `<iframe srcdoc>` containing the full HTML+importmap+module script. Entry point is `/App.jsx` by default, with fallback search for `App.tsx`, `index.jsx`, etc. Re-renders on every `refreshTrigger` change.

### Auth & persistence
`src/lib/auth.ts` — JWT-based sessions using `jose`. Tokens stored in `httpOnly` cookies, 7-day expiry. Marked `server-only` — never import on the client.

`src/middleware.ts` — Protects `/[projectId]` routes; unauthenticated users are redirected.

`prisma/schema.prisma` — SQLite DB with two models: `User` (email + bcrypt password) and `Project` (stores serialized messages + file system JSON). Prisma client is output to `src/generated/prisma/`. Reference this file anytime you need to understand the structure of data stored in the database.

### Page routing
- `/` (`src/app/page.tsx`) — Home page, anonymous or authenticated users can start generating
- `/[projectId]/` — Loads a saved project for authenticated users, redirects to `/` if not found

`src/app/main-content.tsx` — Shared layout rendered by both pages; contains the three-panel UI (chat / code editor / preview) using `react-resizable-panels`.

### Prompt caching
The system prompt (`src/lib/prompts/generation.tsx`) is injected with `cacheControl: { type: "ephemeral" }` to enable Anthropic prompt caching and reduce latency on multi-turn conversations.

## Code style

Use comments sparingly. Only add a comment when the code is complex or the intent is non-obvious. Do not write comments that restate what the code already says.
