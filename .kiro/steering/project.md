# FinBot — Project Steering

## Stack

- **Runtime:** Python 3.12
- **Compute:** AWS Lambda (one function per handler)
- **Database:** DynamoDB single-table design
- **AI:** Amazon Bedrock (Claude Haiku for cost efficiency)
- **Auth:** Amazon Cognito (user pool for WhatsApp users)
- **API:** API Gateway HTTP API
- **Messaging:** WhatsApp Business API (Meta Cloud API)
- **IaC:** Terraform
- **CI/CD:** GitHub Actions
- **Region:** us-east-1

## Architecture

```
WhatsApp → Meta Webhook → API Gateway → Lambda (router)
                                            ↓
                                    ┌───────┴───────┐
                                    │               │
                              DynamoDB          Bedrock
                           (user data)      (AI responses)
```

## Conventions

### Code
- Language: Python 3.12
- One Lambda handler per file in `src/handlers/`
- Shared logic in `src/lib/`
- Use type hints
- Docstrings on public functions
- No classes unless necessary — prefer functions
- Error handling: catch specific exceptions, log with context

### Naming
- Files: snake_case (`expense_tracker.py`)
- Functions: snake_case (`register_user`)
- Constants: UPPER_SNAKE (`MAX_BUDGET_CATEGORIES`)
- DynamoDB attributes: camelCase (`phoneNumber`, `monthlyBudget`)

### Testing
- Framework: pytest
- Tests mirror src structure: `tests/handlers/`, `tests/lib/`
- Mock AWS services with moto
- Minimum: 1 happy path + 1 error case per function

### Git
- Branch naming: `feat/short-description` or `fix/short-description`
- Commit messages: conventional commits (`feat:`, `fix:`, `docs:`, `test:`)
- Always open PR — never push directly to main
- Reference issue number in PR: `Closes #N`

### Infrastructure
- All resources in Terraform (`infra/`)
- Use variables for environment-specific values
- Tag everything: `Project=finbot`, `Environment=dev`
- Budget: hard limit $10/month (AWS Budget + Lambda concurrency cap)

## DynamoDB Single-Table Design

| Entity | PK | SK |
|--------|----|----|  
| User | `USER#<phone>` | `PROFILE` |
| Budget | `USER#<phone>` | `BUDGET#<yyyy-mm>` |
| Expense | `USER#<phone>` | `EXP#<timestamp>` |
| Goal | `USER#<phone>` | `GOAL#<goal-id>` |

## Cost Controls

- Lambda: 10 concurrent executions max
- Bedrock: throttle to 5 req/min per user
- DynamoDB: on-demand (pay per request)
- API Gateway: 100 req/s throttle
- AWS Budget: $10/month alert + auto-stop
