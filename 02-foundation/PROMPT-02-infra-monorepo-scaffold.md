# PROMPT-02: Infrastructure Monorepo Scaffold

## 1. Role and objective

You are a senior SRE creating the full scaffold of the `<INFRA_REPO>` infrastructure monorepo. You will set up the repository-level configuration: `atlantis.yaml` (one project per layer), a `.terraform-version` file, pre-commit hooks, a validation script, a `.gitignore`, and placeholder directories for all layers with their `README.md` files. You do not write any Terraform resources inside the layers — those are covered by PROMPT-10 through PROMPT-17.

---

## 2. Preconditions

- [ ] PROMPT-01 is complete: S3 state buckets, KMS keys, and exec roles exist.
- [ ] `00-standards/conventions.md` and `00-standards/decisions.md` have been read.
- [ ] `fill-in-the-blanks.local.md` has values for all non-`[DISCOVERY]` fields.
- [ ] The infra repo exists in Bitbucket DC and is cloned locally.

---

## 3. Required inputs

1. `<ORG>`, `<INFRA_REPO>`
2. `<STAGING_ACCOUNT_ID>`, `<STAGING_REGION>`
3. `<PRODUCTION_ACCOUNT_ID>`, `<PRODUCTION_REGION>`
4. State bucket names from PROMPT-01 (e.g. `<ORG>-tfstate-uat-<STAGING_REGION>`)
5. Exec role ARNs from PROMPT-01
6. `<ATLANTIS_WEBHOOK_SECRET>` — a pre-generated secret (from the secrets tool)

---

## 4. In scope / out of scope

**In scope:**
- Repository root files: `atlantis.yaml`, `.terraform-version`, `.gitignore`, `README.md`, `BOOTSTRAP.md`.
- `scripts/tf-validate.sh` — iterates all root modules and runs `terraform init -backend=false && terraform validate`.
- `.pre-commit-config.yaml` with terraform fmt, tflint, terraform-docs, detect-secrets.
- Placeholder `live/<env>/<layer>/` directories with a minimal `README.md` and empty `main.tf` stub.
- `atlantis.yaml` with one project per layer per environment.

**Out of scope:**
- Actual Terraform resources (handled in PROMPT-10 through PROMPT-17).
- Atlantis EC2 deployment (PROMPT-04 Part A); EKS migration is Part B.
- `piaas.yml` (PROMPT-05).

---

## 5. Reference material

- `00-standards/conventions.md` — section 1 (layout), section 8 (commit messages).
- `00-standards/decisions.md` — ADR-002 (Atlantis), ADR-005 (directory layout), ADR-006 (piaas).

---

## 6. Step-by-step procedure

### Step 1: `.terraform-version`

```
1.11.4
```

Use the latest patch release of Terraform 1.11 available at the time of writing. Check https://releases.hashicorp.com/terraform/ if unsure.

### Step 2: `.gitignore`

```gitignore
# Terraform
.terraform/
*.tfplan
*.tfstate
*.tfstate.backup
*.auto.tfvars
*.secret.tfvars
crash.log
crash.*.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json
.terraformrc
terraform.rc

# Local values
00-standards/fill-in-the-blanks.local.md

# OS
.DS_Store
Thumbs.db

# Editor
.idea/
.vscode/
*.swp
```

**Important:** `.terraform.lock.hcl` is NOT in `.gitignore`. It must be committed.

### Step 3: `atlantis.yaml`

Generate one project per layer per environment. Use `autoplan` anchors for DRY config:

```yaml
version: 3

projects:
  - name: staging-00-bootstrap
    dir: live/staging/00-bootstrap
    autoplan: &autoplan
      enabled: true
      when_modified:
        - "*.tf"
        - "*.tfvars"
        - ".terraform.lock.hcl"
    apply_requirements:
      - approved
      - mergeable

  - name: staging-10-identity
    dir: live/staging/10-identity
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: staging-20-network
    dir: live/staging/20-network
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: staging-30-dns
    dir: live/staging/30-dns
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: staging-40-data
    dir: live/staging/40-data
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: staging-50-compute
    dir: live/staging/50-compute
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: staging-60-storage
    dir: live/staging/60-storage
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: staging-70-observability
    dir: live/staging/70-observability
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  # Production projects — require stricter approval
  - name: production-00-bootstrap
    dir: live/production/00-bootstrap
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: production-10-identity
    dir: live/production/10-identity
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: production-20-network
    dir: live/production/20-network
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: production-30-dns
    dir: live/production/30-dns
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: production-40-data
    dir: live/production/40-data
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: production-50-compute
    dir: live/production/50-compute
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: production-60-storage
    dir: live/production/60-storage
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable

  - name: production-70-observability
    dir: live/production/70-observability
    autoplan: *autoplan
    apply_requirements:
      - approved
      - mergeable
```

`80-eks` projects are NOT added here — they are added after the EKS questionnaire is answered.

### Step 4: `scripts/tf-validate.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
ERRORS=0

