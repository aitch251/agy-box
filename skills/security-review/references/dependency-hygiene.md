# Dependency & Supply-Chain Hygiene Guide

This guide details procedures for auditing third-party dependencies, detecting known CVEs, and assessing upstream package maintenance across major programming ecosystems.

---

## 1. Ecosystem Manifests & Audit Commands

When evaluating a repository or sub-directory, inspect the manifest files and run native non-destructive audit commands when available:

| Ecosystem | Manifest & Lockfiles | Native Audit / Dependency Inspection |
| :--- | :--- | :--- |
| **Node.js / JS / TS** | `package.json`<br>`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml` | `npm audit --dry-run`<br>`npm outdated` |
| **Go** | `go.mod`<br>`go.sum` | `go list -m -u all`<br>`govulncheck ./...` (if installed) |
| **Python** | `requirements.txt`, `pyproject.toml`<br>`poetry.lock`, `Pipfile.lock` | `pip list --outdated`<br>`pip-audit` (if installed) |
| **PHP / WordPress** | `composer.json`, `composer.lock`<br>`wp-content/plugins/*/readme.txt` | `composer audit`<br>`composer outdated`<br>WPScan database search for plugin slugs |
| **Rust** | `Cargo.toml`<br>`Cargo.lock` | `cargo audit`<br>`cargo outdated` |
| **Ruby** | `Gemfile`<br>`Gemfile.lock` | `bundle audit`<br>`bundle outdated` |

---

## 2. Automated & Online CVE Investigation

When local CLI scanners are absent or restricted by permission boundaries, inspect dependency versions directly and search vulnerability feeds:

1. **Vulnerability Databases**:
   - **OSV (Open Source Vulnerabilities)**: `https://api.osv.dev/v1/query` (covers Go, PyPI, npm, Cargo, Packagist).
   - **GitHub Advisory Database**: Search by package name and ecosystem.
   - **NVD (National Vulnerability Database)**: Search CVE ID or package CPE.
   - **WPScan / Wordfence / Patchstack**: Essential for WordPress plugins, themes, and CMS extensions.

2. **Web Search Formula**:
   Search with targeted queries:
   ```text
   "<package-name>" "<version>" vulnerability CVE
   "<package-name>" security advisory exploit
   "<plugin-slug>" wordpress vulnerability WPScan
   ```

3. **Severity Assessment**:
   - **Critical (CVSS 9.0–10.0)**: Remote code execution (RCE), unauthenticated SQL injection, unauthenticated file upload, deserialization.
   - **High (CVSS 7.0–8.9)**: Local file inclusion (LFI), broken access control / privilege escalation, authentication bypass, stored XSS in administrative contexts.
   - **Medium (CVSS 4.0–6.9)**: Authenticated reflected XSS, CSRF, information disclosure, rate-limiting flaws.
   - **Low (CVSS 0.1–3.9)**: Denial of service requiring heavy resources, path disclosure, verbose error messages.

---

## 3. Package Maintenance & Staleness Evaluation

A package with zero known CVEs can still be dangerous if it is abandoned, as new security flaws will never be patched.

### Red Flags for Abandoned Packages
- **Time Since Last Release**: No release or commit in > 18–24 months.
- **Repository Archived**: GitHub/GitLab repository is marked `Archived` or `Read-Only`.
- **Registry Deprecation**: Package page on npmjs.com, PyPI, or Packagist displays a deprecation or "unmaintained" notice.
- **Unaddressed Security Issues**: Open GitHub issues or pull requests detailing security bugs with zero maintainer response.
- **Single-Maintainer Bus Factor**: Project maintained by an unresponsive single developer with no succession plan.

### Remediation for Stale Packages
1. **Active Fork**: Check if the community has maintained an active, vetted fork under a new namespace.
2. **Modern Alternative**: Replace legacy packages (e.g. replacing unmaintained hit counters with lightweight modern analytics like Koko Analytics or Plausible).
3. **In-Tree Replacement**: If the package provides trivial functionality, reimplement it directly within the project to drop the external supply-chain surface.

---

## 4. Supply-Chain Integrity Best Practices

- **Lockfile Enforcement**: Never commit bare manifests without their accompanying lockfile (`package-lock.json`, `go.sum`, `Cargo.lock`, etc.). Ensure deployment pipelines verify cryptographic hashes.
- **Dependency Minimization**: Audit direct dependencies regularly. Remove unneeded transitive bloat or unused administrative tools (e.g., `wordpress-importer` left enabled in production).
- **Private Registry / Typo-Squatting**: Verify package namespaces and origins to ensure internal packages cannot be hijacked via public namespace confusion.
