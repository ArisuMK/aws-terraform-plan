# Import Playbook

This playbook describes the exact process for importing existing AWS resources into Terraform state without destroying or recreating them. Read this before any import PR is opened.

---

## Principle

> **Never use `terraform import` CLI against shared state. Always use declarative `import {}` blocks.**

The `terraform import` CLI command requires an interactive shell with write access to the state file. In an Atlantis workflow, there is no interactive shell. The CLI also leaves no PR record of what was imported — the team cannot review or audit it.

Declarative `import {}` blocks (Terraform 1.5+) appear in the plan output, get reviewed in the PR as a normal diff, and are applied by Atlantis like any other change. Every import is documented in git history forever.

---

## The four-step import workflow

### Step 1: Discover the resource's import ID

Every AWS resource type has a specific import ID format. Find the correct format in the [Terraform AWS provider documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) under the "Import" section of each resource.

Common formats:

| Resource type | Import ID format | Example |
|---|---|---|
| `aws_vpc` | VPC ID | `vpc-0abc123def456789` |
| `aws_subnet` | Subnet ID | `subnet-0abc123def456789` |
| `aws_security_group` | Security group ID | `sg-0abc123def456789` |
| `aws_iam_role` | Role name | `my-role-name` |
| `aws_iam_policy` | Policy ARN | `arn:aws:iam::123456789012:policy/MyPolicy` |
| `aws_s3_bucket` | Bucket name | `my-bucket-name` |
| `aws_ecr_repository` | Repository name | `my-repo-name` |
| `aws_docdb_cluster` | Cluster identifier | `my-docdb-cluster` |
| `aws_docdb_cluster_instance` | Instance identifier | `my-docdb-instance-1` |
| `aws_route53_zone` | Zone ID (without `/hostedzone/`) | `Z1234567890ABC` |
| `aws_acm_certificate` | Certificate ARN | `arn:aws:acm::...` |
| `aws_lb` | ALB ARN | `arn:aws:elasticloadbalancing::...` |
| `aws_eks_cluster` | Cluster name | `my-cluster-name` |
| `aws_iam_openid_connect_provider` | Provider ARN | `arn:aws:iam::...` |
| `aws_cloudtrail` | Trail name | `my-trail` |
| `aws_guardduty_detector` | Detector ID | `abc123def456...` |

When in doubt: `terraform import` CLI on a dev machine (throwaway state) shows the exact ID — then translate it to an `import {}` block without touching shared state.

### Step 2: Generate a config skeleton

Use `terraform plan -generate-config-out=generated.tf` to produce a skeleton HCL config from the live resource. This requires the `import {}` block to already be present.

```bash
# In the target layer directory
cd live/staging/20-network

# Add the import block to a temporary file
cat >> imports.tf << 'EOF'
import {
  to = module.vpc.aws_vpc.this[0]
  id = "vpc-0abc123def456789"
}
EOF

# Generate the skeleton
terraform init -backend=false
terraform plan -generate-config-out=generated.tf
```

The `generated.tf` file contains a raw resource block matching the live resource. Do not commit this file as-is — it is a starting point only.

### Step 3: Refactor into the correct module call

The generated skeleton uses raw `resource "aws_vpc" "this"` blocks. Refactor into the module structure defined in the infra repo:

**Before (generated skeleton):**
```hcl
resource "aws_vpc" "this" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  instance_tenancy     = "default"
  # ... 40 more attributes Terraform detected
}
```

**After (module call matching your conventions):**
```hcl
module "vpc" {
  source = "git::ssh://git@<BITBUCKET_SSH_HOST>:7999/<BB_PROJECT>/<MODULES_REPO>.git//modules/vpc?ref=v1.0.0"

  name                 = "<org>-stg-vpc"
  cidr_block           = "10.0.0.0/16"
  azs                  = ["us-east-1a", "us-east-1b"]
  private_subnet_cidrs = ["10.0.0.0/24", "10.0.1.0/24"]
  public_subnet_cidrs  = ["10.0.2.0/24", "10.0.3.0/24"]
  enable_nat_gateway   = true
  single_nat_gateway   = true
}
```

The `import {}` block target (`to`) must be updated to point to the module path:

```hcl
import {
  to = module.vpc.aws_vpc.this[0]
  id = "vpc-0abc123def456789"
}
```

### Step 4: Open an import PR through Atlantis

1. Create a branch: `git checkout -b import/staging-vpc`.
2. Add the `import {}` block(s) and the resource/module config.
3. Push and open a PR in Bitbucket.
4. Atlantis auto-plans. The plan output shows `# module.vpc.aws_vpc.this[0] will be imported`.
5. Review the plan carefully:
   - Every resource listed as "will be imported" is correct and expected.
   - There are NO resources listed as "will be destroyed."
   - If the config differs from the live resource, there will be "will be updated in-place" lines — review each one. In-place updates to existing resources are usually safe; check that no critical attribute (CIDR block, master password) is being changed.
