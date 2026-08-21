# PROMPT-14: Compute — EC2 and ALB

## 1. Role and objective

You are a senior SRE building the `50-compute` layer for both staging and production. This layer manages EC2 instances (standalone or in Auto Scaling Groups), Application Load Balancers, target groups, listeners, and their associated security groups. Existing EC2 instances and ALBs from the inventory are imported. New resources follow the org conventions.

---

## 2. Preconditions

- [ ] PROMPT-11 (network) complete: VPC ID, subnet IDs, security group IDs from remote state.
- [ ] PROMPT-12 (dns/ACM) complete: ACM certificate ARN from remote state.
- [ ] PROMPT-10 (identity) complete: IAM instance profiles available if EC2 needs SSM access.
- [ ] Inventory has: EC2 instance IDs, AMI IDs, ALB ARNs, target group ARNs.
- [ ] Human has confirmed what EC2 instances exist and their purpose.

---

## 3. Required inputs

1. List of existing EC2 instances from inventory: instance ID, type, purpose, subnet.
2. List of existing ALBs from inventory: ARN, listener ports, target groups.
3. Should new EC2 instances use SSM Session Manager (no SSH key pairs) or SSH?
4. What is the expected compute architecture: standalone EC2, ASG, or everything on EKS (EC2 only for special cases)?
5. Desired ALB for EKS ingress: internal or internet-facing?

---

## 4. In scope / out of scope

**In scope:**
- Standalone EC2 instances (for bastion, NAT proxy, or legacy apps).
- Auto Scaling Groups with Launch Templates.
- Application Load Balancers and HTTPS listeners.
- Target groups for HTTP/HTTPS backends.
- Security groups for EC2 and ALB.
- IAM instance profile with SSM permissions.
- Import blocks for all existing EC2 and ALB resources.

**Out of scope:**
- EKS node groups (managed in `80-eks`).
- RDS or DocumentDB instances (managed in `40-data`).
- Fargate (managed in `80-eks`).
- Network Load Balancers (add if needed — they follow the same pattern).

---

## 5. Reference material

- `00-standards/conventions.md` — naming: `<org>-<env>-alb-main`, `<org>-<env>-ec2-<purpose>`.
- `01-discovery/inventory.md` — compute section.

---

## 6. Step-by-step procedure

### Step 1: File structure

```
live/staging/50-compute/
  backend.tf, versions.tf, providers.tf, locals.tf, data.tf
  security_groups.tf
  alb.tf
  ec2.tf              (only if standalone EC2 exists)
  asg.tf              (only if ASGs exist)
  outputs.tf
  .terraform.lock.hcl
```

### Step 2: `data.tf` — read upstream state

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket      = "<ORG>-tfstate-stg-<STAGING_REGION>"
    key         = "staging/20-network/terraform.tfstate"
    region      = "<STAGING_REGION>"
    assume_role = { role_arn = "arn:aws:iam::<STAGING_ACCOUNT_ID>:role/<ORG>-stg-terraform-exec" }
  }
}

data "terraform_remote_state" "dns" {
  backend = "s3"
  config = {
    bucket      = "<ORG>-tfstate-stg-<STAGING_REGION>"
    key         = "staging/30-dns/terraform.tfstate"
    region      = "<STAGING_REGION>"
    assume_role = { role_arn = "arn:aws:iam::<STAGING_ACCOUNT_ID>:role/<ORG>-stg-terraform-exec" }
  }
}

locals {
  vpc_id             = data.terraform_remote_state.network.outputs.vpc_id
  private_subnet_ids = data.terraform_remote_state.network.outputs.private_subnet_ids
  public_subnet_ids  = data.terraform_remote_state.network.outputs.public_subnet_ids
  cert_arn           = data.terraform_remote_state.dns.outputs.staging_wildcard_cert_arn
}
```

### Step 3: `security_groups.tf` — ALB security groups

```hcl
# ALB: public internet-facing HTTPS
resource "aws_security_group" "alb_public" {
  name        = "${local.org}-${local.env}-sg-alb-public"
  description = "Public ALB — allows HTTPS from internet."
  vpc_id      = local.vpc_id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS from internet."
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP from internet (redirect to HTTPS)."
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound."
  }

  tags = { Name = "${local.org}-${local.env}-sg-alb-public", Service = "compute" }
}
```

### Step 4: `alb.tf`

```hcl
resource "aws_lb" "main" {
  name               = "${local.org}-${local.env}-alb-main"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_public.id]
  subnets            = local.public_subnet_ids

  enable_deletion_protection = true   # false for staging

  access_logs {
    bucket  = "<LOG_BUCKET>"   # from 60-storage layer — leave as data source reference
    prefix  = "alb/main"
    enabled = true
  }

  tags = { Name = "${local.org}-${local.env}-alb-main", Service = "compute" }
}

