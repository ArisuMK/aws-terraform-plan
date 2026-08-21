# PROMPT-01: Bootstrap State Backend

## 1. Role and objective

You are a senior SRE setting up the Terraform state backend — the S3 buckets, KMS keys, and IAM execution roles that all other Terraform layers depend on. This layer bootstraps itself: it starts with a local state file, provisions the S3 bucket, then migrates its own state to that bucket. After this prompt is complete, every subsequent layer can use S3 remote state.

You will produce the `live/staging/00-bootstrap/` and `live/production/00-bootstrap/` directories inside the `<INFRA_REPO>` monorepo.

---

## 2. Preconditions

Confirm with the human before starting:

- [ ] `00-standards/conventions.md` has been read in full.
- [ ] `00-standards/decisions.md` ADR-001 and ADR-005 have been read.
- [ ] `00-standards/fill-in-the-blanks.local.md` has values for: `<ORG>`, `<STAGING_ACCOUNT_ID>`, `<PRODUCTION_ACCOUNT_ID>`, `<STAGING_REGION>`, `<PRODUCTION_REGION>`.
- [ ] `PROMPT-00` (inventory) has been completed. No existing state bucket found with a conflicting name, OR the human has confirmed the exact bucket names to use.
- [ ] The human has local AWS credentials with `AdministratorAccess` or equivalent for both accounts, to run the one-time manual bootstrap apply.
- [ ] The infra repo (`<INFRA_REPO>`) exists in Bitbucket DC and has been cloned locally.

---

## 3. Required inputs

Ask the human to confirm:

1. `<ORG>` — organization short name (lowercase, no hyphens longer than one word).
2. `<STAGING_ACCOUNT_ID>` and `<STAGING_REGION>`.
3. `<PRODUCTION_ACCOUNT_ID>` and `<PRODUCTION_REGION>`.
4. Any existing IAM roles that must NOT be modified or deleted (from the inventory, Section D of the questionnaire).
5. Staging EKS OIDC for Atlantis trust: **usually "not yet"**. If the staging EKS cluster is still in another account (to be transferred later), skip OIDC provider creation and Atlantis trust statements now. Revisit after transfer when `<STAGING_OIDC_URL>` and `<STAGING_OIDC_ARN>` exist in this account. Only if the cluster is already in `<STAGING_ACCOUNT_ID>` should bootstrap wire IRSA trust.

---

## 4. In scope / out of scope

**In scope:**
- S3 state bucket per account with versioning, SSE-KMS, public access block, TLS-only bucket policy.
- KMS key and alias for state bucket encryption.
- IAM execution role (`<ORG>-<env>-terraform-exec`) per account, with an inline policy granting all Terraform operations.
- IAM trust policy on the execution role: human/bootstrap principal now; **add** Atlantis pod OIDC trust later when staging EKS is in this account (PROMPT-04).
- OIDC provider for the staging EKS cluster — **only if** the cluster already exists in this account; otherwise defer.
- Instructions for the one-time manual bootstrap procedure.

**Out of scope:**
- VPC, subnets, security groups.
- Any application resources.
- The Atlantis Kubernetes deployment itself (covered in PROMPT-04).

---

## 5. Reference material

Read before generating code:
- `00-standards/conventions.md` — sections 1 (layout), 2 (naming), 3 (tagging), 5 (variable style).
- `00-standards/decisions.md` — ADR-001 (S3 native locking), ADR-002 (Atlantis), ADR-005 (directory layout).

---

## 6. Step-by-step procedure

### Step 1: Create the directory scaffold

Create this exact directory structure in the infra repo:

```
live/
  staging/
    00-bootstrap/
      backend.tf
      versions.tf
      providers.tf
      locals.tf
      data.tf
      kms.tf
      s3.tf
      iam.tf
      oidc.tf          (only if OIDC provider does not already exist per inventory)
      outputs.tf
      .terraform.lock.hcl
  production/
    00-bootstrap/
      (same files)
```

