# Jenkins → OCI/AWS Workflow: Authentication Options  
## Executive Brief

**Audience:** Senior Director  
**Purpose:** Present viable approaches for secure, credential-free Jenkins-to-cloud (AWS/OCI) deployment pipelines.  
**Scope:** **Jenkins-only** — all pipelines run in Jenkins; GitHub Actions (or other external CI) are not in use.  
**Length:** ~3 pages

---

## 1. Executive Summary

Jenkins does **not** natively issue OIDC tokens that AWS (or OCI) can trust. With **Jenkins as the only pipeline engine** (no GitHub Actions), we have **two feasible options**: (1) make Jenkins an OIDC issuer and federate to AWS IAM, or (2) use an **external OIDC broker** (e.g. Keycloak, Okta) that issues tokens to Jenkins and is trusted by AWS. Both are feasible; the choice depends on whether Jenkins can be public HTTPS and whether an enterprise IdP is available.

| Option | Approach | Best when |
|--------|----------|-----------|
| **1** | Jenkins as OIDC issuer → AWS IAM | Jenkins is already (or can be) public HTTPS; you accept key/issuer management. |
| **2** | External OIDC broker (Keycloak/Okta) | You have (or can introduce) an IdP; want Jenkins to stay private and avoid running an OIDC issuer. |

---

## 2. Problem Statement

- **Goal:** Jenkins runs pipelines that deploy to AWS (and/or OCI) using **temporary credentials** (no long-lived keys in Jenkins).
- **Constraint:** AWS (and OCI) trust **OIDC identity providers**. Jenkins does not act as a standard, AWS-trustable OIDC issuer out of the box.
- **Implication (Jenkins-only):** We need either (a) make Jenkins an OIDC issuer and have AWS trust it, or (b) use an external IdP that Jenkins calls for tokens and AWS trusts for federation. No GitHub or other external CI is in scope.

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

**Verdict:** Strong choice when an enterprise IdP is already in place (or can be introduced) and you want clear boundaries between identity and CI — **fully feasible with Jenkins-only pipelines.**

---

### Out of scope: GitHub OIDC

Using **GitHub Actions** to obtain OIDC tokens and pass credentials to Jenkins is **not in scope** for this brief, as we are standardizing on **Jenkins-only** for pipeline execution. If that constraint changes in the future, GitHub OIDC can be revisited.

---

## 4. Comparison Summary (Jenkins-only)

| Criteria | Option 1 (Jenkins OIDC) | Option 2 (External IdP) |
|----------|-------------------------|--------------------------|
| **Jenkins public?** | Yes (required) | No |
| **Extra components** | None | IdP (Keycloak/Okta) |
| **Key/cert burden** | High (Jenkins issuer) | In IdP |
| **Enterprise IdP fit** | Weak | Strong |
| **Implementation effort** | Medium | Medium–High |
| **Feasibility (Jenkins-only)** | Yes | Yes |

**Conclusion:** Both options are **feasible** with Jenkins as the sole pipeline engine. Option 2 is generally preferable when an IdP exists or can be adopted, to avoid exposing Jenkins and managing OIDC issuer lifecycle.

---

## 5. Recommendation and Next Steps

- **With Jenkins-only pipelines:** Prefer **Option 2 (External OIDC broker)** if an IdP (Keycloak, Okta, or similar) is available or can be introduced — no need for Jenkins to be public, and auth stays in a dedicated IdP.
- Choose **Option 1 (Jenkins as OIDC issuer)** when Jenkins must be the sole identity source, it is (or can be) exposed over public HTTPS, and the organization can own key rotation and OIDC issuer compliance.

**Suggested next steps:**

1. Confirm whether an enterprise IdP (Keycloak, Okta, etc.) is in place or approved for use.
2. If yes: scope **Option 2** — design the token request and AWS STS exchange in the Jenkins pipeline; create IAM OIDC provider and role for the IdP.
3. If no (and Jenkins can be public): scope **Option 1** — expose Jenkins over HTTPS, configure OIDC issuer, create AWS IAM OIDC provider and role; establish key/cert rotation.
4. Define ownership (security, platform, development) and document the chosen approach in runbooks.

---

*Document prepared for senior leadership. Technical details (exact IAM policies, pipeline snippets) can be appended or linked in an annex.*
