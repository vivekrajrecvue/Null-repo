# Jenkins → OCI/AWS Workflow: Authentication Options  
## Executive Brief

**Audience:** Senior Director  
**Purpose:** Present viable approaches for secure, credential-free Jenkins-to-cloud (AWS/OCI) deployment pipelines.  
**Length:** ~3 pages

---

## 1. Executive Summary

Jenkins does **not** natively issue OIDC tokens that AWS (or OCI) can trust. To run Terraform or other cloud workloads from Jenkins without long-lived access keys, we must introduce an **identity bridge**: either use Jenkins as an OIDC issuer (with caveats), use an **external OIDC broker** (e.g., Okta, Keycloak), or use **GitHub OIDC** for auth while Jenkins orchestrates execution.

| Option | Approach | Best when |
|--------|----------|-----------|
| **1** | Jenkins as OIDC issuer → AWS IAM | Jenkins is already public HTTPS and you accept key/issuer management. |
| **2** | External OIDC broker (Keycloak/Okta) | You already have an IdP; want clear separation of auth and CI. |
| **3** | GitHub OIDC + Jenkins | You use GitHub; want AWS-native, low-friction auth and minimal secrets. |

---

## 2. Problem Statement

- **Goal:** Jenkins runs pipelines that deploy to AWS (and/or OCI) using **temporary credentials** (no long-lived keys in Jenkins).
- **Constraint:** AWS (and OCI) trust **OIDC identity providers**. Jenkins does not act as a standard, AWS-trustable OIDC issuer out of the box.
- **Implication:** We need one of: (a) make Jenkins an OIDC issuer, (b) use an external IdP that Jenkins and AWS both trust, or (c) use another system (e.g., GitHub Actions) to obtain credentials and pass them to Jenkins.

---

## 3. Options in Detail

### Option 1: Jenkins as OIDC Issuer → AWS IAM Role

**Idea:** Configure Jenkins to expose an OIDC-compatible endpoint; in AWS, create an IAM OIDC identity provider with that issuer (e.g. `https://jenkins.yourdomain.com`) and an IAM role whose trust policy allows federation from that provider. The pipeline then requests short-lived AWS credentials via the OIDC flow.

**Steps (high level):**

1. Expose Jenkins over **public HTTPS** with a stable issuer URL.
2. In **AWS**: Create IAM OIDC provider (issuer = Jenkins URL).
3. Create an **IAM role** with a trust policy that allows the Jenkins OIDC provider and restricts the audience/claims.
4. In the **Jenkins pipeline**: call AWS STS (e.g. `AssumeRoleWithWebIdentity`) using the token issued by Jenkins.

**Advantages**

- Single system: no extra IdP to run.
- Direct Jenkins → AWS integration once configured.

**Disadvantages**

- **Jenkins must be publicly reachable over HTTPS** (security and compliance concern).
- **Key/issuer lifecycle**: certificate and OIDC signing key rotation become critical; misconfiguration can break all pipelines.
- Operational burden for maintaining a secure, compliant OIDC issuer.

**Verdict:** Use only if Jenkins is already the designated public OIDC issuer and you have strong PKI and key-rotation practices.

---

### Option 2: External OIDC Broker (e.g. Keycloak, Okta)

**Idea:** Jenkins does **not** issue tokens. An external IdP (Keycloak, Okta, etc.) issues OIDC tokens; AWS trusts that IdP. Jenkins (or a step in the pipeline) obtains a token from the IdP and exchanges it with AWS STS for temporary credentials. Terraform (or other tools) run with these credentials (e.g. via environment variables).

**Steps (high level):**

1. **IdP:** Configure client and scopes so Jenkins (or a service account) can request tokens (e.g. JWT) for AWS.
2. **AWS:** Create IAM OIDC provider for the IdP’s issuer URL; create IAM role with trust policy for that provider.
3. **Pipeline:**  
   - Trigger pipeline (e.g. on commit).  
   - Pull Terraform (or other) code.  
   - Request token from IdP (e.g. client credentials or token exchange).  
   - Call AWS STS (e.g. `AssumeRoleWithWebIdentity`) with that token.  
   - Set returned credentials as env vars; run Terraform/CLI.

**Advantages**

- **Separation of concerns:** Auth lives in a dedicated IdP; Jenkins is “just” a consumer.
- No need for Jenkins to be public or to act as an OIDC issuer.
- Aligns with enterprise SSO (Okta/Keycloak already in use).
- Centralized user and access policy in the IdP.

