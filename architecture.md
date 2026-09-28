# Data platform architecture proposal



## Contents

- [1. Target platform](#1-target-platform)
- [2. Ingestion prerequisites](#2-ingestion-prerequisites)
- [3. Layers inside BigQuery](#3-layers-inside-bigquery)
- [4. Migration and decommission](#4-migration-and-decommission)
- [Connector map](#connector-map)
- [Open decisions](#open-decisions)

---

## 1. Target platform

Every component needed for a working platform, not only the data path.
Solid lines carry data; dotted lines are services that act on the platform.

```mermaid
flowchart LR
  subgraph SRC["SOURCE SYSTEMS - ONGOING"]
    direction TB
    MY["MySQL self-hosted<br/>binlog ROW format<br/>replication user"]
    PG["PostgreSQL on AWS<br/>wal_level = logical<br/>publication + slot"]
    TU["TUNE API"]
    GS["Google Sheets"]
  end

  subgraph LEG["LEGACY - ONE TIME ONLY"]
    RS["AWS Redshift & MySQL <br/>historical data<br/>decommissioned after load"]
  end

  subgraph ING["INGESTION"]
    direction TB
    PSC["Private Service Connect<br/>or IP allowlist + SSL"]
    DS["Datastream<br/>two streams<br/>explicit table allowlist<br/>max staleness 15 min"]
    CR["Cloud Run job + Cloud Scheduler<br/>TUNE REST pull, hourly"]
    EXT["External table over Drive<br/>+ scheduled query snapshot"]
    DTS["BigQuery Data Transfer Service<br/>Redshift migration<br/>runs once, then removed"]
    SM["Secret Manager<br/>DB credentials, API keys"]
  end

  subgraph BQ["BIGQUERY - EU MULTI-REGION"]
    direction TB
    RAW["RAW<br/>raw_mysql, raw_postgres<br/>raw_tune, raw_sheets"]
    STG["STAGING<br/>typed, deduplicated<br/>PII hashed + policy tags"]
    CORE["CORE<br/>conformed entities<br/>crosswalk tables"]
    MART["MARTS<br/>finance, risk, ops, exec"]
    REF["REFERENCE<br/>business mapping tables"]
    RAW --> STG --> CORE --> MART
    REF --> CORE
  end

  subgraph CONS["CONSUMPTION"]
    direction TB
    BIE["BI Engine reservation"]
    LS["Looker Studio"]
    SQL["Ad hoc SQL"]
  end

  MY --> PSC
  PG --> PSC
  PSC --> DS --> RAW
  TU --> CR --> RAW
  GS --> EXT --> RAW
  RS --> DTS --> RAW
  MART --> BIE --> LS
  CORE --> SQL

  DFM["Dataform<br/>models, assertions, scheduling"]
  GIT["GitHub<br/>Dataform + Terraform"]
  DPX["Dataplex<br/>catalog, profiling, quality scans"]
  MON["Cloud Monitoring<br/>stream failure, workflow failure, freshness"]
  AUD["Cloud Audit Logs<br/>sink to clar-admin"]

  SM -.-> DS
  SM -.-> CR
  GIT -.-> DFM
  DFM -.-> STG
  DFM -.-> CORE
  DFM -.-> MART
  DPX -.-> BQ
  MON -.-> DS
  MON -.-> DFM
  AUD -.-> BQ
```

Two AWS-hosted systems appear here and they are not the same job:

- **PostgreSQL on AWS** is a live lending database, replicated continuously by Datastream.
- **AWS Redshift** is the current warehouse. Its history is copied once by the BigQuery Data Transfer Service, then Redshift is decommissioned.

---

## 2. Ingestion prerequisites

What must be true on each source before the connector will run.
These prerequisites, not the connector configuration, are the real work of the ingestion phase.

```mermaid
flowchart TB
  subgraph S1["MySQL, self-hosted"]
    A1["Enable binary logging, ROW format"]
    A2["Create replication user"]
    A3["Set binlog retention to survive an outage"]
    A4["Connectivity: IP allowlist + SSL,<br/>SSH tunnel, or Private Service Connect"]
    A1 --> A2 --> A3 --> A4
  end
  A4 --> D1["Datastream stream<br/>Explicit table allowlist<br/>Max staleness 15 min"]
  D1 --> R1["raw_mysql<br/>one table per source table<br/>schema drift handled"]

  subgraph S2["PostgreSQL on AWS"]
    B1["Set wal_level = logical"]
    B2["Create publication and replication slot"]
    B3["Monitor slot lag - an unread slot fills the disk"]
    B4["Connectivity: IP allowlist + SSL<br/>or Private Service Connect"]
    B1 --> B2 --> B3 --> B4
  end
  B4 --> D2["Datastream stream<br/>Production tables only<br/>not remarketing or analytical"]
  D2 --> R2["raw_postgres"]

  subgraph S3["TUNE API"]
    C1["API key in Secret Manager"]
    C2["Cloud Run job, incremental by date, idempotent"]
    C1 --> C2
  end
  C2 --> D3["Cloud Scheduler<br/>hourly"]
  D3 --> R3["raw_tune"]

  subgraph S4["Google Sheets"]
    E1["Share sheet with service account"]
    E2["External table over Drive"]
    E1 --> E2
  end
  E2 --> D4["Scheduled query<br/>daily snapshot with ingest date"]
  D4 --> R4["raw_sheets<br/>native table, versioned history"]
```

---

## 3. Layers inside BigQuery

```mermaid
flowchart TB
  R["RAW<br/>Written only by connectors<br/>No transformations<br/>Partitioned by ingestion date<br/>Kept for the full regulatory period"]
  S["STAGING<br/>Types applied, duplicates removed<br/>Names standardised<br/>National ID hashed<br/>Policy tags on personal data"]
  C["CORE<br/>Conformed customer, partner, product, application<br/>Crosswalk maps source keys to one surrogate key<br/>Country is a column, not a separate table"]
  M["MARTS<br/>One dataset per consuming team<br/>Only layer connected to BI"]
  REF["REFERENCE<br/>Business owned mapping tables<br/>Product taxonomy, status mapping<br/>Version controlled in Git"]
  SBX["SANDBOX<br/>Per analyst<br/>30 day table expiry<br/>Never used by dashboards"]

  R -->|"Dataform"| S
  S -->|"Dataform"| C
  C -->|"Dataform"| M
  REF --> C
  C -.-> SBX

  CHK1["Contract check<br/>schema, freshness, row count"]
  CHK2["Assertions<br/>uniqueness, referential integrity"]
  CHK1 -.-> S
  CHK2 -.-> C
```

Access follows the layers: `raw` is the data team and service accounts only, `core` adds analysts,
`marts` is the only layer BI connects to. Nothing outside the platform reads `raw` or `stg`.

---

## 4. Migration and decommission

Nothing is switched off until the numbers match for a full month, including a month-end close.

```mermaid
flowchart LR
  subgraph NOW["TODAY"]
    N1["n8n custom extract scripts"]
    N2["ClickHouse, local<br/>daily analytics"]
    N3["AWS Redshift<br/>EDW"]
  end

  subgraph MIG["MIGRATION"]
    M1["BigQuery Data Transfer Service<br/>Redshift migration connector<br/>one time historical load"]
    M2["Datastream streams live<br/>in parallel with n8n"]
    M3["Rebuild ClickHouse queries<br/>as Dataform models"]
    M4["Parallel run and reconcile<br/>numbers match for one full month"]
  end

  subgraph TARGET["TARGET"]
    T1["BigQuery + Dataform"]
    T2["Looker Studio + BI Engine"]
  end

  N3 --> M1 --> T1
  N1 --> M2 --> T1
  N2 --> M3 --> T1
  M1 --> M4
  M2 --> M4
  M3 --> M4
  M4 --> T2
  M4 --> D1["Decommission n8n data jobs"]
  M4 --> D2["Decommission ClickHouse"]
  M4 --> D3["Decommission Redshift"]
```

---

## Connector map

| Source | Tool | Mechanism | Frequency |
|---|---|---|---|
| MySQL, self-hosted | Datastream | Reads the binary log, serverless CDC | Continuous, staleness configurable |
| PostgreSQL on AWS | Datastream | Logical replication slot | Continuous, staleness configurable |
| Google Sheets | BigQuery external table over Drive | Queried in place, snapshotted to a native table | Daily snapshot |
| TUNE API | Cloud Run job + Cloud Scheduler | REST pull, incremental by date | Hourly |
| AWS Redshift | BigQuery Data Transfer Service | Purpose-built migration connector | Once |
| Google Ads *(future)* | BigQuery Data Transfer Service | First-party connector, no orchestration charge | Daily |

No table is ingested by more than one mechanism. Adding a table to a Datastream stream is a
deliberate change, not a wildcard — see the cost note below.

---

## Open decisions

| Decision | Why it is open | How to close it |
|---|---|---|
| Datastream cost | CDC is priced per GiB processed, and a processed byte is 2–5× the source data. Streaming everything would cost several times the warehouse itself. | Measure change volume: `SHOW BINARY LOGS` on MySQL, WAL generation on PostgreSQL, for one week. |
| Datastream vs DTS for the databases | The DTS MySQL and PostgreSQL connectors are priced in slot-hours rather than per GiB, which is an order of magnitude cheaper — but batch does not capture hard deletes. | Evaluate delete handling and watermark requirements against the measured volume. |
| Database connectivity | Datastream must reach a self-hosted MySQL host and an AWS-hosted PostgreSQL. | Choose between IP allowlist with SSL, SSH tunnel, and Private Service Connect. Only the last needs a VPC. |
| dbt vs Dataform | Both work. Dataform needs no infrastructure and brings its own scheduler; dbt has the larger talent pool. | Decide before the first model is written. |

### Tables that should not be replicated

- The 3.5 TB of MySQL log tables — no dashboard reads them, and they would dominate the Datastream bill.
- The PostgreSQL remarketing (0.7 TB) and analytical (0.75 TB) databases — these are already derived
  data. Rebuild them as Dataform models on top of the production tables rather than paying to move them.

---

## Editing these diagrams

The Mermaid blocks above are the source of truth. To preview a change before committing, paste the
block into [mermaid.live](https://mermaid.live). To use the diagrams elsewhere:

- **Confluence** — a draw.io macro will import Mermaid via Arrange → Insert → Advanced → Mermaid.
- **Slides** — export SVG or PNG from mermaid.live.
- **Locally** — `npx @mermaid-js/mermaid-cli -i ARCHITECTURE.md -o out.md` renders every block to images.
