# How Deployments Happen

**Audience:** Developers, Team Leads, DevOps  
**Purpose:** End-to-end view of how infrastructure and application changes get from code to a running environment (dev, QA, prod).

---

## 1. Overview

We have **two kinds of deployments**:

| Type | What is deployed | Where it runs | Pipeline / trigger |
|------|------------------|---------------|--------------------|
| **Infrastructure** | Terraform (VPC, EKS, RDS, S3, etc.) | AWS account per env | Jenkins: plan → approval → apply |
| **Applications** | Container images (12 Java + 1 Python) and React static build | EKS (backend) and S3/CloudFront (frontend) | Jenkins: build → test → push image / upload assets → deploy |

Infra changes are **separate** from app releases: infra is applied from the infra repo (or infra path); app deployments use built artifacts and deploy to an env that already has the right VPC, EKS, and data services.

---

## 2. Infrastructure Deployment (Terraform)

### 2.1 Flow

1. **Change:** Edits under `infra/modules/` or `infra/envs/<env>/` (e.g. dev, qa, prod).
2. **PR:** CI runs `terraform fmt`, `terraform validate`, `terraform plan` for the target env. Plan output is reviewed.
3. **Merge:** Code is merged to the main (or release) branch.
4. **Pipeline run:** Jenkins runs again (e.g. on main). Stages: checkout → init → validate → **plan** → save plan artifact.
5. **Approval:** A **manual approval** gate is required. Only authorized people (e.g. Team Lead or DevOps) approve.
6. **Apply:** Jenkins runs `terraform apply` with the saved plan. State is updated in S3; DynamoDB lock prevents concurrent applies.
7. **Result:** AWS resources for that env are created or updated (e.g. dev-vpc, EKS cluster, RDS, S3). No app code or container images are deployed in this pipeline.

### 2.2 Per environment

- Each environment has its own Terraform state (e.g. `dev/platform/terraform.tfstate`, `prod/platform/terraform.tfstate`). Deploying “to dev” means running the pipeline with `TF_DIR=infra/envs/dev` (or equivalent); deploying to prod means running with prod backend and tfvars.
- Typically: **dev** is applied first and more often; **QA** and **prod** after review and with explicit approval for that env.

### 2.3 Who and how

- **Trigger:** Push/merge to the branch that the infra pipeline watches (e.g. `main`).
- **Approval:** Manual step in Jenkins before `terraform apply`.
- **Identity:** Pipeline uses OIDC/assume-role (no static AWS keys). See `AWS-Terraform-Jenkins-OIDC-Implementation-Plan.md`.

---

## 3. Backend Application Deployment (Java + Python on EKS)

### 3.1 What gets deployed

- **12 Java microservices** (Spring Boot): each built as a Docker image, pushed to ECR, then deployed as Kubernetes Deployments (and optionally Services/Ingress) in the env’s EKS cluster.
- **1 Python AI service:** same pattern — Docker image → ECR → Kubernetes Deployment in EKS.

All run inside the **environment’s VPC** (e.g. dev-vpc, qa-vpc, prod-vpc) in that env’s EKS cluster.

### 3.2 High-level flow

1. **Code change:** Application code (and/or Dockerfile, Helm chart) in the app repo.
2. **Build:** Jenkins (or CI) checks out repo, runs tests, builds the Docker image(s) for the changed service(s).
3. **Tag:** Image is tagged (e.g. `git commit SHA`, or `branch-<name>-<build#>`) for traceability.
4. **Push:** Image is pushed to **ECR** in the same account/region as the target env (e.g. dev ECR repo).
5. **Deploy to EKS:** Pipeline updates Kubernetes so the env’s EKS cluster uses the new image:
   - **Option A — kubectl:** Update the Deployment’s image tag and run `kubectl apply` (or `kubectl set image`). Kubernetes performs a **rolling update** by default (replaces pods gradually).
   - **Option B — Helm:** Run `helm upgrade` with the new image tag in values. Same idea: rolling update unless a different strategy is configured.
   - **Option C — Blue/green:** If adopted, the pipeline deploys to a “green” set of resources, then switches traffic (e.g. Service selector or Ingress) from “blue” to “green” after validation. Rollback = switch back to blue.
