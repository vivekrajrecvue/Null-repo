# Do's and Don'ts – Recvue Dev/QA/Prod Setup

Use this as a quick checklist for safe and consistent work on AWS, Terraform, and EKS.

---

## 1. Credentials and secrets

| Do | Don't |
|----|--------|
| Use IAM user credentials via **environment variables** (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) or **AWS profile** (`AWS_PROFILE`). | Put Access Key ID or Secret Access Key in `.tf`, `.tfvars`, or any file committed to git. |
| Store secrets in **AWS Secrets Manager** or **Parameter Store**; reference them in Terraform or app config. | Hardcode DB passwords, API keys, or tokens in code or config files. |
| Add `*.tfvars` (if they contain secrets), `backend.tfvars`, and `.env` to **`.gitignore`**. | Commit `backend.tfvars` or env files that contain credentials. |
| Prefer **one IAM user per environment** (e.g. Terraform-dev, Terraform-qa) with least privilege. | Use a single shared IAM user with broad permissions across all environments. |

---

## 2. Terraform

| Do | Don't |
|----|--------|
| Use a **remote backend** (S3 + DynamoDB lock) for state; use a **different state key per environment** (e.g. `dev/terraform.tfstate`). | Store Terraform state only on your laptop or in a shared drive. |
| Run **`terraform plan`** before every **`terraform apply`** and review the plan. | Run `terraform apply` without reviewing changes, especially for prod. |
| Use **separate Terraform state** (or workspaces) per environment so dev/qa/prod don’t share state. | Use one state file for all environments. |
| **Tag all resources** (e.g. `Environment = dev`, `Project = recvue`) for cost and security. | Create resources without consistent tags. |
| Pin **provider and module versions** in Terraform (e.g. `required_providers`, version in module source). | Use unversioned or `latest` providers/modules in production. |
| Keep **sensitive variables** out of default values; use env vars or a non-committed tfvars file. | Put passwords or keys in default values in `.tf` or committed `.tfvars`. |

---

## 3. Environments (dev, QA, prod)

| Do | Don't |
|----|--------|
| Use **separate VPCs** per environment (e.g. `recvue-dev-vpc`, `recvue-qa-vpc`, `recvue-prod-vpc`). | Put dev, QA, and prod in the same VPC or same subnets. |
| Use **different CIDR ranges** per VPC (e.g. 10.1.0.0/16, 10.2.0.0/16, 10.3.0.0/16). | Overlap CIDRs between VPCs if you ever need to peer or connect them. |
| Apply **stronger controls in prod** (WAF, backups, smaller blast radius, stricter IAM). | Use the same security and access rules for dev and prod. |
| Treat **prod changes** as higher risk: extra review, change windows, and rollback plan. | Apply prod Terraform or config changes without review or plan. |

---

## 4. EKS and Kubernetes

| Do | Don't |
|----|--------|
| Set **resource requests/limits** on pods (CPU/memory) so the scheduler and autoscaler work correctly. | Run pods without requests/limits; this can starve other workloads or cause OOMs. |
| Use **Dev: 1 pod per microservice**, **QA: 2 pods per microservice** (via Helm/Kustomize or Deployment replicas). | Use the same replica count for dev and QA without documenting why. |
| Right-size **EKS node types** for 13 microservices at 1 GB each (e.g. one node with 16 GB RAM or two with 8 GB each). | Oversize nodes for dev (e.g. large instances when one medium/large is enough). |
| Use **namespaces** to separate apps or environments when appropriate. | Put everything in `default` namespace without a reason. |
| Prefer **Helm or Kustomize** for consistent, environment-specific config (replicas, resources). | Maintain duplicate YAML per environment without a single source of truth. |

---

## 5. Networking and security

| Do | Don't |
|----|--------|
| Put **EKS nodes and Aurora in private subnets**; use NAT or VPC endpoints for outbound traffic. | Put application or database instances in public subnets unless required. |
| Restrict **security groups** by port and source (e.g. ALB → app only; app → DB only). | Use 0.0.0.0/0 for ingress on application or database security groups. |
| Use **one NAT Gateway** (or VPC endpoints) in dev to control cost; consider two in prod for HA. | Add multiple NAT Gateways in dev without a cost/redundancy need. |
| Enable **S3 versioning and encryption** on the Terraform state bucket. | Use an unencrypted or non-versioned bucket for state. |

---

## 6. Database (Aurora / RDS)

| Do | Don't |
|----|--------|
| Put **Aurora in private DB subnets** with no direct internet access. | Expose the DB to the internet or to 0.0.0.0/0. |
| Use **RDS Proxy** for connection pooling when many pods talk to the DB. | Rely only on direct DB connections from every pod without pooling. |
| Use **Secrets Manager** (or Parameter Store) for DB credentials and reference from RDS Proxy/app. | Store DB credentials in code, config files, or plain Terraform variables. |
| Take **automated backups** and test restore; in prod, use a retention policy that meets RTO/RPO. | Skip backups in prod or never test restores. |

---

## 7. General operations

| Do | Don't |
|----|--------|
| **Document** instance types, replica counts, and env-specific choices (e.g. in DEV-SETUP-PLAN, RESOURCES-CHECKLIST). | Change sizing or topology without updating docs. |
| Use **meaningful names** and tags so resources are identifiable (e.g. `recvue-dev-alb`). | Create resources with generic or auto-generated names only. |
| **Review costs** periodically (Cost Explorer, tags by Environment/Project). | Ignore cost until the first bill. |
| Prefer **VPC endpoints** (e.g. S3, ECR) in private subnets to reduce NAT cost and improve security. | Rely only on NAT for all outbound traffic in long-lived environments. |

---

## 8. Quick checklist before apply

- [ ] Credentials are from env or profile, not in code.
- [ ] Terraform state is remote (S3 + lock); correct backend key for this environment.
- [ ] `terraform plan` reviewed; no surprise destroys or broad security group changes.
- [ ] Correct environment (dev/qa/prod); no prod apply by mistake.
- [ ] Sensitive values are not in committed files.
- [ ] Resources are tagged (Environment, Project).

---

*Adjust any item to match your org’s policies (e.g. separate accounts per env, SSO instead of IAM users).*
