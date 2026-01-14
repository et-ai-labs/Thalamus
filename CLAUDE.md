# Thalamus Monorepo Architecture

This document provides an overview of the Thalamus monorepo architecture and how the three submodules interact to enable remote access to Claude Code and Codex sessions.

## System Overview

Thalamus is a distributed system that allows you to control Claude Code and Codex from mobile devices with end-to-end encryption. The architecture consists of three main components organized as git submodules:

```
Thalamus/
├── CLI/           # Command-line wrapper for Claude Code/Codex
├── Server/        # Backend synchronization server
└── App/           # Mobile/web client application
```

## Architecture Components

### 1. CLI (Thalamus-CLI)

**Repository**: https://github.com/Et-Ai-Labs/Thalamus-CLI
**Technology**: Node.js, TypeScript, Ink (React for CLI)
**Package**: `@et-ai-labs/thalamus`

**Purpose**: Wraps Claude Code and Codex CLI to enable remote access and synchronization.

**Key Features**:
- Spawns and manages Claude Code/Codex processes
- Intercepts and forwards stdin/stdout between local and remote sessions
- Displays QR codes for mobile device authentication
- Manages daemon process for background operation
- Implements MCP (Model Context Protocol) bridge for permission forwarding

**Key Components**:
- `bin/thalamus.mjs` - Main CLI entry point
- `bin/thalamus-mcp.mjs` - MCP bridge for permission forwarding
- Socket.io client for real-time communication with Server
- End-to-end encryption using tweetnacl

**Commands**:
- `thalamus` - Start a Claude Code session with mobile access
- `thalamus codex` - Start Codex mode
- `thalamus auth` - Manage authentication
- `thalamus connect` - Store AI vendor API keys in cloud
- `thalamus notify` - Send push notifications to devices
- `thalamus daemon` - Manage background service
- `thalamus doctor` - System diagnostics

### 2. Server (Thalamus-Server)

**Repository**: https://github.com/Et-Ai-Labs/Thalamus-Server
**Technology**: Fastify, PostgreSQL (Prisma), Redis, Socket.io
**Hosted Instance**: `https://api-thalamus.etai.app` (free to use)

**Purpose**: Zero-knowledge synchronization server that relays encrypted data between CLI and App.

**Key Features**:
- End-to-end encrypted message relay (server cannot decrypt)
- Real-time WebSocket synchronization via Socket.io
- Cryptographic authentication (public key signatures, no passwords)
- Multi-device session management
- Push notifications (encrypted content)
- Session sharing with granular access control (view/edit/admin)
- Public link generation for read-only access
- Horizontal scaling support via Redis pub/sub

**Architecture**:
```
sources/
├── app/              # Application entry points
├── apps/api/         # API server with routes
├── modules/          # Reusable modules (eventbus, lock, media)
├── services/         # Core services (pubsub)
├── storage/          # Database and caching utilities
└── utils/            # Low-level utilities
```

**Database**: PostgreSQL with Prisma ORM
- User accounts (public key based)
- Sessions (encrypted)
- Machines (devices)
- Access control (sharing permissions)
- Push notification tokens

**Key Principles**:
- **Zero Knowledge**: Server stores encrypted blobs it cannot decrypt
- **Privacy First**: No analytics, tracking, or data mining
- **Minimal Surface**: Only essential features for secure sync
- **Idempotent Operations**: All API operations handle retries gracefully

### 3. App (Thalamus-App)

**Repository**: https://github.com/Et-Ai-Labs/Thalamus-App
**Technology**: React Native (Expo), TypeScript, Socket.io
**Platforms**: iOS, Android, Web (web is secondary)

**Purpose**: Mobile and web client for accessing Claude Code/Codex sessions remotely.

**Key Features**:
- QR code-based authentication with CLI
- Real-time session viewing and interaction
- Push notifications for permission requests and errors
- End-to-end encryption (tweetnacl)
- Multi-device support with instant switching
- Session sharing and collaboration
- File browser and code viewing
- Internationalization (en, ru, pl, es)

**Architecture**:
```
sources/
├── app/              # Expo Router screens
│   ├── (app)/        # Authenticated app screens
│   └── (auth)/       # Authentication flow
├── auth/             # Authentication logic
├── components/       # Reusable UI components
├── sync/             # Real-time sync engine with encryption
├── hooks/            # Custom React hooks
├── text/             # Internationalization
└── utils/            # Utility functions
```

