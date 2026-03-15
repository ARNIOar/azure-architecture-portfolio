# Architecture: Secure Data Pipeline

> **Domain:** Banking — Data & Analytics / Compliance
> **Pattern:** Medallion architecture (Bronze/Silver/Gold), ETL/ELT
> **Azure services:** Data Factory, Data Lake Storage Gen2, Synapse Analytics, Purview, Key Vault, Private Endpoints

---

## Business Context

A bank needs a centralized data platform that:

- Ingests data from multiple source systems (core banking, CRM, payments, external feeds)
- Transforms and cleanses data for analytics and regulatory reporting
- Enforces data governance (classification, lineage, access control)
- Meets strict security requirements (encryption, network isolation, auditability)
- Enables self-service analytics for business users while protecting sensitive data

---

## Architecture Diagram

```mermaid
graph LR
    subgraph "Source Systems"
        CBS[Core Banking<br/>SQL Server On-Prem]
        CRM[CRM System<br/>Dynamics 365]
        PAY[Payment Gateway<br/>REST API]
        EXT[External Data<br/>Credit Bureau, Market Data]
    end

    subgraph "Integration Layer"
        SHIR[Self-Hosted<br/>Integration Runtime]
        ADF[Azure Data Factory<br/>Orchestration & ETL]
    end

    subgraph "Bronze Layer — Raw"
        DL_BRONZE[Data Lake Gen2<br/>bronze/ container<br/>Raw data, original format]
    end

    subgraph "Silver Layer — Cleansed"
        SYNAPSE_SPARK[Synapse Spark Pool<br/>Data Cleansing &<br/>Transformation]
        DL_SILVER[Data Lake Gen2<br/>silver/ container<br/>Cleansed, typed, deduplicated]
    end

    subgraph "Gold Layer — Business"
        SYNAPSE_SQL[Synapse Dedicated<br/>SQL Pool<br/>Star Schema / Data Warehouse]
        DL_GOLD[Data Lake Gen2<br/>gold/ container<br/>Aggregated, business-ready]
    end

    subgraph "Consumption"
        PBI[Power BI<br/>Dashboards & Reports]
        ADHOC[Synapse Serverless<br/>SQL Pool<br/>Ad-hoc Queries]
        API_OUT[API Layer<br/>Data Products]
    end

    subgraph "Governance"
        PURVIEW[Microsoft Purview<br/>Catalog, Classification<br/>Lineage, Glossary]
    end

    subgraph "Security"
        KV[Key Vault]
        PE[Private Endpoints<br/>All Services]
        ENTRA[Entra ID<br/>RBAC & Access Control]
    end

    CBS --> SHIR
    CRM --> ADF
    PAY --> ADF
    EXT --> ADF
    SHIR --> ADF

    ADF --> DL_BRONZE
    DL_BRONZE --> SYNAPSE_SPARK
    SYNAPSE_SPARK --> DL_SILVER
    DL_SILVER --> SYNAPSE_SQL
    DL_SILVER --> DL_GOLD
    SYNAPSE_SQL --> DL_GOLD

    DL_GOLD --> PBI
    DL_SILVER --> ADHOC
    DL_GOLD --> API_OUT

    DL_BRONZE -.-> PURVIEW
    DL_SILVER -.-> PURVIEW
    DL_GOLD -.-> PURVIEW
    SYNAPSE_SQL -.-> PURVIEW

    ADF -.-> KV
    SYNAPSE_SPARK -.-> KV
```

## Medallion Architecture

| Layer | Container | Purpose | Format | Retention |
|-------|-----------|---------|--------|-----------|
| **Bronze** | `bronze/` | Raw data exactly as received from source | JSON, CSV, Parquet, XML | 2 years |
| **Silver** | `silver/` | Cleansed, typed, deduplicated, standardized | Delta / Parquet | 5 years |
| **Gold** | `gold/` | Business-ready aggregations, KPIs, report-ready | Delta / Parquet | 7 years |

### Bronze → Silver Transformations
- Schema validation and type casting
- Null handling and default values
- Deduplication based on business keys
- PII masking for non-privileged consumers
- Standardize date formats, currencies, codes

