# Design Document

## Overview

This document describes the technical design for the WhatsApp Webhook Setup feature. The system receives webhook events from Meta's WhatsApp Cloud API via an API Gateway HTTP API, validates their authenticity, parses message payloads, and routes them to downstream handlers. The design prioritizes simplicity (functions over classes), security (HMAC validation with constant-time comparison), and Meta compliance (5-second response window).

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Meta WhatsApp Cloud API                   │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                    GET (verification) / POST (messages)
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│              API Gateway HTTP API (/webhook)                      │
│              - Throttle: 100 req/s, burst 50                     │
│              - Region: us-east-1                                  │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                     Lambda Proxy Integration
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│              Lambda: webhook_handler (Python 3.12)                │
│              - Memory: 128 MB                                    │
│              - Timeout: 30s                                       │
│              - Concurrency: 10                                    │
│                                                                   │
│  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  Verification │  │ Signature        │  │ Message          │  │
│  │  Handler      │  │ Validator        │  │ Parser           │  │
│  └──────────────┘  └──────────────────┘  └──────────────────┘  │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                     Parsed message dict
                               │
                               ▼
                    Downstream handlers (future)
                    (registration, expense, etc.)
```

## Components and Interfaces

### 1. `src/handlers/webhook.py` — Lambda Entry Point

The main Lambda handler function. Routes GET/POST requests and orchestrates the processing pipeline.

```python
"""WhatsApp webhook handler for API Gateway Lambda proxy integration."""

import base64
import json
import logging
import os
import time
import uuid
from typing import Any

from src.lib.whatsapp import verify_signature, parse_message

logger = logging.getLogger()


def handler(event: dict[str, Any], context: Any) -> dict[str, Any]:
    """Lambda handler for WhatsApp webhook events.

    Routes GET requests to verification and POST requests
    through signature validation → message parsing → routing.
    """
    request_id = str(uuid.uuid4())
    start_time = time.time()
    http_method = event.get("requestContext", {}).get("http", {}).get("method", "")

    if http_method == "GET":
        return _handle_verification(event, request_id)
    elif http_method == "POST":
        return _handle_message(event, request_id, start_time)
    else:
        return _response(405, "")


def _handle_verification(event: dict[str, Any], request_id: str) -> dict[str, Any]:
    """Handle Meta webhook verification challenge-response."""
    params = event.get("queryStringParameters") or {}
    mode = params.get("hub.mode")
    token = params.get("hub.verify_token")
    challenge = params.get("hub.challenge")

    if not mode or not token or not challenge:
        return _response(400, "")

    if mode != "subscribe":
        return _response(400, "")

    verify_token = os.environ.get("WHATSAPP_VERIFY_TOKEN", "")
    if token != verify_token:
        return _response(403, "")

    return _response(200, challenge, content_type="text/plain")


def _handle_message(event: dict[str, Any], request_id: str, start_time: float) -> dict[str, Any]:
    """Handle incoming webhook POST — validate, parse, route."""
    source_ip = event.get("requestContext", {}).get("http", {}).get("sourceIp", "unknown")
    body = event.get("body", "")
    is_base64 = event.get("isBase64Encoded", False)

    # Signature validation
    app_secret = os.environ.get("WHATSAPP_APP_SECRET", "")
    if not app_secret:
        logger.error(json.dumps({
            "event": "missing_config",
            "detail": "WHATSAPP_APP_SECRET not configured",
            "request_id": request_id,
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
            "severity": "ERROR",
        }))
        return _response(500, "")

    signature_header = (event.get("headers") or {}).get("x-hub-signature-256", "")
    raw_body = base64.b64decode(body) if is_base64 else body.encode("utf-8")

    if not verify_signature(raw_body, signature_header, app_secret):
        logger.error(json.dumps({
            "event": "signature_validation_failed",
            "source_ip": source_ip,
            "request_id": request_id,
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
            "severity": "ERROR",
        }))
        return _response(401, "")

    # Parse message
    payload = json.loads(raw_body)
    message = parse_message(payload)

    if message:
        phone_masked = message.get("phone", "")[-4:]
        logger.info(json.dumps({
            "event": "message_parsed",
            "phone_last4": phone_masked,
            "message_type": message.get("type"),
            "message_id": message.get("message_id"),
            "request_id": request_id,
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
            "severity": "INFO",
        }))
        # TODO: Route to downstream handler based on message type

    duration_ms = (time.time() - start_time) * 1000
    logger.info(json.dumps({
        "event": "request_complete",
        "duration_ms": round(duration_ms, 2),
        "request_id": request_id,
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "severity": "INFO",
    }))

    return _response(200, "")


