# PROMPT-17: Observability and Audit Layer

## 1. Role and objective

You are a senior SRE building the `70-observability` layer for both staging and production. This layer manages CloudTrail (account-wide audit logging), AWS Config (resource configuration compliance), GuardDuty (threat detection), CloudWatch log groups with retention policies, and AWS Budgets (cost guardrails). These resources are foundational security and compliance controls that should be enabled from day one.

---

## 2. Preconditions

- [ ] PROMPT-60-storage is complete (or at minimum the ALB logs bucket exists): a logs S3 bucket is available for CloudTrail.
- [ ] PROMPT-10 (identity) complete: IAM roles available.
- [ ] `fill-in-the-blanks.local.md` has org name and account IDs.

---

## 3. Required inputs

1. CloudTrail: should it be a multi-region trail or single-region? (Recommend: multi-region).
2. CloudWatch Logs retention period for CloudTrail logs (recommend 365 days for production, 90 for staging).
3. GuardDuty: enable in staging? (Recommend: yes, even if findings are lower priority).
4. AWS Config: list of rules to enable immediately (the AI will provide a recommended starter set).
5. Budget alerts: monthly budget threshold in USD for staging and production. Who receives the alert email?
6. Does the company use a SIEM or log aggregation platform? If so, CloudTrail logs may need to be forwarded via Kinesis or EventBridge — confirm.

---

## 4. In scope / out of scope

**In scope:**
- CloudTrail multi-region trail with S3 + CloudWatch Logs.
- KMS encryption for CloudTrail S3 objects.
- AWS Config recorder and delivery channel.
- Recommended AWS Config managed rules (starter set).
- GuardDuty detector.
- CloudWatch log groups with retention for application namespaces.
- AWS Budgets with email alert.
- SNS topic for alarm notifications.

