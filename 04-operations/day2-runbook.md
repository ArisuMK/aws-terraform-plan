# Day 2 Operations Runbook

This runbook covers the regular operational tasks that arise after the initial Terraform build-out: upgrading providers, handling state issues, onboarding new team members, rotating secrets, and responding to incidents.

---

## 1. Provider upgrade procedure

Provider upgrades should be done monthly (via Dependabot or Renovate PRs, or manually).

### Steps

1. Create a branch: `git checkout -b chore/provider-bump-aws-5.98.0`.
2. Update the pinned version in every `live/*/versions.tf`:
   ```hcl
   aws = {
     source  = "hashicorp/aws"
     version = "= 5.98.0"   # new version
   }
   ```
3. Re-generate the lock file for each root:
   ```bash
   for dir in live/staging/* live/production/*; do
     if [ -f "$dir/versions.tf" ]; then
       echo "Locking $dir"
       (cd "$dir" && terraform providers lock -platform=linux_amd64 -platform=darwin_arm64)
     fi
   done
   ```
4. Open a PR. Atlantis plans all affected projects.
5. Review: plan should show `No changes` for all projects — a provider version bump alone should not change resources.
6. Apply via `atlantis apply -p staging-00-bootstrap` (start with bootstrap, proceed through layers).
7. Merge.

### When a provider upgrade does cause resource changes

This happens when the provider changes defaults or attribute handling between versions. Common examples:
- AWS provider 5.x changed how `aws_s3_bucket` handles encryption — a plan may show in-place updates.
- New required attributes: plan shows "attribute required but null."

In these cases: apply staging first, monitor for 24 hours, then apply production.

---

## 2. Adding a new infrastructure layer

1. Create the directory: `live/<env>/<NN>-<name>/`.
2. Write the standard files: `backend.tf`, `versions.tf`, `providers.tf`, `locals.tf`, `data.tf`, `main.tf`, `outputs.tf`.
3. Run `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64` and commit `.terraform.lock.hcl`.
4. Add the new project to `atlantis.yaml`:
   ```yaml
   - name: <env>-<NN>-<name>
     dir: live/<env>/<NN>-<name>
     autoplan: *autoplan
     apply_requirements:
       - approved
       - mergeable
   ```
5. Open a PR. Atlantis plans the new project (plan should be empty on first run if no resources are defined yet).
6. Merge.

---

## 3. Adding a new team member

### Access requirements

The new engineer needs:
- Bitbucket DC account with access to `<INFRA_REPO>` and `<MODULES_REPO>`.
- AWS access via SSO (do not create IAM users). Assign the appropriate SSO permission set.
- Kubernetes access via the EKS access entry or `aws-auth` ConfigMap (managed in `80-eks` layer).

### Terraform workflow onboarding

Walk through these steps with the new engineer:

1. Clone the infra repo.
2. Install: `terraform ~> 1.11`, `tflint`, `trivy`, `checkov`, `terraform-docs`, `pre-commit`.
3. Run `pre-commit install` in the repo root.
4. Create a test branch, make a trivial comment change, open a PR — observe the Atlantis plan comment.
5. Merge the PR without applying (or apply a no-op change to confirm the workflow).
6. Read `04-operations/import-playbook.md` and `04-operations/review-checklist.md`.

---

## 4. Rotating secrets (database passwords, tokens)

### General principle

Terraform generates secrets with `random_password` and writes them to the secrets tool. Rotation involves:
1. Generating a new password outside Terraform (or in Terraform via `lifecycle { ignore_changes = [result] }` and `terraform taint`).
2. Updating the secret in the secrets tool.
3. Updating the database (if required).
4. Updating any application config that reads the secret.

### Rotating a DocumentDB master password

**Step 1:** Generate a new password outside Terraform (e.g. AWS Secrets Manager rotation or a script):
```bash
NEW_PASS=$(openssl rand -base64 32 | tr -dc 'a-zA-Z0-9!#$%&*()-_=+' | head -c 32)
```

**Step 2:** Update in the secrets tool:
```bash
aws secretsmanager put-secret-value \
  --secret-id "<org>/<env>/docdb/master-password" \
  --secret-string "{\"username\":\"masteruser\",\"password\":\"$NEW_PASS\"}"
```

**Step 3:** Update the DocumentDB master password:
```bash
aws docdb modify-db-cluster \
  --db-cluster-identifier "<org>-<env>-docdb-main" \
  --master-user-password "$NEW_PASS" \
  --apply-immediately
```

**Step 4:** Taint the `random_password` resource so Terraform re-generates it on next apply. This keeps the Terraform state in sync with the new password without a full destroy/recreate.

```
# Run locally, NOT via Atlantis
terraform -chdir=live/<env>/40-data state rm random_password.docdb_master
```

Then add a new `import {}` block or re-run `terraform apply` to regenerate. This is one of the few cases where a local state operation is acceptable — document it in the PR description and inform the team.

**Alternative (cleaner):** Use a Secrets Manager rotation Lambda with automatic rotation — Terraform is not involved in rotation at all. This is the recommended approach for production.

---

## 5. Recovering from a failed apply

### Symptoms
- Atlantis shows "Apply failed" in the PR comment.
- Some resources were created or modified, some were not.
- The state file may be in a partial state.

### Steps

**Step 1:** Read the Atlantis error log carefully. Identify:
- Which resource(s) failed.
- What error they returned (AWS API error, timeout, resource conflict).

**Step 2:** Do not immediately retry `atlantis apply` — it may make the situation worse.

