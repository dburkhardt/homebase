# Knowledge Base Specification

## Overview

The knowledge base is a markdown projection of structured knowledge extracted during agent runs. Files are written to `/.homebase/` in the workspace as a human-readable audit trail and context for future tasks.

## Directory Structure

```
workspace/
├── .homebase/
│   ├── TASKS.md         # Extracted action items
│   ├── DECISIONS.md     # Captured decisions
│   ├── SUMMARY.md       # Run summaries
│   ├── LOG.md           # Execution history
│   └── runs/
│       ├── 2026-01-18_task-123/
│       │   ├── plan.json
│       │   ├── logs.jsonl
│       │   └── outputs.json
│       └── ...
└── [user files]
```

## File Formats

### TASKS.md

Extracted action items from conversations and runs.

```markdown
# Tasks

## Pending

- [ ] Update API documentation after schema change (from run: 2026-01-18_task-123)
- [ ] Refactor authentication module (from user request: 2026-01-15)
- [ ] Add error handling to data pipeline (extracted: 2026-01-17)

## In Progress

- [x] Implement user profile page (started: 2026-01-16)

## Completed

- [x] Fix login bug (completed: 2026-01-14, run: task-098)
- [x] Deploy to staging (completed: 2026-01-13)
```

**Extraction**:
- LLM identifies action items during execution
- Agent writes to TASKS.md after each run
- Append-only (never delete completed tasks)

---

### DECISIONS.md

Architectural decisions and rationales.

```markdown
# Decisions

## 2026-01-18: Use Zustand over Redux

**Context**: Need state management for React app

**Decision**: Use Zustand for simplicity

**Rationale**:
- Less boilerplate than Redux
- TypeScript support is excellent
- Sufficient for our use case

**Alternatives Considered**:
- Redux: Too complex for our needs
- Context API: Doesn't scale well

**Run**: task-123

---

## 2026-01-15: SQLite over PostgreSQL

**Context**: Need database for agent runs

**Decision**: Use SQLite for local storage

**Rationale**:
- No server setup required
- Sufficient for single-user desktop app
- Easy backup (single file)

**Run**: task-098
```

**Extraction**:
- LLM identifies decision points during planning
- Agent prompts: "Record this decision?"
- User can manually add via UI

---

### SUMMARY.md

High-level summaries of each run.

```markdown
# Run Summaries

## 2026-01-18: Implement User Profile Page

**Status**: ✓ Completed

**Duration**: 14 minutes

**Description**: Created new user profile page with avatar upload, bio editing, and account settings.

**Steps**:
1. Created ProfilePage component
2. Implemented avatar upload with image resizing
3. Added form validation
4. Styled with Tailwind
5. Added unit tests

**Outputs**:
- `src/components/ProfilePage.tsx`
- `src/components/AvatarUpload.tsx`
- `src/__tests__/ProfilePage.test.tsx`

**Decisions Made**:
- Use Zustand for profile state
- Max avatar size: 5MB

**Follow-up Tasks**:
- Add profile photo editing (crop/rotate)
- Implement social links section

---

## 2026-01-15: Fix Login Bug

**Status**: ✓ Completed

**Duration**: 8 minutes

**Description**: Fixed authentication token expiry issue causing users to be logged out prematurely.

**Steps**:
1. Identified bug in token validation
2. Updated expiry check logic
3. Added tests for edge cases

**Outputs**:
- `src/auth/validateToken.ts`
- `src/__tests__/auth.test.ts`
```

**Generation**:
- LLM generates summary after run completes
- Includes: status, duration, key actions, outputs, decisions

---

### LOG.md

Detailed execution log (chronological).

```markdown
# Execution Log

## 2026-01-18 14:32:15 - Task Started: task-123

User Request: "Create a user profile page"

---

## 2026-01-18 14:32:18 - Plan Generated

5 steps planned:
1. Create ProfilePage component
2. Implement avatar upload
3. Add form validation
4. Style with Tailwind
5. Add unit tests

---

## 2026-01-18 14:32:20 - Step 1: Running

Tool: write_file
Args: { path: "src/components/ProfilePage.tsx", ... }

---

## 2026-01-18 14:32:22 - Step 1: Completed

Created: src/components/ProfilePage.tsx (248 lines)

---

## 2026-01-18 14:46:03 - Task Completed

Duration: 13m 48s
Status: Success
Outputs: 3 files created
```