### Step 2: Write `backend.tf`

**Important:** The bootstrap layer starts with a local backend. It is migrated to S3 after the bucket is created.

```hcl
# live/staging/00-bootstrap/backend.tf

terraform {
  # INITIAL: local backend for bootstrapping
  # After the S3 bucket is created and 'terraform apply' succeeds:
  # 1. Uncomment the s3 backend block below
  # 2. Comment out or remove the local backend
  # 3. Run 'terraform init -migrate-state' to migrate

  backend "local" {}

  # backend "s3" {
  #   bucket       = "<ORG>-tfstate-uat-<STAGING_REGION>"
  #   key          = "staging/00-bootstrap/terraform.tfstate"
  #   region       = "<STAGING_REGION>"
  #   encrypt      = true
  #   kms_key_id   = "alias/<ORG>-uat-tfstate"
  #   use_lockfile = true
  #   assume_role  = {
  #     role_arn = "arn:aws:iam::<STAGING_ACCOUNT_ID>:role/<ORG>-uat-terraform-exec"
  #   }
  # }

  required_version = "~> 1.11"
}
```

For production, use `backend "local" {}` with the equivalent commented-out S3 block pointing to the production bucket.

### Step 3: Write `versions.tf`

```hcl
# live/staging/00-bootstrap/versions.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 5.97.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "= 3.6.3"
    }
  }
}
```

### Step 4: Write `providers.tf`

```hcl
# live/staging/00-bootstrap/providers.tf

provider "aws" {
  region = local.region

  default_tags {
    tags = local.default_tags
  }
}
```

During bootstrapping, the human runs with their own credentials — no `assume_role` here yet. After the exec role exists, this can be updated to assume it.

### Step 5: Write `locals.tf`

```hcl
# live/staging/00-bootstrap/locals.tf

locals {
  org        = "<ORG>"
  env      = "uat"
  env_long   = "staging"
  layer      = "00-bootstrap"
  region     = "<STAGING_REGION>"
  account_id = data.aws_caller_identity.current.account_id

  # Atlantis OIDC identity for trust policy
  # Format: system:serviceaccount:<namespace>:<service-account-name>
  atlantis_sa_subject = "system:serviceaccount:<ATLANTIS_NAMESPACE>:atlantis"

  default_tags = {
    Organization = local.org
    Environment  = local.env_long
    ManagedBy    = "terraform"
    Repository   = "<INFRA_REPO>"
    Layer        = local.layer
  }
}
```

### Step 6: Write `data.tf`

```hcl
# live/staging/00-bootstrap/data.tf

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# Only include if OIDC provider already exists (check inventory):
# data "aws_iam_openid_connect_provider" "eks" {
#   url = "https://<STAGING_OIDC_URL>"
# }
```

### Step 7: Write `kms.tf`

```hcl
# live/staging/00-bootstrap/kms.tf

resource "aws_kms_key" "tfstate" {
  description             = "KMS key for Terraform state bucket encryption - ${local.env_long}"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  tags = {
    Name    = "${local.org}-${local.env}-tfstate-kms"
    Service = "bootstrap"
  }
}

resource "aws_kms_alias" "tfstate" {
  name          = "alias/${local.org}-${local.env}-tfstate"
  target_key_id = aws_kms_key.tfstate.key_id
}
```

### Step 8: Write `s3.tf`

```hcl
# live/staging/00-bootstrap/s3.tf

resource "aws_s3_bucket" "tfstate" {
  bucket = "${local.org}-tfstate-${local.env}-${local.region}"

  tags = {
    Name    = "${local.org}-tfstate-${local.env}-${local.region}"
    Service = "bootstrap"
  }
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.tfstate.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "tfstate" {
  bucket                  = aws_s3_bucket.tfstate.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_policy" "tfstate_tls_only" {
  bucket = aws_s3_bucket.tfstate.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "DenyNonTLS"
      Effect    = "Deny"
      Principal = "*"
      Action    = "s3:*"
      Resource  = [
        aws_s3_bucket.tfstate.arn,
        "${aws_s3_bucket.tfstate.arn}/*"
      ]
      Condition = {
        Bool = { "aws:SecureTransport" = "false" }
      }
    }]
  })
}
```