**Step 3:** Check if the partial apply left orphaned resources in AWS (resources created by Terraform but not in the state file because the state write failed).

```bash
# Check if the resource exists in AWS
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=<org>-stg-vpc"

# Check if it exists in state
terraform -chdir=live/staging/20-network state list | grep vpc
```

**Step 4:** Options depending on the situation:

| Situation | Action |
|---|---|
| Resource exists in AWS but not in state | Add an `import {}` block and re-plan |
| Resource does not exist in AWS or state | Re-run `atlantis apply` — Terraform will create it |
| Resource exists in state with wrong attributes | Fix the Terraform config to match the live state, then re-plan and apply |
| AWS API error (e.g. throttling, transient) | Wait 5 minutes and retry `atlantis apply` |
| Timeout | Check if the resource eventually completed in AWS. If yes, import. If no, investigate. |

**Step 5:** After recovery, run `terraform plan` and confirm it shows `No changes` before closing the PR.

---

## 6. Handling state lock issues

With S3 native locking (`use_lockfile = true`), a stale lock appears as a `.terraform.tfstate.lock` object in the state bucket.

### Symptoms
- Atlantis reports: `Error acquiring the state lock`.
- The previous apply may have crashed or been killed mid-apply.

### Steps

**Step 1:** Confirm the lock is stale (no active Atlantis job for this project):

```bash
aws s3 ls s3://<ORG>-tfstate-<env>-<region>/<env>/<layer>/
# Look for a .lock file
```

**Step 2:** Check that no Atlantis job is currently running:
- Check Atlantis pod logs: `kubectl logs -n <ATLANTIS_NAMESPACE> -l app.kubernetes.io/name=atlantis`
- Check if anyone else on the team is applying via `atlantis apply` in a PR.

**Step 3:** If the lock is confirmed stale, delete it:

```bash
aws s3 rm s3://<ORG>-tfstate-<env>-<region>/<env>/<layer>/terraform.tfstate.lock
```

**Step 4:** Retry the apply.

**Step 5:** Document the stale lock event in the PR or an incident report — repeated stale locks may indicate Atlantis instability.

---

## 7. Upgrading Terraform version

This must be coordinated across all team members and the Atlantis deployment.

### Steps

1. Announce the upgrade with 1-week notice.
2. Update `.terraform-version` in the infra repo.
3. Update `required_version = "~> X.Y"` in all `backend.tf` and `versions.tf` files.
4. Update the Atlantis pod `ATLANTIS_DEFAULT_TF_VERSION` environment variable via Helm values update.
5. Update `piaas.yml` to use the new Terraform image/version.
6. Update all developers' local Terraform installations.
7. Open a PR with all the above changes. Atlantis re-plans all projects with the new version.
8. Apply and monitor.

### Terraform 1.11 → next minor version note

The `use_lockfile = true` feature is stable in 1.11. S3 native locking format may change between major versions — check the migration guide before upgrading.

---

## 8. Atlantis restart or pod recovery

If the Atlantis pod crashes or is deleted:

```bash
# Check pod status
kubectl get pods -n <ATLANTIS_NAMESPACE>

# Helm rolls back automatically with liveness/readiness probes.
# If the deployment is stuck:
kubectl rollout restart deployment -n <ATLANTIS_NAMESPACE> <ATLANTIS_RELEASE>

# Check logs after restart
kubectl logs -n <ATLANTIS_NAMESPACE> -l app.kubernetes.io/name=atlantis --tail=100
```

Any Atlantis plans that were in progress when the pod crashed are lost. Comment `atlantis plan` in the affected PRs to re-trigger.

Any applies that were in progress when the pod crashed: check the Atlantis log for the last successful resource. Check the AWS console for the resources. Follow the "failed apply" recovery steps above.

---

## 9. Emergency break-glass procedure

If Atlantis is down and a production apply is urgently needed:

1. Inform the team and document the emergency.
2. Assume the `<ORG>-prd-terraform-exec` IAM role directly (via the break-glass mechanism — confirm this with the identity team).
3. Run `terraform apply` locally from a clean checkout:
   ```bash
   cd live/production/<layer>
   terraform init
   terraform plan -out=emergency.tfplan
   # Review the plan
   terraform apply emergency.tfplan
   ```
4. After the emergency is resolved, open a PR with the exact same changes that were applied — confirm Atlantis plans show `No changes`.
5. Write a post-mortem documenting the emergency and why Atlantis was unavailable.
6. Treat direct CLI applies as technical debt — they bypass all controls. Make them exceptional.

---

## 10. Monthly operational checklist

Run these checks at the beginning of each month:

- [ ] Review AWS Budgets alert status — any accounts near threshold?
- [ ] Check GuardDuty findings — any HIGH or CRITICAL items unresolved?
- [ ] Review AWS Config non-compliant resources — any new compliance violations?
- [ ] Dependabot/Renovate: are there pending provider upgrade PRs? Review and merge.
- [ ] Check Atlantis pod health: logs clean? Any error spikes?
- [ ] Review CloudTrail for any unexpected API calls with high privilege.
- [ ] Rotate any secrets approaching their rotation deadline.
- [ ] Check `.terraform.lock.hcl` files are up to date — run `terraform providers lock` if providers were updated.
- [ ] Verify state bucket versioning is enabled: `aws s3api get-bucket-versioning --bucket <ORG>-tfstate-<env>-<region>`.
- [ ] Run `tf-validate.sh` locally and confirm all roots validate.
