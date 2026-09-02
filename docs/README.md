# Banking Data Lakehouse

A data platform that ingests banking source systems (Core Banking DB, Card
Processor, CRM, Transaction Files/APIs), processes them through a medallion
architecture (Bronze/Silver/Gold) on Databricks, and serves regulatory
reporting, operational analytics, and AI/ML use cases — with governance
(access control, PII masking, lineage, audit) applied throughout.

See [`docs/design-doc.md`](docs/design-doc.md) for the full Phase 1 scope,
goals, and architecture decisions.

## Status

🚧 Phase 1 in progress — single-source (Accounts) pipeline, Bronze → Silver →
Gold, one dashboard. See the design doc for what's in/out of scope.

## Architecture

```
Banking Sources → Ingestion (Glue + Airflow) → S3 Raw Zone
    → Databricks Lakehouse (Bronze → Silver → Gold)
    → Consumption (Regulatory Reporting / Operational Analytics / AI-ML)

Governance (Unity Catalog, IAM/RBAC, PII masking, lineage, audit, data
quality) applies across every layer.
```

Full architecture diagram: `docs/architecture.md` (add diagram here).

## Repo Structure

```
banking-lakehouse/
├── README.md
├── .gitignore
├── infra/                      # Terraform for AWS + Databricks
│   └── terraform/
│       ├── s3.tf
│       ├── iam.tf
│       ├── glue.tf
│       └── databricks.tf
├── ingestion/                  # Glue extraction jobs
│   ├── glue_jobs/
│   └── config/
│       └── sources.yaml
├── orchestration/               # Airflow DAGs
│   └── dags/
├── transformations/             # Databricks notebooks/jobs
│   ├── bronze/
│   ├── silver/
│   └── gold/
├── governance/
│   ├── unity_catalog/
│   └── data_quality/
├── tests/
│   ├── unit/
│   └── integration/
├── docs/
│   ├── design-doc.md
│   ├── architecture.md
│   └── data_dictionary.md
└── requirements.txt
```

## Prerequisites

- AWS account with access to S3, Glue, IAM
- Databricks workspace with Unity Catalog enabled
- Python 3.10+
- Terraform 1.x (for infra provisioning)
- Access credentials for source systems (see `docs/design-doc.md` §5)

## Setup

```bash
# Clone the repo
git clone <repo-url>
cd banking-lakehouse

# Install Python dependencies
pip install -r requirements.txt

# Copy env template and fill in real values (never commit the real file)
cp .env.example .env

# Provision infra (review the plan before applying)
cd infra/terraform
terraform init
terraform plan
terraform apply
```

## Running the pipeline locally

1. Configure source connection details in `ingestion/config/sources.yaml`
2. Deploy the Airflow DAG in `orchestration/dags/` to your Airflow instance
3. Trigger the DAG manually for a first test run, or wait for the nightly schedule
4. Check table status in Databricks:
   ```sql
   SELECT * FROM bronze.accounts LIMIT 10;
   SELECT * FROM silver.account_master LIMIT 10;
   SELECT * FROM gold.daily_balances LIMIT 10;
   ```

## Testing

```bash
# Unit tests
pytest tests/unit

# Integration tests (requires a configured dev environment)
pytest tests/integration
```

## Branching strategy

- `main` — always working, deployable
- `dev` — integration branch
- `feature/<short-description>` — one branch per unit of work, merged into `dev` via PR

## Governance notes

- All tables are registered in Unity Catalog; see `governance/unity_catalog/grants.sql`
- PII fields live in separate `*_pii` tables with restricted access; masked
  views are provided for general query access
- Data quality rules live in `governance/data_quality/`

## Roadmap

- [x] Phase 0: Design doc, repo structure, infra plan
- [ ] Phase 1: Single-source pipeline (Accounts) — Bronze/Silver/Gold + one dashboard
- [ ] Phase 2: Additional sources (Cards, CRM, Transactions)
- [ ] Phase 3: Regulatory reporting (AnaCredit/BaFin)
- [ ] Phase 4: AI/ML models (credit risk, churn, fraud)

## Contact

[Your name / team] — [contact info]
