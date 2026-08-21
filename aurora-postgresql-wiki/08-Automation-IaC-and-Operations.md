# Automation, IaC, and Operations

## 1. Infrastructure as Code

Manage repeatable Aurora infrastructure with Terraform or CloudFormation.

Typical resources/configuration:

```text
Aurora cluster
Aurora cluster instances
cluster parameter group
DB parameter groups
DB subnet groups
security groups
KMS keys
Secrets Manager
RDS Proxy + endpoints
CloudWatch alarms
IAM roles/policies
backup configuration
Global Database resources (if used)
```

Principles:

- reusable modules,
- environment-specific inputs,
- no manual production drift,
- peer review,
- plan/policy validation,
- stronger approvals for production.

## 2. Schema CI/CD

Use a migration framework such as Flyway/Liquibase where appropriate.

```text
Git -> PR -> validation -> DEV -> QA -> approval -> PROD -> verify
```

Check before production:

- backward compatibility,
- lock level,
- table rewrite risk,
- migration duration,
- rollback/roll-forward strategy,
- large-table impact.

For high-risk schema changes, prefer **expand-and-contract**:

1. add new compatible structure,
2. deploy code that understands old + new,
3. backfill,
4. switch reads/writes,
5. remove old structure later.

## 3. Automation backlog

“If it was done manually twice, consider scripting it.”

Good candidates:

- backup/restore verification,
- replica/slot health checks,
- database refresh workflows,
- parameter compliance reports,
- user/role audits,
- capacity reports,
- alert enrichment,
- DR readiness checks,
- post-deployment validation.

## 4. Operational runbooks

Minimum runbooks:

- writer failover,
- connection saturation,
- blocking/deadlock,
- runaway query,
- storage/replication-slot growth,
- failed migration,
- PITR restore,
- regional DR,
- certificate/credential rotation,
- major-version upgrade rollback/escalation.

Each runbook should define:

- detection,
- severity/impact,
- immediate containment,
- diagnostic commands,
- decision points,
- recovery procedure,
- verification,
- escalation,
- post-incident actions.

## 5. AI-assisted operations

Useful AI workflow:

```text
Logs / plan / sanitized error
          |
          v
AI analysis / draft script
          |
          v
Engineer verification
          |
          +--> official docs check
          +--> code review
          +--> lint/static checks
          +--> test/non-prod
          +--> Terraform plan / SQL EXPLAIN
          |
          v
Controlled deployment
```

Never submit PHI, production secrets, private keys, tokens, or restricted production data to an unapproved AI system.


## 6. Multi-account delivery pattern

A strong pattern for DEV/STAGE/PROD:

```text
Git / PR
   |
terraform validate + lint + security checks
   |
plan per account
   |
DEV apply
   |
integration/performance tests
   |
STAGE apply
   |
approval
   |
PROD apply
   |
post-deploy DB validation
```

Use separate deployment roles per account and avoid long-lived access keys.

For database schema:

```text
migration artifact
   |
DEV
   |
STAGE
   |
production approval
   |
PROD
```

Keep infrastructure and schema pipelines coordinated, but make rollback/roll-forward decisions explicit.

## 7. Automation examples Bernard/Nadeem may like

- restore a snapshot/PITR target and run validation SQL automatically,
- compare parameter groups against approved baseline,
- alert on stale replication slots,
- identify long-running/idle-in-transaction sessions,
- generate top-query reports from `pg_stat_statements`,
- validate indexes after schema deployment,
- collect pre/post deployment performance metrics,
- test RDS Proxy connectivity after failover.