for dir in "$ROOT_DIR"/live/*/*; do
  if [ -d "$dir" ] && ls "$dir"/*.tf >/dev/null 2>&1; then
    echo "Validating $dir..."
    (
      cd "$dir"
      terraform init -backend=false -input=false -no-color >/dev/null
      terraform validate -no-color
    ) || ERRORS=$((ERRORS + 1))
  fi
done

if [ "$ERRORS" -gt 0 ]; then
  echo "FAILED: $ERRORS directory/directories failed validation."
  exit 1
fi

echo "All modules validated successfully."
```

Make it executable: `chmod +x scripts/tf-validate.sh`.

### Step 5: `.pre-commit-config.yaml`

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-merge-conflict
      - id: detect-private-key

  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.1
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
        args:
          - --args=-no-color
          - --hook-config=--retry-once-with-cleanup=true
        additional_dependencies:
          - "terraform>=1.11.0"
      - id: terraform_tflint
        args:
          - --args=--config=__GIT_WORKING_DIR__/.tflint.hcl
      - id: terraform_docs
        args:
          - --hook-config=--path-to-file=README.md
          - --hook-config=--add-to-existing-file=true
          - --hook-config=--create-file-if-not-exist=true
      - id: terraform_checkov
        args:
          - --args=--quiet
          - --args=--compact

  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
```

### Step 6: `.tflint.hcl`

```hcl
plugin "aws" {
  enabled = true
  version = "0.37.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_naming_convention" {
  enabled = true
}

rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}

rule "terraform_required_version" {
  enabled = true
}

rule "terraform_required_providers" {
  enabled = true
}
```

### Step 7: Placeholder layer directories

For each layer in `live/staging/` and `live/production/` (excluding `00-bootstrap` which already exists from PROMPT-01), create:

- `main.tf` (empty stub with a comment)
- `README.md` pointing to the relevant PROMPT

Example for `live/staging/10-identity/`:

```hcl
# live/staging/10-identity/main.tf
# Populated by PROMPT-10-identity-iam.md
```

```markdown
<!-- live/staging/10-identity/README.md -->
# 10-identity (staging)

Manages IAM roles, policies, OIDC providers, and SSO assignments for the staging account.

See `aws-terraform-plan/03-layers/PROMPT-10-identity-iam.md` for implementation instructions.

<!-- BEGIN_TF_DOCS -->
<!-- END_TF_DOCS -->
```

The `<!-- BEGIN_TF_DOCS -->` / `<!-- END_TF_DOCS -->` markers let terraform-docs auto-fill the section.

Do this for all 8 numbered layers (10 through 80) in both environments.

### Step 8: Root `README.md`

Write a concise README at the repo root:

```markdown
# <INFRA_REPO>

Infrastructure-as-Code monorepo for <ORG> AWS accounts.

## Structure

live/
  staging/      — staging account (<STAGING_ACCOUNT_ID>), <STAGING_REGION>
  production/   — production account (<PRODUCTION_ACCOUNT_ID>), <PRODUCTION_REGION>

## Workflow

1. Create a branch, make changes.
2. Open a PR in Bitbucket.
3. Atlantis auto-plans the affected layers and posts results as PR comments.
4. Get PR approved (minimum 1 reviewer).
5. Type `atlantis apply` in the PR comment to apply.
6. Merge the PR.

## First-time setup

See BOOTSTRAP.md for the one-time state bucket provisioning procedure.

## Conventions

See aws-terraform-plan/00-standards/conventions.md (internal reference pack).

## Adding a new layer

1. Create the directory under live/<env>/<layer>/.
2. Add the project to atlantis.yaml.
3. Write backend.tf, versions.tf, providers.tf, locals.tf, data.tf, main.tf, outputs.tf.
4. Run `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64` and commit .terraform.lock.hcl.
5. Open a PR.
```

---

## 7. Expected file tree after completion

```
<INFRA_REPO>/
├── .gitignore
├── .terraform-version
├── .tflint.hcl
├── .pre-commit-config.yaml
├── atlantis.yaml
├── BOOTSTRAP.md
├── README.md
├── scripts/
│   └── tf-validate.sh
├── docs/
│   └── adr/              (empty, ADRs go here in future)
└── live/
    ├── staging/
    │   ├── 00-bootstrap/  (from PROMPT-01)
    │   ├── 10-identity/   (placeholder)
    │   ├── 20-network/    (placeholder)
    │   ├── 30-dns/        (placeholder)
    │   ├── 40-data/       (placeholder)
    │   ├── 50-compute/    (placeholder)
    │   ├── 60-storage/    (placeholder)
    │   ├── 70-observability/ (placeholder)
    │   └── 80-eks/        (placeholder — see EKS questionnaire)
    └── production/
        └── (same structure)
```

---

## 8. Code contracts

The `atlantis.yaml` must use `apply_requirements: [approved, mergeable]` on every project. No project may have `autoplan.enabled: false` unless explicitly discussed with the human.

---

## 9. Acceptance criteria

- [ ] `scripts/tf-validate.sh` exits 0 when run locally against all placeholder stubs.
- [ ] `atlantis.yaml` contains exactly 16 projects (8 layers × 2 environments; `80-eks` excluded).
- [ ] `.terraform.lock.hcl` exists in `live/staging/00-bootstrap/` and `live/production/00-bootstrap/` (from PROMPT-01) — verify it has not been deleted.
- [ ] `pre-commit run --all-files` passes after installing hooks (`pre-commit install`).
- [ ] No `.auto.tfvars` or `*.secret.tfvars` files present.

---

## 10. Guardrails

- Do not add provider credentials or account IDs to `atlantis.yaml`.
- Do not add the `80-eks` projects to `atlantis.yaml` until the EKS questionnaire is answered.
- Do not create `.terraform/` directories — they are gitignored and generated by `terraform init`.
- Every placeholder `main.tf` must be valid HCL (a comment is valid).

---

## 11. Handoff note

When complete, report:
1. List of all files created.
2. The exact number of Atlantis projects in `atlantis.yaml`.
3. Confirm `tf-validate.sh` passes.
4. Any files where you deviated from the template (e.g. adjusted pre-commit hook versions).
