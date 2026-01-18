# Agent Runtime Specification

## Overview

The agent runtime is the core execution loop running inside the VM. It receives tasks, generates plans, executes tools, and reports progress to the host.

## Architecture

```
┌─────────────────────────────────────┐
│         Agent Loop (VM)             │
│  ┌─────────────────────────────┐   │
│  │  1. Receive Task            │   │
│  │  2. Generate Plan (LLM)     │   │
│  │  3. Execute Steps           │   │
│  │  4. Observe Results         │   │
│  │  5. Adapt/Continue/Complete │   │
│  └─────────────────────────────┘   │
│                │                    │
│                ▼                    │
│  ┌─────────────────────────────┐   │
│  │       Tool System           │   │
│  │  - File operations          │   │
│  │  - Code execution           │   │
│  │  - Web access               │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

## Core Loop

### 1. Task Reception

```typescript
interface Task {
  id: string
  description: string // User's natural language request
  workspace: string   // Path to mounted workspace
  context?: Record<string, unknown> // Optional context
}
```

Agent receives task via RPC from host.

### 2. Plan Generation

```typescript
interface Plan {
  steps: Step[]
  estimatedDuration?: number
  requiredApprovals: string[] // Which steps need approval
}

interface Step {
  id: string
  description: string
  tool: string
  args: Record<string, unknown>
  dependsOn?: string[] // IDs of prerequisite steps
}
```

**LLM Prompt** (planning phase):
```
You are an AI agent with access to a workspace folder. 
The user wants: {task.description}

Available tools: {toolDefinitions}

Generate a step-by-step plan as JSON. Each step should:
1. Have a clear description
2. Specify which tool to use
3. Include all required arguments
4. Mark dependencies on previous steps

Consider:
- Break complex tasks into smaller steps
- Use parallel execution where possible
- Request approval for destructive actions
```

### 3. Step Execution

```typescript
async function executeStep(step: Step): Promise<StepResult> {
  // Check if approval required
  if (requiresApproval(step)) {
    const approved = await requestApproval(step)
    if (!approved) {
      return { ok: false, error: 'User denied approval', skipped: true }
    }
  }
  
  // Execute tool
  const tool = tools[step.tool]
  if (!tool) {
    return { ok: false, error: `Unknown tool: ${step.tool}` }
  }
  
  const result = await tool.execute(step.args)
  
  // Report progress
  await reportProgress({
    taskId: currentTask.id,
    stepId: step.id,
    status: result.ok ? 'completed' : 'failed',
    result,
  })
  
  return result
}
```

### 4. Observation & Adaptation

After each step:
- Collect results (file contents, command output, errors)
- Optionally query LLM: "Should we continue, adapt, or stop?"
- Update plan if needed (insert new steps, skip steps)

**LLM Prompt** (observation phase):
```
Previous step: {step.description}
Result: {result}

Should we:
1. Continue with next step
2. Adapt plan (suggest new steps)
3. Complete (goal achieved)
4. Abort (unrecoverable error)

Respond with decision and reasoning.
```

### 5. Completion

```typescript
interface TaskResult {
  status: 'completed' | 'failed' | 'cancelled'
  outputs: string[] // Paths to generated files
  summary: string   // LLM-generated summary
  logs: LogEntry[]
}
```

## LLM Integration (NVIDIA Gateway)

### Configuration

```typescript
interface LLMConfig {
  endpoint: string // e.g., 'https://integrate.api.nvidia.com/v1'
  apiKey: string
  model: string    // e.g., 'nvidia/nemotron-4-340b-instruct'
  temperature: number
  maxTokens: number
}
```

### Streaming Responses

```typescript
async function* streamCompletion(
  prompt: string,
  options: LLMConfig
): AsyncGenerator<string> {
  const response = await fetch(`${options.endpoint}/chat/completions`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${options.apiKey}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: options.model,
      messages: [{ role: 'user', content: prompt }],
      temperature: options.temperature,
      max_tokens: options.maxTokens,
      stream: true,
    }),
  })
  
  for await (const chunk of response.body) {
    const parsed = parseSSE(chunk)
    if (parsed.choices[0]?.delta?.content) {
      yield parsed.choices[0].delta.content
    }
  }
}
```

### Tool Use

**Structured Output** (JSON mode):
```typescript
const response = await llm.complete({
  prompt: planningPrompt,
  responseFormat: { type: 'json_object' },
  schema: PlanSchema, // Zod schema for validation
})

const plan = PlanSchema.parse(JSON.parse(response.content))
```

## Error Handling

### Retry Logic

```typescript
async function executeWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries = 3
): Promise<Result<T>> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const result = await fn()
      return { ok: true, value: result }
    } catch (error) {
      if (i === maxRetries - 1) {
        return { ok: false, error: error as Error }
      }
      await sleep(1000 * Math.pow(2, i)) // Exponential backoff
    }
  }
  throw new Error('Unreachable')
}
```

### Graceful Degradation

- LLM timeout → Use fallback heuristics
- Tool failure → Skip step, continue if non-critical
- Network error → Pause, retry, or abort

## Performance

- **Latency**: Stream LLM responses for perceived speed
- **Parallelization**: Execute independent steps concurrently
- **Caching**: Cache LLM responses for identical prompts
- **Batching**: Combine multiple small file operations

## Logging

All operations logged to structured JSON:

```typescript
interface LogEntry {
  timestamp: string
  level: 'debug' | 'info' | 'warn' | 'error'
  component: string
  message: string
  data?: Record<string, unknown>
}
```

Logs streamed to host, stored in database, viewable in UI.

## Future Enhancements

- Multi-agent collaboration (agents communicate)
- Learning from past tasks (vector DB of successful plans)
- User feedback loop (thumbs up/down on steps)
- Custom tool plugins (user-defined tools)
