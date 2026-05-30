# Spec: Expense Tracking

## Requirements

1. User sends a text message describing an expense (e.g., "gastei 50 no mercado")
2. Bedrock extracts: amount, category, description from natural language
3. Expense is saved to DynamoDB with timestamp
4. Bot confirms: "Got it! R$50.00 in Groceries. Total this month: R$X"
5. User can send audio (future) or photo of receipt (future) — text only for MVP
6. Categories are auto-detected by AI: Food, Transport, Health, Entertainment, Bills, Shopping, Other

## Design

### Components
- `src/handlers/expense.py` — Expense recording logic
- `src/lib/bedrock.py` — Bedrock AI client (extract expense data)
- `src/lib/dynamo.py` — Add `put_expense()` and `get_monthly_total()`

### Bedrock Prompt
```
Extract expense information from this message.
Return JSON: {"amount": number, "category": string, "description": string}
Categories: Food, Transport, Health, Entertainment, Bills, Shopping, Other
Message: "{user_message}"
```

### DynamoDB Record
```json
{
  "PK": "USER#+5511999999999",
  "SK": "EXP#2026-06-01T10:30:00Z",
  "amount": 50.00,
  "category": "Food",
  "description": "mercado",
  "raw_message": "gastei 50 no mercado",
  "created_at": "2026-06-01T10:30:00Z"
}
```

### Flow
```
User message → Bedrock (extract) → DynamoDB (save)
  → get_monthly_total(phone, month)
  → Reply: "Got it! R$50 in Food. Month total: R$350"
```

## Tasks

- [ ] Implement Bedrock client (`extract_expense` function)
- [ ] Implement `put_expense(phone, amount, category, description)` in dynamo.py
- [ ] Implement `get_monthly_total(phone, year_month)` in dynamo.py
- [ ] Implement expense handler (parse → save → reply with total)
- [ ] Create Terraform for Bedrock model access (IAM permissions)
- [ ] Write tests for Bedrock extraction (mock responses)
- [ ] Write tests for expense handler (happy path + invalid input)
