# Homebase — Desktop Agent for File-Based Work

## Summary

Homebase is an Electron desktop app that gives an LLM agent controlled access to a user-selected folder. The agent can read, write, and execute code within a sandboxed VM environment. Users describe outcomes in natural language; the agent plans, executes, and reports progress with the ability to steer mid-task.

**LLM Backend:** NVIDIA LLM Gateway (model TBD — Nemotron or similar with strong tool-use support)

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Workspace** | A user-selected folder mounted into the agent's sandbox. Agent can only access files here. |
| **Task** | A user prompt describing desired outcome. Agent decomposes into a plan. |
| **Plan** | Ordered steps with tool calls. Visible and editable before/during execution. |
| **Run** | An execution session. Streams progress, captures logs, produces outputs. |
| **Sub-agent** | Parallel worker for independent subtasks (e.g., process 10 files concurrently). |
| **Knowledge Base** | Markdown projections of extracted knowledge (decisions, tasks, summaries). |

## Primary User Flow

```
1. Select workspace folder
2. Describe task ("Organize these receipts into a spreadsheet")
3. Review plan (agent shows steps)
4. Execute (stream progress, approve destructive actions)
5. Steer if needed (pause, edit plan, add context)
6. Receive outputs in workspace folder
```

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Electron Main                        │
│  ┌───────────┐  ┌───────────┐  ┌───────────────────┐   │
│  │ Window    │  │ Workspace │  │ Run Orchestrator  │   │
│  │ Manager   │  │ Manager   │  │ (approval gates)  │   │
│  └───────────┘  └───────────┘  └───────────────────┘   │
│                        │                                │
│                        ▼                                │
│              ┌─────────────────┐                        │
│              │   VM Manager    │                        │
│              │  (sandbox host) │                        │
│              └────────┬────────┘                        │
└───────────────────────┼─────────────────────────────────┘
                        │
          ┌─────────────▼─────────────┐
          │     Linux VM (isolated)   │
          │  ┌─────────────────────┐  │
          │  │   Agent Runtime     │  │
          │  │  - LLM calls        │  │
          │  │  - Tool execution   │  │
          │  │  - Sub-agent pool   │  │
          │  └─────────────────────┘  │
          │                           │
          │  /mnt/workspace (mounted) │
          └───────────────────────────┘
```

## Core Components

### 1. Workspace Sandbox
- User selects folder; mounted read/write into VM at `/mnt/workspace`
- Agent cannot access anything outside this mount
- File watching for live sync between host and VM

**Spec:** `specs/workspace-sandbox.md`

### 2. VM Isolation
- Lightweight Linux VM (consider Apple Virtualization Framework on macOS, or containerization cross-platform)
- Agent runtime executes inside VM
- Network access controlled (allowlist for LLM API, web fetch)
- Host communicates with VM via RPC

**Spec:** `specs/vm-isolation.md`

### 3. Agent Runtime
- Runs inside VM
- Agent loop: prompt → plan → tool calls → observe → iterate
- Calls NVIDIA LLM Gateway for completions
- Executes tools locally within sandbox

**Spec:** `specs/agent-runtime.md`

### 4. Tool System
Core tools available to agent:

| Tool | Description | Approval |
|------|-------------|----------|
| `list_directory` | List files/folders | Auto |
| `read_file` | Read file contents | Auto |
| `write_file` | Create/overwrite file | Auto (new) / Confirm (overwrite) |
| `delete_file` | Delete file | **Confirm** |
| `move_file` | Move/rename file | Auto |
| `run_code` | Execute Python/bash in sandbox | Auto |
| `web_fetch` | Fetch URL content | Auto |
| `web_search` | Search the web | Auto |

**Spec:** `specs/tool-system.md`

### 5. Planning & Execution
- Agent produces structured plan (JSON)
- UI renders as interactive checklist
- User can edit/skip/reorder steps before or during execution
- Streaming logs per step
- Approval gate pauses execution for destructive actions

**Spec:** `specs/planning-execution.md`

### 6. Parallel Sub-Agents
- Orchestrator can spawn sub-agents for independent work
- Example: "Process all 50 images" → 5 sub-agents, 10 images each
- Sub-agents share workspace mount, isolated execution
- Results aggregated by orchestrator

**Spec:** `specs/sub-agents.md`

### 7. Knowledge Base (Markdown Export)
- Agent can extract structured knowledge during runs
- Writes to `/.homebase/` in workspace:
  - `TASKS.md` — extracted action items
  - `DECISIONS.md` — captured decisions
  - `SUMMARY.md` — run summaries
  - `LOG.md` — execution history
- Projection only; source of truth is agent's structured output

**Spec:** `specs/knowledge-base.md`

### 8. Connectors (v2)
- Pluggable integrations: Email, Slack, Calendar, Notion, etc.
- Each connector exposes tools to the agent
- Write actions require approval

**Spec:** `specs/connectors.md` (future)

## Safety Model

1. **Sandbox isolation** — Agent runs in VM, only sees mounted workspace
2. **Network allowlist** — Only LLM API and approved domains
3. **Approval gates** — Destructive file ops require user confirmation
4. **Plan visibility** — User sees what agent intends before execution
5. **Kill switch** — User can abort run at any time
6. **Audit log** — All tool calls logged with timestamps

## Tech Stack

| Layer | Choice |
|-------|--------|
| Shell | Electron |
| UI | React + Vite + Tailwind |
| State | Zustand or TanStack Query |
| IPC | tRPC over Electron IPC |
| VM | TBD (macOS: Virtualization.framework, Linux: microVM/container) |
| Agent | Custom agent loop in TypeScript/Python |
| LLM | NVIDIA LLM Gateway |
| DB | SQLite (run history, settings) |

## Project Structure

```
homebase/
├── package.json
├── electron/
│   ├── main.ts
│   ├── preload.ts
│   ├── ipc/
│   ├── vm/              # VM lifecycle management
│   └── workspace/       # Folder mounting, file sync
├── agent/               # Agent runtime (runs in VM)
│   ├── loop.py          # Main agent loop
│   ├── tools/           # Tool implementations
│   ├── planner.py       # Plan generation
│   └── subagent.py      # Sub-agent spawning
├── src/                 # React renderer
│   ├── App.tsx
│   ├── components/
│   └── hooks/
├── specs/               # Component specs
│   ├── workspace-sandbox.md
│   ├── vm-isolation.md
│   ├── agent-runtime.md
│   ├── tool-system.md
│   ├── planning-execution.md
│   ├── sub-agents.md
│   └── knowledge-base.md
└── drizzle/
    └── schema.ts
```

## Non-Goals (v1)

- Browser automation (Claude in Chrome equivalent)
- Mobile app
- Multi-user / collaboration
- Bi-directional markdown sync (edits in MD don't flow back)
- Automatic execution without plan review

## Success Criteria

1. User can select a folder and run a file-organization task end-to-end
2. Agent executes in isolated VM with no access outside workspace
3. Destructive actions require explicit approval
4. Sub-agents can parallelize independent work
5. Knowledge artifacts are written to workspace as markdown

---

## Next Steps

1. Draft `specs/vm-isolation.md` — critical path, defines sandbox strategy
2. Draft `specs/agent-runtime.md` — agent loop + NVIDIA LLM integration
3. Draft `specs/tool-system.md` — core tool definitions
4. Prototype: Electron shell + VM boot + simple file listing task