### Step 9: Write `iam.tf`

```hcl
# live/staging/00-bootstrap/iam.tf

data "aws_iam_policy_document" "terraform_exec_trust" {
  # Trust Atlantis pod via OIDC (EKS IRSA)
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = ["<STAGING_OIDC_ARN>"]
    }
    condition {
      test     = "StringEquals"
      variable = "<STAGING_OIDC_URL>:sub"
      values   = [local.atlantis_sa_subject]
    }
    condition {
      test     = "StringEquals"
      variable = "<STAGING_OIDC_URL>:aud"
      values   = ["sts.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "terraform_exec" {
  name               = "${local.org}-${local.env}-terraform-exec"
  assume_role_policy = data.aws_iam_policy_document.terraform_exec_trust.json

  tags = {
    Name    = "${local.org}-${local.env}-terraform-exec"
    Service = "bootstrap"
  }
}

# Attach AdministratorAccess for initial build-out.
# Scope this down to a custom policy after all layers are stable.
resource "aws_iam_role_policy_attachment" "terraform_exec_admin" {
  role       = aws_iam_role.terraform_exec.name
  policy_arn = "arn:aws:partition:iam::aws:policy/AdministratorAccess"
}
```

**Note to the executing AI:** Replace `arn:aws:partition` with the correct partition for the account (`aws`, `aws-cn`, or `aws-us-gov`). For standard commercial AWS it is `arn:aws:iam::aws:policy/AdministratorAccess`.

For production, the trust policy must reference the **staging** OIDC ARN and URL — Atlantis runs in staging but assumes the production exec role cross-account.

```hcl
# live/production/00-bootstrap/iam.tf (trust policy portion)
data "aws_iam_policy_document" "terraform_exec_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = ["<STAGING_OIDC_ARN>"]   # staging OIDC — Atlantis lives in staging
    }
    condition {
      test     = "StringEquals"
      variable = "<STAGING_OIDC_URL>:sub"
      values   = [local.atlantis_sa_subject]
    }
    condition {
      test     = "StringEquals"
      variable = "<STAGING_OIDC_URL>:aud"
      values   = ["sts.amazonaws.com"]
    }
  }
}
```

### Step 10: Write `oidc.tf` (only if OIDC provider not already in inventory)

```hcl
# live/staging/00-bootstrap/oidc.tf
# SKIP THIS FILE if the OIDC provider already exists. Import it instead.

resource "aws_iam_openid_connect_provider" "eks" {
  url = "https://<STAGING_OIDC_URL>"

  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["<EKS_OIDC_THUMBPRINT>"]

  tags = {
    Name    = "${local.org}-${local.env}-eks-oidc"
    Service = "bootstrap"
  }
}
```

If the OIDC provider already exists, add an import block instead:

```hcl
import {
  to = aws_iam_openid_connect_provider.eks
  id = "<STAGING_OIDC_ARN>"
}
```

### Step 11: Write `outputs.tf`

```hcl
# live/staging/00-bootstrap/outputs.tf

output "tfstate_bucket_name" {
  description = "Name of the Terraform state S3 bucket."
  value       = aws_s3_bucket.tfstate.bucket
}

output "tfstate_kms_key_arn" {
  description = "ARN of the KMS key used to encrypt the state bucket."
  value       = aws_kms_key.tfstate.arn
}

output "tfstate_kms_alias" {
  description = "KMS alias for the state bucket encryption key."
  value       = aws_kms_alias.tfstate.name
}

output "terraform_exec_role_arn" {
  description = "ARN of the IAM role Atlantis assumes to run Terraform."
  value       = aws_iam_role.terraform_exec.arn
}
```

