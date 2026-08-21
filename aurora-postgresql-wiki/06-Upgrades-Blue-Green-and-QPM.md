# Upgrades, Blue/Green, and Query Plan Management

## 1. Blue/Green Deployments

- **Blue:** current production.
- **Green:** synchronized staging/candidate production environment.

```text
BLUE: Aurora PostgreSQL old version
        |
        | replication
        v
GREEN: Aurora PostgreSQL new version
```

Recommended process:

1. Check source/target engine and feature compatibility.
2. Update/test extensions and drivers.
3. Create green environment.
4. Keep green read-only unless a carefully reviewed test requires writes.
5. Monitor replication lag and resource limits.
6. Run application regression/performance testing.
7. Define go/no-go and rollback criteria.
8. Switchover in a controlled window.
9. Validate critical transactions immediately.
10. Monitor closely after cutover.

Long-running transactions and bulk operations can increase replication lag during Aurora PostgreSQL Blue/Green preparation.

RDS Proxy and AWS-aware drivers can help reduce connection disruption around topology changes.

## 2. Query Plan Management (QPM)

Aurora QPM uses the `apg_plan_mgmt` extension to control plan evolution.

```text
Optimizer discovers plan
        |
        v
      QPM
        |
   +----+-----+
   |          |
Approved     New
plan         plan
   |          |
  use      unapproved
              |
            test
          /      \
      better     worse
        |          |
     approve     reject
```

QPM goals:

- protect critical queries from plan regression,
- preserve known-good plan baselines,
- evaluate newly discovered plans,
- approve improvements deliberately,
- reduce upgrade/statistics-change risk.

Good candidates:

- high-frequency patient-facing OLTP SQL,
- queries with severe performance sensitivity,
- major-version upgrade validation,
- queries previously affected by optimizer plan regression.

QPM does **not** replace:

- correct indexes,
- fresh statistics,
- SQL tuning,
- autovacuum,
- capacity management.

## 3. QPM + Blue/Green

A strong upgrade pattern:

```text
BLUE
Known production queries
   |
Approved QPM baselines
   |
   v
GREEN/new engine
   |
Run regression workload
   |
Capture/evaluate new plans
   |
Approve only improvements
   |
Controlled switchover
```

This combines:

- **environment-level safety** from Blue/Green,
- **SQL-plan-level safety** from QPM.

## 4. Version migrations

Before major upgrades, check:

- extension compatibility,
- application driver compatibility,
- parameter-group differences,
- logical replication requirements,
- long-running transactions,
- schema compatibility,
- plan regressions,
- rollback approach.

Use current AWS Blue/Green support matrix; do not assume every engine/version/Region combination supports every feature.


## 5. QPM limitations and operational cautions

QPM is powerful, but treat it as a guardrail rather than a substitute for tuning.

Operational points:

- It is implemented through the `apg_plan_mgmt` extension.
- QPM can manage `SELECT`, `INSERT`, `UPDATE`, and `DELETE` plans.
- The **plan baseline** is the set of Approved plans for a managed statement.
- New plans can be captured and evaluated rather than automatically trusted.
- Plan regression can occur after:
  - statistics changes,
  - constraints/environment changes,
  - query parameter changes,
  - PostgreSQL upgrades.
- AWS recommends adequate instance resources; plan management uses background workers.
- Keep the extension current for the Aurora PostgreSQL version in use.

Best target:

```text
small set of high-impact,
high-frequency,
latency-sensitive SQL
```

rather than treating every trivial query as equally critical.

## 6. Upgrade validation checklist

Before Blue/Green switchover:

- source/target version support,
- extension compatibility,
- parameter-group differences,
- driver/application compatibility,
- logical replication/CDC behavior,
- QPM baselines for critical SQL,
- replication lag,
- long-running transactions,
- performance test,
- rollback/go-no-go criteria,
- connection failover behavior,
- post-cutover SQL/latency checks.