6. **Result:** New pods run with the new image; old pods are terminated. Apps read config and secrets from ConfigMaps and Secrets Manager (or injected env) as per [Environment-Variables-and-Secrets-Management.md](./Environment-Variables-and-Secrets-Management.md).

### 3.3 Deployment strategy (Kubernetes)

- **Rolling update (default):** New pods are created and old ones removed gradually. No second full copy of the deployment; lower cost, acceptable for most dev/QA and many prod workloads. Rollback = revert the image tag and redeploy, or use `kubectl rollout undo` if the previous revision is still available.
- **Blue/green:** Two full sets of pods; traffic is switched in one step. Faster rollback (switch back). Requires more resources and pipeline logic. Use if release/rollback SLA demands it.
- **Who decides:** Team Lead / DevOps. Strategy is configured in the Deployment (e.g. `maxSurge`, `maxUnavailable`) or in the Helm chart / pipeline.

### 3.4 Per environment

- **Dev:** Often auto-deploy on merge to a dev branch (e.g. `develop`) or on demand; minimal or no approval.
- **QA:** Deploy after merge to a release/QA branch; may require approval before deploy to QA.
- **Prod:** Deploy only from a protected branch (e.g. `main` or `release/*`) with **approval** and, if needed, a change window. Image is usually the same one already validated in QA (same tag promoted).

### 3.5 Who and how

- **Trigger:** Push/merge to the branch that the app pipeline watches; or manual “Deploy to dev/QA/prod” job with branch/tag selection.
- **Approval:** Per env policy (e.g. prod always; QA optional; dev often none).
- **Identity:** Jenkins agents use the same OIDC/IRSA pattern where they need to push to ECR and call the EKS API (e.g. `kubectl` or Helm). EKS cluster access is configured (e.g. IAM role for Jenkins + `aws eks update-kubeconfig` or in-cluster service account).

---

## 4. Frontend Deployment (React on S3 + CloudFront)

### 4.1 What gets deployed

- **1 React app:** Built (e.g. with Vite) into static assets (HTML, JS, CSS). These are uploaded to the **S3** bucket used for static hosting and served to users via **CloudFront**.

### 4.2 High-level flow

1. **Code change:** React app code (and/or build config) in the frontend repo.
2. **Build:** CI runs the frontend build (e.g. `npm run build` / `vite build`). Output is a directory (e.g. `dist/`).
3. **Upload:** Pipeline uploads the contents of `dist/` to the **S3** bucket configured for that env (e.g. dev portal bucket). Objects are overwritten; optional versioning or prefix by build id can be used.
4. **Invalidate cache:** Pipeline calls CloudFront **invalidation** (e.g. `/*` or key paths) so edge caches serve the new assets.
5. **Result:** Next user requests get the new frontend from CloudFront (or S3 if not cached). No Kubernetes or container restart involved.

### 4.3 Per environment

- Each env has its own S3 bucket (and optionally its own CloudFront distribution), or a shared bucket with prefix per env. Pipeline uses the correct bucket and distribution id for the target env (dev/QA/prod).

### 4.4 Who and how

- **Trigger:** Push/merge to the branch the frontend pipeline watches; or manual “Deploy frontend to dev/QA/prod.”
- **Approval:** Same idea as backend (stricter for prod).
- **Identity:** Jenkins needs permission to upload to S3 and create CloudFront invalidations (same OIDC/assume-role or deployer role).

---

## 5. End-to-End Picture (Text Diagram)

