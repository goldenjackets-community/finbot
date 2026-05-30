# Requirements Document

## Introduction

This document defines the requirements for the WhatsApp Webhook Setup feature of FinBot. The feature establishes the ingress point for all WhatsApp messages by deploying an API Gateway HTTP API that receives webhook events from Meta's WhatsApp Cloud API, validates their authenticity via HMAC signature verification, and routes parsed messages to downstream handlers. The system must comply with Meta's webhook protocol including challenge-response verification and the 5-second response time constraint.

## Glossary

- **Webhook_Handler**: The AWS Lambda function (`src/handlers/webhook.py`) that processes incoming HTTP requests from Meta's WhatsApp Cloud API
- **API_Gateway**: The AWS API Gateway HTTP API resource that exposes the `/webhook` route to the internet
- **Signature_Validator**: The component within the Webhook_Handler responsible for verifying the HMAC SHA-256 signature of incoming POST requests
- **Verification_Endpoint**: The GET `/webhook` route that responds to Meta's challenge-response verification handshake
- **Message_Parser**: The component that extracts structured data (sender phone number, message type, message content) from validated webhook payloads
- **Meta_Platform**: Meta's WhatsApp Cloud API infrastructure that sends webhook events and performs verification handshakes
- **Verify_Token**: A shared secret (`WHATSAPP_VERIFY_TOKEN`) used during the GET verification handshake to confirm endpoint ownership
- **App_Secret**: The WhatsApp application secret (`WHATSAPP_APP_SECRET`) used to compute and verify HMAC SHA-256 signatures on incoming payloads

## Requirements

### Requirement 1: Webhook Verification Handshake

**User Story:** As a developer, I want the webhook endpoint to respond to Meta's verification challenge, so that Meta can confirm the endpoint is valid and begin delivering webhook events.

#### Acceptance Criteria

1. WHEN the Verification_Endpoint receives a GET request with `hub.mode` set to "subscribe", `hub.verify_token` matching the configured Verify_Token, and `hub.challenge` present, THE Verification_Endpoint SHALL respond with HTTP 200, Content-Type `text/plain`, and the exact value of `hub.challenge` as the response body within 5 seconds
2. IF the Verification_Endpoint receives a GET request with `hub.verify_token` not matching the configured Verify_Token, THEN THE Verification_Endpoint SHALL respond with HTTP 403 and an empty body
3. IF the Verification_Endpoint receives a GET request without any of the required query parameters (`hub.mode`, `hub.verify_token`, or `hub.challenge`), THEN THE Verification_Endpoint SHALL respond with HTTP 400 and an empty body
4. IF the Verification_Endpoint receives a GET request with `hub.mode` set to a value other than "subscribe" while `hub.verify_token` matches the configured Verify_Token, THEN THE Verification_Endpoint SHALL respond with HTTP 400 and an empty body

### Requirement 2: Webhook Signature Validation

**User Story:** As a developer, I want incoming webhook payloads to be validated against their HMAC SHA-256 signature, so that only authentic messages from Meta are processed.

#### Acceptance Criteria

1. WHEN the Webhook_Handler receives a POST request, THE Signature_Validator SHALL compute an HMAC SHA-256 hash of the raw, unmodified request body bytes using the App_Secret and compare it to the hex digest portion of the `X-Hub-Signature-256` header value after stripping the `sha256=` prefix
2. WHEN the computed signature matches the `X-Hub-Signature-256` header value, THE Signature_Validator SHALL allow the request to proceed to message parsing
3. WHEN the computed signature does not match the `X-Hub-Signature-256` header value, THE Webhook_Handler SHALL respond with HTTP 401 and log the failed validation attempt including the source IP address and timestamp
4. WHEN the `X-Hub-Signature-256` header is missing from a POST request, THE Webhook_Handler SHALL respond with HTTP 401 and log the failed validation attempt including the source IP address and timestamp
5. THE Signature_Validator SHALL use constant-time comparison when comparing signature values to prevent timing attacks
6. IF the App_Secret is unavailable or empty at the time of signature validation, THEN THE Webhook_Handler SHALL respond with HTTP 500 and log an error indicating the missing configuration

### Requirement 3: Message Parsing

**User Story:** As a developer, I want incoming webhook payloads to be parsed into structured message data, so that downstream handlers can process different message types.

#### Acceptance Criteria

