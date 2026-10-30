---
name: work
description: >-
  Main-chat orchestrator that delegates real coding work to subagents, each in its own git worktree and branch, ending in one
  pull request per task (works for 1 or many tasks). Use this skill on "/work" or "work" followed by free-form text: plain "/work"
  gives a state review with next-step suggestions; "/work <one or more task descriptions>" or "/work do N kanban tasks" dispatches
  the work to subagents.
---

# Work (Main Chat Orchestrates, Subagents Implement)

The main chat (orchestrator) stays interactive and does **no implementation itself**. Each task, even a single one, runs in a subagent: **research → plan → worktree/branch → implement → verify → PR → report**.

## Modes

`/work` takes free-form natural language. There are no sub-commands; work out the intent:

| Example | Intent | Behavior |
| :--- | :--- | :--- |
| `/work` | **Review** | Status review + suggested next steps (§2.R). Starts nothing. |
| `/work fix the login redirect; add dark mode to settings` | **Run given tasks** | Split into tasks (1..N), echo the task list, dispatch right away (§2.A–E). |
| `/work i need a script which can resize all images in ./assets` | **Run one task** | A single described need = one task. Dispatch right away. |
| `/work let's do 3 kanban tasks. what's next?` | **Run from board** | Pick the top N `Next Up` cards from `kanban.md` (N given, or as many as are independent, max 4), propose them, dispatch after one confirmation (§2.A–E). |

**Intent parsing:**
- **Task boundaries**: separate goals = separate tasks. Several sentences about the same goal = one task. If the split changes the outcome (overlap, ordering), state the chosen split in the echo.
- **Explicit tasks**: no confirmation round trip. Echo `slug | goal` and dispatch. Ask first only if a goal or acceptance criterion is genuinely ambiguous (e.g. "fix the bug" without saying which one).
- **Board-based or suggestion-based** ("what's next?", "pick something"): always propose first, then dispatch after one confirmation. The kanban rule is never to move cards unasked.
- **Mixed messages**: answer questions inline, and dispatch only the actual work.
- **Not a task**: quick lookups or explanations are answered directly without a subagent.

---

## 1. Ground Rules

- **No user prompts mid-flight**: Subagents only run commands from the allowlist in `~/.gemini/antigravity-cli/settings.json`. If a task needs something else, the subagent files a **permission request** (§4) right away and keeps going on unblocked work. It never triggers the prompt itself.
- **No `cd` / `-C`**: Worktrees come from `Workspace: share`, so the subagent's cwd *is* the worktree and plain `git …` commands match the allowlist.
- **Git autonomy in worktree branches**: Subagents commit, push and open PRs without asking. The PR review is the approval gate. Never push to `main`, never force-push, never merge.
- **Never kill a subagent before its branch is pushed**: Killing deletes the worktree **and** the local branch.
- **Main chat stays light**: it only reads, plans, dispatches, reviews reports and asks the user. Its own commits are limited to `kanban.md`/`settings.json` bookkeeping (normal y/n flow).

---

## 2. Orchestrator Workflow

