# Contributing to FinBot

Welcome! This guide explains how to contribute to the FinBot project.

## Before You Start

1. Read the [Setup Guide](SETUP.md) to configure your environment
2. Read the [Architecture](ARCHITECTURE.md) to understand the system
3. Check the [Specs](./../.kiro/specs/) for available tasks
4. Look at the [GitHub Project board](https://github.com/goldenjackets-community/finbot/projects) for current status

## Workflow

### 1. Pick a task

- Open the GitHub Project board
- Choose an issue from the "Todo" column
- Assign yourself
- Move it to "In Progress"

### 2. Create a branch

```bash
git checkout main
git pull
git checkout -b feat/short-description
```

Branch naming:
- `feat/webhook-validation` — new feature
- `fix/signature-check` — bug fix
- `docs/setup-guide` — documentation
- `test/expense-handler` — tests only

### 3. Implement

Use Kiro IDE to help:
- Open the spec in the Specs panel
- Click on your task
- Let the agent implement, then review the code
- Make sure you understand what was generated!

### 4. Test

```bash
pytest tests/ -v
ruff check src/
```

Every function needs at minimum:
- 1 happy path test
- 1 error case test

### 5. Commit

Use conventional commits:
```bash
git commit -m "feat: implement webhook signature validation"
git commit -m "fix: handle missing phone number in message"
git commit -m "test: add expense handler error cases"
git commit -m "docs: update setup guide with AWS profile"
```

### 6. Push and open PR

```bash
git push -u origin feat/short-description
```

Then on GitHub:
- Open a Pull Request to `main`
- Title: same as your commit message
- Body: "Closes #N" (where N is the issue number)
- Wait for review

### 7. Address feedback

If changes are requested:
```bash
# Make changes
git add .
git commit -m "fix: address PR feedback"
git push
```

## Code Standards

### Python
- Type hints on all functions
- Docstrings on public functions
- No classes unless necessary — prefer functions
- Catch specific exceptions (never bare `except:`)

### Example
```python
def get_user(phone: str) -> dict | None:
    """Retrieve user profile from DynamoDB.
    
    Args:
        phone: E.164 format phone number (e.g., "+5511999999999")
    
    Returns:
        User profile dict or None if not found.
    """
    try:
        response = table.get_item(Key={"PK": f"USER#{phone}", "SK": "PROFILE"})
        return response.get("Item")
    except ClientError as e:
        logger.error("DynamoDB error", extra={"phone": mask_phone(phone), "error": str(e)})
        raise
```

### Naming
- Files: `snake_case.py`
- Functions: `snake_case()`
- Constants: `UPPER_SNAKE`
- DynamoDB attributes: `camelCase`

## What NOT to do

- ❌ Push directly to `main`
- ❌ Commit `.env` files or secrets
- ❌ Merge your own PR without review
- ❌ Leave failing tests
- ❌ Log PII (phone numbers, names) without masking

## Getting Help

- **Stuck on a task?** Ask in the WhatsApp group
- **Kiro not working?** Reload window (Ctrl+Shift+P → Reload)
- **AWS access issue?** Ping Ricardo
- **Git conflict?** Ask for help before force-pushing

## Recognition

Every merged PR counts as a contribution. Your GitHub profile will show:
- Contributions to `goldenjackets-community/finbot`
- Green squares on your activity graph
- PR history visible to recruiters

This is real experience for your portfolio. Own it! 🚀
