---
name: security-review
description: >-
  Performs systematic, high-confidence security reviews, vulnerability audits, and dependency supply-chain assessments across any repository or subdirectory.
  Use this skill whenever the user asks for a security review, vulnerability audit, CVE check, dependency health/maintenance assessment, secret hygiene scan, or code hardening analysis.
---

# Universal Security Reviewer

A language-agnostic, high-signal security audit coordinator for any codebase, framework, or sub-directory. Focuses on genuinely exploitable risks, supply-chain health, and configuration hardening while eliminating false positives through data-flow verification.

---

## 1. Scope & Triage Strategy

1. **Establish Audit Boundaries**:
   - **Target Scope**: The specific repository, sub-directory, or diff requested by the user.
   - **Research Scope**: Trace callers, configuration files, and data sources across the broader codebase to verify whether inputs are attacker-controlled.
2. **Identify Tech Stack & Package Managers**:
   - Discover manifest files (`go.mod`, `package.json`, `requirements.txt`/`pyproject.toml`, `composer.json`, `Cargo.toml`, `Gemfile`, `pom.xml`, etc.).
   - Identify runtime configuration, container definitions, and deployment manifests (`Dockerfile`, `docker-compose.yml`, `.env.example`, `.htaccess`, server configs).
3. **Execution Guardrails**:
   - **Defensive Focus**: Identify, verify, and document vulnerabilities and remediation steps. Never generate weaponized exploit payloads.
   - **Non-Destructive**: Use read-only commands for audits. Do not modify remote targets or trigger deployments without explicit user confirmation.

---

## 2. Multi-Tier Audit Workflow

Perform the review across five distinct security vectors:

### Tier 1: Secrets, Credentials & Git Hygiene
- Scan target paths and `.gitignore` for leaked or tracked secrets: API tokens, private keys (`.pem`, `.key`), SSH credentials, database connection strings, and `.env` files.
- Inspect process invocation patterns for secrets leaked via CLI flags (e.g. `mysql -p<pass>` visible in `ps aux` on shared systems).
- Verify file system permissions for configuration files (e.g., `0600` / `0640` for credentials vs world-readable `0644`).

### Tier 2: Package Dependencies & Supply-Chain Health
*Consult [Dependency Hygiene Guide](./references/dependency-hygiene.md) for ecosystem-specific commands and signals.*

- **Known CVEs**: Extract direct and transitive dependencies and check against known vulnerability databases (via ecosystem audit tools or online vulnerability searches: WPScan, OSV, NVD, Snyk).
- **Maintenance & Abandonware**:
  - Check when packages were last updated or released.
  - Flag abandoned, deprecated, or archived packages that lack security maintenance.
  - Identify single-maintainer, unmaintained, or superseded packages (e.g., legacy hit counters, abandoned plugins/libraries).
- **Supply-Chain Integrity**: Verify lockfiles exist (`package-lock.json`, `go.sum`, `poetry.lock`, `Cargo.lock`, `composer.lock`) and enforce checksum verification.

### Tier 3: Data-Flow & Vulnerability Sinks
*Consult [Vulnerability Patterns Reference](./references/vulnerability-patterns.md) for detailed sink signatures.*

Trace external inputs (HTTP parameters, headers, file uploads, CLI arguments, scraped third-party APIs) down to critical execution sinks:
- **Command & Shell Execution**: Check if arguments are concatenated into shell strings (`sh -c`, `bash -c`, `sess.Run`). Verify if argument-quoting (e.g. Go `%q`) safely handles shell expansions (`$VAR`, `$(...)`).
- **Path Traversal & Zip Slip**: Audit archive extraction (`zip`, `tar`) and file uploads. Verify that destination paths use `filepath.Rel` or containment boundaries to block directory traversal (`../`).
- **Injection (SQL, NoSQL, Template, LDAP)**: Verify prepared statements / parameterized queries. Flag unescaped strings interpolated directly into queries.
- **Cross-Site Scripting (XSS)**: Ensure dynamic content scraped from external APIs or user inputs is HTML-escaped before inserting into markup or block templates.
- **SSRF & Network Clients**: Audit HTTP/network requests against user-provided URLs. Check for internal IP restrictions, connection timeouts, and redirect limits.
- **Cryptography & Transport Security**: Flag hardcoded secrets, weak algorithms (MD5/SHA1 used for security tokens), disabled certificate checks (`InsecureSkipVerify: true`), or disabled SSH host key verification (`InsecureIgnoreHostKey`).

### Tier 4: Logic, Authorization & Integrity Blind Spots
- **Authentication & Authorization**: Verify access control gates on public endpoints, AJAX actions, and CLI commands. Ensure low-privileged roles cannot execute administrative routines.
- **Integrity Check Completeness**: Check scanning/verification tools for depth limits (`-maxdepth`), missing file detection, or unchecked arrays (e.g., flagging modified files but ignoring missing files).
- **Default Hardening Constants**: Verify platform security flags (e.g. WordPress `DISALLOW_FILE_EDIT`, `FORCE_SSL_ADMIN`, auto-updates enabled; production `DEBUG=False`).

### Tier 5: Web Server & Infrastructure Configuration
- **Directory Indexing & Inclusion Limits**: Ensure directory listing is disabled (`Options -Indexes`) and include-only directories are blocked from direct browser execution.
- **Attack Surface Minimization**: Remove legacy setup artifacts, test files, demo scripts, and parking page leftovers in webroot.

---

## 3. Findings Classification & Reporting

Evaluate each finding before reporting to eliminate false positives:

| Confidence | Description | Action |
| :--- | :--- | :--- |
| **High / Confirmed** | Vulnerable pattern + verified data-flow / attacker input + missing mitigation. | **Report immediately** with exact file/line reference, impact, and fix. |
| **Medium / Suspicious** | Vulnerable pattern identified, but data source or sanitization path is ambiguous. | Report with explicit tag: `[Needs Verification: <condition>]`. |
| **Low / Hardening** | Defense-in-depth improvement, missing security header, or best practice tweak. | Report under Defense-in-Depth / Hardening section. |

### Output Format

Format the review concisely using bullet points grouped by component/file:

```markdown
### Security Review: <Target Path / Repository>

#### 1. Vulnerability Findings
* **[Severity] Issue Title**
  * `repo: file:line` (`function_or_symbol`)
  * **Exploit Vector**: Explain how input reaches sink and why mitigations are absent.
  * **Remediation**: Concrete code patch or configuration change.

#### 2. Dependency & Supply-Chain Health
* **Vulnerable / Outdated Packages**: Package name, current version, CVE/advisory, fixed version.
* **Maintenance & Abandonware Warnings**: Packages inactive for >2 years, archived upstreams, or unmaintained components.

#### 3. Configuration & Defense-in-Depth Hardening
* Specific configuration adjustments (`wp-config.php`, `.env`, `.htaccess`, headers, permissions).
```

---

## References

- [Dependency Hygiene & Maintenance Guide](./references/dependency-hygiene.md)
- [Universal Vulnerability Patterns & Sinks](./references/vulnerability-patterns.md)
