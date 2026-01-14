# 🧠 Thalamus

This monorepo unifies the three components of Thalamus, a system for remote access to Claude Code and Codex sessions with end-to-end encryption.

## What This Repo Contains

Thalamus is split into three independent repositories organized as git submodules:

### CLI (`./CLI/`)
Node.js wrapper around the official Claude Code CLI tool. Spawns Claude processes, intercepts I/O, encrypts data, and syncs it to the server via WebSocket. Shows QR codes for mobile device pairing.

**Tech:** Node.js, TypeScript, Ink (React for CLI), Socket.io, TweetNaCl

### Server (`./Server/`)
Fastify backend that relays encrypted messages between CLI and App. Zero-knowledge design—it stores encrypted blobs it cannot decrypt. Handles auth, session management, push notifications, and sharing.

**Tech:** Fastify, PostgreSQL (Prisma), Redis, Socket.io

### App (`./App/`)
React Native mobile/web client for viewing and interacting with Claude sessions remotely. Scans QR codes for auth, receives real-time updates, decrypts messages locally.

**Tech:** React Native, Expo, Socket.io, TweetNaCl, MMKV

## Repository Structure

```
Thalamus/
├── CLI/                    # Thalamus-CLI submodule
├── Server/                 # Thalamus-Server submodule
├── App/                    # Thalamus-App submodule
├── CLAUDE.md               # Architecture docs and dev guide
├── SETUP.md                # VM deployment instructions
├── docker-compose.yaml     # Combined deployment config
└── README.md               # This file
```

## How It Works Together

```
┌─────────┐         ┌────────┐         ┌─────┐
│   CLI   │◄───────►│ Server │◄───────►│ App │
└─────────┘         └────────┘         └─────┘
    │                    │
    ├─ Wraps Claude      ├─ PostgreSQL (sessions, users)
    ├─ Encrypts I/O      ├─ Redis (pub/sub)
    ├─ WebSocket sync    ├─ MinIO (file storage)
    └─ QR auth           └─ Push notifications

All data is encrypted end-to-end. Server only sees encrypted blobs.
```

## Data Flow Example

1. **User types in CLI** → CLI encrypts input → Server relays to App
2. **Claude responds** → CLI encrypts output → Server relays to App
3. **User approves permission on phone** → App encrypts response → Server relays to CLI
4. **CLI forwards to Claude** → Process continues

The server never sees plaintext. Encryption keys live only on CLI and App.

## Quick Start

**For deployment:** See [SETUP.md](./SETUP.md) for VM deployment with docker-compose.

**For development:** See [CLAUDE.md](./CLAUDE.md) for architecture details and local development setup.

**Working with submodules:** Each component (CLI/Server/App) is a separate git repository. When you make changes in a submodule, you must commit in both the submodule and the parent repo to track the updated commit reference.

## Submodule Links

- [Thalamus CLI](https://github.com/Et-Ai-Labs/Thalamus-CLI)
- [Thalamus Server](https://github.com/Et-Ai-Labs/Thalamus-Server)
- [Thalamus App](https://github.com/Et-Ai-Labs/Thalamus-App)
