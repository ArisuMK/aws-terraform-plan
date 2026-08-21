# AWS Terraform Bootstrap Prompt Pack

A structured set of instruction files for building a production-grade Terraform monorepo on AWS, governed by Atlantis on Bitbucket Data Center. Each file in this pack is a precise prompt or questionnaire that you feed to an AI agent to execute a specific phase of the work.

---

## How to use this pack

1. **Read the standards first.** Before running any prompt, read all three files in `00-standards/`. The executing AI must also read them at the start of each session.

2. **Fill in the blanks.** Copy `00-standards/fill-in-the-blanks.md` to `00-standards/fill-in-the-blanks.local.md` (gitignored). Fill in every placeholder. Some values come from `PROMPT-00` (marked `[DISCOVERY]`).

3. **Run prompts in order.** Each prompt declares its preconditions. Do not skip ahead. Prompts in the same phase can be run in parallel by separate AI sessions if they have no dependency on each other.

4. **Feed a prompt to the AI.** Open a new AI chat session. Paste the entire contents of the `PROMPT-*.md` file as your first message, then follow the "Required inputs" section to provide your specific values. The AI will ask for any remaining inputs before starting.

5. **Answer questionnaires yourself.** Files named `QUESTIONNAIRE-*.md` are not for the AI — they are for you. Read them, gather the answers, and record them before running the dependent prompts.

6. **Commit `.terraform.lock.hcl`.** After each `terraform init` inside a new root module, the AI must generate and commit the lock file for `linux_amd64` and `darwin_arm64`.

---

## Execution order

```
Phase 0 — Read & Fill
  00-standards/conventions.md          [YOU READ]
  00-standards/decisions.md            [YOU READ]
  00-standards/fill-in-the-blanks.md   [YOU FILL]

Phase 1 — Discovery
  PROMPT-00-inventory-existing-aws     [AI executes] → fills [DISCOVERY] values

Phase 2 — Org context (questionnaire)
  QUESTIONNAIRE-org-context            [YOU ANSWER] → required for PROMPT-04, PROMPT-05, PROMPT-16

Phase 3 — Foundation (run in order; 02 and 03 can run in parallel after 01)
  PROMPT-01-bootstrap-state-backend    [AI executes]
  PROMPT-02-infra-monorepo-scaffold    [AI executes]  ─┐ parallel
  PROMPT-03-module-library-repo        [AI executes]  ─┘
  PROMPT-04-atlantis-on-bitbucket-dc   [AI executes]  (needs QUESTIONNAIRE-org-context)
  PROMPT-05-piaas-quality-gates        [AI executes]  (needs QUESTIONNAIRE-org-context)

Phase 4 — Infrastructure layers (can be parallelized within an environment; staging before production)
  PROMPT-10-identity-iam               [AI executes]
  PROMPT-11-network-vpc                [AI executes]  (needs PROMPT-10 complete)
  PROMPT-12-dns-route53-acm            [AI executes]  (needs PROMPT-11 complete)
  PROMPT-13-data-documentdb            [AI executes]  (needs PROMPT-11 complete)
  PROMPT-14-compute-ec2-alb            [AI executes]  (needs PROMPT-11 complete)
  PROMPT-15-storage-s3-ecr             [AI executes]  (needs PROMPT-10 complete)
  PROMPT-16-secrets-integration        [AI executes]  (needs QUESTIONNAIRE-org-context)
  PROMPT-17-observability-audit        [AI executes]  (needs PROMPT-10 complete)

Phase 5 — EKS (deferred)
  QUESTIONNAIRE-20-eks-handover        [YOU ANSWER]
  PROMPT-20-eks-layer                  [AI writes — generated from questionnaire answers]

Phase 6 — Operations
  04-operations/import-playbook        [YOU READ before any import work]
  04-operations/review-checklist       [AI and YOU use on every PR]
  04-operations/day2-runbook           [YOU READ after initial build-out]
```

---

## Dependency diagram

