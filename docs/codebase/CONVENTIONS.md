# Coding Conventions

## Core Sections (Required)

### 1) Naming Rules

| Item | Rule | Example | Evidence |
|------|------|---------|----------|
| Module directories | `NN. descriptive-kebab-case` (two-digit number prefix + space) | `03. gitlab-runner-with-shell-executor` | Root directory listing |
| Docker Compose files | `docker-compose.yml` (lowercase) | `docker-compose.yml` | All modules |
| GitLab CI files (modules 3–5) | `gitlab-ci.yml` (no leading dot — **inconsistent**) | `gitlab-ci.yml` | `03.*/`, `04.*/`, `05.*/` |
| GitLab CI files (modules 7–11) | `.gitlab-ci.yml` (with leading dot — **correct GitLab convention**) | `.gitlab-ci.yml` | `08.*/`, `09.*/`, `10.*/`, `11.*/` |
| Runner config | `config.toml` (lowercase) | `config.toml` | Modules 03–10 |
| Container names | `kebab-case` | `gitlab-server`, `gitlab-runner` | All `docker-compose.yml` files |
| Docker networks | `kebab-case` | `gitlab-in-docker`, `kind` | `05.*/docker-compose.yml`, `06.*/docker-compose.yml` |

---

### 2) Formatting and Linting

- Formatter: **None configured** — no `.prettierrc`, `.editorconfig`, or YAML linter
- Linter: **None configured** — no `yamllint`, `hadolint`, or shellcheck
- Most relevant enforced rules: None (all formatting is manual and inconsistent)
- Run commands: N/A

---

### 3) Import and Module Conventions

- GitLab CI template inclusion (modules 10, 11):
  ```yaml
  include:
    - template: Jobs/Container-Scanning.gitlab-ci.yml
  ```
  This is the standard GitLab `include: template:` pattern — it pulls official GitLab-managed CI templates.
- No other import or module conventions apply (no application code).

---

### 4) Error and Logging Conventions

- CI pipeline failure strategy: Jobs default to `allow_failure: false` (implicit). Modules 10 and 11 explicitly set `allow_failure: false` on scanning jobs to block deployment on vulnerabilities.
- Severity-gated failure pattern (modules 10 and 11):
  ```bash
  HIGH_SEVERITY_COUNT=$(jq '.vulnerabilities | map(select(.severity=="High")) | length' report.json)
  if [[ $HIGH_SEVERITY_COUNT -gt 0 ]]; then exit 1; fi
  ```
- No application logging conventions (no application code).

---

### 5) Testing Conventions

- Test file naming/location rule: **No tests exist** in this repository
- Mocking strategy norm: N/A
- Coverage expectation: N/A

---

### 6) Evidence

- `03. gitlab-runner-with-shell-executor/gitlab-ci.yml` (no leading dot)
- `08. build-docker-images-using-kaniko/.gitlab-ci.yml` (with leading dot)
- `10. scan-container-images-in-registry/.gitlab-ci.yml` (allow_failure + jq pattern)
- `11. scan-dependencies-in-gitlab-ci/.gitlab-ci.yml` (dependency scanning pattern)

## Extended Sections

### Known Convention Violations

| Violation | Location | Impact |
|-----------|----------|--------|
| CI files named `gitlab-ci.yml` (no leading dot) | Modules 03, 04, 05 | GitLab will **not** auto-detect these files — they must be manually specified in GitLab project settings or these are purely documentation samples |
| Docker Compose `version: '3.8'` (deprecated key) | All modules | Triggers deprecation warnings in newer Compose versions; functionally harmless |
| Mixed `docker-compose.yml` vs `docker-compose.yaml` extension | Module 02 uses `.yaml`; all others use `.yml` | Minor inconsistency |
