# VM Isolation Specification

## Overview

Homebase runs the agent in an isolated Linux VM using macOS Virtualization Framework. This provides strong security boundaries while enabling controlled workspace access.

## Requirements

### VM Lifecycle
- Boot on first task execution
- Keep running between tasks (reduce latency)
- Shutdown on app quit or prolonged idle
- Health monitoring (detect crashes, restart if needed)

### Resource Limits
- CPU: 2 cores (configurable)
- RAM: 4GB (configurable)
- Disk: 20GB (agent runtime, tools, cache)
- Network: Outbound only to LLM API + allowlist

### Communication
- Host-to-VM RPC for control plane
- File sharing via 9P/VirtFS mount
- No SSH, no network exposure

## Architecture

### VM Stack

**Host** (macOS):
- Virtualization.framework
- VM configuration and lifecycle
- 9P filesystem server

**Guest** (Linux):
- Minimal Alpine or Ubuntu Server
- Node.js runtime for agent
- Standard POSIX tools (bash, python3)
- No GUI, no unnecessary services

### Boot Sequence

1. **VM Image Preparation** (build time):
   - Create base disk image with OS + runtime
   - Bundle in `app.asar.unpacked/vm/`
   - Read-only base, per-session overlay

2. **VM Boot** (runtime):
   - Load VM configuration (CPU, RAM, disk)
   - Attach workspace mount (9P)
   - Start VM via Virtualization.framework
   - Wait for health check (agent RPC ready)

3. **Agent Initialization** (in VM):
   - Mount workspace at `/mnt/workspace`
   - Connect to host RPC server
   - Load tools and LLM client
   - Signal ready

### RPC Protocol

**Transport**: Unix domain socket (host) → virtio-vsock (VM)

**Messages**:
```typescript
// Host → VM
type HostMessage = 
  | { type: 'execute_task', taskId: string, plan: Plan }
  | { type: 'cancel_task', taskId: string }
  | { type: 'health_check' }

// VM → Host
type VMMessage =
  | { type: 'task_progress', taskId: string, progress: Progress }
  | { type: 'task_complete', taskId: string, result: Result }
  | { type: 'approval_request', request: ApprovalRequest }
  | { type: 'health_ok' }
```

## Implementation

### Virtualization.framework Integration

```typescript
import { Virtualization } from '@node-swift/virtualization' // hypothetical

class VMManager {
  private vm?: VirtualMachine
  
  async boot(): Promise<Result<void>> {
    const config = new VirtualMachineConfiguration()
    
    // CPU
    config.cpuCount = 2
    
    // Memory
    config.memorySize = 4 * 1024 * 1024 * 1024 // 4GB
    
    // Disk (read-only base + writable overlay)
    const diskImage = path.join(app.getAppPath(), 'vm/homebase-agent.img')
    config.storageDevices = [
      new VirtioBlockDevice(diskImage, readOnly: true),
      new VirtioBlockDevice(overlayPath, readOnly: false),
    ]
    
    // Network (outbound only)
    config.networkDevices = [new VirtioNetworkDevice({
      nat: true,
      allowlist: ['api.nvidia.com'],
    })]
    
    // 9P mount
    config.directorySharingDevices = [
      new VirtioFileSystemDevice({
        tag: 'workspace',
        directory: workspacePath,
      })
    ]
    
    this.vm = new VirtualMachine(config)
    await this.vm.start()
    
    // Wait for agent to signal ready
    await this.waitForHealthCheck()
  }
  
  async shutdown(): Promise<void> {
    await this.vm?.stop()
  }
}
```

### Health Monitoring

```typescript
async function monitorHealth() {
  setInterval(async () => {
    const response = await sendToVM({ type: 'health_check' })
    if (!response || response.type !== 'health_ok') {
      logger.error('VM health check failed')
      await restartVM()
    }
  }, 30_000) // Check every 30s
}
```

## Security Model

### Network Isolation
- Outbound connections allowlisted (LLM API, specific domains)
- No inbound connections
- No VM-to-VM communication
- No direct internet access (DNS filtered)

### File System Isolation
- Only workspace mounted (read/write)
- Host filesystem not accessible
- VM disk isolated (cannot access host)

### Process Isolation
- Agent runs as non-root user in VM
- No sudo/privilege escalation
- Resource limits enforced by VM

### Audit & Monitoring
- All RPC messages logged
- File operations audited
- Network connections logged
- Process execution tracked

## Performance Considerations

- **Boot Time**: 2-5 seconds (acceptable for first task)
- **Memory Overhead**: ~4GB for VM + host buffers
- **Disk I/O**: 9P adequate for typical file operations
- **Keep-Alive**: VM stays running between tasks (avoid re-boot)

## Platform Notes

### macOS Virtualization.framework
- **Pros**: Native, efficient, Apple Silicon optimized
- **Cons**: macOS-only, requires macOS 14+
- **Status**: Primary target for v1

### Future: Cross-Platform (Docker/Firecracker)
- **Docker**: Cross-platform but weaker isolation
- **Firecracker**: Strong isolation but more complex setup
- **Status**: Post-v1 consideration

## Known Limitations

1. **macOS-only**: Virtualization.framework is macOS-specific
2. **Apple Silicon**: Intel support possible but not optimized
3. **Disk Size**: 20GB VM image increases app download size
4. **Boot Latency**: 2-5s initial boot (mitigated by keep-alive)

## Future Enhancements

- GPU passthrough for ML workloads
- Multiple VM profiles (lightweight vs. full)
- Snapshot/restore for faster boot
- VM image updates (security patches)
