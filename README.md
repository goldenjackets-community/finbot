# FinBot

AI-powered personal finance assistant on WhatsApp.

## What is this?

FinBot helps users manage their personal finances through natural conversation on WhatsApp. Users can track expenses, set budgets, define goals, and get AI-powered insights — all without leaving their favorite messaging app.

## Stack

| Layer | Technology |
|-------|------------|
| Runtime | Python 3.12 |
| Compute | AWS Lambda |
| Database | DynamoDB (single-table) |
| AI | Amazon Bedrock (Claude Haiku) |
| Auth | Amazon Cognito |
| API | API Gateway (HTTP API) |
| Messaging | WhatsApp Business API |
| IaC | Terraform |
| CI/CD | GitHub Actions |

## Project Structure

```
finbot/
├── .kiro/
│   ├── steering/       # Project rules and conventions
│   └── specs/          # Feature specifications
├── src/
│   ├── handlers/       # Lambda function handlers
│   └── lib/            # Shared business logic
├── infra/              # Terraform modules
├── tests/              # Unit and integration tests
└── docs/               # Additional documentation
```

## Getting Started

1. Clone this repo
2. Open in Kiro IDE
3. Read the specs in `.kiro/specs/`
4. Pick a task and start building!

## Contributing

This project is built by [Golden Jackets Contributors](https://goldenjacketsbrazil.com). See the specs for available tasks.

### Workflow

1. Create a branch from `main`
2. Implement your task (use Kiro to help!)
3. Open a PR referencing the issue number
4. Wait for review and merge

## MVP Scope (8 weeks)

1. User registration via WhatsApp
2. Monthly budget setup
3. Expense tracking (text/audio/photo)
4. Smart alerts
5. Financial goals
6. Basic web dashboard
7. AI conversational assistant (Bedrock)

## License

Private — Golden Jackets Community
