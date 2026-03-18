# Who Has What Access – Recvue Dev/QA/Prod

This document defines which roles or personas have access to which systems and what they can do. Adjust names and scope to match your org and the IAM users the client provides.

---

## 1. Roles overview

| Role | Typical use | Environments |
|------|-------------|--------------|
| **Platform / DevOps** | Provision and manage infra (Terraform, EKS, networking, DB). | Dev, QA, Prod (prod with extra controls). |
| **Developer** | Deploy and debug apps on EKS; read infra where needed. | Dev, optionally QA. |
| **QA** | Deploy and test in QA; read QA (and sometimes dev) resources. | QA, Dev (read or limited). |
| **Viewer / Read-only** | View dashboards, logs, costs; no changes. | Dev, QA, Prod (read-only). |
| **Release / Prod deployer** | Deploy to prod only; limited infra change. | Prod (deploy only), Dev/QA (read if needed). |

---

## 2. Access matrix

**Legend:**  
- **Full** = Create, update, delete, read  
- **Read/write** = Deploy/change apps or config; read infra  
- **Read** = View only; no changes  
- **None** = No access  
- **Restricted** = Scoped (e.g. only certain namespaces or actions)

| Resource / system | Platform / DevOps | Developer | QA | Viewer | Release / Prod deployer |
|------------------|-------------------|-----------|-----|--------|--------------------------|
| **AWS Console (dev)** | Full (within IAM policy) | Read or restricted | Read or restricted | Read | Read (if needed) |
| **AWS Console (QA)** | Full | Read or restricted | Read/write (QA only) | Read | Read |
| **AWS Console (prod)** | Full (with change process) | None | None | Read | Read; deploy only (no infra) |
| **Terraform (state, apply)** | Full (per env); prod with review | None or read state | None | None | None (or read prod state only) |
| **IAM user / credentials** | Use Terraform IAM user(s); rotate keys | Use dev/QA user if provided | Use QA user | Use read-only user | Use prod-deploy user only |
| **EKS – kubectl (dev)** | Full (admin) | Read/write (deploy, logs, exec in app namespaces) | Read (or limited write in QA) | Read | Read |
| **EKS – kubectl (QA)** | Full | Read or restricted | Read/write (deploy, test) | Read | Read |
| **EKS – kubectl (prod)** | Full (with process) | None | None | Read (logs, get pods) | Deploy only (apply manifests; no cluster config) |
| **ECR** | Push/pull all; manage repos | Push/pull dev (and QA if allowed) | Push/pull QA | Pull only (if needed) | Pull prod; push only via CI or controlled process |
| **Aurora / RDS (dev)** | Full (create, connect, schema) | Connect (read/write app data) | Connect (read/write QA data) | Read only or None | None |
| **Aurora / RDS (QA)** | Full | Read or connect if allowed | Connect (read/write) | Read only or None | None |
| **Aurora / RDS (prod)** | Full (with change process) | None | None | Read only (e.g. reporting) or None | None (or read-only for verification) |
| **S3 (app buckets)** | Full | Dev/QA read/write per bucket policy | QA read/write | Read | Prod read or write only as needed |
| **S3 (Terraform state)** | Read/write (lock, apply) | None | None | None | None |
| **Secrets Manager / Parameter Store** | Full (create, update, read) | Read (app secrets for dev/QA) | Read (QA secrets) | None | Read (prod app secrets only if needed) |
| **CloudWatch / logs** | Full | Read dev/QA logs | Read QA (and dev if allowed) | Read | Read prod (for release verification) |
| **Cost Explorer / billing** | Read (and set budgets if allowed) | None or Read | None | Read (if allowed) | None or Read |

---

## 3. Per-role summary

**Platform / DevOps**  
- AWS: Full access to dev, QA, prod within IAM policy (VPC, EKS, RDS, S3, IAM for Terraform, etc.).  
- Terraform: Run plan/apply for all environments; prod only after review/approval.  
- EKS: Full kubectl (admin) on all clusters.  
- DB: Create, connect, schema changes; credentials in Secrets Manager.

**Developer**  
- AWS: Console/CLI for dev (and QA if assigned); read or restricted so no infra deletion.  
- Terraform: No apply; optional read-only state for dev.  
- EKS: Deploy apps, view logs, exec into pods in dev (and QA if allowed); no cluster or node changes.  
- ECR: Push/pull images for dev (and QA if allowed).  
- DB: Connect to dev (and QA if allowed) with app-level read/write.

**QA**  
- AWS: Console/CLI for QA (and dev if needed); no prod.  
- Terraform: No access.  
- EKS: Deploy and test in QA; read (and limited write) in QA namespaces.  
- ECR: Pull/push for QA.  
- DB: Connect to QA (and dev if needed) for tests.

**Viewer**  
- AWS: Read-only console (or read-only IAM) for dev, QA, prod.  
- EKS: Read-only kubectl (get, logs) where allowed.  
- DB: No access or read-only for reporting.  
- Cost: Read Cost Explorer if policy allows.

**Release / Prod deployer**  
- AWS: Console/CLI for prod; no infra changes (no Terraform, no EKS cluster config).  
- EKS: Deploy app manifests to prod (e.g. via CI or controlled kubectl); no node or cluster changes.  
- ECR: Pull prod images; push only via pipeline or approved process.  
- DB: No direct prod DB access (or read-only for verification only).

---

## 4. IAM and Kubernetes mapping

- **IAM (AWS):** One IAM user (or role) per role type and optionally per environment (e.g. `recvue-terraform-dev`, `recvue-developer-dev`, `recvue-qa-qa`, `recvue-viewer`). Permissions via IAM policies; no sharing of credentials.  
- **EKS:** Map IAM users/roles to Kubernetes RBAC (e.g. `aws-auth` ConfigMap or IRSA). Developers get limited namespace-scoped rights; Platform gets `cluster-admin`; Viewer gets read-only cluster role.  
- **DB:** Access via IAM auth or username/password stored in Secrets Manager; app and user access scoped by RDS/Proxy and security groups.

---

## 5. Quick reference

- **Who can run Terraform apply?** Platform/DevOps only; prod with review.  
- **Who can deploy to prod?** Release/Prod deployer (apps only); Platform/DevOps (infra and apps).  
- **Who can see prod DB or state?** Platform/DevOps; Viewer/Release only if explicitly granted read-only.  
- **Who can create/delete EKS nodes or VPC?** Platform/DevOps only.  
- **Who can push to ECR?** Platform (all); Developer (dev/QA); QA (QA); Release via pipeline only for prod.

---

*Update this doc when the client adds/removes IAM users or when you change RBAC (AWS or EKS).*
