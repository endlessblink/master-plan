---
name: done
description: Task completion workflow — run tests, commit, push, and update MASTER_PLAN.md. Use when a task is finished and ready to ship. Triggers on "/done", "mark done", "task complete", "finish task", "ship it".
---

# Task Completion Workflow

Finalize a completed task: verify tests pass, commit changes, update MASTER_PLAN.md, and push.

## Triggers

- `/master-plan:done` - Main command
- "mark done", "task complete", "finish task", "ship it"

## Workflow

### Step 1: Check What Changed

Run git status to see what files were modified:

```bash
git status
git diff --stat
```

Output immediately:
```
Files changed:
- [list of modified files]
```

### Step 2: Get Task Information

Use `AskUserQuestion` to collect:

1. **Task ID** (header: "Task ID")
   - Options: "Tracked task (enter ID)" or "Quick fix (no task ID)"

2. **Run tests?** (header: "Tests")
   - Options: "Yes — run test suite (Recommended)" or "Skip tests"

Then ask in plain text: "What's a brief summary of the changes? (1-2 sentences)"

**IMPORTANT**: Wait for user to provide the summary before proceeding.

### Step 3: Run Tests (unless skipped)

Detect and run the project's test command. Check in order:
1. If `package.json` exists → `npm test`
2. If `Cargo.toml` exists → `cargo test`
3. If `pyproject.toml` or `setup.py` exists → `pytest`
4. If `Makefile` with `test` target → `make test`
5. If `go.mod` exists → `go test ./...`

**If tests fail**: STOP immediately. Report failures. Do NOT proceed.
**If tests pass**: Continue.
**If no test command found**: Warn user and continue.

### Step 4: Update MASTER_PLAN.md (if tracked task)

**Skip this step if user selected "Quick fix (no task ID)".**

**CRITICAL**: Tasks may appear in **multiple locations**. Update ALL of them:

#### 4a. Summary/Roadmap Table

Find the task row and update:

```markdown
# Before:
| **TASK-XXX** | **Title** | **P2** | PLANNED | ... |

# After:
| ~~**TASK-XXX**~~ | ✅ **Title** | **P2** | ✅ DONE (YYYY-MM-DD) | ... |
```

#### 4b. Subtask/Bullet Lists

```markdown
# Before:
- TASK-XXX: Description

# After:
- ~~TASK-XXX~~: ✅ Description
```

#### 4c. Detailed Section Header

```markdown
# Before:
### TASK-XXX: Title (IN PROGRESS)

# After:
### ~~TASK-XXX~~: Title (✅ DONE)
```

#### 4d. Verify Subtask Bullets One by One

Collect all `- [ ]` bullets within the task's `###` section. For each one, determine how to verify it:

**If a directly related test exists and tests passed (Step 3):**
A bullet is considered test-covered if the test name, file, or output explicitly references the same feature or behavior described in the bullet. Mark it `[x]` automatically.

**Otherwise — interview the user for each unchecked bullet:**

```
AskUserQuestion({
  questions: [{
    question: "Was this subtask completed? \"[subtask text]\"",
    header: "Subtask",
    multiSelect: false,
    options: [
      { label: "Yes — mark complete", description: "Change [ ] to [x]" },
      { label: "No — leave incomplete", description: "Keep as [ ] for follow-up" }
    ]
  }]
})
```

- **Yes** → change `- [ ]` to `- [x]`
- **No** → leave as `- [ ]` and record in an **incomplete list**

**After reviewing all bullets — decide the outcome:**

- **All bullets are `[x]`** → proceed to mark the task ✅ DONE (steps 4e–4f)
- **Any bullet remains `[ ]`** → the task is NOT done. Skip steps 4a, 4c, 4e, 4f entirely. Instead:
  1. Keep the `###` section header as-is (do not add strikethrough or ✅)
  2. Update `**Status**:` to `IN PROGRESS (YYYY-MM-DD)` with today's date
  3. Leave the section in `## Active Work` — do NOT move it to `## Completed`
  4. Commit and push the progress (using `wip(TASK-XXX):` prefix), then report incomplete items (Step 6)

