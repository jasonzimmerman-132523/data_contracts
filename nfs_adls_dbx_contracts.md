# Data Contract Workflow (NFS → ADLS Gen2 → Databricks)

```mermaid
flowchart LR
  %% === On-Premises ===
  subgraph ONPREM["On-Premises"]
    direction TB
    NFS["NFS File Share\n(e.g. /exports/data)"]
    EDGE["Edge Sync VM\n(Data Box Gateway / Azure File Sync / AzCopy)"]
  end

  %% === Azure Storage & Services ===
  subgraph AZURE["Azure Cloud"]
    direction TB
    ADLS["ADLS Gen2 Storage Account\n(container: <dataset>)"]
    IN["Landing zone\n/container/incoming/"]
    VAL["Validated zone\n/container/validated/"]
    REJ["Quarantine / Rejected\n/container/rejected/"]
    EVG["Event Grid\n(blob created)"]
    TRG["Trigger\n(Azure Function / Logic App)\nstarts Databricks Job"]
    KV["Key Vault\n(secrets: SP creds, tokens)"]
    PUR["Data Catalog\n(Microsoft Purview)"]
    LOG["Monitoring\n(Log Analytics / Monitor)"]
  end

  %% === Databricks ===
  subgraph DATABRICKS["Databricks Workspace / Control Plane"]
    direction TB
    REPO["Repo (contract.yaml)\n(GitHub / Azure Repos)"]
    JOB["Databricks Job\n(job cluster or serverless)"]
    NB["Validation Notebook\n(Great Expectations / Pandera)"]
    DLT["Delta Live Tables\n(optional)"]
    DELTA["Delta Lake\n(/mnt/adls/validated/<dataset>)"]
  end

  %% === CI/CD & Governance ===
  subgraph CICD["CI/CD & Governance"]
    direction TB
    GH["GitHub Actions\n(contract lint + tests)"]
    PR["Pull Requests & Reviews"]
    RBAC["AAD / RBAC\n(service principals & groups)"]
  end

  %% === Consumers ===
  subgraph CONSUMERS["Downstream Consumers"]
    direction TB
    BI["BI / Reporting\n(Power BI, Notebooks)"]
    ML["ML Training / Features\n(model training)"]
  end

  %% === Connections ===
  NFS --> EDGE
  EDGE -->|azcopy / synctool| IN

  IN --> ADLS
  ADLS --> IN
  ADLS --> VAL
  ADLS --> REJ

  IN --> EVG
  EVG --> TRG
  TRG --> JOB

  %% Databricks execution and contract
  REPO --> NB
  JOB --> NB
  NB -->|uses secrets| KV
  NB -->|validate: PASS| VAL
  NB -->|validate: FAIL| REJ
  NB --> DELTA
  DLT --> DELTA
  JOB --> DLT

  %% Governance & CI/CD
  GH --> REPO
  PR --> REPO
  REPO -.->|deploy contracts / notebooks| JOB
  RBAC --> DATABRICKS

  %% Observability & Catalog
  DELTA --> PUR
  NB --> LOG
  DELTA --> LOG

  %% Consumers
  DELTA --> BI
  DELTA --> ML

  %% Optional / informative links (dashed)
  EDGE -.->|optional direct mount (not recommended)| DATABRICKS
  TRG -.-> NB

  %% === LEGEND ===
  subgraph LEGEND["Legend"]
    direction TB
    L1["📂 Storage Zones:\nLanding, Validated, Rejected"]
    L2["⚙️ Processing:\nDatabricks Jobs, Notebooks, DLT"]
    L3["📜 Contracts:\nYAML (schema + rules) in Git Repo"]
    L4["🔑 Security:\nAAD RBAC, Key Vault secrets"]
    L5["📡 Integration:\nEvent Grid + Triggers"]
    L6["📊 Consumers:\nBI, ML, Data Catalog"]
  end
