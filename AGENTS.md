# AGENTS.md — Homebase Architecture & Development Guide

Desktop agent for file-based work: LLM-powered task execution in sandboxed VM with controlled workspace access.

**Authoritative docs**: [SPEC.md](SPEC.md) (product spec) · This file (architecture + commands + patterns)

## Quick Reference

| Aspect | Value |
|--------|-------|
| Platform | macOS 14+ (Sonoma), Apple Silicon |
| Languages | TypeScript (strict mode) |
| UI | React + Vite + Tailwind |
| State | Zustand |
| IPC | tRPC over Electron IPC |
| VM | macOS Virtualization Framework |
| Agent | TypeScript agent loop |
| LLM | NVIDIA LLM Gateway |
| DB | SQLite via Drizzle ORM |

## Working Rules

1. **Follow SPEC.md architecture** — implement components per specification
2. **Small diffs, test often** — keep the app functional at all times
3. **Check in frequently** — confirm approach before significant work
4. **TypeScript strict** — no `any`, enable all strict checks
5. **Modern patterns** — async/await, proper error handling, type safety
6. **Document decisions** — update specs when architecture changes
7. **Test coverage** — maintain 70%+ overall, 80%+ for new code

## Commands

**Development**:
```bash
pnpm dev          # Start dev server with hot reload
pnpm build        # Build for production
pnpm build:dev    # Build without packaging
pnpm preview      # Preview production build
```

**Testing**:
```bash
pnpm test              # Run unit tests
pnpm test:ui           # Run tests with UI
pnpm test:e2e          # Run E2E tests
pnpm test:coverage     # Generate coverage report
```

**Code Quality**:
```bash
pnpm lint         # Check linting
pnpm lint:fix     # Fix linting issues
pnpm format       # Format code
pnpm typecheck    # Check TypeScript types
```

**Database**:
```bash
pnpm db:generate  # Generate migrations
pnpm db:migrate   # Run migrations
pnpm db:studio    # Open Drizzle Studio
```

**Utilities**:
```bash
pnpm clean        # Clean build artifacts
```

## Architecture

### Component Overview

```
Electron Main Process
├── Window Manager          # Window lifecycle, state persistence
├── Workspace Manager       # Folder selection, file watching
├── VM Manager              # VM boot, shutdown, health checks
└── Run Orchestrator        # Plan execution, approval gates

VM Environment (Isolated)
├── Agent Runtime           # Main agent loop
│   ├── LLM Client          # NVIDIA Gateway integration
│   ├── Planner             # Task → structured plan
│   └── Sub-agent Pool      # Parallel workers
└── Tool System             # File ops, code execution, web access

React Renderer
├── Workspace View          # Folder selection, workspace info
├── Task Input              # Natural language task description
├── Plan Review             # Interactive plan editor
├── Execution View          # Streaming logs, progress
└── Knowledge Base          # Markdown artifacts viewer
```

### VM Isolation Strategy

**macOS Virtualization Framework** provides:
- Lightweight Linux VM (minimal footprint)
- Host-to-VM RPC for controlled communication
- Workspace mount at `/mnt/workspace` (read/write)
- Network allowlist (LLM API + approved domains)
- No access to host filesystem outside workspace

**Security Model**:
1. VM can only see mounted workspace folder
2. All file operations require path validation
3. Destructive actions pause for user approval
4. Network limited to LLM API and allowlisted domains
5. User can abort execution at any time

### State Management

**Main Process** (Node.js):
- SQLite database (runs, workspaces, settings)
- tRPC routers expose operations to renderer
- VM manager maintains VM lifecycle state

**Renderer Process** (React):
- Zustand stores for UI state
- TanStack Query for server state caching
- React hooks for tRPC mutations/queries

**Agent Runtime** (VM):
- Stateless execution model
- Receives plan, executes steps, reports progress
- No persistent state (all logged to database via RPC)

### IPC Communication

**tRPC Architecture**:
```typescript
// Electron main exposes tRPC routers
const appRouter = {
  workspace: workspaceRouter,    // select, list, watch
  task: taskRouter,              // submit, cancel, stream logs
  vm: vmRouter,                  // boot, shutdown, health
  settings: settingsRouter,      // get, update
}

// Renderer consumes via tRPC client
const trpc = createTRPCReact<AppRouter>()
```

**Benefits**:
- End-to-end type safety
- Auto-generated TypeScript types
- No manual IPC channel definitions
- Built-in error handling

## Directory Structure

### `/electron` - Main Process

**Purpose**: Electron main process, VM management, IPC layer

**Key Files**:
- `main.ts` - App entry, window creation, lifecycle
- `preload.ts` - Context bridge, IPC exposure
- `ipc/` - tRPC routers
- `vm/` - VM boot, RPC, health monitoring
- `workspace/` - Folder mounting, file watching

