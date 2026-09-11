# AWS Multi-Account Provisioning: Terraform + Control Tower + Organizations + SCP

**Purpose:** Reference architecture and best practices for provisioning and governing a multi-account AWS platform using AWS Organizations, Control Tower, Account Factory for Terraform (AFT), Service Control Policies (SCPs), and Terraform.

---

## 1. Architecture Overview

```
AWS Organizations
      │
      ▼
AWS Control Tower
      │
      ├── OUs (Organizational Units)
      ├── Accounts
      ├── Control Tower guardrails/controls
      ├── SCP / RCP policies
      ├── IAM Identity Center (SSO)
      └── Log Archive / Audit accounts
      │
      ▼
Account Factory for Terraform (AFT)
      │
      ├── Account requests
      ├── Global customizations (every account)
      └── Account-specific customizations
      │
      ▼
Separate workload Terraform pipelines
      (networking, databases, EKS/ECS, data platform, applications)
```

**Key principle:** Keep account *vending/baseline* (AFT) separate from *application infrastructure* (workload Terraform). AWS explicitly scopes AFT to provisioning/customizing Control Tower accounts, not deploying app resources like EC2/EKS workloads.

---

## 2. Recommended OU / Account Structure

```
Root
├── Security OU
│   ├── Log Archive
│   └── Audit / Security Tooling
├── Infrastructure OU
│   ├── Network
│   ├── Shared Services
│   └── AFT Management
├── Workloads OU
│   ├── Production
│   │   ├── app1-prod
│   │   └── data-prod
│   └── NonProduction
│       ├── app1-dev
│       └── app1-test
├── Sandbox OU
│   └── developer-sandboxes
└── Suspended / Quarantine OU
```

**Practices:**
- Treat each **AWS account as an isolation boundary**, not a folder — this is AWS's own guidance for reducing blast radius.
- Keep the **Organizations management account empty** of workloads. This matters because SCPs and RCPs *cannot restrict the management account itself* — anything sensitive there is ungoverned.
- Separate Log Archive and Audit/Security accounts from everything else so logs can't be tampered with by workload account admins.

---

## 3. Division of Responsibility (avoid overlap)

| Layer | Responsibility |
|---|---|
| AWS Organizations | Account hierarchy, OU structure |
| Control Tower | Landing-zone guardrails/controls |
| SCP | Maximum permissions a *principal* can ever have |
| RCP (Resource Control Policy) | Maximum permissions a *resource* can grant to others |
| IAM Identity Center | Human/federated access |
| AFT | Terraform-based account vending & baseline customization |
| Workload Terraform repos | Application/cloud infrastructure |
| AWS Config | Detective compliance |
| CloudTrail | Audit logging |
| Security Hub / GuardDuty | Findings & threat detection |
| CI/CD (GitHub Actions, etc.) | Plan → review → apply |

Note: Control Tower is built on top of Organizations, IAM Identity Center, Service Catalog (Account Factory), and SCPs/RCPs — it's an orchestration layer, not a replacement for them.

---

## 4. Governance: Controls Before Custom SCPs

Use **Control Tower's built-in controls first**, then add custom SCP/RCP only for company-specific requirements.

Control Tower controls fall into three types:

- **Preventive** — blocks an action outright (implemented via SCP)
- **Detective** — flags non-compliant configuration after the fact (via AWS Config rules)
- **Proactive** — checks resources *before* deployment (via CloudFormation hooks)

Control Tower has moved toward using SCPs, RCPs, and declarative policies together for its preventive control library — this is a relatively recent evolution, so if you're following older tutorials, double-check the current Control Tower console for which control types are available in your region.

---

## 5. SCP Design Mental Model

SCPs are **not** grants of permission — they're a ceiling on what IAM can ever allow, even for admins.

```
IAM says:  "You may do this."
SCP says:  "Even if IAM says yes, this is the max you're allowed."

Effective permissions = IAM permissions ∩ SCP permissions ∩ (RCP, if resource-side)
```

Typical guardrail denies:
- `organizations:LeaveOrganization`
- Disabling CloudTrail / GuardDuty / Config
- Deleting security/log resources
- Actions outside approved regions

