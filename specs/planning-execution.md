# Planning & Execution Specification

## Overview

The planning phase converts user requests into structured, reviewable, and executable plans. The execution phase runs these plans step-by-step with streaming progress and approval gates.

## Plan Structure

```typescript
interface Plan {
  id: string
  taskId: string
  version: number // Increments when plan is modified
  steps: Step[]
  estimatedDuration?: number // seconds
  createdAt: string
}

interface Step {
  id: string
  order: number
  description: string
  tool: string
  args: Record<string, unknown>
  dependsOn: string[] // Step IDs that must complete first
  status: StepStatus
  result?: StepResult
}

type StepStatus = 
  | 'pending'
  | 'blocked'    // Waiting for dependencies
  | 'ready'      // Dependencies satisfied
  | 'running'
  | 'completed'
  | 'failed'
  | 'skipped'    // User skipped or approval denied

interface StepResult {
  ok: boolean
  value?: unknown
  error?: string
  duration: number // milliseconds
  timestamp: string
}
```

## Plan Generation

### LLM Prompt

```typescript
const planningPrompt = `
You are an AI assistant helping a user accomplish a task in their workspace.

User's request: "${task.description}"

Workspace contents:
${workspaceSnapshot}

Available tools:
${toolDefinitions.map(t => `- ${t.name}: ${t.description}`).join('\n')}

Generate a detailed plan as a JSON array of steps. Each step must:
1. Have a clear, specific description
2. Use exactly one tool from the available tools
3. Include all required arguments for that tool
4. Specify dependencies on previous steps (by step ID)
5. Be achievable with the given tools

Guidelines:
- Break complex tasks into smaller, manageable steps
- Use parallel execution where possible (steps with no dependencies)
- Request approval for destructive actions (deletes, overwrites)
- Include verification steps (e.g., read file after writing)
- Consider edge cases and error handling

Output format:
{
  "steps": [
    {
      "id": "step_1",
      "description": "...",
      "tool": "...",
      "args": {...},
      "dependsOn": []
    },
    ...
  ],
  "estimatedDuration": 120
}
`
```

### Plan Validation

```typescript
function validatePlan(plan: Plan): Result<void> {
  // Check for cycles in dependencies
  const graph = buildDependencyGraph(plan.steps)
  if (hasCycle(graph)) {
    return { ok: false, error: 'Plan contains circular dependencies' }
  }
  
  // Validate each step
  for (const step of plan.steps) {
    // Tool exists?
    const tool = toolRegistry.get(step.tool)
    if (!tool) {
      return { ok: false, error: `Unknown tool: ${step.tool}` }
    }
    
    // Args valid?
    const validation = tool.inputSchema.safeParse(step.args)
    if (!validation.success) {
      return { ok: false, error: `Invalid args for ${step.tool}` }
    }
    
    // Dependencies exist?
    for (const depId of step.dependsOn) {
      if (!plan.steps.find(s => s.id === depId)) {
        return { ok: false, error: `Unknown dependency: ${depId}` }
      }
    }
  }
  
  return { ok: true, value: undefined }
}
```

## Plan Review UI

User sees plan before execution:

```
┌─────────────────────────────────────────────────┐
│ Plan Review                                     │
├─────────────────────────────────────────────────┤
│                                                 │
│ ☐ 1. Read config.json                          │
│      Tool: read_file                            │
│      Args: { path: "config.json" }              │
│                                                 │
│ ☐ 2. Update version to 2.0.0                   │
│      Tool: write_file                           │
│      Args: { path: "config.json", ... }         │
│      Requires approval ⚠️                       │
│                                                 │
│ ☐ 3. Create backup                             │
│      Tool: write_file                           │
│      Args: { path: "config.backup.json", ... }  │
│                                                 │
├─────────────────────────────────────────────────┤
│ [Edit Plan]  [Start Execution]  [Cancel]       │
└─────────────────────────────────────────────────┘
```

**Actions**:
- Edit step (modify args, description)
- Reorder steps (drag & drop, respecting dependencies)
- Skip step (mark as skipped)
- Add step (insert new step)
- Regenerate plan (ask LLM to revise)

## Execution Engine

