---
type: ChDR
title: Merged CI routing PR with red Terraform OIDC check
description: Temporary workaround for AWS OIDC trust failure blocking the Terraform plan gate
resource: ./docs/adlc/memory/chdr/ChDR-001.md
tags: [chdr]
generated:
  by: agent:change-publish
  at: 2026-11-19T00:00:00Z
id: ChDR-001
created: 2026-11-19
verified:
  - by: agent:change-publish
    at: 2026-11-19T00:00:00Z
status: stable
stale_after: 365d
sources:
  - id: 2d1b163
    resource: git:2d1b163
    title: feat(ci): route jobs by changed paths and branch, plan terraform (#26)
  - id: d7f5b94
    resource: git:d7f5b94
    title: Merge pull request #27 from OrenOren1/fix/ci-add-workflow-dispatch
  - id: '#26'
    resource: https://github.com/OrenOren1/refael/issues/26
    title: CI: route checks by changed paths and branch, plan Terraform on tf changes
---

# ChDR-001: Merged CI routing PR with red Terraform OIDC check

### Status: **Accepted**

### Date: 2026-11-19

### Issue Links: https://github.com/OrenOren1/refael/issues/26

### Commits: 2d1b163, d7f5b94

### Context
PR [#27](https://github.com/OrenOren1/refael/pull/27) (commit `2d1b163`, merge `d7f5b94`) routed CI jobs by branch and changed paths and added `terraform plan` to the Terraform PR gate using OIDC via `AWS_ROLE_TO_ASSUME`. Issue [#26](https://github.com/OrenOren1/refael/issues/26) defined the scope.

The repo had no `AWS_ROLE_TO_ASSUME` secret. It was populated from the local Terraform state output `github_actions_role_arn` (`arn:aws:iam::294747659026:role/github-actions-refael`) and workflow run [44508278988](https://github.com/OrenOren1/refael/actions/runs/44508278988) still failed with `Not authorized to perform sts:AssumeRoleWithWebIdentity`. No AWS credentials were available locally to inspect the live IAM trust policy, and the new `plan-read` inline policy exists only in Terraform code, not yet applied.

### Decision (confidence: HIGH)
Merge PR [#27](https://github.com/OrenOren1/refael/pull/27) despite the failing Terraform plan check, as chosen by the user, leaving the AWS-side OIDC fix as a follow-up. The routing and release-gate changes in merge `d7f5b94` land independently of the plan job's credentials.

### Consequences
- Issue [#26](https://github.com/OrenOren1/refael/issues/26) is closed with a known follow-up: fix OIDC trust and apply the `plan-read` policy.
- Terraform plan coverage is not yet verified in CI; local checks in commit `2d1b163` passed fmt/validate only.

### Evidence
- Issue: https://github.com/OrenOren1/refael/issues/26
- PR: https://github.com/OrenOren1/refael/pull/27
- Commits: `2d1b163` (feature), `d7f5b94` (merge)
- Failed run: https://github.com/OrenOren1/refael/actions/runs/44508278988

### Revisit When
GitHub Actions can assume `github-actions-refael` and `terraform plan` passes on a PR that changes Terraform files; then remove the known-failure caveat from issue [#26](https://github.com/OrenOren1/refael/issues/26).

## Source

Promoted from: .adlc/drafts/chdr/ChDR-001.md
