# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Common Commands

```bash
# Setup (install dependencies, generate Prisma client, run migrations)
npm run setup

# Development
npm run dev

# Development with daemon logging
npm run dev:daemon

# Build for production
npm build

# Start production server
npm start

# Run tests (Vitest)
npm test

# Run single test file
npx vitest src/components/chat/__tests__/ChatInterface.test.tsx

# Run tests in watch mode
npx vitest --watch

# Run tests with UI
npx vitest --ui

# Lint code
npm run lint

# Reset database (drops all data)
npm run db:reset
```

## Architecture Overview

### Three-Panel Layout
The app has a three-panel architecture:
- **Left Panel**: Chat interface for user prompts and AI responses
- **Right Panel (Preview)**: Live preview of generated React components (default view)
- **Right Panel (Code)**: File tree + code editor for viewing/editing generated files

### Core Data Flow

1. **User Input** → ChatInterface → `/api/chat` (Next.js server action)
2. **AI Processing**: Claude generates JSX code with tool calls to create/modify files
3. **Tool Calls**: Two tools handle file operations:
   - `str_replace_editor`: Create/modify file content (create, str_replace, insert operations)
   - `file_manager`: Rename/delete files
4. **File Operations**: Tool calls invoke `handleToolCall` in FileSystemProvider
5. **State Sync**: FileSystemProvider updates VirtualFileSystem + triggers UI refresh
6. **Preview Rendering**: JSX Transformer converts files to blob URLs via import map + renders in iframe
7. **Persistence**: For authenticated users, projects are saved to Prisma/SQLite

### Key Contexts (Client-Side State)

**FileSystemProvider** (`src/lib/contexts/file-system-context.tsx`)
- Manages in-memory virtual file system (VirtualFileSystem class)
- Handles file CRUD operations
- Processes tool calls from AI (str_replace_editor, file_manager)
- Provides: fileSystem, selectedFile, createFile, updateFile, deleteFile, renameFile, handleToolCall

**ChatProvider** (`src/lib/contexts/chat-context.tsx`)
- Wraps Vercel AI SDK's `useChat` hook
- Serializes fileSystem state and sends to `/api/chat` on each request
- Processes tool calls via `onToolCall` callback
- Tracks anonymous work via `anon-work-tracker`

### Virtual File System (VirtualFileSystem class)

Located in `src/lib/file-system.ts`. Key features:
- Completely in-memory (no disk writes to user's filesystem)
- Tree structure with FileNode (file or directory)
- Normalized paths (always start with `/`, no trailing slashes)
- Methods: createFile, updateFile, deleteFile, rename, readFile, getAllFiles
- Parent directory auto-creation via `createFileWithParents`
- Serialization/deserialization for persistence and API communication

### JSX to Preview Pipeline

`src/lib/transform/jsx-transformer.ts` transforms generated code to preview:
1. **transformJSX**: Uses Babel to transpile JSX → JavaScript (handles TypeScript, React automatic runtime)
2. **createImportMap**: Creates ES module import map with:
   - Local files → blob URLs (for generated components)
   - React/React-DOM → esm.sh CDN
   - Third-party packages → esm.sh CDN
   - @/ alias support for root directory imports
   - Missing imports → placeholder modules (prevents 404 errors)
3. **createPreviewHTML**: Generates complete HTML with:
   - ImportMap script
   - Error boundary + dynamic module loading
   - Tailwind CSS from CDN
   - Syntax error display if Babel fails
4. **PreviewFrame**: Renders HTML in iframe (src/components/preview/PreviewFrame.tsx)

### AI Integration (`src/app/api/chat/route.ts`)

- Uses Vercel AI SDK + Anthropic provider
- System prompt from `src/lib/prompts/generation.tsx` (cached with ephemeral cache control)
- Accepts messages + serialized file system in request body
- Returns streaming response with tool calls
- Saves project state (messages + files) to Prisma after response completes (authenticated users only)
- max 40 AI steps (4 for mock provider without API key)

### Authentication & Projects

- Session-based auth via JWT (see `src/lib/auth.ts`)
- Server actions in `src/actions/`: getUser, getProjects, createProject, getProject
- Anonymous users: state stored in browser localStorage only
- Authenticated users: projects persisted to SQLite via Prisma
- Prisma models: User (with password hashing via bcrypt) + Project (messages/data stored as JSON strings)

### Component Structure

- **UI Components** (`src/components/ui/`): Radix UI primitives (button, dialog, tabs, etc.) + custom resizable panels
- **Chat** (`src/components/chat/`): ChatInterface, MessageList, MessageInput, MarkdownRenderer
- **Editor** (`src/components/editor/`): FileTree (nested file browser), CodeEditor (Monaco editor)
- **Preview** (`src/components/preview/`): PreviewFrame (renders HTML in iframe)
- **Auth** (`src/components/auth/`): SignInForm, SignUpForm, AuthDialog
- **HeaderActions** (`src/components/HeaderActions.tsx`): Export, save, auth buttons

## Testing Strategy

Tests use Vitest + React Testing Library. Key test patterns:
- Component tests with rendering assertions
- Context provider tests with useContext mocks
- FileSystem tests for virtual file operations
- Chat context tests with mocked AI responses

Run all tests with `npm test`. Tests auto-reload in watch mode.

## Important Notes

- **No Disk Writes**: Files stay in-memory only (VirtualFileSystem). User's disk is never touched.
- **Ephemeral Cache**: System prompt uses Anthropic's ephemeral cache for cost savings on repeated requests.
- **Fallback Mode**: If ANTHROPIC_API_KEY is not set, a mock provider returns static placeholder code instead of AI-generated code.
- **Import Aliasing**: `@/` prefix resolves to root directory. Relative and absolute imports both work.
- **Error Boundaries**: Preview iframe has error boundary + syntax error display for debugging.
- **CSS Imports**: CSS files are collected and injected as `<style>` tags; import statements are stripped from transpiled code.
