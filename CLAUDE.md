# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates React+Tailwind code that renders in a sandboxed iframe.

## Commands

```bash
npm run setup        # Install deps, generate Prisma client, run migrations
npm run dev          # Dev server (Turbopack) on port 3000
npm run dev:daemon   # Dev server in background (logs to logs.txt)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest tests
npm run db:reset     # Force-reset database
npx prisma migrate dev   # Run new migrations
npx prisma generate      # Regenerate Prisma client after schema changes
```

To run a single test file: `npx vitest run <path>`

## Environment

Copy `.env.example` to `.env`. `ANTHROPIC_API_KEY` is optional — if absent, the app uses `MockLanguageModel` so the full UI works without API calls.

## Architecture

### Data Flow

User chat message → `/api/chat` (route.ts) → Vercel AI SDK `streamText` with Claude → AI invokes tools (`str_replace_editor`, `file_manager`) → tools mutate `VirtualFileSystem` → updated FS is streamed back → `FileSystemContext` re-renders preview.

### Key Abstractions

**`VirtualFileSystem`** (`src/lib/file-system.ts`) — in-memory file system (no disk writes). Supports create/view/replace/insert/delete. Serialized to JSON for database storage. Shared between frontend context and AI tools.

**AI Tools** (`src/lib/tools/`) — two tools the model can call:
- `str_replace_editor`: single-file create/view/str_replace/insert operations
- `file_manager`: bulk file operations (create/delete multiple files)

**Preview system** (`src/lib/transform/`) — Babel standalone transforms JSX at runtime; `createPreviewHTML()` builds a sandboxed iframe document with an import map so component files reference each other as local URLs.

**Provider** (`src/lib/provider.ts`) — `getLanguageModel()` returns Claude Haiku 4.5 when `ANTHROPIC_API_KEY` is set, otherwise `MockLanguageModel`.

### State Management

Two React contexts:
- `ChatContext` — messages, streaming state, `useChat` from Vercel AI SDK
- `FileSystemContext` — `VirtualFileSystem` instance, active file, view mode (preview/code)

### Auth

JWT sessions via `jose`, bcrypt passwords, HttpOnly cookies (7-day expiry). Server actions in `src/actions/index.ts` handle sign-up/sign-in/sign-out. Anonymous users get a localStorage work tracker (`anon-work-tracker.ts`) to recover work on registration.

### Database

Prisma + SQLite (`prisma/dev.db`). Two models: `User` and `Project`. Projects store serialized chat messages and the full VirtualFileSystem snapshot.
