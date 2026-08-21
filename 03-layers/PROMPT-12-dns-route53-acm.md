# PROMPT-12: DNS — Route53 and ACM

## 1. Role and objective

You are a senior SRE building the `30-dns` layer for both staging and production. This layer manages Route53 hosted zones, ACM certificates, and the records that the ALB and EKS ingress controller will use. ACM certificates are created with DNS validation, with the validation records also managed here.

---

## 2. Preconditions

- [ ] PROMPT-11 (network) complete: VPC IDs available from remote state.
- [ ] `fill-in-the-blanks.local.md` has: `<PUBLIC_DOMAIN>`, `<STAGING_DOMAIN>`, `<PRODUCTION_DOMAIN>`, hosted zone IDs if they already exist.
- [ ] DNS registrar is known. If the domain is not registered in Route53, the human must manually update NS records at the registrar after this layer creates the hosted zone.

---

## 3. Required inputs

1. `<PUBLIC_DOMAIN>` — e.g. `acme.com`.
2. `<STAGING_DOMAIN>` — e.g. `stg.acme.com`.
3. `<PRODUCTION_DOMAIN>` — e.g. `acme.com` (apex) or `prd.acme.com`.
4. Existing Route53 zone IDs, if any (import or use data sources).
5. Whether the apex domain (`acme.com`) is used for production or only subdomains.
6. Whether a wildcard certificate (`*.<env>.domain`) or specific SANs are needed.

---

## 4. In scope / out of scope

**In scope:**
- Route53 public hosted zones.
- ACM certificates with DNS validation (one wildcard per environment).
- Route53 validation records for ACM.
- SOA and NS records (managed by Route53 automatically — do not override).
- Optional: private hosted zones for internal service discovery.

**Out of scope:**
- Route53 health checks (add when compute layer is ready).
- Route53 records pointing to ALBs (managed in 50-compute).
- DNSSEC (add if required by security policy).

---

## 5. Reference material

- `00-standards/conventions.md` — naming conventions.
- `01-discovery/inventory.md` — Route53 and ACM sections.

---

## 6. Step-by-step procedure

### Step 1: `data.tf` — read network outputs

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket       = "<ORG>-tfstate-stg-<STAGING_REGION>"
    key          = "staging/20-network/terraform.tfstate"
    region       = "<STAGING_REGION>"
    assume_role  = { role_arn = "arn:aws:iam::<STAGING_ACCOUNT_ID>:role/<ORG>-stg-terraform-exec" }
  }
}
```

Or, if no cross-layer data is needed from network (dns is self-contained), skip remote state and use data sources only.

### Step 2: `zones.tf`

**Create new zones:**
```hcl
resource "aws_route53_zone" "staging" {
  name    = "<STAGING_DOMAIN>"
  comment = "Staging public zone managed by Terraform."

  tags = { Name = "<STAGING_DOMAIN>", Service = "dns" }
}
```

**Import existing zone:**
```hcl
import {
  to = aws_route53_zone.staging
  id = "<STAGING_ZONE_ID>"
}

resource "aws_route53_zone" "staging" {
  name    = "<STAGING_DOMAIN>"
  comment = "Staging public zone managed by Terraform."
  tags    = { Name = "<STAGING_DOMAIN>", Service = "dns" }
}
```

### Step 3: `acm.tf` — wildcard certificate

```hcl
resource "aws_acm_certificate" "staging_wildcard" {
  domain_name               = "<STAGING_DOMAIN>"
  subject_alternative_names = ["*.<STAGING_DOMAIN>"]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name    = "${local.org}-${local.env}-wildcard-cert"
    Service = "dns"
  }
}

resource "aws_route53_record" "staging_cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.staging_wildcard.domain_validation_options :
    dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = aws_route53_zone.staging.zone_id
}

resource "aws_acm_certificate_validation" "staging_wildcard" {
  certificate_arn         = aws_acm_certificate.staging_wildcard.arn
  validation_record_fqdns = [for record in aws_route53_record.staging_cert_validation : record.fqdn]
}
```

### Step 4: `outputs.tf`

```hcl
output "staging_zone_id" {
  description = "Route53 hosted zone ID for the staging domain."
  value       = aws_route53_zone.staging.zone_id
}

output "staging_zone_name_servers" {
  description = "NS records to configure at the DNS registrar."
  value       = aws_route53_zone.staging.name_servers
}

output "staging_wildcard_cert_arn" {
  description = "ARN of the wildcard ACM certificate for staging."
  value       = aws_acm_certificate_validation.staging_wildcard.certificate_arn
}
```

---

## 7. Expected file tree

```
live/staging/30-dns/
  backend.tf, versions.tf, providers.tf, locals.tf, data.tf
  zones.tf, acm.tf, outputs.tf
  .terraform.lock.hcl
live/production/30-dns/
  (same)
```

---

## 8. Code contracts

ACM certificate creation and validation must be in the same Terraform apply. Use `aws_acm_certificate_validation` to block until the certificate is ISSUED before the ALB layer consumes it.

Certificates for the ALB must be in the same region as the ALB. CloudFront certificates must be in `us-east-1`.

---

## 9. Acceptance criteria

- [ ] `terraform validate` passes.
- [ ] After apply: certificate status is `ISSUED` (not `PENDING_VALIDATION`).
- [ ] NS records shown in outputs — human confirms they are set at the registrar.
- [ ] `nslookup <STAGING_DOMAIN>` resolves after NS propagation.

---

## 10. Guardrails

- Never delete a hosted zone that has production records — this causes an outage.
- Never modify SOA or NS records.
- `create_before_destroy = true` on all ACM certificates — required to avoid downtime during cert rotation.
- Do not hardcode IP addresses in DNS records — use ALB DNS names and alias records.

---

## 11. Handoff note

Report: zone IDs created/imported, certificate ARNs, NS records for manual registrar update, and certificate validation status.
