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

### 1. Initial Authentication Flow

```
┌─────┐                ┌────────┐              ┌───────┐
│ CLI │                │ Server │              │  App  │
└──┬──┘                └───┬────┘              └───┬───┘
   │                       │                       │
   │ POST /v1/auth/request │                       │
   ├──────────────────────>│                       │
   │ (challenge)            │                       │
   │<──────────────────────┤                       │
   │                       │                       │
   │ Display QR Code       │                       │
   │ (contains challenge)  │                       │
   │                       │                       │
   │                       │  Scan QR, sign        │
   │                       │  POST /v1/auth/response
   │                       │<──────────────────────┤
   │                       │  (signed challenge)   │
   │                       │                       │
   │ GET /v1/auth/poll     │                       │
   ├──────────────────────>│                       │
   │ (auth token)           │                       │
   │<──────────────────────┤                       │
   │                       │                       │
```

### 2. Session Synchronization Flow

```
┌─────┐                ┌────────┐              ┌───────┐
│ CLI │                │ Server │              │  App  │
└──┬──┘                └───┬────┘              └───┬───┘
   │                       │                       │
   │ Connect WebSocket     │   Connect WebSocket   │
   ├──────────────────────>│<──────────────────────┤
   │                       │                       │
   │ POST /v1/sessions     │                       │
   ├──────────────────────>│                       │
   │ (encrypted session)   │                       │
   │                       │ "new-session" event   │
   │                       ├──────────────────────>│
   │                       │                       │
   │ Send message (E2E)    │                       │
   ├──────────────────────>│ Forward (encrypted)   │
   │                       ├──────────────────────>│
   │                       │                       │
   │                       │ Send reply (E2E)      │
   │       Forward         │<──────────────────────┤
   │<──────────────────────┤                       │
   │                       │                       │
```

### 3. Encryption Model

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

1. **Start Server**:
   ```bash
   cd Server
   yarn install
   yarn db      # Start PostgreSQL
   yarn redis   # Start Redis
   yarn dev     # Start server
   ```

2. **Start CLI**:
   ```bash
   cd CLI/thalamus-cli
   yarn install
   yarn build
   # Set environment for local server
   THALAMUS_SERVER_URL=http://localhost:3005 ./bin/thalamus.mjs
   ```

3. **Start App**:
   ```bash
   cd App
   yarn install
   # Create .env with: EXPO_PUBLIC_THALAMUS_SERVER_URL=http://localhost:3005
   yarn start
   yarn ios     # or android/web
   ```

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

### CLI
- Use yarn (not npm)
- Tests required before publishing
- Binary is built before running integration tests
- See CLI/thalamus-cli/CLAUDE.md for details

### Server
- Use yarn (not npm)
- 4 spaces for indentation
- Absolute imports with `@/` prefix
- Prefer functions over classes
- Write tests for utility functions
- Never create migrations manually
- See Server/CLAUDE.md for details

### App
- Use yarn (not npm)
- 4 spaces for indentation
- Always use `t()` for user-visible strings (i18n)
- Run `yarn typecheck` after all changes
- Use Unistyles for styling
- Wrap pages in `memo`
- See App/CLAUDE.md for details

## Related Links

- **App Repository**: https://github.com/et-ai-labs/Thalamus-App
- **CLI Repository**: https://github.com/et-ai-labs/Thalamus-CLI
- **Server Repository**: https://github.com/et-ai-labs/Thalamus-Server
- **Production Server**: https://api-thalamus.etai.app
- **Web App**: https://thalamus.etai.app

## License

All components: MIT License