**Disadvantages**

- **Extra dependency:** IdP must be highly available and secure.
- **Integration work:** Token request and STS exchange must be implemented and maintained (scripts or plugins).
- Operational ownership of the IdP and its configuration.

**Verdict:** Strong choice when an enterprise IdP is already in place and you want clear boundaries between identity and CI.

---

### Option 3: GitHub OIDC + Jenkins Orchestration

**Idea:** Use **GitHub’s OIDC** (supported natively by AWS) to obtain temporary AWS credentials. Jenkins does not issue or broker OIDC; it either **consumes** credentials produced by GitHub (Method A) or is used only as the **execution engine** while GitHub handles only the auth step (Method B).

**Common setup:**

- **AWS:** Create IAM OIDC provider with issuer `https://token.actions.githubusercontent.com` and IAM role with a trust policy that restricts by repo, branch, and optionally workflow/environment.

**Method A – GitHub does auth; Jenkins consumes credentials**

1. GitHub Action runs (e.g. on push), requests OIDC token from GitHub, exchanges with AWS STS.
2. Action receives temporary AWS credentials and passes them **securely** to Jenkins (e.g. via parameterized build with credentials or a secure channel).
3. Jenkins runs Terraform/scripts using those credentials.

**Method B – GitHub does only auth; Jenkins is the execution engine**

1. GitHub Action runs only an “auth” job: get OIDC token → exchange with AWS STS → store credentials in a short-lived store (e.g. Vault, AWS Secrets Manager, or secure parameter).
2. Same or another job triggers Jenkins (e.g. via API) to run the actual pipeline; Jenkins retrieves the temporary credentials and runs Terraform/scripts.

**Advantages**

- **AWS-native:** GitHub OIDC is well documented and widely used; no custom OIDC issuer.
- **Minimal secrets:** No long-lived AWS keys in GitHub or Jenkins when done correctly.
- **Fine-grained trust:** IAM policy can limit which repo/branch/workflow can assume the role.
- Method B keeps “heavy” execution in Jenkins while centralizing auth in GitHub.

**Disadvantages**

- **GitHub dependency:** Requires GitHub (or similar) in the flow; not suitable if all CI must stay inside Jenkins only.
- **Secure handoff:** Passing credentials from GitHub to Jenkins must be done carefully (secure parameters, no logging).

**Verdict:** Best when the codebase and workflows already use GitHub and you want the simplest, most maintainable path to credential-free AWS access.

---

## 4. Comparison Summary

| Criteria | Option 1 (Jenkins OIDC) | Option 2 (External IdP) | Option 3 (GitHub OIDC) |
|----------|-------------------------|--------------------------|-------------------------|
| **Jenkins public?** | Yes (required) | No | No |
| **Extra components** | None | IdP (Keycloak/Okta) | GitHub (Actions) |
| **Key/cert burden** | High (Jenkins issuer) | In IdP | Low (GitHub-managed) |
| **Enterprise IdP fit** | Weak | Strong | Depends (GitHub as IdP) |
| **Implementation effort** | Medium | Medium–High | Low–Medium |
| **Best for** | Jenkins-centric, public Jenkins | Existing Okta/Keycloak | GitHub-centric workflows |

---

## 5. Recommendation and Next Steps

- **Default recommendation:** Prefer **Option 3 (GitHub OIDC)** if the organization uses GitHub; it is the most straightforward and keeps long-lived keys out of both GitHub and Jenkins.
- If GitHub is not in scope, prefer **Option 2 (External OIDC broker)** where an IdP already exists; it keeps auth out of Jenkins and avoids making Jenkins a public OIDC issuer.
- Choose **Option 1** only when Jenkins must be the sole identity source and you can meet the security and operational requirements (public HTTPS, key rotation, compliance).

**Suggested next steps:**

1. Confirm whether GitHub (or similar) is part of the approved toolchain.
2. If yes: pilot Option 3 (Method A or B) in one repo; document the credential handoff and IAM trust policy.
3. If no: validate IdP availability (Keycloak/Okta) and scope Option 2; design the token request and STS exchange flow.
4. For any option: define ownership (security, platform, development) and document the chosen approach in the organization’s runbooks.

---

*Document prepared for senior leadership. Technical details (exact IAM policies, pipeline snippets) can be appended or linked in an annex.*
