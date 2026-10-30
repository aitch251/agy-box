# AI Agent Guidelines (`AGENTS.md`)

Operational rules and CLI conventions for the `agy-box` repository.

---

## General Principles

- **No AI bloat**: Skip filler words; keep responses short, direct, and DRY.
- **Format**: Bullet points preferred over long explanations. Group or prefix by repository (`agy-box: file`).
- **Documentation**: Update [`README.md`](README.md) whenever CLI flags, profiles, or workflows change.
- **Single source of truth**: Real documentation (architecture, setup, CLI usage) belongs in `README.md` or `docs/` for humans. `AGENTS.md` is strictly for agent rules, CLI constraints, and allowlists. Never duplicate facts across them.

---

## How to Run Commands

- **Avoid `-C` flags**: Run commands directly in the working directory so they match CLI allowlists.
- **No in-place shell editing**: Never use `sed`, `awk`, `perl`, or shell redirection to edit files. Use `edit_file` or `write_file`.
- **Git workflow & safety**:
  - Present diffs for review and ask to commit + push (`yes`/`no` or `y`/`n`).
  - Never run destructive git commands (`git reset --hard`, `git checkout .`, `git clean -fd`).

---

## Repository Layout

- [`agy-box`](agy-box): Executable zsh launcher (`install`, `codium`, `status`, `wp`, default repo sandbox).
- [`profiles/sandbox-strict.json`](profiles/sandbox-strict.json): Canonical `Deny > Ask > Allow` security policy.
- [`prompts/wp-creator.md`](prompts/wp-creator.md): System prompt template for WordPress essay & theme collaboration.
- [`skills/`](skills/): Portable skills bundle (`kanban`, `work`, `security-review`, `streamline-agy`).
- [`codium/`](codium/): Barebones VSCodium configuration (`settings.json`, `keybindings.json`).
