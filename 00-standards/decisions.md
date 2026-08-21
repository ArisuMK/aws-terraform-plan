# Architecture Decision Records

This file contains the key architectural decisions for this Terraform setup. Each ADR records the context, the decision, and the rationale. Future team members should read these before proposing changes to the patterns.

---

## ADR-001: S3 remote state with native locking (no DynamoDB)

**Status:** Accepted  
**Date:** 2026-08

### Context

Terraform state must be stored remotely and locked to prevent concurrent applies from corrupting it. Historically the standard pattern used S3 for storage plus DynamoDB for locking. As of Terraform 1.11, DynamoDB-based locking is officially deprecated.

The previous org (reference: `pier-infrastructure`) used S3 backends with no locking at all, which is not acceptable for a multi-person team.

### Decision

- Remote state: S3 bucket per AWS account.
- Locking: `use_lockfile = true` in the S3 backend (Terraform 1.11+ native S3 object locking). No DynamoDB table.
- Encryption: SSE-KMS with a per-account KMS key. Key alias: `alias/<org>-<env>-tfstate`.
- Versioning: enabled on the state bucket.
- Public access: blocked on all axes.
- TLS enforcement: bucket policy denies non-HTTPS requests.

```hcl
terraform {
  backend "s3" {
    bucket       = "<org>-tfstate-<env>-<region>"
    key          = "<env>/<layer>/terraform.tfstate"
    region       = "<region>"
    encrypt      = true
    kms_key_id   = "alias/<org>-<env>-tfstate"
    use_lockfile = true
    assume_role  = {
      role_arn = "arn:aws:iam::<account_id>:role/<org>-<env>-terraform-exec"
    }
  }
  required_version = "~> 1.11"
}
```

### Consequences

- `terraform >= 1.11` is a hard requirement everywhere. Annotated in `.terraform-version`.
- `00-bootstrap` layer must create the S3 bucket before any other layer runs — it uses a local backend initially and migrates to S3 after the bucket exists.
- Removing DynamoDB removes one resource family to manage, monitor, and pay for.

---

## ADR-002: Atlantis for PR-driven plan and apply

**Status:** Accepted  
**Date:** 2026-08

### Context

We need a GitOps-style workflow where:

1. Terraform `plan` runs automatically on every PR.
2. Plan output is visible in the PR as a comment.
3. `apply` requires explicit human approval and only runs on approved PRs.
4. State is never touched directly by developer workstations.
5. Multiple concurrent PRs do not corrupt each other's state (locking).

We evaluated:
- **Plain `piaas.yml` pipeline** — gives plan visibility but has no PR-level locking, no apply gating, and requires credentials on the pipeline runner.
- **Spacelift / env0** — SaaS platforms that need a private worker to reach Bitbucket Data Center. Additional cost and vendor dependency.
- **Digger** — newer, less mature Bitbucket DC support.
- **Atlantis** — battle-tested, first-class Bitbucket Data Center support (webhook secret, PR comments, apply command), self-hosted, no AWS credentials on the pipeline runner.

### Decision

Deploy Atlantis on the existing EKS staging cluster with IRSA. It assumes `terraform-exec` IAM roles in each target account using the EKS pod's OIDC identity.

- Bitbucket DC VCS config: `--vcs bitbucketserver`, `--bitbucket-base-url`, `--bitbucket-webhook-secret`.
- Atlantis config: `atlantis.yaml` at repo root, one project per `live/<env>/<layer>` directory.
- Apply command: `atlantis apply` typed as a PR comment by an authorized user.
- Auto-plan: enabled on `**.tf`, `**.tfvars`, `**/.terraform.lock.hcl` file changes.
- Merge requirements: plan must succeed AND at least one non-author approval before apply is allowed.

### Consequences

- Atlantis must have network access to Bitbucket DC (HTTPS webhooks inbound to Atlantis, Atlantis calls Bitbucket API).
- EKS cluster and IRSA are required before Atlantis can operate — bootstrapping must handle this chicken-and-egg problem (see `PROMPT-01`).
- Team members approve PRs in Bitbucket; they type `atlantis apply` in the PR comment, not via AWS console or CLI.

---

## ADR-003: Split into two repos — infra monorepo + module library

**Status:** Accepted  
**Date:** 2026-08

### Context

Pier used one module repo per module (`pier-rds-tf-module`, `pier-s3-tf-module`, `pier-irsa-tf-module`, …), resulting in version drift: the same `pier-data-tf-module` was at v1.9.2 in prod and v1.10.0 in staging.

An alternative is to colocate modules in the infra monorepo (`modules/` subdirectory). This simplifies the source string but makes it impossible to version modules independently, and every module change triggers an Atlantis plan across all roots.

### Decision

Two repositories:

1. **`<org>-infrastructure`** — live root modules under `live/<env>/<layer>/`. Thin wiring: calls modules, defines state backend, sets providers.
2. **`<org>-terraform-modules`** — one repo, all modules under `modules/<name>/`. Tagged with semver. Released via semantic-release.

Module source in live configs:

```hcl
source = "git::ssh://git@<BITBUCKET_HOST>:7999/<PROJECT>/<org>-terraform-modules.git//modules/vpc?ref=v1.3.0"
```

Version bumps are a deliberate PR to the infra repo — visible in the diff, reviewed, and applied via Atlantis. No silent drift.

### Consequences