def _response(status_code: int, body: str, content_type: str = "application/json") -> dict[str, Any]:
    """Build API Gateway proxy response."""
    return {
        "statusCode": status_code,
        "headers": {"Content-Type": content_type},
        "body": body,
    }
```

**Requirement traceability:** Req 1 (verification), Req 2 (signature), Req 3 (parsing), Req 4 (response time), Req 6 (logging)

---

### 2. `src/lib/whatsapp.py` — WhatsApp Utilities

Shared library for signature verification, message parsing, and sending messages.

```python
"""WhatsApp API utilities: signature verification, message parsing, sending."""

import hashlib
import hmac
import json
import logging
from typing import Any

import httpx

logger = logging.getLogger()

# --- Signature Validation (Req 2) ---

def verify_signature(raw_body: bytes, signature_header: str, app_secret: str) -> bool:
    """Verify HMAC SHA-256 signature from Meta webhook.

    Uses constant-time comparison to prevent timing attacks.
    Returns False if header is missing or malformed.
    """
    if not signature_header or not signature_header.startswith("sha256="):
        return False

    expected_sig = signature_header[7:]  # Strip "sha256=" prefix
    computed_sig = hmac.new(
        app_secret.encode("utf-8"),
        raw_body,
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(computed_sig, expected_sig)


# --- Message Parsing (Req 3) ---

def parse_message(payload: dict[str, Any]) -> dict[str, Any] | None:
    """Parse Meta webhook payload into structured message dict.

    Returns None for unrecognized or malformed payloads.
    Handles: text, image, audio, document, status updates.
    """
    try:
        entry = payload.get("entry", [{}])[0]
        changes = entry.get("changes", [{}])[0]
        value = changes.get("value", {})

        # Status updates
        if "statuses" in value:
            status = value["statuses"][0]
            return {
                "type": "status",
                "status": status["status"],
                "message_id": status["id"],
                "phone": status["recipient_id"],
                "timestamp": int(status["timestamp"]),
            }

        # Messages
        if "messages" not in value:
            return None

        msg = value["messages"][0]
        contact = value.get("contacts", [{}])[0]
        phone = contact.get("wa_id", msg.get("from", ""))
        msg_type = msg.get("type", "")
        timestamp = int(msg.get("timestamp", 0))
        message_id = msg.get("id", "")

        if msg_type == "text":
            return {
                "type": "text",
                "phone": phone,
                "message_id": message_id,
                "timestamp": timestamp,
                "text": msg.get("text", {}).get("body", ""),
            }
        elif msg_type in ("image", "audio", "document"):
            media_obj = msg.get(msg_type, {})
            return {
                "type": msg_type,
                "phone": phone,
                "message_id": message_id,
                "timestamp": timestamp,
                "media_id": media_obj.get("id", ""),
            }
        else:
            logger.warning(json.dumps({
                "event": "unrecognized_message_type",
                "message_type": msg_type,
                "payload": payload,
            }))
            return None

    except (KeyError, IndexError, TypeError, ValueError) as e:
        logger.error(json.dumps({
            "event": "message_parse_error",
            "error": str(e),
            "payload": payload,
        }))
        return None


# --- Send Message (outbound) ---

WHATSAPP_API_URL = "https://graph.facebook.com/v18.0/{phone_number_id}/messages"


def send_text_message(phone: str, text: str, access_token: str, phone_number_id: str) -> bool:
    """Send a text message via WhatsApp Cloud API.

    Returns True if the message was accepted by Meta (HTTP 200).
    """
    url = WHATSAPP_API_URL.format(phone_number_id=phone_number_id)
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json",
    }
    payload = {
        "messaging_product": "whatsapp",
        "to": phone,
        "type": "text",
        "text": {"body": text},
    }

    try:
        response = httpx.post(url, headers=headers, json=payload, timeout=10.0)
        return response.status_code == 200
    except httpx.HTTPError as e:
        logger.error(json.dumps({
            "event": "send_message_failed",
            "phone_last4": phone[-4:],
            "error": str(e),
        }))
        return False
