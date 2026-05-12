# Codebase Structure

## Core Sections (Required)

### 1) Top-Level Map

| Path | Purpose | Evidence |
|------|---------|----------|
| `01. gitlab-in-docker/` | Module 1 — Run GitLab CE via plain `docker run` commands | `01.*/README.md` |
| `02. gitlab-in-docker-compose/` | Module 2 — GitLab CE via Docker Compose with volume persistence | `02.*/docker-compose.yaml`, `02.*/README.md` |
| `03. gitlab-runner-with-shell-executor/` | Module 3 — GitLab Runner using shell executor | `03.*/docker-compose.yml`, `03.*/config.toml` |
| `04. gitlab-runner-with-docker-executor-socket-binding/` | Module 4 — Runner with Docker executor via socket binding | `04.*/docker-compose.yml`, `04.*/Dockerfile` |
| `05. gitlab-runner-with-docker-executor-dind/` | Module 5 — Runner with Docker-in-Docker (privileged) | `05.*/docker-compose.yml`, `05.*/config.toml` |
| `06. gitlab-runner-with-kubernetes-executor/` | Module 6 — Runner with Kubernetes executor (kind cluster) | `06.*/config.toml`, `06.*/kind-cluster-config.yaml` |
| `07. auto-register-gitlab-runner-with-docker-executor/` | Module 7 — Auto-register runner via Docker Compose command | `07.*/docker-compose.yml` |
| `08. build-docker-images-using-kaniko/` | Module 8 — Build Docker images in CI using Kaniko (rootless) | `08.*/.gitlab-ci.yml`, `08.*/src/` |
| `09. setup-container-registry/` | Module 9 — Enable GitLab managed Container Registry | `09.*/docker-compose.yml`, `09.*/.gitlab-ci.yml` |
| `10. scan-container-images-in-registry/` | Module 10 — Container image scanning with Trivy via GitLab CI | `10.*/.gitlab-ci.yml` |
| `11. scan-dependencies-in-gitlab-ci/` | Module 11 — Dependency scanning via GitLab CI template | `11.*/.gitlab-ci.yml`, `11.*/requirements.txt` |
| `docs/codebase/` | Codebase knowledge documents (this folder) | — |
| `README.md` | Top-level repo overview and topic index | `README.md` |
| `.gitignore` | Excludes `gitlab/` data dirs and `*.pem` files | `.gitignore` |

---

### 2) Entry Points

- Main runtime entry: **None** — this is a reference/tutorial repo, not a runnable application
- Secondary entry points: Each module is a self-contained unit. Start from its `docker-compose.yml` with `docker compose up`
- How entry is selected: User navigates to the relevant numbered module folder and runs Docker Compose

---

### 3) Module Boundaries

| Boundary | What belongs here | What must not be here |
|----------|-------------------|------------------------|
| Each numbered module folder | Docker Compose file, runner `config.toml`, sample `Dockerfile`, sample `.gitlab-ci.yml`, `README.md` | Application source code, cross-module shared logic |
| `08.*/src/` | Sample Dockerfiles to be built by Kaniko | CI pipeline logic |
| `docs/codebase/` | Codebase documentation | Config files, CI scripts |

---

### 4) Naming and Organization Rules

- Directory naming: `NN. descriptive-kebab-case` (two-digit prefix + space + description), e.g. `03. gitlab-runner-with-shell-executor`
- File naming: lowercase kebab-case for config files (`docker-compose.yml`, `config.toml`); GitLab CI files use `.gitlab-ci.yml` (modules 8–11) or `gitlab-ci.yml` (modules 3–5, inconsistently named — missing the leading dot)
- Organization pattern: **feature/topic-based** — each module is a standalone lesson
- No path aliases, no source imports

---

### 5) Evidence

- Repo root directory listing (scan output)
- `03. gitlab-runner-with-shell-executor/gitlab-ci.yml` (no leading dot)
- `08. build-docker-images-using-kaniko/.gitlab-ci.yml` (with leading dot)
- `.gitignore` (excludes `gitlab/` and `*.pem`)
