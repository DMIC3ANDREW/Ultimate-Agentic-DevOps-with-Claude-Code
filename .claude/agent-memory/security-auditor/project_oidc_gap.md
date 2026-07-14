---
name: project-oidc-gap
description: terraform/ has no IAM OIDC provider/role resources even though CLAUDE.md describes one; real CI role is unmanaged and hardcoded in the workflow file
metadata:
  type: project
---

As of 2026-07-14, `terraform/` contains only providers.tf, variables.tf, main.tf, outputs.tf, backend.tf —
there is no OIDC provider (`aws_iam_openid_connect_provider`) or IAM role/policy resource for GitHub Actions,
despite CLAUDE.md's architecture section describing "GitHub OIDC provider + IAM role for keyless CI/CD auth"
as part of this project's infrastructure.

The live role actually used is `arn:aws:iam::533267262133:role/github-actions-deploy`, referenced directly
(hardcoded) in `.github/workflows/deploy.yml` along with a hardcoded S3 bucket name
(`pravinmishradmi-site-production`) and CloudFront distribution ID (`E3V6O6MRE2E21P`).

**Why:** This means the OIDC trust policy (audience/subject scoping to repo+branch) and the role's IAM
permissions cannot be reviewed from code — they were provisioned out-of-band, which both violates the
project's stated convention ("All infrastructure changes go through Terraform — never modify AWS resources
manually") and makes it impossible to verify least-privilege / confused-deputy protections from the repo alone.

**How to apply:** On every future security audit of this project, re-check whether IAM/OIDC `.tf` files have
since been added to `terraform/`. If still absent, this remains a top-severity finding (unauditable trust
policy + hardcoded account ID/ARNs in `.github/workflows/deploy.yml`). If added, verify the trust policy's
`sub` condition is scoped to a specific `repo:<org>/<repo>:ref:refs/heads/main` (or environment) and that the
attached policy is scoped to only the specific bucket ARN and distribution ARN — then update this memory.
