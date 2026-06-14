# Document Information, Standards, Authentication Foundation & Shared Contracts

---

# 1. Document Information

| Field           | Value                  |
| --------------- | ---------------------- |
| Product         | OpsPilot AI Lite       |
| Version         | 1.0                    |
| Status          | Approved               |
| API Style       | REST                   |
| API Version     | v1                     |
| Base Path       | `/api/v1`              |
| Transport       | HTTPS Only             |
| Authentication  | Google OAuth2 + JWT    |
| Session Storage | HttpOnly Secure Cookie |
| Response Format | JSON                   |
| OpenAPI Source  | FastAPI Generated      |
| Swagger UI      | `/docs`                |
| ReDoc           | `/redoc`               |

---

# 2. API Design Standards

## API Principles

The API must:

* Be RESTful
* Be resource-oriented
* Be versioned
* Use JSON payloads
* Use consistent response envelopes
* Use machine-readable error codes
* Support idempotency for write operations
* Include correlation IDs for traceability

---

## Base URL

### Production

```http
https://api.opspilot.akirasi.in/api/v1
```

### Local

```http
http://localhost:8000/api/v1
```

---

## Content Type

All requests and responses:

```http
Content-Type: application/json
```

---

## Character Encoding

```http
UTF-8
```

---

# 3. Authentication Model

---

## Authentication Provider

Google OAuth2

---

## Session Strategy

```text
Google OAuth2
        ↓
Backend Callback
        ↓
JWT Generated
        ↓
HttpOnly Cookie
        ↓
Authenticated Requests
```

---

## Authentication Mechanism

Authentication is performed using:

```text
JWT stored inside HttpOnly Secure Cookie
```

Frontend never stores JWT.

Frontend never accesses JWT.

Frontend never uses localStorage for authentication.

---

## Authorization Strategy

Resource ownership enforcement.

Every protected resource must belong to the authenticated user.

---

## User Permissions

Authenticated users may:

```text
Start Voice Session
Create Tickets
View Own Tickets
View Own Workflow Runs
View Own Voice Sessions
Receive WebSocket Events
```

---

## User Restrictions

Authenticated users may NOT:

```text
Access Another User's Tickets
Access Another User's Workflow Runs
Access Another User's Voice Sessions
Access Internal APIs
```

---

# 4. Common Headers

---

## Request Headers

### Required

```http
Content-Type: application/json
```

---

### Optional

```http
X-Correlation-ID
```

Example:

```http
X-Correlation-ID: 7d8c1d7d-fd4a-4db9-b0d8-5b3f08c91a32
```

---

### Optional (Write Operations)

```http
Idempotency-Key
```

Example:

```http
Idempotency-Key: f50ec0b7-f960-4006-b57a-df3a57f46c4f
```

---

## Response Headers

Every response must include:

```http
X-Correlation-ID
```

Example:

```http
HTTP/1.1 200 OK

X-Correlation-ID: 7d8c1d7d-fd4a-4db9-b0d8-5b3f08c91a32
```

---

# 5. Correlation ID Contract

---

## Purpose

Provide end-to-end request tracing.

---

## Generation Rules

### Incoming Header Exists

Use incoming value.

---

### Incoming Header Missing

Backend generates:

```text
UUID v4
```

Example:

```text
7d8c1d7d-fd4a-4db9-b0d8-5b3f08c91a32
```

---

## Propagation

Correlation ID must travel through:

```text
Browser
    ↓
FastAPI
    ↓
LiveKit Session Creation
    ↓
Ticket Creation
    ↓
Workflow Trigger
    ↓
n8n
    ↓
Workflow Callback
```

---

## Logging Requirements

Every application log entry must contain:

```text
timestamp
level
correlation_id
user_id
event
message
```

---

# 6. Response Envelope Standard

All API responses must follow a consistent structure.

---

## Success Response

```json
{
  "success": true,
  "correlation_id": "7d8c1d7d-fd4a-4db9-b0d8-5b3f08c91a32",
  "data": {}
}
```

---

## Error Response

```json
{
  "success": false,
  "correlation_id": "7d8c1d7d-fd4a-4db9-b0d8-5b3f08c91a32",
  "error": {
    "code": "TICKET_NOT_FOUND",
    "message": "Ticket not found"
  }
}
```

---

# 7. Error Contract

---

## Error Structure

```json
{
  "success": false,
  "correlation_id": "uuid",
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable message"
  }
}
```

---

## Standard Error Codes

### Authentication

```text
UNAUTHORIZED
SESSION_EXPIRED
INVALID_TOKEN
OAUTH_FAILED
```

---

### Authorization

```text
FORBIDDEN
RESOURCE_OWNERSHIP_VIOLATION
```

---

### Voice

```text
VOICE_SESSION_NOT_FOUND
VOICE_SESSION_TIMEOUT
TRANSCRIPT_EMPTY
VOICE_PROCESSING_FAILED
```

---

### Ticket

```text
TICKET_NOT_FOUND
INVALID_TICKET_DATA
TICKET_CREATION_FAILED
```

---

### Workflow

```text
WORKFLOW_FAILED
WORKFLOW_RUN_NOT_FOUND
WEBHOOK_DELIVERY_FAILED
```

---

### Validation

```text
VALIDATION_ERROR
INVALID_REQUEST
MISSING_REQUIRED_FIELD
```

