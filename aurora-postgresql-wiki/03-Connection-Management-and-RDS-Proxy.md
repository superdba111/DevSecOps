# Connection Management and RDS Proxy

## 1. Why connection pooling matters

PostgreSQL connections consume memory and process/scheduling resources.

Without pooling:

```text
300 clients
     |
     v
~300 PostgreSQL sessions
```

With RDS Proxy:

```text
300+ client connections
          |
          v
      RDS Proxy
  pool + multiplexing
          |
          v
20-50 DB connections
(depending on active concurrency)
          |
          v
 Aurora PostgreSQL
```

This is exactly what the AWS workshop experiment demonstrates.

---

## 2. Client connection vs database connection

- **Client connection:** application -> RDS Proxy
- **Database connection:** RDS Proxy -> Aurora PostgreSQL

They are not 1:1 when multiplexing works.

### Borrow / execute / return

```text
Client A
   |
borrow DB #5
   |
transaction
   |
COMMIT
   |
return DB #5
   |
Client B reuses DB #5
```

This explains how 300+ connected clients may require only 20+ underlying DB sessions if only a small fraction are active concurrently.

---

## 3. When RDS Proxy is valuable

Good fits:

- EKS/ECS with many pods/tasks,
- Lambda concurrency,
- bursty connection creation,
- many short transactions,
- protecting Aurora from connection storms,
- smoother application recovery around failover,
- IAM/Secrets Manager integration,
- read-only endpoints for read scaling.

RDS Proxy does **not** fix:
- bad SQL,
- insufficient CPU,
- poor indexes,
- lock contention.

---

## 4. Pool sizing

Review:

```text
application instances
x pool size
= potential client connections
```

Then monitor:

- client connections,
- database connections,
- borrow latency,
- `max_connections`,
- CPU/memory,
- active/idle sessions,
- pinning.

Memorable rule:

> Connection pooling controls concurrency; it does not create database capacity.

---

## 5. Connection pinning

RDS Proxy normally multiplexes at **transaction boundaries**.

Pinning occurs when session-specific state makes it unsafe for another client to reuse the DB connection.

```text
Normal:
Client A -> DB #1 -> COMMIT -> pool -> Client B

Pinned:
Client A ------------------------> DB #1
                                  dedicated until session ends
```

For Aurora PostgreSQL, AWS documents pinning for interactions such as:

- `SET`,
- `PREPARE`, `EXECUTE`, `DEALLOCATE`, `DISCARD`,
- temporary tables/views/sequences,
- cursors,
- `LISTEN`,
- library loading such as `auto_explain`,
- sequence manipulation such as `nextval` / `setval`,
- session advisory locks,
- some session-state reset behavior.

Transaction-level advisory locks such as `pg_advisory_xact_lock` do **not** pin for that lock behavior.

Any SQL statement larger than 16 KB is also a general pinning condition.

### Best practice

Prefer:

```text
BEGIN
  short stateless work
COMMIT
```

over sessions that maintain state across many transactions.

Pinning is correct when session affinity is required; the goal is not to eliminate it at the expense of correctness.

Monitor:

```text
DatabaseConnectionsCurrentlySessionPinned
```

If almost every session is pinned, redesigning application behavior can restore multiplexing benefits.

---

## 6. Reducing avoidable pinning

- Remove unnecessary session-level `SET` statements.
- If every connection uses identical initialization settings, consider the proxy initialization query where appropriate.
- Avoid long-lived temporary objects in high-concurrency API paths unless truly needed.
- Review prepared-statement and pool reset behavior.
- Keep transactions short.
- Do not use `DISCARD ALL` blindly as a pool reset strategy with RDS Proxy.

---

## 7. Reader traffic

Architecture:

```text
Application
   |
   +--> Proxy RW endpoint -> Aurora writer
   |
   +--> Proxy RO endpoint -> Aurora readers
```

Use read-only routing only for workloads that tolerate reader semantics and possible replication/read-after-write considerations.

---

## 8. Interview answer

> RDS Proxy separates client connections from database connections and multiplexes many clients over a smaller pool of PostgreSQL sessions. That protects Aurora from connection storms and reduces connection setup overhead. I monitor borrow latency and pinning because session-specific PostgreSQL behavior can pin clients to individual DB sessions and reduce multiplexing efficiency.