### Example: Region-restriction SCP

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnapprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "organizations:*",
        "route53:*",
        "cloudfront:*",
        "support:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2"]
        }
      }
    }
  ]
}
```

⚠️ The `NotAction` exemption list needs careful review per account — global services and edge cases vary. Don't copy this into production unmodified.

---

## 6. Policies as Code

```
organization-policies/
├── scp/
│   ├── deny-disable-security.tf
│   ├── restrict-regions.tf
│   ├── protect-cloudtrail.tf
│   └── deny-leave-org.tf
├── rcp/
│   └── protect-sensitive-resources.tf
├── tag-policies/
└── tests/
```

AWS publishes Prescriptive Guidance for managing SCPs/RCPs as code with Terraform or CloudFormation, including CI/CD and manifest-based OU/account targeting. Worth reading before building your own pattern from scratch.

---

## 7. Account Factory for Terraform (AFT)

```
Git push → aft-account-request repo → AFT pipeline → Control Tower Account Factory
   → new AWS account → baseline + global customization + account customization
```

AFT follows a GitOps model and supports Terraform (Community Edition, HCP Terraform, or Terraform Enterprise) plus alternate Git providers via AWS CodeConnections.

**On the Terraform version requirement:** AWS's AFT module documentation has stated different minimums as the module has evolved (older docs cite Terraform 0.15.x+; requirements have moved forward since). Always check the current `aft` module README/registry page for the minimum supported version at the time you deploy — don't rely on a fixed number from any single source, including this doc.

### The four AFT repositories

```
aws-platform/
├── aft-account-request                    # creates/updates accounts
├── aft-account-provisioning-customizations
├── aft-global-customizations              # applies to every account
└── aft-account-customizations             # per-workload-type config
```

**Example account request:**

```hcl
module "app1_prod" {
  source = "./modules/aft-account-request"

  control_tower_parameters = {
    AccountEmail              = "aws-app1-prod@example.com"
    AccountName               = "app1-prod"
    ManagedOrganizationalUnit = "Production"
    SSOUserEmail              = "platform@example.com"
    SSOUserFirstName          = "Platform"
    SSOUserLastName           = "Team"
  }

  account_tags = {
    Environment = "prod"
    Application = "app1"
    CostCenter  = "IT"
  }

  account_customizations_name = "production-workload"
}
```

A `git push` of the account request triggers provisioning.

**Global customizations** (every account gets these): CloudWatch baseline, AWS Config, Security Hub, GuardDuty integration, central log forwarding, backup config, common IAM roles, tagging, SSM, VPC Flow Logs, DNS integration. AFT customization stages support Terraform, Python, and Bash.

**Account customizations** (per workload type):

```
aft-account-customizations/
├── production-workload/terraform/
├── sandbox/terraform/
├── data-platform/terraform/
└── security/terraform/
```

The `account_customizations_name` field ties an account request to one of these.

---

## 8. Do NOT Deploy Application Infrastructure Through AFT

```
AFT → creates/baselines the AWS account
         │
         ▼
   workload Terraform (separate repo/pipeline) → EKS, RDS, ALBs, Lambda, DynamoDB, etc.
```

Mixing account-vending Terraform with application Terraform turns AFT into an unmaintainable monolith. Keep them in separate repos/state.

```
platform-iac/           workloads/
├── organization/        ├── app1/
├── control-tower/        ├── data-platform/
├── policies/            ├── db-platform/
├── aft/                 └── ai-platform/
└── shared-services/
```

---

## 9. Terraform State Strategy

- Use a dedicated state/security account (or tightly controlled shared-services account) for Community Edition.
- Backend: S3 + KMS encryption + versioning + native locking.

```hcl
terraform {
  backend "s3" {
    bucket       = "company-terraform-state-prod"
    key          = "network/prod/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
  }
}
```

**Correction/nuance:** `use_lockfile` (native S3 locking, no DynamoDB table needed) requires **Terraform 1.10+**. It was introduced as an experimental feature in 1.10 and has since become the recommended path forward, with DynamoDB-based locking being phased toward deprecation — but check the Terraform changelog for your exact version before assuming it's fully stable/default in your pinned version. If you're on Terraform <1.10, you still need the `dynamodb_table` argument for locking.

Also enable: S3 Block Public Access, KMS encryption, versioning, CloudTrail data events, restricted IAM on the state bucket.

### One state file per boundary, not per org

```
Bad:  one giant terraform.tfstate for 100 accounts / 5000 resources
Good: state per (layer × environment) — e.g.
      network/prod, network/dev, security/prod, data-platform/prod, app1/prod