```

**Requirement traceability:** Req 2 (verify_signature), Req 3 (parse_message)

---

### 3. `infra/api_gateway.tf` — Terraform Infrastructure

```hcl
# --- Variables ---
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}

variable "whatsapp_verify_token" {
  description = "Token for Meta webhook verification"
  type        = string
  sensitive   = true
}

variable "whatsapp_app_secret" {
  description = "WhatsApp app secret for signature validation"
  type        = string
  sensitive   = true
}

variable "whatsapp_access_token" {
  description = "WhatsApp API access token"
  type        = string
  sensitive   = true
}

variable "whatsapp_phone_number_id" {
  description = "WhatsApp phone number ID"
  type        = string
}

# --- API Gateway HTTP API ---
resource "aws_apigatewayv2_api" "webhook" {
  name          = "finbot-webhook-${var.environment}"
  protocol_type = "HTTP"

  tags = {
    Project     = "finbot"
    Environment = var.environment
  }
}

resource "aws_apigatewayv2_stage" "default" {
  api_id      = aws_apigatewayv2_api.webhook.id
  name        = "$default"
  auto_deploy = true

  default_route_settings {
    throttling_burst_limit = 50
    throttling_rate_limit  = 100
  }

  tags = {
    Project     = "finbot"
    Environment = var.environment
  }
}

resource "aws_apigatewayv2_integration" "webhook_lambda" {
  api_id                 = aws_apigatewayv2_api.webhook.id
  integration_type       = "AWS_PROXY"
  integration_uri        = aws_lambda_function.webhook.invoke_arn
  payload_format_version = "2.0"
}

resource "aws_apigatewayv2_route" "webhook_get" {
  api_id    = aws_apigatewayv2_api.webhook.id
  route_key = "GET /webhook"
  target    = "integrations/${aws_apigatewayv2_integration.webhook_lambda.id}"
}

resource "aws_apigatewayv2_route" "webhook_post" {
  api_id    = aws_apigatewayv2_api.webhook.id
  route_key = "POST /webhook"
  target    = "integrations/${aws_apigatewayv2_integration.webhook_lambda.id}"
}

# --- Lambda Function ---
resource "aws_lambda_function" "webhook" {
  function_name = "finbot-webhook-${var.environment}"
  runtime       = "python3.12"
  handler       = "src.handlers.webhook.handler"
  memory_size   = 128
  timeout       = 30

  filename         = data.archive_file.webhook_zip.output_path
  source_code_hash = data.archive_file.webhook_zip.output_base64sha256
  role             = aws_iam_role.webhook_lambda.arn

  reserved_concurrent_executions = 10

  environment {
    variables = {
      WHATSAPP_VERIFY_TOKEN    = var.whatsapp_verify_token
      WHATSAPP_APP_SECRET      = var.whatsapp_app_secret
      WHATSAPP_ACCESS_TOKEN    = var.whatsapp_access_token
      WHATSAPP_PHONE_NUMBER_ID = var.whatsapp_phone_number_id
    }
  }

  tags = {
    Project     = "finbot"
    Environment = var.environment
  }
}

