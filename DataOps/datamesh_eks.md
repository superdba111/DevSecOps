# Data Mesh on EKS — The "3+N" Cluster Architecture

A production-grade Data Mesh built on Amazon EKS, designed to ingest **1M+ records/second** while keeping streaming, batch, and governance workloads isolated. The platform team operates **3 core clusters** (Management, Streaming, Batch) and federates compute out to **N domain clusters** (Sales, Marketing, Finance) owned by the business domains themselves.

---

## 1. High-Level Architecture

The platform splits responsibilities between a central **AWS Platform Account** (the "Mesh Backbone") and one or more **Domain AWS Accounts** that own their own Gold data products.

```mermaid
flowchart TB
    subgraph OnPrem["On-Premises Data Sources"]
        LegacyDB[(Legacy Databases)]
        InternalApps[Internal Apps]
    end

    subgraph Networking["AWS Networking"]
        DirectConnect[AWS Direct Connect]
    end

    subgraph Platform["AWS Platform Account — The Mesh Backbone (3 Core Clusters)"]
        subgraph Mgmt["1. Management EKS Cluster (Control & Governance)"]
            ArgoCD[ArgoCD<br/>GitOps Controller]
            Airflow[Apache Airflow<br/>Orchestration]
            OpenMeta[OpenMetadata<br/>Global Catalog & Lineage]
            LakeFormation[AWS Lake Formation]
        end

        subgraph Stream["2. Streaming EKS Cluster (Nitro + NVMe)"]
            KConnect[Kafka Connect<br/>Ingestion]
            Strimzi[Strimzi Kafka Brokers<br/>& Schema Registry]
            Flink[Apache Flink<br/>Real-Time Processing]
        end

        subgraph Batch["3. Batch EKS Cluster (Karpenter + Spot)"]
            Spark[Apache Spark<br/>Iceberg Compaction / ETL]
        end

        S3[(Amazon S3 / Iceberg Tables<br/>Bronze / Silver Layer)]
    end

    subgraph Sales["Sales Domain Account (an 'N' Cluster)"]
        SalesEKS[Sales EKS Cluster<br/>Domain Compute]
        Redshift[(Amazon Redshift<br/>Sales Gold Data Product)]
    end

    subgraph Marketing["Marketing Domain Account (an 'N' Cluster)"]
        MktgEKS[Marketing EKS Cluster<br/>Domain Compute]
        Athena[Amazon Athena<br/>Marketing Consumer]
    end

    LegacyDB --> DirectConnect
    InternalApps --> DirectConnect
    DirectConnect --> KConnect
    KConnect -- "Publishes Events" --> Strimzi
    Strimzi -- "Stateful Sub" --> Flink
    Flink -- "Writes Bronze/Silver" --> S3

    Spark -- "Reads Small Files" --> S3
    Spark -- "Writes Compacted Files" --> S3

    Airflow -- "Schedules Tasks" --> Spark
    OpenMeta -- "Harvests Metadata" --> S3
    OpenMeta -- "Discovers Data" --> Spark
    OpenMeta -- "Discovers Data" --> Flink
    LakeFormation -- "Secures Access" --> S3

    ArgoCD -- "Deploys Helm / CRDs" --> Stream
    ArgoCD -- "Deploys SparkApps" --> Batch
    ArgoCD -- "Deploys Domain Apps" --> SalesEKS
    ArgoCD -- "Deploys Domain Apps" --> MktgEKS

    LakeFormation -- "Grants Access" --> SalesEKS
    LakeFormation -- "Grants Access" --> MktgEKS

    S3 -- "Zero-ETL Query / Domain Streaming" --> SalesEKS
    S3 -- "SQL Query" --> MktgEKS
    SalesEKS -- "Transforms to Gold" --> Redshift
    MktgEKS --> Athena
```

---

## 2. Why a 3-Cluster Backbone?

Running Kafka, Spark, and OpenMetadata in the **same** Kubernetes cluster will cause them to fight for resources, exhaust the VPC IP space, and crash the Kubernetes API server. Isolating workloads by their *failure mode* and *scaling profile* is the core idea.

