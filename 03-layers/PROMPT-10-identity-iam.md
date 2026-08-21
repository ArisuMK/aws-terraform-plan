# PROMPT-10: Identity and IAM Layer

## 1. Role and objective

You are a senior SRE building the `10-identity` layer for both staging and production AWS accounts. This layer manages all IAM roles, policies, OIDC providers, and GitHub Actions OIDC federation (if used). It is the first infrastructure layer applied in each account and is a dependency for almost every subsequent layer — the EKS IRSA roles, the ALB controller role, the Terraform execution role trust policies, etc.

---

## 2. Preconditions

- [ ] PROMPT-01 complete: exec roles and state buckets exist.
- [ ] PROMPT-02 complete: infra repo scaffolded.
- [ ] PROMPT-04 Part A complete **or in progress**: Atlantis IAM role `<ORG>-uat-atlantis` (EC2 instance profile) should be defined here; EKS IRSA pieces wait for Part B.
- [ ] `PROMPT-00` inventory complete: existing IAM roles listed. Import candidates identified.
- [ ] `fill-in-the-blanks.local.md` has: `<ORG>`, `<STAGING_ACCOUNT_ID>`, `<PRODUCTION_ACCOUNT_ID>`. OIDC fields only if EKS is already in this account.

Prefer Atlantis for applies once EC2 Atlantis is healthy; until then use the interim local-apply exception (exec role), documented in git/PRs.

---

## 3. Required inputs

1. `<ORG>`, `<STAGING_ACCOUNT_ID>`, `<PRODUCTION_ACCOUNT_ID>`.
2. Atlantis role: EC2 instance-profile trust now (from PROMPT-04 Part A). `<STAGING_OIDC_URL>` / `<STAGING_OIDC_ARN>` only when migrating to EKS (Part B).
3. List of existing IAM roles from the inventory that must be imported (not deleted and recreated).
4. Does the company use GitHub Actions for application CI/CD? If yes, provide the GitHub org name — an OIDC provider for GitHub will be added.
5. Does the company use AWS SSO (IAM Identity Center)? If yes, are SSO assignments managed here or by a separate team?
6. List of application IRSA roles that are already known — defer if the cluster is not in this account yet.

---

## 4. In scope / out of scope

**In scope:**
- Terraform execution roles (if not already handled by PROMPT-01 — confirm).
- Atlantis IAM role (`<ORG>-uat-atlantis`) — **EC2 instance profile trust** for interim hosting; switch or dual-trust OIDC when Part B runs.
- OIDC providers: EKS OIDC (for IRSA) **when the cluster exists in this account**; optionally GitHub Actions OIDC.
- AWS Load Balancer Controller / External DNS / EBS CSI IRSA roles — defer until EKS is in this account unless roles already exist here to import.
- Application IRSA roles for services running on EKS (import existing ones) — same deferral rule.
- Customer-managed IAM policies (audit and import from inventory).
- KMS key policies (if KMS keys are managed in this layer).

**Out of scope:**
- IAM users (avoid creating IAM users; use IRSA and SSO instead).
- AWS Organizations, SCPs (those are in a separate org layer if needed).
- EKS `aws-auth` ConfigMap or access entries (managed in `80-eks` layer).

---

## 5. Reference material

- `00-standards/conventions.md` — sections 2 (naming: `<org>-<env>-<service>`), 3 (tagging), 5 (locals).
- `00-standards/decisions.md` — ADR-004 (import blocks).
- `01-discovery/inventory.md` — IAM section.
- AWS IAM best practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html

---

## 6. Step-by-step procedure

### Step 1: Standard file structure for 10-identity

```
live/staging/10-identity/
  backend.tf
  versions.tf
  providers.tf
  locals.tf
  data.tf
  oidc.tf              # OIDC providers (EKS, GitHub)
  irsa.tf              # All IRSA roles
  atlantis.tf          # Atlantis IAM role (from PROMPT-04)
  policies.tf          # Customer-managed policies
  outputs.tf
  .terraform.lock.hcl
```

### Step 2: `backend.tf`

```hcl
terraform {
  backend "s3" {
    bucket       = "<ORG>-tfstate-uat-<STAGING_REGION>"
    key          = "staging/10-identity/terraform.tfstate"
    region       = "<STAGING_REGION>"
    encrypt      = true
    kms_key_id   = "alias/<ORG>-uat-tfstate"
    use_lockfile = true
    assume_role  = {
      role_arn = "arn:aws:iam::<STAGING_ACCOUNT_ID>:role/<ORG>-uat-terraform-exec"
    }
  }
  required_version = "~> 1.11"
}
```

### Step 3: `providers.tf`

```hcl
provider "aws" {
  region = local.region

  assume_role {
    role_arn     = "arn:aws:iam::${local.account_id}:role/<ORG>-${local.env}-terraform-exec"
    session_name = "atlantis-10-identity"
  }

  default_tags {
    tags = local.default_tags
  }
}
```

### Step 4: `locals.tf`

