# Environment Variables and Secrets Management

**Audience:** Developers, Team Leads, DevOps  
**Purpose:** How we manage environment variables and secrets across environments (dev, QA, prod) so that config is clear, secrets are never in code or repo, and each env stays isolated.

---

## 1. Overview

We separate **secrets** (passwords, API keys, tokens) from **non-sensitive configuration** (URLs, feature flags, log levels). Secrets are stored in **AWS Secrets Manager** and injected at runtime; we do **not** use `.env` files or plain environment variables in the repo for secrets. Non-sensitive config can live in **Kubernetes ConfigMaps**, deployment manifests, or app config and can differ per environment.

---

## 2. Principles

| Type | Where it lives | How apps get it | In repo? |
|------|----------------|-----------------|----------|
| **Secrets** (DB passwords, Okta client secret, API keys, signing keys) | AWS Secrets Manager (per env) | Injected at startup via K8s + IRSA or sidecar/init | **Never** — only secret **names/ARNs** in Terraform/K8s |
| **Non-sensitive config** (service URLs, feature flags, log level, region) | ConfigMaps, `terraform.tfvars`, or app config repo | Env vars or config files mounted from ConfigMaps | **Yes** — values can be in manifests or config; no secrets |

- **No long-lived AWS keys** for app code; workloads use **IRSA** (IAM Roles for Service Accounts) where they need to read from Secrets Manager.
- **One set of secrets per environment** (e.g. dev has its own DB password and Okta client secret; prod has different ones).
- **Rotation:** Secrets are rotated in AWS Secrets Manager; app restarts or refresh logic pick up new values as per our rotation process.

---

## 3. Secrets (AWS Secrets Manager)

### 3.1 What goes in Secrets Manager

- Database credentials (RDS/Aurora username and password)
- Okta client ID and client secret (or other IdP credentials)
- API keys for external services (e.g. RecVue Core API, notification provider)
- Signing keys or other sensitive material (if not in HSM/KMS-only)
- Any value that must not appear in source code, Jenkins pipelines, or ConfigMaps

### 3.2 Naming and per-environment isolation

- **Naming:** Use a consistent naming scheme so Terraform and apps can reference by name. Example: `{env}/portal/{service}/{secret-name}` (e.g. `dev/portal/api/db-password`, `prod/portal/api/db-password`).
- **Per env:** Each environment (dev, QA, prod) has its own secrets in Secrets Manager. Terraform for that env creates/updates only that env’s secrets (e.g. in `dev-vpc` account/region, create `dev/portal/...` secrets).
- **KMS:** Secrets are encrypted at rest using AWS KMS (per ADR). The Terraform deployer role has permissions to create secrets and use the shared KMS key for the env.

### 3.3 How applications get secrets

- **Option A — Env var from Secrets Manager at deploy time:**  
  CI/CD (e.g. Jenkins) or a deploy script assumes a role with access to Secrets Manager, fetches the secret (e.g. JSON key), and injects it as an env var into the K8s Deployment (e.g. `DB_PASSWORD` from `dev/portal/api/db`). The secret value is never stored in Git; only the **secret name or ARN** is in the deployment template.

- **Option B — Runtime read via IRSA (recommended for new apps):**  
  The pod runs with an IAM role (IRSA) that has `secretsmanager:GetSecretValue` on the relevant secret(s). On startup, the application reads the secret from Secrets Manager (e.g. by secret name or ARN passed as a non-sensitive env var like `DB_SECRET_ARN`). This supports rotation without redeploy: after rotating the secret in Secrets Manager, the app can reload (or use a TTL cache) and get the new value.

- **Option C — Sidecar or init container:**  
  A small sidecar or init container with IRSA fetches secrets from Secrets Manager and writes them to a shared volume or env file; the main container reads from that. Useful if the main app cannot use the AWS SDK.

**Recommendation:** Prefer **Option B** (runtime read with IRSA) for new services so rotation does not require a redeploy. Use Option A where existing deployment pipelines already inject at deploy time and rotation is done with a restart.

### 3.4 What appears in code and repo

- **Terraform:** Creates the secret *resource* in AWS and can set initial value (e.g. from a one-time random password). Terraform state may reference the secret ARN; avoid storing the actual secret value in Terraform if possible (e.g. use `aws_secretsmanager_secret` and set value outside Terraform or via a one-time bootstrap).
- **Kubernetes:** Deployment YAML or Helm can reference the secret **by name/ARN** (e.g. `SECRET_ARN=arn:aws:secretsmanager:...:secret:dev/portal/api/db`). No secret value in the manifest.
- **Application code:** Reads from env (e.g. `DB_SECRET_ARN`) and calls Secrets Manager, or reads an env var that was injected at deploy time (Option A). No hardcoded passwords or API keys in source.

---

## 4. Non-Sensitive Configuration (Env Vars and ConfigMaps)

### 4.1 What can be plain env vars or ConfigMaps

