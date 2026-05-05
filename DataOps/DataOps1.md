# DataOps Interview Prep Wiki
> Healthcare Data Engineering · AWS Stack · KTLO & Reliability Focus

---

## Table of Contents

- [1. Role Overview & Narrative Pivot](#1-role-overview--narrative-pivot)
- [2. AWS Technical Deep Dive](#2-aws-technical-deep-dive)
  - [2.1 AWS Glue & Athena](#21-aws-glue--athena)
  - [2.2 AWS Lambda](#22-aws-lambda)
  - [2.3 Terraform (IaC)](#23-terraform-iac)
- [3. KTLO — Keep The Lights On](#3-ktlo--keep-the-lights-on)
  - [3.1 Incident Management (STAR Story)](#31-incident-management-star-story)
  - [3.2 Automation Angle](#32-automation-angle)
- [4. Healthcare Domain & Compliance](#4-healthcare-domain--compliance)
  - [4.1 HIPAA Compliance](#41-hipaa-compliance)
  - [4.2 Data Masking / Obfuscation](#42-data-masking--obfuscation)
  - [4.3 Auditability](#43-auditability)
  - [4.4 Data Reconciliation](#44-data-reconciliation)
- [5. Skills & Tools Checklist](#5-skills--tools-checklist)
- [6. Sample Interview Questions & Keywords](#6-sample-interview-questions--keywords)
  - [Q1 — Glue Job Failure at 3 AM](#q1--glue-job-failure-at-3-am)
  - [Q2 — Unannounced Schema Change](#q2--unannounced-schema-change)
  - [Q3 — Balancing KTLO vs New Automation](#q3--balancing-ktlo-vs-new-automation)
  - [Q4 — Handling PHI/PII Data](#q4--handling-phipii-data)
- [7. Ownership Mindset — Final Strategy](#7-ownership-mindset--final-strategy)

---

## 1. Role Overview & Narrative Pivot

**What this role really is:** A classic **DataOps / Reliability** role — not a feature-builder.  
They need a **"Reliability Hero"** who ensures data flows, stays clean, and remains compliant in a high-stakes healthcare environment.

### Narrative Shift

| ❌ Don't say | ✅ Say instead |
|---|---|
| "I build pipelines." | "I manage the lifecycle and health of data systems." |
| "I waited for the DevOps team." | "I investigated the Terraform logs, identified the IAM policy bottleneck, and proposed a fix to the platform team." |
| "I escalated it." | "I owned the triage end-to-end and restored service in 2 hours." |

> **Core framing:** Treat every data failure as a personal challenge to solve. The JD mentions "end-to-end ownership" multiple times — demonstrate it in every answer.

---

## 2. AWS Technical Deep Dive

Discuss every AWS service through an **operational lens** — not just "I've used it," but how you've kept it running in production.

---

### 2.1 AWS Glue & Athena

**Scenario:** A Glue job failed, or Athena queries are suddenly slow/expensive.

**Key concepts to know:**
- Partitioning strategies in S3
- Optimizing Parquet file sizes and formats
- Handling **schema evolution** (e.g., a new column in a claims file breaks the crawler)
- Managing the Glue Data Catalog
- Crawler settings and scheduling

**Pro Tip:**  
> Mention how you use **Athena to "smoke test" data quality** immediately after an ingestion job finishes. This signals operational instinct, not just build instinct.

---

### 2.2 AWS Lambda

**Scenario:** Using Lambda for event-driven triggers (e.g., a new file hits S3).

**Key concepts to know:**
- Handling **timeouts** and **memory allocation**
- **Dead-letter queues (DLQs)** — what happens when Lambda fails to process a specific file
- Retry logic and error handling patterns
- IAM roles scoped per function (least privilege)

---

### 2.3 Terraform (IaC)

**Strategy:** They use existing modules. They don't want a cowboy — they want discipline.

**Key concepts to know:**
- Understanding and managing **state files**
- Running and reviewing `terraform plan` before applying
- Safely promoting changes from **Staging → Production** without dropping a database
- Avoiding drift between environments

---

## 3. KTLO — Keep The Lights On

KTLO is a **major focus** of this role. In most interviews, people downplay maintenance. For this role, **celebrate it**.

---

### 3.1 Incident Management (STAR Story)

Use this template and fill in your own experience:

| Step | Example |
|---|---|
| **Situation** | A major claims file failed to load in production at an unexpected time. |
| **Task** | Triage the issue without losing data or missing SLA. |
| **Action** | Identified a schema mismatch using Python/SQL, patched the script, and reran the backfill. |
| **Result** | Restored service in 2 hours. Implemented a CloudWatch alert to catch the same issue earlier next time. |

---

### 3.2 Automation Angle

Show how you turned a **manual "restart" process into an automated self-healing pipeline**.

Keywords to use: `toil reduction`, `self-healing`, `automated remediation`, `operational efficiency`

---

## 4. Healthcare Domain & Compliance

You aren't just moving bits — you're moving **PHI (Protected Health Information)**. The stakes are higher. Proactively mention these concepts even if not directly asked.

---

### 4.1 HIPAA Compliance

- **Encryption at rest:** S3 AES-256
- **Encryption in transit:** TLS
- Know when each applies and how to verify compliance

---

### 4.2 Data Masking / Obfuscation

**Problem:** How do you let developers troubleshoot without seeing actual patient names?

**Answer:**
- **Hashing** (one-way, irreversible)
- **PII scrubbing** (strip fields before passing to dev/test environments)
- Synthetic data generation for lower environments

---

### 4.3 Auditability

Use AWS-native tools to prove **who touched what data and when**:

- **AWS CloudTrail** — API-level audit logging
- **S3 Access Logs** — object-level access tracking
- Retention policies for compliance windows

---

### 4.4 Data Reconciliation

In healthcare, you must **prove data integrity end-to-end**:

- **Row count checks** — if 1,000 records left the source, 1,000 must arrive at the target
- **Summation validations** — hash totals, financial field sums
- Reconciliation reports as part of every pipeline run

---

## 5. Skills & Tools Checklist

Prepare a concrete **"win" story** for each — not just familiarity, but a specific outcome you drove.

| Skill | Priority | What to Emphasize |
|---|---|---|
| **Python** | 🔵 Core | `boto3` (AWS SDK) + `pandas` for data validation. Scripting focus, not AI. |
| **PostgreSQL** | 🔵 Core | Indexing, `EXPLAIN ANALYZE`, handling large joins on claims tables. |
| **CI/CD** | 🟢 Important | GitHub Actions or GitLab. Frame it as: "code is data infrastructure." |
| **Observability** | 🟢 Important | CloudWatch, Datadog, or New Relic — monitoring "pipeline heartbeat." |
| **Terraform** | 🟡 Plus | State files, plan reviews, safe environment promotion. |
| **AWS Glue** | 🔵 Core | Schema evolution, crawler config, DPU tuning. |
| **AWS Athena** | 🔵 Core | Partition pruning, cost control, post-ingestion smoke tests. |
| **AWS Lambda** | 🟢 Important | DLQs, timeout handling, event-driven S3 triggers. |
| **SQL** | 🔵 Core | Complex joins, window functions, reconciliation queries. |

---

## 6. Sample Interview Questions & Keywords

Use the keywords below to anchor your answers and signal domain fluency.

---

### Q1 — Glue Job Failure at 3 AM

> **"A scheduled Glue job failed at 3 AM. Walk me through your troubleshooting steps."**

**Keyword anchors:**
`CloudWatch Logs` · `Error logs` · `S3 source file check` · `IAM permission check` · `Memory/DPU scaling` · `Glue job bookmarks` · `Schema mismatch` · `Retry policy`

**Talking points:**
1. Check CloudWatch Logs for the exact error
2. Verify the S3 source file exists and is well-formed
3. Check IAM role permissions (common silent failure)
4. Review memory/DPU allocation if OOM
5. Re-run with detailed logging enabled
6. Add alerting to catch it earlier next time

---

### Q2 — Unannounced Schema Change

> **"How do you handle a schema change in a source file that you weren't notified about?"**

**Keyword anchors:**
`Schema evolution` · `Glue Crawler settings` · `Data validation layer` · `Alerting upstream team` · `Schema registry` · `Backward compatibility`

**Talking points:**
1. Detection: validation layer catches the mismatch before it hits the target
2. Triage: compare new schema vs. expected schema, identify changed/added/removed columns
3. Decision: is it additive (safe) or breaking (halt pipeline)?
4. Communication: alert the upstream data provider immediately
5. Fix: update Glue Catalog, adjust ETL script, backfill if needed

---

### Q3 — Balancing KTLO vs New Automation

> **"How do you balance 'Keep the Lights On' work with your desire to build new automation?"**

**Keyword anchors:**
`Technical debt` · `Identifying toil` · `Operational efficiency roadmap` · `Toil budget` · `SRE principles`

**Talking points:**
1. Track recurring manual tasks — anything done more than twice gets automated
2. Use a toil budget (e.g., max 30% of sprint on repetitive work)
3. Prioritize automation that reduces future KTLO load (multiplier effect)
4. Document operational runbooks so toil is at least repeatable before it's automated

---

### Q4 — Handling PHI/PII Data

> **"Talk about a time you had to handle sensitive PII/PHI data."**

**Keyword anchors:**
`Least privilege (IAM)` · `Encryption at rest/transit` · `Data masking` · `Compliance audits` · `Access logging` · `Data minimization`

**Talking points:**
1. IAM roles scoped to minimum required access (least privilege)
2. Data never lands in lower environments unmasked
3. Encryption enforced at bucket policy level (not just application level)
4. Participated in or prepared for compliance audits with CloudTrail evidence

---

## 7. Ownership Mindset — Final Strategy

The job description mentions **"end-to-end ownership"** multiple times.

### What ownership looks like in answers:

```
Passive ❌:  "I reported the issue to the platform team and waited."
Owner  ✅:  "I investigated the Terraform logs, identified the IAM policy 
             bottleneck, and proposed a concrete fix to the platform team."

Passive ❌:  "The upstream team changed the schema without telling us."
Owner  ✅:  "I caught the schema drift via our validation layer, quarantined 
             the bad batch, and flagged the upstream team with a diff report."

Passive ❌:  "We had a manual process to restart failed jobs."
Owner  ✅:  "I identified that restart as toil, wrote a Lambda-based 
             self-healing trigger, and eliminated the 3 AM pages."
```

### Mindset checklist before every answer:
- [ ] Did I investigate before escalating?
- [ ] Did I implement a fix or just report a symptom?
- [ ] Did I add observability so it doesn't happen again?
- [ ] Did I document it so the next engineer doesn't have to figure it out from scratch?

---

*Last updated: 2026 · Role type: DataOps / Healthcare Data Engineering · Stack: AWS, Python, PostgreSQL, Terraform*
