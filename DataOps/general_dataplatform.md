# Medallion vs Data Warehouse vs Data Mesh

A side-by-side comparison of three foundational data platform architectures: **Medallion Architecture**, **Data Warehouse**, and **Data Mesh**. Each solves the same problem — turning raw operational data into business-ready insights — but they make very different choices about *where* data lives, *who* owns it, and *how* it flows.

---

## 1. Medallion Architecture

Data is organized into progressive layers (**Bronze → Silver → Gold**) to ensure data quality, structure, and preparation for analytics & machine learning.

```mermaid
flowchart LR
    subgraph Src["Source"]
        Batch[Batch]
        Stream[Stream]
    end

    Bronze[("Bronze<br/>Raw Ingestion<br/>(Immutable)")]
    Silver[("Silver<br/>Filtered, Cleaned,<br/>Structured")]
    Gold[("Gold<br/>Business-Level<br/>Aggregated Data")]

    subgraph Cons["Consumption"]
        AIML[AI / ML / RAG]
        BI[BI & Analytics]
        SelfSvc[Self-Service Data]
    end

    Batch --> Bronze
    Stream --> Bronze
    Bronze --> Silver
    Silver --> Gold
    Gold --> AIML
    Gold --> BI
    Gold --> SelfSvc
```

**How it works:** Raw events from batch and streaming sources land in **Bronze** as immutable history. **Silver** cleans, deduplicates, and conforms the data. **Gold** aggregates it into analytics-ready, business-level tables that power AI/ML, BI dashboards, and self-service consumption.

---

## 2. Data Warehouse

Centralized, optimized storage for integrated, clean, and historical data — ready for reporting and advanced analytics.

```mermaid
flowchart LR
    subgraph Src["Source"]
        CRM[CRM / ERP]
        Apps[Apps / Website]
        Files[Files / Emails]
    end

    Staging[(Staging Area)]
    DW[(Data Warehouse)]

    subgraph Marts["Data Marts"]
        DM1[(Mart 1)]
        DM2[(Mart 2)]
        DM3[(Mart 3)]
    end

    subgraph Cons["Consumption"]
        AIML[AI — ML Models]
        BI[BI & Analytics]
        SelfSvc[Self-Service Data]
    end

    CRM -- "ETL / ELT" --> Staging
    Apps -- "ETL / ELT" --> Staging
    Files -- "ETL / ELT" --> Staging
    Staging -- "ETL" --> DW
    DW --> DM1
    DW --> DM2
    DW --> DM3
    DM1 --> AIML
    DM2 --> BI
    DM3 --> SelfSvc
```

**How it works:** Source systems (CRM/ERP, web apps, files) are pulled by **ETL/ELT** into a **Staging Area**, then transformed into the central **Data Warehouse**. Subject-specific **Data Marts** carve the warehouse into focused slices for AI/ML, BI, and ad-hoc analytics.

---

## 3. Data Mesh

Decentralized ownership by **domain teams**. Each domain publishes trusted **data products** with interoperability and federated governance via shared standards.

```mermaid
flowchart LR
    subgraph Src["Source"]
        CRM[CRM / ERP]
        Apps[Apps / Website]
        Files[Files / Emails]
    end

    subgraph Domains["Data Products & Lakes"]
        D1[Domain 1<br/>DataLake]
        D2[Domain 2<br/>DataLake]
        D3[Domain 3<br/>DataLake]
        P1[Data Product 1]
        P2[Data Product 2]
        P3[Data Product 3]
        D1 --> P1
        D2 --> P2
        D3 --> P3
    end

    Catalog["Mesh Catalogs & Data Contracts<br/>(Schema · Security · SLA)"]

    subgraph Consumers["Data Consumers<br/>(Distributed Teams)"]
        U1[Team A]
        U2[Team B]
        U3[Team C]
    end

    CRM -- "ETL" --> D1
    Apps -- "ETL" --> D2
    Files -- "ETL" --> D3
    P1 --> Catalog
    P2 --> Catalog
    P3 --> Catalog
    Catalog --> U1
    Catalog --> U2
    Catalog --> U3
```

