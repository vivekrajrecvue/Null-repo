# Databases and Backups

**Audience:** Developers, Team Leads, DevOps  
**Purpose:** What databases we use, how they are backed up, and how to restore when needed. Keeps backup strategy and ownership clear across environments.

---

## 1. Overview

We use **AWS-managed databases** per environment (one VPC per env). The main data store is **PostgreSQL** (RDS or Aurora); **Redis** (ElastiCache) is used for caching. Database credentials are in **AWS Secrets Manager** and are never stored in code or repo (see [Environment-Variables-and-Secrets-Management.md](./Environment-Variables-and-Secrets-Management.md)).

| Component | Role | Backed up? |
|-----------|------|------------|
| **PostgreSQL (RDS / Aurora)** | Primary database: app data, AI agent memory, RLS (tenant_id + entity_id) | Yes — automated backups + optional manual snapshots |
| **RDS Proxy** | Connection pooling in front of PostgreSQL | No — stateless; no backup |
| **Redis (ElastiCache)** | API cache, session acceleration | Optional — snapshot if required by policy |
| **Weaviate** (vector DB) | RAG / AI vectors | Per deployment choice (EKS or managed); backup strategy TBD per offering |

This document focuses on **PostgreSQL backups and restore**; Redis and Weaviate are summarized briefly.

---

## 2. PostgreSQL (RDS / Aurora) — Where and How

- **Service:** AWS RDS for PostgreSQL or Amazon Aurora PostgreSQL (version 15+). Choice is per environment (e.g. Aurora Serverless v2 for dev, Aurora provisioned for prod).
- **Placement:** Each environment has its own PostgreSQL instance(s) inside that env’s **private DB subnets** (e.g. dev-vpc, qa-vpc, prod-vpc). No direct internet access; only app tier (EKS) can connect via security groups.
- **Encryption:** Data at rest is encrypted with **AWS KMS** (ADR-AUTH-005). Backups inherit the same encryption.
- **Access:** Applications connect using credentials from **Secrets Manager**; optionally via **RDS Proxy** for connection pooling. No DB passwords in env vars or ConfigMaps.
- **Schema and migrations:** **Flyway** (ADR-DB-002) runs schema migrations at application startup. Phase 1 uses additive-only migrations; fix-forward for issues (no automated rollback of migrations).

---

## 3. PostgreSQL Backups

### 3.1 Automated backups (managed by AWS)

- **RDS:** Automated daily snapshots and **continuous backup** for Point-in-Time Recovery (PITR). Retention is configurable per instance.
- **Aurora:** Automated backups are always enabled; PITR is supported. Cluster volume is continuously backed up; retention is configurable (e.g. 1–35 days).

Backup window (maintenance window) is set in Terraform per env (e.g. low-traffic period). No application code is required; AWS performs backups.

### 3.2 Recommended retention by environment

| Environment | Automated backup retention | Notes |
|-------------|----------------------------|--------|
| **Dev** | 7 days | Balance cost and ability to restore after mistakes. |
| **QA** | 7–14 days | Enough for test data and failed upgrade recovery. |
| **Prod** | 30 days (or per compliance) | Align with RPO; extend if required by policy. |

Exact values are defined in Terraform (e.g. `infra/modules/rds_postgres/` or env tfvars: `backup_retention_period`).

### 3.3 Manual snapshots

- **When:** Before major schema or data changes (e.g. Flyway migration that is not purely additive), before bulk data fixes, or before upgrading PostgreSQL minor version.
- **How:** Create via AWS Console (RDS → Snapshots → Create snapshot) or CLI/API. Name with date and purpose (e.g. `prod-portal-pre-migration-2025-03-17`).
- **Retention:** Manual snapshots are kept until explicitly deleted. Prefer deleting old manual snapshots to avoid cost growth; keep at least one known-good snapshot for critical restores if policy requires.

---

## 4. PostgreSQL Restore

### 4.1 Restore from automated backup (PITR or snapshot)

- **Point-in-Time Recovery (PITR):** Restore the instance to any second within the backup retention window. Use when you need to recover to a moment before a bad write or failed migration. Results in a **new** instance; you then point apps (or RDS Proxy) to the new endpoint or replace the old instance after validation.
- **Snapshot restore:** Restore from a specific automated or manual snapshot. Also creates a new instance. Use when you need a known-good point (e.g. “restore from snapshot from yesterday”).

Steps (high level):

1. In AWS Console (or CLI): RDS → Restore (from snapshot or to point in time). Choose the correct env (e.g. prod) and region.
2. Configure the restored instance (same or different size, same VPC and DB subnets, same KMS key so encryption is consistent).
3. Restore to a **new** instance name/endpoint so the existing instance is not overwritten.
4. After restore: run app health checks or Flyway (if needed) against the new instance; update **RDS Proxy** or app config to point to the new endpoint if you are switching over.
5. When switching production traffic to the restored instance, follow your change process (e.g. Team Lead/DevOps approval, window). Decommission the old instance only after validation.