**Tech Stack**:
- React Native 0.81 (new architecture)
- Expo SDK 54
- Expo Router v6 (file-based navigation)
- Unistyles (theming and styling)
- Socket.io (WebSocket communication)
- TweetNaCl (encryption)
- MMKV (storage)

## Data Flow

**End-to-End Encryption**: All sensitive data is encrypted on the client (CLI or App) before transmission.

- **Key Generation**: Each device generates keypairs locally using tweetnacl
- **Session Keys**: Unique encryption keys per session, never stored on server
- **Server Role**: Relays encrypted blobs without ability to decrypt
- **Public Key Auth**: Users authenticate with cryptographic signatures

**Encrypted Data**:
- Session messages and history
- File contents
- User input and AI responses
- Metadata (session names, descriptions)

**Unencrypted Data** (Server needs to see):
- User IDs (public keys)
- Session IDs
- Access control lists
- Timestamps
- Machine online status

## Component Interaction Patterns

### CLI ↔ Server
- **Protocol**: HTTPS (REST API) + WebSocket (Socket.io)
- **Authentication**: Bearer tokens (obtained via challenge-response)
- **Endpoints**:
  - `/v1/auth/*` - Authentication
  - `/v1/sessions` - Session management
  - `/v1/machines` - Machine registration and status
  - `/v1/notify` - Push notifications
- **WebSocket Events**: Bi-directional message relay

### App ↔ Server
- **Protocol**: HTTPS + WebSocket (Socket.io)
- **Authentication**: Same token-based system as CLI
- **Endpoints**: Same REST API as CLI
- **WebSocket Events**:
  - Receive session updates
  - Send user input
  - Permission request handling

### CLI ↔ Claude Code/Codex
- **Protocol**: Process spawning, stdin/stdout interception
- **Integration**: Wraps official Claude CLI
- **MCP Bridge**: Forwards Model Context Protocol permissions to mobile

## Deployment Architecture

### Local Development
```
[CLI Process] ←→ [Local Server] ←→ [Mobile App]
                 (Docker: PostgreSQL + Redis)
```

### Production
```
[CLI Process] ←→ [Cloud Server] ←→ [Mobile App(s)]
  (User's      (api-thalamus.    (iOS/Android/Web)
   Machine)      etai.app)
                 ↓
           [PostgreSQL]
           [Redis Cluster]
           [S3/MinIO]
```

## Security Model

1. **Zero-Knowledge Server**: Server cannot decrypt user data
2. **Cryptographic Auth**: No passwords; public key signatures only
3. **E2E Encryption**: TweetNaCl (NaCl crypto library)
4. **Transport Security**: HTTPS/WSS (TLS)
5. **Session Isolation**: Each session has unique encryption keys
6. **Granular Permissions**: Sharing with view/edit/admin levels
7. **Audit Trails**: Optional consent-based access logging

## Development Workflow

### Working with Submodules

**Making changes to a submodule**:
```bash
# Navigate to submodule
cd CLI  # or Server, or App

# Make changes, commit, and push
git add .
git commit -m "Your change"
git push origin main

# Update parent repo to track new commit
cd ..
git add CLI
git commit -m "Update CLI submodule"
git push
```

**Pulling updates**:
```bash
# Pull parent repo changes
git pull

# Update all submodules
git submodule update --remote --merge
```

### Running the Full Stack Locally

#### Prerequisites
- Node.js 20+
- Yarn (not npm)
- Docker (for Server dependencies)

#### 1. Start Server

```bash
cd Server
yarn install

# Start PostgreSQL and Redis via Docker
docker compose up -d postgres redis minio minio-init

# Run database migrations
yarn migrate

# Start development server
yarn dev
# Server will run on http://localhost:3005
```

#### 2. Start CLI

```bash
cd CLI/thalamus-cli
yarn install

# Build the CLI
yarn build

# Run with local server
THALAMUS_SERVER_URL=http://localhost:3005 ./bin/thalamus.mjs

# Or start daemon mode
THALAMUS_SERVER_URL=http://localhost:3005 ./bin/thalamus.mjs daemon start
```

**Note:** CLI logs are written to `~/.thalamus-dev/logs/` to avoid interfering with Claude's terminal output.

#### 3. Start App

```bash
cd App
yarn install

# Create .env file
echo "EXPO_PUBLIC_THALAMUS_SERVER_URL=http://localhost:3005" > .env

# Start Expo dev server
yarn start

# Run on specific platform
yarn ios      # iOS simulator
yarn android  # Android emulator
yarn web      # Web browser
```

**Note:** For web development, the app will run on http://localhost:8081

## Environment Variables

