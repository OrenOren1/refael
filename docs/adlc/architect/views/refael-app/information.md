# Information View — Refael App

> ADRs: ADR-001, ADR-002, ADR-005, ADR-008, ADR-010
> Date: 2026-11-19
> Dependencies: context.md, functional.md

## Data Entities

| Entity | Type | Owner | Persistence | ADR |
|--------|------|-------|------------|-----|
| HTTP Request/Response | Transient | Express | None (stateless) | ADR-001 |
| Prometheus Metrics | In-memory | prom-client registry | Process lifetime only | ADR-002 |
| Docker Image | Artifact | AWS ECR | Immutable, tagged by semver | ADR-004, ADR-010 |
| `helm/values.yaml` | Config file | Git repo | Git history | ADR-005, ADR-008 |
| Git Tag (`v1.x.x`) | SCM ref | GitHub repo | Permanent | ADR-010 |
| Terraform State | State file | Local / S3 | Persistent (operator-managed) | ADR-011 |
| CI Workflow Run | Log | GitHub Actions | 90-day GitHub retention | ADR-003 |

## Configuration Data Flow

```mermaid
flowchart TD
    Tag["git tag v1.2.3"]
    CI["GitHub Actions\n(CI Pipeline)"]
    ECR["AWS ECR\nimage: app:v1.2.3"]
    ValuesYaml["helm/values.yaml\nimage.tag: v1.2.3"]
    PR["Deploy PR"]
    Argo["ArgoCD\n(reads helm/)"]
    Pod["Running Pod\nimage: app:v1.2.3"]
    Metrics["Prometheus Metrics\n(in-process, ephemeral)"]

    Tag -->|"triggers"| CI
    CI -->|"push image"| ECR
    CI -->|"yq patch"| ValuesYaml
    ValuesYaml --> PR
    PR -->|"merge"| Argo
    Argo -->|"helm upgrade"| Pod
    ECR -->|"imagePullPolicy: IfNotPresent"| Pod
    Pod -->|"collectDefaultMetrics()"| Metrics
```

## Image Versioning Model

```mermaid
erDiagram
    GIT_TAG {
        string ref "v1.2.3"
        string sha "abc123"
        string pusher "developer"
    }
    ECR_IMAGE {
        string repository "sample-nodejs"
        string tag "v1.2.3"
        string digest "sha256:..."
        bool immutable "true"
    }
    HELM_VALUES {
        string image_repository "ECR URI"
        string image_tag "v1.2.3"
        int replicas "2"
        string ingress_host "app.example.com"
    }
    ARGOCD_APP {
        string path "helm/"
        string targetRevision "HEAD"
        string syncPolicy "automated"
    }

    GIT_TAG ||--|| ECR_IMAGE : "triggers build of"
    ECR_IMAGE ||--|| HELM_VALUES : "referenced by"
    HELM_VALUES ||--|| ARGOCD_APP : "watched by"
```

## Data Quality & Immutability

- ECR repo configured with **image tag immutability** — once `v1.2.3` is pushed, it cannot be overwritten
- `helm/values.yaml` changes tracked in Git — full audit trail of every deploy
- No application-level database — zero schema migration risk
- Metrics are ephemeral: pod restart resets all counters (by design for Prometheus pull model)

## Security Perspective (Information Level)

- No PII or secrets stored in `values.yaml` — Kubernetes Secrets used for sensitive config
- Terraform state may contain sensitive ARNs — store in S3 with encryption + versioning for production
- ECR image scanning (ADR-007) produces vulnerability reports — stored as CI artifacts
