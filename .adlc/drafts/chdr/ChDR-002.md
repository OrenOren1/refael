## ChDR-002: ACM certificate and Route53 validation added, then removed in the EKS Terraform rewrite

### Status: **Discovered**

### Date: 2026-11-19

### Source: Git history via /change-init (targeted scan of `terraform/`)

### Issue Links: none detected (merge commit references PR #16 only)

### Commits: afb8cf4, 2ee2093, 9601cb1

### Target Module: `docs/adlc/memory/chdr/acm-cert-added-then-removed.md`

### Descriptor: Why ACM/route53 TLS resources existed briefly in Terraform and were removed when EKS was added; current TLS binding is Helm-side only.

### Context

The first Terraform pass provisioned an ACM certificate for Ingress TLS.
Commit `afb8cf4` added `aws_acm_certificate`, `aws_route53_record` for DNS
validation, `aws_acm_certificate_validation`, the `domain_name` and
`route53_zone_id` variables, and an `acm_certificate_arn` output described as
"Reference in Helm values or Ingress annotation". It was merged to `main` as
PR #16 by `2ee2093` about two minutes later.

Roughly six minutes after the original commit, `9601cb1` rewrote
`terraform/main.tf` to add the EKS cluster and node group. The same diff
deleted all ACM certificate, Route53 validation, and related variables and
outputs. No revert commit exists, and neither commit message cites an issue.
Later docs-only work (PR #37) aligned `ADR-011` and `README.md` to the actual
scope: ECR, GitHub OIDC/IAM, and EKS only, with no ACM/Route53 resources.

### Decision

* With **HIGH** confidence, `afb8cf4` decided to provision the TLS
  certificate in Terraform: the commit message names
  "aws_acm_certificate + aws_route53_record for DNS validation" and the diff
  adds those resources plus the `acm_certificate_arn` output.
* With **HIGH** confidence, `9601cb1` decided to drop that infrastructure and
  make Terraform the ECR/OIDC/IAM/EKS foundation: its diff removes every ACM
  and Route53 resource and its commit message lists only the EKS-era
  resources.
* **Reason for removal is unknown.** No linked issue, no revert, and no
  explanatory text in `9601cb1`. The evidence shows the ACM work was
  superseded by the EKS scope, not why that trade-off was chosen. Inference
  that it was simply "out of scope" is not treated as a documented decision.

### Consequences

* Negative knowledge: an ACM certificate and its DNS validation were added
  and removed within the same session; re-adding them later needs the
  original `domain_name` / `route53_zone_id` inputs reconstructed.
* The `acm_certificate_arn` output disappeared, so Helm had no Terraform
  output to consume. `helm/values.yaml` retained a `certArn` field and
  `helm/templates/ingress.yaml` retained an AWS load-balancer annotation,
  but the annotation key did not match the values comment and nginx-ingress
  would not honor it.
* `ADR-011` and `README.md` initially disagreed with the code; PR #37 had to
  document "no ACM/Route53" explicitly to close that gap.
* Current state: EKS TLS is a Helm concern (ALB ingress class plus an
  externally supplied ACM ARN, captured in draft ADR-016), not a Terraform
  one.
* Revert/fix chains observed: none. `9601cb1` is a scope rewrite, not a git
  revert.

### Evidence

- `afb8cf4`: "feat(terraform): provision ECR, GitHub OIDC, IAM role, ACM cert"
  — added `aws_acm_certificate`, `aws_route53_record.cert_validation`,
  `aws_acm_certificate_validation`, and the `acm_certificate_arn` output.
- `2ee2093`: "Merge pull request #16 from
  OrenOren1/feat/terraform-infra" — merged the ACM commit to `main`.
- `9601cb1`: "feat(aws): add Terraform for ECR, OIDC IAM, EKS + env tasks in
  Taskfile" — deleted the ACM/Route53 resources while adding EKS resources.
- `terraform/main.tf`: present-day file contains ECR, OIDC/IAM, and EKS only;
  no `aws_acm_certificate` or Route53 records.
- `docs/adlc/memory/adr/ADR-011.md:33`: "ACM certificates and Route53
  records/DNS validation are not provisioned"; earlier draft mentioned them.
- `https://github.com/OrenOren1/refael/pull/16`: PR link from merge commit
  message (link-only; body not fetched).
