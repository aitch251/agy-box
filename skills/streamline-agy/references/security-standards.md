# Antigravity Permissions & Security Standards

Guidelines and rules for evaluating, tightening, and managing `settings.json`, `mcp_config.json`, and `AGENTS.md` alignment.

---

## 1. Permission Safety Categories

Permissions in `permissions.allow` are evaluated using three risk tiers:

### Tier A: Inherently Broad / High-Risk (Requires Confinement)
- **Arbitrary network callers**:
  - Broad `command(curl)` or `command(wget)` permits arbitrary outbound downloads, credential exfiltration, or blind SSRF.
  - *Standard*: Must be scoped to exact flags and domain/origin (e.g., `command(curl -s -I https://api.github.com)` or `command(curl -s http://localhost:8080)`).
- **Execution shells / Interpreters**:
  - `command(bash)`, `command(sh)`, `command(zsh)`, `command(python)`, `command(node)` without subcommands/scripts allow executing arbitrary unreviewed code.
  - *Standard*: Disallow unconfined interpreters. Allow only specific scripts (e.g. `command(python3 ./scripts/verify.py)`).
- **Destructive filesystem utilities**:
  - `command(rm)`, `command(unlink)`, `command(shred)` without strict paths.
- **Unconfined Git write commands**:
  - Broad `command(git)` or `command(git push)` permits pushing to `main` or force-pushing.
  - *Standard*: Explicit prefix targeting: `command(git push -u origin agent/)`, `command(git fetch origin)`, `command(git rebase origin/main)`.

### Tier B: Contextual / Medium-Risk (Audited for Specificity)
- **Package managers**:
  - `npm`, `pip`, `brew`, `composer`, `go`.
  - Prefer safe subcommands: `npm test`, `npm list`, `go test`, `brew list`, `brew info`.
  - Scrutinize install/update commands (`npm install`, `brew install`) that run arbitrary lifecycle scripts.
- **Process & diagnostic tools**:
  - `ps`, `lsof`, `kill`. `kill` must be tightly audited.

### Tier C: Safe Read-Only Defaults (Standard CLI)
- Standard inspection utilities: `ls`, `cat`, `head`, `tail`, `grep`, `diff`, `stat`, `file`, `df`, `du`, `uname`, `which`, `wc`, `sort`, `uniq`.
- Read-only git queries: `git status`, `git diff`, `git log`, `git branch`, `git rev-parse`.

---

## 2. Path Containment Rules

1. **`read_file` / `write_file`**:
   - Must point inside declared `trustedWorkspaces` or known shared paths (`~/.gemini/config/skills/`, `/Users/.../git/...`).
   - Flag paths into sensitive directories: `~/.ssh/`, `~/.aws/`, `~/.gnupg/`, `~/.bash_history`, `~/.zsh_history`.
2. **Dual-Path Registration for Shared Skills**:
   - Global skills in `~/.gemini/config/skills/` may be symlinks to a versioned repository.
   - Both the canonical target path and the symlinked location (`~/.gemini/config/skills/...`) must be registered so agent invocations resolve cleanly.

---

## 3. MCP Configuration Standards

1. **Endpoint Transport Security**:
   - Remote MCP servers must use HTTPS or secure WebSockets (`wss://`). Flag unencrypted HTTP for non-localhost endpoints.
2. **Local MCP Scripts**:
   - Verify local MCP server scripts reside within tracked repository roots or trusted paths.
3. **Tool Allowance Synchronization**:
   - If MCP tools require permission approval, ensure corresponding `mcp(<server>/<tool>)` rules in `settings.json` are minimal and match only intended tool names.

---

## 4. Repository Scan & Tooling Alignment Matrix

When scanning a repository (`~/git/<repo>`), discover standard test, build, lint, and runner commands from manifest files:

| Technology / Stack | Indicator Files | Candidate Permissions |
| :--- | :--- | :--- |
| **Node / JavaScript** | `package.json` (`scripts`) | `npm test`, `npm run lint`, `npm run build` |
| **Python** | `pyproject.toml`, `requirements.txt`, `pytest.ini` | `pytest`, `python3 -m pytest`, `ruff check`, `ruff format` |
| **Go** | `go.mod` | `go test ./...`, `go build`, `go vet`, `go mod tidy` |
| **Make** | `Makefile` | Phony targets: `make test`, `make lint`, `make check`, `make build` |
| **Docker** | `Dockerfile`, `docker-compose.yml` | `docker compose up -d`, `docker compose ps`, `docker compose down` |
| **Rust** | `Cargo.toml` | `cargo test`, `cargo check`, `cargo clippy` |
