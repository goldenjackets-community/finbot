# Spec: Monthly Budget

## Requirements

1. User sets a monthly budget via message (e.g., "meu limite é 3000 por mês")
2. Bedrock extracts: total_limit from natural language
3. User can set per-category limits (e.g., "limite de 500 pra comida")
4. Budget is saved to DynamoDB (resets monthly)
5. After each expense, bot shows remaining budget: "R$50 in Food. Budget remaining: R$2,450/R$3,000 (82%)"
6. Alert when reaching 80% of budget: "⚠️ You've used 80% of your monthly budget!"
7. Alert when exceeding 100%: "🚨 Budget exceeded! R$3,150/R$3,000"
8. User can ask "quanto ainda posso gastar?" to see remaining

## Design

### Components
- `src/handlers/budget.py` — Budget management logic
- `src/lib/bedrock.py` — Add budget extraction prompt
- `src/lib/dynamo.py` — Add `put_budget()`, `get_budget()`, `update_spent()`

### Bedrock Prompt
```
Extract budget information from this message.
Return JSON: {"total_limit": number, "category": string|null}
If no category specified, it's a total monthly budget.
Message: "{user_message}"
```

### DynamoDB Record
```json
{
  "PK": "USER#+5511999999999",
  "SK": "BUDGET#2026-06",
  "total_limit": 3000.00,
  "category_limits": {"Food": 500, "Transport": 300},
  "spent": 550.00,
  "alert_80_sent": false,
  "alert_100_sent": false,
  "created_at": "2026-06-01T00:00:00Z"
}
```

### Flow
```
User sets budget → Bedrock (extract) → DynamoDB (save)
User adds expense → update spent → check thresholds → alert if needed
User asks remaining → get_budget → Reply: "R$2,450 remaining (82% used)"
```

## Tasks

- [ ] Implement Bedrock prompt for budget extraction
- [ ] Implement `put_budget(phone, total_limit, category_limits)` in dynamo.py
- [ ] Implement `get_budget(phone, year_month)` in dynamo.py
- [ ] Implement `update_spent(phone, year_month, amount)` in dynamo.py
- [ ] Implement budget handler (set budget, check remaining, alerts)
- [ ] Integrate with expense handler (update budget after each expense)
- [ ] Create Terraform for CloudWatch scheduled rule (monthly reset)
- [ ] Write tests for budget handler (set, check, 80% alert, 100% alert)
