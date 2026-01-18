# Tool System Specification

## Overview

Tools are the agent's interface to the outside world. Each tool is a function with a defined schema that the agent can call to perform actions.

## Core Tools

### File Operations

#### `list_directory`

List files and directories.

**Arguments**:
```typescript
{
  path: string // Relative to workspace root
  recursive?: boolean // Default: false
  includeHidden?: boolean // Default: false
}
```

**Returns**:
```typescript
{
  entries: Array<{
    name: string
    type: 'file' | 'directory' | 'symlink'
    size: number
    modified: string // ISO 8601 timestamp
  }>
}
```

**Approval**: Auto

---

#### `read_file`

Read file contents.

**Arguments**:
```typescript
{
  path: string // Relative to workspace root
  encoding?: 'utf-8' | 'base64' // Default: 'utf-8'
}
```

**Returns**:
```typescript
{
  content: string
  size: number
  modified: string
}
```

**Approval**: Auto

---

#### `write_file`

Create or overwrite file.

**Arguments**:
```typescript
{
  path: string
  content: string
  encoding?: 'utf-8' | 'base64'
  createDirs?: boolean // Create parent directories if missing
}
```

**Returns**:
```typescript
{
  path: string
  size: number
}
```

**Approval**: 
- Auto if new file
- Confirm if overwriting existing file

---

#### `delete_file`

Delete file or directory.

**Arguments**:
```typescript
{
  path: string
  recursive?: boolean // For directories
}
```

**Returns**:
```typescript
{
  deleted: string[]
}
```

**Approval**: **Confirm** (always)

---

#### `move_file`

Move or rename file.

**Arguments**:
```typescript
{
  from: string
  to: string
  overwrite?: boolean // Default: false
}
```

**Returns**:
```typescript
{
  from: string
  to: string
}
```

**Approval**: Auto (unless overwriting)

---

### Code Execution

#### `run_code`

Execute Python or bash code in sandboxed environment.

**Arguments**:
```typescript
{
  language: 'python' | 'bash'
  code: string
  args?: string[] // Command-line arguments
  env?: Record<string, string> // Environment variables
  timeout?: number // Seconds, default: 60
}
```

**Returns**:
```typescript
{
  stdout: string
  stderr: string
  exitCode: number
  duration: number // milliseconds
}
```

**Approval**: Auto

**Security**:
- No network access (unless explicitly enabled)
- Resource limits (CPU, memory, time)
- Cannot access files outside workspace

---

### Web Access

#### `web_fetch`

Fetch content from URL.

**Arguments**:
```typescript
{
  url: string
  method?: 'GET' | 'POST'
  headers?: Record<string, string>
  body?: string
  timeout?: number // Seconds
}
```

**Returns**:
```typescript
{
  status: number
  headers: Record<string, string>
  body: string
}
```

**Approval**: Auto (if URL on allowlist)

---

#### `web_search`

Search the web via API.

**Arguments**:
```typescript
{
  query: string
  numResults?: number // Default: 5
}
```

**Returns**:
```typescript
{
  results: Array<{
    title: string
    url: string
    snippet: string
  }>
}
```

**Approval**: Auto

---

## Tool Definition Schema

```typescript
interface Tool {
  name: string
  description: string
  inputSchema: JSONSchema // Zod schema compiled to JSON Schema
  outputSchema: JSONSchema
  requiresApproval: boolean | ((args: unknown) => boolean)
  execute: (args: unknown) => Promise<Result<unknown>>
}
```

**Example**:
```typescript
const readFileTool: Tool = {
  name: 'read_file',
  description: 'Read the contents of a file in the workspace',
  inputSchema: z.object({
    path: z.string().describe('Path relative to workspace root'),
    encoding: z.enum(['utf-8', 'base64']).optional(),
  }).describe('read_file input'),
  outputSchema: z.object({
    content: z.string(),
    size: z.number(),
    modified: z.string(),
  }),
  requiresApproval: false,
  async execute(args) {
    const { path, encoding = 'utf-8' } = inputSchema.parse(args)
    // Implementation
  },
}
```

## Tool Registry

```typescript
class ToolRegistry {
  private tools = new Map<string, Tool>()
  
  register(tool: Tool): void {
    this.tools.set(tool.name, tool)
  }
  
  get(name: string): Tool | undefined {
    return this.tools.get(name)
  }
  
  list(): Tool[] {
    return Array.from(this.tools.values())
  }
  
  // Generate OpenAI-compatible function definitions for LLM
  toOpenAIFunctions(): Array<{
    name: string
    description: string
    parameters: JSONSchema
  }> {
    return this.list().map(tool => ({
      name: tool.name,
      description: tool.description,
      parameters: tool.inputSchema,
    }))
  }
}
```

## Approval System

### Approval Gate

```typescript
interface ApprovalRequest {
  stepId: string
  tool: string
  args: Record<string, unknown>
  reason: string // Human-readable explanation
}

interface ApprovalResponse {
  approved: boolean
  modifiedArgs?: Record<string, unknown> // User can modify args
}
```

**Flow**:
1. Agent calls tool requiring approval
2. Execution pauses
3. UI shows approval dialog with details
4. User approves/denies (optionally modifies args)
5. Execution resumes or aborts

### Dynamic Approval

Some tools require conditional approval:

```typescript
const writeFileTool: Tool = {
  // ...
  requiresApproval: (args) => {
    const { path } = args as { path: string }
    const exists = fs.existsSync(path)
    return exists // Require approval only if overwriting
  },
}
```

## Error Handling

```typescript
type ToolResult<T> = 
  | { ok: true; value: T }
  | { ok: false; error: ToolError }

interface ToolError {
  code: string // e.g., 'FILE_NOT_FOUND', 'PERMISSION_DENIED'
  message: string
  details?: Record<string, unknown>
}
```

**Common Error Codes**:
- `INVALID_PATH` - Path outside workspace or malformed
- `FILE_NOT_FOUND` - File doesn't exist
- `PERMISSION_DENIED` - Cannot read/write file
- `TIMEOUT` - Operation exceeded time limit
- `EXECUTION_ERROR` - Code execution failed

## Future Tools

- `git_*` - Git operations (status, commit, push)
- `search_code` - Semantic code search
- `run_tests` - Execute test suites
- `format_code` - Auto-format code files
- `install_package` - Install npm/pip packages
