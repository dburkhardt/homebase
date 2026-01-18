# Agent Runtime

## Purpose

The agent runtime runs inside the VM and executes user tasks by:
- Receiving tasks from host via RPC
- Generating plans using LLM
- Executing tools (file ops, code execution, web access)
- Reporting progress to host
- Managing sub-agents for parallel work

## Structure

```
agent/
├── loop.ts          # Main agent loop
├── planner.ts       # LLM-based plan generation
├── executor.ts      # Plan execution engine
├── subagent.ts      # Sub-agent spawning and coordination
├── tools/           # Tool implementations
│   ├── file.ts      # File operations
│   ├── code.ts      # Code execution (Python, bash)
│   └── web.ts       # Web fetch, search
├── llm/             # LLM integration
│   ├── client.ts    # NVIDIA Gateway client
│   └── prompts.ts   # Prompt templates
└── utils/
    ├── logger.ts    # Structured logging
    └── rpc.ts       # Host communication
```

## Key Patterns

### Agent Loop

```typescript
// agent/loop.ts
async function agentLoop() {
  while (true) {
    const task = await receiveTask()
    
    // 1. Generate plan
    const plan = await planner.generate(task)
    await reportPlan(plan)
    
    // 2. Execute steps
    const result = await executor.execute(plan)
    
    // 3. Report completion
    await reportCompletion(result)
  }
}
```

### Tool System

```typescript
// agent/tools/file.ts
export const readFileTool: Tool = {
  name: 'read_file',
  description: 'Read file contents',
  inputSchema: z.object({
    path: z.string(),
  }),
  outputSchema: z.object({
    content: z.string(),
  }),
  requiresApproval: false,
  async execute(args) {
    const { path } = args
    const content = await fs.readFile(
      `/mnt/workspace/${path}`, 
      'utf-8'
    )
    return { ok: true, value: { content } }
  },
}
```

### LLM Integration

```typescript
// agent/llm/client.ts
async function* streamCompletion(
  prompt: string
): AsyncGenerator<string> {
  const response = await fetch(NVIDIA_API_ENDPOINT, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${API_KEY}`,
    },
    body: JSON.stringify({
      model: 'nvidia/nemotron-4-340b-instruct',
      messages: [{ role: 'user', content: prompt }],
      stream: true,
    }),
  })
  
  for await (const chunk of response.body) {
    yield parseChunk(chunk)
  }
}
```

## Implementation Guidelines

1. **Workspace Isolation**: Only access files under `/mnt/workspace`
2. **Error Handling**: All tool executions return Result types
3. **Streaming**: Stream LLM responses and progress updates
4. **Approval Gates**: Request approval for destructive operations
5. **Logging**: Log all operations for debugging and audit

## Environment

The agent runs in a minimal Linux VM with:
- Node.js runtime
- Python 3 (for code execution)
- Standard POSIX tools (bash, curl, etc.)
- Network access to LLM API only

## See Also

- [specs/agent-runtime.md](../specs/agent-runtime.md) - Detailed architecture
- [specs/tool-system.md](../specs/tool-system.md) - Tool definitions
- [specs/planning-execution.md](../specs/planning-execution.md) - Execution model
- [AGENTS.md](../AGENTS.md) - Development patterns
