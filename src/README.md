# React Renderer

## Purpose

The React renderer provides the UI for:
- Workspace selection and management
- Task input (natural language)
- Plan review and editing
- Execution monitoring (streaming logs, progress)
- Knowledge base viewer
- Settings and preferences

## Structure

```
src/
├── App.tsx          # Main app component, routing
├── components/      # Reusable UI components
│   ├── workspace/
│   │   ├── WorkspaceSelector.tsx
│   │   └── WorkspaceInfo.tsx
│   ├── task/
│   │   ├── TaskInput.tsx
│   │   └── TaskHistory.tsx
│   ├── plan/
│   │   ├── PlanReview.tsx
│   │   ├── StepEditor.tsx
│   │   └── StepList.tsx
│   ├── execution/
│   │   ├── ExecutionView.tsx
│   │   ├── ProgressBar.tsx
│   │   └── LogViewer.tsx
│   └── knowledge/
│       ├── KnowledgeBase.tsx
│       └── MarkdownViewer.tsx
├── hooks/           # Custom React hooks
│   ├── useTRPC.ts   # tRPC React hooks
│   ├── useWorkspace.ts
│   └── useTask.ts
├── stores/          # Zustand state stores
│   ├── workspace.ts
│   ├── task.ts
│   └── ui.ts
├── styles/          # Tailwind CSS
│   └── index.css
└── utils/
    ├── formatting.ts
    └── validation.ts
```

## Key Patterns

### tRPC Client

```typescript
// src/hooks/useTRPC.ts
import { createTRPCReact } from '@trpc/react-query'
import type { AppRouter } from '@electron/ipc'

export const trpc = createTRPCReact<AppRouter>()

// Usage in components
function WorkspaceSelector() {
  const selectWorkspace = trpc.workspace.select.useMutation()
  
  const handleSelect = async () => {
    const result = await selectWorkspace.mutateAsync()
    console.log('Selected:', result.path)
  }
  
  return <button onClick={handleSelect}>Select Folder</button>
}
```

### State Management

```typescript
// src/stores/workspace.ts
import { create } from 'zustand'

interface WorkspaceState {
  current: Workspace | null
  recent: Workspace[]
  setCurrent: (workspace: Workspace) => void
}

export const useWorkspaceStore = create<WorkspaceState>((set) => ({
  current: null,
  recent: [],
  setCurrent: (workspace) => set({ current: workspace }),
}))
```

### Streaming Progress

```typescript
// src/components/execution/ExecutionView.tsx
function ExecutionView({ taskId }: { taskId: string }) {
  const [progress, setProgress] = useState<Progress[]>([])
  
  trpc.task.subscribeProgress.useSubscription(
    { taskId },
    {
      onData: (event) => {
        setProgress(prev => [...prev, event])
      }
    }
  )
  
  return (
    <div>
      {progress.map(p => (
        <ProgressItem key={p.stepId} progress={p} />
      ))}
    </div>
  )
}
```

## UI Design Principles

1. **Clean & Minimal**: Inspired by modern dev tools (Linear, Vercel)
2. **Responsive**: Adapt to window size changes
3. **Accessible**: Keyboard navigation, ARIA labels
4. **Fast**: Optimistic updates, skeleton loading states
5. **Informative**: Clear error messages, helpful tooltips

## Styling

Uses Tailwind CSS with custom design tokens:

```css
/* src/styles/index.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --color-primary: 59 130 246; /* blue-500 */
    --color-success: 34 197 94;  /* green-500 */
    --color-danger: 239 68 68;   /* red-500 */
  }
}
```

## Implementation Guidelines

1. **Type Safety**: Use tRPC types, avoid `any`
2. **Error Boundaries**: Wrap components in error boundaries
3. **Loading States**: Show skeletons during async operations
4. **Optimistic UI**: Update UI before server confirms
5. **Testing**: Test user flows with Vitest + Testing Library

## See Also

- [AGENTS.md](../AGENTS.md) - Development patterns
- [Tailwind CSS Docs](https://tailwindcss.com/)
- [tRPC React Docs](https://trpc.io/docs/client/react)
