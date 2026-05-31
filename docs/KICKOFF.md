# FinBot — Kickoff Document

## Project Overview

| Field | Value |
|-------|-------|
| **Project** | FinBot |
| **Type** | SaaS — AI-powered personal finance assistant |
| **Channel** | WhatsApp (Meta Cloud API) |
| **Team** | Golden Jackets Contributors |
| **Tech Lead** | Ricardo Gulias |
| **Start Date** | 2026-05-31 |
| **MVP Target** | 8 weeks (2026-07-26) |
| **Repo** | github.com/goldenjackets-community/finbot |
| **AWS Account** | 801010597252 (FinBot) |
| **Budget** | $2/month (hard cap) |

---

## High-Level Design (HLD)

### System Context

```
┌──────────┐     ┌───────────────┐     ┌──────────────────┐
│  User    │────▶│  WhatsApp     │────▶│  Meta Cloud API  │
│(phone)   │◀────│  (messaging)  │◀────│  (webhook)       │
└──────────┘     └───────────────┘     └────────┬─────────┘
                                                 │
                                                 ▼
                                      ┌──────────────────┐
                                      │  API Gateway     │
                                      │  (HTTP API)      │
                                      └────────┬─────────┘
                                               │
                                               ▼
                                      ┌──────────────────┐
                                      │  Lambda Router   │
                                      │  (webhook.py)    │
                                      └────────┬─────────┘
                                               │
                              ┌────────────────┼────────────────┐
                              ▼                ▼                ▼
                    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
                    │Registration │  │  Expense    │  │  Budget     │
                    │  Handler    │  │  Handler    │  │  Handler    │
                    └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
                           │                │                │
                           ▼                ▼                ▼
                    ┌─────────────────────────────────────────────┐
                    │            DynamoDB (Single Table)          │
                    └─────────────────────┬───────────────────────┘
                                          │
                                          ▼
                                 ┌─────────────────┐
                                 │ Amazon Bedrock  │
                                 │ (Claude Haiku)  │
                                 └─────────────────┘
```

### Components

| Component | Service | Purpose |
|-----------|---------|--------|
| API Entry | API Gateway HTTP API | Receive webhooks from Meta |
| Router | Lambda (Python 3.12) | Validate signature, parse, route |
| Handlers | Lambda (Python 3.12) | Business logic per feature |
| Database | DynamoDB | User data, expenses, budgets, goals |
| AI Engine | Bedrock (Claude Haiku) | NLU, categorization, insights |
| IaC | Terraform | All infrastructure |
| CI/CD | GitHub Actions | Lint → Test → Deploy |
| Monitoring | CloudWatch | Structured JSON logs |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| Response time | < 5 seconds (Meta requirement) |
| Availability | 99.9% (Lambda SLA) |
| Cost | < $2/month (free tier + budget cap) |
| Security | HMAC signature validation, no PII in logs |
| Scalability | 10 concurrent Lambda executions (cap) |

---

## Low-Level Design (LLD)

### DynamoDB Single-Table Schema

| Entity | PK | SK | Attributes |
|--------|----|----|------------|
| User | `USER#<phone>` | `PROFILE` | name, status, onboarding_complete, created_at |
| Budget | `USER#<phone>` | `BUDGET#<yyyy-mm>` | categories[], total_limit, spent |
| Expense | `USER#<phone>` | `EXP#<timestamp>` | amount, category, description, raw_message |
| Goal | `USER#<phone>` | `GOAL#<goal-id>` | name, target_amount, current_amount, deadline |

### Lambda Functions

| Function | Handler | Trigger | Timeout | Memory |
|----------|---------|---------|---------|--------|
| finbot-webhook-{env} | webhook.handler | API GW POST /webhook | 30s | 128MB |
| finbot-registration-{env} | registration.handler | Invoked by router | 10s | 128MB |
| finbot-expense-{env} | expense.handler | Invoked by router | 15s | 128MB |
| finbot-budget-{env} | budget.handler | Invoked by router | 10s | 128MB |

### API Gateway Routes

| Method | Path | Purpose | Auth |
|--------|------|---------|------|
| GET | /webhook | Meta verification (challenge-response) | None |
| POST | /webhook | Receive messages | Signature validation in Lambda |

### Security

| Layer | Mechanism |
|-------|----------|
| Webhook | HMAC SHA-256 signature validation |
| API | Throttle 100 req/s |
| Lambda | 10 concurrent executions max |
| Data | DynamoDB encryption at rest (default) |
| Logs | Phone numbers masked (last 4 digits only) |
| Secrets | Environment variables (no hardcoded values) |
| IAM | Least privilege per function |

### Terraform Structure

```
infra/
├── main.tf              # Provider, backend
├── variables.tf         # Environment, region, tags
├── api_gateway.tf       # HTTP API + routes
├── lambda.tf            # Functions + IAM roles
├── dynamodb.tf          # Table + GSIs
├── bedrock.tf           # IAM permissions for model access
├── cloudwatch.tf        # Log groups + alarms
├── budget.tf            # AWS Budget + alerts
└── outputs.tf           # API URL, table name, function ARNs
```

---

## Methodology

- **Kanban** (no sprints, no ceremonies)
- **Board:** Todo → In Progress → In Review → Done
- **WIP Limit:** 1-2 tasks per person
- **Communication:** WhatsApp (async) + PR comments (technical)
- **Reviews:** Tech Lead reviews all PRs before merge
- **Deploy:** Tech Lead only (contributors focus on code + tests)

---

## Timeline

| Week | Milestone | Specs |
|------|-----------|-------|
| 1-2 | Foundation | Webhook + Registration |
| 2-3 | Core Feature | Expense Tracking |
| 3-4 | Budgeting | Monthly Budget |
| 4-5 | Intelligence | Smart Alerts |
| 5-6 | Goals | Financial Goals |
| 6-8 | Frontend | Basic Web Dashboard |

---

## Team & Assignments

| Role | Person | Spec | Responsibility |
|------|--------|------|---------------|
| Tech Lead | Ricardo Gulias | 01 - Webhook Setup | Architecture, specs, reviews, deploy, infra |
| Contributor | Daniel | 02 - User Registration | Implementation, tests, PRs |
| Contributor | Ricardo | 03 - Expense Tracking | Implementation, tests, PRs |
| Contributor | Luiz Andrade | 04 - Monthly Budget | Implementation, tests, PRs |

---

## Cost Analysis

| Service | Free Tier | Estimated Monthly |
|---------|-----------|-------------------|
| Lambda | 1M requests | $0 |
| DynamoDB | 25 RCU/WCU | $0 |
| API Gateway | 1M requests | $0 |
| Bedrock (Haiku) | N/A | $1-2 |
| CloudWatch | 5GB logs | $0 |
| **Total** | | **$1-2/month** |

Budget hard cap: $2/month with auto-alerts at 80% and 100%.

---

*Document generated: 2026-05-31*
*Project: FinBot — Golden Jackets Contributors*