```mermaid
flowchart LR
    subgraph Brain["1. Management — The Brain"]
        direction TB
        B1[ArgoCD]
        B2[Airflow]
        B3[OpenMetadata]
        B4[Prometheus / Grafana]
    end

    subgraph Heart["2. Streaming — The Heart"]
        direction TB
        H1[Strimzi Kafka]
        H2[Kafka Connect]
        H3[Apache Flink]
        H4[i4i Nitro + NVMe nodes]
    end

    subgraph Muscle["3. Batch — The Muscle"]
        direction TB
        M1[Spark Operator]
        M2[Iceberg Compaction]
        M3[Backfills]
        M4[Karpenter + Spot]
    end

    Brain -. "Triggers Jobs" .-> Muscle
    Brain -. "Harvests Metadata" .-> Heart
    Brain -. "Harvests Metadata" .-> Muscle
    Heart -- "Streams to S3" --> S3[(S3 Iceberg<br/>Bronze / Silver)]
    Muscle -- "Compacts S3" --> S3
```

### Cluster Responsibilities at a Glance

| Cluster | Persona | Workloads | Scaling Profile | Why Isolate |
|---|---|---|---|---|
| **Management** | The Brain | ArgoCD, Airflow, OpenMetadata, Prom/Grafana | Static, modest, long-running | Must never go down — a runaway Spark job cannot be allowed to crash the scheduler or catalog. Platform admins only. |
| **Streaming** | The Heart | Strimzi Kafka, Kafka Connect, Flink | Static nodes, pods run for months | Needs AWS Nitro + NVMe (i4i) for sequential I/O. Kafka brokers are intolerant of network jitter — a noisy neighbor breaks ISR and Flink checkpoints. |
| **Batch** | The Muscle | Spark (via Spark Operator), Iceberg compaction, backfills | Hyper-elastic, 0 → thousands of pods → 0 | Uses EC2 Spot for cost. Heavy etcd pressure when Spark requests 2,000 executor pods — isolation prevents "Control Plane Exhaustion." |

---

## 3. The "N" Domain Clusters (Decentralized Mesh)

Once the platform team has built the 3 core clusters, **ArgoCD spins up dedicated EKS clusters inside each domain's own AWS account**. Each domain owns its compute, its Gold Data Products, and its bill.

```mermaid
flowchart TB
    Platform[AWS Platform Account<br/>Central Silver Layer in S3 Iceberg]

    subgraph SalesAcct["Sales AWS Account"]
        SalesK8s[Sales EKS Cluster]
        SalesGold[(Sales Gold Product<br/>Amazon Redshift)]
    end

    subgraph MktgAcct["Marketing AWS Account"]
        MktgK8s[Marketing EKS Cluster]
        MktgGold[(Marketing Gold Product<br/>S3 + Athena)]
    end

    subgraph FinAcct["Finance AWS Account"]
        FinK8s[Finance EKS Cluster]
        FinGold[(Finance Gold Product<br/>Redshift)]
    end

    Platform -- "Cross-Account Share (Lake Formation)" --> SalesK8s
    Platform -- "Cross-Account Share (Lake Formation)" --> MktgK8s
    Platform -- "Cross-Account Share (Lake Formation)" --> FinK8s

    SalesK8s -- "Domain ETL (Silver → Gold)" --> SalesGold
    MktgK8s -- "Domain ETL (Silver → Gold)" --> MktgGold
    FinK8s -- "Domain ETL (Silver → Gold)" --> FinGold
```

This gives each domain freedom to write their own Flink and Spark code, query OpenMetadata, and — crucially — **pay for their own compute**, completely isolating their mistakes from the central pipeline.

---

## 4. The Data Pipeline (Bronze → Silver → Gold)

A common misconception is that the Batch cluster creates the Gold layer. **It does not.** Responsibilities are split between the central Platform team and the domain teams.

