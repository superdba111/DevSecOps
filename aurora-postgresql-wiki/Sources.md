# Sources

**Last reviewed:** 2026-08-21

## User-supplied AWS workshops

1. [AWS Workshop Studio – Aurora/RDS PostgreSQL performance and scalability](https://catalog.us-east-1.prod.workshops.aws/workshops/098605dc-8eee-4e84-85e9-c5c6c9e43de2/en-US)
2. [AWS Workshop Studio – Aurora PostgreSQL workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/2a5fc82d-2b5f-4105-83c2-91a1b4d7abfe/en-US)

Workshop topics used heavily in this wiki:
- RDS Proxy pooling/multiplexing
- connection pinning
- Aurora performance/scalability
- operational tuning patterns

## Official AWS Aurora PostgreSQL documentation

- [Working with Amazon Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraPostgreSQL.html)
- [Best practices with Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.BestPractices.html)
- [Tuning with wait events for Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Tuning.html)
- [Essential Aurora PostgreSQL tuning concepts](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Tuning.concepts.html)
- [Aurora PostgreSQL wait-event summary](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Tuning.concepts.summary.html)
- [Initial troubleshooting for PostgreSQL performance issues](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/PostgreSQL.InitialTroubleshooting.html)
- [Working with PostgreSQL autovacuum](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Appendix.PostgreSQL.CommonDBATasks.Autovacuum.html)
- [Tuning Aurora PostgreSQL memory parameters](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.BestPractices.Tuning-memory-parameters.html)
- [Managing PostgreSQL temporary files](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/PostgreSQL.ManagingTempFiles.html)
- [High availability for Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)
- [Backing up and restoring Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Backups.html)
- [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [Aurora Serverless v2](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html)
- [RDS Proxy for Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy.html)
- [Avoiding RDS Proxy pinning](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy-pinning.html)
- [IAM database authentication](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/UsingWithRDS.IAMDBAuth.html)
- [Aurora PostgreSQL logical replication](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Replication.Logical.html)
- [Aurora Blue/Green Deployments](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/blue-green-deployments-overview.html)
- [Blue/Green best practices](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/blue-green-deployments-best-practices.html)
- [Aurora PostgreSQL Query Plan Management](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Optimize.overview.html)
- [QPM best practices](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Optimize.BestPractice.html)
- [CloudWatch Database Insights / Performance Insights](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_PerfInsights.html)

## Current details specifically reflected

- Aurora-specific wait events should be interpreted using Aurora guidance because storage behavior differs from open-source/RDS PostgreSQL.
- Autovacuum is responsible for dead-tuple cleanup/reuse, planner statistics maintenance, and XID wraparound protection.
- Temp files are created when query work exceeds available working memory and consume local DB-instance storage.
- RDS Proxy multiplexing is reduced by PostgreSQL session pinning; `DatabaseConnectionsCurrentlySessionPinned` is a key metric.
- QPM is implemented with `apg_plan_mgmt` and protects critical SQL against plan regressions.
- The Performance Insights console experience has transitioned to CloudWatch Database Insights; the Performance Insights API remains.
