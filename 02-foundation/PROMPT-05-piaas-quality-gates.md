# PROMPT-05: piaas.yml Quality Gates

## 1. Role and objective

You are a senior SRE writing the `piaas.yml` pipeline configuration for the infra monorepo. This pipeline runs fast, stateless quality gates on every push and PR — formatting check, schema validation, lint, security scan, and docs check. It never runs `terraform plan` or `apply`. Those belong to Atlantis. By the end of this prompt, every PR to the infra repo triggers a passing or failing quality gate in under 5 minutes, giving the team early feedback before Atlantis plans.

---

## 2. Preconditions

- [ ] `QUESTIONNAIRE-org-context.md` Section B (piaas.yml platform) has been answered.
- [ ] PROMPT-02 is complete: the infra repo scaffold exists.
- [ ] PROMPT-04 is complete: Atlantis is running (so the team understands that piaas.yml and Atlantis are separate, complementary systems).
- [ ] You know the exact `piaas.yml` schema version and available runner images from the org context questionnaire.

---

## 3. Required inputs

Ask the human to confirm before writing:

1. `piaas.yml` schema version (`<PIAAS_VERSION>`).
2. Runner image that has Terraform 1.11.x (`<PIAAS_TF_IMAGE>`).
3. Runner image with general tooling, or confirmation that tools can be installed at runtime (`<PIAAS_TOOLS_IMAGE>`).
4. How to trigger on PR open/update (`<PIAAS_PR_TRIGGER>`).
5. How to trigger on push to `main` (`<PIAAS_PUSH_TRIGGER>`).
6. How environment variables are passed to steps (`<PIAAS_ENV_VARS>`).
7. Maximum job timeout (if any).
8. Whether the pipeline runner has internet access to download tflint, trivy, checkov. If not, which internal registry hosts them.

---

## 4. In scope / out of scope

**In scope:**
- `piaas.yml` at the root of the infra repo.
- Jobs: `fmt-check`, `validate`, `tflint`, `trivy`, `checkov`, `terraform-docs-check`.
- Matching `.tflint.hcl` and `trivy` / `checkov` config files.
- Instructions for installing tools at pipeline runtime if no pre-built image is available.

**Out of scope:**
- `terraform plan` or `terraform apply` in `piaas.yml`.
- Any AWS credential configuration in the pipeline.
- Notification integrations (Slack, email) — add those later.

---

## 5. Reference material

- `00-standards/decisions.md` ADR-006 (piaas owns quality gates only).
- `00-standards/conventions.md` section 9 (code quality gates).
- `QUESTIONNAIRE-org-context.md` Section B answers.

---

## 6. Step-by-step procedure

### Step 1: Understand the piaas.yml schema

Before writing any YAML, ask the human to paste a minimal working `piaas.yml` from another repo in the organization. Read it carefully to understand:
- How stages/jobs/steps are declared.
- How runners/images are specified.
- How environment variables are injected.
- How PR vs push triggers differ.

**Do not invent piaas.yml syntax.** Use only syntax confirmed by the human. The template below uses a generic structure — replace it with the actual schema.

### Step 2: Write the `piaas.yml`

Based on the answers from Step 1, write a `piaas.yml` that runs the following jobs. Adapt the exact structure to the actual schema:

```yaml
# piaas.yml — generic template
# REPLACE <PIAAS_*> placeholders with actual schema syntax confirmed by the human

version: <PIAAS_VERSION>

# Trigger on PRs and pushes to main
on:
  <PIAAS_PR_TRIGGER>
  <PIAAS_PUSH_TRIGGER>

jobs:

  # Job 1: Terraform format check
  fmt-check:
    image: <PIAAS_TF_IMAGE>
    steps:
      - name: Check formatting
        run: terraform fmt -check -recursive live/
        # Exits non-zero if any .tf file is not formatted.
        # Fix locally: terraform fmt -recursive live/

  # Job 2: Terraform validate (all roots, no backend)
  validate:
    image: <PIAAS_TF_IMAGE>
    steps:
      - name: Validate all roots
        run: ./scripts/tf-validate.sh
        # Runs 'terraform init -backend=false && terraform validate' in each root.
        # Private module downloads are skipped due to -backend=false.

  # Job 3: TFLint
  tflint:
    image: <PIAAS_TOOLS_IMAGE>
    steps:
      - name: Install tflint
        run: |
          curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
          tflint --init
      - name: Run tflint
        run: tflint --recursive --config=.tflint.hcl
        # Uses .tflint.hcl in the repo root

  # Job 4: Trivy security scan
  trivy:
    image: <PIAAS_TOOLS_IMAGE>
    steps:
      - name: Install trivy
        run: |
          curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
      - name: Run trivy config scan
        run: |
          trivy config \
            --exit-code 1 \
            --severity HIGH,CRITICAL \
            --ignorefile .trivyignore \
            live/
        # Create .trivyignore to suppress known false positives

  # Job 5: Checkov compliance
  checkov:
    image: <PIAAS_TOOLS_IMAGE>
    steps:
      - name: Install checkov
        run: pip install checkov
      - name: Run checkov
        run: |
          checkov \
            --directory live/ \
            --compact \
            --quiet \
            --framework terraform \
            --skip-check <COMMA_SEPARATED_SKIPPED_CHECKS>
            # Ask human for initial skip list — some checks will be false positives.
            # Document every skipped check with a reason in .checkov.yaml

  # Job 6: terraform-docs check
  terraform-docs-check:
    image: <PIAAS_TOOLS_IMAGE>
    steps:
      - name: Install terraform-docs
        run: |
          curl -sSLo /tmp/terraform-docs.tar.gz \
            https://github.com/terraform-docs/terraform-docs/releases/download/v0.18.0/terraform-docs-v0.18.0-linux-amd64.tar.gz
          tar -xzf /tmp/terraform-docs.tar.gz -C /usr/local/bin/
      - name: Check README is up to date
        run: |
          terraform-docs --config terraform-docs.yaml .
          if ! git diff --exit-code; then
            echo "ERROR: terraform-docs generated changes. Run 'terraform-docs .' locally and commit the updated README."
            exit 1
          fi
```