1. WHEN a validated payload contains a text message, THE Message_Parser SHALL return a structured dictionary containing the message type as "text", the sender phone number in E.164 format, the message ID, the timestamp as a Unix epoch integer, and the message text body
2. WHEN a validated payload contains a media message (image, audio, or document), THE Message_Parser SHALL return a structured dictionary containing the message type as the specific media type, the sender phone number in E.164 format, the message ID, the timestamp as a Unix epoch integer, and the media ID
3. WHEN a validated payload contains a status update (sent, delivered, or read), THE Message_Parser SHALL return a structured dictionary containing the message type as "status", the status value, the referenced message ID, the recipient phone number in E.164 format, and the timestamp as a Unix epoch integer
4. IF a validated payload contains an unrecognized message type, THEN THE Message_Parser SHALL log the unrecognized type with the full payload for debugging and return None to indicate no actionable message was produced
5. IF a validated payload has a structure that cannot be parsed due to missing or malformed required fields, THEN THE Message_Parser SHALL log the parsing error with the raw payload and return None while the calling handler responds with HTTP 200 to prevent Meta from retrying

### Requirement 4: Response Time Compliance

**User Story:** As a developer, I want the webhook endpoint to respond within Meta's required timeframe, so that Meta does not mark the endpoint as unhealthy or retry deliveries.

#### Acceptance Criteria

1. WHEN the Webhook_Handler receives a POST request, THE Webhook_Handler SHALL return an HTTP 200 response with an empty body within 5 seconds of receiving the request
2. WHEN the Webhook_Handler receives a valid POST request, THE Webhook_Handler SHALL return the HTTP 200 response before initiating any asynchronous downstream processing
3. IF downstream processing encounters an error after the HTTP 200 response has been sent, THEN THE Webhook_Handler SHALL log the error including the request identifier, timestamp, error type, and a reference to the originating payload without affecting the response already sent to Meta_Platform
4. IF the Webhook_Handler encounters an internal error before the HTTP 200 response is sent, THEN THE Webhook_Handler SHALL return an HTTP 500 response within 5 seconds and log the error including the request identifier, timestamp, and error type

### Requirement 5: Infrastructure Deployment

**User Story:** As a developer, I want the webhook infrastructure defined as Terraform code, so that the API Gateway and Lambda function can be deployed reproducibly.

#### Acceptance Criteria

1. THE API_Gateway SHALL be defined as an HTTP API exposing a `/webhook` route accepting both GET and POST HTTP methods
2. THE API_Gateway SHALL integrate with the Webhook_Handler Lambda function using a Lambda proxy integration
3. THE API_Gateway SHALL be deployed in the us-east-1 region
4. THE Webhook_Handler SHALL be configured with environment variables for WHATSAPP_VERIFY_TOKEN, WHATSAPP_APP_SECRET, WHATSAPP_ACCESS_TOKEN, and WHATSAPP_PHONE_NUMBER_ID
5. THE API_Gateway SHALL enforce a throttle limit of 100 requests per second with a burst limit of 50 requests to protect downstream resources
6. THE Webhook_Handler SHALL be configured with the Python 3.12 runtime, a memory allocation of 128 MB, a timeout of 30 seconds, and a maximum concurrency of 10
7. THE Terraform configuration SHALL tag all created resources with `Project=finbot` and `Environment` set via a Terraform variable

### Requirement 6: Logging and Observability

**User Story:** As a developer, I want all webhook activity to be logged with structured context, so that I can debug issues and monitor system health.

#### Acceptance Criteria

1. WHEN the Signature_Validator rejects a request, THE Webhook_Handler SHALL log the event with severity ERROR including the source IP, timestamp, reason for rejection, and request ID
2. WHEN the Message_Parser successfully parses a message, THE Webhook_Handler SHALL log the event with severity INFO including the sender phone number (last 4 digits only), message type, message ID, and request ID
3. WHEN the Webhook_Handler returns a 200 response, THE Webhook_Handler SHALL log the event with severity INFO including the total processing duration in milliseconds and request ID
4. THE Webhook_Handler SHALL use structured JSON logging format for all log entries, where each entry includes at minimum: timestamp in ISO 8601 format, severity level, request ID, and event description
5. WHEN the Webhook_Handler receives a request, THE Webhook_Handler SHALL generate a unique request ID and include it in all log entries produced during that request's processing