### Silver → Gold Transformations
- Business logic aggregations (daily balances, monthly totals)
- Star schema dimensional modeling
- KPI calculations (NPL ratio, loan-to-value, risk scores)
- Regulatory report datasets (COREP, FINREP, local reporting)

---

## Data Factory Pipeline Design

```
Master Pipeline (scheduled daily at 02:00 UTC)
├── 1. Ingest Pipeline
│   ├── CBS → Bronze (via Self-Hosted IR, incremental CDC)
│   ├── CRM → Bronze (via REST connector, delta query)
│   ├── Payments → Bronze (via REST connector, date filter)
│   └── External → Bronze (via HTTP connector)
│
├── 2. Transform Pipeline
│   ├── Bronze → Silver (Synapse Spark notebook)
│   ├── Data quality checks (row counts, null checks, schema validation)
│   └── PII classification scan (Purview)
│
├── 3. Load Pipeline
│   ├── Silver → Gold (Synapse SQL stored procedures)
│   ├── Gold → Synapse DW (CTAS / MERGE operations)
│   └── Refresh Power BI datasets
│
└── 4. Governance Pipeline
    ├── Update Purview lineage
    ├── Run data quality report
    └── Send pipeline status notification
```

---

## Security Design

### Network Isolation

```mermaid
graph TB
    subgraph "Hub VNet"
        FW[Azure Firewall]
    end

    subgraph "Data Platform VNet (10.10.0.0/16)"
        subgraph "snet-data-integration (10.10.1.0/24)"
            ADF_IR[Data Factory<br/>Managed VNet IR]
            SHIR_VM[Self-Hosted IR VM]
        end
        subgraph "snet-data-storage (10.10.2.0/24)"
            PE_DL[Private Endpoint<br/>Data Lake]
            PE_SQL[Private Endpoint<br/>Synapse SQL]
        end
        subgraph "snet-analytics (10.10.3.0/24)"
            SYNAPSE[Synapse Workspace]
            PBI_GW[Power BI Gateway]
        end
    end

    FW <--> ADF_IR
    FW <--> SHIR_VM
    ADF_IR --> PE_DL
    ADF_IR --> PE_SQL
    SYNAPSE --> PE_DL
    SYNAPSE --> PE_SQL
    PBI_GW --> PE_SQL
```

### Access Control Matrix

| Role | Bronze | Silver | Gold | Synapse DW |
|------|--------|--------|------|------------|
| Data Engineer | Read/Write | Read/Write | Read | Full Access |
| Data Analyst | No Access | Read (masked PII) | Read | Read |
| Business User | No Access | No Access | No Access | Read (via Power BI) |
| Compliance Officer | Read | Read | Read | Read |
| Data Factory MI | Read/Write | Read/Write | Read/Write | Write |

### Data Protection

- **Encryption at rest:** Customer-managed keys (CMK) via Key Vault for Data Lake and Synapse
- **Encryption in transit:** TLS 1.2 enforced on all connections
- **PII masking:** Dynamic data masking in Synapse SQL; column-level security for sensitive fields
- **Row-level security:** Users see only their department's data in Power BI
- **Audit logging:** All data access logged to Log Analytics; Data Factory pipeline runs tracked

---

## Data Governance with Purview

| Capability | Implementation |
|------------|---------------|
| **Data Catalog** | Automatic scanning of Data Lake, Synapse, and SQL sources |
| **Classification** | Built-in classifiers for PII (names, IDs, credit card numbers) |
| **Lineage** | Auto-captured from Data Factory pipelines and Synapse notebooks |
| **Glossary** | Business terms defined (e.g., "Net Loan Balance", "Non-Performing Loan") |
| **Access Policies** | Purview controls who can read/deny specific data assets |

---

## Cost Optimization

| Strategy | Savings |
|----------|---------|
| Synapse serverless for ad-hoc (vs. always-on dedicated pool) | ~70% for exploratory queries |
| Data Lake lifecycle management (Hot → Cool → Archive) | ~60% on storage after 1 year |
| ADF pipeline scheduling (off-peak hours, batch) | Reduced integration runtime costs |
| Synapse auto-pause on dedicated pool (dev/test) | ~80% when not in use |
| Reserved capacity for Synapse (1-year) | ~40% discount |