### Step 3: Tool configuration files

**`.trivyignore`** (suppress known false positives — start empty, add entries as needed):

```
# Trivy false positive suppressions
# Format: <CVE-ID or check-id>  # reason: <explanation>
```

**`.checkov.yaml`** (repository-level Checkov config):

```yaml
# .checkov.yaml
framework:
  - terraform
directory:
  - live/
compact: true
quiet: true
# Add skip-check entries with documented reasons:
# skip-check:
#   - CKV_AWS_18   # S3 access logging: deferred, tracked in issue #42
#   - CKV_AWS_144  # S3 cross-region replication: not required for state buckets
```

**`.tflint.hcl`** was already created in PROMPT-02 — reference it here.

### Step 4: Validate that `tf-validate.sh` works without backend

The script uses `-backend=false` so it never touches S3 or AWS. Confirm this works in the pipeline by running it locally:

```bash
./scripts/tf-validate.sh
```

All roots should pass `terraform validate`. If any root fails due to missing provider configs (e.g. the vault or datadog provider being declared but not installed), add `-upgrade=false` to the `terraform init` call or pin the provider in the root's `versions.tf` before the pipeline runs.

### Step 5: Configure pipeline to fail the PR

Ensure the pipeline is configured to:
1. Block PR merge on pipeline failure (ask the human how to configure this in Bitbucket DC — it is typically a "Required build" or "Mandatory check" setting on the repo).
2. Show pass/fail status on the PR page.

This is separate from Atlantis — Atlantis plans separately and posts its own comment.

---

## 7. Expected file tree after completion

```
<INFRA_REPO>/
├── piaas.yml
├── .tflint.hcl             (from PROMPT-02, referenced here)
├── .trivyignore
├── .checkov.yaml
├── terraform-docs.yaml
└── scripts/
    └── tf-validate.sh      (from PROMPT-02, used here)
```

---

## 8. Code contracts

The pipeline must NOT:
- Configure any AWS credentials.
- Run `terraform init` with a real backend.
- Run `terraform plan` or `terraform apply`.
- Use `--auto-approve`.

The pipeline MUST:
- Exit non-zero on any quality gate failure.
- Run in under 10 minutes total.
- Use only the pinned tool versions.

---

## 9. Acceptance criteria

- [ ] `piaas.yml` is syntactically valid per the platform schema (the human confirms it passes the platform's own lint).
- [ ] A test PR shows all 6 jobs passing in the pipeline UI.
- [ ] Introducing a formatting error (`terraform fmt` leaves a diff) causes the `fmt-check` job to fail.
- [ ] Introducing an invalid HCL resource type causes `validate` to fail.
- [ ] Pipeline completes in under 10 minutes.

---

## 10. Guardrails

- Never add `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, or similar to the pipeline config.
- Never skip the `fmt-check` step — formatting is non-negotiable.
- Do not silence all checkov or trivy checks — every suppressed check needs a documented reason.
- Do not reference `.terraform/` or `.terraform.lock.hcl` as artifacts — they are generated at runtime during validate and should not be cached between runs naively.

---

## 11. Handoff note

When complete, report:
1. The exact piaas.yml schema version used.
2. Tool versions pinned (tflint, trivy, checkov, terraform-docs).
3. Any checkov or trivy checks suppressed, with reasons.
4. Confirmed pipeline pass on a real test PR.
5. Any limitations of the pipeline runner that required workarounds (e.g. no internet access → used internal registry at `<url>`).