### 4.2 Who performs restore

- **Dev/QA:** Developers or DevOps can restore when needed for recovery or testing.
- **Prod:** **DevOps** (or designated person) performs the restore. **Team Lead approval** (or per your policy) before switching traffic to the restored prod instance. Document the restore in your incident or change log.

### 4.3 After restore — migrations and data

- If the restore is to a point **before** a bad Flyway migration, ensure the migration is fixed (or reverted in code) before re-running deployments; otherwise the same migration may run again and fail or cause the same issue.
- Fix-forward approach (ADR-DB-002): Prefer fixing the migration and re-applying rather than relying on Flyway “rollback” scripts. Use restored backups to get to a good state, then deploy corrected code/migrations.

---

## 5. Schema Migrations (Flyway) and Backups

- **Flyway** runs at application startup and applies pending migrations in order. Migrations are versioned SQL files in the application repo.
- **Backup before big changes:** Before deploying a migration that is not purely additive (e.g. dropping a column, big data change), create a **manual snapshot** so you can restore if the migration fails or causes data issues.
- **Restore + re-deploy:** If you restore from a backup taken before a failed migration, the DB is in the previous schema state. Fix the migration (or revert the code change), then redeploy so Flyway runs the corrected migration (or no migration if reverted). Do not rely on automatic “down” migrations unless your team has adopted and tested them.

---

## 6. Redis (ElastiCache) — Backups

- **Role:** Cache and session acceleration; data can be recreated from PostgreSQL or app logic. Not the source of truth.
- **Backup:** ElastiCache supports **automated snapshots** (configurable retention). For dev, short retention (e.g. 1 day) or none is often acceptable. For prod, enable snapshots if you need to recover cache state (e.g. 1–7 days).
- **Restore:** Restore from snapshot creates a new cluster; point the application to the new endpoint (via ConfigMap or Secrets Manager). Less critical than PostgreSQL; often “flush and repopulate” is acceptable for cache.

---

## 7. RDS Proxy

- **Role:** Connection pooling between apps and PostgreSQL. No data stored; no backup required.
- **Failover:** If you restore or replace the PostgreSQL instance, update the RDS Proxy target to the new instance endpoint (or recreate the proxy in Terraform pointing to the new instance). Applications keep using the proxy endpoint.

---

## 8. Who Does What (RACI-style)

| Activity | Developer | Team Lead | DevOps |
|----------|-----------|-----------|--------|
| Write and version Flyway migrations | ✓ | Review | — |
| Create manual snapshot before risky migration | Request / suggest | — | ✓ (or delegated) |
| Configure backup retention in Terraform | — | Review | ✓ |
| Restore from backup (dev/QA) | ✓ (if allowed) | — | ✓ |
| Restore from backup (prod) and switch traffic | — | Approve | ✓ (execute) |
| Delete old manual snapshots | — | — | ✓ |
| Handle Redis snapshot/restore | — | — | ✓ |

---

## 9. RTO / RPO (High-Level)

- **RPO (Recovery Point Objective):** How much data loss is acceptable. With PITR, you can restore to any second within the retention window; effective RPO is near zero for that window. Retention length (e.g. 7 vs 30 days) is a policy choice per env.
- **RTO (Recovery Time Objective):** How quickly you must be back up. Depends on restore time (AWS restore + validation + traffic switch). For prod, define target RTO and test restore periodically.
- **Dev:** Relaxed RTO/RPO; 7-day retention is usually enough.  
- **Prod:** Stricter; 30-day (or more) retention and documented restore runbook. Run a restore drill at least occasionally.

---

## 10. Quick Reference

- **PostgreSQL:** RDS or Aurora 15+, per env, in private subnets. Encrypted with KMS. Credentials in Secrets Manager; optional RDS Proxy.
- **Backups:** Automated backups + PITR; retention 7 days (dev/QA) and 30 days (prod) as a baseline. Manual snapshots before major migrations or risky changes.
- **Restore:** New instance from snapshot or PITR; point RDS Proxy/app to new endpoint after validation. Prod restore requires approval.
- **Flyway:** Additive migrations at startup; fix-forward on failure. Take a manual snapshot before non-additive or high-impact migrations.
- **Redis:** Optional automated snapshots; restore or repopulate as needed. RDS Proxy: no backup; update target after DB restore.

---

## 11. Related Docs

- [Infrastructure-Organization-Guide.md](./Infrastructure-Organization-Guide.md) — Where DB and backup resources are defined (Terraform modules, envs, VPC per env).
- [Environment-Variables-and-Secrets-Management.md](./Environment-Variables-and-Secrets-Management.md) — How DB credentials are stored and injected (Secrets Manager, no env vars).
- [How-Deployments-Happen.md](./How-Deployments-Happen.md) — How app deployments (and thus Flyway runs) are triggered.

Backup retention and manual snapshot practices should be reflected in your runbooks and Terraform so the team has a single source of truth for databases and backups.
