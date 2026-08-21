# PROMPT-04: Atlantis on Bitbucket Data Center

> **Deferred until staging EKS is in this account.** The cluster currently lives elsewhere and will be transferred later. Do not run this prompt until `<STAGING_EKS_CLUSTER>`, `<STAGING_OIDC_ARN>`, and `<STAGING_OIDC_URL>` are discoverable in `<STAGING_ACCOUNT_ID>` and `kubectl` works against that cluster. Bootstrap, scaffold, modules, and non-EKS layers proceed without Atlantis (interim local applies allowed).

## 1. Role and objective

You are a senior SRE deploying Atlantis onto the existing staging EKS cluster using Helm and IRSA (IAM Roles for Service Accounts). Atlantis will use Bitbucket Data Center webhooks to receive PR events, post plan output as PR comments, and apply Terraform changes when authorized. After this prompt is complete, the team can manage all Terraform through PR comments — no one runs `terraform apply` locally ever again.

---

## 2. Preconditions

- [ ] PROMPT-01 is complete: the `<ORG>-uat-terraform-exec` and `<ORG>-prd-terraform-exec` IAM roles exist.
- [ ] PROMPT-02 is complete: `atlantis.yaml` is in the infra repo.
- [ ] `QUESTIONNAIRE-org-context.md` Sections A (Bitbucket DC) and D (identity) have been answered.
- [ ] `fill-in-the-blanks.local.md` has: `<STAGING_EKS_CLUSTER>`, `<STAGING_OIDC_ARN>`, `<STAGING_OIDC_URL>`, `<ATLANTIS_NAMESPACE>`, `<ATLANTIS_WEBHOOK_SECRET>`, `<ATLANTIS_BB_USER>`, `<ATLANTIS_BB_TOKEN>`, `<BITBUCKET_BASE_URL>`.
- [ ] `kubectl` is configured to access the staging EKS cluster.
- [ ] Helm 3 is installed locally.

---

## 3. Required inputs

