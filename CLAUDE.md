# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY` to use real Claude generation. Without it, the app falls back to a `MockLanguageModel` that returns canned static components — useful for dev without an API key.

## Commands

```bash
# Install dependencies, generate Prisma client, and run migrations
npm run setup

# Development server (with Turbopack)
npm run dev

# Development server in background (logs to logs.txt)
npm run dev:daemon

# Production build
npm run build

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/editor/__tests__/file-tree.test.tsx

# Run tests in watch mode
npx vitest

# Lint
npm run lint

# Reset and re-run all DB migrations
npm run db:reset
```

## Architecture

### Request Flow

1. User types in `ChatInterface` → `ChatProvider` (wraps `useAIChat` from Vercel AI SDK) sends a POST to `/api/chat/route.ts`
2. The API route reconstructs a `VirtualFileSystem` from the serialized `files` payload, then calls `streamText` with two tools: `str_replace_editor` and `file_manager`
3. As the stream returns tool calls, the client-side `onToolCall` handler in `ChatContext` calls `handleToolCall` from `FileSystemContext`, which applies mutations to the in-memory `VirtualFileSystem`
4. Every mutation triggers a `refreshTrigger` increment, which causes `PreviewFrame` to re-render the iframe

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is the core data structure — an in-memory tree of `FileNode` objects keyed by absolute path. It is **not persisted to disk**. The entire FS is serialized to JSON and sent with every chat request as `files`, then reconstructed server-side.

For authenticated users with a `projectId`, the API saves updated messages and serialized FS state to the `Project` model in SQLite via Prisma on stream finish.

### Preview Pipeline

`PreviewFrame` calls `createImportMap` + `createPreviewHTML` from `src/lib/transform/jsx-transformer.ts`:
- Each `.jsx/.tsx` file is compiled with `@babel/standalone` (in the browser)
- Compiled output is stored as `Blob` URLs
- An import map is injected into a `<script type="importmap">` in the iframe `srcdoc`
- Third-party packages are resolved via `https://esm.sh/`; missing local imports get placeholder stub modules
- Tailwind CSS is loaded via CDN in the preview iframe

The entry point is `/App.jsx` by default; fallbacks are `/App.tsx`, `/index.jsx`, `/index.tsx`, `/src/App.jsx`.

### AI Tools (server-side)

Both tools operate on the same `VirtualFileSystem` instance created per request:

- **`str_replace_editor`** (`src/lib/tools/str-replace.ts`): Claude's primary editing tool. Commands: `view`, `create`, `str_replace`, `insert`. (`undo_edit` is unsupported.)
- **`file_manager`** (`src/lib/tools/file-manager.ts`): Handles `rename` and `delete`.

### LLM Provider

`src/lib/provider.ts` exports `getLanguageModel()`:
- If `ANTHROPIC_API_KEY` is set → uses `claude-haiku-4-5` via `@ai-sdk/anthropic`
- If not set → falls back to `MockLanguageModel`, which returns canned static components (Counter, Card, ContactForm). The mock is useful for local dev without an API key.

### Generation Prompt Rules

Defined in `src/lib/prompts/generation.tsx`. Key constraints Claude must follow when generating:
- Every project needs a root `/App.jsx` with a default export
- Always start new projects by creating `/App.jsx` first
- Use Tailwind CSS for styling (no hardcoded styles, no HTML files)
- Import local files using the `@/` alias (e.g., `@/components/Button`)

### Auth

JWT-based sessions via `jose` (`src/lib/auth.ts`). Middleware (`src/middleware.ts`) handles protected routes. Anonymous usage is fully supported — work is tracked in `src/lib/anon-work-tracker.ts` and can be claimed after sign-up.

### Database

Prisma with SQLite (`prisma/dev.db`). Two models: `User` and `Project`. `Project.messages` and `Project.data` store JSON as strings. The Prisma client is generated to `src/generated/prisma`.

### Context Hierarchy

```
FileSystemProvider        ← owns VirtualFileSystem instance + refreshTrigger
  └── ChatProvider        ← owns useAIChat, calls handleToolCall on tool use
        └── MainContent   ← layout: resizable chat | preview/code panels
```
