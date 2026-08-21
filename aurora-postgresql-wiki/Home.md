# Aurora PostgreSQL Best Practices

**Purpose:** Production-focused Aurora PostgreSQL guidance tailored to a high-availability, patient-facing, event-driven AWS environment and the DrFirst Senior PostgreSQL DBA panel.

**Last reviewed:** 2026-08-21

## Interview-required core

Bernard and Nadeem are expected to focus on:

1. **Aurora/RDS operations across multiple AWS accounts and environments**
   - parameter groups and tuning
   - patching
   - minor/major version upgrades
   - Blue/Green and low-downtime migration
2. **Performance at scale**
   - `EXPLAIN (ANALYZE, BUFFERS)`
   - `pg_stat_statements`
   - CloudWatch Database Insights / Performance Insights terminology
   - indexes and query plans
   - connection pooling for OLTP
3. **Backup, recovery, HA, and DR**
   - RTO/RPO
   - PITR
   - Multi-AZ/Aurora failover
   - restore validation
   - quarterly DR drills
   - runbooks
4. **Event-driven persistence**
   - Kafka / Kinesis
   - logical replication / WAL
   - Debezium CDC
   - Redis / ElastiCache
5. **Automation**
   - Python / shell
   - Terraform / CloudFormation
   - database schema CI/CD
6. **AI-assisted operations**
   - Claude Code / Codex / Gemini
   - human verification before production

## Workshop topics worth adding to the core

These topics are highly valuable even if not called out explicitly in the JD:

- Aurora-specific **wait-event analysis**
- MVCC / autovacuum / dead tuples / XID wraparound
- temporary files and `work_mem`
- RDS Proxy multiplexing and connection pinning
- provisioned vs Serverless v2 sizing
- Query Plan Management (`apg_plan_mgmt`)
- reader failover priorities
- IAM DB authentication with workload identities
- replication-slot/WAL retention monitoring
- practical restore and failover validation

## Core production principles

1. **Design for failure.** Run a writer plus at least one Aurora Replica in another AZ and test application failover.
2. **Use the right endpoint.** Writer/cluster endpoint for writes; reader endpoint for scalable read-only traffic.
3. **Protect PostgreSQL from connection storms.** Use application pools and/or RDS Proxy.
4. **Troubleshoot from waits and evidence.** Database load -> wait events -> sessions -> SQL -> plan.
5. **Keep autovacuum healthy.** High-write OLTP often needs table-specific tuning.
6. **Treat temp-file growth as a symptom.** Fix SQL/indexes/data volume before globally increasing `work_mem`.
7. **Control plan regression.** Use QPM for the most critical SQL where plan stability matters.
8. **Upgrade safely.** Blue/Green + application regression testing + rollback criteria.
9. **Prove backups.** Restore them and validate critical business transactions.
10. **Use least privilege.** Dedicated DB identities, workload IAM roles, TLS, and controlled admin access.
11. **Automate repeatable work.** IaC, scripted checks, schema migrations, DR validation.
12. **Treat PostgreSQL as part of an event ecosystem.** Kafka/Kinesis -> consumer -> Aurora -> CDC/cache/search/analytics.
13. **Verify AI output.** No AI-generated SQL/IaC goes directly to production.

## Reference architecture

```text
Prescriber / Pharmacy / EHR / Patient Apps
                   |
             APIs / Services
                   |
       +-----------+-----------+
       |                       |
 Kafka / Kinesis             Redis
       |                       |
    Consumers                  |
       +-----------+-----------+
                   |
               RDS Proxy
                   |
       +-----------+-----------+
       |                       |
 Writer endpoint          Reader endpoint
       |                       |
 Aurora Writer          Aurora Readers
       \                       /
        \                     /
       Distributed Aurora Storage
          across multiple AZs
                   |
       +-----------+-----------+
       |                       |
 Automated backup/PITR    Global Database
                              if required
                   |
              Logical CDC
                   |
               Debezium
                   |
                 Kafka
                   |
          Search / Analytics
```

## Current monitoring terminology

The DrFirst material says **Performance Insights**. As of 2026, AWS has moved the console experience to **CloudWatch Database Insights**. In an interview, say:

> “Performance Insights / now CloudWatch Database Insights”

That shows you understand both the job description terminology and the current AWS console.

## Fast study links

- [Architecture and Sizing](01-Architecture-and-Sizing)
- [Performance Tuning and Monitoring](02-Performance-Tuning-and-Monitoring)
- [Connection Management and RDS Proxy](03-Connection-Management-and-RDS-Proxy)
- [HA, Backup, and DR](05-HA-Backup-and-DR)
- [Upgrades, Blue/Green, and QPM](06-Upgrades-Blue-Green-and-QPM)
- [Workshop High-Value Topics](10-Workshop-High-Value-Topics)
- [60-Minute Panel Interview Runbook](11-Panel-Interview-Runbook)
