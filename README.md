# Homebase

**Desktop agent for file-based work.** Homebase gives an LLM agent controlled access to a user-selected folder to read, write, and execute code within a sandboxed VM environment.

## Overview

Homebase is an Electron desktop app where users describe outcomes in natural language, and an AI agent plans, executes, and reports progress with the ability to steer mid-task. All work happens in a user-selected workspace folder mounted into an isolated VM.

**LLM Backend**: NVIDIA LLM Gateway (Nemotron or similar with strong tool-use support)

## Features

- **Workspace Isolation**: Agent can only access files in user-selected folder
- **Task Planning**: Agent decomposes user requests into reviewable plans
- **Live Execution**: Stream progress, approve destructive actions
- **VM Sandboxing**: Agent runs in isolated Linux VM (macOS Virtualization Framework)
- **Parallel Sub-Agents**: Spawn workers for independent subtasks
- **Knowledge Base**: Markdown projections of extracted knowledge

## Requirements

- **macOS 14.0+** (Sonoma or later)
- **Apple Silicon** (M1/M2/M3/M4)
- **Node.js 20+**
- **pnpm 9+**

## Quick Start

```bash
# Install dependencies
pnpm install

# Run in development mode
pnpm dev

# Build for production
pnpm build

# Run tests
pnpm test

# Run E2E tests
pnpm test:e2e
```

## Project Structure

```
homebase/
├── electron/         # Main process (VM mgmt, workspace, IPC)
├── agent/            # Agent runtime (runs in VM)
├── src/              # React renderer (UI)
├── specs/            # Component specifications
├── scripts/          # Build, dev, release scripts
├── docs/             # Additional documentation
└── drizzle/          # Database schema
```

See [AGENTS.md](AGENTS.md) for architecture details and development guidelines.

## Documentation

- **[AGENTS.md](AGENTS.md)** - Architecture, commands, and development patterns
- **[SPEC.md](SPEC.md)** - Product specification and implementation phases
- **[specs/](specs/)** - Component-specific specifications

## Development

This project uses modern tooling:

- **TypeScript** - Strict mode with comprehensive type safety
- **Biome** - Fast linting and formatting
- **Vitest** - Unit testing
- **Playwright** - E2E testing
- **Drizzle ORM** - Type-safe database access
- **tRPC** - Type-safe IPC communication

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Electron Main                        │
│  ┌───────────┐  ┌───────────┐  ┌───────────────────┐   │
│  │ Window    │  │ Workspace │  │ Run Orchestrator  │   │
│  │ Manager   │  │ Manager   │  │ (approval gates)  │   │
│  └───────────┘  └───────────┘  └───────────────────┘   │
│                        │                                │
│                        ▼                                │
│              ┌─────────────────┐                        │
│              │   VM Manager    │                        │
│              │  (sandbox host) │                        │
│              └────────┬────────┘                        │
└───────────────────────┼─────────────────────────────────┘
                        │
          ┌─────────────▼─────────────┐
          │     Linux VM (isolated)   │
          │  ┌─────────────────────┐  │
          │  │   Agent Runtime     │  │
          │  │  - LLM calls        │  │
          │  │  - Tool execution   │  │
          │  │  - Sub-agent pool   │  │
          │  └─────────────────────┘  │
          │                           │
          │  /mnt/workspace (mounted) │
          └───────────────────────────┘
```

## License

MIT
