# sample-nodejs — Production-Grade CI/CD on Kubernetes

> DevOps home assignment: a lightweight Node.js app wired into a production-grade pipeline — SAST, image scanning, ECR, ArgoCD GitOps, and EKS — built the way I'd actually do it on a real engagement.

---

## TL;DR

Fork a sample Node.js/Express app. Ship it with a fully automated CI/CD pipeline: parallel GitHub Actions jobs (SAST, lint, Trivy scan), keyless ECR push via OIDC, and GitOps delivery to EKS via ArgoCD. Every architectural decision is documented in an ADR. Every change goes through a PR. Every release is a semver git tag.

---

## My Approach: ADRs to Deployment and CI

I approached this mission decision-first: before writing pipeline YAML, Helm charts, or Terraform, I captured the key trade-offs as ADRs (ADR-001–ADR-013) so each implementation choice had explicit drivers, alternatives, and consequences. Starting from the app itself, I kept it as a stateless Express monolith (ADR-001) with in-process Prometheus metrics (ADR-002), which made Kubernetes `Deployment`, probes, and rolling updates straightforward.

For CI/CD, I chose GitHub Actions (ADR-003) with a single workflow that routes by branch and changed paths: a `detect` job classifies verify vs `v*` release runs and fans out only the relevant gates (SAST/SCA with `njsscan` + `npm audit`, Hadolint/Helm lint, Terraform fmt/validate/plan, ArgoCD manifest validation). On app changes I build locally, scan with Trivy (HIGH/CRITICAL, ignore-unfixed) before any registry push, and only on a release tag push the image to ECR using keyless AWS OIDC auth (ADR-004, ADR-006, ADR-007, ADR-011). Releases follow semver tags (ADR-010): after a successful build/scan/push, CI opens a PR that updates `helm/values.yaml` rather than pushing to `main` directly (ADR-008), keeping every deployment reviewable and auditable.

For deployment, I used same-repo GitOps with ArgoCD watching `helm/` (ADR-005) so a merged deploy PR triggers auto-sync to the cluster. The Helm chart carries the production concerns—probes, resource limits, security context, HPA/PDB options—and is kept environment-agnostic so the same manifests run on EKS for production and k3d for local development (ADR-009). I provisioned AWS foundations (ECR, GitHub OIDC/IAM, EKS-related resources) with Terraform (ADR-011) and mirrored the pipeline locally with a Taskfile (ADR-013) so `task ci` gives fast feedback before pushing. The rest of this README documents the resulting system and how to run it.

---

## What's Running

```
GET /my-app       → Hello, World!  (increments Prometheus counter)
GET /ready        → 200 OK         (readiness probe)
GET /live         → 200 OK         (liveness probe)
GET /metrics      → Prometheus scrape endpoint
```

Port: `8080` (override with `PORT` env var)

---

## Repo Map

```
refael/
├── app.js                      # Express app — single file, stateless
├── Dockerfile                  # Multi-stage: deps → hardened runtime (npm removed)
├── package.json
│
├── .github/workflows/
│   └── ci.yml                  # Full CI/CD pipeline (see Pipeline section)
│
├── helm/                       # Helm chart — Deployment, Service, Ingress, HPA, PDB
│   ├── Chart.yaml
│   ├── values.yaml             # Defaults; CI patches image.tag on release
│   └── templates/
│
├── argocd/
│   └── application.yaml        # ArgoCD Application CR — watches helm/, auto-syncs on merge
│
├── terraform/                  # AWS infra: ECR, GitHub OIDC provider, IAM role, EKS + node group
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
│
├── Taskfile.yml                # Local task runner (go-task) — mirrors CI steps exactly
│
└── .adlc/drafts/adr/           # Architecture Decision Records (ADR-001 → ADR-013)
```

---

## Pipeline

One router workflow: the `detect` job classifies the branch (verify vs `v*` release) and the changed paths, then every downstream job is gated on those outputs. There are no trigger-level `paths` filters — routing is decided inside the run.

| Trigger | Changed paths | Jobs |
|---|---|---|
| PR / push `main` | app.js, package*.json, Dockerfile | `sast` + `lint` + `build-and-scan` (parallel) |
| PR / push `main` | helm/ | `lint-helm` |
| PR / push `main` | terraform/ | `check-terraform` |
| PR / push `main` | argocd/ | `validate-argocd` |
| Push `v*` tag | app code since previous tag | `build-scan-push` → `update-image-tag` |

