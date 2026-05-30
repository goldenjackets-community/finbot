# Spec: WhatsApp Webhook Setup

## Requirements

1. API Gateway HTTP API receives POST requests from Meta WhatsApp Cloud API
2. API Gateway handles GET requests for webhook verification (challenge response)
3. Lambda function validates the webhook signature (X-Hub-Signature-256)
4. Invalid signatures return 401 and log the attempt
5. Valid messages are parsed and routed to the appropriate handler
6. API responds with 200 within 5 seconds (Meta requirement)

## Design

### Components
- `infra/api_gateway.tf` — HTTP API with `/webhook` route
- `src/handlers/webhook.py` — Receives and validates incoming messages
- `src/lib/whatsapp.py` — WhatsApp API client (send messages, verify signature)

### Flow
```
Meta → POST /webhook → Lambda webhook.py
  1. Verify X-Hub-Signature-256
  2. Parse message type (text, audio, image)
  3. Extract sender phone number
  4. Route to appropriate handler
  5. Return 200 immediately
```

### Environment Variables
- `WHATSAPP_VERIFY_TOKEN` — Token for GET verification
- `WHATSAPP_APP_SECRET` — For signature validation
- `WHATSAPP_ACCESS_TOKEN` — For sending replies
- `WHATSAPP_PHONE_NUMBER_ID` — Bot's phone number ID

## Tasks

- [ ] Create Terraform for API Gateway HTTP API with `/webhook` route (GET + POST)
- [ ] Implement webhook verification handler (GET challenge-response)
- [ ] Implement signature validation (HMAC SHA-256)
- [ ] Implement message parser (extract phone, text, media type)
- [ ] Implement WhatsApp client library (send text message)
- [ ] Write tests for signature validation (valid + invalid)
- [ ] Write tests for message parsing (text, audio, image payloads)