---

### Infrastructure

```text
DATABASE_ERROR
INTERNAL_SERVER_ERROR
SERVICE_UNAVAILABLE
```

---

## HTTP Status Mapping

| HTTP Status | Error Code Examples   |
| ----------- | --------------------- |
| 400         | VALIDATION_ERROR      |
| 401         | UNAUTHORIZED          |
| 403         | FORBIDDEN             |
| 404         | TICKET_NOT_FOUND      |
| 409         | DUPLICATE_REQUEST     |
| 429         | RATE_LIMIT_EXCEEDED   |
| 500         | INTERNAL_SERVER_ERROR |
| 503         | SERVICE_UNAVAILABLE   |

---

# 8. Rate Limiting Contract

Rate limiting is enforced per authenticated user.

---

## Authentication Endpoints

```text
10 requests/minute
```

Endpoints:

```http
/auth/*
```

---

## Voice Endpoints

```text
5 requests/minute
```

Endpoints:

```http
/voice/*
```

---

## Ticket Endpoints

```text
60 requests/minute
```

Endpoints:

```http
/tickets/*
```

---

## Workflow Endpoints

```text
60 requests/minute
```

Endpoints:

```http
/workflow-runs/*
```

---

## Rate Limit Response

HTTP Status:

```http
429 Too Many Requests
```

Response:

```json
{
  "success": false,
  "correlation_id": "uuid",
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded"
  }
}
```

---

# 9. Enum Catalog

---

# Ticket Status

```text
OPEN
IN_PROGRESS
COMPLETED
FAILED
```

---

# Ticket Priority

```text
LOW
MEDIUM
HIGH
CRITICAL
```

---

# Ticket Source

```text
VOICE
```

---

# Workflow Status

```text
PENDING
RUNNING
COMPLETED
FAILED
```

---

# Workflow Trigger Event

```text
TICKET_CREATED
```

---

# Workflow Event Types

```text
WORKFLOW_STARTED
EMAIL_SENDING
EMAIL_SENT
WORKFLOW_COMPLETED
WORKFLOW_FAILED
```

---

# Provider Types

```text
GOOGLE
```

---

# Authentication Provider

```text
google
```

---

# Sort Direction

Reserved for future use.

```text
ASC
DESC
```

---

# Supported Search Fields

Tickets can be searched by:

```text
ticket_number
title
description
```

---

# Supported Filters

Ticket Status

```text
OPEN
IN_PROGRESS
COMPLETED
FAILED
```

Ticket Priority

```text
LOW
MEDIUM
HIGH
CRITICAL
```

---

# Idempotent Operations

The following endpoints require idempotency support:

```http
POST /voice/sessions
POST /voice/sessions/{session_id}/complete
```

---

# Authentication APIs, Voice APIs & Ticket APIs

---

# 10. Authentication APIs

---

# 10.1 Google Login

## Endpoint

```http
GET /api/v1/auth/login/google
```

---

## Purpose

Redirect user to Google OAuth consent screen.

---

## Authentication

Not Required

---

## Request Body

None

---

## Response

```http
302 Redirect
```

Redirects to Google OAuth.

---

# 10.2 Google OAuth Callback

## Endpoint

```http
GET /api/v1/auth/callback/google
```

---

## Purpose

Handle OAuth callback from Google.

---

## Authentication

Not Required

---

## Query Parameters

| Field | Type   | Required |
| ----- | ------ | -------- |
| code  | string | Yes      |
| state | string | Yes      |

---

## Processing Rules

### Existing User

```text
Find user
Generate JWT
Set Cookie
Redirect Dashboard
```

---

### New User

```text
Create User
Generate JWT
Set Cookie
Redirect Dashboard
```

---

## User Creation Rules

Persist:

```json
{
  "email": "user@example.com",
  "full_name": "John Doe",
  "avatar_url": "https://..."
}
```

---

## JWT Claims

```json
{
  "sub": "user_uuid",
  "email": "user@example.com",
  "provider": "google",
  "exp": 9999999999
}
```

---

## Cookie Configuration

```text
HttpOnly
Secure
SameSite=Lax
Max-Age=86400
```

---

## Response

```http
302 Redirect
Location: /
```

---

# 10.3 Current User

## Endpoint

```http
GET /api/v1/auth/me
```

---

## Authentication

Required

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "id": "uuid",
    "email": "user@example.com",
    "full_name": "John Doe",
    "avatar_url": "https://..."
  }
}
```

---

## Error Responses

```text
401 Unauthorized
```

---

# 10.4 Logout

## Endpoint

```http
POST /api/v1/auth/logout
```

---

## Authentication

Required

---

## Purpose

Invalidate user session.

---

## Processing

```text
Clear Cookie
Return Success
```

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "message": "Logged out successfully"
  }
}
```

---

# 11. Voice APIs

---

# Voice Session Lifecycle

```text
Create Session
      ↓
Connect LiveKit
      ↓
Record Audio
      ↓
Stop Recording
      ↓
Submit Transcript
      ↓
AI Extraction
      ↓
Ticket Creation
      ↓
Workflow Trigger
```

---

# 11.1 Create Voice Session

## Endpoint

```http
POST /api/v1/voice/sessions
```

---

## Authentication

Required

---

## Idempotency

Required

Header:

```http
Idempotency-Key
```

---

## Request

