# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Essential Commands

### Development
```bash
npm run dev              # Start dev server with Turbopack
npm run dev:daemon       # Start dev server in background, logs to logs.txt
npm run build            # Build for production
npm run start            # Start production server
```

### Testing
```bash
npm test                 # Run all tests with Vitest
npm test -- path/to/test # Run specific test file
```

### Database
```bash
npm run setup            # Initial setup: install deps, generate Prisma client, run migrations
npm run db:reset         # Reset database (destructive - use with caution)
npx prisma migrate dev   # Create and apply new migration
npx prisma studio        # Open Prisma Studio to browse data
```

### Linting
```bash
npm run lint            # Run ESLint
```

## Architecture Overview

UIGen is an AI-powered React component generator with live preview. The application uses a virtual file system to generate and render components in-browser without writing to disk.

### Core Components

#### Virtual File System (`src/lib/file-system.ts`)
- **VirtualFileSystem class**: In-memory file tree using Maps
- All paths are normalized with leading `/`
- Files are stored as `FileNode` objects with type, name, path, content, and children
- Supports serialization/deserialization for persistence in Prisma database
- Methods: `createFile()`, `createDirectory()`, `deleteFile()`, `rename()`, `serialize()`, `deserializeFromNodes()`

#### Chat API Route (`src/app/api/chat/route.ts`)
- POST endpoint that streams responses using Vercel AI SDK's `streamText()`
- Reconstructs VirtualFileSystem from serialized data sent by client
- Uses two AI tools for file manipulation:
  - `str_replace_editor`: Creates/edits files using string replacement (from `src/lib/tools/str-replace.ts`)
  - `file_manager`: Renames/deletes files and directories (from `src/lib/tools/file-manager.ts`)
- System prompt from `src/lib/prompts/generation.tsx` instructs AI to:
  - Create React components with Tailwind CSS styling
  - Use `/App.jsx` as root entrypoint (must export default React component)
  - Use `@/` import alias for all non-library imports
  - Operate on virtual file system root `/`
- Saves conversation messages and file system state to database on completion
- `maxDuration: 120` for long-running AI operations

#### AI Provider (`src/lib/provider.ts`)
- Uses Claude Haiku 4.5 model when `ANTHROPIC_API_KEY` is provided
- Falls back to `MockLanguageModel` for demo mode (generates static Counter/Form/Card components)
- Mock provider simulates multi-step AI workflow with tool calls

#### JSX Transformer (`src/lib/transform/jsx-transformer.ts`)
- Uses Babel standalone to transpile JSX/TSX to JavaScript in browser
- Handles missing imports by creating placeholder modules
- Strips CSS imports and returns them separately for handling
- Returns `TransformResult` with transformed code, errors, and missing imports

#### Preview Frame (`src/components/preview/PreviewFrame.tsx`)
- Renders generated components in isolated iframe
- Uses Babel to transform JSX files on-the-fly
- Implements custom module resolution for `@/` imports
- Injects Tailwind CSS and React/ReactDOM via CDN

#### Contexts
- **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`): Manages virtual file system state, file operations, and serialization
- **ChatContext** (`src/lib/contexts/chat-context.tsx`): Manages chat messages, streaming, and AI interaction

#### Authentication (`src/lib/auth.ts`)
- JWT-based auth with 7-day sessions stored in httpOnly cookies
- Anonymous users can use app; registered users get project persistence
- Anonymous work tracked separately (`src/lib/anon-work-tracker.ts`)
- Session payload: `userId`, `email`, `expiresAt`

#### Database (Prisma + SQLite)
- **User model**: id, email, password (bcrypt hashed), timestamps
- **Project model**: id, name, userId (nullable for anonymous), messages (JSON string), data (JSON string of file system), timestamps
- Prisma client generated to `src/generated/prisma` (not `node_modules`)
- Server actions in `src/actions/` for project CRUD operations

### Import Conventions

- Use `@/` alias for all internal imports (configured in `tsconfig.json` as alias to `./src`)
- Example: `import { VirtualFileSystem } from '@/lib/file-system'`
- External libraries use standard imports: `import { streamText } from 'ai'`

### Testing

- Vitest with jsdom for React component tests
- Tests in `__tests__` folders alongside source files
- React Testing Library for component tests
- Example test locations:
  - `src/components/chat/__tests__/`
  - `src/lib/__tests__/`
  - `src/lib/contexts/__tests__/`

### UI Components

- Radix UI primitives in `src/components/ui/`
- Tailwind CSS v4 for styling
- shadcn/ui style components with class-variance-authority

### Key Files

- `src/app/api/chat/route.ts`: Main AI chat endpoint
- `src/lib/file-system.ts`: Virtual file system implementation
- `src/lib/transform/jsx-transformer.ts`: JSX/TSX compilation
- `src/components/preview/PreviewFrame.tsx`: Live component preview
- `prisma/schema.prisma`: Database schema
- `.env`: Contains `ANTHROPIC_API_KEY` (optional) and `JWT_SECRET`

## Development Notes

### Working with the Virtual File System
- All file paths in the VFS start with `/` (e.g., `/App.jsx`, `/components/Button.jsx`)
- The VFS is purely in-memory during chat sessions
- Changes persist to database only for authenticated users with projects
- Use FileSystemContext methods to interact with VFS in React components

### AI Tool Development
- AI tools are defined using Vercel AI SDK's `tool()` function with Zod schemas
- Tools receive the VirtualFileSystem instance and perform operations directly
- Tool results are automatically sent back to AI for next iteration
- See `src/lib/tools/` for examples

### Adding New Features
- Preview frame requires React 19 and ReactDOM from CDN (see PreviewFrame.tsx)
- Generated components must export default to be rendered in preview
- System prompt defines AI behavior - modify `src/lib/prompts/generation.tsx` to change generation style

### Environment Variables
- `ANTHROPIC_API_KEY`: Optional, falls back to mock provider if missing
- `JWT_SECRET`: Required for production, defaults to 'development-secret-key'
- `NODE_OPTIONS='--require ./node-compat.cjs'`: Required for Next.js compatibility (see package.json scripts)
