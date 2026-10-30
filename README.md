# `agy-box`

Sandboxed launcher and security policy for Google Antigravity CLI (`agy`) across all repositories, with a built-in WordPress shortcut.

---

## 1. Setup

```bash
cd ~/git/agy-box
./agy-box install
agy-box status
```

- Symlinks `agy-box` into `~/.local/bin/agy-box`.
- Merges `deny`, `ask`, and `allow` rules from [`profiles/sandbox-strict.json`](profiles/sandbox-strict.json) into `~/.gemini/antigravity-cli/settings.json` (and `~/git/local/agy/settings.json`).

---

## 2. Default Usage (Any Repository)

Run inside any repository (`action-data`, `local`, `hack-server`, `aws`, `wordpress`) or pass the repo path directly:

```bash
# Option A: From inside any repo
cd ~/git/action-data
agy-box

# Option B: Pass repo path directly
agy-box ~/git/hack-server
```

- Resolves git repository root automatically.
- Launches `agy --sandbox --mode auto --settings profiles/sandbox-strict.json`.
- Zero prompts for local edits, tests, builds, and git commits (`pytest`, `ruff`, `npm test`, `npm run build`, `go test`, `cargo test`, `make`, `git commit`).
- Blocks writes outside the active repository and blocks sensitive files (`.env`, `~/.ssh`, `~/.aws`, `*.pem`).

---

## 3. WordPress Mode (`agy-box wp <site>`)

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

## 4. Security Boundaries (`profiles/sandbox-strict.json`)

| Tier | Behavior | Scope |
| :--- | :--- | :--- |
| **OS Sandbox** | Kernel-enforced (`--sandbox`) | Confines shell writes to current repo root + `/tmp`. |
| **`deny`** | Hard-blocked (never runs) | `*.env*`, `*wp-config.php`, `*.pem`, `*credentials*`, `~/.ssh/*`, `~/.aws/*`, `~/.gnupg/*`, `sudo*`, `rm -rf /*`, `rm -rf ~*`, `rm -rf .*`, `git push --force*`, `git reset --hard*`, `git clean -fd*`, `wpctl decommission*`, `wpctl prune*`. |
| **`ask`** | Pauses for `y/n` confirmation | `git push*`, `gh pr merge*`, `gh repo delete*`, `ssh *`, `aws *`, `cdk deploy/destroy*`, `terraform apply/destroy*`, `wpctl deploy*`, `wpctl post/page/media push*`, `wpctl update-core*`, `wpctl clean-malware*`, `wpctl rollback*`. |
| **`allow`** | Automatic (zero prompts) | Local file reads/edits, `git status/diff/add/commit/fetch`, `pytest`, `ruff`, `npm test/build/lint`, `go test/build/vet`, `cargo test/check`, `make build/check/test/lint/dev`, `docker compose up/down/ps/logs`, `wpctl health/state/list/pull/post pull/page pull`. |
