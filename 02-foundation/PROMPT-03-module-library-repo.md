# PROMPT-03: Module Library Repository

## 1. Role and objective

You are a senior SRE setting up the `<MODULES_REPO>` Terraform module library. This repo hosts all reusable, versioned Terraform modules consumed by the infra monorepo. You will create the repo scaffold, the first six core modules (vpc, iam-role, security-group, s3-bucket, ecr-repo, documentdb), and the CI/CD release pipeline. You will also write one complete working example per module and a native Terraform test for each.

---

## 2. Preconditions

- [ ] `00-standards/conventions.md` has been read — especially sections 4 (provider versions), 5 (variable/output style), and 7 (import discipline).
- [ ] `00-standards/decisions.md` ADR-003 (two-repo split) has been read.
- [ ] `<MODULES_REPO>` has been created in Bitbucket DC and cloned locally.
- [ ] Semantic-release and conventional commits are understood by the team.

---

## 3. Required inputs

1. `<ORG>` and `<MODULES_REPO>` slug.
2. `<BITBUCKET_SSH_HOST>` and `<BB_PROJECT>` — for writing example module source strings.
3. AWS provider version to pin (from `00-standards/conventions.md` section 4 — e.g. `= 5.97.0`).
4. Which additional modules beyond the six core ones are needed immediately (ask the human).

---

## 4. In scope / out of scope

**In scope:**
- Repository root: `README.md`, `.gitignore`, `.releaserc.json`, `.pre-commit-config.yaml`, `.terraform-version`.
- Six core modules: `modules/vpc`, `modules/iam-role`, `modules/security-group`, `modules/s3-bucket`, `modules/ecr-repo`, `modules/documentdb`.
- Each module: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, `README.md` (terraform-docs markers), `examples/complete/`.
- `.github/workflows/` or equivalent CI for lint + terraform-docs + release (adapt to whatever CI the module repo uses — ask the human if unsure; if Bitbucket DC pipelines or piaas.yml, adapt accordingly).
- `.releaserc.json` using semantic-release with conventional commits.

**Out of scope:**
- Application-level modules (Kafka topics, Datadog monitors, etc.) — those come later.
- Actual Terraform state or apply — modules are never applied directly, only via root modules.

---

## 5. Reference material

- `00-standards/conventions.md` — sections 1 (module layout), 2 (naming), 4 (provider versions), 5 (variable/output style).
- Reference module convention: Pier's `pier-anomalies-tf-module` had no outputs and no tests — explicitly improve on this.

---

## 6. Step-by-step procedure

### Step 1: Repository root files

**`.terraform-version`**
```
1.11.4
```

**`.gitignore`**
```gitignore
.terraform/
*.tfplan
*.tfstate
*.tfstate.backup
*.auto.tfvars
crash.log
.DS_Store
Thumbs.db
.idea/
.vscode/
```

Note: `.terraform.lock.hcl` inside `examples/` directories IS committed.

**`.releaserc.json`**
```json
{
  "branches": ["main"],
  "ci": false,
  "plugins": [
    ["@semantic-release/commit-analyzer", {
      "preset": "conventionalcommits",
      "releaseRules": [
        {"type": "feat", "release": "minor"},
        {"type": "fix", "release": "patch"},
        {"type": "perf", "release": "patch"},
        {"breaking": true, "release": "major"}
      ]
    }],
    ["@semantic-release/release-notes-generator", {"preset": "conventionalcommits"}],
    ["@semantic-release/changelog", {"changelogFile": "CHANGELOG.md"}],
    ["@semantic-release/github"],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md"],
      "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
    }]
  ]
}
```

**`terraform-docs.yaml`** (at repo root — used by pre-commit and CI)
```yaml
formatter: markdown table
output:
  file: README.md
  mode: inject
  template: |-
    <!-- BEGIN_TF_DOCS -->
    {{ .Content }}
    <!-- END_TF_DOCS -->
settings:
  show-required-in-type: true
  anchor: true
sort:
  enabled: true
  by: required
```

### Step 2: Standard module file layout

Every module under `modules/<name>/` must have exactly these files:

```
modules/<name>/
├── main.tf          # resources
├── variables.tf     # all inputs
├── outputs.tf       # all outputs (never empty content — at minimum one output)
├── versions.tf      # required_version + required_providers (NO backend)
├── locals.tf        # internal locals (create only if needed)
├── README.md        # terraform-docs injected between markers
└── examples/
    └── complete/
        ├── main.tf
        ├── outputs.tf
        ├── versions.tf
        └── .terraform.lock.hcl
```