```mermaid
flowchart TB
    Sources[Data Sources<br/>Legacy DBs / Internal Apps]
    StreamCluster[Streaming EKS Cluster<br/>Apache Flink]
    S3Iceberg[(Central S3 / Iceberg Tables<br/>Bronze + Silver Layer)]
    BatchCluster[Batch EKS Cluster<br/>Apache Spark<br/>Maintenance / Compaction Only]
    DomainAccts[Domain AWS Accounts<br/>Sales / Marketing / Finance]
    GoldProducts[(Amazon Redshift / Athena<br/>Gold Data Products)]

    Sources -- "Real-time Ingestion" --> StreamCluster
    StreamCluster --> S3Iceberg
    BatchCluster <-- "Maintenance / Compaction" --> S3Iceberg
    S3Iceberg -- "Cross-Account Sharing" --> DomainAccts
    DomainAccts -- "Domain Compute<br/>transforms Silver → Gold" --> GoldProducts
```

### Step-by-Step

1. **Streaming EKS Cluster (Flink):** Consumes raw events from Kafka and writes them directly into the central **S3 Iceberg Tables** as Bronze/Silver.
2. **Batch EKS Cluster (Spark):** Behaves like a *janitor*, not a transformer. It sees all the tiny files Flink is generating, **compacts them** into larger files, and exits. It does **not** touch or create Gold data.
3. **Domain AWS Accounts (Sales, Marketing, Finance):** This is where Gold is made. Each domain pulls clean Silver data, applies its own business logic, and materializes its **Gold Data Products** in its own Redshift cluster or S3 + Athena environment.

---

## 5. Metadata Flow (How Governance Actually Works)

OpenMetadata runs on the **Management** cluster — but the Batch cluster is **not** the one pushing metadata to it. Instead, **AWS Glue is the central engine** and OpenMetadata is the viewer.

```mermaid
flowchart TB
    Producers[Flink / Spark / S3<br/>Producers & Writers]
    Glue[(AWS Glue Data Catalog)]
    OpenMeta[OpenMetadata<br/>on Management EKS]
    UI[End-User Web UI Dashboard<br/>Search · Lineage · Quality]

    Producers -- "Automatic Schema Registration" --> Glue
    OpenMeta -- "Hourly / Scheduled Crawl" --> Glue
    OpenMeta --> UI
```

### How Metadata Actually Moves

1. **Tables talk to Glue.** When Flink writes data or when the Batch Spark cluster compacts files, they register schemas and partition updates directly to **AWS Glue Data Catalog**.
2. **OpenMetadata ingests from Glue.** A scheduled connector on the Management cluster pulls schemas, table names, and lineage from Glue and indexes them into its own database.
3. **End users hit OpenMetadata.** They search for data, view lineage, and check data quality through the OpenMetadata web UI.

**Key takeaway:** The Batch EKS cluster does not "manage" metadata. Storage layers update Glue automatically; OpenMetadata aggregates Glue into a single enterprise catalog.

---

## 6. The Summary One-Liner

> *"Our Streaming EKS cluster handles real-time ingestion directly into the Central S3 Silver layer as Iceberg tables. The Batch EKS cluster is dedicated purely to table maintenance — Spark compaction jobs on that Silver layer to solve the small-file problem. The data is then federated out to Domain AWS Accounts, where domain-specific compute transforms Silver into Gold Data Products inside their own Redshift or Athena environments. For governance, all schemas are registered centrally in AWS Glue, which is automatically scraped by OpenMetadata running on our Management Cluster to provide an enterprise-wide data catalog."*

---

## 7. Technology Stack Summary

| Layer | Technology |
|---|---|
| Orchestration / GitOps | ArgoCD, Apache Airflow |
| Streaming | Strimzi Kafka, Kafka Connect, Apache Flink |
| Batch / ETL | Apache Spark (Spark Operator), Karpenter, EC2 Spot |
| Storage | Amazon S3 + Apache Iceberg (Bronze / Silver) |
| Domain Analytics | Amazon Redshift, Amazon Athena |
| Governance | AWS Glue Data Catalog, AWS Lake Formation, OpenMetadata |
| Observability | Prometheus, Grafana |
| Networking | AWS Direct Connect |
| Compute (Streaming) | AWS Nitro instances (i4i with NVMe Instance Store) |
