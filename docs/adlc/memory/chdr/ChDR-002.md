---
type: ChDR
title: ACM certificate + Route53 DNS validation — external Terraform prerequisites
description: Documents that ACM certificate provisioning and Route53 DNS validation records are out-of-Terraform scope; they are manual prerequisites for EKS ALB Ingress (ADR-016).
resource: ./docs/adlc/memory/chdr/ChDR-002.md
tags: [chdr, ingress, acm, route53, terraform]
generated:
  by: agent:architect-implement
  at: 2026-11-19T00:00:00Z
id: ChDR-002
created: 2026-11-19
verified:
  - by: agent:architect-implement
    at: 2026-11-19T00:00:00Z
status: stable
stale_after: 365d
---

# ChDR-002: ACM Certificate + Route53 DNS Validation — External Prerequisites

## Change Summary

ADR-016 adopts the AWS ALB Ingress class for EKS production traffic, requiring
an ACM certificate ARN to be supplied at deploy time via `ingress.tls.certArn`.
This record documents that neither the ACM certificate lifecycle nor the
Route53 DNS validation records are managed by the project's Terraform module.

## What Changed

| Component | Before (ADR-009 / original AD.md) | After (ADR-016) |
|-----------|----------------------------------|-----------------|
| EKS Ingress class | `nginx` (nginx-ingress controller) | `alb` (AWS Load Balancer Controller) |
| TLS termination | nginx controller (chart used wrong annotation) | ALB via ACM certificate |
| ACM cert in Terraform | Listed as `aws_acm_certificate.ingress` | **Removed** — external prerequisite |
| Route53 record in Terraform | Listed as `aws_route53_record.cert_validation` | **Removed** — external prerequisite |
| k3d/local Ingress class | `nginx` | `nginx` (unchanged) |

## Why Out of Terraform

* The `assignment-tests` AWS account is Free Tier restricted; ACM certificates
  require a validated domain that the assignment does not provide.
* Route53 hosted zones imply real domain ownership outside assignment scope.
* Keeping these out of Terraform avoids `terraform destroy` deleting a
  potentially reused certificate.
* ADR-011 explicitly scopes Terraform to ECR + OIDC/IAM + EKS only.

## Prerequisite Steps (manual, before first EKS deploy)

1. Request or import an ACM certificate in `us-east-1` for the target domain.
2. Complete DNS validation via Route53 (or add the CNAME manually).
3. Wait for certificate status `ISSUED`.
4. Install AWS Load Balancer Controller in the EKS cluster:
   ```bash
   helm repo add eks https://aws.github.io/eks-charts
   helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
     -n kube-system \
     --set clusterName=refael \
     --set serviceAccount.create=false \
     --set serviceAccount.name=aws-load-balancer-controller
   ```
5. Supply the ARN to ArgoCD (or via `helm --set-string`):
   ```bash
   argocd app set sample-nodejs \
     --helm-set-string ingress.tls.certArn=arn:aws:acm:us-east-1:...
   ```

## Impact on AD.md

* §2 Architectural Goals: "Local dev must mirror prod (nginx-ingress on both)"
  updated to reflect reduced parity (ADR-016 intentionally diverges).
* §3.5 Deployment View: Production diagram updated — ALB replaces nginx-ingress;
  Terraform table removes ACM and Route53 resources.
* §5 ADR Summary: ADR-016 added.

## Related

* [ADR-016: EKS ALB Ingress with ACM Certificate; nginx for k3d](../adr/ADR-016.md)
* [ADR-009: Kubernetes: EKS (prod) + k3d (local dev)](../adr/ADR-009.md)
* [ADR-011: Terraform for AWS Infrastructure Provisioning](../adr/ADR-011.md)