### CLI
- `THALAMUS_SERVER_URL` - Server endpoint (default: https://api-thalamus.etai.app)
- `THALAMUS_WEBAPP_URL` - Web app URL (default: https://thalamus.etai.app)
- `THALAMUS_HOME_DIR` - Data directory (default: ~/.thalamus)
- `THALAMUS_EXPERIMENTAL` - Enable experimental features

### Server
- `DATABASE_URL` - PostgreSQL connection string
- `REDIS_URL` - Redis connection string
- `S3_*` - S3/MinIO configuration for file storage
- `DANGEROUSLY_LOG_TO_SERVER_FOR_AI_AUTO_DEBUGGING` - Enable remote logging

### App
- `EXPO_PUBLIC_THALAMUS_SERVER_URL` - Server endpoint

## Key Technologies

| Component | Framework | Language | Database | Real-time | Encryption |
|-----------|-----------|----------|----------|-----------|------------|
| CLI | Node.js/Ink | TypeScript | N/A | Socket.io Client | tweetnacl |
| Server | Fastify | TypeScript | PostgreSQL | Socket.io Server | N/A (relay only) |
| App | React Native/Expo | TypeScript | MMKV | Socket.io Client | tweetnacl |

## Contributing Guidelines

### Common Rules (All Repositories)

These rules apply to CLI, Server, and App:

**Environment:**
- **Package Manager:** Always use `yarn`, never `npm`
- **Node Version:** Node.js 20+
- **TypeScript:** Strict mode enabled - all code must be properly typed

**Code Style:**
- **Indentation:** 4 spaces (not 2, not tabs)
- **Prefer functions over classes** - Functional programming patterns
- **Named exports preferred** over default exports
- **All imports at the top** - Never import modules mid-code
- **Descriptive variable names** with auxiliary verbs (e.g., `isLoading`, `hasError`)

**Testing:**
- Write tests for new functionality
- Test files colocated with source files
- Descriptive test names and proper async handling

**File Management:**
- Do what has been asked; nothing more, nothing less
- NEVER create files unless absolutely necessary
- ALWAYS prefer editing existing files
- NEVER proactively create documentation files

**Git & Commits:**
- **No Attribution:** Never include Claude, AI, or bot attribution in commits/PRs
- Write clear, concise commit messages
- Follow conventional commit format when applicable

---

### CLI-Specific (Thalamus-CLI)

**Import Style:**
- Use `@/` alias for src imports: `import { logger } from '@/ui/logger'`

**Code Principles:**
- **Strict typing required** - "I despise untyped code"
- **JSDoc comments:** Comprehensive file-level and function-level documentation

**Do NOT:**
- Create small unnecessary getters/setters
- Use excessive `if` statements (design better control flow)

**Error Handling:**
- Graceful error handling with proper error messages
- Use try-catch blocks with specific error logging
- Abort controllers for cancellable operations

**Logging:**
- All debugging through file logs (`~/.thalamus-dev/logs/`)
- Console output only for user-facing messages
- Never interfere with Claude's terminal UI

**Testing:**
- Unit tests using Vitest
- No mocking - tests make real API calls when needed
- Test files: `.test.ts` suffix

**Commands:**
```bash
yarn build          # Build the CLI
yarn test           # Run tests
./bin/thalamus.mjs  # Run CLI locally
```

**Key Files:**
- `src/api/` - Server communication and encryption
- `src/claude/` - Claude Code integration
- `src/ui/` - User interface components
- `bin/thalamus.mjs` - CLI entry point

---

### Server-Specific (Thalamus-Server)

**Import Style:**
- Use `@/` prefix for absolute imports: `import "@/utils/log"`

**Code Style:**
- **Avoid enums** - Use maps instead
- **Prefer interfaces** over types

**Project Structure:**
```
sources/
├── app/       # Application entry points and business logic
├── modules/   # Reusable modules (eventbus, lock, media)
├── services/  # Core services (pubsub)
├── storage/   # Database and caching utilities
└── utils/     # Low-level utilities
```

**Database (Prisma):**
- **NEVER create migrations manually** - Only run `yarn generate`
- Use `inTx` to wrap operations in transactions
- Use `afterTx` to send events after transaction commits
- For complex fields, use `Json` type

**API Development:**
- Routes in `/sources/apps/api/routes`
- Use Fastify with Zod for type-safe route definitions
- Always validate inputs using Zod
- **Idempotency required** - All operations must handle retries gracefully

**Writing Utilities:**
1. Name file and function identically
2. Keep functions modular and focused
3. **Write tests BEFORE implementation**
4. Always write documentation

**Writing Modules:**
- Bigger than utilities, abstract away complexity
- No application-specific logic
- Can depend on other modules
- Use for: external service integration, library abstraction, related function groups

**Writing Actions:**
- Create dedicated files in `@sources/app/` subfolders
- Name format: `[entity][Action]` (e.g., `friendAdd`, `sessionDelete`)
- Add documentation comment explaining logic
- Only return essential data, not "just in case"
- Don't add logging unless asked
- Don't run non-transactional operations (like file uploads) in transactions

**Commands:**
```bash
yarn dev       # Start development server
yarn build     # TypeScript type checking
yarn test      # Run tests
yarn migrate   # Run Prisma migrations
yarn generate  # Generate Prisma client
yarn db        # Start local PostgreSQL
yarn redis     # Start local Redis
```

---

### App-Specific (Thalamus-App)

**Import Style:**
- Path alias `@/*` maps to `./sources/*`

**Code Style:**
- **Always wrap pages in `memo`**
- **Put styles at the end** of component/page files

**Internationalization (i18n):**
- **CRITICAL:** Always use `t()` for ALL user-visible strings
- **Check existing keys first** before adding new ones
- **Add to ALL languages** when creating new strings (en, ru, pl, es)
- **Use descriptive key names** (e.g., `newSession.machineOffline`)
- **No hardcoded strings** in JSX
- Language metadata centralized in `sources/text/_all.ts`
- Use **i18n-translator agent** for adding/verifying translations
- Dev pages can skip i18n

**Styling (Unistyles):**
- Always use `StyleSheet.create` from 'react-native-unistyles'
- Provide styles directly to React Native components
- Use `useStyles` hook only for other components (avoid when possible)
- Use function mode when needing theme or runtime access
- Use variants for component state-based styling
- **Never use Unistyles for expo-image** - use classical styling

**Component Guidelines:**
- Use `ItemList` for most UI containers
- Always use `Avatar` component for avatars
- Never use `Alert` module - use `@sources/modal/index.ts` instead
- Always apply layout width constraints from `@/components/layout`
- Use `useThalamusAction` from `@sources/hooks/useThalamusAction.ts` for async operations
- For exclusive async locks, use `AsyncLock` class

**Navigation:**
- Never use custom headers (use default header on all screens)
- Almost never use `Stack.Screen` options in individual pages
- Always use expo-router API, not react-navigation
- Set screen parameters in `_layout.tsx` to avoid layout shifts
- Store app pages in `@sources/app/(app)/`

**Hooks:**
- For non-trivial hooks, create dedicated file in `hooks/` folder
- Add comment explaining logic
- Use `useGlobalKeyboard` for hotkeys (Web only)

**Changelog Management:**
- Always update `/CHANGELOG.md` when adding features/fixes
- Run `npx tsx sources/scripts/parseChangelog.ts` after updates
- Write entries from user's perspective
- Start with verbs (Added, Fixed, Improved, Updated, Removed)
- Include brief summary paragraph for each version
- Automatically parsed during OTA deployments

**Core Principles:**
- Never show loading errors - always retry
- Always sync main data in "sync" class using invalidate
- No backward compatibility unless explicitly stated
- Web is secondary platform - avoid web-specific implementations

**Testing:**
- Testing framework: Jest with jest-expo preset
- Always run `yarn typecheck` after changes

**Commands:**
```bash
yarn start      # Start Expo dev server
yarn typecheck  # Run TypeScript type checking
yarn test       # Run tests in watch mode
yarn ota        # Deploy OTA updates to production
```

**Project Structure:**
```
sources/
├── app/          # Expo Router screens
│   ├── (app)/    # Authenticated screens
│   └── (auth)/   # Auth flow
├── auth/         # Authentication logic
├── components/   # Reusable components
├── sync/         # Real-time sync with encryption
├── hooks/        # Custom React hooks
├── text/         # Internationalization
└── utils/        # Utilities
```

**Key Files:**
- `sources/sync/types.ts` - Core sync protocol types
- `sources/sync/reducer.ts` - Sync state management
- `sources/auth/AuthContext.tsx` - Auth state
- `sources/app/_layout.tsx` - Root navigation

## Related Links

- **App Repository**: https://github.com/et-ai-labs/Thalamus-App
- **CLI Repository**: https://github.com/et-ai-labs/Thalamus-CLI
- **Server Repository**: https://github.com/et-ai-labs/Thalamus-Server
- **Production Server**: https://api-thalamus.etai.app
- **Web App**: https://thalamus.etai.app
