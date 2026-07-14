---
name: project-missing-gitignore
description: repo has no .gitignore anywhere, and backend.tf's bootstrap flow requires a local tfstate phase before S3 backend migration
metadata:
  type: project
---

As of 2026-07-14, there is no `.gitignore` file anywhere in this repository (checked repo root and
recursively). `terraform/backend.tf` documents an intentional bootstrap sequence: run `terraform apply` with
local state first, then migrate to the S3+DynamoDB backend once it exists. At the time of this check no
`terraform/*.tfstate` file existed yet, but `terraform/.terraform/` (provider cache) and
`terraform/.terraform.lock.hcl` were already present and untracked.

**Why:** Without a `.gitignore`, the local `terraform.tfstate` produced during the bootstrap phase (which
contains the AWS account ID, S3/CloudFront ARNs, and any other resource metadata) is one `/commit` or `git
add .` away from being committed to version control history — hard to fully purge afterward.

**How to apply:** On future audits/scaffolds of this project, check whether a `.gitignore` has been added
covering `terraform/.terraform/`, `terraform/*.tfstate*`, and `*.tfvars`. If still missing, keep flagging it
(severity HIGH/MEDIUM depending on whether local state currently exists). Also re-verify whether
`terraform/` has since been committed and, if so, `git log` for that commit to confirm no `.tfstate` file was
swept in.