**How it works:** Each business domain (Sales, Marketing, Finance, etc.) owns its own data lake and publishes **data products** as a first-class deliverable. A central **Mesh Catalog** registers every product alongside its data contract — schema, security policy, SLA — and **distributed consumer teams** discover and subscribe to products through that catalog.

---

## 4. Side-by-Side Comparison

| Point | Medallion Architecture | Data Warehouse | Data Mesh |
|---|---|---|---|
| **Core Idea** | Organizes data into layers: Bronze, Silver, and Gold. | Centralizes data into one structured warehouse. | Decentralizes data ownership across business domains. |
| **Data Flow** | Raw data moves from Bronze → Silver → Gold. | Source data goes through ETL/ELT → Staging → Warehouse → Data Marts. | Each domain creates and manages its own data products. |
| **Data Ownership** | Usually managed by a central data engineering team. | Mostly controlled by central BI / data warehouse teams. | Owned by domain teams like sales, finance, product, or operations. |
| **Best Use Case** | Lakehouse, analytics, ML, and AI-ready data pipelines. | Reporting, dashboards, historical analysis, and business intelligence. | Large organizations where teams need independent data ownership. |
| **Governance Style** | Layered data quality improvement. | Centralized through warehouse rules and data models. | Federated using data contracts, catalogs, quality rules, and SLAs. |

---

## 5. How to Choose

```mermaid
flowchart TB
    Start{What is your<br/>primary need?}
    Q1{Mostly BI dashboards<br/>and historical reporting?}
    Q2{Mix of ML, streaming,<br/>and structured analytics?}
    Q3{Many independent teams,<br/>each with their own data?}

    DW[Data Warehouse<br/>Snowflake / Redshift / BigQuery]
    Med[Medallion Architecture<br/>Lakehouse — Delta / Iceberg]
    Mesh[Data Mesh<br/>Federated Domain Ownership]

    Start --> Q1
    Q1 -- "Yes" --> DW
    Q1 -- "No" --> Q2
    Q2 -- "Yes" --> Med
    Q2 -- "No" --> Q3
    Q3 -- "Yes" --> Mesh
    Q3 -- "No" --> Med
```

### Rules of Thumb

- **Pick a Data Warehouse** when the work is overwhelmingly structured reporting and BI, governance is centralized, and the data volume fits comfortably in a managed warehouse engine.
- **Pick Medallion / Lakehouse** when workloads span ML, streaming, and analytics, when data is semi-structured or large-scale, and when one central data engineering team can own the pipeline.
- **Pick Data Mesh** when the organization is too large for a single team to be the bottleneck — each business domain becomes a data publisher and consumer through shared contracts and a central catalog.

---

## 6. They Are Not Mutually Exclusive

In practice, mature platforms combine all three: **Medallion is the storage pattern**, **Data Warehouse is the serving layer**, and **Data Mesh is the organizational model** layered on top.

```mermaid
flowchart LR
    subgraph SalesDomain["Sales Domain"]
        SB[Bronze] --> SS[Silver] --> SG[Gold Data Product]
    end
    subgraph MktgDomain["Marketing Domain"]
        MB[Bronze] --> MS[Silver] --> MG[Gold Data Product]
    end
    subgraph FinDomain["Finance Domain"]
        FB[Bronze] --> FS[Silver] --> FG[Gold Data Product]
    end

    MeshCatalog[Federated Mesh Catalog<br/>+ Data Contracts]
    Warehouse[(Enterprise Warehouse<br/>for cross-domain BI)]

    SG --> MeshCatalog
    MG --> MeshCatalog
    FG --> MeshCatalog
    MeshCatalog --> Warehouse
```

Each domain internally uses **Medallion** (Bronze/Silver/Gold), publishes its Gold layer as a **Data Mesh** product, and the federated catalog feeds a thin **Data Warehouse** layer for cross-domain reporting. This is exactly the pattern the EKS-based platform in `datamesh_eks.md` implements.
