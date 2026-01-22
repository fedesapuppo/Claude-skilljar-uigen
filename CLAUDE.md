# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude AI (via Vercel AI SDK) to generate React components that render in real-time in a browser preview. The entire file system is virtual (no files written to disk) and components are transformed and executed using Babel and blob URLs.

## Development Commands

### Setup and Installation
```bash
npm run setup          # Install dependencies, generate Prisma client, run migrations
npm install            # Install dependencies only
npx prisma generate    # Generate Prisma client
npx prisma migrate dev # Run database migrations
```

### Development
```bash
npm run dev            # Start Next.js dev server with Turbopack
npm run dev:daemon     # Start dev server in background (logs to logs.txt)
npm run build          # Build for production
npm start              # Start production server
```

### Testing and Linting
```bash
npm run lint           # Run ESLint
npm test               # Run all tests with Vitest
npm test -- <path>     # Run specific test file
npm test -- --watch    # Run tests in watch mode
```

### Database
```bash
npm run db:reset       # Reset database (drops all data, re-runs migrations)
npx prisma studio      # Open Prisma Studio UI
```

## Architecture

### Virtual File System

The core innovation is `VirtualFileSystem` (src/lib/file-system.ts), which maintains all user-generated code in memory:

- Files stored in a Map with tree structure (FileNode)
- Operations: create, read, update, delete, rename files and directories
- Serialization to/from JSON for database persistence
- No actual files written to disk
- All paths are absolute and start with `/`

The virtual file system is exposed to the AI via two tools:
1. `str_replace_editor` - Create files, replace strings, insert lines (src/lib/tools/str-replace.ts)
2. `file_manager` - Rename and delete files/folders (src/lib/tools/file-manager.ts)

### JSX Transformation Pipeline

The transformation pipeline (src/lib/transform/jsx-transformer.ts) converts user code into executable modules:

1. **Transform JSX/TSX to JS**: Uses Babel standalone to compile React/TypeScript code
2. **Create blob URLs**: Each transformed file becomes a blob URL
3. **Build import map**: Maps file paths to blob URLs, supports `@/` alias for root
4. **Generate preview HTML**: Creates standalone HTML with import map and styles
5. **Handle CSS imports**: Extracts and injects CSS into preview

Key functions:
- `transformJSX()` - Transforms a single file
- `createImportMap()` - Creates import map from all files
- `createPreviewHTML()` - Generates the preview HTML with error boundary

### AI Integration

Chat API endpoint (src/app/api/chat/route.ts):
- Uses Vercel AI SDK's `streamText` with tool calling
- Streams responses back to client
- Saves conversation and file state to database on completion
- System prompt in src/lib/prompts/generation.tsx sets AI behavior
- Uses prompt caching (Anthropic's ephemeral cache control)

The AI is instructed to:
- Create React components with Tailwind CSS styling
- Always have `/App.jsx` as the root entry point
- Use `@/` alias for local imports
- Keep responses brief

### React Context Architecture

Two main contexts provide application state:

1. **FileSystemContext** (src/lib/contexts/file-system-context.tsx)
   - Wraps VirtualFileSystem instance
   - Provides CRUD operations for files
   - Handles tool calls from AI responses
   - Triggers re-renders on file changes

2. **ChatContext** (src/lib/contexts/chat-context.tsx)
   - Manages conversation messages
   - Interfaces with AI chat API
   - Handles streaming responses

### Authentication

JWT-based session management (src/lib/auth.ts):
- Sessions stored in HTTP-only cookies
- 7-day expiration
- Optional authentication (anonymous users supported)
- Middleware (src/middleware.ts) protects auth routes

Anonymous users can create projects tracked via localStorage (src/lib/anon-work-tracker.ts).

### Database Schema

Prisma with SQLite (prisma/schema.prisma):
- **User**: Basic auth (email/password)
- **Project**: Stores serialized messages and file system data
- Generated client output to `src/generated/prisma`

### Component Structure

- **src/components/chat/**: Chat interface, message rendering with markdown support
- **src/components/editor/**: Monaco-based code editor and file tree
- **src/components/preview/**: Preview iframe that renders generated components
- **src/components/auth/**: Sign up/sign in forms and dialog
- **src/components/ui/**: Radix UI primitives (shadcn/ui style)

### Preview Rendering

Preview iframe (src/components/preview/PreviewFrame.tsx):
1. Gets all files from file system
2. Transforms files and creates import map
3. Generates preview HTML with embedded import map
4. Sets iframe srcDoc to render
5. Includes error boundary and syntax error display

The preview uses:
- ES modules with import maps
- Tailwind CDN for styling
- React/ReactDOM from esm.sh CDN
- Blob URLs for user-generated code

## Key Patterns

### Path Resolution
- All file paths are absolute: `/App.jsx`, `/components/Button.jsx`
- Import alias `@/` maps to root: `import Button from '@/components/Button'`
- Import map handles all path variations (with/without extensions, with/without leading slash)

### File System Synchronization
- FileSystemContext calls `triggerRefresh()` after mutations
- Components using file system depend on `refreshTrigger` to re-render
- Tool calls from AI are processed via `handleToolCall()` which updates state

### Error Handling
- Syntax errors caught during JSX transformation
- Errors displayed in preview with styled error messages
- Runtime errors caught by React error boundary in preview

### Mock Mode
When `ANTHROPIC_API_KEY` is not set, the app uses a mock provider that returns static code instead of making API calls. This allows testing without API access.

## Testing

Tests use Vitest with React Testing Library:
- File system operations (src/lib/__tests__/file-system.test.ts)
- JSX transformation (src/lib/transform/__tests__/jsx-transformer.test.ts)
- React components (src/components/**/__tests__/)
- Context providers (src/lib/contexts/__tests__/)

Test environment configured in vitest.config.mts with jsdom.

## Configuration

- **TypeScript**: Target ES2017, path alias `@/*` maps to `src/*`
- **Tailwind CSS**: v4, configured in postcss.config.mjs
- **Next.js**: App router, React 19, Turbopack in dev
- **ESLint**: Next.js config
