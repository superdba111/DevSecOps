# Architecture and Sizing

## 1. Understand Aurora's architecture

Aurora separates **compute** from its distributed **cluster storage**.

```text
                  Application
                      |
          +-----------+-----------+
          |                       |
     Writer endpoint          Reader endpoint
          |                       |
      +--------+              +--------+
      | Writer |              | Reader |
      +--------+              +--------+
          \                     /
           \                   /
         Distributed Aurora storage
         across multiple AZs
```

Aurora synchronously replicates writes across multiple storage nodes in multiple AZs. A cluster can have up to 15 Aurora Replicas.

### Best practices

- Place the writer and at least one reader in **different AZs**.
- Use the **cluster/writer endpoint** rather than an instance endpoint for normal writes.
- Use the **reader endpoint** for appropriate read-only traffic.
- Set **promotion tiers** deliberately so the preferred failover reader is appropriately sized.
- Do not confuse:
  - storage durability,
  - reader failover,
  - backups/PITR,
  - and cross-Region DR.

They solve different problems.

## 2. Provisioned vs Serverless

### Provisioned

Use when you want predictable baseline CPU/memory and stable heavy workloads.

```text
Writer: db.r7g.4xlarge
Reader: db.r7g.2xlarge
Reader: db.r7g.2xlarge
```

### Aurora Serverless v2

Use when workloads are highly variable or when avoiding overprovisioning matters.

```text
db.serverless
min ACU -------------------- max ACU
           automatic scaling
```

### Practical selection rule

| Workload | Starting preference |
|---|---|
| Stable, high-volume, latency-sensitive OLTP | Provisioned |
| Variable/spiky traffic | Serverless v2 |
| Dev/test | Serverless v2 |
| Mixed predictable writer + bursty reads | Consider provisioned writer + serverless reader(s) |
| Unknown new workload | Serverless v2 can simplify initial sizing |

Do not choose by fashion. Measure database load, memory, connections, latency, traffic variability, and cost.

## 3. Parameter groups

Aurora has two layers:

```text
Aurora Cluster
   |
   +-- DB Cluster Parameter Group
   |      cluster-wide/default settings
   |
   +-- Writer
   |      DB Parameter Group
   |
   +-- Reader
          DB Parameter Group
```

### Best practices

- Create **custom version-controlled parameter groups** instead of editing defaults ad hoc.
- Keep a cluster-level baseline for:
  - logging,
  - autovacuum,
  - timeouts,
  - logical replication,
  - plan management,
  - shared preload libraries.
- Use instance-specific overrides only when writer and reader workload characteristics justify them.
- Know which parameters are **dynamic** and which require a **reboot**.
- Never call a value “optimized” until validated against the actual workload.

## 4. Capacity guardrails

Monitor before saturation:

- CPU utilization
- freeable memory
- database connections
- database load / waits
- read/write latency
- replica lag
- temp-file activity
- storage/WAL/replication-slot usage
- network throughput

Scale **before** sustained resource exhaustion causes latency, restart, or failover.


## 5. Multi-account and environment design

DrFirst explicitly calls out multiple accounts and environments.

Preferred pattern:

```text
Shared Terraform module
        |
   +----+-----+------+
   |          |      |
 DEV        STAGE   PROD
 account     account  account
```

Keep common controls standardized:

- Aurora engine/version policy
- cluster and DB parameter groups
- subnet groups
- encryption/KMS
- security groups
- monitoring and alarms
- backup retention
- RDS Proxy
- tags
- IAM roles/policies

Keep environment differences as inputs, not copied infrastructure.

```text
DEV:  smaller capacity, relaxed retention
STAGE: production-like validation
PROD: HA, stronger approvals, full monitoring/DR
```

Use cross-account IAM roles and CI/CD rather than shared long-lived credentials.

## 6. Provisioned / Serverless conversion pattern

Aurora can mix provisioned and Serverless v2 instances where supported.

A safer production migration pattern is often:

```text
Provisioned writer
       |
add/convert Serverless reader
       |
test scaling + latency
       |
controlled failover
       |
Serverless reader becomes writer
```

This reduces risk compared with experimenting first on the active writer.

For the most latency-sensitive OLTP workload, favor a measured decision:
- predictable heavy baseline -> provisioned is often simpler
- variable/spiky load -> Serverless v2 can reduce overprovisioning