```hcl
locals {
  org        = "<ORG>"
  env      = "uat"
  env_long   = "staging"
  layer      = "10-identity"
  region     = data.aws_region.current.name
  account_id = data.aws_caller_identity.current.account_id

  # Atlantis IRSA subject
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

### Step 5: `data.tf`

```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# Reference the EKS OIDC provider (created in 00-bootstrap or pre-existing)
data "aws_iam_openid_connect_provider" "eks" {
  url = "https://<STAGING_OIDC_URL>"
}
```

### Step 6: `oidc.tf` — GitHub Actions OIDC (if applicable)

```hcl
# Only include if GitHub Actions is used for application CI/CD
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]

  tags = {
    Name    = "${local.org}-${local.env}-github-oidc"
    Service = "identity"
  }
}
```

### Step 7: `irsa.tf` — standard EKS add-on IRSA roles

Use the `modules/iam-role` module from the module library for each IRSA role.

**AWS Load Balancer Controller:**

```hcl
data "aws_iam_policy_document" "alb_controller_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [data.aws_iam_openid_connect_provider.eks.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "${data.aws_iam_openid_connect_provider.eks.url}:sub"
      values   = ["system:serviceaccount:kube-system:aws-load-balancer-controller"]
    }
    condition {
      test     = "StringEquals"
      variable = "${data.aws_iam_openid_connect_provider.eks.url}:aud"
      values   = ["sts.amazonaws.com"]
    }
  }
}

module "irsa_alb_controller" {
  source = "git::ssh://git@<BITBUCKET_SSH_HOST>:7999/<BB_PROJECT>/<MODULES_REPO>.git//modules/iam-role?ref=v1.0.0"

  name              = "${local.org}-${local.env}-irsa-alb-controller"
  trust_policy_json = data.aws_iam_policy_document.alb_controller_trust.json
  policy_arns       = [aws_iam_policy.alb_controller.arn]

  tags = { Service = "identity" }
}

resource "aws_iam_policy" "alb_controller" {
  name        = "${local.org}-${local.env}-alb-controller"
  description = "Policy for the AWS Load Balancer Controller."
  # Download the latest policy from:
  # https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
  policy = file("${path.module}/policies/alb-controller-policy.json")
}
```

Create `live/staging/10-identity/policies/alb-controller-policy.json` from the official source above.

**EBS CSI Driver** and **External DNS** follow the same pattern. Use the official policy documents from their respective AWS and Kubernetes repos.

### Step 8: `policies.tf` — import existing customer-managed policies

For each customer-managed IAM policy found in the inventory, add an import block and resource:

```hcl
import {
  to = aws_iam_policy.example
  id = "arn:aws:iam::<STAGING_ACCOUNT_ID>:policy/<existing-policy-name>"
}

resource "aws_iam_policy" "example" {
  name        = "<existing-policy-name>"
  description = "Imported from existing account."
  policy      = data.aws_iam_policy_document.example.json
}
```

### Step 9: `outputs.tf`

```hcl
output "eks_oidc_provider_arn" {
  description = "ARN of the EKS OIDC provider."
  value       = data.aws_iam_openid_connect_provider.eks.arn
}

output "irsa_alb_controller_role_arn" {
  description = "ARN of the ALB Controller IRSA role."
  value       = module.irsa_alb_controller.role_arn
}

output "atlantis_role_arn" {
  description = "ARN of the Atlantis IRSA role."
  value       = aws_iam_role.atlantis.arn
}
```

Expose all IRSA role ARNs as outputs — they are consumed by the `80-eks` layer for configuring EKS add-ons.

### Step 10: Lock file

```bash
cd live/staging/10-identity
terraform init
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64
```

Commit `.terraform.lock.hcl`. Repeat for production.

---

## 7. Expected file tree

```
live/
  staging/10-identity/
    backend.tf, versions.tf, providers.tf, locals.tf, data.tf
    oidc.tf, irsa.tf, atlantis.tf, policies.tf, outputs.tf
    policies/
      alb-controller-policy.json
      ebs-csi-driver-policy.json
      external-dns-policy.json
    .terraform.lock.hcl
  production/10-identity/
    (same structure)
```

---

## 8. Code contracts

Every IRSA trust policy must include BOTH conditions:
- `<oidc-url>:sub` = `system:serviceaccount:<namespace>:<sa-name>`
- `<oidc-url>:aud` = `sts.amazonaws.com`

Omitting the `aud` condition is a security vulnerability.

---

## 9. Acceptance criteria

- [ ] `terraform validate` passes.
- [ ] `terraform plan` shows only expected imports and creations — no unexpected deletions.
- [ ] All existing IAM roles found in the inventory are either imported or explicitly documented as "left unmanaged, reason: <reason>".
- [ ] `terraform apply` succeeds (via Atlantis) and plan is clean on a second run.
- [ ] `outputs.tf` exposes at minimum: EKS OIDC ARN, ALB controller role ARN, Atlantis role ARN.

---

## 10. Guardrails

- Never create IAM users for application workloads — only IRSA.
- Never attach `AdministratorAccess` to any IRSA role.
- Never commit IAM policy JSON documents containing sensitive data.
- Do not delete any IAM role from the inventory without explicit human confirmation — some may be used by workloads the team doesn't yet know about.
- Import existing roles before writing new resources that conflict with their names.

---

## 11. Handoff note

When complete, report:
1. Count of IAM roles created vs imported.
2. Count of customer-managed policies created vs imported.
3. List of IRSA roles created and their associated Kubernetes service accounts.
4. Any IAM roles from the inventory that were intentionally left unmanaged (and why).
5. Outputs exposed that downstream layers will consume.
