# Fill in the Blanks

**Instructions:** Before running any prompt, fill in every `<PLACEHOLDER>` in this file. The executing AI will reference this document at the start of each prompt. Values marked `[REQUIRED]` block progress until supplied. Values marked `[DISCOVERY]` will be found during `PROMPT-00` and should be filled in after that prompt completes.

Save a copy of this file with real values as `fill-in-the-blanks.local.md` (gitignored). Never commit real account IDs, CIDRs, or other production details to this file in the repository.

---

## Organization

| Key | Placeholder | Your value | Notes |
|---|---|---|---|
| Short org name (lowercase, no spaces) | `<ORG>` | | Used in all resource names, e.g. `acme` |
| Bitbucket project key | `<BB_PROJECT>` | | e.g. `INFRA` |
| Infra repo slug | `<INFRA_REPO>` | | e.g. `acme-infrastructure` |
| Module library repo slug | `<MODULES_REPO>` | | e.g. `acme-terraform-modules` |

---

## AWS Accounts

| Key | Placeholder | Your value | Notes |
|---|---|---|---|
| Staging account ID | `<STAGING_ACCOUNT_ID>` | | 12-digit number |
| Production account ID | `<PRODUCTION_ACCOUNT_ID>` | | 12-digit number |
| AWS Organization management account ID (if any) | `<ORG_ACCOUNT_ID>` | | Leave blank if no AWS Org |

---

## Regions

| Key | Placeholder | Your value | Notes |
|---|---|---|---|
| Staging primary region | `<STAGING_REGION>` | | e.g. `us-east-1` |
| Production primary region | `<PRODUCTION_REGION>` | | e.g. `us-east-1` |
| Secondary region (if any) | `<SECONDARY_REGION>` | | Leave blank if not used |

---

## Networking

| Key | Placeholder | Your value | Status |
|---|---|---|---|
| Staging VPC ID (if it exists) | `<STAGING_VPC_ID>` | | [DISCOVERY] |
| Staging VPC CIDR | `<STAGING_VPC_CIDR>` | | [DISCOVERY] e.g. `10.0.0.0/16` |
| Staging private subnet IDs | `<STAGING_PRIVATE_SUBNET_IDS>` | | [DISCOVERY] comma-separated |
| Staging public subnet IDs | `<STAGING_PUBLIC_SUBNET_IDS>` | | [DISCOVERY] comma-separated |
| Production VPC ID (if it exists) | `<PRODUCTION_VPC_ID>` | | [DISCOVERY] |
| Production VPC CIDR | `<PRODUCTION_VPC_CIDR>` | | [DISCOVERY] e.g. `10.1.0.0/16` |
| Production private subnet IDs | `<PRODUCTION_PRIVATE_SUBNET_IDS>` | | [DISCOVERY] |
| Production public subnet IDs | `<PRODUCTION_PUBLIC_SUBNET_IDS>` | | [DISCOVERY] |
| VPN/office CIDR (if any) | `<OFFICE_CIDR>` | | Used in security group ingress rules |

---

## DNS and Domains

| Key | Placeholder | Your value | Notes |
|---|---|---|---|
| Public domain root | `<PUBLIC_DOMAIN>` | | e.g. `acme.com` |
| Staging subdomain pattern | `<STAGING_DOMAIN>` | | e.g. `stg.acme.com` |
| Production subdomain pattern | `<PRODUCTION_DOMAIN>` | | e.g. `acme.com` or `prd.acme.com` |
| Route53 staging hosted zone ID (if it exists) | `<STAGING_ZONE_ID>` | | [DISCOVERY] |
| Route53 production hosted zone ID (if it exists) | `<PRODUCTION_ZONE_ID>` | | [DISCOVERY] |
| DNS registrar (if not Route53) | `<DNS_REGISTRAR>` | | e.g. `GoDaddy`, `Cloudflare` |

---

## Bitbucket Data Center

| Key | Placeholder | Your value | Notes |
|---|---|---|---|
| Bitbucket base URL (no trailing slash) | `<BITBUCKET_BASE_URL>` | | e.g. `https://bitbucket.acme.com` |
| Bitbucket API URL | `<BITBUCKET_API_URL>` | | Usually `<BITBUCKET_BASE_URL>/rest/api/1.0` |
| Bitbucket SSH host:port | `<BITBUCKET_SSH_HOST>` | | e.g. `bitbucket.acme.com:7999` |
| Atlantis webhook secret | `<ATLANTIS_WEBHOOK_SECRET>` | | Generate with `openssl rand -hex 32`. Store in secrets tool. |
| Service account username for Atlantis | `<ATLANTIS_BB_USER>` | | Bitbucket service account |
| Service account token | `<ATLANTIS_BB_TOKEN>` | | Store in secrets tool, not here. |

