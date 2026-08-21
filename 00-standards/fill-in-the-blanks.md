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
| Staging subdomain pattern | `<STAGING_DOMAIN>` | | e.g. `uat.acme.com` |
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

## EKS (staging) — DEFERRED (cluster not in this account yet)

**Skip this section for now.** The staging EKS cluster lives in **another AWS account** and will be transferred later. Until it exists in `<STAGING_ACCOUNT_ID>`, you cannot discover OIDC/IRSA values and **cannot run PROMPT-04 (Atlantis)**.

Continue with bootstrap, repo scaffold, modules, and non-EKS layers. After transfer, fill these from inventory in the destination account, then run PROMPT-04.

| Key | Placeholder | Your value | Status |
|---|---|---|---|
| Staging EKS cluster name | `<STAGING_EKS_CLUSTER>` | | [DEFERRED] after account transfer |
| Staging EKS OIDC provider ARN | `<STAGING_OIDC_ARN>` | | [DEFERRED] |
| Staging EKS OIDC provider URL (no https://) | `<STAGING_OIDC_URL>` | | [DEFERRED] |
| Kubernetes version | `<K8S_VERSION>` | | [DEFERRED] |
| Atlantis namespace on EKS | `<ATLANTIS_NAMESPACE>` | | [DEFERRED] e.g. `atlantis` |
| Atlantis Helm release name | `<ATLANTIS_RELEASE>` | | [DEFERRED] e.g. `atlantis` |
| Source account ID where cluster lives today (optional note) | — | | For your own tracking only |

---

## EKS (production) — existing cluster (if any) — usually DEFERRED

Same rule: leave blank until the cluster is reachable in the production account you manage with this pack.

| Key | Placeholder | Your value | Status |
|---|---|---|---|
| Production EKS cluster name | `<PRODUCTION_EKS_CLUSTER>` | | [DEFERRED] unless already in this account |
| Production EKS OIDC provider ARN | `<PRODUCTION_OIDC_ARN>` | | [DEFERRED] |

---

## Secrets Management

**What this section is:** Terraform sometimes needs to *create or read* passwords/tokens (e.g. a DocumentDB master password). Those values must live in your company's secrets product — not in git and not in Terraform outputs. This section only records *which* product you use so later prompts can write the right HCL.

**How to fill it (you do not need deep Terraform knowledge):**

1. Ask a teammate / platform / security: *"Where do we store app and infra secrets today?"*
2. Put that product name in `<SECRETS_TOOL>`.
3. Leave provider fields blank until you know the tool — fill them when you run `PROMPT-16` (or ask whoever owns the secrets platform).

| Key | Placeholder | Your value | Notes |
|---|---|---|---|
| Secrets tool name | `<SECRETS_TOOL>` | | **Start here.** Whatever your company already uses: `AWS Secrets Manager`, `HashiCorp Vault`, `CyberArk`, `1Password`, etc. If unsure, ask IT/platform. |
| Secrets tool endpoint / address | `<SECRETS_TOOL_URL>` | | Only if the tool has a URL (Vault often does, e.g. `https://vault.acme.com`). Secrets Manager usually has **no** separate URL — leave blank. |
| Terraform provider for secrets tool | `<SECRETS_PROVIDER>` | | **Optional until PROMPT-16.** This is the Terraform plugin name that talks to that tool — not a secret itself. Lookup once you know the tool (table below). |
| Terraform provider version | `<SECRETS_PROVIDER_VERSION>` | | **Optional until PROMPT-16.** Pin used elsewhere in the company, or latest stable when you implement secrets. |

**Provider cheat sheet** (fill `<SECRETS_PROVIDER>` from the tool you picked):

| If `<SECRETS_TOOL>` is… | Use `<SECRETS_PROVIDER>` | Why |
|---|---|---|
| AWS Secrets Manager or SSM Parameter Store | `hashicorp/aws` | Same AWS provider you already use; no extra provider. |
| HashiCorp Vault | `hashicorp/vault` | Dedicated Vault provider. |
| Something else / unknown | Leave blank for now | PROMPT-16 will ask again; do not guess. |

You only need `<SECRETS_TOOL>` before secrets-heavy layers (PROMPT-13 / PROMPT-16). Provider fields can wait. Atlantis is deferred until EKS is in this account.

---

## piaas.yml Platform — DEFERRED

**Skip this section for now.** Quality-gate CI (`piaas.yml` / PROMPT-05) is separate from Atlantis and the Terraform layers. You do not need runner images, schema version, or trigger syntax to bootstrap state, scaffold repos, or run Atlantis.

Fill these later when you deliberately start PROMPT-05 and have platform-team answers (or a sample `piaas.yml` from another repo).

| Key | Placeholder | Your value | Notes |
|---|---|---|---|
| piaas.yml schema version | `<PIAAS_VERSION>` | | [DEFERRED] Ask platform team when ready |
| Available runner image with Terraform | `<PIAAS_TF_IMAGE>` | | [DEFERRED] |
| Available runner image with general tools | `<PIAAS_TOOLS_IMAGE>` | | [DEFERRED] For tflint, trivy, checkov |
| Pipeline trigger syntax (PR open) | `<PIAAS_PR_TRIGGER>` | | [DEFERRED] |
| Pipeline trigger syntax (push to branch) | `<PIAAS_PUSH_TRIGGER>` | | [DEFERRED] |
| How to set pipeline env variables | `<PIAAS_ENV_VARS>` | | [DEFERRED] |

---

## Terraform State Buckets

These will be created by `00-bootstrap` but names must be decided now.

| Environment | Bucket name | Region |
|---|---|---|
| Staging | `<ORG>-tfstate-uat-<STAGING_REGION>` | `<STAGING_REGION>` |
| Production | `<ORG>-tfstate-prd-<PRODUCTION_REGION>` | `<PRODUCTION_REGION>` |

---

## IAM Execution Roles

Created by `00-bootstrap`. Names are pre-decided here.

| Environment | Role name | Account | Notes |
|---|---|---|---|
| Staging | `<ORG>-uat-terraform-exec` | `<STAGING_ACCOUNT_ID>` | |
| Production | `<ORG>-prd-terraform-exec` | `<PRODUCTION_ACCOUNT_ID>` | |
| Atlantis pod (staging EKS) | `<ORG>-uat-atlantis` | `<STAGING_ACCOUNT_ID>` | [DEFERRED] after EKS transfer + PROMPT-04 |

---

## Existing Resources to Import

Fill this in after running `PROMPT-00-inventory-existing-aws.md`. Add rows as needed.

| Resource type | AWS resource ID | Terraform target | Layer | Notes |
|---|---|---|---|---|
| `aws_vpc` | `<STAGING_VPC_ID>` | `module.vpc.aws_vpc.this[0]` | `20-network` | |
| `aws_eks_cluster` | `<STAGING_EKS_CLUSTER>` | See EKS questionnaire | `80-eks` | Deferred |
| | | | | |
| | | | | |
