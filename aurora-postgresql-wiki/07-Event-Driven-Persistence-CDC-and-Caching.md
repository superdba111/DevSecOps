# Event-Driven Persistence, CDC, and Caching

## 1. Persistence architecture

```text
EHR / Pharmacy / Prescriber
          |
          v
     Kafka / Kinesis
          |
          v
     Consumer Service
          |
          v
   Aurora PostgreSQL
   system of record
          |
      +---+----+
      |        |
    Redis     CDC
               |
            Debezium
               |
             Kafka
          +----+----+
          |         |
      OpenSearch  Analytics
```

## 2. Why Kafka/Kinesis before PostgreSQL?

Benefits:

- producer/consumer decoupling,
- buffering during downstream pressure,
- replay,
- independent scaling,
- fan-out to multiple consumers.

Design for:

- ordering,
- duplicate delivery,
- retry behavior,
- poison messages,
- backpressure,
- idempotency,
- transaction/offset ordering.

## 3. Idempotency

Do not assume “exactly once” magically eliminates duplicates.

Use:

- unique event IDs,
- unique business keys,
- processed-event tables where appropriate,
- upsert/idempotent business logic,
- careful DB commit vs message acknowledgement/offset commit ordering.

## 4. WAL and logical replication

WAL = **Write-Ahead Log**.

Simplified:

```text
DB change
   |
write WAL first
   |
COMMIT durable
   |
data pages can flush later
```

WAL underpins recovery and replication.

Aurora PostgreSQL logical replication:

```text
Aurora WAL
   |
logical decoder
   |
replication slot
   |
subscriber / CDC
```

## 5. Debezium

Debezium reads logical changes and publishes them as change events.

Operational guardrail:

> A stalled logical replication slot can retain WAL and consume storage.

Monitor:

```sql
SELECT *
FROM pg_replication_slots;
```

Also monitor replication-slot disk/WAL usage and whether slots are advancing.

## 6. Redis/ElastiCache

Use Redis for reads that can tolerate the defined consistency/staleness model.

Common cache-aside flow:

```text
Application -> Redis
             hit -> return
             miss
               |
               v
             Aurora
               |
            populate cache
```

The hard part is not “using Redis”; it is defining:

- TTL,
- invalidation,
- stale-data tolerance,
- authoritative-source rules,
- failure behavior.

For medication/patient-facing data, explicitly classify which data is safe to cache and which must be read authoritatively.
