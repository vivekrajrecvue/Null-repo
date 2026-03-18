# Do's and Don'ts – Recvue Dev/QA/Prod Setup

Use this as a quick checklist for safe and consistent work on AWS, Terraform, and EKS. Do's and don'ts are mixed in one list.

---

1. **Do** use IAM credentials via environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) or AWS profile (`AWS_PROFILE`); **don't** put Access Key ID or Secret Access Key in `.tf`, `.tfvars`, or any file committed to git.

2. **Do** store secrets in AWS Secrets Manager or Parameter Store and reference them from Terraform or app config; **don't** hardcode DB passwords, API keys, or tokens in code or config files.

3. **Do** add `backend.tfvars`, `.env`, and any tfvars that contain secrets to `.gitignore`; **don't** commit files that contain credentials.

4. **Do** use a remote Terraform backend (S3 + DynamoDB lock) with a different state key per environment (e.g. `dev/terraform.tfstate`); **don't** keep state only on your laptop or in a shared drive.

5. **Do** run `terraform plan` and review it before every `terraform apply`; **don't** run `apply` without reviewing, especially for prod.

6. **Do** use separate Terraform state per environment so dev, QA, and prod never share one state file; **don't** use a single state file for all environments.

7. **Do** tag all resources (e.g. `Environment = dev`, `Project = recvue`); **don't** create resources without consistent tags.

8. **Do** use separate VPCs per environment (e.g. `recvue-dev-vpc`, `recvue-qa-vpc`, `recvue-prod-vpc`) with different CIDR ranges; **don't** put dev, QA, and prod in the same VPC or use overlapping CIDRs if you may peer VPCs.

9. **Do** put EKS nodes and Aurora in private subnets and use NAT or VPC endpoints for outbound traffic; **don't** put application or database instances in public subnets unless there is a clear requirement.

10. **Do** restrict security groups by port and source (e.g. ALB → app only, app → DB only); **don't** use `0.0.0.0/0` for ingress on application or database security groups.

11. **Do** set resource requests and limits on pods (CPU/memory) so the scheduler and autoscaler work correctly; **don't** run pods without requests/limits.

12. **Do** use 1 pod per microservice in dev and 2 pods per microservice in QA (via Helm/Kustomize or Deployment replicas); **don't** use the same replica count everywhere without documenting why.

13. **Do** right-size EKS node types for your workload (e.g. 13 microservices at 1 GB each); **don't** oversize nodes for dev or leave capacity undocumented.

14. **Do** use RDS Proxy for connection pooling when many pods talk to the DB; **don't** rely only on direct DB connections from every pod without pooling.

15. **Do** enable S3 versioning and encryption on the Terraform state bucket; **don't** use an unencrypted or non-versioned bucket for state.

16. **Do** apply stronger controls in prod (WAF, backups, stricter IAM) and treat prod changes as higher risk with review and rollback plan; **don't** use the same security and change process for dev and prod.

17. **Do** take automated backups for Aurora/RDS and test restore; in prod use a retention policy that meets RTO/RPO; **don't** skip backups in prod or never test restores.

18. **Do** pin provider and module versions in Terraform (`required_providers`, version in module source); **don't** use unversioned or `latest` in production.

19. **Do** use one NAT Gateway (or VPC endpoints) in dev to control cost and consider two in prod for HA; **don't** add multiple NAT Gateways in dev without a real need.

20. **Do** document instance types, replica counts, and environment-specific choices (e.g. in DEV-SETUP-PLAN, RESOURCES-CHECKLIST); **don't** change sizing or topology without updating docs.

21. **Do** use meaningful resource names and tags (e.g. `recvue-dev-alb`) so resources are identifiable; **don't** rely only on generic or auto-generated names.

22. **Do** prefer one IAM user per environment with least privilege for Terraform; **don't** use a single shared IAM user with broad permissions across all environments.

23. **Do** prefer VPC endpoints (e.g. S3, ECR) in private subnets to reduce NAT cost and improve security where it makes sense; **don't** ignore cost and rely only on NAT for everything in long-lived environments.

24. **Do** review costs periodically (Cost Explorer, tags by Environment/Project); **don't** ignore cost until the first bill.

25. Before any `terraform apply`, **do** confirm credentials are from env/profile, state is remote with the correct backend key for this environment, the plan is reviewed, you're targeting the right environment (not prod by mistake), and no sensitive values are in committed files; **don't** apply without this quick check.

---

*Adjust any item to match your org’s policies (e.g. separate accounts per env, SSO instead of IAM users).*