**Out of scope:**
- Datadog integration (add when Datadog is confirmed in the project).
- CloudWatch dashboards and alarms for application metrics (those go in app teams' IaC).
- Security Hub (add in a follow-up if required by compliance).
- AWS Macie (add if PII data classification is required).

---

## 5. Reference material

- `00-standards/conventions.md` — naming, tagging.
- `01-discovery/inventory.md` — CloudTrail section (existing trails).
- AWS Security best practices: https://docs.aws.amazon.com/security/

---

## 6. Step-by-step procedure

### Step 1: File structure

```
live/staging/70-observability/
  backend.tf, versions.tf, providers.tf, locals.tf, data.tf
  cloudtrail.tf
  config.tf
  guardduty.tf
  loggroups.tf
  budgets.tf
  sns.tf
  outputs.tf
  .terraform.lock.hcl
```

### Step 2: `data.tf` — upstream state

```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

data "terraform_remote_state" "storage" {
  backend = "s3"
  config = {
    bucket      = "<ORG>-tfstate-stg-<STAGING_REGION>"
    key         = "staging/60-storage/terraform.tfstate"
    region      = "<STAGING_REGION>"
    assume_role = { role_arn = "arn:aws:iam::<STAGING_ACCOUNT_ID>:role/<ORG>-stg-terraform-exec" }
  }
}
```

### Step 3: `cloudtrail.tf`

```hcl
resource "aws_cloudwatch_log_group" "cloudtrail" {
  name              = "/aws/cloudtrail/${local.org}-${local.env}"
  retention_in_days = 90   # 365 for production
  kms_key_id        = data.terraform_remote_state.bootstrap.outputs.tfstate_kms_key_arn

  tags = { Name = "${local.org}-${local.env}-cloudtrail-logs", Service = "observability" }
}

resource "aws_iam_role" "cloudtrail_cw" {
  name = "${local.org}-${local.env}-cloudtrail-cw"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "cloudtrail.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "cloudtrail_cw" {
  name = "cloudwatch-logs"
  role = aws_iam_role.cloudtrail_cw.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["logs:CreateLogStream", "logs:PutLogEvents"]
      Resource = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
    }]
  })
}

resource "aws_s3_bucket" "cloudtrail" {
  bucket = "${local.org}-${local.env}-cloudtrail-${local.region}"

  tags = { Name = "${local.org}-${local.env}-cloudtrail", Service = "observability" }
}

resource "aws_s3_bucket_policy" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AWSCloudTrailAclCheck"
        Effect = "Allow"
        Principal = { Service = "cloudtrail.amazonaws.com" }
        Action   = "s3:GetBucketAcl"
        Resource = aws_s3_bucket.cloudtrail.arn
      },
      {
        Sid    = "AWSCloudTrailWrite"
        Effect = "Allow"
        Principal = { Service = "cloudtrail.amazonaws.com" }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.cloudtrail.arn}/AWSLogs/${local.account_id}/*"
        Condition = {
          StringEquals = { "s3:x-amz-acl" = "bucket-owner-full-control" }
        }
      }
    ]
  })
}

resource "aws_cloudtrail" "main" {
  name                          = "${local.org}-${local.env}-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.bucket
  cloud_watch_logs_group_arn    = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn     = aws_iam_role.cloudtrail_cw.arn
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true

  tags = { Name = "${local.org}-${local.env}-trail", Service = "observability" }
}
```

**Import existing trail:**
```hcl
import {
  to = aws_cloudtrail.main
  id = "<existing-trail-name>"
}
```

### Step 4: `guardduty.tf`

```hcl
resource "aws_guardduty_detector" "main" {
  enable = true

  datasources {
    s3_logs { enable = true }
    kubernetes {
      audit_logs { enable = true }
    }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes { enable = true }
      }
    }
  }

  tags = { Name = "${local.org}-${local.env}-guardduty", Service = "observability" }
}
```

**Import existing detector:**
```hcl
import {
  to = aws_guardduty_detector.main
  id = "<existing-detector-id>"
}
```

### Step 5: `config.tf` — AWS Config

```hcl
resource "aws_config_configuration_recorder" "main" {
  name     = "${local.org}-${local.env}-config-recorder"
  role_arn = aws_iam_role.config.arn

  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}

resource "aws_config_delivery_channel" "main" {
  name           = "${local.org}-${local.env}-config-channel"
  s3_bucket_name = aws_s3_bucket.cloudtrail.bucket

  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_configuration_recorder_status" "main" {
  name       = aws_config_configuration_recorder.main.name
  is_enabled = true
  depends_on = [aws_config_delivery_channel.main]
}

# IAM role for Config
resource "aws_iam_role" "config" {
  name = "${local.org}-${local.env}-aws-config"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "config.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
  managed_policy_arns = ["arn:aws:iam::aws:policy/service-role/AWS_ConfigRole"]
}

# Starter Config rules — expand as compliance requirements emerge
resource "aws_config_config_rule" "s3_bucket_public_write_prohibited" {
  name = "s3-bucket-public-write-prohibited"
  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_WRITE_PROHIBITED"
  }
  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_config_rule" "root_mfa_enabled" {
  name = "root-account-mfa-enabled"
  source {
    owner             = "AWS"
    source_identifier = "ROOT_ACCOUNT_MFA_ENABLED"
  }
  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_config_rule" "encrypted_volumes" {
  name = "encrypted-volumes"
  source {
    owner             = "AWS"
    source_identifier = "ENCRYPTED_VOLUMES"
  }
  depends_on = [aws_config_configuration_recorder.main]
}
```

### Step 6: `budgets.tf`

```hcl
resource "aws_budgets_budget" "monthly" {
  name         = "${local.org}-${local.env}-monthly-budget"
  budget_type  = "COST"
  limit_amount = "<MONTHLY_BUDGET_USD>"   # ask human
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["<BUDGET_ALERT_EMAIL>"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["<BUDGET_ALERT_EMAIL>"]
  }
}
```

### Step 7: `loggroups.tf` — application log groups

```hcl
locals {
  application_log_groups = [
    "/aws/eks/${local.org}-${local.env}-eks/application",
    "/aws/eks/${local.org}-${local.env}-eks/dataplane",
    "/aws/eks/${local.org}-${local.env}-eks/host",
    "/aws/eks/${local.org}-${local.env}-eks/authenticator",
    "/aws/eks/${local.org}-${local.env}-eks/controlplane",
  ]
}

resource "aws_cloudwatch_log_group" "app" {
  for_each = toset(local.application_log_groups)

  name              = each.value
  retention_in_days = 30   # 90 for production

  tags = { Service = "observability" }
}
```

### Step 8: `outputs.tf`

```hcl
output "cloudtrail_name" {
  description = "CloudTrail trail name."
  value       = aws_cloudtrail.main.name
}

output "guardduty_detector_id" {
  description = "GuardDuty detector ID."
  value       = aws_guardduty_detector.main.id
}

output "cloudtrail_bucket_name" {
  description = "S3 bucket for CloudTrail logs."
  value       = aws_s3_bucket.cloudtrail.bucket
}
```

---

## 7. Expected file tree

```
live/staging/70-observability/
  backend.tf, versions.tf, providers.tf, locals.tf, data.tf
  cloudtrail.tf, config.tf, guardduty.tf, loggroups.tf, budgets.tf, outputs.tf
  .terraform.lock.hcl
live/production/70-observability/
  (log retention 365 days for cloudtrail, 90 for apps; longer budget threshold)
```

---

## 8. Code contracts

- CloudTrail log file validation MUST be enabled (`enable_log_file_validation = true`).
- Multi-region trail (`is_multi_region_trail = true`) is required.
- All CloudTrail S3 objects MUST be encrypted.
- GuardDuty MUST be enabled — if it already exists and is managed by a security team, import it (do not disable and recreate).

---

## 9. Acceptance criteria

- [ ] `terraform validate` passes.
- [ ] CloudTrail trail is active: `aws cloudtrail get-trail-status --name <trail-name>` shows `IsLogging: true`.
- [ ] GuardDuty detector is enabled.
- [ ] AWS Config recorder is running.
- [ ] Budget alert email is confirmed by the human.
- [ ] `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=LookupEvents` returns recent events.

---

## 10. Guardrails

- Never disable an existing CloudTrail trail that a security team may be monitoring.
- Do not import and modify GuardDuty if a security team manages it separately — use a data source instead.
- Budget alerts are non-blocking — they do not stop Terraform applies, only send emails.
- Do not set `retention_in_days = 0` (infinite retention) — it incurs unbounded cost.

---

## 11. Handoff note

Report: CloudTrail trail name and S3 bucket, GuardDuty detector ID, Config recorder status, budget name and threshold, and any existing resources imported.
