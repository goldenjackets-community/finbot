# Spec: User Registration

## Requirements

1. When a new user sends any message, check if they exist in DynamoDB
2. If user doesn't exist, create a new record with phone number and timestamp
3. Send welcome message asking for their name
4. When user replies with name, update their profile
5. Send confirmation message with quick instructions
6. If user already exists, skip registration and route to normal flow

## Design

### Components
- `src/handlers/registration.py` — Registration flow logic
- `src/lib/dynamo.py` — DynamoDB operations (get_user, create_user, update_user)

### DynamoDB Record
```json
{
  "PK": "USER#+5511999999999",
  "SK": "PROFILE",
  "name": "Ricardo",
  "phone": "+5511999999999",
  "status": "active",
  "onboarding_complete": true,
  "created_at": "2026-06-01T10:00:00Z",
  "updated_at": "2026-06-01T10:01:00Z"
}
```

### Flow
```
Message received → get_user(phone)
  ├── User exists → route to normal handler
  └── User not found → create_user(phone)
       → Send: "Welcome to FinBot! What's your name?"
       → Wait for reply
       → update_user(phone, name)
       → Send: "Hi {name}! Here's how I can help..."
```

## Tasks

- [ ] Implement `get_user(phone)` in dynamo.py
- [ ] Implement `create_user(phone)` in dynamo.py
- [ ] Implement `update_user(phone, data)` in dynamo.py
- [ ] Implement registration handler (check existence → create → welcome)
- [ ] Implement name capture flow (detect state, update profile)
- [ ] Create Terraform for DynamoDB table (finbot-table)
- [ ] Write tests for dynamo.py operations (moto)
- [ ] Write tests for registration flow (new user + existing user)
