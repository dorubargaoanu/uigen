# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server with Turbopack (http://localhost:3000)
npm run build        # Production build
npm run test         # Run all tests (Vitest)
npx vitest run src/components/chat  # Run tests in a specific directory
npx vitest run src/lib/transform    # Run a specific test file/folder
npm run lint         # ESLint
npm run setup        # First-time setup: install deps + Prisma generate + migrate
npm run db:reset     # Reset and re-migrate the SQLite database
npx prisma studio    # Browse/edit database in browser UI
npx prisma migrate dev --name <name>  # Create a new migration
```

Set `ANTHROPIC_API_KEY` in `.env` to use real Claude. Without it, the app falls back to a `MockLanguageModel` that returns hardcoded example components.

## Architecture

UIGen is an AI-powered React component generator. The user types a prompt, the AI generates/edits files in a virtual file system, and the output is compiled and previewed live in the browser.

### Request Flow

1. User submits prompt → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. API calls `streamText` (Vercel AI SDK) with the Anthropic provider (`src/lib/provider.ts`)
3. The model uses two tools to write code:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — create/overwrite files or do targeted string replacements
   - `file_manager` (`src/lib/tools/file-manager.ts`) — delete files/directories
4. Tool call results stream back to the client via `useChat` (`@ai-sdk/react`)
5. `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) intercepts tool calls and applies changes to the in-memory virtual file system
6. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) compiles JSX on-the-fly using `@babel/standalone` and renders it

### State Management

Two React contexts carry all runtime state:
- **`FileSystemContext`** — owns the virtual file system tree, tool call processing, and serialization to JSON for persistence
- **`ChatContext`** — owns chat messages and drives the `useChat` hook; delegates file changes to `FileSystemContext`

### Authentication

JWT sessions via `jose` stored in an `httpOnly` cookie (7-day expiry). Auth logic lives in `src/lib/auth.ts`. Server actions in `src/actions/index.ts` handle sign-up (bcrypt passwords) and sign-in. The middleware (`src/middleware.ts`) only protects `/api/projects` and `/api/filesystem` routes — the main `/api/chat` endpoint is open.

Anonymous users can generate components but their work is ephemeral (tracked client-side in `src/lib/anon-work-tracker.ts`). Authenticated users get their projects saved to SQLite via Prisma.

### Database

SQLite with Prisma. Schema (`prisma/schema.prisma`):
- `User` — id, email, hashed password
- `Project` — id, name, optional userId (anonymous projects have no user), `messages` (JSON string), `data` (JSON string — serialized virtual file system)

Generated client is output to `src/generated/prisma` (not the default location).

### Virtual File System

`src/lib/file-system.ts` implements an in-memory tree. The AI writes all generated code here; nothing touches the real filesystem at runtime. The tree serializes to a flat node list for database storage.

### Provider / Model

`src/lib/provider.ts` exports either a real Anthropic provider (Claude Haiku 4.5) or a `MockLanguageModel` depending on whether `ANTHROPIC_API_KEY` is set. The mock implements `LanguageModelV1` directly and returns static tool calls generating one of three example components.

### UI Layout

Three-panel resizable layout in `src/app/main-content.tsx`:
- Left (35%): `ChatInterface` — prompt input + message history
- Right (65%): tabs
  - **Preview**: `PreviewFrame` live rendering
  - **Code**: `FileTree` (30%) + `CodeEditor` (Monaco, 70%)

### Path Alias

`@/*` maps to `src/*` (configured in `tsconfig.json`).

## Git & GitHub

Repository: https://github.com/dorubargaoanu/uigen

- Always commit with clean, descriptive messages after meaningful changes
- Always push to GitHub after committing — remote is the source of truth and enables easy reverts
- GitHub Actions workflow for Claude integration is at `.github/workflows/claude.yml`
- Use `@claude` in any PR or issue comment to trigger Claude GitHub Actions

```bash
git add <files>
git commit -m "descriptive message"
git push
```

## Testing

Tests use Vitest + jsdom + `@testing-library/react`. Test files live alongside their component in `__tests__/` subdirectories. The vitest config is `vitest.config.mts`.
