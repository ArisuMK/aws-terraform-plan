# PROMPT-15: Storage — S3 and ECR

## 1. Role and objective

You are a senior SRE building the `60-storage` layer for both staging and production. This layer manages S3 buckets (application data, ALB access logs, assets) and ECR repositories. Existing buckets and repos from the inventory are imported using declarative import blocks. New resources follow the org naming and security conventions (SSE-KMS, versioning, public access block).

---

## 2. Preconditions

- [ ] PROMPT-10 (identity) complete: IAM roles available for IRSA access to S3.
- [ ] PROMPT-01 complete: the `modules/s3-bucket` and `modules/ecr-repo` modules exist in the module library.
- [ ] Inventory has: S3 bucket names, ECR repository names, existing replication configs.
- [ ] KMS key ARN from `00-bootstrap` layer (for SSE-KMS on new buckets).

---

## 3. Required inputs

1. List of S3 buckets from inventory to import.
2. List of ECR repositories from inventory to import.
3. New S3 buckets to create (e.g. ALB access logs, application asset storage).
4. ECR repos: are they shared across staging and production (in the staging account) or per-account?
5. What KMS key should be used for S3 encryption — the state bucket key or a dedicated per-bucket key?
6. Should ECR repos have image tag immutability? (Recommend: `IMMUTABLE` for production, `MUTABLE` for staging).
7. ECR lifecycle policy: how many untagged images and how many tagged images to keep?

---

## 4. In scope / out of scope

**In scope:**
- Application S3 buckets (assets, uploads, reports, backups).
- ALB access log S3 bucket (required for PROMPT-14 `access_logs` block).
- ECR repositories for all application container images.
- S3 bucket policies allowing ALB log delivery.
- ECR lifecycle policies.
- Import blocks for existing buckets and repos.

**Out of scope:**
- The Terraform state S3 bucket (managed in `00-bootstrap`).
- S3 static website hosting (add if needed per application requirements).
- S3 replication (add if required by DR policy).
- Cross-region ECR replication (add if required).

---

## 5. Reference material

- `00-standards/conventions.md` — naming: `<org>-<env>-<purpose>`, S3 buckets must be globally unique.
- `01-discovery/inventory.md` — storage section.

---

## 6. Step-by-step procedure

### Step 1: File structure

```
live/staging/60-storage/
  backend.tf, versions.tf, providers.tf, locals.tf, data.tf
  s3.tf
  ecr.tf
  outputs.tf
  .terraform.lock.hcl
```

### Step 2: `data.tf` — upstream state and account info

```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
data "aws_elb_service_account" "main" {}   # needed for ALB access log bucket policy

data "terraform_remote_state" "bootstrap" {
  backend = "s3"
  config = {
    bucket      = "<ORG>-tfstate-stg-<STAGING_REGION>"
    key         = "staging/00-bootstrap/terraform.tfstate"
    region      = "<STAGING_REGION>"
    assume_role = { role_arn = "arn:aws:iam::<STAGING_ACCOUNT_ID>:role/<ORG>-stg-terraform-exec" }
  }
}

locals {
  kms_key_arn = data.terraform_remote_state.bootstrap.outputs.tfstate_kms_key_arn
  # If a dedicated storage KMS key is preferred, create one in this layer instead.
}
```

### Step 3: `s3.tf` — ALB access logs bucket

The ALB access logs bucket has specific requirements: must allow the regional ELB service account to write.

```hcl
module "s3_alb_logs" {
  source = "git::ssh://git@<BITBUCKET_SSH_HOST>:7999/<BB_PROJECT>/<MODULES_REPO>.git//modules/s3-bucket?ref=v1.0.0"

  bucket_name        = "${local.org}-${local.env}-alb-logs-${local.region}"
  versioning_enabled = false   # logs are write-once
  kms_key_id         = local.kms_key_arn

  tags = { Service = "storage", Purpose = "alb-access-logs" }
}

# ALB service account needs s3:PutObject on the logs bucket
resource "aws_s3_bucket_policy" "alb_logs" {
  bucket = module.s3_alb_logs.bucket_id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowALBLogging"
      Effect    = "Allow"
      Principal = { AWS = data.aws_elb_service_account.main.arn }
      Action    = "s3:PutObject"
      Resource  = "${module.s3_alb_logs.bucket_arn}/alb/*"
    }]
  })
}
```

