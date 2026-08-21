# PR Review Checklist

Use this checklist on every Terraform PR — both as the author (before requesting review) and as the reviewer (before approving).

---

## For the author: pre-review checklist

Complete all items before marking the PR ready for review.

### Code quality

- [ ] `terraform fmt -recursive` has been run — no formatting diff.
- [ ] `terraform validate` passes locally in all changed roots.
- [ ] `tflint --recursive` produces no errors.
- [ ] `trivy config .` produces no HIGH or CRITICAL findings (or all findings are in `.trivyignore` with documented reasons).
- [ ] `checkov -d .` produces no failures (or all skipped checks are in `.checkov.yaml` with documented reasons).
- [ ] `terraform-docs .` was run and README files are up to date in changed modules.

### State and locking

- [ ] `.terraform.lock.hcl` is committed for any root module that was newly created or that had `terraform init` re-run.
- [ ] `.terraform.lock.hcl` was generated with both platforms: `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64`.
- [ ] No `.terraform/` directories are committed.

### Imports

- [ ] If this PR contains `import {}` blocks: the import plan shows no destroy operations.
- [ ] A follow-up PR to remove the `import {}` blocks is planned (link to it in the description).

### Secrets

- [ ] No `*.secret.tfvars` or `*.auto.tfvars` files added.
- [ ] No secret values in `outputs.tf` (only ARNs, names, IDs).
- [ ] No secret values in inline HCL strings.
- [ ] No sensitive data in PR description or comments.

### Conventions

- [ ] All new AWS resource names follow `<org>-<env>-<service>[-<detail>]` (hyphens, lowercase).
- [ ] All new Terraform resource identifiers are snake_case with no environment suffix.
- [ ] `locals.tf` defines `default_tags` and the provider uses `default_tags`.
- [ ] New resources have `Name` and `Service` tags at minimum.

### Scope

- [ ] The PR changes only the layers and environments that the description claims.
- [ ] The PR does not mix infrastructure changes with module library changes — those go in separate PRs.
- [ ] Production changes are in a separate PR from staging changes, unless staging and production use exactly the same code (identical DRY pattern).

---

## For the reviewer: review checklist

### Atlantis plan output

Review the Atlantis plan comment carefully:

- [ ] The plan output is present (Atlantis auto-planned).
- [ ] The number of resources to add/change/destroy is reasonable for this PR's stated scope.
- [ ] Zero resources are being destroyed unexpectedly. (Destroys of resources introduced in this same PR are acceptable.)
- [ ] Any in-place updates are intentional and safe (tag changes, retention period changes are low risk; CIDR changes, engine version changes, cluster config changes are high risk).
- [ ] If there are import operations: each "will be imported" resource is expected and its ID matches the inventory.

### Resource naming and tagging

- [ ] Spot-check 2–3 new resources: do they follow `<org>-<env>-<service>` naming?
- [ ] `default_tags` is configured on the provider block, not only on individual resources.

### Security

- [ ] No ingress rules allow `0.0.0.0/0` on anything other than ports 80/443 on a public ALB.
- [ ] No IAM policies grant `"Action": "*"` or `"Resource": "*"` without explicit justification.
- [ ] No EC2 instances have key pairs — SSM only.
- [ ] No S3 buckets allow public access.
- [ ] All new secrets are written to the secrets tool, not to Terraform outputs.

### Production changes (extra scrutiny)

If the PR touches `live/production/`:

- [ ] The change was also applied in staging first (or is a deliberate exception with justification).
- [ ] Any `lifecycle { prevent_destroy = true }` is present on critical resources (DocumentDB, S3 buckets with data, Route53 zones).
- [ ] `deletion_protection = true` is set on RDS, DocumentDB clusters.
- [ ] `skip_final_snapshot = false` on all production databases.
- [ ] ALB `enable_deletion_protection = true`.
- [ ] KMS key `deletion_window_in_days >= 14`.

### Module changes

If the PR changes a module in the module library:

- [ ] `outputs.tf` is not empty.
- [ ] Every new variable has `description` and `type`.
- [ ] `validation {}` blocks are present for enum-like variables.
- [ ] `examples/complete/` compiles successfully.
- [ ] The commit message follows Conventional Commits (which drives semantic-release versioning).
- [ ] If this is a breaking change (removes a variable, changes an output name, changes resource naming): the commit message starts with `feat!:` or includes `BREAKING CHANGE:` in the footer.

---

## Approval policy

| PR type | Required approvals | Notes |
|---|---|---|
| Staging only | 1 non-author | Self-approval not allowed |
| Production only | 2 non-authors | At least one must be a senior SRE |
| Both staging and production | 2 non-authors | Always split into separate PRs if possible |
| Module library (new module) | 1 non-author | Tag bump drives versioning — review the commit message |
| Emergency hotfix | 1 non-author + post-mortem | Document the emergency in the PR description |

---

## Common issues to flag (reject the PR and ask the author to fix)

1. **Plan was not triggered.** Type `atlantis plan` to trigger it before approving.
2. **Plan shows destroys.** Do not approve until the author explains each destroy.
3. **`.terraform.lock.hcl` is missing.** Ask the author to run `terraform providers lock` and commit it.
4. **`terraform import` CLI was used instead of `import {}` blocks.** The state change happened outside the PR. Ask the author to add the import block and explain what was imported.
5. **Provider version changed without updating all roots.** If one root bumps AWS provider to 5.98.0, all roots must bump simultaneously.
6. **Secret value appears in plan output.** Terraform may print sensitive values in certain scenarios. Ask the author to mark the value `sensitive = true`.
7. **Production PR without a preceding staging apply.** Require the staging apply link (Atlantis comment or merged staging PR).
