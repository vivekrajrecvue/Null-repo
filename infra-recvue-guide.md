# Infrastructure Organization Guide

**Audience:** Developers, Team Leads, DevOps  
**Purpose:** How our AWS infrastructure is organized, where things live, and how to work with it safely.

---

## 1. Overview

Our portal runs on **AWS**. Infrastructure is defined as **code (Terraform)** and applied via a **Jenkins pipeline** using **OIDC/assume-role** (no long-lived access keys). This guide explains how that infra is organized so you can:

- Find where a given resource is defined
- Understand what you can change vs what needs infra/lead input
- Run plan locally (when allowed) and interpret pipeline runs
- Know who to ask for what

---

## 2. High-Level Layout

```
Repository (e.g. recvue-portal-infra or monorepo/infra)
├── infra/
│   ├── bootstrap/          # One-time: state backend + IAM roles for Jenkins/Terraform
│   ├── modules/            # Reusable Terraform modules (network, EKS, DB, etc.)
│   └── envs/
│       └── dev/            # Dev environment: wires modules together, tfvars, backend
├── Jenkinsfile             # Pipeline: plan → approval → apply
└── docs/
    └── Infrastructure-Organization-Guide.md   # This document
```

- **Bootstrap:** Run once per account/region (state bucket, lock table, Jenkins OIDC + Terraform deployer roles).
- **Modules:** Building blocks (VPC, EKS, RDS, S3+CloudFront, etc.). No environment-specific values inside.
- **Envs/dev:** Dev-specific configuration; calls modules and passes variables. This is what the Jenkins pipeline runs against.

---

## 3. What Lives Where

| You care about… | Look here |
|-----------------|-----------|
| **Networking** (VPC, subnets, NAT, security groups) | `infra/modules/network/` + `infra/envs/dev/main.tf` (module usage) |
| **EKS cluster and node groups** | `infra/modules/eks/` + `infra/envs/dev/main.tf` |
| **API Gateway / ALB / WAF** | `infra/modules/api_gateway/`, `alb/`, `waf/` + env `main.tf` |
| **Databases** (PostgreSQL, RDS Proxy, Redis, RabbitMQ) | `infra/modules/rds_postgres/`, `redis/`, `rabbitmq/` + env `main.tf` |
| **Frontend** (S3 + CloudFront + Route53 + ACM) | `infra/modules/s3_frontend/`, `cloudfront/`, `route53_acm/` + env `main.tf` |
| **Secrets, KMS, IAM for apps** | `infra/modules/security/` + env `main.tf` |
| **Logs, metrics, tracing** | `infra/modules/observability/` + env `main.tf` |
| **Dev-specific values** (region, instance sizes, names) | `infra/envs/dev/terraform.tfvars` and `infra/envs/dev/variables.tf` |
| **How plan/apply is run** | `Jenkinsfile` (and optionally `AWS-Terraform-Jenkins-OIDC-Implementation-Plan.md`) |

---

## 4. Environment Strategy

- **Environments** are represented as separate **directories** under `infra/envs/` (e.g. `dev`, `qa`, `prod`).
- Each environment has its own:
  - **State file** (different `key` in the same or different S3 bucket)
  - **tfvars** (sizes, counts, feature flags)
  - **Backend config** (`backend.hcl`) pointing to that state
- **Dev** is the primary environment for day-to-day development; QA/Prod may use the same modules with different variables.

---

## 5. Module Conventions

- **Reusability:** Modules under `infra/modules/` do not hardcode environment names, account IDs, or region. They receive these via variables.
- **Naming:** Module names reflect the AWS product or area (e.g. `network`, `eks`, `rds_postgres`). Resource names inside use a `prefix` or `environment` variable so resources are identifiable (e.g. `dev-portal-*`).
- **State:** Only the **env** (e.g. `infra/envs/dev`) holds state. Modules are not applied standalone; they are called from env `main.tf`.
- **Outputs:** Modules expose outputs (e.g. VPC ID, subnet IDs, EKS cluster name, RDS endpoint) so other modules or the env can reference them (e.g. RDS module uses VPC from network module).

---

## 6. How Changes Flow

