# Component Specifications

## Purpose

Detailed design specifications for each major component of Homebase. These docs guide implementation and serve as authoritative references.

## Specifications

| File | Component | Description |
|------|-----------|-------------|
| [workspace-sandbox.md](workspace-sandbox.md) | Workspace Sandbox | Isolated file system access, path validation |
| [vm-isolation.md](vm-isolation.md) | VM Isolation | Linux VM architecture, security model |
| [agent-runtime.md](agent-runtime.md) | Agent Runtime | Agent loop, LLM integration, execution flow |
| [tool-system.md](tool-system.md) | Tool System | Tool definitions, approval gates, error handling |
| [planning-execution.md](planning-execution.md) | Planning & Execution | Plan generation, step execution, progress streaming |
| [sub-agents.md](sub-agents.md) | Sub-Agents | Parallel execution, work distribution, aggregation |
| [knowledge-base.md](knowledge-base.md) | Knowledge Base | Markdown projections, task extraction, summaries |

## Reading Order

For new agents or comprehensive understanding:

1. Start with [SPEC.md](../SPEC.md) - overall product vision
2. Read [AGENTS.md](../AGENTS.md) - development patterns
3. Read specs in order:
   - workspace-sandbox.md - foundational security
   - vm-isolation.md - execution environment
   - agent-runtime.md - core agent logic
   - tool-system.md - available operations
   - planning-execution.md - task handling
   - sub-agents.md - parallelization
   - knowledge-base.md - learning & memory

## Usage

Component agents should:
1. Read relevant spec before implementing
2. Follow architecture and patterns described
3. Update spec if implementation diverges (with user approval)
4. Cross-reference related specs

## See Also

- [SPEC.md](../SPEC.md) - Product specification
- [AGENTS.md](../AGENTS.md) - Development guide