```
Infrastructure (infra repo or path)
  Code (Terraform) → PR → Plan (CI) → Merge → Jenkins → Plan again → [Approval] → Apply
  → AWS updated (VPC, EKS, RDS, S3, etc.) for that env

Backend (app repo)
  Code (Java/Python) → PR → Build & Test → Merge → Jenkins → Build image → Push ECR
  → Deploy to EKS (kubectl/Helm) → [Approval for QA/Prod] → New pods run

Frontend (app repo)
  Code (React) → PR → Build → Merge → Jenkins → Build assets → Upload S3 → Invalidate CloudFront
  → [Approval for QA/Prod] → New UI live
```

---

## 6. Approval and Safety

| Deployment type | Typical approval |
|----------------|------------------|
| **Infra – dev** | Manual approval before `terraform apply` in Jenkins |
| **Infra – QA/Prod** | Same; separate pipeline run or parameter for env; explicit approval for that env |
| **Backend – dev** | Often auto after merge; optional approval |
| **Backend – QA** | Optional approval before deploy to QA |
| **Backend – prod** | Approval required; optionally same image as QA (tag promotion) |
| **Frontend – dev/QA/prod** | Same pattern as backend per env |

- **No direct production deploys** from developer laptops unless explicitly allowed by policy; production deploys run in Jenkins with the right role and audit trail.
- **Secrets and config** are not baked into images; they come from Secrets Manager and ConfigMaps as per [Environment-Variables-and-Secrets-Management.md](./Environment-Variables-and-Secrets-Management.md).

---

## 7. Rollback

| What | How to rollback |
|------|------------------|
| **Infra** | Revert the Terraform change in Git and run plan → approval → apply again; or fix-forward with a new change. State is in S3; avoid editing state by hand. |
| **Backend (K8s)** | Redeploy the previous image tag (e.g. `kubectl set image` or Helm with old tag), or `kubectl rollout undo deployment/<name>`. For blue/green, switch traffic back to the previous set. |
| **Frontend** | Redeploy a previous build (re-upload that build’s `dist/` to S3 and invalidate CloudFront), or keep versioned prefixes and switch CloudFront origin/behavior if configured that way. |

---

## 8. Who Does What (RACI-style)

| Activity | Developer | Team Lead | DevOps |
|----------|------------|-----------|--------|
| Push code, open PR (infra or app) | ✓ | ✓ | ✓ |
| Approve infra apply in Jenkins | — | ✓ (or delegated) | ✓ |
| Approve app deploy to QA/Prod in Jenkins | — | ✓ (or delegated) | ✓ |
| Configure pipelines (stages, env, approval) | — | Review | Implement |
| Deploy to dev (after merge or on demand) | ✓ (if allowed) | ✓ | ✓ |
| Deploy to prod | — | ✓ (or delegated) | ✓ |
| Rollback infra or app | — | ✓ | ✓ (execute) |

---

## 9. Quick Reference

- **Infra:** Terraform in Jenkins → plan → **approval** → apply per env. One VPC per env (dev-vpc, qa-vpc, prod-vpc). See [Infrastructure-Organization-Guide.md](./Infrastructure-Organization-Guide.md) and `AWS-Terraform-Jenkins-OIDC-Implementation-Plan.md`.
- **Backend:** Build → push image to ECR → deploy to EKS (rolling or blue/green). Approval for QA/Prod. Config/secrets from ConfigMaps and Secrets Manager.
- **Frontend:** Build → upload to S3 → CloudFront invalidation. Approval per env policy.
- **Rollback:** Infra = revert + apply; Backend = previous image or rollout undo; Frontend = redeploy previous build.

---

## 10. Related Docs

- [Infrastructure-Organization-Guide.md](./Infrastructure-Organization-Guide.md) — How infra is organized (modules, envs, VPC per env).
- [Environment-Variables-and-Secrets-Management.md](./Environment-Variables-and-Secrets-Management.md) — How config and secrets are provided to apps at runtime.
- `AWS-Terraform-Jenkins-OIDC-Implementation-Plan.md` — Infra pipeline and OIDC/assume-role setup.

If your team’s pipelines differ (e.g. separate “Plan” and “Apply” jobs, or GitHub Actions for apps), update this doc to match so it stays the single source of truth for how deployments happen.