### Verify jobs (PR / main push, only for affected paths)

**SAST** (`sast` — app changes):
- `npm audit --audit-level=high` — SCA, blocks on any HIGH+
- `njsscan` — Node.js-specific SAST, blocks on any finding

**Lint** (`lint` — app changes):
- `hadolint` — Dockerfile best-practice linter

**Helm lint** (`lint-helm` — helm/ changes):
- `helm lint helm/ --strict`

**Terraform** (`check-terraform` — terraform/ changes):
- `terraform fmt -check` + `init -backend=false` + `validate` + `plan` (OIDC via `AWS_ROLE_TO_ASSUME`)

**ArgoCD** (`validate-argocd` — argocd/ changes):
- YAML parse + `apiVersion` / `kind` presence check

**Build & Scan** (`build-and-scan` — app changes):
- `docker build --load` — build image into local daemon
- Trivy scan: `severity: HIGH,CRITICAL`, `ignore-unfixed: true`, SARIF output to GitHub Security tab

### Release Chain (sequential, tag-triggered)

```
build-scan-push
  └─ docker build → trivy (gate) → ECR push (OIDC, no long-lived keys)
       └─ update-image-tag
            └─ yq patch helm/values.yaml → open PR → ArgoCD auto-syncs on merge
```

No deploy reaches the cluster without a merged PR. That's intentional — human oversight on every release.

---

## GitOps Flow

```
git tag v1.2.0 && git push origin v1.2.0
        │
        ▼
GitHub Actions (build-scan-push)
  build image → trivy scan → push to ECR as :v1.2.0
        │
        ▼
GitHub Actions (update-image-tag)
  yq: helm/values.yaml image.tag = "v1.2.0"
  git commit → open PR "chore: deploy image v1.2.0"
        │
        ▼
Merge PR → ArgoCD detects helm/ change → auto-sync → EKS
```

ArgoCD Application (`argocd/application.yaml`) watches `helm/` on `main`, prunes removed resources, self-heals on drift, retries up to 3×.

For the full branch/PR rules, SemVer tag policy, and the reviewer checklist, see [Git Workflow & Version Bumping](docs/GIT_WORKFLOW.md).

---

## Release Run Evidence — v1.0.0 (2026-11-19)

