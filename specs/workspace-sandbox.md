# Workspace Sandbox Specification

## Overview

The workspace sandbox provides isolated file system access to the agent. Users select a folder, which is mounted into the VM at `/mnt/workspace`. The agent can only access files within this mount point.

## Requirements

### Workspace Selection
- User browses and selects folder via native dialog
- Selected path stored in database
- Recent workspaces list for quick access
- Validation: path must exist, be readable, and be a directory

### File Watching
- Detect external changes to workspace files
- Sync changes into VM (bidirectional)
- Debounce rapid changes (100ms)
- Ignore `.homebase/` metadata directory

### Path Validation
- All agent file operations validated against workspace root
- Prevent `..` escapes and symlink attacks
- Normalize paths before comparison
- Reject absolute paths outside workspace

### Permissions
- Read/write access within workspace
- No execution of binaries by default (requires approval)
- Destructive operations (delete, overwrite) require approval

## Architecture

### Main Process (Workspace Manager)

```typescript
interface WorkspaceManager {
  // Selection
  selectWorkspace(): Promise<Result<Workspace>>
  getRecentWorkspaces(): Promise<Workspace[]>
  
  // Mounting
  mountWorkspace(workspaceId: string): Promise<Result<void>>
  unmountWorkspace(workspaceId: string): Promise<Result<void>>
  
  // Watching
  startWatching(workspaceId: string): void
  stopWatching(workspaceId: string): void
  
  // Validation
  validatePath(workspacePath: string, targetPath: string): boolean
}
```

### VM Integration

**Mount Point**: `/mnt/workspace` (read/write)

**9P/VirtFS**: Use 9P filesystem protocol for host-to-VM sharing
- Supported by Linux kernel (no custom drivers)
- Proper POSIX semantics
- Performance adequate for file-based work

## Implementation Notes

### File Watching (chokidar)

```typescript
const watcher = chokidar.watch(workspacePath, {
  ignored: /(^|[\/\\])\.homebase($|[\/\\])/,
  persistent: true,
  ignoreInitial: true,
  awaitWriteFinish: {
    stabilityThreshold: 100,
    pollInterval: 100,
  },
})

watcher
  .on('add', path => syncToVM('add', path))
  .on('change', path => syncToVM('change', path))
  .on('unlink', path => syncToVM('unlink', path))
```

### Path Validation

```typescript
function validatePath(workspaceRoot: string, targetPath: string): boolean {
  const normalized = path.normalize(targetPath)
  const resolved = path.resolve(workspaceRoot, normalized)
  return resolved.startsWith(workspaceRoot)
}
```

## Safety Guarantees

1. **Isolation**: Agent cannot access files outside workspace
2. **Approval Gates**: Destructive operations require user confirmation
3. **Audit Log**: All file operations logged with timestamps
4. **Rollback**: Future enhancement - file versioning for undo

## Future Enhancements

- Version control integration (git status, diffs)
- File size limits (prevent filling disk)
- Quota management (max workspace size)
- Backup/snapshot before destructive operations
