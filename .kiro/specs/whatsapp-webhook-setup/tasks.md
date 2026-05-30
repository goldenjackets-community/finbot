# Implementation Plan: WhatsApp Webhook Setup

## Overview

Implement the WhatsApp webhook ingress point for FinBot. The plan covers: Terraform infrastructure for API Gateway + Lambda, the signature validation and message parsing library, the Lambda handler, and tests. Each task builds incrementally so there is no orphaned code.

## Tasks

- [ ] 1. Set up project structure and dependencies
  - [ ] 1.1 Create project configuration files
    - Create `requirements.txt` with runtime dependencies (`httpx`)
    - Create `requirements-dev.txt` with test dependencies (`pytest`, `hypothesis`, `respx`)
    - Add `src/__init__.py`, `src/handlers/__init__.py`, `src/lib/__init__.py` package markers
    - Add `tests/__init__.py`, `tests/handlers/__init__.py`, `tests/lib/__init__.py` package markers
    - _Requirements: 5.6_

- [ ] 2. Implement WhatsApp utilities library
  - [ ] 2.1 Implement `verify_signature` in `src/lib/whatsapp.py`
    - Compute HMAC SHA-256 of raw body bytes using the app secret
    - Strip `sha256=` prefix from the header value
    - Use `hmac.compare_digest()` for constant-time comparison
    - Return `False` for missing or malformed signature headers
    - _Requirements: 2.1, 2.2, 2.4, 2.5_

  - [ ] 2.2 Implement `parse_message` in `src/lib/whatsapp.py`
    - Parse text messages extracting phone, message_id, timestamp, text body
    - Parse media messages (image, audio, document) extracting media_id
    - Parse status updates extracting status value, message_id, recipient phone
    - Return `None` for unrecognized types or malformed payloads
    - Log warnings for unrecognized types, errors for parse failures
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ] 2.3 Implement `send_text_message` in `src/lib/whatsapp.py`
    - Send outbound text message via WhatsApp Cloud API using `httpx`
    - Return `True` on HTTP 200, `False` on failure
    - Log errors with phone last 4 digits only
    - _Requirements: 6.2_

  - [ ]* 2.4 Write property test: Signature Integrity
    - **Property 1: Signature Integrity** — Any body+secret pair that produces a valid HMAC must pass verification; any mutation to body, secret, or signature must fail
    - **Validates: Requirements 2.1, 2.2**

  - [ ]* 2.5 Write property test: Constant-Time Comparison
    - **Property 2: Constant-Time Comparison** — Verify that `hmac.compare_digest` is the only comparison used (structural/code inspection test)
    - **Validates: Requirements 2.5**

  - [ ]* 2.6 Write property test: Graceful Degradation
    - **Property 5: Graceful Degradation** — For any arbitrary dict payload, `parse_message` never raises an exception; it returns a valid dict or None
    - **Validates: Requirements 3.4, 3.5**

  - [ ]* 2.7 Write unit tests for `verify_signature` and `parse_message`
    - Test valid signature passes
    - Test invalid signature returns False
    - Test missing/malformed header returns False
    - Test text, image, audio, document, and status parsing
    - Test unrecognized type returns None
    - Test malformed payload returns None without raising
    - _Requirements: 2.1, 2.2, 2.4, 2.5, 3.1, 3.2, 3.3, 3.4, 3.5_

- [ ] 3. Checkpoint - Ensure library tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Implement Lambda webhook handler
  - [ ] 4.1 Implement `_handle_verification` in `src/handlers/webhook.py`
    - Validate presence of `hub.mode`, `hub.verify_token`, `hub.challenge` query params
    - Check `hub.mode` equals "subscribe"
    - Compare `hub.verify_token` against `WHATSAPP_VERIFY_TOKEN` env var
    - Return 200 with challenge as plain text on success
    - Return 400 for missing params or wrong mode, 403 for token mismatch
    - _Requirements: 1.1, 1.2, 1.3, 1.4_

  - [ ] 4.2 Implement `_handle_message` in `src/handlers/webhook.py`
    - Read `WHATSAPP_APP_SECRET` from environment; return 500 if missing
    - Extract raw body (handle base64 encoding)
    - Call `verify_signature`; return 401 on failure with structured error log
    - Parse payload with `parse_message`; log parsed message info (phone last 4 only)
    - Return 200 with empty body after processing
    - Log request duration in milliseconds
    - _Requirements: 2.1, 2.3, 2.4, 2.6, 4.1, 4.2, 6.1, 6.2, 6.3, 6.4, 6.5_

  - [ ] 4.3 Implement `handler` entry point and `_response` helper in `src/handlers/webhook.py`
    - Route GET to `_handle_verification`, POST to `_handle_message`
    - Return 405 for unsupported methods
    - Generate unique request_id per invocation
    - Build API Gateway proxy response format
    - _Requirements: 4.1, 6.5_

  - [ ]* 4.4 Write property test: No Partial Processing
    - **Property 4: No Partial Processing** — When signature validation fails, the handler returns 401 and `parse_message` is never called
    - **Validates: Requirements 2.3, 2.4**

  - [ ]* 4.5 Write property test: PII Protection
    - **Property 6: PII Protection** — For any phone number in a parsed message, no log entry contains more than the last 4 digits
    - **Validates: Requirements 6.2**

  - [ ]* 4.6 Write integration tests for webhook handler
    - Test GET verification success (valid token + challenge returns 200)
    - Test GET verification bad token returns 403
    - Test GET verification missing params returns 400
    - Test GET verification wrong mode returns 400
    - Test POST valid signature returns 200
    - Test POST invalid signature returns 401
    - Test POST missing signature header returns 401
    - Test POST missing app secret env var returns 500
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 2.1, 2.3, 2.4, 2.6_

- [ ] 5. Checkpoint - Ensure all handler tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Implement Terraform infrastructure
  - [ ] 6.1 Create `infra/api_gateway.tf` with API Gateway and Lambda resources
    - Define HTTP API with `/webhook` route for GET and POST
    - Define Lambda function with Python 3.12, 128 MB, 30s timeout, concurrency 10
    - Define Lambda proxy integration with payload format 2.0
    - Configure throttle: 100 req/s rate, 50 burst
    - Set environment variables for all WhatsApp secrets
    - Define IAM role with `AWSLambdaBasicExecutionRole`
    - Define Lambda permission for API Gateway invocation
    - Tag all resources with `Project=finbot` and `Environment` variable
    - Output the webhook URL
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7_

- [ ] 7. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties from the design document
- Unit tests validate specific examples and edge cases
- The design uses Python 3.12 with pytest for testing and Hypothesis for property-based tests

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["2.1", "2.2", "2.3"] },
    { "id": 2, "tasks": ["2.4", "2.5", "2.6", "2.7"] },
    { "id": 3, "tasks": ["4.1", "4.2", "4.3"] },
    { "id": 4, "tasks": ["4.4", "4.5", "4.6"] },
    { "id": 5, "tasks": ["6.1"] }
  ]
}
```