**Format**: Append-only, timestamped entries

---

## Extraction Process

### During Execution

```typescript
interface ExtractionContext {
  tasks: string[]        // Action items discovered
  decisions: Decision[]  // Decisions made
  insights: string[]     // Notable observations
}

async function extractKnowledge(
  stepResult: StepResult,
  context: ExtractionContext
): Promise<void> {
  const prompt = `
Analyze this step result:
${JSON.stringify(stepResult)}

Extract:
1. Any action items for future work
2. Important decisions made
3. Notable insights

Format as JSON:
{
  "tasks": ["..."],
  "decisions": [{ "title": "...", "rationale": "..." }],
  "insights": ["..."]
}
`

  const extraction = await llm.complete(prompt)
  context.tasks.push(...extraction.tasks)
  context.decisions.push(...extraction.decisions)
  context.insights.push(...extraction.insights)
}
```

### After Run Completion

```typescript
async function writeKnowledgeBase(
  taskId: string,
  context: ExtractionContext
): Promise<void> {
  const kbPath = path.join(workspace, '.homebase')
  
  // Append tasks
  if (context.tasks.length > 0) {
    await appendToFile(
      path.join(kbPath, 'TASKS.md'),
      formatTasks(context.tasks, taskId)
    )
  }
  
  // Append decisions
  if (context.decisions.length > 0) {
    await appendToFile(
      path.join(kbPath, 'DECISIONS.md'),
      formatDecisions(context.decisions, taskId)
    )
  }
  
  // Append summary
  const summary = await generateSummary(taskId, context)
  await appendToFile(
    path.join(kbPath, 'SUMMARY.md'),
    summary
  )
  
  // Append log
  await appendToFile(
    path.join(kbPath, 'LOG.md'),
    formatLog(taskId, context)
  )
}
```

## Read-Only Projection

**Important**: `.homebase/` is a projection, not source of truth.

- Agent can read from `.homebase/` for context
- Agent cannot edit `.homebase/` directly
- All updates go through extraction process
- Database is source of truth for run history

## UI Integration

### Knowledge Base Viewer

```
┌─────────────────────────────────────────────────┐
│ Knowledge Base                 [Refresh] [⚙️]   │
├─────────────────────────────────────────────────┤
│                                                 │
│ ┌─ TASKS.md ───────────────────────────────┐   │
│ │ ## Pending (3)                           │   │
│ │ - [ ] Update API docs                    │   │
│ │ - [ ] Refactor auth module               │   │
│ │ - [ ] Add error handling                 │   │
│ └──────────────────────────────────────────┘   │
│                                                 │
│ ┌─ DECISIONS.md ───────────────────────────┐   │
│ │ ## 2026-01-18: Use Zustand               │   │
│ │ **Rationale**: Less boilerplate...       │   │
│ └──────────────────────────────────────────┘   │
│                                                 │
│ ┌─ SUMMARY.md ─────────────────────────────┐   │
│ │ ## 2026-01-18: User Profile Page         │   │
│ │ Status: ✓ Completed                      │   │
│ │ Duration: 14 minutes                     │   │
│ └──────────────────────────────────────────┘   │
│                                                 │
└─────────────────────────────────────────────────┘
```

## Future Context

Agent reads `.homebase/` on new tasks:

```typescript
async function loadWorkspaceContext(): Promise<WorkspaceContext> {
  const kbPath = path.join(workspace, '.homebase')
  
  return {
    recentTasks: await readFile(path.join(kbPath, 'TASKS.md')),
    decisions: await readFile(path.join(kbPath, 'DECISIONS.md')),
    summaries: await readFile(path.join(kbPath, 'SUMMARY.md')),
  }
}

// Include in planning prompt
const prompt = `
Previous work in this workspace:
${workspaceContext.summaries}

Known decisions:
${workspaceContext.decisions}

New task: ${task.description}

Generate plan...
`
```

## Future Enhancements

- **Search**: Full-text search across knowledge base
- **Linking**: Cross-reference tasks, decisions, runs
- **Export**: Convert to PDF, HTML for sharing
- **Version Control**: Git integration (auto-commit to `.homebase/`)
