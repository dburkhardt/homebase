# Sub-Agents Specification

## Overview

Sub-agents enable parallel execution of independent subtasks. The main agent orchestrates by spawning workers, distributing work, and aggregating results.

## Use Cases

### 1. Batch File Processing

**Task**: "Process all images in /photos to grayscale"

**Plan**:
1. List images in /photos
2. Spawn 5 sub-agents
3. Distribute 10 images per sub-agent
4. Each sub-agent processes its batch
5. Aggregate results

**Speedup**: 5x (50 images in time of 10)

### 2. Parallel API Calls

**Task**: "Fetch data from 20 APIs and combine results"

**Plan**:
1. Spawn 20 sub-agents (or rate-limited pool)
2. Each fetches from one API
3. Main agent combines responses

### 3. Multi-File Analysis

**Task**: "Analyze code quality across all TypeScript files"

**Plan**:
1. Find all .ts files
2. Spawn sub-agents (1 per file or batched)
3. Each runs linter, type checker
4. Main agent summarizes findings

## Architecture

```
┌────────────────────────────────────────┐
│      Main Agent (Orchestrator)        │
│  ┌──────────────────────────────────┐ │
│  │ 1. Identify parallel work        │ │
│  │ 2. Spawn sub-agents              │ │
│  │ 3. Distribute tasks              │ │
│  │ 4. Monitor progress              │ │
│  │ 5. Aggregate results             │ │
│  └──────────────────────────────────┘ │
└────────────────┬───────────────────────┘
                 │
      ┌──────────┼──────────┐
      │          │          │
      ▼          ▼          ▼
  ┌──────┐  ┌──────┐  ┌──────┐
  │Sub-  │  │Sub-  │  │Sub-  │
  │Agent │  │Agent │  │Agent │
  │  1   │  │  2   │  │  3   │
  └──────┘  └──────┘  └──────┘
```

## Implementation

### Sub-Agent Pool

```typescript
class SubAgentPool {
  private agents: SubAgent[] = []
  private maxConcurrency: number
  
  constructor(maxConcurrency = 5) {
    this.maxConcurrency = maxConcurrency
  }
  
  async spawn(count: number): Promise<SubAgent[]> {
    const toSpawn = Math.min(count, this.maxConcurrency)
    
    for (let i = 0; i < toSpawn; i++) {
      const agent = await this.createSubAgent(i)
      this.agents.push(agent)
    }
    
    return this.agents
  }
  
  private async createSubAgent(id: number): Promise<SubAgent> {
    // Each sub-agent is a separate process/thread
    const worker = new Worker('./agent/worker.ts')
    
    return {
      id,
      worker,
      status: 'idle',
      execute: (task) => this.executeOnWorker(worker, task),
    }
  }
  
  async executeAll<T>(
    tasks: Task[],
    onProgress?: (result: T) => void
  ): Promise<T[]> {
    const results: T[] = []
    const queue = [...tasks]
    
    // Start initial batch
    const workers = this.agents.map(agent => 
      this.processQueue(agent, queue, results, onProgress)
    )
    
    await Promise.all(workers)
    return results
  }
  
  private async processQueue<T>(
    agent: SubAgent,
    queue: Task[],
    results: T[],
    onProgress?: (result: T) => void
  ): Promise<void> {
    while (queue.length > 0) {
      const task = queue.shift()
      if (!task) break
      
      agent.status = 'running'
      const result = await agent.execute(task)
      agent.status = 'idle'
      
      results.push(result)
      onProgress?.(result)
    }
  }
  
  async shutdown(): Promise<void> {
    await Promise.all(
      this.agents.map(a => a.worker.terminate())
    )
    this.agents = []
  }
}
```

### Orchestration Tool

New tool available to main agent:

```typescript
const spawnSubAgentsTool: Tool = {
  name: 'spawn_sub_agents',
  description: 'Spawn multiple sub-agents to process tasks in parallel',
  inputSchema: z.object({
    count: z.number().min(1).max(10),
    tasks: z.array(z.object({
      description: string(),
      tool: z.string(),
      args: z.record(z.unknown()),
    })),
  }),
  outputSchema: z.object({
    results: z.array(z.unknown()),
    duration: z.number(),
  }),
  requiresApproval: false,
  async execute(args) {
    const pool = new SubAgentPool(args.count)
    await pool.spawn(args.count)
    
    const results = await pool.executeAll(args.tasks, (result) => {
      // Stream progress to UI
      emitProgress({ type: 'sub_agent_progress', result })
    })
    
    await pool.shutdown()
    return { ok: true, value: { results } }
  },
}
```