```typescript
class ExecutionEngine {
  async execute(plan: Plan): Promise<Result<ExecutionSummary>> {
    const context = new ExecutionContext()
    
    while (hasRemainingSteps(plan)) {
      // Find all ready steps (dependencies satisfied, not completed)
      const readySteps = plan.steps.filter(s => 
        s.status === 'pending' && 
        allDependenciesSatisfied(s, plan)
      )
      
      if (readySteps.length === 0) {
        // Deadlock or all steps done
        break
      }
      
      // Execute ready steps in parallel
      await Promise.all(
        readySteps.map(step => this.executeStep(step, context))
      )
    }
    
    return this.summarize(plan, context)
  }
  
  private async executeStep(
    step: Step,
    context: ExecutionContext
  ): Promise<void> {
    step.status = 'running'
    this.emitProgress({ step, status: 'running' })
    
    try {
      // Check approval
      if (requiresApproval(step)) {
        const approved = await this.requestApproval(step)
        if (!approved) {
          step.status = 'skipped'
          step.result = { ok: false, error: 'User denied approval' }
          return
        }
      }
      
      // Execute tool
      const tool = toolRegistry.get(step.tool)!
      const result = await tool.execute(step.args)
      
      step.status = result.ok ? 'completed' : 'failed'
      step.result = result
      
      // Store output in context
      context.set(step.id, result.value)
      
      this.emitProgress({ step, status: step.status, result })
      
    } catch (error) {
      step.status = 'failed'
      step.result = { ok: false, error: String(error) }
      this.emitProgress({ step, status: 'failed', error })
    }
  }
}
```

## Progress Streaming

```typescript
interface ProgressEvent {
  taskId: string
  stepId: string
  status: StepStatus
  progress?: number // 0-100 for long-running steps
  result?: StepResult
  logs?: LogEntry[]
}

// tRPC subscription
const progressSubscription = trpc.task.subscribeProgress.useSubscription({
  taskId,
}, {
  onData: (event: ProgressEvent) => {
    updateUI(event)
  }
})
```

## Execution Context

Store intermediate results for use in later steps:

```typescript
class ExecutionContext {
  private store = new Map<string, unknown>()
  
  set(stepId: string, value: unknown): void {
    this.store.set(stepId, value)
  }
  
  get(stepId: string): unknown | undefined {
    return this.store.get(stepId)
  }
  
  // Resolve references in step args
  resolveArgs(args: Record<string, unknown>): Record<string, unknown> {
    return mapValues(args, value => {
      if (typeof value === 'string' && value.startsWith('$')) {
        // Reference to previous step's output
        const stepId = value.slice(1)
        return this.get(stepId)
      }
      return value
    })
  }
}
```

**Example**:
```json
{
  "steps": [
    {
      "id": "step_1",
      "tool": "read_file",
      "args": { "path": "data.json" }
    },
    {
      "id": "step_2",
      "tool": "write_file",
      "args": {
        "path": "processed.json",
        "content": "$step_1" // Reference to step_1's output
      }
    }
  ]
}
```

## Error Recovery

### Strategy 1: Skip and Continue

For non-critical steps, continue execution:

```typescript
if (step.result?.ok === false && !step.critical) {
  step.status = 'failed'
  // Mark dependent steps as 'blocked' but continue with independent steps
  continue
}
```

### Strategy 2: Retry

For transient failures (network, timeouts):

```typescript
const maxRetries = 3
for (let i = 0; i < maxRetries; i++) {
  const result = await executeStep(step)
  if (result.ok) break
  if (i < maxRetries - 1) await sleep(1000 * (i + 1))
}
```

### Strategy 3: Ask LLM

For unexpected failures, ask LLM for recovery plan:

```typescript
const recoveryPrompt = `
Step failed: ${step.description}
Error: ${step.result.error}

Suggest one of:
1. Retry with different args
2. Skip and continue
3. Insert new recovery steps
4. Abort execution

Respond with recovery plan.
`
```

## Future Enhancements

- **Conditional Steps**: `if (condition) then step`
- **Loops**: `for each file in directory, do...`
- **Human-in-the-Loop**: Pause for user input mid-execution
- **Plan Templates**: Save and reuse plans for common tasks