### R. Review Mode (plain `/work`)
Gather state (all allowlisted, read-only):
```bash
git status && git log -n 10 --oneline        # local state, recent work
git branch --list 'agent/*'                  # local agent branches
gh pr list --author @me                      # open PRs
gh pr checks <n>                             # CI for each open agent/* PR
```
Plus: `manage_subagents list` (running subagents; only this chat's, so open PRs and `agent/*` branches cover work from other chats), `kanban.md` (`In Progress`, `Next Up`, blocked items) if present, and pending entries in `scratch/permission-requests.jsonl`.

Output (short, bullets):
- **In flight**: running subagents / open PRs with CI status; PRs ready to merge or failing.
- **Attention**: blocked tasks, pending permission requests, stale branches, dirty working tree.
- **Next up**: 2–3 concrete suggestions (kanban `Next Up` first; otherwise inferred from recent commits, TODOs, failing CI), each phrased as a ready-to-run `/work <task description>`.
- End by asking which to start. Never dispatch in review mode.

### A. Preflight (Run modes, once)
```bash
git status                 # main worktree clean? (base = current local HEAD)
git fetch origin           # base freshness; subagents rebase onto origin/main
gh auth status             # PR creation possible?
git remote -v              # remote exists?
```
Stop and report if there's no remote or `gh` isn't authenticated.

### B. Task Intake & Split
1. Gather tasks from the free-form prompt, or from `kanban.md` (`Next Up`) if the user asked to pull from the board. When pulling from the board, show the proposed selection and confirm once (the board rule: never move cards to `In Progress` unasked).
2. For each task, write down: **goal**, **acceptance criteria**, **files/areas likely touched**, **verify command**. If a criterion is unclear, ask before dispatching. A clarification now is cheaper than a wrong PR.
3. **Overlap check** (only when there are 2+ tasks): Tasks that touch the same files, lockfiles, migrations or shared config → merge into one task or run them one after another (the second branches off after the first is merged). Tell the user which tasks were serialized.
4. **Concurrency cap**: ≤ 4 subagents at once (rate limits, CPU, port collisions from dev servers).
5. Slug each task: `[a-z0-9-]`, ≤ 30 chars → branch `agent/<slug>`.

### C. Dispatch
- For each task: `invoke_subagent` with `TypeName: self`, `Workspace: share`, `Role: "<Slug> Implementer"`, and a prompt built from the [Subagent Prompt Template](./references/subagent-prompt.md).
- Kanban (if present): move cards to `In Progress`, tagged `(branch: agent/<slug>)`.
- Return control to the user right away with a table: slug | branch | status. The user can keep chatting or start more work.

### D. Monitor
- Don't poll; wait for subagent messages.
- **Plan checkpoint** (optional, for risky/ambiguous tasks): the subagent sends its plan and waits; the orchestrator approves or corrects it (escalate to the user only if a real product decision is needed).
- Answer a blocked subagent's questions from context where possible; forward to the user only when needed.
- On `permission-request` messages: append them to the request log and offer the user a review whenever they want (see §4). Don't wait for the end if a subagent is blocked.
- **Review fixes**: when the user wants changes to an open PR, `send_message` the same subagent if it's still alive (same worktree). Otherwise dispatch a new one that checks out the PR branch (`gh pr checkout <n>`).

### E. Collect & Wrap Up
1. Check each report: PR URL present, verification result, and the `needs-user` list.
2. Check CI: `gh pr checks <url>`.
3. Kill finished subagents (only after the PR URL is confirmed) to free their worktrees.
4. Kanban: annotate cards with the PR URL (check off only after merge).
5. **Final batch ask**: one message with all still-pending permission requests and `needs-user` items (secrets, product decisions). Run approved actions last.
6. Final recap: bullet list of PR links + CI status (per `AGENTS.md` summary rules).

---

## 3. Subagent Workflow (summary)

Full instructions live in the [prompt template](./references/subagent-prompt.md).

1. **Research**: read the relevant code, `AGENTS.md`, `README.md`, recent `git log`.
2. **Plan**: short checklist; send it to the orchestrator if a checkpoint was requested.
3. **Branch**: `git branch -m agent/<slug>`, `git fetch origin`, `git rebase origin/main`.
4. **Implement**: small, topic-split commits; update docs in the same branch.
5. **Verify**: run tests/lint/build from the allowlist; fix failures; self-review `git diff origin/main`.
6. **PR**: `git push -u origin agent/<slug>` → `gh pr create --base main --fill-first` (or `--title/--body`). Use `--draft` if verification is incomplete.
7. **Report**: send the structured report to the orchestrator.

---

## 4. Permission Requests

**Transport**: the subagent sends a `send_message` to the orchestrator. **Record**: the orchestrator appends it to `<orchestrator artifact dir>/scratch/permission-requests.jsonl`.
- Why not have subagents write a tmp file? Writing outside their own worktree may itself trigger a prompt, and a file inside the worktree could get committed by accident. Messages arrive in real time, and with one writer there are no races.

**Subagent message** (one line per missing permission, exact allowlist syntax):
```text
permission-request: {"task":"<slug>","permission":"command(npm ci)","scope":"allowlist|once","reason":"install deps to run tests","blocking":true,"tried":"npm ci"}
```
- `permission` uses the same syntax as `settings.json`: `command(<prefix>)`, `read_file(<abs path>)`, `write_file(<abs path>)`, `mcp(<server>/<tool>)`.
- Request the **narrowest** prefix (`command(npm ci)`, not `command(npm)`).
- `scope: once` = a one-off action, not an allowlist entry (the orchestrator runs it after approval).

**Orchestrator review** (whenever the user asks, or right away if anything is `blocking`):
1. Dedupe the log, then flag risky entries: broad prefixes (`command(git)`, `command(npm)`, `command(bash)`), destructive commands (`rm`, `reset`, `--force`), network/exec (`curl`, `sh -c`, `sudo`), secret paths.
2. Ask the user with `ask_question` (multi-select): one option per request → `permission — task — reason`, marking risky ones and offering narrower alternatives.
3. For approved `allowlist` entries:
   - Add them to `agy/settings.json` (`~/git/local`), then run `./scripts/sync-runtime.sh --apply`. It merges (union) both sides, so runtime-only approvals survive. Show the user its `+=` output so any unrelated additions are visible.
   - To revoke an entry: `./scripts/sync-runtime.sh --remove '<entry>'` (removes it from both files).
4. Mark each log entry `approved|denied`, then `send_message` the affected subagents: `permission-granted: <permission>` (retry) or `permission-denied: <permission>` (work around it, or finish as draft).
5. Commit `agy/settings.json` together with the skill/workflow commits (normal review → y/n flow).

> [!NOTE]
> Unverified: whether a running subagent picks up applied settings live. If a retried command still prompts, the subagent reports `blocked`; the orchestrator re-dispatches it in a fresh subagent after the settings are applied.

---

## 5. Known Limitations

- **Base is local HEAD**: `Workspace: share` branches off the orchestrator's current commit; the subagent rebase onto `origin/main` fixes stale bases.
- **Untracked/ignored files are absent**: `.env`, `node_modules`, `.venv` don't exist in a fresh worktree. Installing dependencies may need non-allowlisted commands → report as `needs-user` or skip.
- **Merge conflicts between PRs**: when PR #1 merges, later PRs may need `git rebase origin/main`. Resume that subagent or start a small follow-up task.
- **Allowlist matching (unverified)**: `agy/settings.json` allows `git push -u origin agent/` to scope pushes to agent branches. If matching is token-based rather than string-prefix, `agent/<slug>` won't match and pushes will prompt → widen to `git push -u origin` (which then also matches `main`; only the subagent rules prevent that).

---

## References

- [Subagent Prompt Template](./references/subagent-prompt.md): copy-paste prompt plus report format for each task subagent.