- Bitbucket DC must allow SSH key access from the Atlantis pod for module downloads.
- Or: use HTTPS with a service-account token in `GIT_CREDENTIALS` env var on the Atlantis deployment.
- Module repo uses semantic-release with conventional commits, same pattern as `pier-anomalies-tf-module`.
- Releases are triggered by push to `main`; a GitHub-style tag (`v1.3.0`) is created automatically.

---

## ADR-004: Declarative import blocks for brownfield resources

**Status:** Accepted  
**Date:** 2026-08

### Context

Several AWS resources already exist: EKS cluster (platform-provisioned), VPC, IAM roles written ad hoc, DocumentDB, EC2 instances. These must be brought under Terraform management without being destroyed and recreated.

The `terraform import` CLI command:
- Requires an interactive shell with write access to the state — incompatible with Atlantis.
- Leaves no PR record of what was imported.
- Does not validate the matching HCL config exists.

### Decision

Use Terraform 1.5+ `import {}` blocks exclusively:

```hcl
import {
  to   = module.vpc.aws_vpc.this[0]
  id   = "vpc-0abc123def456"
}
```

Workflow:
1. Write the `import {}` block alongside the matching resource config.
2. Run `terraform plan` (via Atlantis). The plan shows "will import" — no resources destroyed.
3. PR review confirms the import is correct.
4. Merge and `atlantis apply`.
5. Follow-up PR removes the `import {}` block. Plan is clean `No changes`.

`-generate-config-out=generated.tf` is used locally to produce a skeleton config from the live resource. The skeleton is then refactored into module calls before the PR.

### Consequences

- Never use `terraform import` CLI against shared state. Documented in `04-operations/import-playbook.md`.
- Import PRs are smaller and focused: one layer at a time.
- The EKS layer is deferred until after the `QUESTIONNAIRE-20-eks-handover.md` is answered, to decide between full adoption, partial adoption with `lifecycle { ignore_changes = all }`, or read-only data sources.

---

## ADR-005: Directory-per-layer numbered layout

**Status:** Accepted  
**Date:** 2026-08

### Context

Pier's `terraform/aws/prod/` was a single flat root with every resource in one state. This caused long plan times, wide blast radius, and implicit dependencies between resources managed in one apply.

Alternative: Terragrunt with `dependency {}` blocks for DRY root module references. Adds a tool, a learning curve, and complexity that the team may not need yet.

### Decision

Multiple independent root modules in numbered directories:

```
live/
  staging/
    00-bootstrap/
    10-identity/
    20-network/
    30-dns/
    40-data/
    50-compute/
    60-storage/
    70-observability/
    80-eks/
```

Numbered prefix encodes the dependency order. Cross-layer data consumed via `data "terraform_remote_state"` or tag-based data sources (preferred).

Each directory is a separate Atlantis project. Plans run in parallel; applies respect the numeric order.

No Terragrunt. The team will re-evaluate after the initial build-out if DRY becomes a pain point.

### Consequences

- Each root has its own backend config (`backend.tf`) — some duplication, but each file is small and explicit.
- Adding a new layer is a PR that adds a directory and an `atlantis.yaml` project entry.
- `00-bootstrap` is special: it must be applied manually once with local state, then migrated to S3.

---

## ADR-006: piaas.yml owns quality gates only

**Status:** Accepted  
**Date:** 2026-08

### Context

The company uses `piaas.yml` as the standard pipeline definition. Terraform plan/apply could be triggered from this pipeline, but:

- Pipeline runners do not hold long-lived AWS credentials safely.
- There is no PR-level state locking in pipeline-based workflows.
- Plan output in a pipeline log is not as visible as an Atlantis PR comment.

### Decision

`piaas.yml` runs fast, stateless quality gates on every push and PR:

- `terraform fmt -check -recursive`
- `terraform validate` (with `-backend=false`)
- `tflint --recursive`
- `trivy config .`
- `checkov -d .`
- `terraform-docs` check (README must be up to date)

Plan and apply are exclusively Atlantis. The pipeline and Atlantis are independent: a failing pipeline blocks merge (protecting code quality), but Atlantis `atlantis plan` can still run for debugging.

### Consequences

- `piaas.yml` never holds `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY`.
- Need to know the exact `piaas.yml` schema and available runner images before writing `PROMPT-05`. See `QUESTIONNAIRE-org-context.md`.
- All Terraform tool versions in `piaas.yml` must match `.terraform-version` and Atlantis's configured version.

---

## ADR-007: EKS layer deferred pending questionnaire

**Status:** Accepted  
**Date:** 2026-08

### Context

EKS was provisioned by a company platform automation tool and has an existing Terraform state file. The platform may continue mutating the cluster. If we write EKS Terraform that conflicts with the platform, we risk destructive plan diffs.

### Decision

The `80-eks/` layer is a placeholder until `QUESTIONNAIRE-20-eks-handover.md` is answered. The questionnaire determines:

- **Full adoption**: we own the state, the platform stops touching the cluster. Import blocks bring the existing cluster into our state.
- **Partial adoption with `lifecycle { ignore_changes = all }`**: we own the config but the platform may still update certain fields. We ignore those fields.
- **Read-only data sources**: we never manage EKS in Terraform; we only read cluster metadata for other layers (IRSA, ALB, etc.).

No EKS Terraform is written until this decision is made.

### Consequences

- `80-eks/` directory exists but is empty except for a `README.md` pointing to the questionnaire.
- IRSA roles for apps that run on EKS can still be written in `10-identity/` using `data "aws_eks_cluster"` data sources.
