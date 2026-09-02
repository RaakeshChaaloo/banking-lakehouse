# Design Doc: Banking Data Lakehouse — Phase 0

**Author:** [Your Name]
**Date:** [Date]
**Status:** Draft

## 1. Problem Statement

We need a unified data platform that ingests data from core banking systems
(Core Banking DB, Card Processor, CRM, Transaction Files/APIs), cleans and
standardizes it, and makes it available for regulatory reporting, operational
analytics, and AI/ML use cases — with governance (access control, PII
protection, lineage, audit) built in from the start rather than bolted on
later.

## 2. Goals (Phase 1 — this build)

- [ ] Ingest **one** source system end-to-end (Accounts data from Core
      Banking DB) into the lake
- [ ] Build Bronze → Silver → Gold Delta tables for that one data domain
- [ ] Register all tables in Unity Catalog with basic access grants
- [ ] Mask/separate PII fields at the Silver layer
- [ ] Connect one downstream consumer (a simple dashboard on `gold.daily_balances`)
- [ ] Orchestrate the whole flow with a single Airflow DAG

## 3. Non-Goals (explicitly out of scope for Phase 1)

- Card Processor, CRM, and Transaction Files ingestion (Phase 2+)
- Regulatory report generation (AnaCredit/BaFin) (Phase 3+)
- ML models (credit risk, churn, fraud) (Phase 4+)
- Production-grade CI/CD, multi-environment promotion (dev → prod)
- High-availability / DR setup

Keeping this list explicit prevents scope creep — anything not listed here
gets a "not now" by default.

## 4. Success Criteria

Phase 1 is "done" when:
1. A nightly Airflow run pulls Accounts data from source → S3 raw zone →
   Bronze → Silver → Gold without manual intervention
2. `silver.account_pii` is access-restricted; only a designated role can
   query unmasked SSN/address fields
3. A dashboard shows daily balances sourced from `gold.daily_balances`
4. Every table involved appears in Unity Catalog with lineage visible from
   Bronze to Gold
5. Basic tests pass in CI on every pull request

## 5. Data Source (Phase 1)

| Field | Detail |
|---|---|
| Source system | Core Banking DB |
| Table(s) | `accounts` |
| Access method | JDBC / DB extract via AWS Glue |
| Load pattern | Full load initially, incremental (CDC or watermark) after |
| Expected volume | [fill in — rows/day, table size] |
| Owner / contact | [who do you ask when something breaks] |

## 6. Architecture (Phase 1 slice)

```
Core Banking DB
      │
      ▼
AWS Glue (extract) ──► S3 raw/landing zone
      │
      ▼
Airflow DAG (orchestrates all steps below)
      │
      ▼
Databricks Bronze (bronze.accounts)
      │
      ▼
Databricks Silver (silver.account_master, silver.account_pii)
      │
      ▼
Databricks Gold (gold.daily_balances)
      │
      ▼
Dashboard (Databricks SQL / Power BI / Tableau)

Governance (Unity Catalog, IAM/RBAC, masking, lineage, audit)
      wraps every table above
```

## 7. Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Storage format | Delta Lake | ACID transactions, time travel, schema enforcement — needed for banking audit trail |
| Load pattern | Full load first, then incremental | Faster to get working end-to-end; optimize later |
| PII handling | Separate table at Silver layer + masked view | Keeps access control simple: grant/deny at the table level |
| Orchestration | Airflow (Glue + Databricks jobs as separate tasks) | Failure in one layer doesn't require rerunning everything |
| Table granularity in Gold | One table per consumer use case | Avoids one generic "do everything" table that nobody trusts |

## 8. Risks / Open Questions

- [ ] Do we have read access to the Core Banking DB yet, or is that pending
      an access request?
- [ ] What's the actual data volume — does full load fit in a reasonable
      time window?
- [ ] Who owns Unity Catalog admin rights to set up the metastore?
- [ ] Is there a sample/masked dataset available for dev before we get
      access to real data?

## 9. Rough Timeline

| Milestone | Target |
|---|---|
| Infra (S3, IAM, Databricks workspace) ready | Week 1 |
| Accounts source → Bronze working | Week 2 |
| Silver (dedup + PII split) working | Week 3 |
| Gold + dashboard connected | Week 4 |
| Governance grants + lineage verified | Week 4 |

## 10. Next Steps

1. Confirm data access for Core Banking DB `accounts` table
2. Provision minimal infra (see `infra/terraform`)
3. Build Glue extract job for `accounts`
4. Build Airflow DAG skeleton
5. Build Bronze notebook, then Silver, then Gold