6. Get the PR approved by a reviewer.
7. Type `atlantis apply` in the PR comment.
8. Verify the apply succeeded. The plan should show `Apply complete! Resources: X imported, Y added.`

### Step 5: Clean-up PR

After the import apply succeeds:

1. Create a follow-up branch: `git checkout -b cleanup/remove-import-blocks`.
2. Delete all `import {}` blocks from the layer.
3. Do NOT delete the resource/module configs — only the `import {}` blocks.
4. Push and open a PR.
5. Atlantis plans. The plan must show: `No changes. Your infrastructure matches the configuration.`
6. If there are any unexpected changes, stop, investigate, and do not apply until the discrepancy is resolved.
7. Merge the cleanup PR.

---

## Handling difficult imports

### Multiple resources to import (e.g. all subnets in a VPC)

Add multiple `import {}` blocks in the same PR. They are all planned and applied atomically:

```hcl
import { to = module.vpc.aws_subnet.private[0]; id = "subnet-aaa" }
import { to = module.vpc.aws_subnet.private[1]; id = "subnet-bbb" }
import { to = module.vpc.aws_subnet.public[0];  id = "subnet-ccc" }
import { to = module.vpc.aws_subnet.public[1];  id = "subnet-ddd" }
```

### Resources inside for_each modules

When the module uses `for_each`, the import target uses the key:

```hcl
import {
  to = module.ecr["service-a"].aws_ecr_repository.this
  id = "org-service-a"
}
```

### Resources with naming that doesn't match conventions

If an existing resource is named `my_old_name_prod` but conventions require `<org>-prd-service`, do not rename it in the import PR. Import it as-is first (clean plan, no changes), then do a separate PR to rename/recreate it if renaming is safe (renaming an S3 bucket destroys and recreates it — never do this for buckets with data).

Flag all non-compliant resource names in the import PR description.

### The plan shows unexpected in-place changes

This means the existing resource has a configuration that differs from what Terraform wants to set. Common causes:

| Symptom | Likely cause | Action |
|---|---|---|
| `tags` will be updated | Resource has tags not in the config | Add the extra tags to the Terraform config, or accept the tag change |
| `description` will be updated | Empty description in config, non-empty in AWS | Add the actual description to the config |
| `cidr_blocks` ordering | AWS returns CIDRs in different order | Use `toset()` if supported, or reorder in config |
| Boolean attribute differs | AWS default vs Terraform default | Explicitly set the attribute in config to match the current value |

Always resolve unexpected changes by updating the Terraform config to match the current live state — not by modifying AWS to match Terraform. Then re-plan. The import PR must end with no unexpected changes (only "will be imported" and at most minor in-place updates that you have intentionally approved).

### The plan shows a destroy

**STOP.** A destroy in an import plan is always wrong. Do not apply.

Diagnose:
1. Is the `to` address correct? A wrong module path causes Terraform to plan destroying the old address and creating a new one.
2. Is there a name conflict? A pre-existing resource block for the same address causes a "replace" (destroy + create).
3. Did `terraform init` pick up the wrong provider version?

Resolve the issue before applying anything.

---

## State file backups

Before any import apply, verify the state file exists in S3:

```bash
aws s3 ls s3://<ORG>-tfstate-<env>-<region>/<env>/<layer>/
```

S3 versioning is enabled on state buckets, so every state write is versioned. In an emergency, restore a previous state version:

```bash
# List versions
aws s3api list-object-versions --bucket <ORG>-tfstate-<env>-<region> --prefix <env>/<layer>/terraform.tfstate

# Restore (copy a specific version to latest)
aws s3api copy-object \
  --bucket <ORG>-tfstate-<env>-<region> \
  --copy-source "<ORG>-tfstate-<env>-<region>/<env>/<layer>/terraform.tfstate?versionId=<version-id>" \
  --key "<env>/<layer>/terraform.tfstate"
```

Only do this during an incident with explicit human approval — restoring state can cause drift.

---

## Import checklist (use before every import PR)

- [ ] The `import {}` block uses the correct ID format for this resource type.
- [ ] The target (`to`) path matches the actual module/resource address.
- [ ] `terraform plan -generate-config-out=generated.tf` was run and reviewed locally before the PR.
- [ ] The config matches the live resource well enough that the plan shows no destroy operations.
- [ ] The PR description lists every resource being imported and its AWS ID.
- [ ] A second engineer has reviewed the plan output in the PR comment.
- [ ] The clean-up PR (removing `import {}` blocks) is planned immediately after this one.
