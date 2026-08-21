# 60-Minute DrFirst Panel Interview Runbook

## Expected flow

| Time | Area |
|---|---|
| 0-5 min | Introduction / production background |
| 5-15 min | Aurora/RDS architecture, parameter groups, multi-account operations |
| 15-28 min | Performance troubleshooting |
| 28-38 min | HA, backup, RTO/RPO, DR |
| 38-47 min | Kafka/Kinesis, CDC/Debezium, Redis |
| 47-54 min | Terraform/Python/CI/CD |
| 54-58 min | AI workflow / behavioral |
| 58-60 min | Your questions |

Actual order can vary; use this as study weighting, not a script.

## 1. Opening positioning

Position yourself as:

```text
Production DBA
   +
AWS cloud engineer
   +
automation / event-driven platform experience
```

Lead with:
- reliability,
- database operations,
- performance,
- HA/DR,
- automation.

Do not lead with peripheral data science topics.

## 2. Performance scenario answer

If asked:

> A patient API went from 100 ms to 5 seconds. What do you do?

Use:

```text
1. Confirm scope / recent change
2. Database Insights + CloudWatch
3. Identify top waits
4. pg_stat_activity
5. pg_stat_statements
6. EXPLAIN (ANALYZE, BUFFERS)
7. Check locks / temp spills / vacuum / connections
8. Apply smallest root-cause fix
9. Measure before/after
10. RCA + monitoring/prevention
```

## 3. Upgrade scenario answer

```text
compatibility assessment
       |
Blue/Green candidate
       |
extensions/drivers/parameters
       |
QPM baselines for critical SQL
       |
replication lag
       |
regression/performance test
       |
go/no-go + rollback criteria
       |
controlled switchover
       |
post-cutover validation
```

## 4. DR scenario answer

```text
business RTO/RPO
      |
architecture
      |
backup/replica/global strategy
      |
runbook
      |
DR drill
      |
app-level validation
      |
actual RTO/RPO
      |
remediation
```

## 5. Event-driven scenario answer

```text
Kafka/Kinesis
     |
consumer
     |
idempotent DB transaction
     |
Aurora system of record
     |
CDC/Debezium
     |
downstream consumers
```

Mention:
- duplicate delivery,
- ordering,
- retries,
- WAL/slots,
- backpressure,
- cache consistency.

## 6. AI answer

Use a concrete workflow:

```text
problem/log/plan
    |
AI-assisted analysis
    |
draft SQL/Python/Terraform/runbook
    |
human verification
    |
official docs
    |
non-prod test
    |
peer review
    |
controlled production change
```

Explicitly mention no PHI/secrets in unapproved AI systems.

## 7. Oceaneering story

Prepare one true example showing:

- a large mixed database estate,
- multi-year uptime/reliability,
- a concrete performance or availability incident,
- diagnosis,
- technical fix,
- verification,
- long-term prevention.

## 8. NOAA story

Prepare a real AWS incident in this shape:

```text
symptom
  |
metrics/evidence
  |
root cause
  |
fix
  |
measured result
```

If the real issue was not Aurora, say so and use the closest genuine database/event-driven incident rather than inventing one.

## 9. Questions to ask

Good options:

1. “What are your biggest PostgreSQL production pain points today—query plans, connections, autovacuum, upgrades, or replication?”
2. “Are the Aurora clusters primarily provisioned, Serverless v2, or a mix?”
3. “How standardized is the Aurora configuration across accounts today—Terraform modules, parameter baselines, and DR runbooks?”
4. “What would you want this person to have improved after the first six months?”
