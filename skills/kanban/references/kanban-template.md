# Kanban Board Canonical Template

Use this template when scaffolding a new `kanban.md` in any repository root or sub-project folder.

```markdown
# <Project Name> Kanban (`kanban.md`)

> **Workflow**: Track all ongoing, planned, and completed tasks for `<project>`. Keep documentation (`README.md`, etc.) in sync with code changes.

---

## In Progress

<!-- 1 active task per branch / worktree. Ask to create a branch & worktree from main if already active. Tag with (branch: <name>) when multi-branching. Leave empty when idle. -->

---

## Next Up

<!-- 2-4 prioritized actionable tasks ready for execution. -->
- [ ] **<Next Task 1>**
  - [ ] <Step 1>
  - [ ] <Step 2>

- [ ] **<Next Task 2>**
  - [ ] <Step 1>
  - [ ] <Step 2>

---

## Backlog

<!-- Ideas, future work, or blocked items. -->
- [ ] **<Backlog Task 1>**
- [ ] **<Blocked Task>** `(blocked: <reason>)`

---

## Hold

<!-- Paused tasks awaiting external input or decisions. -->

---

## Completed

<!-- Recent ~5-10 completed tasks with date stamps. Older items pruned. -->
```