1. **Code change:** Edit Terraform under `infra/modules/` or `infra/envs/dev/` (and/or other envs).
2. **Pull request:** Open a PR; CI runs `terraform fmt -check`, `terraform validate`, and `terraform plan` for the target env (e.g. dev).
3. **Review:** Team lead or DevOps reviews plan output and merge.
4. **Apply:** After merge, the Jenkins pipeline runs again. A **manual approval** gate is required before `terraform apply`. Only then is live infra updated.
5. **No direct apply from laptops** for shared envs unless explicitly allowed by policy; apply runs in Jenkins with the assume-role identity.

This keeps infra changes auditable and consistent.

---

## 7. Who Does What (RACI-style)

| Activity | Developer | Team Lead | DevOps / Infra Owner |
|---------|-----------|-----------|----------------------|
| Read this doc and know where things are | ✓ | ✓ | ✓ |
| Propose changes to module inputs (e.g. replica count, instance size in tfvars) | ✓ | Review | Review / Merge |
| Change or add Terraform modules (new resources, new modules) | Optionally draft | Review | Implement / Review / Merge |
| Run `terraform plan` locally (if allowed by policy) | ✓ | ✓ | ✓ |
| Approve and run `terraform apply` in Jenkins | — | ✓ (or delegated) | ✓ |
| Change bootstrap (state bucket, IAM roles for Jenkins/Terraform) | — | — | ✓ |
| Change Jenkins pipeline (stages, approval, notifications) | — | Review | Implement / Merge |
| Access AWS console for debugging (read-only or with dev role) | ✓ (if granted) | ✓ | ✓ |

Developers are not expected to apply Terraform to shared environments; they can still run plan and suggest changes via PRs.

---

## 8. Naming and Tagging

- **Resource naming:** We use a consistent prefix per environment (e.g. `dev-portal-*`). The exact pattern is defined in `infra/envs/dev/terraform.tfvars` (e.g. `name_prefix = "dev-portal"`).
- **Tags:** All resources created by Terraform should carry at least:
  - `Environment` (e.g. dev, qa, prod)
  - `Project` (e.g. recvue-portal)
  - `ManagedBy` = terraform
  - Optional: `Module`, `Service`

This helps with cost allocation, debugging, and compliance.

---

## 9. State and Backend

- **Remote state** is stored in **S3** (e.g. bucket `recvue-tf-state-dev`).
- **Locking** is done with a **DynamoDB** table (e.g. `recvue-tf-lock-dev`) so two applies never run at once.
- **Backend config** for dev lives in `infra/envs/dev/backend.hcl`. The Jenkinsfile uses this for `terraform init -backend-config=backend.hcl`.
- **Sensitive outputs** (e.g. DB passwords) are not stored in plain text in state when possible; they come from Secrets Manager or similar, and Terraform only references the secret ARN.

Developers and leads do not need direct S3/DynamoDB access for day-to-day work; Jenkins has the role that can read/write state.

---

## 10. Quick Reference: Key Files

| File / directory | Purpose |
|------------------|---------|
| `infra/envs/dev/main.tf` | Composes all modules for dev; data sources and providers if needed |
| `infra/envs/dev/variables.tf` | Variable declarations for dev |
| `infra/envs/dev/terraform.tfvars` | Dev-specific values (sizes, counts, names) |
| `infra/envs/dev/backend.hcl` | S3 bucket, state key, DynamoDB table, region for dev |
| `infra/envs/dev/providers.tf` | AWS provider (region, assume role) and required Terraform/providers versions |
| `infra/modules/<name>/` | Reusable Terraform for one area (network, EKS, RDS, etc.) |
| `Jenkinsfile` | Pipeline: checkout → init → validate → plan → approval → apply |

---

## 11. Where to Learn More

- **Full CI/CD and OIDC setup:** `AWS-Terraform-Jenkins-OIDC-Implementation-Plan.md`
- **What resources exist in dev and at what scale:** `AWS-Dev-Resources-Config.csv` (and related CSVs)
- **Architecture decisions:** ADR documents and architecture diagrams in the repo or Confluence

---

## 12. Summary

- **Infra is code** under `infra/`, applied via **Jenkins** with **no static AWS keys** (OIDC + assume-role).
- **Bootstrap** = state + IAM for pipelines; **modules** = reusable building blocks; **envs/dev** = dev wiring and config.
- **Change flow:** PR → plan in CI → review → merge → apply in Jenkins after **manual approval**.
- **Developers** can read this doc, run plan, and propose changes; **leads/DevOps** own review and apply.
- **Naming and tagging** are consistent so we can find and manage resources by environment and project.

If something isn’t clear or you need access to run plan locally, ask your Team Lead or DevOps.
