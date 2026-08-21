# Aurora PostgreSQL Interview Cheat Sheet

## Architecture

**Aurora vs RDS PostgreSQL**  
Aurora is PostgreSQL-compatible but separates compute from distributed cluster storage. A cluster has a writer and optional readers; readers support read scaling and failover.

**Why readers?**  
Read scaling + already-running failover targets. Put a capable reader in another AZ and configure promotion priority deliberately.

**Provisioned vs Serverless v2?**  
Provisioned gives fixed/predictable capacity. Serverless v2 automatically scales within min/max ACUs. Choose from workload variability, latency, connections, cost, and required baseline headroom.

---

## Parameter tuning

**Two parameter-group levels?**  
Cluster parameter group = cluster/default behavior. DB parameter group = instance-specific behavior.

**Parameters to discuss**  
`work_mem`, connection limits, autovacuum parameters, slow-query logging, timeouts, logical replication, QPM settings.

**Rule**  
Do not call a parameter “optimized” until measured against actual workload.

---

## Performance

**Slow patient transaction?**

```text
scope -> Database Insights -> waits ->
pg_stat_activity -> pg_stat_statements ->
EXPLAIN ANALYZE BUFFERS -> root cause -> verify
```

**EXPLAIN ANALYZE**  
Look at estimates vs actuals, scan type, joins, loops, filters, buffers, temp spills, execution time.

**pg_stat_statements**  
Find high total time, high mean latency, high call volume, and changing workload behavior.

**Wait events**  
Ask what sessions are waiting for: CPU, I/O, locks, client, LWLocks.

**Temp file**  
Disk spill when sort/hash/etc. exceeds available working memory. Fix SQL/indexes/data volume before blindly raising `work_mem`.

**Autovacuum**  
Reclaims/reuses dead-tuple space, updates stats, limits bloat, and prevents XID wraparound.

---

## Connection management

**RDS Proxy**  
Managed connection pool/proxy; multiplexes many client sessions over fewer DB connections and improves resilience around failover.

**300 clients, 20 DB connections?**  
Most clients are not simultaneously executing. They borrow a DB connection for a transaction, return it, and another client reuses it.

**Pinning**  
Session state forces one client to stay on one DB connection. Correct when needed, but excessive pinning destroys multiplexing efficiency.

---

## Backup / HA / DR

**RTO** = time to restore service.  
**RPO** = acceptable data-loss window.

**Backup proof?**  
Restore it and run application-level read/write validation.

**Failover proof?**  
Measure DB promotion + pool/proxy reconnect + application recovery, not only DB status.

**Region DR?**  
Evaluate Global Database plus application/network/IAM/secrets dependencies.

---

## Upgrades

**Blue/Green**  
Blue = current production. Green = synchronized candidate. Validate green, define rollback criteria, then controlled switchover.

**QPM**  
`apg_plan_mgmt` lets you keep Approved plan baselines and evaluate new plans before adoption, reducing plan regression risk.

**Major upgrade**  
Check extensions, drivers, parameters, logical replication, plan regression, long transactions, performance, connection failover.

---

## Event driven

**Kafka/Kinesis -> Aurora**  
Broker decouples/buffers. Consumer writes authoritative transactional state to Aurora.

**Duplicate events**  
Design idempotent consumers with event IDs/unique business keys and careful DB commit vs offset acknowledgement ordering.

**WAL**  
Write-Ahead Log records changes before corresponding data pages are persisted; foundation for crash recovery, replication, PITR, logical decoding.

**Debezium**  
Reads logical changes through replication slots and publishes CDC events. Monitor stalled slots/WAL retention.

**Redis**  
Use only where the consistency/staleness model is explicitly acceptable. Cache invalidation is the hard part.

---

## Security

**IAM DB auth**  
Dedicated PostgreSQL user + `rds_iam`, narrowly scoped `rds-db:connect`, TLS, short-lived token, no master user in applications.

**EKS identity**  
For new workloads, evaluate EKS Pod Identity; IRSA/OIDC remains a secure established pattern.

---

## Automation

**Terraform/CloudFormation**  
Version infrastructure, parameter groups, proxy, IAM, monitoring, backups, and networking.

**Schema CI/CD**  
Version migrations; validate locking/rewrite risk; use expand/contract for high-risk large-table changes.

---

## AI

**Good answer**

> I use Claude Code, Codex, Gemini or ChatGPT to accelerate log analysis, plan interpretation, Python/shell/Terraform drafting, and documentation. I verify generated output against official documentation, review diffs, test in non-production, inspect Terraform plans and SQL execution plans, and use peer review/change controls before production. PHI, credentials, or restricted production data never go to an unapproved AI tool.

---

## Two stories to have ready

### Oceaneering
Long-term reliability across a mixed Oracle/SQL Server/PostgreSQL estate:
- production symptom,
- diagnosis,
- change,
- verification,
- prevention,
- multi-year operations mindset.

### NOAA / AWS
Use a real database or event-driven reliability/performance story:
- symptom,
- metrics/evidence,
- root cause,
- fix,
- measured outcome.

Never invent an Aurora incident if the real incident involved another AWS layer.