```
PROMPT-00 (inventory)
    │
    ├──► PROMPT-01 (bootstrap) ──► PROMPT-02 (infra scaffold) ──► PROMPT-10 (IAM)
    │                                                              │
    │                                                              ├──► PROMPT-11 (network)
    │                                                              │        │
    │                                                              │        ├──► PROMPT-12 (dns)
    │                                                              │        ├──► PROMPT-13 (data/docdb)
    │                                                              │        └──► PROMPT-14 (compute)
    │                                                              │
    │                                                              ├──► PROMPT-15 (storage)
    │                                                              └──► PROMPT-17 (observability)
    │
    ├──► PROMPT-03 (module library)
    │
QUESTIONNAIRE-org-context
    │
    ├──► PROMPT-04 (atlantis)
    ├──► PROMPT-05 (piaas)
    └──► PROMPT-16 (secrets)

QUESTIONNAIRE-20-eks-handover
    └──► PROMPT-20 (eks layer — generated)
```

---

## Prompt anatomy

Every `PROMPT-*.md` follows the same eleven-section structure. When feeding a prompt to an AI, expect it to:

1. **Role and objective** — confirms what it is building and for whom.
2. **Preconditions** — lists what must already be done. The AI will ask you to confirm these before starting.
3. **Required inputs** — explicit list of values the AI will ask you to provide (from `fill-in-the-blanks.local.md`).
4. **In scope / out of scope** — hard boundaries. The AI must not drift outside scope.
5. **Reference material** — which convention docs to read before generating code.
6. **Step-by-step procedure** — ordered tasks.
7. **Expected file tree** — exact output structure.
8. **Code contracts** — exact HCL blocks (backend, provider, tagging) to use verbatim.
9. **Acceptance criteria** — verifiable conditions (e.g. `terraform validate` passes, plan is clean).
10. **Guardrails** — things the AI must never do.
11. **Handoff note** — what to report back when done.

---

## Guardrails that apply to all prompts

These are repeated in every prompt, but listed here for reference:

- Never run `terraform apply` directly. All applies go through Atlantis.
- Never run `terraform import` CLI. Use declarative `import {}` blocks only.
- Never hardcode account IDs, CIDRs, or secrets in `.tf` files. They go in `locals.tf` (from data sources) or the secrets tool.
- Never commit files ending in `.secret.tfvars` or `*.auto.tfvars`.
- Never skip committing `.terraform.lock.hcl`.
- Never use `terraform workspace` commands or features.
- Never delete a state file or run `terraform state rm` without explicit human instruction and a backup.
- Every resource must be reachable by a `No changes` plan after any import PR is merged.

---

## Repository structure produced by this pack

### `<org>-infrastructure` (infra monorepo)

```
atlantis.yaml
.terraform-version
.pre-commit-config.yaml
piaas.yml
docs/adr/
scripts/tf-validate.sh
live/
  staging/
    00-bootstrap/
    10-identity/
    20-network/
    30-dns/
    40-data/
    50-compute/
    60-storage/
    70-observability/
    80-eks/          (placeholder until EKS questionnaire answered)
  production/
    (same structure)
```

### `<org>-terraform-modules` (module library)

```
modules/
  vpc/
  iam-role/
  documentdb/
  s3-bucket/
  ecr-repo/
  alb/
  security-group/
  acm-certificate/
.releaserc.json
```

---

## Files in this pack

```
aws-terraform-plan/
├── README.md                                   ← you are here
├── 00-standards/
│   ├── conventions.md
│   ├── decisions.md
│   └── fill-in-the-blanks.md
├── 01-discovery/
│   ├── PROMPT-00-inventory-existing-aws.md
│   └── QUESTIONNAIRE-org-context.md
├── 02-foundation/
│   ├── PROMPT-01-bootstrap-state-backend.md
│   ├── PROMPT-02-infra-monorepo-scaffold.md
│   ├── PROMPT-03-module-library-repo.md
│   ├── PROMPT-04-atlantis-on-bitbucket-dc.md
│   └── PROMPT-05-piaas-quality-gates.md
├── 03-layers/
│   ├── PROMPT-10-identity-iam.md
│   ├── PROMPT-11-network-vpc.md
│   ├── PROMPT-12-dns-route53-acm.md
│   ├── PROMPT-13-data-documentdb.md
│   ├── PROMPT-14-compute-ec2-alb.md
│   ├── PROMPT-15-storage-s3-ecr.md
│   ├── PROMPT-16-secrets-integration.md
│   ├── PROMPT-17-observability-audit.md
│   └── QUESTIONNAIRE-20-eks-handover.md
└── 04-operations/
    ├── import-playbook.md
    ├── review-checklist.md
    └── day2-runbook.md
```
