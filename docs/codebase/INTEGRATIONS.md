# External Integrations

## Core Sections (Required)

### 1) Integration Inventory

| System | Type | Purpose | Auth model | Criticality | Evidence |
|--------|------|---------|------------|-------------|----------|
| GitLab CE (self-hosted) | Self-hosted SaaS | Git hosting, CI/CD orchestration, Container Registry | HTTP Basic / runner token | High (central hub) | All `docker-compose.yml` files |
| Docker Hub | Container registry (pull) | Pull base images (`python:alpine`, `docker:latest`, etc.) | Anonymous (public images) | High (pipeline execution) | All `Dockerfile`, `docker-compose.yml` |
| Docker Hub (push) | Container registry (push) | Push built images via Kaniko | Base64-encoded credentials in CI env var | Medium | `08.*/.gitlab-ci.yml` |
| Google Container Registry (GCR) | Container registry (pull) | Pull Kaniko executor image | Anonymous (public) | Medium | `08.*/.gitlab-ci.yml` |
| GitLab Container Registry (self-hosted) | Container registry | Push/pull CI-built images | `CI_REGISTRY_USER` / `CI_REGISTRY_PASSWORD` env vars | High (modules 09–10) | `09.*/.gitlab-ci.yml`, `10.*/.gitlab-ci.yml` |
| kind (Kubernetes in Docker) | Local Kubernetes cluster | Run CI jobs as Kubernetes Pods | Client certificate (cert_file, key_file, ca_file) | Medium | `06.*/config.toml` |
| GitLab CI Security Templates | GitLab-managed CI templates | Container scanning (Trivy), Dependency scanning (Gemnasium) | N/A (embedded in GitLab) | Medium | `10.*/.gitlab-ci.yml`, `11.*/.gitlab-ci.yml` |

---

### 2) Data Stores

| Store | Role | Access layer | Key risk | Evidence |
|-------|------|--------------|----------|----------|
| `./gitlab/config` (host volume) | GitLab server config, secrets, initial root password | Docker volume mount to `/etc/gitlab` | If not excluded by `.gitignore`, secrets could be committed | `.gitignore` (correctly excludes `gitlab/`) |
| `./gitlab/data` (host volume) | All GitLab application data: repos, Postgres, Redis, registry blobs | Docker volume mount to `/var/opt/gitlab` | Large; must be backed up manually | All `docker-compose.yml` files |
| `./gitlab/logs` (host volume) | GitLab server logs | Docker volume mount to `/var/log/gitlab` | Log rotation not configured | Modules 03, 07 `docker-compose.yml` |
| `/certs/client` (Docker volume) | TLS client certificates for DinD TLS | Docker volume shared between runner and dind | Ephemeral — lost on container removal | `05.*/config.toml` |
| `/cache` (Docker volume) | GitLab Runner build cache | Docker volume in runner containers | Cache invalidation not configured | All `config.toml` files |

---

### 3) Secrets and Credentials Handling

- Credential sources: **Hardcoded directly in `docker-compose.yml` and `config.toml`** files — no `.env` files, no secrets manager
- Hardcoded secrets found:
  - Root password: `Abcd@0123456789` — repeated across all modules (`02.*/` through `11.*/`)
  - Root email: `admin@buildwithlal.com` / `admin@BuildWithLal.com`
  - Shared runner registration token: `r3g1str4t10n` (module 07)
  - Actual runner tokens (`glrt-*`): committed in `config.toml` files in modules 03–10 — these are live tokens from real registration sessions
  - Docker Hub credentials: placeholder strings `REGISTRY_USERNAME` / `REGISTRY_PASSWORD` in module 08 CI file (not actual secrets, but the pattern base64-encodes them inline)
- Rotation or lifecycle notes: **Unknown** — no credential rotation policy exists. The `glrt-*` tokens committed to git history cannot be considered secret.

---

### 4) Reliability and Failure Behavior

- Retry/backoff behavior: None configured for any service
- Timeout policy: GitLab healthcheck in module 07: `interval: 60s`, `timeout: 3s`, `retries: 5` (waits up to 5 minutes for GitLab to be ready before starting runner)
- Circuit-breaker or fallback behavior: None

---

### 5) Observability for Integrations

- Logging around external calls: GitLab server logs written to `./gitlab/logs` volume (modules 03, 07). Not centralized.
- Metrics/tracing coverage: None
- Missing visibility gaps: No alerting, no metrics collection, no distributed tracing — appropriate for a local tutorial setup, but noted for completeness.

---

### 6) Evidence

- `07. auto-register-gitlab-runner-with-docker-executor/docker-compose.yml` (healthcheck + depends_on)
- `06. gitlab-runner-with-kubernetes-executor/config.toml` (cert auth to kind)
- `08. build-docker-images-using-kaniko/.gitlab-ci.yml` (Docker Hub push credentials)
- `10. scan-container-images-in-registry/.gitlab-ci.yml` (registry auth via CI vars)
- `.gitignore` (excludes `gitlab/` data from git)
