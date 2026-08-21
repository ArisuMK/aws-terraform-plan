# PROMPT-13: Data Layer — DocumentDB

## 1. Role and objective

You are a senior SRE building the `40-data` layer for both staging and production. This layer manages DocumentDB clusters, subnet groups, parameter groups, and security groups. It also handles any RDS instances if they exist. The layer imports existing clusters from the inventory before any new resources are created. Secrets (master passwords) are generated and stored in the company secrets tool — never in Terraform outputs.

---

## 2. Preconditions

- [ ] PROMPT-11 (network) complete: VPC ID and private subnet IDs available.
- [ ] PROMPT-10 (identity) complete: IAM roles available if IRSA is needed for the app accessing DocumentDB.
- [ ] Inventory has DocumentDB cluster IDs, instance IDs, subnet group names.
- [ ] Module `modules/documentdb` exists in the module library (from PROMPT-03).
- [ ] Company secrets tool is known (from questionnaire Section C).

---

## 3. Required inputs

1. DocumentDB cluster identifier(s) from the inventory (or "create new").
2. MongoDB-compatible engine version (e.g. `5.0.0`).
3. Instance class (e.g. `db.r6g.large` for production, `db.t4g.medium` for staging).
4. Number of instances (1 for staging, 2–3 for production).
5. Master username.
6. Backup retention period (days) — recommend 7 for staging, 14+ for production.
7. Maintenance window (e.g. `sun:05:00-sun:06:00`).
8. Whether deletion protection is needed (always `true` for production).

---

## 4. In scope / out of scope

**In scope:**
- DocumentDB cluster and cluster instances.
- DocumentDB subnet group (private subnets only).
- DocumentDB cluster parameter group (TLS enforced).
- Security group for DocumentDB (port 27017, ingress from EKS nodes only).
- Master password: generated with `random_password`, written to the secrets tool.
- Import blocks for existing clusters.
- RDS clusters/instances if present in inventory.

**Out of scope:**
- Application user creation inside DocumentDB (done by app bootstrapping scripts).
- ElastiCache (add a separate block in this layer if confirmed in inventory).
- Backup snapshots management (automated by AWS retention policy).

---

## 5. Reference material

- `00-standards/conventions.md` — naming: `<org>-<env>-docdb-main`.
- `00-standards/decisions.md` — ADR-004 (import blocks).
- `01-discovery/inventory.md` — data section.
- `modules/documentdb` in the module library.
- DocumentDB best practices: https://docs.aws.amazon.com/documentdb/latest/developerguide/best_practices.html

---

## 6. Step-by-step procedure

### Step 1: File structure

```
live/staging/40-data/
  backend.tf
  versions.tf
  providers.tf
  locals.tf
  data.tf
  documentdb.tf
  security_groups.tf
  secrets.tf
  outputs.tf
  .terraform.lock.hcl
```

### Step 2: `data.tf` — read network outputs

```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket      = "<ORG>-tfstate-stg-<STAGING_REGION>"
    key         = "staging/20-network/terraform.tfstate"
    region      = "<STAGING_REGION>"
    assume_role = { role_arn = "arn:aws:iam::<STAGING_ACCOUNT_ID>:role/<ORG>-stg-terraform-exec" }
  }
}

locals {
  vpc_id             = data.terraform_remote_state.network.outputs.vpc_id
  private_subnet_ids = data.terraform_remote_state.network.outputs.private_subnet_ids
}
```

### Step 3: `security_groups.tf`

```hcl
resource "aws_security_group" "documentdb" {
  name        = "${local.org}-${local.env}-sg-documentdb"
  description = "Allows MongoDB traffic from EKS nodes to DocumentDB."
  vpc_id      = local.vpc_id

  ingress {
    from_port       = 27017
    to_port         = 27017
    protocol        = "tcp"
    security_groups = [data.terraform_remote_state.network.outputs.sg_internal_id]
    description     = "MongoDB from internal VPC security group."
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound."
  }

  tags = { Name = "${local.org}-${local.env}-sg-documentdb", Service = "data" }
}
```

### Step 4: `secrets.tf` — master password

```hcl
resource "random_password" "docdb_master" {
  length           = 32
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

# Write to secrets tool — example for AWS Secrets Manager:
resource "aws_secretsmanager_secret" "docdb_master" {
  name        = "${local.org}/${local.env}/docdb/master-password"
  description = "DocumentDB master password for ${local.env_long}."

  tags = { Name = "${local.org}-${local.env}-docdb-master", Service = "data" }
}

resource "aws_secretsmanager_secret_version" "docdb_master" {
  secret_id = aws_secretsmanager_secret.docdb_master.id
  secret_string = jsonencode({
    username = "masteruser"
    password = random_password.docdb_master.result
  })
}
```

