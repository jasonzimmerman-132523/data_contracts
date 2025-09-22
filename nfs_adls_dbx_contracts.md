# Data Contract Workflow (NFS → ADLS Gen2 → Databricks)

```mermaid
flowchart LR
  %% === On-Premises ===
  subgraph ONPREM["On-Premises"]
    NFS["NFS File Share"]
    EDGE["Edge Sync VM\n(Data Box Gateway / AzCopy)"]
  end

  %% === Azure Storage & Services ===
  subgraph AZURE["Azure Cloud"]
    ADLS["ADLS Gen2 Storage Account"]
    IN["Landing zone (/incoming)"]
    VAL["Validated zone (/validated)"]
    REJ["Rejected zone (/rejected)"]
    EVG["Event Grid (blob created)"]
    TRG["Trigger (Function / Logic App)\nstarts Databricks Job"]
    KV["Key Vault (secrets)"]
    PUR["Data Catalog (Purview)"]
    LOG["Monitoring (Log Analytics)"]
  end

  %% === Databricks ===
  subgraph DATABRICKS["Databricks"]
    REPO["Repo (contract.yaml)"]
    JOB["Databricks Job"]
    NB["Validation Notebook"]
    DLT["Delta Live Tables (optional)"]
    DELTA["Delta Lake (/validated dataset)"]
  end

  %% === Governance ===
  subgraph CICD["CI/CD & Governance"]
    GH["GitHub Actions"]
    PR["Pull Requests"]
    RBAC["AAD / RBAC"]
  end

  %% === Consumers ===
  subgraph CONSUMERS["Downstream Consumers"]
    BI["BI / Reporting (Power BI)"]
    ML["ML / Training Data"]
  end

  %% === Connections ===
  NFS --> EDGE --> IN
  IN --> ADLS
  IN --> EVG --> TRG --> JOB

  REPO --> NB
  JOB --> NB
  NB -->|pass| VAL
  NB -->|fail| REJ
  NB --> DELTA
  JOB --> DLT --> DELTA
  NB --> KV

  GH --> REPO
  PR --> REPO
  RBAC --> DATABRICKS

  DELTA --> PUR
  NB --> LOG
  DELTA --> LOG

  DELTA --> BI
  DELTA --> ML

