# Development Setup

Get your environment ready to contribute to FinBot.

## Prerequisites

| Tool | Version | Purpose |
|------|---------|--------|
| Python | 3.12+ | Runtime |
| Git | 2.x | Version control |
| Terraform | 1.5+ | Infrastructure |
| AWS CLI | 2.x | AWS access |
| Kiro IDE | Latest | AI-assisted development |

## Step 1: Clone the repo

```bash
git clone git@github.com:goldenjackets-community/finbot.git
cd finbot
```

## Step 2: Python environment

```bash
python3.12 -m venv .venv
source .venv/bin/activate  # Linux/Mac
# .venv\Scripts\activate   # Windows

pip install -r requirements.txt
pip install -r requirements-dev.txt
```

## Step 3: AWS credentials

You'll receive SSO access to the FinBot AWS account (801010597252).

Add to `~/.aws/config`:
```ini
[profile finbot]
sso_session = cloud2point
sso_account_id = 801010597252
sso_role_name = AWSAdministratorAccess
region = us-east-1
```

Login:
```bash
aws sso login --profile finbot
```

## Step 4: Open in Kiro IDE

1. Open Kiro IDE
2. File → Open Folder → select the `finbot` directory
3. The IDE will automatically read `.kiro/steering/` and `.kiro/specs/`
4. Check the Specs panel on the left sidebar

## Step 5: Run tests

```bash
pytest tests/ -v
```

## Step 6: Pick a task

1. Go to the Specs panel in Kiro IDE
2. Choose a spec with unchecked tasks
3. Click on a task to start implementing
4. Or ask the agent: "Implement task X from spec Y"

## Workflow

```bash
# 1. Create a branch
git checkout -b feat/my-feature

# 2. Implement (with Kiro helping)
# ... code ...

# 3. Run tests
pytest tests/ -v

# 4. Commit
git add .
git commit -m "feat: implement webhook signature validation"

# 5. Push and open PR
git push -u origin feat/my-feature
# Then open PR on GitHub referencing the issue: "Closes #1"
```

## Useful Commands

| Command | Purpose |
|---------|--------|
| `pytest tests/ -v` | Run all tests |
| `pytest tests/ -x` | Stop on first failure |
| `ruff check src/` | Lint code |
| `ruff format src/` | Format code |
| `terraform plan` | Preview infra changes |
| `terraform apply` | Deploy infra |

## Environment Variables (for local testing)

Create a `.env` file (never commit this!):
```env
WHATSAPP_VERIFY_TOKEN=your-test-token
WHATSAPP_APP_SECRET=your-test-secret
WHATSAPP_ACCESS_TOKEN=your-test-token
WHATSAPP_PHONE_NUMBER_ID=123456789
DYNAMO_TABLE=finbot-table-dev
AWS_REGION=us-east-1
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `ModuleNotFoundError` | Activate venv: `source .venv/bin/activate` |
| AWS credentials expired | `aws sso login --profile finbot` |
| Tests fail with DynamoDB error | Make sure moto is installed: `pip install moto` |
| Kiro doesn't see specs | Reload window: Ctrl+Shift+P → "Reload Window" |