If using a different secrets tool (HashiCorp Vault, CyberArk, etc.), replace the `aws_secretsmanager_secret*` resources with the appropriate provider resources. Refer to `QUESTIONNAIRE-org-context.md` Section C.

### Step 5: `documentdb.tf`

```hcl
resource "aws_docdb_subnet_group" "main" {
  name       = "${local.org}-${local.env}-docdb-subnet-group"
  subnet_ids = local.private_subnet_ids

  tags = { Name = "${local.org}-${local.env}-docdb-subnet-group", Service = "data" }
}

resource "aws_docdb_cluster_parameter_group" "main" {
  family      = "docdb5.0"
  name        = "${local.org}-${local.env}-docdb-params"
  description = "DocumentDB parameter group — TLS enforced."

  parameter {
    name  = "tls"
    value = "enabled"
  }

  tags = { Name = "${local.org}-${local.env}-docdb-params", Service = "data" }
}

module "documentdb" {
  source = "git::ssh://git@<BITBUCKET_SSH_HOST>:7999/<BB_PROJECT>/<MODULES_REPO>.git//modules/documentdb?ref=v1.0.0"

  cluster_identifier             = "${local.org}-${local.env}-docdb-main"
  engine_version                 = "5.0.0"
  master_username                = "masteruser"
  master_password                = random_password.docdb_master.result
  instance_class                 = "db.t4g.medium"   # override per env
  instance_count                 = 1                  # 1 for staging, 2+ for production
  vpc_id                         = local.vpc_id
  subnet_ids                     = local.private_subnet_ids
  allowed_security_group_ids     = [aws_security_group.documentdb.id]
  cluster_parameter_group_name   = aws_docdb_cluster_parameter_group.main.name
  backup_retention_period        = 7
  preferred_maintenance_window   = "sun:05:00-sun:06:00"
  deletion_protection            = false   # true for production
  skip_final_snapshot            = true    # false for production
  apply_immediately              = true    # false for production

  tags = { Service = "data" }
}
```

**If importing an existing cluster:**

```hcl
import {
  to = module.documentdb.aws_docdb_cluster.this
  id = "<existing-cluster-identifier>"
}

# Also import each instance:
import {
  to = module.documentdb.aws_docdb_cluster_instance.this[0]
  id = "<existing-instance-identifier>"
}
```

Run `terraform plan -generate-config-out=generated.tf` to produce a skeleton matching the live cluster, then reconcile with the module interface.

### Step 6: `outputs.tf`

```hcl
output "docdb_cluster_endpoint" {
  description = "DocumentDB writer endpoint."
  value       = module.documentdb.cluster_endpoint
}

output "docdb_cluster_reader_endpoint" {
  description = "DocumentDB reader endpoint."
  value       = module.documentdb.cluster_reader_endpoint
}

output "docdb_cluster_port" {
  description = "DocumentDB port (27017)."
  value       = module.documentdb.cluster_port
}

output "docdb_security_group_id" {
  description = "Security group ID for DocumentDB."
  value       = aws_security_group.documentdb.id
}

output "docdb_master_secret_arn" {
  description = "ARN of the Secrets Manager secret containing the master password."
  value       = aws_secretsmanager_secret.docdb_master.arn
  sensitive   = true
}
```

---

## 7. Expected file tree

```
live/staging/40-data/
  backend.tf, versions.tf, providers.tf, locals.tf, data.tf
  documentdb.tf, security_groups.tf, secrets.tf, outputs.tf
  .terraform.lock.hcl
live/production/40-data/
  (same — production values: 2 instances, deletion_protection=true, skip_final_snapshot=false)
```

---

## 8. Code contracts

Production DocumentDB clusters MUST have:
- `deletion_protection = true`
- `skip_final_snapshot = false`
- `backup_retention_period >= 14`
- `instance_count >= 2`
- TLS enforced via parameter group

The master password MUST NOT appear in `outputs.tf`. Only the secret ARN is exposed.

---

## 9. Acceptance criteria

- [ ] `terraform validate` passes.
- [ ] Plan shows no deletions of existing cluster or instances.
- [ ] After apply: cluster status is `available`.
- [ ] TLS parameter is `enabled` in the parameter group.
- [ ] Master password is stored in the secrets tool and NOT in Terraform state outputs.
- [ ] Connection test: `mongosh --tls <endpoint>:27017` returns a prompt.

---

## 10. Guardrails

- Never output the master password value — only the secret ARN.
- Never set `skip_final_snapshot = true` in production.
- Never set `deletion_protection = false` in production.
- `apply_immediately = true` is acceptable for staging only — production changes must use the maintenance window.
- Do not modify `engine_version` after creation without a documented major version upgrade plan.

---

## 11. Handoff note

Report: cluster endpoint, reader endpoint, instance count, security group ID, and secret ARN. Flag if any existing cluster had configuration drift from the desired state (e.g. TLS was not enabled — it may require a restart).
