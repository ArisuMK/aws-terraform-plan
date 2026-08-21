# PROMPT-11: Network and VPC Layer

## 1. Role and objective

You are a senior SRE building the `20-network` layer for both staging and production. This layer manages the VPC, subnets, NAT gateways, internet gateway, security groups, VPC endpoints, and route tables. If a VPC already exists (brownfield), you will import it. If it does not exist, you will create it from scratch using the `modules/vpc` module. Downstream layers (30-dns, 40-data, 50-compute, 80-eks) all depend on the outputs of this layer.

---

## 2. Preconditions

- [ ] PROMPT-10 (identity) is complete for the target environment.
- [ ] `01-discovery/inventory.md` has the VPC section filled in.
- [ ] `fill-in-the-blanks.local.md` has: VPC IDs (if existing), CIDR blocks, subnet IDs, AZs.
- [ ] Module `modules/vpc` exists in the module library (from PROMPT-03).

---

## 3. Required inputs

1. Existing VPC ID(s) or "none — create new."
2. Desired VPC CIDR block (e.g. `10.0.0.0/16` for staging, `10.1.0.0/16` for production).
3. Number of AZs (2 or 3 recommended).
4. Private subnet CIDRs (one per AZ).
5. Public subnet CIDRs (one per AZ).
6. Should staging use a single NAT gateway (cost saving) or one per AZ?
7. Should VPC endpoints be created for S3 and DynamoDB (Gateway type, free) and/or SSM/ECR/STS (Interface type, billed)?
8. Any security groups from the inventory that must be imported.

---

## 4. In scope / out of scope

**In scope:**
- VPC with DNS hostnames and DNS support enabled.
- Public and private subnets with correct tags for EKS (`kubernetes.io/role/elb`, `kubernetes.io/role/internal-elb`, `kubernetes.io/cluster/<cluster-name>`).
- Internet Gateway, NAT Gateways, route tables.
- VPC endpoints: S3 Gateway (always), DynamoDB Gateway (always), SSM/EC2Messages/SSMMessages Interface (recommended), ECR API/DKR Interface (if ECR is heavily used from EKS nodes).
- Default security group: all ingress/egress rules removed (hardened).
- Security groups for common use: `sg-internal` (within VPC), `sg-https-public` (0.0.0.0/0:443).

**Out of scope:**
- VPN Gateway, Direct Connect (add if needed per questionnaire Section F).
- Transit Gateway peering (add if needed).
- VPC peering (add only if confirmed).
- ALB security groups (managed in 50-compute layer).

---

## 5. Reference material

- `00-standards/conventions.md` — naming: `<org>-<env>-vpc`, `<org>-<env>-subnet-private-<az>`.
- `00-standards/decisions.md` — ADR-004 (import blocks).
- `01-discovery/inventory.md` — networking section.
- EKS subnet tagging requirements: https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html

---

## 6. Step-by-step procedure

### Step 1: File structure

```
live/staging/20-network/
  backend.tf
  versions.tf
  providers.tf
  locals.tf
  data.tf
  vpc.tf
  endpoints.tf
  security_groups.tf
  outputs.tf
  .terraform.lock.hcl
```

### Step 2: `locals.tf`

```hcl
locals {
  org        = "<ORG>"
  env      = "uat"
  env_long   = "staging"
  layer      = "20-network"
  region     = data.aws_region.current.name
  account_id = data.aws_caller_identity.current.account_id

  # EKS cluster name — needed for subnet tags
  eks_cluster_name = "<ORG>-uat-eks"

  azs = ["<REGION>a", "<REGION>b", "<REGION>c"]  # adjust to actual AZs

  vpc_cidr             = "<STAGING_VPC_CIDR>"
  private_subnet_cidrs = ["<CIDR_A>", "<CIDR_B>", "<CIDR_C>"]
  public_subnet_cidrs  = ["<CIDR_A>", "<CIDR_B>", "<CIDR_C>"]

  default_tags = {
    Organization = local.org
    Environment  = local.env_long
    ManagedBy    = "terraform"
    Repository   = "<INFRA_REPO>"
    Layer        = local.layer
  }
}
```

### Step 3: `vpc.tf`

**Case A — Create new VPC:**

```hcl
module "vpc" {
  source = "git::ssh://git@<BITBUCKET_SSH_HOST>:7999/<BB_PROJECT>/<MODULES_REPO>.git//modules/vpc?ref=v1.0.0"

  name                 = "${local.org}-${local.env}-vpc"
  cidr_block           = local.vpc_cidr
  azs                  = local.azs
  private_subnet_cidrs = local.private_subnet_cidrs
  public_subnet_cidrs  = local.public_subnet_cidrs
  enable_nat_gateway   = true
  single_nat_gateway   = true   # single NAT for staging; false for production

  tags = {
    Service = "network"
    # EKS shared cluster tags (set on VPC and subnets for EKS auto-discovery)
    "kubernetes.io/cluster/${local.eks_cluster_name}" = "shared"
  }
}
```

