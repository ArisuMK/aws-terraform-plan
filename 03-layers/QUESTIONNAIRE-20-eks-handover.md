# Questionnaire: EKS Handover from Platform Automation

**Instructions for you (the human):** Answer every question in this file before any AI works on the `80-eks` layer. The AI cannot make safe decisions about EKS without these answers, because the wrong approach can destroy a running cluster or permanently lose the platform automation's ability to manage it. This questionnaire takes priority over everything else in Phase 4.

Take your time. Some answers require access to the platform automation's source code, the existing Terraform state file, and the Kubernetes API. Involve the platform team if needed.

---

## Section A: Platform Automation

**A1.** What is the name and type of the platform automation tool that provisioned the EKS cluster?

> **Your answer:** (e.g. "Internal tool called piaas-cluster-provisioner, written in Terraform", "Crossplane running on a management cluster", "A custom Python script that calls the Terraform AWS EKS module", "An internal Helm chart")

**A2.** Where is the source code for the platform automation? Can you read it?

> **Your answer:** (e.g. "Yes, it's at bitbucket.acme.com/PLATFORM/eks-automation — I have read access", "No, I need to ask the platform team")

**A3.** Does the platform automation continue to run against the cluster after initial provisioning, or was provisioning a one-time operation?

> **Your answer:** (This is the most critical question. Options: "One-time only — the cluster has not been touched by automation since creation", "It runs on a schedule or can be triggered at any time by the platform team", "Unknown — need to check with platform team")

**A4.** If the automation continues to run: what does it manage after provisioning? (node groups, add-ons, aws-auth/access entries, etc.)

> **Your answer:**

**A5.** Is there a Terraform state file from the initial platform provisioning run? If so, where is it stored (S3 bucket name, key, region, AWS account)?

> **Your answer:** (e.g. "Yes: s3://platform-tfstate-prod/eks/terraform.tfstate in us-east-1 account 012345678901")

**A6.** Can you get read access to the platform's state file? (`aws s3 cp s3://<bucket>/<key> - | terraform show -json -`)

> **Your answer:** yes / no / need to ask

**A7.** What is the exact Terraform module and version the platform used to provision EKS? (Check the state file or the automation source.)

> **Your answer:** (e.g. "`terraform-aws-modules/eks/aws` version `20.36.0`" or "custom internal module")

---

## Section B: EKS Cluster Configuration

**B1.** Cluster name(s) — staging and production.

> **Staging cluster name:**
> **Production cluster name:**

**B2.** Kubernetes version currently running.

> **Your answer:** (run `kubectl version --short` or check the AWS console)

**B3.** AWS region for each cluster.

> **Staging region:**
> **Production region:**

**B4.** Cluster endpoint access mode.

> **Your answer:** (run `aws eks describe-cluster --name <cluster> --query "cluster.resourcesVpcConfig"`)
> - Public endpoint enabled: yes/no
> - Private endpoint enabled: yes/no
> - Public access CIDRs (if restricted): 

**B5.** EKS add-ons currently installed. Run:
```bash
aws eks list-addons --cluster-name <cluster-name>
aws eks describe-addon --cluster-name <cluster-name> --addon-name <addon>
```
and paste the output.

> **Your answer:**

**B6.** EKS API authentication mode: `CONFIG_MAP` (aws-auth) or `API` (access entries)?

> **Your answer:** (run `aws eks describe-cluster --name <cluster> --query "cluster.accessConfig"`)

**B7.** If `CONFIG_MAP`: paste the current `aws-auth` ConfigMap:
```bash
kubectl get configmap aws-auth -n kube-system -o yaml
```

> **Your answer:**

**B8.** If `API` (access entries): list the current access entries:
```bash
aws eks list-access-entries --cluster-name <cluster-name>
```

> **Your answer:**

---

## Section C: Node Groups

**C1.** List all managed node groups. For each, provide:
- Node group name
- Instance type(s)
- Min/Max/Desired size
- Subnet IDs (private?)
- AMI type (AL2, AL2023, Bottlerocket)
- Labels and taints

```bash
aws eks list-nodegroups --cluster-name <cluster>
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng>
```

> **Your answer:**

**C2.** Are there any self-managed node groups (EC2 ASGs not managed by EKS)?

> **Your answer:**

**C3.** Is Karpenter deployed? If so, what version and what NodePool/EC2NodeClass configs exist?

> **Your answer:**