```json
{}
```

---

## Processing Rules

### Create

1. Voice Session Record
2. LiveKit Room
3. LiveKit Token

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "session_id": "uuid",
    "livekit_room_id": "room_uuid",
    "livekit_token": "jwt_token",
    "expires_in": 300
  }
}
```

---

## Validation

Maximum active sessions per user:

```text
1
```

---

## Errors

```text
VOICE_SESSION_CREATION_FAILED
RATE_LIMIT_EXCEEDED
```

---

# 11.2 Complete Voice Session

## Endpoint

```http
POST /api/v1/voice/sessions/{session_id}/complete
```

---

## Authentication

Required

---

## Idempotency

Required

---

## Purpose

Submit final transcript and trigger processing pipeline.

---

## Path Parameters

| Field      | Type |
| ---------- | ---- |
| session_id | UUID |

---

## Request

```json
{
  "transcript": "The printer in finance room is offline and requires urgent maintenance."
}
```

---

## Validation Rules

### Transcript

Required

---

### Minimum Length

```text
10 characters
```

---

### Maximum Length

```text
5000 characters
```

---

## Processing Flow

```text
Store Transcript
      ↓
Pipecat Extraction
      ↓
Create Ticket
      ↓
Create Workflow Run
      ↓
Trigger n8n
      ↓
Return Result
```

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "session_id": "uuid",
    "ticket_id": "uuid",
    "ticket_number": "TKT-000001",
    "workflow_run_id": "uuid",
    "title": "Finance Printer Offline",
    "description": "Printer unavailable in finance room",
    "priority": "HIGH",
    "confidence": 0.94
  }
}
```

---

## Errors

```text
VOICE_SESSION_NOT_FOUND
TRANSCRIPT_EMPTY
VOICE_PROCESSING_FAILED
```

---

# 11.3 Get Voice Session

## Endpoint

```http
GET /api/v1/voice/sessions/{session_id}
```

---

## Authentication

Required

---

## Ownership Validation

Required

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "id": "uuid",
    "transcript": "The printer in finance room is offline.",
    "ticket_id": "uuid",
    "created_at": "2026-06-14T10:00:00Z"
  }
}
```

---

# 11.4 List Voice Sessions

## Endpoint

```http
GET /api/v1/voice/sessions
```

---

## Authentication

Required

---

## Query Parameters

| Field     | Default |
| --------- | ------- |
| page      | 1       |
| page_size | 20      |

---

## Maximum Page Size

```text
100
```

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "items": [],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 50,
      "total_pages": 3
    }
  }
}
```

---

# 12. Ticket APIs

---

# Ticket Lifecycle

```text
Voice Request
      ↓
OPEN
      ↓
Workflow Running
      ↓
IN_PROGRESS
      ↓
Workflow Completed
      ↓
COMPLETED
```

Workflow failures do NOT update ticket status.

---

# 12.1 Get Ticket

## Endpoint

```http
GET /api/v1/tickets/{ticket_id}
```

---

## Authentication

Required

---

## Ownership Validation

Required

---

## Path Parameters

| Field     | Type |
| --------- | ---- |
| ticket_id | UUID |

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "id": "uuid",
    "ticket_number": "TKT-000001",
    "title": "Finance Printer Offline",
    "description": "Printer unavailable in finance room",
    "priority": "HIGH",
    "status": "OPEN",
    "source": "VOICE",
    "created_at": "2026-06-14T10:00:00Z"
  }
}
```

---

## Errors

```text
TICKET_NOT_FOUND
FORBIDDEN
```

---

# 12.2 List Tickets

## Endpoint

```http
GET /api/v1/tickets
```

---

## Authentication

Required

---

## Query Parameters

### Pagination

| Parameter | Default |
| --------- | ------- |
| page      | 1       |
| page_size | 20      |

---

### Filtering

| Parameter | Values                               |
| --------- | ------------------------------------ |
| status    | OPEN, IN_PROGRESS, COMPLETED, FAILED |
| priority  | LOW, MEDIUM, HIGH, CRITICAL          |

---

### Search

```http
GET /tickets?search=printer
```

Searches:

```text
ticket_number
title
description
```

---

### Sorting

Fixed:

```sql
ORDER BY created_at DESC
```

No custom sorting supported.

---

## Example

```http
GET /api/v1/tickets?
status=OPEN&
priority=HIGH&
search=printer&
page=1&
page_size=20
```

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "items": [
      {
        "id": "uuid",
        "ticket_number": "TKT-000001",
        "title": "Finance Printer Offline",
        "priority": "HIGH",
        "status": "OPEN",
        "created_at": "2026-06-14T10:00:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 50,
      "total_pages": 3
    }
  }
}
```

---

# 12.3 Get Ticket Workflow Runs

## Endpoint

```http
GET /api/v1/tickets/{ticket_id}/workflow-runs
```

---

## Authentication

Required

---

## Ownership Validation

Required

---

## Purpose

