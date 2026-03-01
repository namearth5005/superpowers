---
name: team-driven-development
description: Use when executing implementation plans with 2+ independent workstreams that can be worked on concurrently by separate teammates
---

# Team-Driven Development

Execute plan by creating a coordinated team with parallel teammates, each owning an independent workstream, with two-stage review after each task.

**Core principle:** One teammate per workstream + parallel execution + two-stage review = fast, high-quality delivery.

**Announce at start:** "I'm using the team-driven-development skill to execute this plan with parallel workstreams."

## When to Use

```dot
digraph when_to_use {
    "Have implementation plan?" [shape=diamond];
    "2+ independent workstreams?" [shape=diamond];
    "Want parallel execution?" [shape=diamond];
    "team-driven-development" [shape=box style=filled fillcolor=lightgreen];
    "subagent-driven-development" [shape=box style=filled fillcolor=lightyellow];
    "Manual execution or brainstorm first" [shape=box];

    "Have implementation plan?" -> "2+ independent workstreams?" [label="yes"];
    "Have implementation plan?" -> "Manual execution or brainstorm first" [label="no"];
    "2+ independent workstreams?" -> "Want parallel execution?" [label="yes"];
    "2+ independent workstreams?" -> "subagent-driven-development" [label="no"];
    "Want parallel execution?" -> "team-driven-development" [label="yes"];
    "Want parallel execution?" -> "subagent-driven-development" [label="no"];
}
```

**vs. Subagent-Driven Development (sequential):**
- Multiple teammates work concurrently (not one subagent at a time)
- Each teammate owns a full workstream (not one task)
- Shared task list for coordination (not controller re-dispatching)
- Same two-stage review quality (spec then quality)

**vs. Executing Plans (batched):**
- Same session (no context switch)
- Parallel workstreams (not sequential batches)
- Automated review (no human-in-loop between tasks)

## Workstream Detection

<HARD-GATE>
Do NOT create a team until you have analyzed the plan for independent workstreams.
Any plan with fewer than 2 independent workstreams MUST use subagent-driven-development instead.
</HARD-GATE>

Analyze the plan using `./workstream-detection.md`:

1. List all files each task creates or modifies
2. Group tasks sharing files into same workstream
3. Tasks with no file overlap = separate workstreams
4. Shared config/types (read-only) do NOT count as overlap
5. Result: < 2 workstreams → use subagent-driven-development

## The Process

```dot
digraph process {
    rankdir=TB;

    "Read plan, detect workstreams (./workstream-detection.md)" [shape=box];
    "2+ independent workstreams?" [shape=diamond];
    "Use superpowers:subagent-driven-development" [shape=box style=filled fillcolor=lightyellow];

    subgraph cluster_setup {
        label="Team Setup";
        "TeamCreate with feature name" [shape=box];
        "Spawn 1 teammate per workstream via Agent (./teammate-prompt.md)" [shape=box];
        "TaskCreate per task with full text in description" [shape=box];
        "Set blockedBy for intra-workstream dependencies" [shape=box];
        "TaskUpdate to assign owner per workstream" [shape=box];
    }

    subgraph cluster_monitor {
        label="Per Completed Task";
        "Teammate sends completion message" [shape=box];
        "Dispatch spec reviewer subagent" [shape=box];
        "Spec compliant?" [shape=diamond];
        "SendMessage to teammate: fix spec gaps" [shape=box];
        "Dispatch code quality reviewer subagent" [shape=box];
        "Quality approved?" [shape=diamond];
        "SendMessage to teammate: fix quality issues" [shape=box];
    }

    "All tasks complete?" [shape=diamond];
    "Dispatch final code reviewer for entire implementation" [shape=box];
    "Use superpowers:finishing-a-development-branch" [shape=box style=filled fillcolor=lightgreen];

    "Read plan, detect workstreams (./workstream-detection.md)" -> "2+ independent workstreams?";
    "2+ independent workstreams?" -> "Use superpowers:subagent-driven-development" [label="no"];
    "2+ independent workstreams?" -> "TeamCreate with feature name" [label="yes"];
    "TeamCreate with feature name" -> "Spawn 1 teammate per workstream via Agent (./teammate-prompt.md)";
    "Spawn 1 teammate per workstream via Agent (./teammate-prompt.md)" -> "TaskCreate per task with full text in description";
    "TaskCreate per task with full text in description" -> "Set blockedBy for intra-workstream dependencies";
    "Set blockedBy for intra-workstream dependencies" -> "TaskUpdate to assign owner per workstream";
    "TaskUpdate to assign owner per workstream" -> "Teammate sends completion message";
    "Teammate sends completion message" -> "Dispatch spec reviewer subagent";
    "Dispatch spec reviewer subagent" -> "Spec compliant?";
    "Spec compliant?" -> "SendMessage to teammate: fix spec gaps" [label="no"];
    "SendMessage to teammate: fix spec gaps" -> "Dispatch spec reviewer subagent" [label="re-review"];
    "Spec compliant?" -> "Dispatch code quality reviewer subagent" [label="yes"];
    "Dispatch code quality reviewer subagent" -> "Quality approved?";
    "Quality approved?" -> "SendMessage to teammate: fix quality issues" [label="no"];
    "SendMessage to teammate: fix quality issues" -> "Dispatch code quality reviewer subagent" [label="re-review"];
    "Quality approved?" -> "All tasks complete?" [label="yes"];
    "All tasks complete?" -> "Teammate sends completion message" [label="no — wait for next"];
    "All tasks complete?" -> "Dispatch final code reviewer for entire implementation" [label="yes"];
    "Dispatch final code reviewer for entire implementation" -> "Use superpowers:finishing-a-development-branch";
}
```

