# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It allows users to describe React components via chat, and generates them in real-time using Claude AI. Components are rendered in a live preview using a virtual file system (no files are written to disk).

## Tech Stack

- **Frontend**: Next.js 15 (App Router), React 19, TypeScript
- **Styling**: Tailwind CSS v4, shadcn/ui components
- **Database**: Prisma with SQLite
- **AI**: Anthropic Claude via Vercel AI SDK
- **Code Processing**: Babel Standalone for JSX/TSX transformation
- **Testing**: Vitest with React Testing Library

## Code Style

- Use comments sparingly. Only comment complex code.

## Development Commands

```bash
# Initial setup - install dependencies, generate Prisma client, run migrations
npm run setup

# Start development server (with Turbopack)
npm run dev

# Run development server as background daemon with logs
npm run dev:daemon

# Build for production
npm run build

# Start production server
npm start

# Run tests (Vitest)
npm test

# Lint code
npm run lint

# Reset database (force migration reset)
npm run db:reset

# Regenerate Prisma client (after schema changes)
npx prisma generate

# Create new database migration
npx prisma migrate dev
```

## Architecture

### Virtual File System

The core of UIGen is a client-side virtual file system (`VirtualFileSystem` class in `src/lib/file-system.ts`) that:

- Stores files and directories in-memory using a Map-based tree structure
- Provides file operations: create, read, update, delete, rename
- Serializes/deserializes to/from JSON for persistence in database
- **Important**: All file paths start with `/` and use Unix-style separators
- Supports `@/` import alias that resolves to root directory (`/`)

The file system is managed via React Context (`FileSystemContext` in `src/lib/contexts/file-system-context.tsx`) and provides:
- State management for selected files
- Integration with AI tool calls (file creation, editing, deletion)
- Auto-refresh triggers when files change

### AI Chat System

The chat system uses Vercel AI SDK's `useChat` hook with custom tools:

1. **Chat Route** (`src/app/api/chat/route.ts`):
   - Receives messages and current file system state
   - Uses Claude AI with prompt caching for system prompt
   - Provides two tools to the AI:
     - `str_replace_editor`: Create files, replace strings, insert text
     - `file_manager`: Rename or delete files/folders
   - Automatically saves messages and file state to database for authenticated users

2. **Chat Context** (`src/lib/contexts/chat-context.tsx`):
   - Wraps `useChat` and integrates with file system context
   - Handles tool call responses by updating the virtual file system
   - Tracks anonymous user work in localStorage

3. **Generation Prompt** (`src/lib/prompts/generation.tsx`):
   - All projects must have a root `/App.jsx` file with default export
   - Use `@/` import alias for local files (e.g., `@/components/Calculator`)
   - Style with Tailwind CSS classes, not hardcoded styles
   - No HTML files needed - App.jsx is the entry point

### Live Preview System

The preview system (`src/components/preview/PreviewFrame.tsx` and `src/lib/transform/jsx-transformer.ts`) works by:

1. **Transformation**:
   - Uses Babel Standalone to transform JSX/TSX files to ES modules
   - Handles TypeScript files with `@babel/preset-typescript`
   - Detects and extracts CSS imports from files
   - Collects all import statements and resolves dependencies

2. **Import Map Generation**:
   - Creates an ES import map mapping file paths to blob URLs
   - Maps React imports to `esm.sh` CDN
   - Handles `@/` alias by creating multiple import map entries
   - Third-party packages auto-resolved from `esm.sh`
   - Creates placeholder modules for missing imports

3. **Preview Rendering**:
   - Generates complete HTML document with import map and inline styles
   - Includes Tailwind CSS from CDN
   - Uses React Error Boundary for runtime errors
   - Displays syntax errors with file location and formatted messages
   - Renders preview in sandboxed iframe with `allow-scripts` and `allow-same-origin`

4. **Entry Point Detection**:
   - Looks for `/App.jsx`, `/App.tsx`, `/index.jsx`, `/index.tsx`, or `/src/App.jsx`
   - Falls back to first `.jsx`/`.tsx` file found

### Database Schema

The database schema is defined in `prisma/schema.prisma`. Reference it anytime you need to understand the structure of the database.

```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String   (bcrypt hashed)
  projects  Project[]
}

model Project {
  id        String   @id @default(cuid())
  name      String
  userId    String?  (nullable for anonymous users)
  messages  String   (JSON array of chat messages)
  data      String   (JSON serialized VirtualFileSystem)
  user      User?    @relation
}
```

### Authentication

Authentication is implemented using JWT with the `jose` library:
- Session tokens stored in HTTP-only cookies
- Passwords hashed with bcrypt
- Middleware (`src/middleware.ts`) protects project routes
- Anonymous users can work without auth, with localStorage tracking
- `src/lib/auth.ts` provides `getSession()` and token management

## Testing

Tests use Vitest with React Testing Library:

- Run all tests: `npm test`
- Test files located in `__tests__` directories alongside source files
- Uses `jsdom` environment for component testing
- Path aliases (`@/`) work in tests via `vite-tsconfig-paths`

## File Organization

```
src/
├── app/                    # Next.js App Router pages
│   ├── api/chat/          # AI chat endpoint
│   ├── [projectId]/       # Dynamic project page
│   └── page.tsx           # Homepage
├── components/
│   ├── auth/              # Authentication forms and dialogs
│   ├── chat/              # Chat interface components
│   ├── editor/            # Code editor and file tree
│   ├── preview/           # Preview frame
│   └── ui/                # shadcn/ui components
├── lib/
│   ├── contexts/          # React contexts (file system, chat)
│   ├── tools/             # AI tool implementations
│   ├── transform/         # JSX transformation logic
│   ├── auth.ts            # Authentication utilities
│   ├── file-system.ts     # Virtual file system implementation
│   └── prompts/           # AI system prompts
├── actions/               # Server actions (create/get projects)
└── hooks/                 # Custom React hooks
```

## Key Implementation Details

### File Path Handling
- Always use `/` as root prefix for file paths
- Import alias `@/` maps to root `/` directory
- File system normalizes paths automatically
- Extensions (.jsx, .tsx) optional in imports due to import map variations

### AI Tool Integration
- Tools are defined in `src/lib/tools/` with Zod schemas
- `str_replace_editor` supports: create, str_replace, insert commands
- `file_manager` supports: rename (also moves), delete commands
- Tool results trigger file system context updates via `handleToolCall`

### Preview Limitations
- No Node.js runtime - client-side only
- External packages must be ES modules from esm.sh
- CSS must be imported properly (auto-extracted and inlined)
- Syntax errors prevent preview rendering but show detailed error UI

### Mock Provider
When `ANTHROPIC_API_KEY` is not set:
- Falls back to mock provider (returns static code)
- Limited to 4 max steps instead of 40
- Useful for development without API key

## Environment Variables

Required in `.env`:
```bash
ANTHROPIC_API_KEY=your-api-key-here  # Optional - app works without it
```

## Common Patterns

### Creating a New Component
AI creates files with:
- Root `/App.jsx` as entry point
- Components in `/components/` directory
- Imports use `@/` alias: `import Foo from '@/components/Foo'`
- Tailwind CSS for styling

### Updating File System
Always use context methods, not direct VirtualFileSystem methods:
```ts
const { createFile, updateFile, deleteFile } = useFileSystem();
createFile('/example.jsx', 'content');
```

### Adding New AI Tools
1. Create tool in `src/lib/tools/`
2. Define with Zod schema and execute function
3. Register in `src/app/api/chat/route.ts` tools object
4. Handle in `FileSystemContext.handleToolCall` if needed
