---
name: next
description: Analyze and pick the next task to work on. Reads MASTER_PLAN.md, scores tasks by priority and status, and presents interactive selection. Use when starting a session or deciding what to tackle next.
---

# What's Next?

Analyze MASTER_PLAN.md tasks, score by priority/status, and let the user pick interactively.

## Triggers

- `/master-plan:next` - Main command
- "what should I work on", "pick a task", "next task", "what's next"

## Arguments

| Argument | Filter |
|----------|--------|
| `bugs` | Only BUG-XXX tasks |
| `progress` | Only IN PROGRESS tasks |
| `planned` | Only PLANNED (backlog) tasks |
| `review` | Only REVIEW tasks |
| `active` | IN PROGRESS + REVIEW |
| `all` | Include DONE tasks |

Example: `/master-plan:next bugs` or `/master-plan:next planned`

## Workflow

### Step 1: Find MASTER_PLAN.md

Search for the plan file in order:
1. `docs/MASTER_PLAN.md`
2. `MASTER_PLAN.md`
3. `master-plan.md`
4. `docs/master-plan.md`

If not found, tell the user: "No MASTER_PLAN.md found. Run `/master-plan:task` to create your first task, or create one from the template."

### Step 2: Parse Tasks

Read the file and extract tasks from `###` headers matching this pattern:

```
### [optional ~~]TASK-123[optional ~~]: Title (STATUS)
```

Regex: `^###\s+(~~)?((TASK|BUG|FEATURE|ROAD|IDEA|ISSUE|INQUIRY)-\d+)(~~)?:\s*(.+?)\s*\(([^)]+)\)`

For each match, extract:
- **ID**: e.g., `TASK-123`, `BUG-456`
- **Title**: The text after `: ` and before ` (`
- **Status**: Text in parentheses. Normalize:
  - Contains "DONE", "COMPLETE", "FIXED", "✅", or has strikethrough → `DONE`
  - Contains "IN PROGRESS", "🔄" → `IN PROGRESS`
  - Contains "REVIEW", "👀" → `REVIEW`
  - Contains "PAUSED", "⏸️" → `PAUSED`
  - Otherwise → `PLANNED`
- **Priority**: Look in the lines following the header for `**Priority**: P0-P3`. Default to P2 if not found.

### Step 3: Check for Active Work

Look for IN PROGRESS tasks first — these should be finished before starting new work.

Also check git status:
```bash
git status --short
```

If uncommitted changes exist, mention: "You have uncommitted changes — consider committing or running `/master-plan:save` first."

### Step 4: Sort and Filter

**Default sort order** (what to START next):
1. PLANNED first, then REVIEW, IN PROGRESS, PAUSED, DONE last
2. Within same status: P0 → P1 → P2 → P3

**If `$ARGUMENTS` contains a filter** (bugs, progress, planned, review, active), apply it before sorting.

**Exclude DONE tasks** unless `$ARGUMENTS` contains `all`.

### Step 5: Present Interactive Selection

Use `AskUserQuestion` to let user pick from the top tasks:

```
AskUserQuestion({
  questions: [{
    question: "Which task would you like to work on?",
    header: "Task",
    multiSelect: false,
    options: [
      // Top 4 tasks, formatted as:
      { label: "TASK-XXX: Title here", description: "P0 · IN PROGRESS" },
      { label: "BUG-YYY: Another task", description: "P1 · PLANNED" },
      // ...
    ]
  }]
})
```

**Option formatting:**
- Label: `{ID}: {title (max 40 chars)}`
- Description: `{priority} · {status}`

If there are IN PROGRESS tasks, always show them first with a note: "You have N task(s) in progress."

### Step 6: Show Task Details

When user selects a task, show the full section from MASTER_PLAN.md:
- ID, title, status, priority
- Description (everything between this `###` header and the next `###` or `##`)

### Step 7: Offer Actions

After showing task details, ask:

```
AskUserQuestion({
  questions: [{
    question: "What would you like to do?",
    header: "Action",
    multiSelect: false,
    options: [
      { label: "Start working on this", description: "Update status to IN PROGRESS" },
      { label: "Pick a different task", description: "Go back to task list" },
      { label: "Just show context", description: "No status change" }
    ]
  }]
})
```

If "Start working on this" is selected:
1. Update the task's status to `IN PROGRESS` in MASTER_PLAN.md (all locations — see update rules below)
2. Update the `**Status**:` line inside the task's `###` section to reflect the current date:
   ```markdown
   **Status**: IN PROGRESS (YYYY-MM-DD)
   ```
3. Confirm: "Started TASK-XXX. Ready to begin implementation."

## MASTER_PLAN.md Update Rules

Tasks may appear in multiple locations. When updating status, check ALL of:

1. **Summary/Roadmap table** — Update the status column
2. **Detailed `###` section header** — Update `(STATUS)` in parentheses
3. **Subtask bullet points** — Add strikethrough + ✅ if marking DONE

Always verify with a grep after updating.

## Rules

1. **Finish before starting** — Always highlight IN PROGRESS tasks first
2. **P0 trumps all** — Critical issues come first regardless of status
3. **Interactive selection** — Always use AskUserQuestion, never just print a list
4. **Context on selection** — Always show full task details when picked
5. **Action oriented** — Offer to start work immediately
