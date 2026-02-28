# Workstream Detection Guide

Analyze an implementation plan to identify independent workstreams for parallel execution.

## Algorithm

### Step 1: Build File Map

For each task in the plan, list all files it creates or modifies:

```
Task 1: src/auth/login.ts, src/auth/session.ts, tests/auth/login.test.ts
Task 2: src/api/routes.ts, src/api/middleware.ts, tests/api/routes.test.ts
Task 3: src/auth/register.ts, tests/auth/register.test.ts
Task 4: src/ui/dashboard.tsx, src/ui/styles.css, tests/ui/dashboard.test.tsx
```

### Step 2: Find Overlaps

Group tasks that share files (create OR modify the same file):
- Tasks 1 and 3 share `src/auth/` — same workstream
- Task 2 has no overlap — separate workstream
- Task 4 has no overlap — separate workstream

Result: 3 workstreams (auth, api, ui)

### Step 3: Handle Shared Infrastructure

Files that are **read-only shared** do NOT count as overlap:
- `tsconfig.json`, `package.json` — config files
- `src/types/index.ts` — if tasks only ADD new exports (append-only)
- `src/utils/helpers.ts` — if tasks only ADD new functions (append-only)

Files that are **write-shared** DO count as overlap:
- Same function modified by 2 tasks — same workstream
- Same component's state/props changed by 2 tasks — same workstream

### Step 4: Identify blockedBy Dependencies

Within a workstream, tasks may depend on each other:
- Task 3 uses types from Task 1 → `TaskCreate` with `addBlockedBy: ["1"]`

Between workstreams, tasks should NOT depend on each other (by definition).

If cross-workstream dependency exists, either:
1. Move the dependent task into the other workstream
2. Have team lead install the shared dependency first, before assigning tasks

### Step 5: Validate

| Workstreams | Action |
|-------------|--------|
| < 2 | Use subagent-driven-development (no parallelism benefit) |
| 2–4 | Team-driven-development with 1 teammate per workstream |
| > 4 | Combine smallest workstreams until ≤ 4 teammates |

## Edge Cases

- **Monorepo:** Each package is a natural workstream boundary
- **Migration files:** Usually append-only — OK to parallelize
- **Shared test fixtures:** Copy fixtures to each workstream's test directory if needed
- **Package.json dependencies:** Team lead adds ALL new dependencies in one commit BEFORE assigning tasks to teammates
- **Database schemas:** Schema changes go in one workstream; other workstreams use the resulting types
- **Tiny workstreams (1 task):** OK — still saves time if other workstreams are large
