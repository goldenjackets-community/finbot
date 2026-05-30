# Architecture

## System Overview

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────────┐
│   WhatsApp   │────▶│  Meta Cloud API │────▶│   API Gateway    │
│   User       │◀────│  (Webhook)      │◀────│   HTTP API       │
└──────────────┘     └─────────────────┘     └────────┬─────────┘
                                                      │
                                                      ▼
                                             ┌──────────────────┐
                                             │  Lambda: Router   │
                                             │  (webhook.py)     │
                                             └────────┬─────────┘
                                                      │
                              ┌────────────────┬──────┴───────┬────────────────┐
                              ▼                ▼              ▼                ▼
                     ┌────────────────┐ ┌────────────┐ ┌───────────┐ ┌──────────────┐
                     │ Registration   │ │  Expense   │ │  Budget   │ │   Goals      │
                     │ Handler        │ │  Handler   │ │  Handler  │ │   Handler    │
                     └───────┬────────┘ └─────┬──────┘ └─────┬─────┘ └──────┬───────┘
                             │                │              │               │
                             ▼                ▼              ▼               ▼
                     ┌─────────────────────────────────────────────────────────────┐
                     │                      DynamoDB                               │
                     │                   (Single Table)                            │
                     │                                                             │
                     │  PK: USER#<phone>  │  SK: PROFILE / BUDGET / EXP / GOAL     │
                     └─────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
                                    ┌──────────────────┐
                                    │  Amazon Bedrock  │
                                    │  (Claude Haiku)  │
                                    │                  │
                                    │  • NLU parsing   │
                                    │  • Categorization│
                                    │  • Insights      │
                                    └──────────────────┘
```

## Data Flow

### Expense Registration
```
User: "gastei 50 no mercado"
  │
  ▼
Webhook receives → validates signature → parses message
  │
  ▼
Router identifies: not a command → send to Expense Handler
  │
  ▼
Expense Handler → Bedrock: extract {amount: 50, category: "Food", description: "mercado"}
  │
  ▼
DynamoDB: PUT {PK: USER#+55..., SK: EXP#2026-06-01T10:30:00Z, amount: 50, ...}
  │
  ▼
DynamoDB: QUERY monthly total for user
  │
  ▼
WhatsApp API: "Got it! R$50 in Food. Month total: R$350"
```

### User Registration
```
New user sends any message
  │
  ▼
Router → get_user(phone) → NOT FOUND
  │
  ▼
create_user(phone) → DynamoDB PUT
  │
  ▼
WhatsApp: "Welcome to FinBot! What's your name?"
  │
  ▼
User replies: "Ricardo"
  │
  ▼
update_user(phone, {name: "Ricardo", onboarding_complete: true})
  │
  ▼
WhatsApp: "Hi Ricardo! Here's how I can help..."
```

## Infrastructure

| Component | Service | Config |
|-----------|---------|--------|
| API | API Gateway HTTP API | 100 req/s throttle |
| Compute | Lambda (Python 3.12) | 128MB, 30s timeout, 10 concurrency |
| Database | DynamoDB | On-demand, single-table |
| AI | Bedrock (Claude Haiku) | 5 req/min per user throttle |
| IaC | Terraform | State in S3 + DynamoDB lock |
| CI/CD | GitHub Actions | Lint → Test → Deploy |
| Monitoring | CloudWatch | Structured JSON logs |

## Security

- Webhook signature validation (HMAC SHA-256)
- Environment variables in Lambda (no hardcoded secrets)
- IAM least privilege per Lambda
- API Gateway throttling (DDoS protection)
- DynamoDB encryption at rest (default)
- No PII in logs (phone numbers masked)

## Cost Model (Monthly)

| Service | Free Tier | Estimated Cost |
|---------|-----------|----------------|
| Lambda | 1M requests | $0 |
| DynamoDB | 25 RCU/WCU | $0 |
| API Gateway | 1M requests | $0 |
| Bedrock | N/A | ~$1-2 |
| CloudWatch | 5GB logs | $0 |
| **Total** | | **~$1-2/month** |

Budget hard cap: $2/month with auto-alerts.