### LLM-Driven Parallelization

**Prompt** (planning phase):

```
Analyze this task for parallelization opportunities:
"${taskDescription}"

If the task involves:
1. Processing multiple similar items
2. Making multiple independent API calls
3. Analyzing multiple files

Then suggest using sub-agents:
- How many sub-agents?
- How to partition the work?
- How to combine results?

Output plan with spawn_sub_agents tool calls.
```

**Example Output**:

```json
{
  "steps": [
    {
      "id": "step_1",
      "description": "List all images",
      "tool": "list_directory",
      "args": { "path": "photos", "pattern": "*.jpg" }
    },
    {
      "id": "step_2",
      "description": "Process images in parallel",
      "tool": "spawn_sub_agents",
      "args": {
        "count": 5,
        "tasks": "$step_1.map(file => ({ tool: 'run_code', args: { ... } }))"
      },
      "dependsOn": ["step_1"]
    },
    {
      "id": "step_3",
      "description": "Verify all images processed",
      "tool": "list_directory",
      "args": { "path": "photos/processed" },
      "dependsOn": ["step_2"]
    }
  ]
}
```

## Resource Management

### Concurrency Limits

```typescript
const limits = {
  maxSubAgents: 10,         // Hard limit
  maxPerTask: 5,            // Default for most tasks
  cpuBased: Math.max(2, os.cpus().length - 2), // Leave headroom
}
```

### Memory Management

Each sub-agent shares workspace mount but has isolated:
- Process memory
- LLM context
- Execution logs

Monitor memory usage, scale down if limits approached.

## Progress Aggregation

```typescript
interface AggregatedProgress {
  total: number
  completed: number
  failed: number
  inProgress: number
  results: Array<{ subAgentId: number; result: unknown }>
}

function aggregateProgress(
  subAgents: SubAgent[]
): AggregatedProgress {
  return {
    total: subAgents.length,
    completed: subAgents.filter(a => a.status === 'completed').length,
    failed: subAgents.filter(a => a.status === 'failed').length,
    inProgress: subAgents.filter(a => a.status === 'running').length,
    results: subAgents.map(a => ({ 
      subAgentId: a.id, 
      result: a.result 
    })),
  }
}
```

## Error Handling

### Partial Failure

If some sub-agents fail:
- Continue with successful results
- Report which items failed
- Optionally retry failed items

```typescript
const results = await pool.executeAll(tasks)
const succeeded = results.filter(r => r.ok)
const failed = results.filter(r => !r.ok)

if (failed.length > 0) {
  logger.warn(`${failed.length}/${tasks.length} sub-agent tasks failed`)
  // Optionally retry failed tasks
}

return { 
  ok: true, 
  value: { succeeded, failed } 
}
```

### Complete Failure

If all sub-agents fail:
- Abort execution
- Report detailed error
- Don't spawn more agents

## UI Representation

```
┌─────────────────────────────────────────────────┐
│ Processing 50 images...                         │
│                                                 │
│ ████████████████░░░░░░░░░░░░  42/50 (84%)     │
│                                                 │
│ Sub-Agent 1: ✓ Completed (10 images)           │
│ Sub-Agent 2: ✓ Completed (10 images)           │
│ Sub-Agent 3: ⟳ Processing... (7/10)            │
│ Sub-Agent 4: ⟳ Processing... (5/10)            │
│ Sub-Agent 5: ⏸ Pending (10 images)             │
│                                                 │
│ Estimated time remaining: 32s                   │
└─────────────────────────────────────────────────┘
```

## Future Enhancements

- **Dynamic Scaling**: Spawn more agents if work grows
- **Priority Queue**: Prioritize certain tasks
- **Result Streaming**: Stream partial results as completed
- **Sub-Agent Specialization**: Different agent types for different tasks