Retrieve all workflow runs associated with a ticket.

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": [
    {
      "workflow_run_id": "uuid",
      "status": "COMPLETED",
      "retry_count": 0,
      "created_at": "2026-06-14T10:00:00Z"
    }
  ]
}
```

---

# 12.4 Get Ticket Timeline

## Endpoint

```http
GET /api/v1/tickets/{ticket_id}/timeline
```

---

## Authentication

Required

---

## Purpose

Provide dashboard activity timeline.

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": [
    {
      "event": "TICKET_CREATED",
      "timestamp": "2026-06-14T10:00:00Z"
    },
    {
      "event": "WORKFLOW_STARTED",
      "timestamp": "2026-06-14T10:00:02Z"
    },
    {
      "event": "EMAIL_SENT",
      "timestamp": "2026-06-14T10:00:05Z"
    }
  ]
}
```

---

# Workflow APIs, WebSocket Contract, Internal APIs, n8n Contract & LiveKit Contract

---

# 13. Workflow APIs

---

# Workflow Lifecycle

```text
Ticket Created
      ↓
Workflow Run Created
      ↓
PENDING
      ↓
RUNNING
      ↓
COMPLETED / FAILED
```

---

# 13.1 Get Workflow Run

## Endpoint

```http
GET /api/v1/workflow-runs/{workflow_run_id}
```

---

## Authentication

Required

---

## Ownership Validation

Required through:

```text
workflow_run
    ↓
ticket
    ↓
user
```

---

## Path Parameters

| Parameter       | Type |
| --------------- | ---- |
| workflow_run_id | UUID |

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "id": "uuid",
    "ticket_id": "uuid",
    "status": "COMPLETED",
    "trigger_event": "TICKET_CREATED",
    "retry_count": 0,
    "created_at": "2026-06-14T10:00:00Z"
  }
}
```

---

## Errors

```text
WORKFLOW_RUN_NOT_FOUND
FORBIDDEN
```

---

# 13.2 List Workflow Runs

## Endpoint

```http
GET /api/v1/workflow-runs
```

---

## Authentication

Required

---

## Query Parameters

| Parameter | Default |
| --------- | ------- |
| page      | 1       |
| page_size | 20      |

---

## Maximum Page Size

```text
100
```

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "items": [
      {
        "id": "uuid",
        "ticket_id": "uuid",
        "status": "COMPLETED",
        "created_at": "2026-06-14T10:00:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 1,
      "total_pages": 1
    }
  }
}
```

---

# 13.3 Get Workflow Events

## Endpoint

```http
GET /api/v1/workflow-runs/{workflow_run_id}/events
```

---

## Authentication

Required

---

## Purpose

Retrieve workflow execution history.

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": [
    {
      "event": "WORKFLOW_STARTED",
      "timestamp": "2026-06-14T10:00:00Z"
    },
    {
      "event": "EMAIL_SENDING",
      "timestamp": "2026-06-14T10:00:01Z"
    },
    {
      "event": "EMAIL_SENT",
      "timestamp": "2026-06-14T10:00:03Z"
    },
    {
      "event": "WORKFLOW_COMPLETED",
      "timestamp": "2026-06-14T10:00:04Z"
    }
  ]
}
```

---

# 14. WebSocket Contract

---

# Endpoint

```http
GET /api/v1/ws
```

---

# Purpose

Provide realtime dashboard updates.

---

# Authentication

Cookie-based authentication.

JWT validated during websocket handshake.

---

# Connection Flow

```text
Browser
    ↓
WebSocket Upgrade
    ↓
JWT Validation
    ↓