---

## EKS (staging) — existing cluster

| Key | Placeholder | Your value | Status |
|---|---|---|---|
| Staging EKS cluster name | `<STAGING_EKS_CLUSTER>` | | [DISCOVERY] |
| Staging EKS OIDC provider ARN | `<STAGING_OIDC_ARN>` | | [DISCOVERY] |
| Staging EKS OIDC provider URL (no https://) | `<STAGING_OIDC_URL>` | | [DISCOVERY] |
| Kubernetes version | `<K8S_VERSION>` | | [DISCOVERY] |
| Atlantis namespace on EKS | `<ATLANTIS_NAMESPACE>` | | e.g. `atlantis` |
| Atlantis Helm release name | `<ATLANTIS_RELEASE>` | | e.g. `atlantis` |

---

## EKS (production) — existing cluster (if any)

| Key | Placeholder | Your value | Status |
|---|---|---|---|
| Production EKS cluster name | `<PRODUCTION_EKS_CLUSTER>` | | [DISCOVERY] |
| Production EKS OIDC provider ARN | `<PRODUCTION_OIDC_ARN>` | | [DISCOVERY] |

---

## Secrets Management

| Key | Placeholder | Your value | Notes |
|---|---|---|---|
| Secrets tool name | `<SECRETS_TOOL>` | | e.g. `AWS Secrets Manager`, `HashiCorp Vault`, `1Password`, `CyberArk` |
| Secrets tool endpoint / address | `<SECRETS_TOOL_URL>` | | If applicable |
| Terraform provider for secrets tool | `<SECRETS_PROVIDER>` | | e.g. `hashicorp/vault`, `hashicorp/aws` (for Secrets Manager) |
| Terraform provider version | `<SECRETS_PROVIDER_VERSION>` | | e.g. `4.8.0` |

---

## piaas.yml Platform

| Key | Placeholder | Your value | Notes |
|---|---|---|---|
| piaas.yml schema version | `<PIAAS_VERSION>` | | Check with your platform team |
| Available runner image with Terraform | `<PIAAS_TF_IMAGE>` | | e.g. `hashicorp/terraform:1.11.x` or a custom image |
| Available runner image with general tools | `<PIAAS_TOOLS_IMAGE>` | | For tflint, trivy, checkov |
| Pipeline trigger syntax (PR open) | `<PIAAS_PR_TRIGGER>` | | From platform docs |
| Pipeline trigger syntax (push to branch) | `<PIAAS_PUSH_TRIGGER>` | | From platform docs |
| How to set pipeline env variables | `<PIAAS_ENV_VARS>` | | From platform docs |

---

## Terraform State Buckets

These will be created by `00-bootstrap` but names must be decided now.

| Environment | Bucket name | Region |
|---|---|---|
| Staging | `<ORG>-tfstate-stg-<STAGING_REGION>` | `<STAGING_REGION>` |
| Production | `<ORG>-tfstate-prd-<PRODUCTION_REGION>` | `<PRODUCTION_REGION>` |

---

## IAM Execution Roles

Created by `00-bootstrap`. Names are pre-decided here.

| Environment | Role name | Account |
|---|---|---|
| Staging | `<ORG>-stg-terraform-exec` | `<STAGING_ACCOUNT_ID>` |
| Production | `<ORG>-prd-terraform-exec` | `<PRODUCTION_ACCOUNT_ID>` |
| Atlantis pod (staging EKS) | `<ORG>-stg-atlantis` | `<STAGING_ACCOUNT_ID>` |

---

## Existing Resources to Import

Fill this in after running `PROMPT-00-inventory-existing-aws.md`. Add rows as needed.

| Resource type | AWS resource ID | Terraform target | Layer | Notes |
|---|---|---|---|---|
| `aws_vpc` | `<STAGING_VPC_ID>` | `module.vpc.aws_vpc.this[0]` | `20-network` | |
| `aws_eks_cluster` | `<STAGING_EKS_CLUSTER>` | See EKS questionnaire | `80-eks` | Deferred |
| | | | | |
| | | | | |