## Team Setup

1. **Pre-tasks first** — install shared dependencies, add shared types, any setup work
   - <HARD-GATE>**COMMIT all pre-task changes before spawning teammates.** Worktrees fork from HEAD — uncommitted changes will NOT be visible to teammates. This is the #1 cause of teammate confusion.</HARD-GATE>
2. **TeamCreate** with a descriptive name (e.g., `"feature-auth"`)
3. **Spawn teammates** — one per workstream, max 4, using `./teammate-prompt.md`
   - Each teammate is `general-purpose` subagent type
   - Each teammate gets: architectural context + their workstream scope
   - **Use `isolation: "worktree"` on the Agent tool** to give each teammate a separate git worktree. This prevents teammates from committing to the same branch.
4. **TaskCreate** for every task from the plan
   - Include full task text in `description` (don't make teammates read plan file)
   - Set `blockedBy` for intra-workstream dependencies
   - Set `activeForm` for progress visibility
5. **TaskUpdate** to assign `owner` based on workstream → teammate mapping

## Team Lead Role

**You coordinate. You do NOT implement.**

- Create team and tasks
- Assign tasks to teammates
- Dispatch review subagents when teammates report completion
- Message teammates with review feedback
- Handle cross-workstream issues (shared dependencies, merge conflicts)
- Shut down teammates when all work is done

## Review Coordination

Same two-stage review as subagent-driven-development. Reviews are lightweight subagents, NOT teammates.

1. Teammate completes task → sends completion message via SendMessage → **teammate STOPS and waits**
2. **Spec compliance review** — dispatch subagent using `subagent-driven-development/spec-reviewer-prompt.md`
   - **IMPORTANT:** Pass the teammate's worktree path to the reviewer, NOT the main repo path. Teammates work in isolated worktrees — the main repo won't have their changes.
3. If spec fails → SendMessage to teammate with specific issues → teammate fixes → re-review
4. **Code quality review** — dispatch subagent using `subagent-driven-development/code-quality-reviewer-prompt.md`
   - Same worktree path rule applies.
5. If quality fails → SendMessage to teammate with issues → teammate fixes → re-review
6. Both pass → SendMessage to teammate: "Task approved, proceed to next" → task confirmed complete

Reviews for different workstreams can happen in parallel.

**Start code quality review ONLY after spec compliance passes.** Order matters.

## Completion

1. All tasks marked complete and reviewed
2. Dispatch final cross-workstream code reviewer (entire diff)
3. **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch
4. SendMessage `type: "shutdown_request"` to all teammates
5. Wait for shutdown confirmations

## Red Flags

**Never:**
- Create a team for < 2 independent workstreams
- Have team lead implement code (coordination only)
- Skip workstream detection analysis
- Spawn > 4 teammates (coordination overhead exceeds parallelism gains)
- Skip reviews for team-completed work
- Let teammates work on overlapping files without coordination
- Start code quality review before spec compliance passes
- Use reviews as teammates (use lightweight subagents)
- Proceed past review failures

**Always:**
- Commit all pre-tasks before spawning teammates
- Analyze plan for parallelism before creating team
- One teammate per workstream
- Use `isolation: "worktree"` when spawning teammates (separate worktree per teammate)
- Pass worktree paths (not main repo path) to review subagents
- Two-stage review (spec then quality) for every completed task
- Shut down teammates when done

## Rationalization Prevention

| Excuse | Reality |
|--------|---------|
| "Only 2 tasks, team is overkill" | Count workstreams not tasks. 2 independent workstreams = use team. |
| "I can coordinate without a formal team" | Without TeamCreate/TaskCreate, no real parallelism. Use the tools. |
| "Teammates can share files" | File overlap = same workstream. Separate workstreams = no overlap. |
| "I'll skip reviews, teammates self-review" | Self-review supplements, never replaces. Two-stage review always. |
| "3+ workstreams, I need 5 teammates" | Max 4. Combine smallest workstreams until ≤ 4. |
| "Setting up a team takes too long" | TeamCreate + TaskCreate takes 30 seconds. Parallelism saves minutes. |

## Integration

**Required workflow skills:**
- **superpowers:writing-plans** — Creates the plan this skill executes
- **superpowers:finishing-a-development-branch** — Complete development after all tasks

**Reviews use (from subagent-driven-development):**
- `subagent-driven-development/spec-reviewer-prompt.md`
- `subagent-driven-development/code-quality-reviewer-prompt.md`

**Teammates should use:**
- **superpowers:test-driven-development** — TDD for each task
- **superpowers:verification-before-completion** — Verify before claiming done

**Alternative workflows:**
- **superpowers:subagent-driven-development** — Use for sequential execution (< 2 workstreams)
- **superpowers:executing-plans** — Use for parallel session with human checkpoints