```

This limits blast radius and avoids lock contention.

---

## 10. Cross-Account Terraform

Use provider aliases with `assume_role` for cross-account access:

```hcl
provider "aws" {
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformExecution"
  }
}

provider "aws" {
  alias  = "network"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformExecution"
  }
}
```

```hcl
module "network_logging" {
  source    = "../../modules/network-logging"
  providers = { aws = aws.network }
}
```

Prefer **account-scoped pipeline execution** (one pipeline/state per account) over stacking many provider aliases in one project — it's simpler to reason about and audit.

---

## 11. CI/CD Authentication

Avoid long-lived AWS access keys in CI:

```
GitHub Actions → OIDC → AWS IAM Role (short-lived, scoped)
```

- Never store `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` as long-lived CI secrets unless truly unavoidable.
- AFT and HCP Terraform also support OIDC/workload-identity integration for AWS auth.

---

## 12. Recommended Git Workflow

```
Branch → PR
   ├─ terraform fmt / validate
   ├─ tflint
   ├─ checkov / tfsec
   ├─ policy tests
   └─ terraform plan
        │
        ▼
Architecture/security review → Merge → plan again
        │
        ▼
Manual approval (required for prod) → terraform apply
```

For Organizations/SCP changes specifically, require **explicit human approval** before deployment — a misapplied SCP can lock an entire OU out of AWS. AWS's own SCP-as-code reference architecture includes a manual approval gate for this reason.

---

## 13. Guardrails for AI Coding Agents (if using Claude Code / Codex / similar)

If AI agents touch this repo, give them explicit, source-controlled instructions (`AGENTS.md`, `CLAUDE.md`) and restrict their AWS credentials to read-only + plan:

```
AI agent role permissions:
  ✔ Describe*, List*, Get*
  ✔ terraform plan
  ✘ terraform apply (especially to prod)
```

Production apply should always go through: PR → CI checks → plan → human approval → scoped production role → apply. This is a standard sandboxing principle for any coding agent with cloud reach, not specific to one vendor's tooling.

**Example `AGENTS.md` snippet:**

```markdown
## Deployment safety
NEVER execute without explicit authorization:
- terraform apply / terraform destroy
- aws organizations delete-*
- aws organizations leave-organization

Agents may run: terraform fmt, terraform validate, tflint, terraform plan
```

---

## 14. Suggested Tech Choices to Start

- **Control Tower** + **AFT** + **Terraform Community Edition** (HCP/Enterprise not required to start)
- **GitHub + GitHub Actions** with OIDC
- **S3 backend** with native locking (Terraform ≥1.10) or DynamoDB locking on older versions
- Separate repos for: org/policies, AFT, and each workload domain

---

## Corrections to the Original Notes

1. **AFT minimum Terraform version** — don't hard-code a specific version (e.g., "1.6.1+") in documentation; AWS has changed this requirement across AFT module releases (older docs cite 0.15.x+). Always pull the current minimum from the live AFT module registry page/README at deploy time.
2. **S3 native state locking (`use_lockfile`)** — correctly requires Terraform 1.10+, but was introduced as an *experimental* feature in that release. Verify its status (stable vs. experimental) against the Terraform version you've actually pinned, and keep a DynamoDB fallback plan if you're not yet on a version where it's fully proven for your use case.
3. Everything else in the original notes (OU structure, SCP vs IAM mental model, AFT's 4-repo model, separating AFT from workload Terraform, per-boundary state files, OIDC over static keys, RCP as a newer complement to SCP) checks out against current AWS guidance.

---

*Last verified: September 2026. AWS's Control Tower/AFT/SCP/RCP tooling changes fairly often — re-verify version-specific details (Terraform minimums, control catalog contents) against AWS's official docs before relying on them for a production rollout.*
