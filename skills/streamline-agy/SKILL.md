---
name: streamline-agy
description: >-
  Audits, tightens, and manages Antigravity CLI configurations (settings.json, mcp_config.json, trusted workspaces, and AGENTS.md).
  Use this skill to audit permission allowlists against least-privilege standards, scan repositories to discover needed commands,
  safely add or remove scoped permissions (e.g. curling specific domains or running builds), inspect MCP server security,
  or run a complete AGY health and confinement check. Trigger with /streamline-agy.
---

# Streamline AGY (Configuration & Permission Coordinator)

Orchestrates the hygiene, security confinement, and repository alignment of **Google Antigravity CLI** (`agy`) configuration state.

---

## 1. Operating Modes

`/streamline-agy` operates in four distinct modes based on user prompt:

| Mode | Trigger Example | Primary Action |
| :--- | :--- | :--- |
| **Full Audit** *(Default)* | `/streamline-agy` | Comprehensive audit of `settings.json`, `mcp_config.json`, workspaces, and repo alignment (§2). |
| **Add Permission** | `/streamline-agy add permissions to curl website xyz` | Confinements-check, constructs the narrowest possible permission, and applies (§3). |
| **Repo Scan** | `/streamline-agy repo scan [repo]` | Scans repositories in `~/git/*` for test/lint/build commands, proposes allowlist entries, and syncs `AGENTS.md` (§4). |
| **Prune / Tighten** | `/streamline-agy tighten` or `prune` | Identifies stale, overly broad, or redundant permissions and prunes them safely via `--remove` (§5). |

---

## 2. Full Audit Workflow (Default)

When invoked without specific arguments, run all audit checks sequentially:

### Step 1: Permission Allowlist Confinement Audit
Inspect `agy/settings.json` against [Security Standards](./references/security-standards.md):
- **Overly Broad Commands**: Flag bare execution interpreters (`sh`, `bash`, `python`), unconfined network callers (`curl` or `wget` without URL/flags), or open destructive utilities (`rm`, bare `git push`).
- **Dead / Orphaned File Paths**: Verify each `read_file(...)` and `write_file(...)` path exists on disk. Flag deleted or renamed files.
- **Symlink Dual-Registration**: Ensure all shared skill files are allowlisted under both their source repository path and `~/.gemini/config/skills/...`.

### Step 2: Trusted Workspaces & Containment Audit
- Verify `trustedWorkspaces` contains valid, active repository roots.
- Flag any `write_file` or `command` operating outside trusted workspaces (e.g., home directory dotfiles, external system paths).

### Step 3: MCP Server & Tool Audit
- Read `agy/mcp_config.json`.
- Verify URLs use secure transports (`https://`, `wss://`).
- Check that all MCP tools defined or granted in `settings.json` have active corresponding servers.

### Step 4: Presentation & Actionable Remedies
Present findings concisely in bullet points grouped by risk severity:
- **High (Overly Broad / Risky)**: Propose narrowed replacement prefixes.
- **Medium (Stale / Redundant)**: Offer batched pruning candidates.
- **Low (Formatting / Sync)**: Note pending runtime or git sync items.
- Ask for user confirmation (`y/n`) before applying any modifications.

---

## 3. Safe Permission Addition Workflow

When asked to allow specific capabilities (e.g., *"add permission to curl api.github.com"*):

1. **Enforce Least Privilege & Flag Scoping**:
   - Never allow unconstrained broad binaries (`command(curl)`).
   - Require explicit domain or flags:
     - Good: `command(curl -s https://api.github.com)` or `command(curl -I http://localhost:8080)`
     - Rejected: `command(curl)` or `command(curl http*)`
2. **Apply via Runtime Merge**:
   - Append the new rule into `agy/settings.json`.
   - Validate JSON syntax: `python3 -m json.tool agy/settings.json > /dev/null`.
   - Run `./scripts/sync-runtime.sh --apply` to merge into live runtime (`~/.gemini/antigravity-cli/settings.json`).
3. **Present for Review & Git Commit**:
   - Show the exact added line and diff.
   - Follow standard review flow (`y/n`) before committing to git.

---

## 4. Repository Scan & Tooling Discovery

When invoked with `repo scan` (or across all repos in `~/git/*`):

1. **Inspect Repositories**:
   Walk `~/git/*` (`local`, `action-data`, `aws`, `hack-server`, `wordpress`, etc.).
2. **Discover Tooling Commands**:
   - Inspect `package.json`, `Makefile`, `pyproject.toml`, `go.mod`, `docker-compose.yml`.
   - Extract test, lint, format, and build commands (e.g., `make check`, `pytest`, `npm test`, `go test ./...`).
3. **Compare Against Allowlist**:
   - Flag standard commands missing from `settings.json` that agents frequently fail on.
4. **Align `AGENTS.md`**:
   - Verify each repo's `AGENTS.md` accurately documents available verification commands.
   - Propose additions to `settings.json` and updates to `AGENTS.md` in one combined review.

---

## 5. Pruning & Deprecation Workflow

When removing obsolete or overly broad permissions:
1. Always use `./scripts/sync-runtime.sh --remove '<entry>'` (or `agy-remove-permission`).
   - *Rationale*: Because the sync script performs a union merge, deleting from only `agy/settings.json` or only runtime causes the removed item to be re-added on the next shell sync. `--remove` prunes both sides simultaneously.
2. Present pruned items and ask for confirmation before committing the resulting `agy/settings.json` diff.

---

## References

- [Security & Confinement Standards](./references/security-standards.md): Risk tiers, confinement rules, and stack command reference.
