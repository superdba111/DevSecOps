# AWS Enterprise AI Architecture

## General-Purpose Reference Architecture for Generative AI, RAG, Agents, Data, Security and Governance

**Platform:** Amazon Web Services
**Primary AI Platform:** Amazon Bedrock
**Last reviewed:** September 2026

---

# 1. Purpose

This document describes a general-purpose architecture for building enterprise artificial-intelligence applications on AWS.

It covers:

* Foundation models
* Generative AI
* Amazon Bedrock
* Retrieval-Augmented Generation (RAG)
* Amazon Bedrock Knowledge Bases
* Vector databases
* S3 Vectors
* Multimodal AI
* AI agents
* Amazon Bedrock AgentCore
* MCP and enterprise tools
* Structured-data analytics
* Text-to-SQL
* Document processing
* Prompt engineering
* AI evaluation
* Responsible AI
* Security
* Governance
* Observability
* FinOps
* CI/CD
* Infrastructure as Code

The architecture is intended to support many workloads rather than one specific application.

Typical use cases include:

* enterprise search
* knowledge assistants
* customer-service assistants
* developer assistants
* operations agents
* financial-analysis assistants
* research systems
* document intelligence
* workflow automation
* data analytics copilots
* multimodal search
* compliance assistants
* autonomous and semi-autonomous agents

---

# 2. The Most Important Architecture Principle

Do not think of enterprise AI as:

```text
User
 |
 v
LLM
 |
 v
Answer
```

A production enterprise system normally looks more like:

```text
                         USERS
                           |
                           v
                    Identity / SSO
                           |
                           v
                   Application Layer
                           |
                           v
                   AI Gateway / API
                           |
                           v
                AI Orchestration Layer
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
 Foundation Model         RAG                Agents
       |                   |                   |
       |                   |                   |
       |            Knowledge Base        AgentCore
       |                   |                   |
       |             Retrieval            Tools / MCP
       |                   |                   |
       +---------+---------+---------+---------+
                 |                   |
                 v                   v
          Enterprise Data       Enterprise APIs
                 |
     +-----------+-------------+
     |           |             |
     v           v             v
    S3       Databases      SaaS / APIs
 Data Lake   Warehouse      ERP / CRM
```

Around the entire architecture:

```text
+----------------------------------------------------------+
|                 ENTERPRISE CONTROLS                      |
|                                                          |
| IAM / RBAC / ABAC                                        |
| IAM Identity Center / Cognito                            |
| KMS                                                      |
| Secrets Manager                                          |
| VPC / PrivateLink                                        |
| Guardrails                                               |
| Agent policies                                           |
| CloudTrail                                               |
| CloudWatch                                               |
| Evaluation                                               |
| Data governance                                          |
| Human approval                                           |
| Cost management                                          |
| CI/CD                                                    |
+----------------------------------------------------------+
```

The model is therefore only one component of the system.

A useful principle is:

> **Enterprise AI is an application, data, security and governance architecture that happens to contain AI models.**

---

# 3. Current AWS AI Platform Architecture

A modern AWS architecture can be divided into approximately eight layers:

```text
1. Experience
2. Identity
3. AI orchestration
4. Models
5. Knowledge and data
6. Agents and tools
7. Security and governance
8. Operations and FinOps
```

Conceptually:

```text
+------------------------------------------------------+
|                 USER EXPERIENCE                      |
| Web / Mobile / Slack / Teams / API / IDE             |
+------------------------------------------------------+
                         |
+------------------------------------------------------+
|                 IDENTITY                             |
| IAM Identity Center / Cognito / Enterprise IdP       |
+------------------------------------------------------+
                         |
+------------------------------------------------------+
|             APPLICATION / ORCHESTRATION              |
| Lambda / ECS / EKS / Step Functions / Bedrock        |
+------------------------------------------------------+
                         |
+------------------------------------------------------+
|                   AI PLATFORM                        |
|                                                      |
| Amazon Bedrock                                       |
| Foundation Models                                    |
| Knowledge Bases                                      |
| Guardrails                                           |
| Prompt Management                                    |
| Evaluation                                           |
| AgentCore                                            |
+------------------------------------------------------+
                         |
+------------------------------------------------------+
|                ENTERPRISE KNOWLEDGE                  |
|                                                      |
| S3 / S3 Vectors                                      |
| OpenSearch                                           |
| Redshift                                             |
| Aurora / RDS                                         |
| Glue / Lake Formation                                |
| Enterprise SaaS                                      |
+------------------------------------------------------+
                         |
+------------------------------------------------------+
|             SECURITY / GOVERNANCE / OPS              |
|                                                      |
| IAM / KMS / PrivateLink                              |
| CloudTrail / CloudWatch                              |
| WAF / GuardDuty / Security Hub                       |
| IaC / CI-CD / FinOps                                 |
+------------------------------------------------------+
```

---

# 4. Amazon Bedrock

Amazon Bedrock is AWS's managed platform for building generative-AI applications using foundation model