data "archive_file" "webhook_zip" {
  type        = "zip"
  source_dir  = "${path.module}/../src"
  output_path = "${path.module}/.build/webhook.zip"
}

# --- IAM Role ---
resource "aws_iam_role" "webhook_lambda" {
  name = "finbot-webhook-lambda-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })

  tags = {
    Project     = "finbot"
    Environment = var.environment
  }
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.webhook_lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# --- Lambda Permission for API Gateway ---
resource "aws_lambda_permission" "apigw" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.webhook.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.webhook.execution_arn}/*/*"
}

# --- Outputs ---
output "webhook_url" {
  value = "${aws_apigatewayv2_stage.default.invoke_url}/webhook"
}
```

**Requirement traceability:** Req 5 (all criteria), Req 4 (Lambda timeout supports 5s response)

## Data Models

### Parsed Message Dictionary (returned by `parse_message`)

| Field | Type | Present When | Description |
|-------|------|-------------|-------------|
| `type` | `str` | Always | Message type: `"text"`, `"image"`, `"audio"`, `"document"`, `"status"` |
| `phone` | `str` | Always | Phone number in E.164 format (sender for messages, recipient for statuses) |
| `message_id` | `str` | Always | Unique message ID from Meta |
| `timestamp` | `int` | Always | Unix epoch timestamp |
| `text` | `str` | type == "text" | Message body text |
| `media_id` | `str` | type in (image, audio, document) | Media asset ID for download |
| `status` | `str` | type == "status" | Status value: `"sent"`, `"delivered"`, `"read"` |

### API Gateway Lambda Proxy Event (input)

Standard AWS API Gateway HTTP API v2.0 payload format:
- `event["requestContext"]["http"]["method"]` — HTTP method (GET/POST)
- `event["requestContext"]["http"]["sourceIp"]` — Client IP
- `event["queryStringParameters"]` — Query params (for GET verification)
- `event["headers"]` — Request headers (lowercase keys)
- `event["body"]` — Request body (string, possibly base64-encoded)
- `event["isBase64Encoded"]` — Whether body is base64

### API Gateway Lambda Proxy Response (output)

```python
{
    "statusCode": int,        # HTTP status code
    "headers": dict[str, str], # Response headers
    "body": str,              # Response body
}
```

### Environment Variables

| Variable | Purpose | Used By |
|----------|---------|---------|
| `WHATSAPP_VERIFY_TOKEN` | GET verification handshake | `_handle_verification()` |
| `WHATSAPP_APP_SECRET` | HMAC signature computation | `_handle_message()` |
| `WHATSAPP_ACCESS_TOKEN` | Outbound API calls | `send_text_message()` |
| `WHATSAPP_PHONE_NUMBER_ID` | Bot's phone number ID | `send_text_message()` |

## Error Handling

| Scenario | Response | Logging |
|----------|----------|---------|
| Missing query params on GET | 400 | None (expected for probes) |
| Invalid `hub.mode` on GET | 400 | None |
| Token mismatch on GET | 403 | None |
| Missing `WHATSAPP_APP_SECRET` | 500 | ERROR with request_id |
| Missing/invalid signature header | 401 | ERROR with source_ip, request_id |
| Signature mismatch | 401 | ERROR with source_ip, request_id |
| Unrecognized message type | 200 | WARNING with full payload |
| Malformed payload (parse error) | 200 | ERROR with raw payload |
| `send_text_message` HTTP failure | N/A (async) | ERROR with phone_last4 |
| Unsupported HTTP method | 405 | None |

Key design decisions:
- **Always return 200 for valid signatures** even if parsing fails — prevents Meta from retrying and flooding the endpoint
- **Return 401 early** for signature failures — no payload processing occurs
- **Return 500 only for missing config** — signals deployment issue, not Meta's fault
- **Catch specific exceptions** in `parse_message` — never let an unexpected payload crash the Lambda

## Testing Strategy

### Unit Tests (`tests/lib/test_whatsapp.py`)

| Test | Validates |
|------|-----------|
| `test_verify_signature_valid` | Correct HMAC passes validation |
| `test_verify_signature_invalid` | Wrong signature returns False |
| `test_verify_signature_missing_header` | Empty/None header returns False |
| `test_verify_signature_missing_prefix` | Header without "sha256=" returns False |
| `test_parse_message_text` | Text message extracts all fields correctly |
| `test_parse_message_image` | Image message extracts media_id |
| `test_parse_message_audio` | Audio message extracts media_id |
| `test_parse_message_status` | Status update extracts status value |
| `test_parse_message_unknown_type` | Unrecognized type returns None |
| `test_parse_message_malformed` | Missing fields returns None without raising |

### Integration Tests (`tests/handlers/test_webhook.py`)

| Test | Validates |
|------|-----------|
| `test_get_verification_success` | Valid challenge returns 200 + challenge |
| `test_get_verification_bad_token` | Wrong token returns 403 |
| `test_get_verification_missing_params` | Missing params returns 400 |
| `test_get_verification_wrong_mode` | Non-subscribe mode returns 400 |
| `test_post_valid_signature` | Valid POST returns 200 |
| `test_post_invalid_signature` | Bad signature returns 401 |
| `test_post_missing_signature` | No header returns 401 |
| `test_post_missing_app_secret` | No env var returns 500 |

### Mocking Strategy

- Environment variables: `monkeypatch.setenv()` or `@patch.dict(os.environ, ...)`
- No AWS services to mock for this feature (no DynamoDB/Bedrock calls)
- HTTP calls in `send_text_message`: mock `httpx.post` with `respx` or `unittest.mock`

## Correctness Properties

### Property 1: Signature Integrity
Every POST request MUST pass HMAC validation before any payload processing occurs. No code path bypasses `verify_signature()`.
**Validates: Requirements 2.1, 2.2**

### Property 2: Constant-Time Comparison
`hmac.compare_digest()` is the ONLY comparison function used for signature matching — never `==`.
**Validates: Requirements 2.5**

### Property 3: Idempotent Responses
The handler is stateless. Replaying the same webhook event produces the same response (200 for valid, 401 for invalid).
**Validates: Requirements 4.1**

### Property 4: No Partial Processing
If signature validation fails, zero downstream side effects occur.
**Validates: Requirements 2.3, 2.4**

### Property 5: Graceful Degradation
Malformed payloads that pass signature validation still return 200 — they are logged but never crash the Lambda.
**Validates: Requirements 3.4, 3.5**

### Property 6: PII Protection
Phone numbers are NEVER logged in full. Only last 4 digits appear in any log entry.
**Validates: Requirements 6.2**

### Property 7: Response Time Bound
The handler returns before any async/downstream work. Lambda timeout (30s) is a safety net, but normal execution completes in <100ms.
**Validates: Requirements 4.1, 4.2**

## File Structure

```
src/
├── handlers/
│   └── webhook.py          # Lambda entry point
└── lib/
    └── whatsapp.py         # Signature validation, message parsing, send API

infra/
└── api_gateway.tf          # API Gateway + Lambda + IAM

tests/
├── handlers/
│   └── test_webhook.py     # Handler integration tests
└── lib/
    └── test_whatsapp.py    # Unit tests for verify_signature, parse_message
```

## Requirement Traceability

| Requirement | Components |
|-------------|-----------|
| Req 1: Verification Handshake | `webhook.py::_handle_verification()` |
| Req 2: Signature Validation | `whatsapp.py::verify_signature()`, `webhook.py::_handle_message()` |
| Req 3: Message Parsing | `whatsapp.py::parse_message()` |
| Req 4: Response Time | `webhook.py` (returns 200 before downstream), Lambda timeout 30s |
| Req 5: Infrastructure | `infra/api_gateway.tf` |
| Req 6: Logging | `webhook.py` (structured JSON logs with request_id throughout) |
