# Clar Data Platform — TO-BE architecture diagrams (Mermaid)


---

## 1. Target platform — Datastream committed

Every component needed for a working platform, not only the data path.
Solid lines carry data. Dotted lines are services that act on the platform.

```mermaid
flowchart LR
  subgraph SRC["SOURCE SYSTEMS"]
    direction TB
    MY["MySQL self-hosted<br/>binlog ROW format<br/>replication user"]
    PG["PostgreSQL on AWS<br/>wal_level = logical<br/>publication + slot"]
    TU["TUNE API"]
    GS["Google Sheets"]
    RS["AWS Redshift<br/>history, one time only"]
  end

  subgraph ING["INGESTION"]
    direction TB
    PSC["Private Service Connect<br/>or IP allowlist + SSL"]
    DS["Datastream<br/>two streams<br/>explicit table allowlist<br/>max staleness 15 min"]
    CR["Cloud Run job + Cloud Scheduler<br/>TUNE REST pull, hourly"]
    EXT["External table over Drive<br/>+ scheduled query snapshot"]
    DTS["BigQuery Data Transfer Service<br/>Redshift migration"]
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

---

## 2. Ingestion detail — prerequisites per source

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

---

## 4. Migration and decommission

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
