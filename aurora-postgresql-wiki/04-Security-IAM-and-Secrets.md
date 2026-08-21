# Security, IAM, and Secrets

## 1. Recommended identity pattern

```text
EKS Pod / EC2 / Lambda
        |
 Workload IAM identity
        |
    rds-db:connect
        |
     RDS Proxy
        |
  Aurora PostgreSQL
        |
 dedicated DB role
```

For EKS, prefer **EKS Pod Identity** for new designs where supported; IRSA/OIDC remains a valid and secure established pattern.

## 2. IAM DB authentication

Best practices:

- Use dedicated PostgreSQL users; do not use the master user in applications.
- Grant `rds_iam` only to users intended for IAM authentication.
- Scope `rds-db:connect` to the specific cluster resource ID and database username.
- Use TLS.
- Generate auth tokens on demand; do not persist them as secrets.
- Reuse connections through application pooling or RDS Proxy.
- Validate that drivers support large IAM auth tokens.
- Account for the extra database memory required by IAM DB authentication.
- Do not casually mix IAM and password authentication behavior for the same PostgreSQL user.

Example:

```sql
CREATE USER prescription_app;
GRANT rds_iam TO prescription_app;
```

## 3. Role separation

Prefer separate DB identities:

```text
rx_api_user      -> OLTP least privilege
reporting_user   -> read-only
migration_user   -> controlled DDL
debezium_user    -> CDC/logical replication needs
dba_user         -> administration
```

Do not use one broad database identity for every workload.

## 4. Network controls

- Deploy Aurora in private subnets.
- Restrict security groups to approved application/proxy sources.
- Keep RDS Proxy in the appropriate VPC architecture.
- Use TLS certificate validation (`verify-full` where client support permits).
- Avoid exposing DB endpoints publicly unless a specifically reviewed architecture requires it.

## 5. Secrets Manager

Use Secrets Manager when password-based credentials remain necessary.

With RDS Proxy, choose an authentication model deliberately:

```text
App --IAM--> Proxy --secret/password--> DB
```

or where supported/appropriate:

```text
App --IAM--> Proxy --IAM--> DB
```

## 6. Encryption and audit

- KMS encryption at rest.
- TLS in transit.
- Least privilege for IAM and PostgreSQL roles.
- Log authentication/application activity to the extent required by policy.
- Centralize operational logs and alerts.
- Treat PHI and credentials as prohibited input to unapproved AI tools.
