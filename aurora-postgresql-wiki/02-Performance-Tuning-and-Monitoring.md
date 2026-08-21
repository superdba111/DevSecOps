# Performance Tuning and Monitoring

## 1. Senior-DBA troubleshooting model

For almost any production performance question:

```text
Patient/API symptom
       |
       v
Scope impact and change window
       |
       v
CloudWatch / Database Insights
       |
       v
TOP WAIT EVENTS
       |
   +---+----+----+
   |        |    |
  CPU      I/O  LOCK
   |        |    |
   +---+----+----+
       |
       v
pg_stat_activity
       |
       v
pg_stat_statements
       |
       v
EXPLAIN (ANALYZE, BUFFERS)
       |
  +----+-------+-------+--------+
  |            |       |        |
bad plan    blocking  temp    vacuum/
                       spill    bloat
  |            |       |        |
  +------------+-------+--------+
               |
               v
        Fix smallest layer
               |
               v
      Measure before/after
               |
               v
          RCA + prevention
```

The strongest interview answer starts with **what the database is waiting for**, not with “I would add an index.”

---

## 2. Aurora wait-event analysis

AWS treats wait events as a primary Aurora PostgreSQL tuning tool.

A wait event tells you **why a session is not making progress right now**.

Important categories:

| Wait | Typical interpretation |
|---|---|
| `CPU` | Active on CPU or waiting for CPU |
| `Client:ClientRead` | DB waiting for client/application input |
| `Client:ClientWrite` | DB waiting to send data to client |
| `IO:BufFileRead/Write` | Temporary-file I/O |
| `IO:DataFileRead` | Required page not in shared memory |
| `IO:XactSync` | Waiting for Aurora storage commit acknowledgement |
| `Lock:Relation` | Table/view lock contention |
| `Lock:transactionid` | Row-level transaction lock contention |
| `Lock:tuple` | Waiting for tuple lock |
| `LWLock:buffer_content` | Contention accessing a shared buffer page |
| `LWLock:BufferIO` | Waiting for another process to complete buffer I/O |

### Important nuance

Aurora's storage architecture differs from open-source PostgreSQL and RDS PostgreSQL. Do **not** assume every same-named I/O wait means exactly the same thing across engines.

A wait event is not automatically a problem. The problem is sustained/high database load dominated by a wait that explains the business symptom.

---

## 3. First-line PostgreSQL tools

### Active sessions / waits

```sql
SELECT pid,
       usename,
       state,
       wait_event_type,
       wait_event,
       xact_start,
       query_start,
       query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

### Top SQL

```sql
SELECT query,
       calls,
       total_exec_time,
       mean_exec_time,
       rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

Ask:

- Is the problem one expensive query?
- A very frequent query?
- A sudden call-volume increase?
- A plan regression?
- Blocking?
- Connection saturation?

### Execution plan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

Look for:

- estimated vs actual row mismatch,
- `Seq Scan` / `Index Scan` / `Bitmap Heap Scan`,
- `Nested Loop` / `Hash Join` / `Merge Join`,
- loops,
- rows removed by filter,
- buffer hits/reads,
- sort/hash spills,
- execution-time hotspots.

---

## 4. Query plan optimization

Do not force an index because a plan “looks wrong.”

Preferred order:

```text
Bad plan
  |
Check estimates
  |
statistics stale/skewed?
  |
ANALYZE / extended stats if justified
  |
index design
  |
SQL shape
  |
parameters / plan cache
  |
QPM for critical plan stability
```

### Example composite index

```sql
CREATE INDEX idx_rx_patient_status
ON prescriptions(patient_id, status);
```

### Example partial index

```sql
CREATE INDEX idx_rx_active_patient
ON prescriptions(patient_id)
WHERE status = 'ACTIVE';
```

Indexes improve reads but cost writes, storage, vacuum work, and maintenance.

---

## 5. Statistics

Large estimate errors can produce bad join/scan choices.

