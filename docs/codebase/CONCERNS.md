# Codebase Concerns

## Core Sections (Required)

### 1) Top Risks (Prioritized)

| Severity | Concern | Evidence | Impact | Suggested action |
|----------|---------|----------|--------|------------------|
| **HIGH** | Root password `Abcd@0123456789` hardcoded across all modules | `02.*/docker-compose.yaml` through `11.*/docker-compose.yml` | Anyone cloning repo can log in to any demo instance; teaches bad security practice to learners | Replace with a `.env` file reference (`${GITLAB_ROOT_PASSWORD}`) and add `.env` to `.gitignore` |
| **HIGH** | Live runner tokens (`glrt-*`) committed to git | `03.*/config.toml` through `10.*/config.toml` | Tokens are in permanent git history; if instance is reachable they could be used to hijack runner | Revoke tokens on GitLab instance; replace values in files with `<YOUR_RUNNER_TOKEN>` placeholders |
| **HIGH** | Shared runner registration token (`r3g1str4t10n`) committed | `07.*/docker-compose.yml` | Allows unauthorized runner registration | Replace with placeholder and document how to obtain it from GitLab Admin |
| **MEDIUM** | Unpinned image tags (`gitlab/gitlab-ce:latest`, `gitlab/gitlab-runner:alpine`) | All `docker-compose.yml` files | Silent version upgrades can break tutorials without warning | Pin to specific versions, e.g. `gitlab/gitlab-ce:17.x.x-ce.0` |
| **MEDIUM** | CI files in modules 03–05 named `gitlab-ci.yml` (no leading dot) | `03.*/gitlab-ci.yml`, `04.*/`, `05.*/` | GitLab cannot auto-detect these; pipeline won't trigger without manual project setting override | Rename to `.gitlab-ci.yml` for consistency |
| **MEDIUM** | `network_mode: host` used on runner — Linux-only | Modules 03–10 `docker-compose.yml` | Compositions silently fail on Docker Desktop (Mac/Windows) — affects most tutorial followers | Add note in READMEs; provide Docker network alternative for non-Linux users |
| **MEDIUM** | Docker socket binding (`/var/run/docker.sock`) without `privileged: true` — gives container root access to host Docker | Modules 04, 07–10 `docker-compose.yml` | Equivalent to root on the Docker host; security risk even in local dev | Document the security implication prominently; offer DinD as the safer alternative |
| **LOW** | Deprecated `version: '3.8'` in Docker Compose files | All `docker-compose.yml` files | Compose v2 ignores this key with a deprecation warning | Remove `version:` key from all files |
| **LOW** | `./gitlab/logs` volume not mounted in most modules | Modules 03, 07 mount logs; modules 04–06, 08–10 do not | Log access inconsistent across modules | Standardize log volume across all modules |

---

### 2) Technical Debt

