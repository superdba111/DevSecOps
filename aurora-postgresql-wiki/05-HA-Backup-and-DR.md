# High Availability, Backup, and Disaster Recovery

## 1. HA architecture

```text
AZ-A                  AZ-B                  AZ-C
Writer                Reader 1              Reader 2
  \                     |                    /
   \                    |                   /
      Distributed Aurora storage across AZs
```

Best practices:

- Have at least one reader in another AZ.
- Size the preferred failover target to handle writer load.
- Configure failover promotion tiers deliberately.
- Use the cluster endpoint so applications follow the current writer.
- Test failover from the application layer, not only the RDS console.

Aurora can promote an existing reader during writer failure; that is much faster than recreating a writer when no reader exists.

## 2. RTO and RPO

- **RTO:** maximum acceptable time to restore service.
- **RPO:** maximum acceptable data loss.

Architecture must be derived from business targets, not the reverse.

Example:

```text
RTO = 30 minutes
RPO = 5 minutes
```

means restore service within 30 minutes and lose no more than 5 minutes of data.

## 3. Backups and PITR

Aurora automated backups are continuous/incremental and retained for the configured retention window (1-35 days).

Use:

- automated backups,
- point-in-time restore,
- manual snapshots for longer retention where needed,
- AWS Backup where centralized policy/compliance requires it.

Important:

> HA is not backup.

An accidental logical `DELETE` can propagate successfully to every reader. Recovery from logical corruption may require PITR/restore.

## 4. Restore validation

A backup is not proven until restored.

Quarterly or scheduled restore test:

1. Restore into isolated environment.
2. Validate DB availability.
3. Validate critical schemas/objects.
4. Validate application connectivity.
5. Run critical read/write business checks.
6. Validate IAM/secrets/network dependencies.
7. Measure actual recovery time.
8. Record gaps and update runbooks.

## 5. DR drill

```text
Define scenario + success criteria
            |
            v
Validate backup/replica readiness
            |
            v
Initiate failover/recovery
            |
            v
Validate DB + application
            |
            v
Measure actual RTO/RPO
            |
            v
RCA / remediation / runbook update
```

Capture timestamps for detection, decision, failover/restore start, DB availability, and application restoration.

## 6. Aurora Global Database

Use when Region-level recovery or global reads justify the cost/complexity.

```text
Primary Region
 Aurora Writer
      |
 storage-level cross-Region replication
      |
Secondary Region
 read-only Aurora cluster
```

Use:

- **switchover** for planned regional movement,
- **failover** for primary-Region outage.

Do not assume a Global Database alone completes DR. Application compute, networking, DNS, IAM, secrets, queues, and dependent services also need regional recovery plans.


## 7. Application-level failover validation

Database status `available` is not the end of a failover test.

Validate:

```text
Aurora promotes reader
       |
cluster endpoint follows writer
       |
RDS Proxy / pool reconnects
       |
application retries correctly
       |
critical read/write transaction succeeds
```

Measure:

- detection time,
- DB promotion time,
- connection recovery time,
- application restoration time,
- failed/retried transactions.

This is the difference between **database HA** and **service HA**.

## 8. DR story structure for interview

```text
Target RTO/RPO
      |
failure scenario
      |
restore/failover mechanism
      |
application dependency validation
      |
business transaction validation
      |
actual RTO/RPO
      |
gaps + remediation
```

A strong sentence:

> “A successful backup job proves that a backup was created; only a restore test proves recoverability.”