Connection Established
```

---

# Authorization

Users only receive events related to their own resources.

No broadcasting.

---

# Connection Response

```json
{
  "event": "CONNECTED",
  "correlation_id": "uuid",
  "payload": {
    "user_id": "uuid"
  }
}
```

---

# Event Types

---

## TRANSCRIPT_UPDATED

### Payload

```json
{
  "event": "TRANSCRIPT_UPDATED",
  "correlation_id": "uuid",
  "payload": {
    "session_id": "uuid",
    "transcript": "The printer in finance room..."
  }
}
```

---

## TICKET_CREATED

### Payload

```json
{
  "event": "TICKET_CREATED",
  "correlation_id": "uuid",
  "payload": {
    "ticket_id": "uuid",
    "ticket_number": "TKT-000001",
    "status": "OPEN"
  }
}
```

---

## WORKFLOW_UPDATED

### Payload

```json
{
  "event": "WORKFLOW_UPDATED",
  "correlation_id": "uuid",
  "payload": {
    "workflow_run_id": "uuid",
    "status": "RUNNING"
  }
}
```

---

## WORKFLOW_EVENT

### Payload

```json
{
  "event": "WORKFLOW_EVENT",
  "correlation_id": "uuid",
  "payload": {
    "workflow_run_id": "uuid",
    "event_type": "EMAIL_SENT",
    "timestamp": "2026-06-14T10:00:00Z"
  }
}
```

---

## ERROR

### Payload

```json
{
  "event": "ERROR",
  "correlation_id": "uuid",
  "payload": {
    "code": "WORKFLOW_FAILED",
    "message": "Workflow execution failed"
  }
}
```

---

# Reconnection Strategy

Frontend reconnects automatically.

Recommended retry:

```text
1 sec
2 sec
5 sec
10 sec
```

---

# Event Replay

Not Supported.

Page refresh restores state via REST APIs.

---

# 15. Internal APIs

---

# Base Namespace

```http
/api/v1/internal/*
```

---

# Security

Required Header:

```http
X-Internal-API-Key
```

---

# Authentication Failure

```http
401 Unauthorized
```

---

# 15.1 Pipecat Extraction Callback

## Endpoint

```http
POST /api/v1/internal/pipecat/extraction
```

---

## Purpose

Receive AI extraction results.

---

## Request

```json
{
  "correlation_id": "uuid",
  "session_id": "uuid",
  "title": "Finance Printer Offline",
  "description": "Printer unavailable in finance room",
  "priority": "HIGH",
  "confidence": 0.94
}
```

---

## Validation

### Title

Required

Maximum:

```text
200 characters
```

---

### Description

Required

Maximum:

```text
5000 characters
```

---

### Priority

Allowed:

```text
LOW
MEDIUM
HIGH
CRITICAL
```

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "accepted": true
  }
}
```

---

# 15.2 Workflow Status Callback

## Endpoint

```http
POST /api/v1/internal/workflows/status
```

---

## Purpose

Receive workflow completion updates from n8n.

---

## Request

```json
{
  "correlation_id": "uuid",
  "workflow_run_id": "uuid",
  "status": "COMPLETED",
  "message": "Workflow completed successfully"
}
```

---

## Allowed Status Values

```text
COMPLETED
FAILED
```

---

## Processing Rules

### COMPLETED

```text
workflow_run.status = COMPLETED
ticket.status = COMPLETED
create workflow event
emit websocket update
```

---

### FAILED

```text
workflow_run.status = FAILED
ticket remains OPEN
create workflow event
emit websocket update
```

---

## Response

```json
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "updated": true
  }
}
```

---

# 16. n8n Integration Contract

---

# Trigger Type

Webhook

---

# n8n Endpoint

Configured via:

```env
N8N_WEBHOOK_URL=
```

---

# Trigger Event

```text
TICKET_CREATED
```

---

# Outbound Payload

```json
{
  "correlation_id": "uuid",
  "workflow_run_id": "uuid",
  "ticket_id": "uuid",
  "ticket_number": "TKT-000001",
  "user_id": "uuid",
  "user_email": "user@example.com",
  "title": "Finance Printer Offline",
  "description": "Printer unavailable in finance room",
  "priority": "HIGH",
  "event": "TICKET_CREATED",
  "created_at": "2026-06-14T10:00:00Z"
}
```

---

# Retry Policy

Attempt 1

```text
Immediate
```

Attempt 2

```text
5 Seconds
```

Attempt 3

```text
15 Seconds
```

---

# Final Failure

```text
workflow_run.status = FAILED
```

Store:

```text
error_payload
retry_count
timestamp
```

---

# Expected n8n Response

```json
{
  "accepted": true
}
```

---

# Email Recipients

Configured as:

```text
Authenticated User
Support Mailbox
```

---

# Support Mailbox

Configured via:

```env
SUPPORT_EMAIL=
```

Example:

```env
SUPPORT_EMAIL=youremail@gmail.com
```

---

# 17. LiveKit Integration Contract

---

# Purpose

Provide realtime audio transport.

---

# Room Strategy

```text
1 Voice Session
=
1 LiveKit Room
```

---

# Room Naming Convention

```text
voice-{session_uuid}
```

Example:

```text
voice-a6e95a58-b5dd-49c7-b8db-e2ec4cb4e4a8
```

---

# Room Lifecycle

```text
Create Session
      ↓
Create Room
      ↓
Connect User
      ↓
Transmit Audio
      ↓
Session Complete
      ↓
Delete Room
```

---

# Token Generation

Generated by backend.

Returned by:

```http
POST /api/v1/voice/sessions
```

---

# Token Expiry

```text
5 Minutes
```

---

# Session Timeout

```text
5 Minutes
```

---

# LiveKit Response Contract

```json
{
  "session_id": "uuid",
  "livekit_room_id": "voice-session-id",
  "livekit_token": "jwt_token",
  "expires_in": 300
}
```

---

# Failure Handling

---

## Room Creation Failure

Error:

```text
VOICE_SESSION_CREATION_FAILED
```

---

## Token Generation Failure

Error:

```text
VOICE_SESSION_CREATION_FAILED
```

---

## Connection Failure

Error:

```text
VOICE_PROCESSING_FAILED
```

---

# Pagination, Search & Filtering, Health APIs, Database Constraints, Validation Rules, Correlation IDs & Idempotency

---

# 18. Pagination Contract

---

## Supported Resources

Pagination applies to:

```http id="t1v5c9"
GET /api/v1/tickets
GET /api/v1/workflow-runs
GET /api/v1/voice/sessions
```

---

## Pagination Strategy

Offset-based pagination.

---

## Query Parameters

| Parameter | Type    | Default | Required |
| --------- | ------- | ------- | -------- |
| page      | integer | 1       | No       |
| page_size | integer | 20      | No       |

---

## Constraints

### page

Minimum:

```text id="8g8a0f"
1
```

---

### page_size

Minimum:

```text id="lkl0j9"
1
```

Maximum:

```text id="9s8t9x"
100
```

---

## Example Request

```http id="ck1l3u"
GET /api/v1/tickets?page=2&page_size=20
```

---

## Pagination Response

```json id="x6z4eq"
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "items": [],
    "pagination": {
      "page": 2,
      "page_size": 20,
      "total": 125,
      "total_pages": 7
    }
  }
}
```

---

## Validation Errors

```text id="wmzjfd"
VALIDATION_ERROR
```

Examples:

```text id="b52j5r"
page < 1
page_size > 100
page_size < 1
```

---

# 19. Search & Filtering Contract

---

# Ticket Search

Supported endpoint:

```http id="a4r0pv"
GET /api/v1/tickets
```

---

## Search Parameter

```http id="sytr8m"
GET /api/v1/tickets?search=printer
```

---

## Search Strategy

Case-insensitive partial matching.

---

## Searchable Fields

```text id="92b2e6"
ticket_number
title
description
```

---

## Examples

Search:

```text id="j3mfxm"
printer
```

Matches:

```text id="0w9vm9"
Finance Printer Offline
Printer Maintenance Required
```

---

# Ticket Filtering

---

## Status Filter

```http id="4q9ebf"
GET /api/v1/tickets?status=OPEN
```

Allowed Values:

```text id="pkxmbx"
OPEN
IN_PROGRESS
COMPLETED
FAILED
```

---

## Priority Filter

```http id="a5n1d8"
GET /api/v1/tickets?priority=HIGH
```

Allowed Values:

```text id="zq5rbi"
LOW
MEDIUM
HIGH
CRITICAL
```

---

## Combined Query

```http id="l2f9i8"
GET /api/v1/tickets?
status=OPEN&
priority=HIGH&
search=printer
```

---

## Sorting

Sorting is fixed.

```sql id="yt1f0t"
ORDER BY created_at DESC
```

Custom sorting not supported.

---

# 20. Health APIs

---

# 20.1 Application Health

## Endpoint

```http id="tn9g2l"
GET /health
```

---

## Authentication

Not Required

---

## Purpose

Infrastructure monitoring.

Load balancer health checks.

Deployment validation.

---

## Dependency Scope

Only validates:

```text id="gxr7sk"
Database Connectivity
```

---

## Healthy Response

HTTP:

```http id="9umc6g"
200 OK
```

Response:

```json id="vyj0xr"
{
  "status": "healthy",
  "services": {
    "database": "healthy"
  },
  "timestamp": "2026-06-14T10:00:00Z"
}
```

---

## Unhealthy Response

HTTP:

```http id="q4f3n7"
503 Service Unavailable
```

Response:

```json id="vg4j9w"
{
  "status": "unhealthy",
  "services": {
    "database": "unhealthy"
  }
}
```

---

# 20.2 Readiness Check

## Endpoint

```http id="m5g0d8"
GET /health/ready
```

---

## Purpose

Determine whether application can accept requests.

---

## Healthy Response

```json id="6b4pj8"
{
  "status": "ready"
}
```

---

# 20.3 Liveness Check

## Endpoint

```http id="n7w3ta"
GET /health/live
```

---

## Purpose

Determine whether application process is alive.

---

## Response

```json id="r8q2mf"
{
  "status": "alive"
}
```

---

# 21. Database Constraints

---

# users

---

## Constraints

```sql id="4n8s0z"
PRIMARY KEY (id)
```

```sql id="o5w7af"
UNIQUE(email)
```

```sql id="g9u3lw"
UNIQUE(provider_user_id)
```

---

## Additional Column

```sql id="6e2g2f"
avatar_url TEXT
```

---

# tickets

---

## Constraints

```sql id="v2x1lu"
PRIMARY KEY (id)
```

```sql id="f5w9sl"
FOREIGN KEY (user_id)
```

```sql id="e0t1zv"
UNIQUE(ticket_number)
```

---

## Ticket Number Format

```text id="m4h9s1"
TKT-000001
```

Generated from database sequence.

---

## Required Fields

```text id="r7k2t8"
title
description
priority
status
source
```

---

# voice_sessions

---

## Constraints

```sql id="y4b8mn"
PRIMARY KEY(id)
```

```sql id="u6s3ke"
FOREIGN KEY(user_id)
```

```sql id="w8j2vq"
FOREIGN KEY(ticket_id)
```

---

## Required Fields

```text id="m2q7fj"
user_id
transcript
```

---

# workflow_runs

---

## Constraints

```sql id="c5v4el"
PRIMARY KEY(id)
```

```sql id="z8w6ut"
FOREIGN KEY(ticket_id)
```

---

## Status Enum

```text id="n0j3rx"
PENDING
RUNNING
COMPLETED
FAILED
```

---

# workflow_events

---

## Table Definition

```sql id="h4f9pc"
CREATE TABLE workflow_events (
    id UUID PRIMARY KEY,
    workflow_run_id UUID REFERENCES workflow_runs(id),

    event_type VARCHAR(50) NOT NULL,
    event_timestamp TIMESTAMP NOT NULL,

    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## Event Types

```text id="g7x1qu"
WORKFLOW_STARTED
EMAIL_SENDING
EMAIL_SENT
WORKFLOW_COMPLETED
WORKFLOW_FAILED
```

---

# 22. Validation Rules

---

# User Validation

---

## Email

Maximum Length:

```text id="k1q3tw"
255
```

Must be valid email format.

---

## Full Name

Maximum Length:

```text id="s7p4va"
255
```

---

# Ticket Validation

---

## Title

Required

Minimum Length:

```text id="b6h2ny"
5
```

Maximum Length:

```text id="u8r4zg"
200
```

---

## Description

Required

Minimum Length:

```text id="d9m5ql"
10
```

Maximum Length:

```text id="p3v8fx"
5000
```

---

## Priority

Allowed:

```text id="a2t9wb"
LOW
MEDIUM
HIGH
CRITICAL
```

---

# Transcript Validation

---

## Transcript

Required

Minimum Length:

```text id="v5j0ce"
10
```

Maximum Length:

```text id="m7g8zr"
5000
```

---

## Empty Transcript

Rejected.

Error:

```text id="w1c4ln"
TRANSCRIPT_EMPTY
```

---

# Pagination Validation

---

## Page

```text id="f0e2dk"
>= 1
```

---

## Page Size

```text id="n9u1yb"
1 - 100
```

---

# Search Validation

---

## Search Length

Minimum:

```text id="y6p3wx"
2 characters
```

Maximum:

```text id="q8h7ms"
100 characters
```

---

# 23. Correlation ID & Idempotency Requirements

---

# Correlation ID

---

## Request Header

```http id="r1v7mk"
X-Correlation-ID
```

---

## Format

```text id="x3t5jb"
UUID v4
```

---

## Generation

If absent:

```text id="c6n8rf"
Backend generates UUID
```

---

## Mandatory Inclusion

All:

```text id="t4w2ku"
API Responses
Application Logs
Workflow Logs
Internal Requests
WebSocket Events
```

must include correlation ID.

---

# Idempotency Contract

---

## Supported Endpoints

```http id="v8m4sa"
POST /api/v1/voice/sessions
POST /api/v1/voice/sessions/{session_id}/complete
```

---

## Header

```http id="e2u7ny"
Idempotency-Key
```

---

## Format

```text id="g4c8wb"
UUID v4
```

---

## Behavior

Duplicate request with same:

```text id="n1f5jh"
User
Endpoint
Idempotency-Key
```

returns original response.

No duplicate resources created.

---

## Retention Window

Store keys for:

```text id="j9z6pq"
24 Hours
```

---

## Duplicate Response

HTTP:

```http id="u3v8lx"
200 OK
```

Response:

```json id="l5y2cm"
{
  "success": true,
  "correlation_id": "uuid",
  "data": {
    "...": "original response"
  }
}
```

---

## Invalid Key

Error:

```text id="b8r4xj"
VALIDATION_ERROR
```

---

# Sequence Flows, OpenAPI Requirements, Non-Functional Requirements, API Change Management & Definition of Done

---

# 24. Sequence Flows

---

# 24.1 Authentication Flow

```text
User
 │
 ▼
GET /auth/login/google
 │
 ▼
Google OAuth
 │
 ▼
GET /auth/callback/google
 │
 ▼
Create User (if needed)
 │
 ▼
Generate JWT
 │
 ▼
Set HttpOnly Cookie
 │
 ▼
Redirect Dashboard
 │
 ▼
GET /auth/me
 │
 ▼
Authenticated Session
```

---

# 24.2 Voice-to-Ticket Happy Path

```text
User
 │
 ▼
POST /voice/sessions
 │
 ▼
LiveKit Room Created
 │
 ▼
LiveKit Token Generated
 │
 ▼
Frontend Connects
 │
 ▼
User Speaks
 │
 ▼
User Stops Session
 │
 ▼
POST /voice/sessions/{id}/complete
 │
 ▼
Store Transcript
 │
 ▼
Pipecat Extraction
 │
 ▼
Create Ticket
 │
 ▼
Generate Ticket Number
 │
 ▼
Create Workflow Run
 │
 ▼
Trigger n8n
 │
 ▼
Return Response
 │
 ▼
Dashboard Updated
```

---

# 24.3 Workflow Success Flow

```text
Ticket Created
 │
 ▼
Workflow Run Created
 │
 ▼
Status = PENDING
 │
 ▼
n8n Triggered
 │
 ▼
Status = RUNNING
 │
 ▼
Email Sent
 │
 ▼
n8n Callback
 │
 ▼
POST /internal/workflows/status
 │
 ▼
workflow_run = COMPLETED
 │
 ▼
ticket = COMPLETED
 │
 ▼
Create Workflow Event
 │
 ▼
Emit WebSocket Event
 │
 ▼
Dashboard Updated
```

---

# 24.4 Workflow Failure Flow

```text
Ticket Created
 │
 ▼
Workflow Run Created
 │
 ▼
Trigger n8n
 │
 ▼
Attempt 1 Failed
 │
 ▼
Retry Immediate
 │
 ▼
Attempt 2 Failed
 │
 ▼
Retry After 5 Seconds
 │
 ▼
Attempt 3 Failed
 │
 ▼
Retry After 15 Seconds
 │
 ▼
workflow_run = FAILED
 │
 ▼
ticket remains OPEN
 │
 ▼
Create Workflow Event
 │
 ▼
Emit WebSocket Event
```

---

# 24.5 Page Refresh Recovery Flow

```text
User Refreshes Page
 │
 ▼
GET /auth/me
 │
 ▼
GET /tickets
 │
 ▼
GET /workflow-runs
 │
 ▼
Current State Restored
 │
 ▼
Reconnect WebSocket
 │
 ▼
Receive Future Events
```

---

# 24.6 WebSocket Flow

```text
Browser
 │
 ▼
GET /ws
 │
 ▼
Cookie Validation
 │
 ▼
Connection Established
 │
 ▼
TICKET_CREATED
 │
 ▼
WORKFLOW_UPDATED
 │
 ▼
WORKFLOW_EVENT
 │
 ▼
Dashboard Updated
```

---

# 24.7 Correlation ID Flow

```text
Browser
 │
 ▼
X-Correlation-ID
 │
 ▼
FastAPI
 │
 ▼
Ticket Service
 │
 ▼
Workflow Service
 │
 ▼
n8n
 │
 ▼
Internal Callback
 │
 ▼
WebSocket Event
 │
 ▼
Application Logs
```

Same correlation ID must be preserved throughout the lifecycle.

---

# 25. OpenAPI Requirements

---

# Source of Truth

FastAPI OpenAPI specification.

---

# Generated Documentation

Swagger UI:

```http
/docs
```

ReDoc:

```http
/redoc
```

---

# Schema Requirements

Every endpoint must define:

```text
Request Model
Response Model
Validation Rules
Response Codes
Authentication Requirements
```

---

# Pydantic Standards

All request and response payloads must use:

```python
BaseModel
```

No anonymous dictionaries.

---

# Response Modeling

Every endpoint must have:

```python
response_model=
```

defined.

---

# Tags

Swagger grouping:

```text
Authentication
Voice
Tickets
Workflow Runs
Internal
Health
WebSocket
```

---

# Example Values

All models must include:

```text
Field Examples
Descriptions
Validation Rules
```

---

# OpenAPI Version

```text
OpenAPI 3.1
```

---

# 26. Non-Functional API Requirements

---

# Performance Requirements

---

## Authentication

Maximum:

```text
5 Seconds
```

---

## Dashboard Load

Maximum:

```text
2 Seconds
```

---

## Voice Session Creation

Maximum:

```text
2 Seconds
```

---

## Transcript Processing

Maximum:

```text
5 Seconds
```

---

## Ticket Creation

Maximum:

```text
2 Seconds
```

---

## Workflow Completion

Maximum:

```text
10 Seconds
```

---

# Availability Target

```text
99%
```

for portfolio deployment.

---

# Scalability Target

Support:

```text
50 Concurrent Users
```

---

# Security Requirements

All production traffic:

```text
HTTPS Only
```

---

## Required Security Controls

```text
JWT Validation
Ownership Validation
Input Validation
Request Sanitization
Rate Limiting
Idempotency Protection
Correlation Tracking
```

---

## Forbidden

```text
Database Access From Frontend
Secrets In Frontend
JWT In LocalStorage
Public Internal APIs
```

---

# Logging Requirements

Application must log:

```text
Authentication Events
Voice Events
Ticket Events
Workflow Events
Errors
```

---

## Mandatory Log Fields

```text
timestamp
level
correlation_id
user_id
event
message
```

---

# Monitoring Requirements

Monitor:

```text
Database Connectivity
Application Health
Workflow Success Rate
Workflow Failure Rate
Voice Session Count
Ticket Creation Count
```

---

# 27. API Change Management

---

# Versioning Strategy

URL versioning.

Current:

```http
/api/v1
```

Future:

```http
/api/v2
```

---

# Backward Compatibility

Rules:

### Minor Changes

Allowed:

```text
New Optional Fields
New Endpoints
Additional Response Fields
```

---

### Breaking Changes

Require:

```text
New API Version
```

Examples:

```text
Remove Fields
Rename Fields
Change Enums
Change Authentication
```

---

# Deprecation Policy

When introducing:

```http
/api/v2
```

Maintain:

```http
/api/v1
```

for a minimum of:

```text
90 Days
```

---

# Contract Ownership

Owner:

```text
OpsPilot AI Project Owner
```

---

# Contract Update Process

```text
Requirement Change
        ↓
PRD Update
        ↓
API Contract Update
        ↓
Database Review
        ↓
Implementation
```

API changes must never bypass the contract.

---

# 28. Definition of API Done

The API layer is considered complete only when all conditions below are satisfied.

---

# Authentication

```text
✓ Google OAuth Works
✓ JWT Generated
✓ HttpOnly Cookie Set
✓ Logout Works
✓ Protected Routes Enforced
```

---

# Voice

```text
✓ Voice Session Created
✓ LiveKit Token Generated
✓ Transcript Submitted
✓ Pipecat Processing Triggered
```

---

# Tickets

```text
✓ Ticket Created
✓ Ticket Number Generated
✓ Ownership Validation Works
✓ Search Works
✓ Filters Work
✓ Pagination Works
```

---

# Workflow

```text
✓ Workflow Run Created
✓ n8n Triggered
✓ Retry Logic Works
✓ Callback Works
✓ Workflow Events Stored
```

---

# WebSocket

```text
✓ Authentication Works
✓ User Isolation Works
✓ Ticket Events Delivered
✓ Workflow Events Delivered
```

---

# Internal APIs

```text
✓ Internal API Key Validation
✓ Pipecat Callback Works
✓ Workflow Callback Works
```

---

# Security

```text
✓ HTTPS Enabled
✓ Rate Limiting Enabled
✓ Idempotency Enabled
✓ Correlation IDs Enabled
✓ Ownership Validation Enabled
```

---

# Observability

```text
✓ Structured Logging
✓ Correlation Tracking
✓ Health Endpoints
✓ Workflow Visibility
```

---

# Documentation

```text
✓ OpenAPI Generated
✓ Swagger Accessible
✓ ReDoc Accessible
✓ Environment Variables Documented
```

---