**`versions.tf` template for every module:**
```hcl
terraform {
  required_version = ">= 1.11"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.97"
    }
  }
}
```

Note: modules use `~>` (pessimistic), not `=` (exact). Only root modules pin exactly.

**`README.md` template:**
```markdown
# <module-name>

Brief one-paragraph description.

## Usage

```hcl
module "<name>" {
  source = "git::ssh://git@<BITBUCKET_SSH_HOST>:7999/<BB_PROJECT>/<MODULES_REPO>.git//modules/<name>?ref=v1.0.0"

  # required inputs
}
```

<!-- BEGIN_TF_DOCS -->
<!-- END_TF_DOCS -->
```

### Step 3: Module — `modules/vpc`

**`variables.tf`:**
```hcl
variable "name" {
  description = "Name to use for all resources. Becomes the VPC Name tag."
  type        = string
}

variable "cidr_block" {
  description = "IPv4 CIDR block for the VPC."
  type        = string
  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "cidr_block must be a valid IPv4 CIDR."
  }
}

variable "azs" {
  description = "List of availability zone names to create subnets in."
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets, one per AZ."
  type        = list(string)
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets, one per AZ."
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Whether to create NAT gateways for private subnets."
  type        = bool
  default     = true
}

variable "single_nat_gateway" {
  description = "Use a single NAT gateway instead of one per AZ (cost saving for non-prod)."
  type        = bool
  default     = false
}

variable "tags" {
  description = "Additional tags to merge with default_tags."
  type        = map(string)
  default     = {}
}
```

**`main.tf`:** Use `terraform-aws-modules/vpc/aws` version `5.19.0` as a thin wrapper, adding org-standard tag enforcement and safe defaults.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.19.0"

  name = var.name
  cidr = var.cidr_block
  azs  = var.azs

  private_subnets = var.private_subnet_cidrs
  public_subnets  = var.public_subnet_cidrs

  enable_nat_gateway   = var.enable_nat_gateway
  single_nat_gateway   = var.single_nat_gateway
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = var.tags
}
```

**`outputs.tf`:**
```hcl
output "vpc_id" {
  description = "ID of the VPC."
  value       = module.vpc.vpc_id
}

output "vpc_cidr_block" {
  description = "CIDR block of the VPC."
  value       = module.vpc.vpc_cidr_block
}

output "private_subnet_ids" {
  description = "IDs of the private subnets."
  value       = module.vpc.private_subnets
}

output "public_subnet_ids" {
  description = "IDs of the public subnets."
  value       = module.vpc.public_subnets
}

output "nat_gateway_ids" {
  description = "IDs of the NAT gateways."
  value       = module.vpc.natgw_ids
}
```

### Step 4: Module — `modules/iam-role`

A general-purpose wrapper for creating an IAM role with a structured trust policy.

**`variables.tf`:**
```hcl
variable "name" {
  description = "Name of the IAM role."
  type        = string
}

variable "trust_policy_json" {
  description = "JSON-encoded IAM trust (assume-role) policy document."
  type        = string
}

variable "policy_arns" {
  description = "List of managed policy ARNs to attach."
  type        = list(string)
  default     = []
}

variable "inline_policies" {
  description = "Map of inline policy name => JSON policy document."
  type        = map(string)
  default     = {}
}

variable "max_session_duration" {
  description = "Maximum CLI/API session duration in seconds (900–43200)."
  type        = number
  default     = 3600
}

variable "tags" {
  description = "Additional tags."
  type        = map(string)
  default     = {}
}
```

**`outputs.tf`:**
```hcl
output "role_arn" {
  description = "ARN of the IAM role."
  value       = aws_iam_role.this.arn
}

output "role_name" {
  description = "Name of the IAM role."
  value       = aws_iam_role.this.name
}

