# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # First-time setup: install deps, generate Prisma client, run migrations
npm run dev            # Start dev server (Turbopack) at localhost:3000
npm run build          # Production build
npm run lint           # ESLint
npm run test           # Run Vitest (watch mode)
npx vitest run         # Run tests once (no watch)
npx vitest run src/lib/__tests__/file-system.test.ts  # Run a single test file
npm run db:reset       # Force reset SQLite database
npx prisma generate    # Regenerate Prisma client after schema changes
npx prisma migrate dev # Create/apply migrations after schema changes
```

The app works without an `ANTHROPIC_API_KEY` — a mock provider returns static code instead of calling Claude.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat interface, Claude generates code via tool calls, and the result renders in a live preview iframe.

### Core Flow

1. **Chat** (`src/app/api/chat/route.ts`) — Streams AI responses via Vercel AI SDK (`ai` package). The model gets two tools: `str_replace_editor` (create/edit files) and `file_manager` (rename/delete). Up to 40 tool-use steps per request.
2. **Virtual File System** (`src/lib/file-system.ts`) — In-memory file tree (never touches disk). Serialized to/from JSON for database persistence and client-server transfer.
3. **Preview** (`src/components/preview/PreviewFrame.tsx`) — Renders generated components in a sandboxed iframe. Uses Babel (`@babel/standalone`) to compile JSX client-side. Import maps resolve `@/` aliases to virtual file contents.
4. **Persistence** — Projects store messages and file system data as JSON strings in SQLite via Prisma. Anonymous users can work without auth; registered users get project saving.

### State Management

Two React contexts drive the UI:
- **ChatContext** (`src/lib/contexts/chat-context.tsx`) — Chat messages, AI streaming state
- **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`) — Virtual file tree, active file selection

### Layout

Three-panel resizable layout (`src/app/main-content.tsx`):
- Left: Chat interface (`src/components/chat/`)
- Right top: Preview iframe or code editor toggle
- Right bottom (in code view): File tree + Monaco editor (`src/components/editor/`)

### Key Directories

- `src/lib/tools/` — AI tool definitions (str-replace, file-manager)
- `src/lib/prompts/` — System prompts for component generation
- `src/lib/transform/` — Babel-based JSX compilation for preview
- `src/actions/` — Next.js server actions (auth, project CRUD)
- `src/components/ui/` — shadcn/ui primitives (New York style, Tailwind v4)

### Tech Specifics

- **Next.js 15 App Router** with React 19 and server components
- **Path alias**: `@/*` maps to `src/*`
- **Prisma** generates client to `src/generated/prisma/` (not default location)
- **shadcn/ui** configured in `components.json` (New York style, neutral base color, CSS variables)
- **Node compat**: `node-compat.cjs` is required via NODE_OPTIONS to fix Node.js 25+ Web Storage SSR issues
- **Database**: SQLite at `prisma/dev.db`; two models: `User` and `Project`
- **Auth**: JWT (jose) with httpOnly cookies, bcrypt password hashing
