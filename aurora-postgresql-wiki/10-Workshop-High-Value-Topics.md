# Workshop High-Value Topics

This page captures topics from the AWS workshops/documentation that are **especially useful for the DrFirst panel**, even when not explicitly named in the JD.

## 1. Wait-event-driven performance analysis

Why it matters:

> A query can be slow for very different reasons. CPU tuning will not fix a lock wait; adding an index will not fix a client/network wait.

Use:

```text
DB load
  |
top waits
  |
SQL/session causing wait
  |
root cause
```

Important Aurora waits:

```text
CPU
Client:ClientRead
Client:ClientWrite
IO:BufFileRead / IO:BufFileWrite
IO:DataFileRead
IO:XactSync
Lock:Relation
Lock:transactionid
Lock:tuple
LWLock:buffer_content
LWLock:BufferIO
```

This is one of the strongest workshop-derived interview topics.

---

## 2. RDS Proxy multiplexing

Workshop observation:

```text
300+ client connections
        |
      Proxy
        |
20+ DB connections
```

Why:

```text
borrow -> execute -> COMMIT -> return -> reuse
```

Client connections and DB connections are separate populations.

Interview value:
- proves you understand PostgreSQL connection cost,
- explains why proxy protects OLTP databases,
- opens discussion of failover and pooling.

---

## 3. RDS Proxy pinning

Pinning is the counter-example to multiplexing.

```text
session state
    |
connection pinned
    |
cannot be reused by other clients
```

Watch for:
- `SET`,
- prepared statements,
- temp objects,
- cursors,
- `LISTEN`,
- session advisory locks,
- session resets.

Monitor:

```text
DatabaseConnectionsCurrentlySessionPinned
```

Design high-volume API paths to be short and transaction-scoped where possible.

---

## 4. Temp files / `work_mem`

Temp-file waits can show up as:

```text
IO:BufFileRead
IO:BufFileWrite
```

Cause:

```text
sort/hash exceeds work_mem
       |
       v
disk temp file
```

Monitor:
- `log_temp_files`
- execution plans
- `FreeLocalStorage`

Do not blindly increase `work_mem`; concurrency can multiply memory use.

---

## 5. Autovacuum as a production feature

Autovacuum is not “background cleanup you ignore.”

It controls:

```text
dead tuples
bloat
statistics freshness
XID safety
```

High-write OLTP tables may need table-specific scale factors and appropriate worker/memory capacity.

---

## 6. Query Plan Management

Useful for critical SQL and upgrades.

```text
Approved plan
   |
baseline
   |
new optimizer plan
   |
Unapproved
   |
test
 /   \
better worse
 |      |
approve reject
```

This helps answer:

> “How do you prevent an engine upgrade from unexpectedly slowing a critical query?”

---

## 7. Serverless v2 vs provisioned

High-value architecture discussion:

```text
Provisioned
= fixed instance capacity

Serverless v2
= min/max ACUs and automatic scaling
```

Hybrid clusters can be useful, for example a predictable provisioned writer with elastic reader capacity where supported and justified.

---

## 8. Failover is an application test

Do not stop at:

```text
DB instance = available
```

Verify:

```text
new writer
  |
endpoint/proxy
  |
pool reconnect
  |
app retry
  |
critical transaction
```

---

## 9. IAM authentication and identity

Good AWS-native path:

```text
EKS Pod
   |
Pod Identity or IRSA/OIDC
   |
IAM Role
   |
rds-db:connect
   |
RDS Proxy / Aurora
   |
dedicated PostgreSQL user
```

Use short-lived credentials, TLS, and least privilege.

---

## 10. Workshop topics that are lower priority for this interview

Do not spend disproportionate preparation time on features unrelated to the JD simply because they appear in a workshop.

Prioritize:

```text
performance
connections
HA/DR
upgrades
CDC
automation
security
```

over peripheral Aurora features unless Bernard or Nadeem introduces them.