### Step 12: Generate and commit `.terraform.lock.hcl`

After writing all files, run:

```bash
cd live/staging/00-bootstrap
terraform init
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_arm64
```

Commit the `.terraform.lock.hcl` file. Repeat for `live/production/00-bootstrap`.

### Step 13: Manual bootstrap procedure

Document these steps in a `BOOTSTRAP.md` file at the root of the infra repo:

```markdown
## One-time bootstrap procedure

### Prerequisites
- AWS CLI configured with admin credentials for staging and production accounts.
- Terraform ~> 1.11 installed locally.

### Staging

1. cd live/staging/00-bootstrap
2. terraform init
3. terraform plan -out=tfplan
4. Review the plan: it should create 1 KMS key, 1 KMS alias, 1 S3 bucket and its
   config resources, 1 IAM role, 1 OIDC provider (if needed).
5. terraform apply tfplan
6. Edit backend.tf: uncomment the s3 backend block, comment out local.
7. terraform init -migrate-state
   (type 'yes' when prompted to copy state to S3)
8. terraform plan  (should show No changes)
9. Commit and push the updated backend.tf.

### Production
Repeat steps 1-9 in live/production/00-bootstrap.
```

---

## 7. Expected file tree after completion

```
live/
  staging/00-bootstrap/
    backend.tf
    versions.tf
    providers.tf
    locals.tf
    data.tf
    kms.tf
    s3.tf
    iam.tf
    oidc.tf              (if needed)
    outputs.tf
    .terraform.lock.hcl
  production/00-bootstrap/
    (same)
BOOTSTRAP.md
```

---

## 8. Code contracts

Every root module in this repo must use this exact backend shape (post-migration):

```hcl
terraform {
  backend "s3" {
    bucket       = "<ORG>-tfstate-<env>-<region>"
    key          = "<env>/<layer>/terraform.tfstate"
    region       = "<region>"
    encrypt      = true
    kms_key_id   = "alias/<ORG>-<env>-tfstate"
    use_lockfile = true
    assume_role  = {
      role_arn = "arn:aws:iam::<account_id>:role/<ORG>-<env>-terraform-exec"
    }
  }
  required_version = "~> 1.11"
}
```

---

## 9. Acceptance criteria

- [ ] `terraform validate` passes in both `staging/00-bootstrap` and `production/00-bootstrap`.
- [ ] `terraform plan` (after migration) shows `No changes. Your infrastructure matches the configuration.`
- [ ] State file is visible in S3: `aws s3 ls s3://<org>-tfstate-uat-<region>/staging/00-bootstrap/`
- [ ] Lock file for the bootstrap state object exists alongside the state file (`.terraform.tfstate.lock` object in S3).
- [ ] The `terraform_exec` role is assumable from the Atlantis pod service account.
- [ ] `.terraform.lock.hcl` is committed for both platforms.

---

## 10. Guardrails

- Never delete the state bucket or KMS key — all other layers depend on them.
- Never apply with `--auto-approve` even during bootstrapping.
- Do not commit real AWS account IDs in `.tf` files — they live in `locals.tf` via `data.aws_caller_identity`.
- Do not use `AdministratorAccess` permanently — after all layers are stable, scope down the exec role policy.
- Never store the bootstrap state in a shared bucket that another team controls.

---

## 11. Handoff note

When complete, report:

1. Exact bucket names created for staging and production.
2. KMS key ARNs.
3. IAM execution role ARNs.
4. Whether state was successfully migrated to S3 (post-migration plan is clean).
5. Any deviations from the standard — for example, if the OIDC provider already existed and was imported.
6. The content of the final `backend.tf` files (post-migration) — needed by PROMPT-02 to configure `atlantis.yaml`.