- Service endpoints (e.g. `API_GATEWAY_URL`, `RABBITMQ_HOST`)
- Feature flags (e.g. `FEATURE_X_ENABLED=true`)
- Log level (e.g. `LOG_LEVEL=INFO`)
- Region, environment name (e.g. `AWS_REGION`, `ENV=dev`)
- Public or non-sensitive identifiers (e.g. `OKTA_ISSUER_URL`, tenant list for dev)
- Connection strings **without** passwords (e.g. `DB_HOST`, `DB_PORT`, `DB_NAME`); password comes from Secrets Manager.

These can live in:

- **Kubernetes ConfigMaps:** Create a ConfigMap per service or shared (e.g. `portal-api-config`) and expose as env vars or mounted files. Values can be different per env (different ConfigMap per namespace/env).
- **Deployment manifests / Helm values:** Env vars defined in `deployment.yaml` or `values-{env}.yaml`. Safe as long as no secrets are included.
- **App config repo or config service:** If you use a central config service or repo, non-sensitive config can live there and be loaded at startup; ensure that repo does not contain secrets.

### 4.2 Per-environment values

- **Dev:** Point to dev endpoints (dev-vpc), dev Okta issuer, dev RabbitMQ, etc. Use dev ConfigMaps and dev secrets in Secrets Manager.
- **QA / Prod:** Same structure, different values (qa-vpc / prod-vpc, prod Okta, prod RDS, etc.). Each env has its own namespace and its own ConfigMaps and secret names in Secrets Manager.

---

## 5. Where Things Live (Summary)

| Item | Stored / defined where | Consumed by |
|------|------------------------|------------|
| DB password, Okta client secret, API keys | AWS Secrets Manager (per env) | Apps via IRSA + GetSecretValue, or inject at deploy |
| Secret ARN or name (non-sensitive) | K8s Deployment env or ConfigMap | App (to call Secrets Manager or expect injected value) |
| Service URLs, feature flags, log level | ConfigMaps or Helm values per env | Apps as env vars or config files |
| Terraform backend / Jenkins pipeline vars | `backend.hcl`, Jenkins env / credentials (e.g. OIDC, no keys) | Terraform / Jenkins only |
| App “environment name” (dev/qa/prod) | ConfigMap or deployment manifest | App for logging and feature toggles |

---

## 6. Safe Practices

- **Never** commit `.env` files or any file containing secrets. Add `.env`, `*.env`, `secrets.yaml` to `.gitignore`.
- **Never** put secret values in Jenkinsfile as plain text. Use Jenkins credentials (e.g. “Secret text”) only for human-operated jobs if needed; for Terraform and app deploy, use OIDC/IRSA and Secrets Manager.
- **Do** use distinct secret names (and ARNs) per environment so dev never points at prod secrets.
- **Do** restrict IAM (and IRSA) so each service can read only the secrets it needs (resource ARN or path prefix).
- **Do** use the same pattern everywhere: secrets in Secrets Manager, non-sensitive config in ConfigMaps or deployment env.

---

## 7. Who Does What

| Activity | Developer | Team Lead | DevOps |
|----------|-----------|-----------|--------|
| Add a new **non-sensitive** env var (e.g. feature flag, URL) | Add to ConfigMap or Helm values, PR | Review | Merge / deploy |
| Add a new **secret** (e.g. new API key) | Request (name, env, which service) | Approve | Create in Secrets Manager, wire in Terraform/K8s, grant IRSA if needed |
| Rotate a secret (e.g. DB password) | — | Approve if required | Rotate in Secrets Manager; restart or refresh app |
| Change secret naming scheme or Terraform layout | — | Review | Implement |
| Add new service that needs secrets | Implement app code to read from Secrets Manager (or injected env) | Review | Create secret, IRSA policy, and deployment env (e.g. SECRET_ARN) |

---

## 8. Quick Reference

- **Secrets:** AWS Secrets Manager, per env, encrypted with KMS. Apps get them via IRSA (runtime) or deploy-time injection; never in repo.
- **Non-sensitive config:** ConfigMaps or deployment env vars; can be in repo (values or per-env Helm/tfvars).
- **Per env:** Each environment has its own VPC, its own secrets (e.g. `dev/portal/...`, `prod/portal/...`), and its own ConfigMaps/values.
- **IRSA:** Used so pods can call Secrets Manager (and other AWS APIs) without static keys; see Infrastructure and ADR docs for role setup.

---

## 9. Related Docs

- **Infrastructure organization:** [Infrastructure-Organization-Guide.md](./Infrastructure-Organization-Guide.md) — where infra (including Secrets Manager and KMS) is defined (Terraform, envs, VPC per env).
- **CI/CD and OIDC:** `AWS-Terraform-Jenkins-OIDC-Implementation-Plan.md` — how Jenkins gets AWS access without keys (OIDC/assume-role); pipeline env vars (e.g. `TF_DIR`) are non-sensitive.
- **ADR:** ADR-CLD-004 (Secrets Manager), ADR-AUTH-005 (KMS) — decisions on secrets and encryption.

If you need a new secret or are unsure whether something is a secret, ask your Team Lead or DevOps.
