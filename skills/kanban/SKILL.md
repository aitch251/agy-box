---
name: kanban
description: >-
  Manages task tracking, refinement, prioritization, and planning using local kanban.md boards across any repository or subdirectory.
  Use this skill whenever the user asks to check, plan, prioritize, or refine tasks, work the kanban, review the backlog, or create a new kanban.md board. Trigger with /kanban.
---

# Universal Kanban Coordinator

A flexible, lightweight task management coordinator that operates directly on `kanban.md` files across any repository or project subdirectory.

---

## 1. Discovery & Board Selection

When `/kanban` is invoked:

1. **Locate `kanban.md`**:
   - Check current working directory (`./kanban.md`).
   - Walk up to the repository root.
   - If not found in parent paths, scan one level of immediate subdirectories:
     ```bash
     find . -maxdepth 2 -name "kanban.md" -not -path "*/.*/*"
     ```
2. **Handle Multiple Boards**:
   - If multiple `kanban.md` files exist (e.g., repo root vs. sub-project), present a concise list and ask which board to operate on.
3. **Handle Missing Board**:
   - If no `kanban.md` is found, ask the user if they would like to scaffold a new `kanban.md` in the current directory or repository root.
4. **Auto-Allowlist Runtime Permissions**:
   - Ensure the resolved `kanban.md` path is registered in runtime `~/.gemini/antigravity-cli/settings.json` under `permissions.allow` for both `read_file` and `write_file`.
   - **CRITICAL**: Modify ONLY runtime `~/.gemini/antigravity-cli/settings.json`.

---

## 2. Standard Board Structure

Every `kanban.md` follows a consistent, readable format:

```markdown
# <Project Name> Kanban (`kanban.md`)

> **Workflow**: Brief scope or context description.

---

## In Progress

- [ ] **Active Task Title**
  - [x] Completed sub-step
  - [ ] Next concrete sub-step

---

## Next Up

- [ ] **High Priority Upcoming Task**
  - [ ] Concrete step 1
  - [ ] Concrete step 2

---

## Backlog

- [ ] **Candidate Task or Idea**
- [ ] **Blocked Task** `(blocked: waiting for API key)`

---

## Hold

- [ ] **Paused Task** `(hold: deferred until Q3)`

---

## Completed

- [x] **2026-10-23 — Completed Feature Title**
  - [x] Implementation details
```

### Column Constraints
- **`In Progress`**: Limit of **1 active task per branch / worktree**. If a task is already active on the current branch/worktree, ask the user to create a new branch and worktree from `main` before starting another task. When multiple tasks run concurrently across branches, tag each card with `(branch: <branch-name>)`. Must have concrete `- [ ]` checkboxes for incremental progress.
- **`Next Up`**: 2–4 prioritized tasks ready for execution, broken into testable steps.
- **`Backlog`**: Unordered or lower-priority items. Blocked tasks are marked `(blocked: <short reason>)`.
- **`Hold`**: Explicitly paused tasks awaiting external decisions. Out of scope for automatic prioritization.
- **`Completed`**: Reverse-chronological list of finished tasks with date stamps.

---

## 3. Interaction Modes: Ad-Hoc vs Dedicated Kanban

- **Ambient Context (Default Chat)**:
  - When working without `/kanban`, `kanban.md` is read purely for ambient context (active branch, pending work, blockers).
  - The user is completely free to prompt ad-hoc requests, implement code, or ignore the board entirely.
  - **Auto-Reconciliation**: If an ad-hoc implementation completes or satisfies an existing card on the board, check it off and report it in the final summary.
  - **Optional Capture**: If the user finishes a significant standalone feature or bugfix not tracked on the board, briefly ask: *"Add this task to kanban.md?"* (never ask for trivial edits or quick lookups).
- **Dedicated Kanban Mode (`/kanban`)**:
  - Opt-in workflow for structured task refinement, prioritization, brainstorming, backlog review, and board scaffolding.

---

## 4. Core Workflows

### A. Task Refinement
- Break large, ambiguous cards into concrete, testable sub-tasks (`- [ ]`).
- Define explicit completion criteria (e.g., tests passing, specific command verifying output).
- Split cards exceeding ~5 sub-steps into distinct logical tasks.

### B. Prioritization & Progression
- Order `Next Up` using the decision policy:
  1. Blockers, security, and stability issues.
  2. Explicit user-requested tasks.
  3. Tasks that unblock multiple follow-up tasks.
  4. Refactors, polish, and optimizations.
- **Do NOT automatically move tasks from `Next Up` into `In Progress`**: Always ask the user to confirm the next task before starting work.
- **Branch / Worktree Concurrency**: To execute cards (one or several at once), hand off to the `work` skill (e.g., `/work let's do 3 kanban tasks`). It runs each card in a subagent with its own worktree/branch and tags cards `(branch: agent/<slug>)`. `/kanban` itself plans; it doesn't implement in parallel.
- If an active task is blocked by missing inputs/credentials, move it to `Backlog` with `(blocked: <reason>)` and prompt the user only for the blocker.

### C. Planning & Ideation
- When requested to plan or brainstorm:
  - Inspect recent git commits (`git log -n 5 --oneline`) and recent code modifications.
  - Check open issues or documentation notes.
  - Propose 2–3 high-value, actionable items for `Backlog` or `Next Up`.

### D. Backlog Accuracy Review
- Fact-check backlog items against the actual codebase and filesystem.
- Prune obsolete tasks, resolved questions, or already-completed items that were not logged.
- Consolidate overlapping or redundant cards.

### E. Task Completion & Documentation Discipline
When a task is completed:
1. **Synchronize Documentation**:
   - **Always inspect and update `README.md` and related documentation together with code changes**.
   - Documentation must never lag behind code; every new flag, command, architectural shift, or config change must be reflected in docs immediately.
2. **Log to Completed**:
   - Move the card to `## Completed` with today's date stamp:
     `- [x] **YYYY-MM-DD — <Task Title>**`
3. **Prune Stale Completed Items**:
   - Keep only the recent ~5–10 completed tasks (or tasks from the last 14–30 days).
   - Delete older completed items from `kanban.md`.
   - **Git history is the permanent historical record**; do not hoard unbounded completed items or maintain redundant `changelog.md` files unless explicitly requested.

### F. Scaffolding a New Board
When creating a new `kanban.md`:
1. Use the [Kanban Template](./references/kanban-template.md).
2. Populate the board with 2–4 realistic initial tasks inferred from the project's `README.md`, `git status`, or user instructions.
3. Save to the chosen directory and present the new board for user confirmation.

---

## References

- [Kanban Template](./references/kanban-template.md): Baseline template for scaffolding new `kanban.md` boards.
