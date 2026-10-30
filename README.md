# `agy-box`

Sandboxed launcher, `Deny/Ask/Allow` security policy, portable skills bundle, and VSCodium setup for Google Antigravity CLI (`agy`).

---

## 1. Quickstart (Install Everything)

```bash
git clone https://github.com/aitch251/agy-box.git ~/git/agy-box
cd ~/git/agy-box
./agy-box install --codium
agy-box status
```

- **CLI**: Symlinks `~/.local/bin/agy-box`.
- **Policy**: Merges `deny`, `ask`, and `allow` rules from [`profiles/sandbox-strict.json`](profiles/sandbox-strict.json) into `~/.gemini/antigravity-cli/settings.json`.
- **Skills**: Installs `/kanban`, `/work`, `/security-review`, and `/streamline-agy` into `~/.gemini/config/skills/` and registers `read_file` permissions automatically.
- **VSCodium (`--codium`)**: Installs barebones VSCodium (via Homebrew if missing) and merges [`codium/settings.json`](codium/settings.json) + [`codium/keybindings.json`](codium/keybindings.json).

---

## 2. Barebones VSCodium Setup & Workflow

### Install / Update VSCodium Config Only
```bash
agy-box codium
```

### What Gets Configured
- **Telemetry disabled**: `telemetry.telemetryLevel: "off"`, `redhat.telemetry.enabled: false`.
- **Clean UI**: No startup welcome page (`workbench.startupEditor: "none"`), word-wrap on, 2-space tabs, trailing whitespace trimmed.
- **Terminal Profiles**:
  - `agy-box (Sandboxed AI)`: Runs `agy-box` in the open workspace folder.
  - `agy-box wp (WordPress)`: Runs `agy-box wp` directly.
- **Shortcuts**:
  - `Cmd+Shift+A`: Opens a new editor-tab terminal running `agy-box` side-by-side with your files.
  - `Shift+Enter`: Inserts a newline in the `agy` prompt without submitting.

### Using VSCodium + `agy-box`
1. Open any repository in VSCodium:
   ```bash
   codium ~/git/<repo>
   ```
2. Press **`Cmd+Shift+A`** (or open integrated terminal with `` Ctrl+` `` and type `agy-box`).
3. Edit files side-by-side while `agy-box` runs sandboxed in the terminal panel.

---

## 3. Commands

| Command | Action |
| :--- | :--- |
| `agy-box [path]` | Run `agy --sandbox --mode auto` locked to current or target git repo |
| `agy-box wp <site> [port]` | Start Docker preview + sandboxed `agy` in `~/git/wordpress` (default port: `8080`) |
| `agy-box install [--codium]` | Install symlink, permission profile, skills, and optionally VSCodium config |
| `agy-box codium` | Install VSCodium (if missing) and apply [`codium/`](codium/) settings/keybindings |
| `agy-box status` | Print binary, skill, VSCodium, and rule counts |

---

## 4. Bundled Skills (`skills/`)

Installed globally into `~/.gemini/config/skills/` and available in every repository:

| Slash Command | Skill | Purpose |
| :--- | :--- | :--- |
| `/kanban` | [`skills/kanban/`](skills/kanban/SKILL.md) | Track, prioritize, refine, and scaffold tasks in `kanban.md` |
| `/work` | [`skills/work/`](skills/work/SKILL.md) | Delegate 1..N tasks to parallel subagents in isolated git worktrees/branches |
| `/security-review` | [`skills/security-review/`](skills/security-review/SKILL.md) | High-confidence vulnerability, secret hygiene, and dependency audit |
| `/streamline-agy` | [`skills/streamline-agy/`](skills/streamline-agy/SKILL.md) | Audit, tighten, and scan repo commands for `settings.json` |

---

## 5. Default Usage (Any Repository)

```bash
cd ~/git/<repo> && agy-box
# or: agy-box ~/git/<repo>
```
- Resolves git root automatically and launches `agy --sandbox --mode auto`.
- Zero prompts for local edits, tests, builds, and commits (`pytest`, `ruff`, `npm test`, `npm run build`, `go test`, `cargo test`, `make`, `git commit`).
- Blocks writes outside the active repo and blocks sensitive files (`.env`, `~/.ssh`, `~/.aws`, `*.pem`).

---

## 6. WordPress Mode (`agy-box wp <site>`)

### Start Session
```bash
agy-box wp example.com
# Or set WP_DEFAULT_SITE=example.com in ~/.zshrc and run: agy-box wp
```
- Changes directory to `~/git/wordpress`.
- Starts local Docker preview at `http://localhost:8080` (streams media from `https://<site>`).
- Launches sandboxed `agy` preloaded with site paths and frontmatter rules.

### Essays & Posts (`sites/<site>/content/posts/*.md`)
- **Draft post**: `"Create post in <site>: title '...', category TEXTE, tags '...'. Add Lesezeit and structure these notes: ..."`
- **Proofread**: `"Fix typos/punctuation in <post>.md. Do not change wording. Show diff."`
- **Pull from remote**: `./wpctl post pull <site>`

### Static Pages (`sites/<site>/content/pages/*.md`)
- **Edit**: `"Update <page>.md with November 2026 dates."`
- **Pull from remote**: `./wpctl page pull <site>`

### Design & Theme (`sites/<site>/wp-content/themes/`)
- **CSS / Layout**: `"In active theme for <site>: set background to #faf8f5, body font-size 18px, line-height 1.7."`
- **Preview**: Refresh `http://localhost:8080`.

### Publish to Live Server (Triggers `y/n` Prompt)
- **Posts**: `./wpctl post push <site>`
- **Pages**: `./wpctl page push <site>`
- **Theme / Full Site**: `./wpctl deploy <site>`

---

## 7. Security Boundaries (`profiles/sandbox-strict.json`)

| Tier | Behavior | Scope |
| :--- | :--- | :--- |
| **OS Sandbox** | Kernel-enforced (`--sandbox`) | Confines shell writes to current repo root + `/tmp`. |
| **`deny`** | Hard-blocked (never runs) | `*.env*`, `*wp-config.php`, `*.pem`, `*credentials*`, `~/.ssh/*`, `~/.aws/*`, `~/.gnupg/*`, `sudo*`, `rm -rf /*`, `rm -rf ~*`, `rm -rf .*`, `git push --force*`, `git reset --hard*`, `git clean -fd*`, `wpctl decommission*`, `wpctl prune*`. |
| **`ask`** | Pauses for `y/n` confirmation | `git push*`, `gh pr merge*`, `gh repo delete*`, `ssh *`, `aws *`, `cdk deploy/destroy*`, `terraform apply/destroy*`, `wpctl deploy*`, `wpctl post/page/media push*`, `wpctl update-core*`, `wpctl clean-malware*`, `wpctl rollback*`. |
| **`allow`** | Automatic (zero prompts) | Local file reads/edits, bundled skill docs, `git status/diff/add/commit/fetch`, `pytest`, `ruff`, `npm test/build/lint`, `go test/build/vet`, `cargo test/check`, `make build/check/test/lint/dev`, `docker compose up/down/ps/logs`, `wpctl health/state/list/pull/post pull/page pull`. |