# HTTPS listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = local.cert_arn

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "No route matched."
      status_code  = "404"
    }
  }
}

# HTTP to HTTPS redirect
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

**If importing an existing ALB:**

```hcl
import {
  to = aws_lb.main
  id = "<existing-alb-arn>"
}
```

### Step 5: `ec2.tf` — standalone EC2 (only if needed per inventory)

```hcl
# Example: NAT proxy or bastion equivalent (prefer SSM over SSH)
resource "aws_iam_instance_profile" "ssm" {
  name = "${local.org}-${local.env}-ec2-ssm-profile"
  role = aws_iam_role.ec2_ssm.name
}

resource "aws_iam_role" "ec2_ssm" {
  name               = "${local.org}-${local.env}-ec2-ssm"
  assume_role_policy = data.aws_iam_policy_document.ec2_trust.json
  managed_policy_arns = [
    "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
  ]
}

data "aws_iam_policy_document" "ec2_trust" {
  statement {
    actions = ["sts:AssumeRole"]
    principals { type = "Service"; identifiers = ["ec2.amazonaws.com"] }
  }
}

resource "aws_instance" "example" {
  ami                    = data.aws_ami.amazon_linux_2023.id
  instance_type          = "t4g.small"
  subnet_id              = local.private_subnet_ids[0]
  iam_instance_profile   = aws_iam_instance_profile.ssm.name
  vpc_security_group_ids = [data.terraform_remote_state.network.outputs.sg_internal_id]

  root_block_device {
    volume_type = "gp3"
    volume_size = 20
    encrypted   = true
  }

  tags = { Name = "${local.org}-${local.env}-ec2-<purpose>", Service = "compute" }
}

data "aws_ami" "amazon_linux_2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}
```

### Step 6: `outputs.tf`

```hcl
output "alb_arn" {
  description = "ARN of the main ALB."
  value       = aws_lb.main.arn
}

output "alb_dns_name" {
  description = "DNS name of the main ALB."
  value       = aws_lb.main.dns_name
}

output "alb_zone_id" {
  description = "Route53 zone ID of the ALB (for alias records)."
  value       = aws_lb.main.zone_id
}

output "alb_https_listener_arn" {
  description = "ARN of the HTTPS listener."
  value       = aws_lb_listener.https.arn
}

output "sg_alb_public_id" {
  description = "Security group ID of the public ALB."
  value       = aws_security_group.alb_public.id
}
```

---

## 7. Expected file tree

```
live/staging/50-compute/
  backend.tf, versions.tf, providers.tf, locals.tf, data.tf
  security_groups.tf, alb.tf, ec2.tf (if needed), outputs.tf
  .terraform.lock.hcl
live/production/50-compute/
  (enable_deletion_protection=true on ALB; access logs enabled)
```

---

## 8. Code contracts

- TLS policy on HTTPS listeners: always `ELBSecurityPolicy-TLS13-1-2-2021-06` (most modern, TLS 1.3 preferred).
- ALB access logs must be enabled for production.
- No EC2 key pairs — use SSM Session Manager exclusively.
- `enable_deletion_protection = true` on production ALBs.

---

## 9. Acceptance criteria

- [ ] `terraform validate` passes.
- [ ] Plan shows no deletions of existing ALBs or EC2 instances.
- [ ] After apply: ALB state is `active`, HTTPS listener is attached, HTTP listener redirects to HTTPS.
- [ ] `curl -I https://<STAGING_DOMAIN>` returns `404 No route matched` (the default action) rather than a connection error.

---

## 10. Guardrails

- Never delete an existing ALB or listener without explicit human confirmation.
- Never use `HTTP` as the final listener protocol — always redirect to HTTPS.
- Never allow SSH ingress to EC2 instances — use SSM.
- `access_logs` with an invalid bucket name causes an ALB creation failure — confirm the S3 bucket name from the 60-storage layer before apply.

---

## 11. Handoff note

Report: ALB ARN, ALB DNS name, security group IDs, and existing EC2 instances imported. Note whether the ALB was imported from inventory or created fresh.
