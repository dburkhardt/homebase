# Electron Main Process

## Purpose

The Electron main process is responsible for:
- Application lifecycle management
- Window creation and management
- VM lifecycle (boot, shutdown, health monitoring)
- Workspace folder selection and mounting
- tRPC server hosting (IPC layer)
- System-level operations (file system, OS integration)

## Structure

```
electron/
├── main.ts          # App entry point, window lifecycle
├── preload.ts       # Context bridge, IPC exposure
├── ipc/             # tRPC routers
│   ├── workspace.ts
│   ├── task.ts
│   ├── vm.ts
│   └── settings.ts
├── vm/              # VM management
│   ├── manager.ts   # Boot, shutdown, health checks
│   ├── rpc.ts       # Host-to-VM communication
│   └── config.ts    # VM configuration
└── workspace/       # Workspace operations
    ├── manager.ts   # Folder selection, mounting
    ├── watcher.ts   # File watching (chokidar)
    └── validator.ts # Path validation
```

## Key Patterns

### tRPC Routers

Each router exposes operations to the renderer process:

```typescript
// electron/ipc/workspace.ts
export const workspaceRouter = router({
  select: procedure
    .output(z.object({ path: z.string() }))
    .mutation(async () => {
      const result = await dialog.showOpenDialog({
        properties: ['openDirectory']
      })
      return { path: result.filePaths[0] }
    }),
    
  list: procedure
    .input(z.object({ workspaceId: z.string() }))
    .query(async ({ input }) => {
      // Implementation
    }),
})
```

### VM Manager

Manage VM lifecycle:

```typescript
// electron/vm/manager.ts
class VMManager {
  async boot(): Promise<Result<void>>
  async shutdown(): Promise<Result<void>>
  async executeInVM(command: VMCommand): Promise<Result<unknown>>
  async healthCheck(): Promise<boolean>
}
```

### Workspace Security

Always validate paths before operations:

```typescript
// electron/workspace/validator.ts
function validatePath(workspaceRoot: string, targetPath: string): boolean {
  const normalized = path.normalize(targetPath)
  const resolved = path.resolve(workspaceRoot, normalized)
  return resolved.startsWith(workspaceRoot)
}
```

## Implementation Guidelines

1. **Error Handling**: Use Result types, never throw across IPC boundary
2. **Type Safety**: tRPC provides end-to-end types
3. **Security**: Validate all inputs, especially file paths
4. **Logging**: Use structured logging for debugging
5. **Testing**: Mock VM and filesystem for unit tests

## See Also

- [specs/vm-isolation.md](../specs/vm-isolation.md) - VM architecture
- [specs/workspace-sandbox.md](../specs/workspace-sandbox.md) - Workspace security
- [AGENTS.md](../AGENTS.md) - Development patterns
