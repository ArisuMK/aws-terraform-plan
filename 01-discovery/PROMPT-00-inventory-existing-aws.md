# PROMPT-00: Inventory Existing AWS Resources

## 1. Role and objective

You are a senior SRE tasked with producing a complete inventory of existing AWS resources across two accounts — staging and production — so that subsequent Terraform prompts can import them into managed state rather than recreate them. Your output is a structured Markdown inventory document and a pre-filled import table for `00-standards/fill-in-the-blanks.md`.

You do not write any Terraform code in this prompt. You only gather, organize, and document what exists.

---

## 2. Preconditions

Confirm with the human before starting:

- [ ] `00-standards/conventions.md` has been read.
- [ ] `00-standards/fill-in-the-blanks.md` is open for filling in `[DISCOVERY]` fields.
- [ ] You have AWS CLI access to both the staging account (`<STAGING_ACCOUNT_ID>`) and the production account (`<PRODUCTION_ACCOUNT_ID>`), or the human can run commands and paste output.
- [ ] At minimum, the IAM permissions `ec2:Describe*`, `rds:Describe*`, `eks:Describe*`, `s3:List*`, `iam:List*`, `route53:List*`, `ecr:Describe*`, `elasticache:Describe*`, `docdb:Describe*`, `cloudtrail:Describe*`, `organizations:Describe*` are available.

---

## 3. Required inputs

Ask the human to provide these before starting:

1. **Staging AWS account ID** (`<STAGING_ACCOUNT_ID>`)
2. **Production AWS account ID** (`<PRODUCTION_ACCOUNT_ID>`)
3. **Staging region** (`<STAGING_REGION>`)
4. **Production region** (`<PRODUCTION_REGION>`)
5. **AWS CLI profile names** for staging and production (or instructions on how to switch accounts — role assumption, SSO, etc.)
6. **Any resource name prefixes or tags** the team uses today so we can filter results meaningfully.

---

## 4. In scope / out of scope

**In scope:**
- VPC, subnets, route tables, internet gateways, NAT gateways, security groups, VPC endpoints, VPC peering connections.
- EKS clusters, node groups, Fargate profiles.
- EC2 instances, launch templates, Auto Scaling groups, key pairs.
- ALBs, NLBs, target groups.
- RDS clusters and instances, DocumentDB clusters and instances, ElastiCache clusters.
- S3 buckets (names, regions, versioning, encryption status).
- ECR repositories.
- IAM roles, policies (customer-managed only), OIDC providers.
- Route53 hosted zones and significant record sets.
- ACM certificates.
- CloudTrail trails.
- Existing S3 buckets used for Terraform state (critical to identify to avoid collisions).
- CloudWatch log groups.
- Secrets Manager secrets (names only, never values).
- SSM Parameter Store parameters (names only, never values).

**Out of scope:**
- Lambda functions (unless they are critical to other resources).
- SQS/SNS topics (unless the human flags them as required).
- Billing and Cost Explorer data.
- Service Catalog, SSO, Organizations (unless the human flags them).

---

## 5. Reference material

Read before starting:
- `00-standards/conventions.md` — naming conventions, this helps identify which resources are already well-named versus ad-hoc.
- `00-standards/fill-in-the-blanks.md` — the template you will fill in.

---

## 6. Step-by-step procedure

Work through each account (staging first, then production) and each resource category below. For each, run the appropriate AWS CLI command (or ask the human to run it and paste output), then record the results.

### Step 1: Account metadata

```bash
# Run in each account
aws sts get-caller-identity
aws configure get region
```

Record the account ID and confirm the region.

### Step 2: Existing Terraform state buckets

This is critical. If a state bucket already exists, we must not create a new one with a conflicting name.

```bash
aws s3api list-buckets --query "Buckets[?contains(Name, 'terraform') || contains(Name, 'tfstate') || contains(Name, 'state')].Name" --output table
```

For each candidate bucket:

```bash
aws s3api get-bucket-location --bucket <bucket-name>
aws s3api get-bucket-versioning --bucket <bucket-name>
aws s3api get-bucket-encryption --bucket <bucket-name>
# List state file keys to understand the key structure
aws s3 ls s3://<bucket-name>/ --recursive | head -50
```

### Step 3: VPC and networking

```bash
aws ec2 describe-vpcs --output table
aws ec2 describe-subnets --query "Subnets[*].{ID:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone,Public:MapPublicIpOnLaunch,Tags:Tags}" --output table
aws ec2 describe-internet-gateways --output table
aws ec2 describe-nat-gateways --output table
aws ec2 describe-route-tables --output table
aws ec2 describe-security-groups --query "SecurityGroups[*].{ID:GroupId,Name:GroupName,VPC:VpcId}" --output table
aws ec2 describe-vpc-endpoints --output table
aws ec2 describe-vpc-peering-connections --output table
```

### Step 4: EKS

**Expected for this org:** staging EKS may be **absent** from the account you are inventorying (cluster still in another account pending transfer). If `list-clusters` is empty, record “none in this account — deferred” and skip the describe calls. Do not invent placeholder cluster names.

