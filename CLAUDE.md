# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server at http://localhost:3000 (Turbopack)
npm run build        # Production build
npm run lint         # ESLint via Next.js
npm run test         # Vitest unit tests
npm run db:reset     # Drop and recreate database schema
```

Add `ANTHROPIC_API_KEY` to `.env` for real AI generation; without it, a `MockLanguageModel` is used.

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, Claude generates code, and the result renders instantly in an iframe.

### Core Data Flow

1. User sends a message → `/src/app/api/chat/route.ts` streams a response via Vercel AI SDK + Anthropic
2. Claude calls tools (`str_replace_editor`, `file_manager`) to create/edit files in the virtual FS
3. File changes propagate through `FileSystemContext` → `PreviewFrame` re-renders the iframe

### Virtual File System (`/src/lib/file-system.ts`)

All project files live in memory as a `VirtualFileSystem` object — no disk writes. It serializes to/from JSON for database persistence. The `FileSystemContext` (`/src/lib/contexts/file-system-context.tsx`) exposes this state to the whole app.

### AI Tools (`/src/lib/tools/`)

Two tools are exposed to Claude:
- **`str_replace_editor`** (`str-replace.ts`): view, create, str_replace, insert operations on files
- **`file_manager`** (`file-manager.ts`): rename and delete files

The system prompt is in `/src/lib/prompts/generation.tsx`.

### Live Preview (`/src/components/preview/PreviewFrame`)

Renders the virtual FS in a sandboxed iframe. Uses Babel (`/src/lib/transform/jsx-transformer.ts`) client-side to transpile JSX. Auto-detects entry point (`/App.jsx`, `/App.tsx`, etc.) and stubs missing imports with placeholder modules.

### Authentication

JWT sessions stored in HTTP-only cookies (`/src/lib/auth.ts`). Server actions in `/src/actions/` handle sign-up, sign-in, sign-out, and project CRUD. Anonymous users can generate without signing in — projects only persist for authenticated users.

### Database

SQLite via Prisma (`prisma/schema.prisma`). Two models:
- **User**: email + bcrypt-hashed password
- **Project**: name, optional userId, `messages` (JSON chat history), `data` (JSON serialized VirtualFileSystem)

### Key Contexts

- `FileSystemContext` — manages virtual FS state and syncs to DB
- `ChatContext` (`/src/lib/contexts/chat-context.tsx`) — manages chat messages and AI streaming state

### Path Alias

`@/*` maps to `./src/*` (configured in `tsconfig.json`).