```sql
ANALYZE prescriptions;
```

For important skewed columns, consider a higher statistics target after measurement:

```sql
ALTER TABLE prescriptions
ALTER COLUMN patient_id
SET STATISTICS 1000;

ANALYZE prescriptions;
```

---

## 6. MVCC and autovacuum

PostgreSQL MVCC allows high concurrency by maintaining row versions.

```text
UPDATE / DELETE
       |
       v
obsolete row versions
       |
       v
dead tuples
       |
       v
VACUUM
```

Autovacuum responsibilities:

- reclaim/reuse dead-tuple space,
- update planner statistics through ANALYZE,
- limit table/index bloat,
- protect against transaction ID wraparound.

Monitor:

```sql
SELECT relname,
       n_live_tup,
       n_dead_tup,
       last_autovacuum,
       last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

High-write OLTP often needs tuning beyond conservative defaults.

Relevant parameters:

```text
autovacuum_max_workers
autovacuum_vacuum_scale_factor
autovacuum_vacuum_threshold
autovacuum_analyze_scale_factor
autovacuum_analyze_threshold
autovacuum_work_mem
```

Example table-specific tuning:

```sql
ALTER TABLE prescriptions SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_analyze_scale_factor = 0.01
);
```

Values are examples, not defaults to copy blindly.

### Important production risks

- Idle-in-transaction sessions can hold locks and old snapshots.
- Autovacuum that cannot keep up leads to bloat and stale statistics.
- XID age approaching wraparound is a database safety issue, not only a performance issue.

---

## 7. Temp files and `work_mem`

Complex operations can use `work_mem` for each sort/hash operation and parallel worker.

When memory is insufficient:

```text
ORDER BY / HASH / GROUP BY / DISTINCT
               |
         use work_mem
               |
          enough?
         /      \
       yes      no
        |        |
     memory    temp file
                  |
               disk I/O
```

Temporary files are automatically removed after query completion, but heavy spilling increases latency and local-storage pressure.

Check plans for:

```text
Sort Method: external merge  Disk: ...
```

Monitor logging with:

```text
log_temp_files
```

Aurora temp files consume **local instance storage**, so also watch:

```text
FreeLocalStorage
```

### Correct tuning order

1. optimize SQL,
2. reduce rows processed,
3. fix indexes/statistics,
4. verify join/sort shape,
5. then tune `work_mem` based on concurrency.

Do not globally set huge `work_mem`; the setting can be consumed multiple times per query and by many concurrent sessions.

---

## 8. Locks and long transactions

Use:

```sql
SELECT *
FROM pg_locks;
```

together with `pg_stat_activity`.

For blocking:

1. identify blocker and waiter,
2. inspect transaction age and SQL,
3. determine business owner/operation,
4. terminate only if operationally justified,
5. fix transaction scope/order/indexing/DDL behavior permanently.

---

## 9. Connection pressure

Symptoms:

- high `DatabaseConnections`,
- memory pressure,
- connection churn,
- application timeouts,
- proxy borrow latency.

Do not respond by only raising `max_connections`.

Investigate:

```text
active vs idle
pool size x pod count
connection leaks
transaction duration
RDS Proxy pinning
database CPU/memory
```

See [Connection Management and RDS Proxy](03-Connection-Management-and-RDS-Proxy).

---

## 10. Monitoring stack

Use multiple layers:

```text
Application APM (AppDynamics)
          |
Grafana / operational dashboards
          |
CloudWatch Database Insights
          |
CloudWatch metrics/alarms
          |
PostgreSQL native views
```

### Current AWS terminology

The **Performance Insights console experience ended on 2026-07-31** and redirects to **CloudWatch Database Insights**. The Performance Insights API remains.

In the interview:

> “I would start with Performance Insights / now CloudWatch Database Insights and identify DB load and waits, then correlate that to `pg_stat_activity`, `pg_stat_statements`, and execution plans.”