#### 4e. Update the **Status**: Line (fully complete tasks only)

Find the `**Status**:` line inside the task's `###` section and replace it:

```markdown
# Before:
**Status**: IN PROGRESS (2026-02-23)

# After:
**Status**: ✅ DONE (YYYY-MM-DD)
```

Use today's date.

#### 4f. Move Section to ## Completed (fully complete tasks only)

Cut the entire `###` block (from the `### ~~TASK-XXX~~:` header line down to, but not including, the next `###` or `##` heading) and append it under the `## Completed` section. If no `## Completed` section exists, create one at the end of the file.

**Never move a section with unchecked `- [ ]` bullets to `## Completed`.**

#### 4g. Verify All Updated

```bash
grep "TASK-XXX" docs/MASTER_PLAN.md
```

Confirm: if fully complete, all occurrences show strikethrough and ✅ DONE. If incomplete, confirm the section remains in Active Work with IN PROGRESS status.

### Step 5: Commit and Push

Stage all relevant files. **NEVER commit**:
- `.env*` files
- `credentials*.json` or files containing secrets
- `node_modules/`, `__pycache__/`, `target/`, `.venv/`
- Backup files, generated stats, lock files (unless intentional)

Prefer staging specific files by name over `git add -A`.

```bash
# Stage code files
git add <changed-files>
git add docs/MASTER_PLAN.md  # if tracked task

# Commit
git commit -m "$(cat <<'EOF'
feat(TASK-XXX): Brief summary from user

- Key change 1
- Key change 2

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"

# Push
git push
```

**For quick fixes (no task ID):**
```bash
git commit -m "$(cat <<'EOF'
fix: Brief summary from user

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
git push
```

**Commit prefix conventions:**
- `feat(TASK-XXX):` — New feature
- `fix(BUG-XXX):` — Bug fix
- `chore(TASK-XXX):` — Maintenance
- `refactor(TASK-XXX):` — Code refactoring
- `docs(TASK-XXX):` — Documentation only

### Step 6: Report Completion

**If all subtasks were completed:**

```
## ✅ Task Complete

- **Task**: TASK-XXX (or "Quick fix")
- **Summary**: [user's summary]
- **Tests**: ✅ Passed (or ⏭️ Skipped)
- **Subtasks**: ✅ All complete
- **Commit**: [short hash] — [message]
- **Push**: ✅ Pushed to origin
- **MASTER_PLAN.md**: Moved to Completed
```

**If any subtasks remain incomplete:**

```
## 🔄 Progress Saved — Task Still In Progress

- **Task**: TASK-XXX
- **Summary**: [user's summary]
- **Tests**: ✅ Passed / ⏭️ Skipped
- **Subtasks**: ⚠️ [N] incomplete (see below)
- **Commit**: [short hash] — wip(TASK-XXX): [message]
- **Push**: ✅ Pushed to origin
- **MASTER_PLAN.md**: Remains IN PROGRESS in Active Work

### Incomplete Subtasks
- [ ] [subtask text]
- [ ] [subtask text]

Run `/master-plan:next` to pick this task back up and finish these items.
```

## Important Rules

1. **NEVER skip commit/push** — Changes must be pushed to the remote
2. **ALWAYS collect summary** — Don't proceed without knowing what changed
3. **Update ALL locations** in MASTER_PLAN.md for tracked tasks
4. **Verify with grep** after updating MASTER_PLAN.md
5. **Wait for user input** — Don't assume or skip questions
6. **Test failures block completion** — If tests fail, stop and report
7. **All subtasks must be `[x]` before marking DONE** — A task with any `[ ]` bullet stays IN PROGRESS
8. **`## Completed` is for fully finished tasks only** — Never move a section there while any subtask is unchecked
9. **Tests only auto-check bullets with an explicitly related test** — All other unchecked bullets require user confirmation

## Files to NEVER Commit

- `.env*` files (secrets)
- `credentials*.json`
- Private keys, tokens
- Generated/cached files (`node_modules/`, `dist/`, `__pycache__/`)
- OS files (`.DS_Store`, `Thumbs.db`)