End-to-end evidence for issue [#31](https://github.com/OrenOren1/refael/issues/31), tag [`v1.0.0`](https://github.com/OrenOren1/refael/tree/v1.0.0) on commit `d657650`.

| Stage | Evidence |
|---|---|
| Release workflow | [run 44535808394](https://github.com/OrenOren1/refael/actions/runs/44535808394) — success: `detect` → `build-scan-push` → `update-image-tag` |
| Trivy gate | `Trivy scan — gate on HIGH+ (ignore unfixed)` passed inside `build-scan-push` **before** any registry login/push; SARIF uploaded as artifact `trivy-sarif` |
| ECR push | `294747659026.dkr.ecr.us-east-1.amazonaws.com/sample-nodejs:v1.0.0`, digest `sha256:46a9b3f1e2f2f5ffc80e367917288f5c91f3b783bb684d43e44d1c22002bf7d0` (OIDC, keyless) |
| Deploy PR | [PR #34](https://github.com/OrenOren1/refael/pull/34) — patches `helm/values.yaml` to the ECR repository and `image.tag: v1.0.0`; merged as `8e99443` |
| Post-merge CI | [run 44536558495](https://github.com/OrenOren1/refael/actions/runs/44536558495) — success (`lint-helm` on the changed chart) |

**Cluster outcome (documented blocker):** no ArgoCD sync/deployment update was observable. Both EKS node groups (`refael-nodes`, `refael-big`) were at `desired=0`, so the ArgoCD controllers were `Pending` and `kubectl get nodes` returned none. The live `Application` also retained manual drift from the earlier test session — `helm.parameters.image.tag=1.0.1` and automated sync suspended — versus the repo manifest, which has no parameter overrides and enables `automated`. Restoring nodes and re-applying `argocd/application.yaml` is required before cluster-side sync can be verified.

**Run notes:** the first two `update-image-tag` attempts failed — first because the repository disallowed PR creation by `GITHUB_TOKEN`, then because the leftover `deploy/image-v1.0.0` branch from the failed attempt rejected a non-fast-forward push. Enabling `can_approve_pull_request_reviews`, deleting the stale branch, and re-running failed jobs produced the green run above. [PR #34's own check run](https://github.com/OrenOren1/refael/actions/runs/44536353828) has zero jobs because events triggered by `GITHUB_TOKEN` do not start new workflows; the merge commit's `main` run is the authoritative verification.

---

## Infrastructure

Provisioned by Terraform (`terraform/`), applied once before first pipeline run (`terraform/main.tf`, `variables.tf`):

| Resource | Details |
|---|---|
| ECR repository | `sample-nodejs`, scan-on-push, lifecycle: keep last 10 |
| GitHub OIDC provider | Keyless auth — no `AWS_SECRET_ACCESS_KEY` in secrets |
| IAM roles/policies | `github-actions-refael` — ECR push + Terraform plan-read, scoped to this repo (`repo:OrenOren1/refael:*`); EKS cluster/node roles with managed policy attachments |
| EKS cluster | `refael`, Kubernetes 1.31, `us-east-1` default-VPC subnets filtered to AZs a/b/c/d/f |
| EKS node group | `refael-nodes`, `t3.small` (default), desired 2 / min 1 / max 3, node role has `AmazonEC2ContainerRegistryReadOnly` |

**Scope note:** current Terraform provisions ECR, GitHub OIDC/IAM, and EKS (cluster + node group). It does **not** provision ACM certificates or Route53 records; ADR-011 has been aligned to this actual scope. Ingress differs by environment: `helm/values.yaml` is the k3d/local default (`ingress.className: nginx`, HTTP, no cert), while `helm/values-eks.yaml` is the EKS profile (AWS Load Balancer Controller `alb` class, ACM cert via `alb.ingress.kubernetes.io/certificate-arn`) and is loaded by `argocd/application.yaml`. Both the AWS Load Balancer Controller and a validated ACM certificate are prerequisites; the certificate ARN must be supplied via `ingress.tls.certArn` (ArgoCD parameter or Helm `--set-string`). The certificate itself is not created by this Terraform and was not part of the recorded demo path.

```bash
# One-time setup
task tf:init
task tf:apply              # ~12 min — EKS cluster takes the longest
task aws:kubeconfig        # configure kubectl

# Register app with ArgoCD
task argocd:install        # install ArgoCD to cluster
task argocd:apply          # apply argocd/application.yaml
```

After `tf:apply`, grab the ECR URI and IAM role ARN:
```bash
terraform -chdir=terraform output ecr_repository_url
terraform -chdir=terraform output github_actions_role_arn
```

Set as GitHub Actions secrets:
- `AWS_ROLE_TO_ASSUME` → the IAM role ARN
- `ECR_REGISTRY` → the ECR registry URL (account.dkr.ecr.us-east-1.amazonaws.com)

---

## Helm Chart

Kubernetes objects deployed:

| Object | Notes |
|---|---|
| `Deployment` | 2 replicas, stateless, rolling update |
| `Service` | ClusterIP, port 80 → container 8080 |
| `Ingress` | k3d/nginx by default; EKS uses `alb` class with ACM cert ARN (`values-eks.yaml`) |
| `ConfigMap` | `PORT` env var injected |
| `HorizontalPodAutoscaler` | optional, CPU-based |
| `PodDisruptionBudget` | `minAvailable: 1` |

Probes configured on `/ready` (readiness) and `/live` (liveness).

```bash
# Local dev with k3d
task k3d:up
task deploy:local          # builds + imports image into k3d, deploys via helm
```

---

## Local Dev

Install `go-task` once: `brew install go-task`

```bash
task lint          # hadolint + helm lint
task security      # npm audit + nodejsscan
task build         # docker build local
task scan          # trivy scan local image (gate: HIGH+)
task ci            # full chain: lint → security → build → scan → deploy:local
```

---

## Architecture Decision Records

Every non-obvious choice is documented. Here's the full set:

| ADR | Decision | Why |
|---|---|---|
| [ADR-001](/.adlc/drafts/adr/ADR-001.md) | Stateless Express monolith → `Deployment` | No shared state, no DB, no domain boundaries — `StatefulSet` would be wrong |
| [ADR-002](/.adlc/drafts/adr/ADR-002.md) | In-process `prom-client` for metrics | Simplest path to Prometheus scraping, matches existing code |
| [ADR-003](/.adlc/drafts/adr/ADR-003.md) | GitHub Actions as CI/CD engine | No external CI infra, AWS OIDC support, first-class Trivy/Semgrep actions |
| [ADR-004](/.adlc/drafts/adr/ADR-004.md) | AWS ECR as container registry | Keyless OIDC auth from GHA, IAM-native access control, same AWS account |
| [ADR-005](/.adlc/drafts/adr/ADR-005.md) | Same-repo GitOps (ArgoCD watches `helm/`) | Single-app solo project — separate gitops repo adds ops overhead for zero gain |
| [ADR-006](/.adlc/drafts/adr/ADR-006.md) | SAST: `njsscan` + `npm audit` | njsscan is Node.js-specific OWASP-listed; npm audit covers dependency CVEs separately |
| [ADR-007](/.adlc/drafts/adr/ADR-007.md) | Trivy: HIGH+, ignore-unfixed | Pragmatic gate — block what can actually be fixed today |
| [ADR-008](/.adlc/drafts/adr/ADR-008.md) | CI opens deploy PR (not direct push) | Every deploy is a reviewable git event; bad image = no PR, cluster stays on last good |
| [ADR-009](/.adlc/drafts/adr/ADR-009.md) | EKS (prod) + k3d (local) | Originally nginx on both for parity; EKS now uses ALB ingress to terminate ACM TLS (see `values-eks.yaml`) |
| [ADR-010](/.adlc/drafts/adr/ADR-010.md) | Semver git tags trigger release pipeline | Human-readable, sortable; decouples release from every commit push |
| [ADR-011](/.adlc/drafts/adr/ADR-011.md) | Terraform for AWS infra | Reproducible, reviewable, idempotent — no click-ops |
| [ADR-012](/.adlc/drafts/adr/ADR-012.md) | Hadolint + helm lint in CI | Catch Dockerfile and chart issues before they reach image build |
| [ADR-013](/.adlc/drafts/adr/ADR-013.md) | Taskfile as local task runner | Single YAML, Go binary, mirrors CI step order exactly |
| [ADR-014](docs/adlc/memory/adr/ADR-014.md) | CI routing by changed paths and branch | One `detect` job gates app/helm/terraform/argocd/release work |
| [ADR-015](docs/adlc/memory/adr/ADR-015.md) | Terraform plan in PR gate via OIDC read policy | Shows the infra diff without long-lived AWS keys |

---

## Security Decisions

A few things I'd flag for a reviewer:

**SAST / SCA gate behavior (requirement mapping):** the assignment asks to fail if critical vulnerabilities are found in code. Actual gates (`.github/workflows/ci.yml`):
- **`njsscan` (source-code SAST)** — runs in `sast` on app changes and fails the job on any finding it reports (default action behavior, no severity-only filter), so reported code issues block.
- **`npm audit --audit-level=high` (dependency SCA, not source SAST)** — fails on HIGH and CRITICAL dependency CVEs in the dependency tree.
- **Trivy image scan** — runs before push in `build-and-scan`/`build-scan-push` with `exit-code: "1"`, `severity: "HIGH,CRITICAL"`, `ignore-unfixed: "true"` (SARIF uploaded), so fixable HIGH/CRITICAL image CVEs block and unfixed ones are logged but not blocking.

Together these satisfy the intent of blocking critical risk and are stricter in places (njsscan fails on any finding; npm audit/Trivy gate HIGH+). They differ from a literal "critical-only, source-code-only" gate by covering dependency/image layers and by blocking fixable HIGH findings as well.

**No long-lived AWS credentials** — GitHub Actions authenticates via OIDC (`sts:AssumeRoleWithWebIdentity`). No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` anywhere. The IAM role is scoped to this exact repo (`repo:OrenOren1/refael:*`).

**npm removed from runtime image** — `node:22-alpine` bundles npm, which bundles `glob@13` and `minimatch` with known CVEs. npm is not needed to run `node app.js`. Removing it entirely (`rm -rf /usr/local/lib/node_modules/npm`) eliminates the CVE surface without suppression. `apk upgrade --no-cache` patches OS-level packages at build time.

**Pipeline fails on fixable CVEs** — `ignore-unfixed: true` means Trivy only blocks on CVEs that have a fix available. We're not blocking on theoretical risk — we're blocking on "you can fix this today."

**Least-privilege CI permissions** — `permissions: contents: read` global default. Only jobs that need elevation declare it (`build-scan-push`: `id-token: write`; `update-image-tag`: `contents: write`, `pull-requests: write`; `check-terraform`: `id-token: write` for PR plan). PR jobs never get write access to repo contents.

---

## Submission Clarifications

- **SAST / severity vs "critical vulnerabilities in code":** see [Security Decisions](#security-decisions) for the exact gate mapping. In short: `njsscan` blocks on any reported source finding; `npm audit --audit-level=high` (SCA) blocks on HIGH/CRITICAL deps; Trivy blocks fixable HIGH/CRITICAL image CVEs before push — satisfying the critical-risk intent while differing from a literal critical-only/code-only gate.
- **Runtime/demo config (v1.0.0):** image `294747659026.dkr.ecr.us-east-1.amazonaws.com/sample-nodejs:v1.0.0` (deploy PR [#34](https://github.com/OrenOren1/refael/pull/34) merged, `helm/values.yaml`). Ingress values: `enabled: true`, `className: nginx`, `host: app.example.com` (placeholder, override per environment), `tls.enabled: false`, `certArn: ""`, path `/` → Service `ClusterIP` port `80` → container `8080` (`/ready`, `/live`, `/my-app`). No public demo URL was recorded because cluster-side ArgoCD sync could not be shown in [Release Run Evidence — v1.0.0](#release-run-evidence--v100-2026-11-19) (EKS nodes at desired 0, live app drift); local demo: `task deploy:local` (k3d) then `curl http://localhost:8080/my-app`.
- **Infra scope / HTTPS path:** Terraform provisions ECR, GitHub OIDC + IAM, and EKS only (no ACM/Route53). k3d/local runs nginx ingress over HTTP; EKS loads `helm/values-eks.yaml` via ArgoCD and uses ALB ingress with an ACM certificate ARN (`ingress.tls.certArn` → `alb.ingress.kubernetes.io/certificate-arn`), requiring the AWS Load Balancer Controller and a pre-existing validated cert. See [Infrastructure](#infrastructure) and [ADR-011](docs/adlc/memory/adr/ADR-011.md).
- **Deployment rationale:** `Deployment` chosen because the app is stateless with no persistent/node-local state, so rolling updates, probes, and scaling fit (ADR-001). Same-repo GitOps: ArgoCD watches `helm/` in this repo (`argocd/application.yaml`, ADR-005) for single-app simplicity and one review stream; CI opens image-tag deploy PRs instead of pushing to `main` (ADR-008).

## For Reviewers

What to look at:

1. **[`Dockerfile`](./Dockerfile)** — multi-stage, hardened runtime. npm removed, not suppressed.
2. **[`.github/workflows/ci.yml`](./.github/workflows/ci.yml)** — DAG structure, concurrency groups, timeout-minutes, permission scoping.
3. **[`helm/`](./helm/)** — values.yaml has resource limits, PDB, probes. Chart is environment-agnostic.
4. **[`argocd/application.yaml`](./argocd/application.yaml)** — auto-sync, prune, self-heal, retry policy.
5. **[`terraform/main.tf`](./terraform/main.tf)** — ECR + OIDC + EKS in one file. Subnets filtered for EKS-compatible AZs.
6. **[`Taskfile.yml`](./Taskfile.yml)** — local CI parity. `task ci` runs the same chain as the pipeline.
7. **[`docs/GIT_WORKFLOW.md`](docs/GIT_WORKFLOW.md)** — branch/PR flow, SemVer tag policy, and how `v*` tags map to ECR push, deploy PR, and ArgoCD sync.

Things intentionally left out:
- **No `.trivyignore`** — CVEs are fixed at the root, not suppressed
- **No direct push to `main` from CI** — every deploy is a PR
- **No `--no-verify` anywhere** — hooks run, always

---

## Quick Start

```bash
# 1. Clone and install
git clone https://github.com/OrenOren1/refael && cd refael
npm install

# 2. Run locally
node app.js
curl http://localhost:8080/my-app

# 3. Local K8s (needs go-task + docker + k3d)
task k3d:up
task deploy:local
curl http://localhost:8080/my-app

# 4. Trigger a release
git tag v1.0.0 && git push origin v1.0.0
# → pipeline builds, scans, pushes to ECR, opens deploy PR
```

---

## Tech Stack

`Node.js 22` · `Express` · `prom-client` · `Docker (Alpine)` · `Helm 3` · `ArgoCD` · `GitHub Actions` · `AWS ECR` · `AWS EKS` · `Terraform` · `Trivy` · `njsscan` · `Hadolint` · `go-task`