**C4.** Is Cluster Autoscaler deployed? If so, what version?

> **Your answer:**

---

## Section D: EKS Add-ons and Platform Components

**D1.** List all Helm releases currently deployed. Run:
```bash
helm list --all-namespaces
```

> **Your answer:**

**D2.** For each of the following, note if it is deployed and how (EKS add-on, Helm, manifests):
- AWS Load Balancer Controller
- External DNS
- EBS CSI Driver
- EFS CSI Driver (if any)
- CoreDNS (managed by EKS)
- kube-proxy (managed by EKS)
- VPC CNI (managed by EKS)
- Metrics Server
- Cluster Autoscaler or Karpenter
- cert-manager
- External Secrets Operator
- ArgoCD (if GitOps is in use)
- Datadog agent (if monitoring is in use)

> **Your answer:**

**D3.** Which of these components are managed by the platform automation and must not be changed by our Terraform?

> **Your answer:**

---

## Section E: IRSA Roles in Use

**E1.** List all IRSA roles currently in the AWS account(s) that are associated with EKS service accounts. Run:
```bash
aws iam list-roles --query "Roles[?contains(AssumeRolePolicyDocument, 'oidc')].[RoleName, Arn]" --output table
```

> **Your answer:**

**E2.** For each IRSA role, what Kubernetes service account does it correspond to?

> **Your answer:**

**E3.** Are any of these IRSA roles managed by the platform automation (and should not be imported into our Terraform)?

> **Your answer:**

---

## Section F: EKS Networking

**F1.** VPC ID used by the cluster.

> **Your answer:**

**F2.** Subnet IDs used for EKS control plane and node groups (these must match what is in the `20-network` layer).

> **Your answer:**

**F3.** Security group IDs created by the platform for EKS. Run:
```bash
aws ec2 describe-security-groups --filters "Name=tag:kubernetes.io/cluster/<cluster>,Values=owned" --query "SecurityGroups[*].[GroupId,GroupName]"
```

> **Your answer:**

**F4.** Are there any EKS-managed security group rules that apply to application pods (cluster security group rules)?

> **Your answer:**

---

## Section G: Platform Coordination Agreement

**G1.** Have you discussed with the platform team whether they agree to freeze automation changes to the cluster while you bring it under our Terraform management?

> **Your answer:** yes / no / in progress

**G2.** Is there a formal process to "hand off" a cluster from the platform to the SRE team's Terraform?

> **Your answer:**

**G3.** After handoff, will the platform team still manage cluster upgrades (Kubernetes version bumps) or will that be the SRE team's responsibility?

> **Your answer:**

---

## Section H: Desired End State (Decision)

Based on your answers above, which adoption strategy should the EKS layer use?

**H1.** Select the adoption strategy:

- [ ] **Full adoption**: Import the EKS cluster and all node groups into our Terraform state. The platform automation is decommissioned for this cluster. We own all future changes.
- [ ] **Partial adoption with `lifecycle { ignore_changes }`**: Import the cluster, but mark fields managed by the platform as `ignore_changes`. Both Terraform and the platform can coexist, but with careful coordination.
- [ ] **Read-only data sources**: Do not manage EKS in Terraform at all. Use `data "aws_eks_cluster"` to read cluster metadata for other layers (IRSA, ALB controller config). The platform owns EKS entirely.

> **Your choice:**

**H2.** If partial adoption — list the specific fields that the platform manages and that must be in `ignore_changes`:

> **Your answer:** (e.g. `ignore_changes = [access_config, kubernetes_network_config, upgrade_policy]`)

**H3.** Any specific timing constraints for the EKS import? (e.g. must happen during a maintenance window because importing the cluster may require a short re-plan period)

> **Your answer:**

---

## What to do after completing this questionnaire

1. Share this completed file with the AI when starting work on the `80-eks` layer.
2. Also share: the platform automation source code (or a summary of it), and the output of `terraform show -json` on the platform's state file (redact secrets).
3. The AI will then produce `PROMPT-20-eks-layer.md` — a custom prompt specific to your EKS configuration, rather than a generic template.
4. If you chose "Read-only data sources," the AI can immediately add the necessary `data "aws_eks_cluster"` and `data "aws_eks_cluster_auth"` blocks to the `10-identity` and `80-eks` placeholder directories without any import work.
5. Update `atlantis.yaml` to add the `staging-80-eks` and `production-80-eks` projects once the layer is ready.