### `/agent` - Agent Runtime

**Purpose**: Agent loop running inside VM

**Key Files**:
- `loop.ts` - Main agent loop (prompt → plan → execute → observe)
- `tools/` - Tool implementations (file ops, code exec, web)
- `planner.ts` - LLM-based plan generation
- `subagent.ts` - Sub-agent spawning and coordination

### `/src` - React Renderer

**Purpose**: UI layer (React + Tailwind)

**Key Files**:
- `App.tsx` - Main app component
- `components/` - Reusable UI components
- `hooks/` - Custom React hooks
- `stores/` - Zustand state stores

### `/specs` - Component Specifications

**Purpose**: Detailed component design docs

**Files** (per SPEC.md):
- `workspace-sandbox.md` - Workspace isolation
- `vm-isolation.md` - VM architecture
- `agent-runtime.md` - Agent loop design
- `tool-system.md` - Tool definitions
- `planning-execution.md` - Plan generation/execution
- `sub-agents.md` - Parallel execution
- `knowledge-base.md` - Markdown projection

### `/scripts` - Build & Dev Scripts

**Purpose**: Build automation, dev tools

**Files**: TBD by component agents

### `/docs` - Documentation

**Purpose**: Architecture diagrams, guides

**Files**: TBD by component agents

### `/drizzle` - Database

**Purpose**: SQLite schema and migrations

**Key Files**:
- `schema.ts` - Table definitions (runs, workspaces, settings)
- `migrations/` - Generated migration files (gitignored)

## Key Patterns

### Error Handling

```typescript
// Use Result type for operations that can fail
type Result<T, E = Error> = 
  | { ok: true; value: T }
  | { ok: false; error: E }

// Example usage
async function readFile(path: string): Promise<Result<string>> {
  try {
    const content = await fs.readFile(path, 'utf-8')
    return { ok: true, value: content }
  } catch (error) {
    return { ok: false, error: error as Error }
  }
}
```

### VM Communication

```typescript
// Main process initiates VM operations
const result = await vmManager.executeInVM({
  type: 'run_task',
  taskId: '...',
  plan: [...],
})

// VM reports progress via callback
vmManager.onProgress((progress) => {
  // Relay to renderer via tRPC subscription
  progressEmitter.emit('progress', progress)
})
```

### Approval Gates

```typescript
// Destructive operations pause for approval
if (requiresApproval(toolCall)) {
  const approved = await requestApproval({
    tool: toolCall.name,
    args: toolCall.args,
    reason: 'This will delete files',
  })
  
  if (!approved) {
    return { ok: false, error: 'User denied approval' }
  }
}
```

## Known Pitfalls

### VM Lifecycle

- VM boot takes 2-5 seconds — show loading state
- Always shutdown VM on app quit (prevent orphans)
- Health check VM before task execution

### Workspace Security

- Validate all file paths before operations
- Use `path.normalize()` to prevent `..` escapes
- Never allow writes outside workspace mount

### LLM Integration

- Handle rate limits (429) with exponential backoff
- Stream responses for better UX
- Validate tool calls before execution
- Log all LLM interactions for debugging

### Electron Packaging

- Use `extraResources` for VM images
- Test packaged app (dev vs production behavior differs)
- Handle file paths correctly (app.asar limitations)

## Testing Strategy

**Unit Tests** (Vitest):
- Pure functions (planner, tool validators)
- React components (UI logic)
- Database operations (Drizzle queries)

**Integration Tests** (Vitest):
- tRPC routers (main process)
- Agent loop (mocked LLM)
- VM communication (mocked VM)

**E2E Tests** (Playwright):
- Full user flows (select workspace → run task → review output)
- Multi-step tasks
- Approval gates

**Coverage Targets**:
- Overall: 70%+
- Critical paths: 90%+ (VM, agent, file ops)
- New code (PRs): 80%+

## Development Workflow

1. **Component Agents**: Specialized agents implement each component
2. **Spec-Driven**: Read relevant spec before implementing
3. **Incremental**: Build one feature at a time, keep app working
4. **Review Plans**: Show approach before major changes
5. **Test Coverage**: Write tests alongside implementation
6. **Document**: Update AGENTS.md when patterns change

## Reference

**Dependencies**:
- [Electron](https://www.electronjs.org/) - Desktop app framework
- [tRPC](https://trpc.io/) - Type-safe IPC
- [Drizzle ORM](https://orm.drizzle.team/) - Type-safe SQL
- [Biome](https://biomejs.dev/) - Fast linter/formatter
- [Vitest](https://vitest.dev/) - Unit test framework
- [Playwright](https://playwright.dev/) - E2E testing

**Useful Commands**:
- `pnpm why <package>` - Why is a package installed?
- `pnpm outdated` - Check for updates
- `pnpm audit` - Security audit
