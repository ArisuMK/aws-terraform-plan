# PROMPT-04: Atlantis on Bitbucket Data Center (EC2 interim)

> **Hosting model:** Run Atlantis on a dedicated **EC2 instance** in the staging account for now. Staging EKS is still in another account and may take a while to transfer. When EKS lands in this account, migrate Atlantis to Helm + IRSA (Part B at the end) and decommission the EC2 host. Do not wait on EKS to get PR-driven plan/apply.

## 1. Role and objective

You are a senior SRE deploying Atlantis on an EC2 instance in the staging AWS account, wired to Bitbucket Data Center webhooks. Atlantis posts plan output as PR comments and applies Terraform when authorized. After this prompt is complete, the team manages Terraform through PR comments — no routine local `terraform apply`.

---

## 2. Preconditions

- [ ] PROMPT-01 is complete: the `<ORG>-uat-terraform-exec` and `<ORG>-prd-terraform-exec` IAM roles exist.
- [ ] PROMPT-02 is complete: `atlantis.yaml` is in the infra repo.
- [ ] `QUESTIONNAIRE-org-context.md` Sections A (Bitbucket DC), D, and E have been answered (enough to create a bot user, token, and webhook).
- [ ] `fill-in-the-blanks.local.md` has: `<ATLANTIS_WEBHOOK_SECRET>`, `<ATLANTIS_BB_USER>`, `<ATLANTIS_BB_TOKEN>`, `<BITBUCKET_BASE_URL>`, `<STAGING_REGION>`, `<STAGING_ACCOUNT_ID>`, `<PRODUCTION_ACCOUNT_ID>`, plus networking answers below (or the human will supply them in this session).
- [ ] A VPC and subnet exist in staging where the instance can reach Bitbucket DC (and Bitbucket can reach Atlantis on HTTPS), **or** the human agrees to create a minimal VPC/subnet as part of this prompt.
- [ ] Staging EKS is **not** required.

---

## 3. Required inputs

Ask the human to confirm before writing:

1. `<ORG>`, `<STAGING_ACCOUNT_ID>`, `<PRODUCTION_ACCOUNT_ID>`, `<STAGING_REGION>`.
2. `<BITBUCKET_BASE_URL>`, `<BB_PROJECT>`, `<INFRA_REPO>`, `<BITBUCKET_SSH_HOST>`.
3. `<ATLANTIS_BB_USER>`, `<ATLANTIS_BB_TOKEN>`, `<ATLANTIS_WEBHOOK_SECRET>` (token/secret stored in secrets tool, not committed).
4. `<STAGING_DOMAIN>` (or a provisional hostname) for `atlantis.<STAGING_DOMAIN>`.
5. Subnet ID for the instance (private preferred) and whether exposure is via **ALB** (recommended) or instance public IP (discouraged).
6. Allowed CIDRs / security-group sources for HTTPS to Atlantis (Bitbucket DC egress IPs, VPN, office — from questionnaire A6).
7. Instance size (default `t3.medium` unless human says otherwise).
8. AMI preference: Amazon Linux 2023 (default) or Ubuntu LTS.
9. Desired Atlantis version (pin from https://github.com/runatlantis/atlantis/releases).
10. Staging and production exec role ARNs from PROMPT-01.

---

## 4. In scope / out of scope

**In scope (Part A — EC2 now):**
- IAM role + instance profile `<ORG>-uat-atlantis` (EC2 trust) with permission to assume both exec roles.
- Trust updates on `<ORG>-uat-terraform-exec` and `<ORG>-prd-terraform-exec` so they accept that Atlantis role.
- EC2 instance, security group, EBS root volume, optional data volume for Atlantis home.
- User-data / cloud-init: Docker (or binary), Atlantis systemd unit, SSM agent.
- ALB + target group + HTTPS listener **or** clear instructions if the human uses an existing reverse proxy.
- Bitbucket DC webhook configuration.
- SSH key for cloning `<MODULES_REPO>` (on the instance, not in git).
- Server-side `repos.yaml` allowlist / apply requirements.
- Runbook notes for migrate-to-EKS later (Part B).

**Out of scope:**
- EKS / Helm / IRSA deployment (Part B only — after cluster transfer).
- Full VPC build (prefer existing subnets; if none exist, ask human before creating a minimal network).
- Production EC2 Atlantis (one Atlantis in staging manages both accounts via assume-role).

---

## 5. Reference material

- `00-standards/decisions.md` ADR-002 (Atlantis — EC2 interim, EKS target).
- `QUESTIONNAIRE-org-context.md` Sections A, D, E, F.
- Atlantis Bitbucket Server docs: https://www.runatlantis.io/docs/configuring-bitbucket-server.html
- Atlantis deployment (generic): https://www.runatlantis.io/docs/deployment.html

---

## 6. Step-by-step procedure

### Step 1: Atlantis IAM role (instance profile) — HCL for `10-identity`

Write `live/staging/10-identity/atlantis.tf` (apply once via local bootstrap/exec role if Atlantis is not up yet; thereafter via Atlantis itself).

```hcl
# live/staging/10-identity/atlantis.tf

data "aws_iam_policy_document" "atlantis_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "atlantis" {
  name               = "<ORG>-uat-atlantis"
  assume_role_policy = data.aws_iam_policy_document.atlantis_trust.json

  tags = {
    Name    = "<ORG>-uat-atlantis"
    Service = "atlantis"
    Hosting = "ec2-interim"
  }
}

resource "aws_iam_instance_profile" "atlantis" {
  name = "<ORG>-uat-atlantis"
  role = aws_iam_role.atlantis.name
}

data "aws_iam_policy_document" "atlantis_assume_exec" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]
    resources = [
      "arn:aws:iam::<STAGING_ACCOUNT_ID>:role/<ORG>-uat-terraform-exec",
      "arn:aws:iam::<PRODUCTION_ACCOUNT_ID>:role/<ORG>-prd-terraform-exec",
    ]
  }
}

resource "aws_iam_role_policy" "atlantis_assume_exec" {
  name   = "assume-exec-roles"
  role   = aws_iam_role.atlantis.id
  policy = data.aws_iam_policy_document.atlantis_assume_exec.json
}

# Optional but recommended: SSM Session Manager (no inbound SSH)
resource "aws_iam_role_policy_attachment" "atlantis_ssm" {
  role       = aws_iam_role.atlantis.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}
```

**Trust on exec roles (PROMPT-01 / bootstrap update):** both exec roles must allow:

```hcl
principals {
  type        = "AWS"
  identifiers = ["arn:aws:iam::<STAGING_ACCOUNT_ID>:role/<ORG>-uat-atlantis"]
}
actions = ["sts:AssumeRole"]
```

Do **not** require EKS OIDC for this interim path. When migrating to EKS later, replace EC2 trust with IRSA (or support both briefly during cutover).

### Step 2: Security group

- Inbound HTTPS `443` from Bitbucket DC (and/or ALB SG only if using ALB).
- Inbound from ALB security group to instance port `4141` if ALB terminates TLS.
- Outbound: HTTPS to Bitbucket, AWS APIs, and whatever is needed to clone modules / download providers.
- Prefer **no** inbound SSH (`22`). Use SSM.

Ask the human for source CIDRs if A6 was answered; otherwise default to “ALB-only + VPN CIDR if provided”.

### Step 3: EC2 instance

Create Terraform under `live/staging/50-compute/` **or** a small dedicated `live/staging/05-atlantis/` root if the human wants Atlantis isolated before the compute layer exists. Prefer `05-atlantis` so Atlantis is not blocked on the full compute layer.

Minimum instance settings:

| Setting | Value |
|---|---|
| Name | `<ORG>-uat-atlantis` |
| Type | `t3.medium` (or human override) |
| AMI | Amazon Linux 2023 |
| IAM instance profile | `<ORG>-uat-atlantis` |
| Subnet | private (preferred) |
| Root volume | ≥ 30 GB gp3 |
| Metadata | IMDSv2 required |
| User data | install Docker + run Atlantis (Step 4) |

Pin AMI via SSM public parameter or a data source — do not hardcode a stale AMI ID without asking.

### Step 4: Install and run Atlantis (user-data / systemd)

Provide cloud-init that:

1. Installs Docker (or downloads the Atlantis binary — Docker image `ghcr.io/runatlantis/atlantis:v<ATLANTIS_VERSION>` is fine).
2. Creates `/etc/atlantis/` with `repos.yaml`.
3. Pulls Bitbucket token + webhook secret from the secrets tool or SSM Parameter Store (never bake into the AMI).
4. Starts a systemd unit or `docker run` with restart policy.

Example `repos.yaml`:

```yaml
repos:
  - id: "<BITBUCKET_BASE_URL>/<BB_PROJECT>/<INFRA_REPO>"
    apply_requirements: [approved, mergeable]
    allowed_overrides: [apply_requirements]
    allow_custom_workflows: false
```

Example container flags / env (adapt to docker run or compose):

```text
--atlantis-url=https://atlantis.<STAGING_DOMAIN>
--bitbucket-user=<ATLANTIS_BB_USER>
--bitbucket-token=<from secret>
--bitbucket-base-url=<BITBUCKET_BASE_URL>
--bitbucket-webhook-secret=<from secret>
--repo-allowlist=<BITBUCKET_BASE_URL>/<BB_PROJECT>/<INFRA_REPO>
--repo-config=/etc/atlantis/repos.yaml
--default-tf-version=1.11.4
```

Also set:

```text
AWS_REGION=<STAGING_REGION>
# Optional: default profile/role chaining is via instance profile → AssumeRole in provider blocks
```

Provider blocks in live roots already use `assume_role` to the exec roles — the instance profile only needs permission to call `sts:AssumeRole` on those ARNs (Step 1).

### Step 5: SSH key for module downloads

On the instance (via SSM), generate or install an ed25519 key used only by Atlantis:

```bash
ssh-keygen -t ed25519 -C "atlantis@<ORG>" -f /home/atlantis/.ssh/id_ed25519 -N ""
```

1. Add the **public** key to the Bitbucket service account SSH keys.
2. Write `known_hosts` for `<BITBUCKET_SSH_HOST>:7999`.
3. Never commit the private key.

### Step 6: Expose HTTPS endpoint

**Preferred:** ALB in front of the instance, ACM cert for `atlantis.<STAGING_DOMAIN>`, target group → instance:4141, health check on `/healthz` (Atlantis health endpoint).

**Fallback:** existing reverse proxy the human names. Document the final URL.

Bitbucket webhook URL:

`https://atlantis.<STAGING_DOMAIN>/events`

### Step 7: Bitbucket DC webhook

Repository Settings → Webhooks → Add webhook:

| Field | Value |
|---|---|
| URL | `https://atlantis.<STAGING_DOMAIN>/events` |
| Secret | `<ATLANTIS_WEBHOOK_SECRET>` |
| Events | PR Opened, PR Modified, PR Merged, PR Comment Created |
| Active | Yes |

Confirm HMAC webhook secret verification is enabled on the Atlantis side.

### Step 8: Verify

```bash
# Via SSM Session Manager on the instance
aws sts get-caller-identity
# Expect: arn:...:assumed-role/<ORG>-uat-atlantis/...

curl -sS https://atlantis.<STAGING_DOMAIN>/healthz
```

Then open a trivial PR in the infra repo and confirm Atlantis posts a plan comment within ~60 seconds.

### Step 9: Stop local applies

Once the test PR plan/apply works, update team process: **all applies go through Atlantis**. Keep a break-glass local apply path only for Atlantis-down emergencies (see day2-runbook).

---

## 7. Expected file tree after completion

```
live/staging/05-atlantis/          # preferred isolated root (or under 50-compute if human prefers)
  backend.tf
  providers.tf
  versions.tf
  locals.tf
  main.tf                          # instance, SG, ALB pieces as agreed
  outputs.tf

live/staging/10-identity/
  atlantis.tf                      # IAM role + instance profile

docs/ or live/staging/05-atlantis/
  atlantis-repos.yaml              # optional committed copy of server repo config
  README-atlantis-ec2.md           # hostname, SSM how-to, migrate-to-EKS notes
```

Also update `atlantis.yaml` projects if a new `05-atlantis` root was added.

---

## 8. Code contracts

- No static `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` on the host.
- Auth = EC2 instance profile `<ORG>-uat-atlantis` → assume exec roles.
- Pin exact Atlantis version.
- Webhook secret required; HTTPS required for the public/Bitbucket-facing URL.
- IMDSv2 required on the instance.
- Tag resources with `Hosting = ec2-interim` so the EKS migration can find them.

---

## 9. Acceptance criteria

- [ ] Instance is running with instance profile `<ORG>-uat-atlantis`.
- [ ] `aws sts get-caller-identity` on the host shows that role.
- [ ] Host can `sts:AssumeRole` into both exec roles.
- [ ] `https://atlantis.<STAGING_DOMAIN>/healthz` returns healthy.
- [ ] Test PR gets an Atlantis plan comment within 60 seconds.
- [ ] `atlantis apply` on an approved PR succeeds for a trivial change.
- [ ] Bitbucket token and webhook secret are not in git.
- [ ] Short migrate-to-EKS note exists (Part B).

---

## 10. Guardrails

- Never commit Bitbucket token, webhook secret, or SSH private key.
- Never attach `AdministratorAccess` to `<ORG>-uat-atlantis` — only `sts:AssumeRole` to the two exec roles (+ SSM).
- Never expose port 4141 to `0.0.0.0/0`; terminate TLS at ALB / controlled CIDRs.
- Never disable autoplan.
- Do not block this prompt on EKS transfer.
- Do not treat the EC2 host as permanent without a written EKS migration note.

---

## 11. Handoff note

When complete, report:

1. Instance ID, type, AMI, Atlantis version.
2. Public URL (`https://atlantis.<STAGING_DOMAIN>`).
3. Role chain: EC2 instance profile `<ORG>-uat-atlantis` → exec roles.
4. Test PR result (plan comment yes/no; apply yes/no).
5. How operators access the host (SSM preferred).
6. Reminder: Part B when EKS arrives.

---

## Part B — Later: migrate Atlantis to EKS (after cluster transfer)

Do **not** execute until `<STAGING_EKS_CLUSTER>`, OIDC ARN/URL exist in `<STAGING_ACCOUNT_ID>`.

1. Create IRSA trust on `<ORG>-uat-atlantis` (or a new role) for `system:serviceaccount:<ATLANTIS_NAMESPACE>:atlantis`.
2. Deploy Atlantis via Helm (chart `runatlantis/atlantis`) with the same Bitbucket settings, `repos.yaml`, and webhook URL (DNS cutover).
3. Point Bitbucket webhook / DNS at the EKS ingress.
4. Confirm plans still work.
5. Stop the EC2 instance, remove ALB target, delete interim compute — keep IAM role if reused for IRSA, or replace trust from `ec2.amazonaws.com` to OIDC.
6. Remove `Hosting = ec2-interim` resources from state after destroy.

Keep webhook secret and bot user unchanged across the migration if possible.