**Application data bucket example:**

```hcl
module "s3_app_assets" {
  source = "git::ssh://git@<BITBUCKET_SSH_HOST>:7999/<BB_PROJECT>/<MODULES_REPO>.git//modules/s3-bucket?ref=v1.0.0"

  bucket_name        = "${local.org}-${local.env}-app-assets"
  versioning_enabled = true
  kms_key_id         = local.kms_key_arn

  cors_rules = [{
    allowed_headers = ["*"]
    allowed_methods = ["GET"]
    allowed_origins = ["https://<STAGING_DOMAIN>"]
    max_age_seconds = 3600
  }]

  tags = { Service = "storage", Purpose = "app-assets" }
}
```

**Import existing bucket:**

```hcl
import {
  to = module.s3_existing.aws_s3_bucket.this
  id = "<existing-bucket-name>"
}
```

### Step 4: `ecr.tf`

```hcl
locals {
  # ECR repos shared across all services — one per application
  ecr_repositories = toset([
    "service-a",
    "service-b",
    "service-c",
    # ... add all application names
  ])
}

module "ecr" {
  source   = "git::ssh://git@<BITBUCKET_SSH_HOST>:7999/<BB_PROJECT>/<MODULES_REPO>.git//modules/ecr-repo?ref=v1.0.0"
  for_each = local.ecr_repositories

  name                 = "${local.org}-${each.key}"
  image_tag_mutability = "IMMUTABLE"   # MUTABLE for staging if needed
  scan_on_push         = true

  lifecycle_policy_json = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 30 tagged images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v"]
          countType     = "imageCountMoreThan"
          countNumber   = 30
        }
        action = { type = "expire" }
      },
      {
        rulePriority = 2
        description  = "Remove untagged images older than 7 days"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 7
        }
        action = { type = "expire" }
      }
    ]
  })

  tags = { Service = "storage" }
}

# Allow staging account to pull from production ECR (cross-account)
# Only needed if apps in staging pull images from a production ECR
# resource "aws_ecr_repository_policy" "cross_account" { ... }
```

**Import existing ECR repo:**

```hcl
import {
  to = module.ecr["service-a"].aws_ecr_repository.this
  id = "<org>-service-a"
}
```

### Step 5: `outputs.tf`

```hcl
output "alb_logs_bucket_name" {
  description = "S3 bucket name for ALB access logs."
  value       = module.s3_alb_logs.bucket_id
}

output "ecr_repository_urls" {
  description = "Map of ECR repository names to their URLs."
  value       = { for k, v in module.ecr : k => v.repository_url }
}
```

---

## 7. Expected file tree

```
live/staging/60-storage/
  backend.tf, versions.tf, providers.tf, locals.tf, data.tf
  s3.tf, ecr.tf, outputs.tf
  .terraform.lock.hcl
live/production/60-storage/
  (IMMUTABLE image tags enforced; may have more S3 buckets)
```

---

## 8. Code contracts

Every S3 bucket must have:
- `block_public_acls = true`, `block_public_policy = true`, `ignore_public_acls = true`, `restrict_public_buckets = true`
- SSE-KMS encryption
- Versioning enabled (except log buckets)

ECR repositories for production must use `IMMUTABLE` image tags. No exceptions without human approval.

---

## 9. Acceptance criteria

- [ ] `terraform validate` passes.
- [ ] All existing buckets from inventory are imported — plan shows no bucket creations for existing names.
- [ ] `aws s3api get-bucket-public-access-block --bucket <name>` shows all four `true`.
- [ ] ECR repos are visible in the console with the lifecycle policy applied.
- [ ] ALB access logs bucket policy allows ELB service account writes.

---

## 10. Guardrails

- Never delete an existing S3 bucket in Terraform — `terraform destroy` on a bucket with objects will fail and require manual intervention.
- Never use `force_destroy = true` on production buckets.
- Never make S3 buckets public — all access must be via IAM policies or presigned URLs.
- ECR image tag immutability prevents image tags from being overwritten — ensure CI/CD pipelines use unique tags (e.g. git SHA) before enabling.

---

## 11. Handoff note

Report: list of buckets created vs imported, list of ECR repos created vs imported, ALB log bucket name (needed by PROMPT-14), and ECR repository URLs.