| Debt item | Why it exists | Where | Risk if ignored | Suggested fix |
|-----------|---------------|-------|-----------------|---------------|
| Copy-pasted `gitlab-server` service block | Each module is a standalone demo | All 9 `docker-compose.yml` files with a server block | Updates (e.g., password change, image pin) must be applied 9 times | Create a shared `base-compose.yml` with a `extends:` or use a templating approach |
| Duplicate `config.toml` files (modules 08–10 reuse module 04's token) | Modules built incrementally for the tutorial | `08.*/config.toml`, `09.*/config.toml`, `10.*/config.toml` | Confusing for learners — same token across different modules implies the same registration | Replace repeated tokens with distinct placeholders per module |
| No validation that configs work end-to-end | Manual tutorial approach | All modules | A broken compose file or CI config goes undetected | Add GitHub Actions workflow to lint YAML and validate Compose syntax |
| `GITLAB_SHARED_RUNNERS_REGISTRATION_TOKEN` env var deprecated in GitLab 16+ | Written for an older GitLab version | `07.*/docker-compose.yml` | Auto-registration may fail on current GitLab CE versions | Update to use the new runner creation API token approach |

---

### 3) Security Concerns

| Risk | OWASP category | Evidence | Current mitigation | Gap |
|------|----------------|----------|--------------------|-----|
| Hardcoded credentials in source | A07: Identification and Authentication Failures | All `docker-compose.yml`, `config.toml` files | `.gitignore` excludes `gitlab/` data but NOT the config files themselves | No secret scanning, no `.env` pattern used |
| Docker socket binding gives container full Docker host access | A05: Security Misconfiguration | Modules 04, 07–10 `docker-compose.yml` | None — documented as intentional | No least-privilege alternative offered (DinD is shown separately) |
| Privileged containers (DinD, k8s executor) | A05: Security Misconfiguration | `05.*/config.toml`, `06.*/config.toml` | None | Documented as required for the pattern; no alternatives shown |
| Kaniko CI pipeline writes Docker Hub credentials as base64 inline | A02: Cryptographic Failures | `08.*/.gitlab-ci.yml` | Placeholder strings used (not real credentials) | Real implementation would use GitLab CI/CD variables — pattern should be documented more clearly |
| `CS_SEVERITY_THRESHOLD: medium` — Low severity issues are suppressed | A06: Vulnerable and Outdated Components | `10.*/.gitlab-ci.yml` | Threshold is configurable | No justification documented for choosing `medium` threshold |

---

### 4) Performance and Scaling Concerns

| Concern | Evidence | Current symptom | Scaling risk | Suggested improvement |
|---------|----------|-----------------|-------------|-----------------------|
| `puma['worker_processes'] = 0` disables Puma clustering | `02.*/docker-compose.yaml`, `10.*/docker-compose.yml` | Reduced throughput — single-process GitLab | Not suitable for any multi-user load | Document this as local-demo-only; remove for production use |
| `concurrent = 1` in all `config.toml` files | All `config.toml` files | Only one CI job runs at a time | Slow pipelines under parallel job load | Increase `concurrent` value when running real workloads |
| No resource limits on any container | All `docker-compose.yml` files | GitLab CE can consume 4–6 GB RAM unconstrained | Host memory exhaustion | Add `deploy.resources.limits` for demo environment awareness |

---

### 5) Fragile/High-Churn Areas

| Area | Why fragile | Churn signal | Safe change strategy |
|------|-------------|-------------|----------------------|
| `07.*/docker-compose.yml` (auto-register) | Uses deprecated `GITLAB_SHARED_RUNNERS_REGISTRATION_TOKEN` API | Multiple recent commits improving this module (`improve`, `refactor` in git log) | Test against current GitLab CE version before publishing |
| Runner `config.toml` files | Contain real registration tokens from past sessions | Repeated across modules with same `glrt-*` value | Replace all with placeholder values; document how to regenerate |
| `.gitlab-ci.yml` pipeline files in modules 08–11 | Referenced directly in GitLab projects; any breaking syntax fails learner pipelines | No automated YAML validation | Add `yamllint` or `docker compose config` check |

---

### 6) `[ASK USER]` Questions

1. **[ASK USER]** Should the hardcoded credentials be replaced with `.env` file references to model best practices for learners, even though it adds setup steps?
2. **[ASK USER]** Is the target audience primarily Linux users? If macOS/Windows users are expected, the `network_mode: host` approach needs an alternative.
3. **[ASK USER]** Are the `glrt-*` runner tokens in the `config.toml` files from decommissioned instances (safe) or potentially live? They should be revoked.
4. **[ASK USER]** Is there a plan to update module 07 for the new GitLab runner registration API (deprecated `GITLAB_SHARED_RUNNERS_REGISTRATION_TOKEN` in GitLab 16)?
5. **[ASK USER]** Would you like to add a GitHub Actions CI workflow for this repo to validate Compose files and YAML syntax on each PR?

---

### 7) Evidence

- Scan output: no TODOs/FIXMEs found in production code
- `02.*` through `11.*` `docker-compose.yml`: hardcoded `GITLAB_ROOT_PASSWORD: "Abcd@0123456789"`
- `03.*` through `10.*` `config.toml`: `token = "glrt-..."` values committed
- `07.*/docker-compose.yml`: `GITLAB_SHARED_RUNNERS_REGISTRATION_TOKEN: r3g1str4t10n`
- `05.*/config.toml`: `privileged = true`
- All `config.toml` files: `concurrent = 1`