1. `<STAGING_EKS_CLUSTER>` — EKS cluster name.
2. `<STAGING_OIDC_ARN>` and `<STAGING_OIDC_URL>` — for IRSA trust.
3. `<ATLANTIS_NAMESPACE>` — Kubernetes namespace (e.g. `atlantis`).
4. `<ATLANTIS_BB_USER>` — Bitbucket service account username.
5. `<ATLANTIS_BB_TOKEN>` — Bitbucket HTTP access token (stored in secrets tool, not committed).
6. `<ATLANTIS_WEBHOOK_SECRET>` — pre-shared secret for webhook validation (stored in secrets tool).
7. `<BITBUCKET_BASE_URL>` — e.g. `https://bitbucket.acme.com`.
8. `<INFRA_REPO>` slug and Bitbucket project key `<BB_PROJECT>`.
9. The staging execution role ARN from PROMPT-01 output.
10. The production execution role ARN from PROMPT-01 output.
11. Desired Atlantis image version (check https://github.com/runatlantis/atlantis/releases for latest stable).

---

## 4. In scope / out of scope

**In scope:**
- IRSA IAM role for the Atlantis pod (`<ORG>-uat-atlantis`).
- Kubernetes namespace, ServiceAccount, and IRSA annotation.
- Atlantis Helm chart values file.
- Kubernetes Secret for Bitbucket token and webhook secret.
- Bitbucket DC webhook configuration instructions.
- Cross-account role chaining: Atlantis assumes `<ORG>-uat-atlantis` (via IRSA), then assumes `<ORG>-uat-terraform-exec` and `<ORG>-prd-terraform-exec`.
- Atlantis server-side configuration for `repos.yaml` (repo allowlist, workflow overrides).
- Instructions for verifying the webhook.

**Out of scope:**
- ALB/Ingress for Atlantis (the human must expose Atlantis via the existing ingress controller or a LoadBalancer service — ask which one).
- TLS certificate (covered in 30-dns layer).
- Terraform code for the Atlantis IAM role — this is added to `live/staging/10-identity/` in PROMPT-10. Provide the HCL snippet to paste there.

---

## 5. Reference material

- `00-standards/decisions.md` ADR-002 (Atlantis), ADR-003 (repo split).
- `QUESTIONNAIRE-org-context.md` answers for Sections A, D, E.
- Atlantis docs: https://www.runatlantis.io/docs/configuring-bitbucket-server.html

---

## 6. Step-by-step procedure

### Step 1: IRSA IAM role (Terraform HCL snippet)

This snippet goes into `live/staging/10-identity/atlantis.tf` and is applied via Atlantis itself (or manually during setup). Provide it to the human now.

```hcl
# live/staging/10-identity/atlantis.tf

data "aws_iam_policy_document" "atlantis_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = ["<STAGING_OIDC_ARN>"]
    }
    condition {
      test     = "StringEquals"
      variable = "<STAGING_OIDC_URL>:sub"
      values   = ["system:serviceaccount:<ATLANTIS_NAMESPACE>:atlantis"]
    }
    condition {
      test     = "StringEquals"
      variable = "<STAGING_OIDC_URL>:aud"
      values   = ["sts.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "atlantis" {
  name               = "<ORG>-uat-atlantis"
  assume_role_policy = data.aws_iam_policy_document.atlantis_trust.json

  tags = {
    Name    = "<ORG>-uat-atlantis"
    Service = "atlantis"
  }
}

# Allow Atlantis to assume the exec roles in each account
data "aws_iam_policy_document" "atlantis_assume_exec" {
  statement {
    effect    = "Allow"
    actions   = ["sts:AssumeRole"]
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
```

**Important:** The `<ORG>-uat-terraform-exec` and `<ORG>-prd-terraform-exec` roles must trust the Atlantis role in their trust policies (set up in PROMPT-01).

### Step 2: Kubernetes namespace and ServiceAccount

```bash
kubectl create namespace <ATLANTIS_NAMESPACE> --dry-run=client -o yaml | kubectl apply -f -
```

ServiceAccount manifest (do not apply manually — let Helm manage it):

```yaml
# Shown for reference; Helm chart creates this
apiVersion: v1
kind: ServiceAccount
metadata:
  name: atlantis
  namespace: <ATLANTIS_NAMESPACE>
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<STAGING_ACCOUNT_ID>:role/<ORG>-uat-atlantis
```

### Step 3: Kubernetes secrets

Store sensitive values as Kubernetes secrets (populated from the secrets tool, not hardcoded):

```bash
kubectl create secret generic atlantis-secrets \
  --namespace <ATLANTIS_NAMESPACE> \
  --from-literal=ATLANTIS_BITBUCKET_TOKEN="<ATLANTIS_BB_TOKEN>" \
  --from-literal=ATLANTIS_BITBUCKET_WEBHOOK_SECRET="<ATLANTIS_WEBHOOK_SECRET>" \
  --dry-run=client -o yaml | kubectl apply -f -
```

In a real deployment, use an external secrets operator (ESO) or sealed-secrets to manage this secret — do not create it manually. Provide the human with a placeholder and ask them to use their secrets tool integration.

### Step 4: Helm values file

Create `live/staging/10-identity/atlantis-helm-values.yaml` (not Terraform — this is a Helm values file for direct `helm upgrade` or an ArgoCD/Atlantis-managed app):

```yaml
# atlantis-helm-values.yaml
# Atlantis Helm chart: https://github.com/runatlantis/helm-charts

replicaCount: 1

image:
  repository: ghcr.io/runatlantis/atlantis
  tag: v<ATLANTIS_VERSION>   # pin exact version
  pullPolicy: IfNotPresent

serviceAccount:
  create: true
  name: atlantis
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::<STAGING_ACCOUNT_ID>:role/<ORG>-uat-atlantis"

service:
  type: ClusterIP    # expose via Ingress below, not LoadBalancer
  port: 4141

ingress:
  enabled: true
  ingressClassName: <INGRESS_CLASS>   # e.g. alb, nginx — ask human
  annotations:
    # ALB ingress annotations if using AWS ALB controller:
    # kubernetes.io/ingress.class: alb
    # alb.ingress.kubernetes.io/scheme: internet-facing
    # alb.ingress.kubernetes.io/target-type: ip
  hosts:
    - host: atlantis.<STAGING_DOMAIN>
      paths:
        - path: /
          pathType: Prefix

atlantisUrl: "https://atlantis.<STAGING_DOMAIN>"

orgAllowlist: "<BITBUCKET_BASE_URL>/<BB_PROJECT>/<INFRA_REPO>"

bitbucketServer:
  user: "<ATLANTIS_BB_USER>"
  baseURL: "<BITBUCKET_BASE_URL>"

# All secrets from the Kubernetes secret created in Step 3
environmentSecrets:
  - name: ATLANTIS_BITBUCKET_TOKEN
    secretKeyRef:
      name: atlantis-secrets
      key: ATLANTIS_BITBUCKET_TOKEN
  - name: ATLANTIS_BITBUCKET_WEBHOOK_SECRET
    secretKeyRef:
      name: atlantis-secrets
      key: ATLANTIS_BITBUCKET_WEBHOOK_SECRET

environment:
  - name: ATLANTIS_DEFAULT_TF_VERSION
    value: "1.11.4"
  - name: AWS_REGION
    value: "<STAGING_REGION>"

# Mount SSH key for module downloads from Bitbucket DC
extraVolumes:
  - name: atlantis-ssh
    secret:
      secretName: atlantis-ssh-key
extraVolumeMounts:
  - name: atlantis-ssh
    mountPath: /home/atlantis/.ssh
    readOnly: true

resources:
  requests:
    memory: "512Mi"
    cpu: "200m"
  limits:
    memory: "1Gi"
    cpu: "500m"

storage:
  enabled: true
  storageClassName: gp3   # adjust to available StorageClass
  size: 5Gi

repoConfig: |
  repos:
    - id: "<BITBUCKET_BASE_URL>/<BB_PROJECT>/<INFRA_REPO>"
      apply_requirements: [approved, mergeable]
      allowed_overrides: [apply_requirements]
      allow_custom_workflows: false
```

### Step 5: SSH key for module downloads

Atlantis needs to clone the module library repo via SSH during `terraform init`. Create a dedicated SSH key pair for the Atlantis service account:

```bash
ssh-keygen -t ed25519 -C "atlantis@<ORG>" -f atlantis_bitbucket_ed25519 -N ""
```

1. Add the **public key** to the Bitbucket service account's SSH keys.
2. Create a Kubernetes secret from the **private key**:

```bash
kubectl create secret generic atlantis-ssh-key \
  --namespace <ATLANTIS_NAMESPACE> \
  --from-file=id_ed25519=./atlantis_bitbucket_ed25519 \
  --from-literal=known_hosts="$(ssh-keyscan -p 7999 <BITBUCKET_SSH_HOST> 2>/dev/null)" \
  --dry-run=client -o yaml | kubectl apply -f -
```

3. Delete the local private key file: `rm atlantis_bitbucket_ed25519`.

### Step 6: Install Atlantis via Helm

```bash
helm repo add runatlantis https://runatlantis.github.io/helm-charts
helm repo update

helm upgrade --install <ATLANTIS_RELEASE> runatlantis/atlantis \
  --namespace <ATLANTIS_NAMESPACE> \
  --create-namespace \
  --version <ATLANTIS_CHART_VERSION> \
  --values live/staging/10-identity/atlantis-helm-values.yaml \
  --wait
```

Check latest chart version at https://github.com/runatlantis/helm-charts/releases.

### Step 7: Configure Bitbucket DC webhook

In Bitbucket DC, navigate to: Repository Settings → Webhooks → Add webhook.

| Field | Value |
|---|---|
| URL | `https://atlantis.<STAGING_DOMAIN>/events` |
| Secret | `<ATLANTIS_WEBHOOK_SECRET>` |
| Events | PR Opened, PR Modified, PR Merged, PR Comment Created |
| Active | Yes |

Bitbucket Data Center supports webhook secrets (HMAC-SHA256 signature on the `X-Hub-Signature` header) — confirm this is verified by the Atlantis `--bitbucket-webhook-secret` flag (set via the `ATLANTIS_BITBUCKET_WEBHOOK_SECRET` environment variable).

### Step 8: Verify the setup

```bash
# Confirm the pod is running
kubectl get pods -n <ATLANTIS_NAMESPACE>

# Check Atlantis logs
kubectl logs -n <ATLANTIS_NAMESPACE> -l app.kubernetes.io/name=atlantis --tail=50

# Confirm IRSA is working (Atlantis pod can call AWS)
kubectl exec -n <ATLANTIS_NAMESPACE> \
  $(kubectl get pod -n <ATLANTIS_NAMESPACE> -l app.kubernetes.io/name=atlantis -o name) \
  -- aws sts get-caller-identity
```

The caller identity should show the `<ORG>-uat-atlantis` role.

### Step 9: Test with a real PR

1. Create a branch in the infra repo.
2. Add a trivial comment to `live/staging/10-identity/main.tf`.
3. Open a PR in Bitbucket DC.
4. Confirm Atlantis posts a plan comment within ~30 seconds.
5. If no comment: check Atlantis logs for webhook delivery errors.

---

## 7. Expected file tree after completion

```
live/staging/10-identity/
  atlantis.tf                       (IAM role snippet)
  atlantis-helm-values.yaml         (Helm values, committed)

(Kubernetes resources managed by Helm — not committed as YAML manifests)
```

---

## 8. Code contracts

The Atlantis pod must:
- Use IRSA (not static AWS credentials).
- Assume `<ORG>-uat-atlantis`, which then assumes `<ORG>-uat-terraform-exec` or `<ORG>-prd-terraform-exec` per project.
- Pin an exact Atlantis version tag.
- Validate webhook signatures via `ATLANTIS_BITBUCKET_WEBHOOK_SECRET`.

---

## 9. Acceptance criteria

- [ ] `kubectl get pods -n <ATLANTIS_NAMESPACE>` shows `Running` and `Ready`.
- [ ] `aws sts get-caller-identity` from inside the Atlantis pod returns the `<ORG>-uat-atlantis` role.
- [ ] Opening a test PR triggers a plan comment from Atlantis within 60 seconds.
- [ ] `atlantis plan` typed as a PR comment re-plans successfully.
- [ ] `atlantis apply` typed as a PR comment (by an authorized user on an approved PR) applies successfully.
- [ ] Atlantis logs show no authentication or webhook signature errors.

---

## 10. Guardrails

- Never commit the Bitbucket token or webhook secret to the repo.
- Never use `--disable-autoplan` in Atlantis — autoplan is the primary safety net.
- Never grant Atlantis a direct admin role — it must chain through the exec roles.
- Never expose Atlantis to the public internet without the webhook secret and HTTPS.
- Do not allow the Atlantis service account to assume arbitrary roles — only the two exec roles.

---

## 11. Handoff note

When complete, report:
1. Atlantis pod name and image version.
2. URL at which Atlantis is reachable.
3. Role chain confirmed: Atlantis pod → `<ORG>-uat-atlantis` → `<ORG>-uat-terraform-exec`.
4. Result of the test PR (plan comment received yes/no).
5. Any deviations (e.g. ingress class used, chart version pinned).