Private subnets need the tag `kubernetes.io/role/internal-elb = 1` and public subnets need `kubernetes.io/role/elb = 1`. Verify the `modules/vpc` wrapper handles this or add them directly via the `terraform-aws-modules/vpc` module parameters `private_subnet_tags` and `public_subnet_tags`.

**Case B — Import existing VPC (brownfield):**

```hcl
import {
  to = module.vpc.aws_vpc.this[0]
  id = "<STAGING_VPC_ID>"
}

# Also import each subnet, IGW, NAT gateway, route tables, etc.
# Get the IDs from the inventory. Example:
import {
  to = module.vpc.aws_subnet.private[0]
  id = "<PRIVATE_SUBNET_ID_A>"
}
# ... repeat for each resource
```

Use `terraform plan -generate-config-out=generated.tf` locally to produce skeleton configs, then refactor into module calls.

### Step 4: `endpoints.tf`

```hcl
# S3 Gateway endpoint (free, improves S3 performance from EKS nodes)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.${local.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = module.vpc.private_route_table_ids

  tags = {
    Name    = "${local.org}-${local.env}-vpce-s3"
    Service = "network"
  }
}

# DynamoDB Gateway endpoint (free)
resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.${local.region}.dynamodb"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = module.vpc.private_route_table_ids

  tags = {
    Name    = "${local.org}-${local.env}-vpce-dynamodb"
    Service = "network"
  }
}

# SSM Interface endpoints (required for EKS node SSM access without public IP)
# Uncomment if EKS nodes are fully private:
# resource "aws_vpc_endpoint" "ssm" {
#   vpc_id              = module.vpc.vpc_id
#   service_name        = "com.amazonaws.${local.region}.ssm"
#   vpc_endpoint_type   = "Interface"
#   subnet_ids          = module.vpc.private_subnet_ids
#   security_group_ids  = [aws_security_group.vpce.id]
#   private_dns_enabled = true
#   tags = { Name = "${local.org}-${local.env}-vpce-ssm", Service = "network" }
# }
```

### Step 5: `security_groups.tf`

```hcl
# Remove all rules from the default security group (hardening)
resource "aws_default_security_group" "default" {
  vpc_id = module.vpc.vpc_id
  # No ingress or egress rules — intentionally empty
  tags = { Name = "${local.org}-${local.env}-sg-default-hardened" }
}

# Internal communication within VPC
resource "aws_security_group" "internal" {
  name        = "${local.org}-${local.env}-sg-internal"
  description = "Allows all traffic within the VPC."
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = [local.vpc_cidr]
    description = "All traffic within VPC CIDR."
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound traffic."
  }

  tags = { Name = "${local.org}-${local.env}-sg-internal", Service = "network" }
}
```

### Step 6: `outputs.tf`

```hcl
output "vpc_id" {
  description = "VPC ID."
  value       = module.vpc.vpc_id
}

output "vpc_cidr_block" {
  description = "VPC CIDR block."
  value       = module.vpc.vpc_cidr_block
}

output "private_subnet_ids" {
  description = "Private subnet IDs."
  value       = module.vpc.private_subnet_ids
}

output "public_subnet_ids" {
  description = "Public subnet IDs."
  value       = module.vpc.public_subnet_ids
}

output "sg_internal_id" {
  description = "Security group ID for internal VPC traffic."
  value       = aws_security_group.internal.id
}
```

---

## 7. Expected file tree

```
live/staging/20-network/
  backend.tf, versions.tf, providers.tf, locals.tf, data.tf
  vpc.tf, endpoints.tf, security_groups.tf, outputs.tf
  .terraform.lock.hcl
live/production/20-network/
  (same)
```

---

## 8. Code contracts

Every public subnet must have tag `kubernetes.io/role/elb = "1"`.
Every private subnet must have tag `kubernetes.io/role/internal-elb = "1"`.
Both must have `kubernetes.io/cluster/<eks-cluster-name> = "shared"`.
Failure to set these tags prevents the EKS ALB controller from discovering subnets.

---

## 9. Acceptance criteria

- [ ] `terraform validate` passes.
- [ ] `terraform plan` shows zero deletions for existing resources (only imports and additions).
- [ ] After apply: `aws ec2 describe-vpcs --filters "Name=tag:Name,Values=<org>-<env>-vpc"` returns the VPC.
- [ ] Public and private subnets have the EKS required tags.
- [ ] S3 and DynamoDB gateway endpoints are visible in the VPC console.
- [ ] Default security group has zero ingress/egress rules.

---

## 10. Guardrails

- Never delete an existing subnet — it may have active ENIs (EKS nodes, RDS, etc.).
- Never use `0.0.0.0/0` as an ingress rule in any security group other than public-facing ALBs.
- If an existing VPC has overlapping CIDRs with the desired design, flag it to the human before making any changes.
- Single NAT gateway is acceptable for staging; production must have one NAT gateway per AZ.

---

## 11. Handoff note

When complete, report:
1. VPC ID created or imported.
2. Subnet IDs for private and public subnets.
3. NAT gateway configuration (single vs per-AZ).
4. VPC endpoints created.
5. All outputs exposed (VPC ID, subnet IDs, security group IDs) — these are consumed by 30-dns, 40-data, 50-compute, and 80-eks.