```bash
aws eks list-clusters
# For each cluster (only if any exist in this account):
aws eks describe-cluster --name <cluster-name>
aws eks list-nodegroups --cluster-name <cluster-name>
# For each node group:
aws eks describe-nodegroup --cluster-name <cluster-name> --nodegroup-name <ng-name>
aws eks list-fargate-profiles --cluster-name <cluster-name>
# Get OIDC provider
aws iam list-open-id-connect-providers
```

Record the OIDC provider URL only when a cluster exists here — required later for IRSA / Atlantis.

### Step 5: EC2 compute

```bash
aws ec2 describe-instances --query "Reservations[*].Instances[*].{ID:InstanceId,Type:InstanceType,State:State.Name,Name:Tags[?Key=='Name']|[0].Value,IP:PrivateIpAddress}" --output table
aws ec2 describe-launch-templates --output table
aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[*].{Name:AutoScalingGroupName,Min:MinSize,Max:MaxSize,Desired:DesiredCapacity}" --output table
```

### Step 6: Load balancers

```bash
aws elbv2 describe-load-balancers --output table
aws elbv2 describe-target-groups --output table
```

### Step 7: Data stores

```bash
# RDS
aws rds describe-db-clusters --output table
aws rds describe-db-instances --output table

# DocumentDB
aws docdb describe-db-clusters --output table
aws docdb describe-db-instances --output table

# ElastiCache
aws elasticache describe-cache-clusters --output table
aws elasticache describe-replication-groups --output table
```

### Step 8: S3 buckets

```bash
aws s3api list-buckets --query "Buckets[*].Name" --output table
# For each bucket — check region, versioning, encryption:
aws s3api get-bucket-location --bucket <bucket-name>
```

### Step 9: ECR

```bash
aws ecr describe-repositories --output table
```

### Step 10: IAM

```bash
# Customer-managed policies only
aws iam list-policies --scope Local --query "Policies[*].{Name:PolicyName,ARN:Arn}" --output table
# Roles
aws iam list-roles --query "Roles[*].{Name:RoleName,ARN:Arn}" --output table | head -80
# OIDC providers
aws iam list-open-id-connect-providers --output table
```

### Step 11: Route53

```bash
aws route53 list-hosted-zones --output table
# For each zone:
aws route53 list-resource-record-sets --hosted-zone-id <zone-id> --output table
```

### Step 12: ACM certificates

```bash
aws acm list-certificates --region <region> --output table
```

### Step 13: CloudTrail

```bash
aws cloudtrail describe-trails --output table
```

### Step 14: Secrets (names only)

```bash
aws secretsmanager list-secrets --query "SecretList[*].Name" --output table
aws ssm describe-parameters --query "Parameters[*].Name" --output table
```

### Step 15: CloudWatch log groups

```bash
aws logs describe-log-groups --query "logGroups[*].{Name:logGroupName,Retention:retentionInDays}" --output table
```

---

## 7. Expected output after completion

Produce a single Markdown file: `01-discovery/inventory.md` with this structure:

```markdown
# AWS Resource Inventory

Generated: <date>

## Staging account (<STAGING_ACCOUNT_ID>, <STAGING_REGION>)

### Existing state buckets
...

### VPC and networking
...

### EKS clusters
...

(... all categories ...)

## Production account (<PRODUCTION_ACCOUNT_ID>, <PRODUCTION_REGION>)

(... same structure ...)

## Import candidates

| Layer | Resource type | Resource ID | Terraform target | Notes |
|---|---|---|---|---|
| 20-network | aws_vpc | vpc-xxx | module.vpc.aws_vpc.this[0] | Existing VPC |
| ... | ... | ... | ... | ... |

## Gaps identified

List any resources that exist in AWS but have no obvious Terraform equivalent planned yet.
These need a decision: import, delete, or leave unmanaged.

## Naming compliance

List any resources whose names do not match the `<org>-<env>-<service>` pattern.
Flag whether to rename them (requires destroy+recreate) or import as-is.
```

Also update `00-standards/fill-in-the-blanks.md` with all `[DISCOVERY]` values found.

---

## 8. Code contracts

None — this prompt produces documentation only, no Terraform code.

---

## 9. Acceptance criteria

- [ ] `01-discovery/inventory.md` exists and covers all 15 resource categories for both accounts.
- [ ] Every `[DISCOVERY]` field in `fill-in-the-blanks.md` is filled in or explicitly marked "not found".
- [ ] The import candidates table has an entry for every resource that needs to be imported in a later layer prompt.
- [ ] Gaps identified are listed explicitly — no silently missing resources.

---

## 10. Guardrails

- Never run commands that modify AWS resources (`aws ec2 create-*`, `aws s3 rm`, etc.).
- Never print secret values — only names/ARNs.
- Never assume a resource does not exist without running the describe command.
- Do not proceed to any other prompt until this inventory is complete and reviewed by the human.

---

## 11. Handoff note

When complete, report:

1. Summary counts: X VPCs, Y EKS clusters, Z RDS instances, etc., per account.
2. Any surprising finds (resources that have no obvious owner or purpose).
3. Estimated import effort: simple (data sources only), moderate (import blocks needed), complex (destructive changes required to align with conventions).
4. The complete list of `[DISCOVERY]` values that were filled in.
5. Any blockers that prevent subsequent prompts from starting (e.g. missing OIDC provider means IRSA cannot be set up).