output "role_id" {
  description = "Stable, unique string identifying the role."
  value       = aws_iam_role.this.unique_id
}
```

### Step 5: Module — `modules/security-group`

Thin wrapper over `aws_security_group` that enforces descriptions on all rules.

**Key variables:** `name`, `description`, `vpc_id`, `ingress_rules` (list of objects with `from_port`, `to_port`, `protocol`, `cidr_blocks`, `description`), `egress_rules`, `tags`.

**Key outputs:** `security_group_id`, `security_group_arn`.

### Step 6: Module — `modules/s3-bucket`

Opinionated S3 bucket with SSE-KMS, versioning, and public access block enforced by default.

**Key variables:** `bucket_name`, `versioning_enabled` (default true), `kms_key_id`, `cors_rules` (list of objects, optional), `lifecycle_rules`, `tags`.

**Key outputs:** `bucket_id`, `bucket_arn`, `bucket_region`.

### Step 7: Module — `modules/ecr-repo`

```hcl
# modules/ecr-repo/variables.tf (key variables)
variable "name" { type = string; description = "Repository name." }
variable "image_tag_mutability" { type = string; default = "IMMUTABLE"; ... }
variable "scan_on_push" { type = bool; default = true }
variable "lifecycle_policy_json" { type = string; default = null }
variable "tags" { type = map(string); default = {} }
```

```hcl
# outputs.tf
output "repository_url" { ... }
output "repository_arn" { ... }
output "registry_id"    { ... }
```

### Step 8: Module — `modules/documentdb`

Wraps `aws_docdb_cluster` and `aws_docdb_cluster_instance` with secure defaults.

**Key variables:** `cluster_identifier`, `engine_version`, `master_username`, `master_password` (sensitive), `instance_class`, `instance_count`, `vpc_id`, `subnet_ids`, `allowed_security_group_ids`, `backup_retention_period`, `deletion_protection`, `skip_final_snapshot`, `tags`.

**Key outputs:** `cluster_endpoint`, `cluster_reader_endpoint`, `cluster_port`, `cluster_id`, `security_group_id`.

**Security defaults to enforce in `main.tf`:**
- `storage_encrypted = true`
- `deletion_protection = var.deletion_protection` (default `true` for production, must be set explicitly)
- `skip_final_snapshot = false` (default — operator must set `true` for non-prod)
- `tls_enabled = true` in the cluster parameter group

### Step 9: examples/complete for each module

Each `examples/complete/main.tf` must:
- Use a real source string with `?ref=v0.0.0` as a placeholder.
- Declare all variables with reasonable defaults.
- Include a `provider "aws"` block with `region = "us-east-1"`.
- Include a `terraform { required_version = "~> 1.11" }` block.
- NOT include a backend block (examples use local state).

### Step 10: CI configuration

Ask the human whether the module library will use Bitbucket DC pipelines, piaas.yml, or GitHub Actions (if it is mirrored to GitHub). Write the CI config accordingly. The CI must:

1. On every PR: run `terraform fmt -check`, `tflint`, `terraform-docs --check` (docs must be up to date).
2. On merge to `main` when `*.tf` files changed: run semantic-release to create a git tag and Bitbucket/GitHub release.

If using piaas.yml, the structure will be confirmed in PROMPT-05.

---

## 7. Expected file tree after completion

```
<MODULES_REPO>/
├── .gitignore
├── .terraform-version
├── .releaserc.json
├── .pre-commit-config.yaml
├── terraform-docs.yaml
├── README.md
└── modules/
    ├── vpc/
    │   ├── main.tf, variables.tf, outputs.tf, versions.tf, README.md
    │   └── examples/complete/
    ├── iam-role/
    │   └── ...
    ├── security-group/
    │   └── ...
    ├── s3-bucket/
    │   └── ...
    ├── ecr-repo/
    │   └── ...
    └── documentdb/
        └── ...
```

---

## 8. Code contracts

Every module `versions.tf` uses:
```hcl
required_version = ">= 1.11"
```
Every module `outputs.tf` has at least one output — no empty `outputs.tf` files.

---

## 9. Acceptance criteria

- [ ] `terraform validate` passes in every module and every `examples/complete/`.
- [ ] `terraform-docs` run on each module produces up-to-date README content.
- [ ] `tflint` produces no errors on any module.
- [ ] `.terraform.lock.hcl` is committed in every `examples/complete/`.
- [ ] Every module has at least one output.
- [ ] Every variable has a `description` and explicit `type`.

---

## 10. Guardrails

- Modules must never contain `backend {}` blocks.
- Modules must never contain `provider {}` blocks (providers are configured in the calling root module only). Exception: alias providers for multi-region modules — document this explicitly.
- Modules must never hardcode account IDs, regions, or environment names.
- Do not mark `master_password` as an output in the DocumentDB module — it is write-only.

---

## 11. Handoff note

When complete, report:
1. List of modules created with their output count.
2. Confirm `terraform validate` and `terraform-docs` pass for all modules.
3. The source string pattern consumers should use (with the SSH host filled in).
4. Any design decisions made (e.g. the DocumentDB module uses a cluster parameter group for TLS — explain why).
