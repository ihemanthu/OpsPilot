# OpsPilot AI

## Database Design Document

### Version 1.0

**Source References**

* PRD v1.0 
* API Contract v1.0 

---

# 1. Database Overview

## Database Engine

```text
PostgreSQL 16+
```

## Purpose

Acts as the system of record for:

```text
Users
Voice Sessions
Tickets
Workflow Runs
Workflow Events
Idempotency Records
```

---

# 2. Entity Relationship Diagram

```text
users
 │
 │ 1:N
 ▼
voice_sessions
 │
 │ N:1
 ▼
tickets
 │
 │ 1:N
 ▼
workflow_runs
 │
 │ 1:N
 ▼
workflow_events


users
 │
 │ 1:N
 ▼
tickets


users
 │
 │ 1:N
 ▼
idempotency_keys
```

---

# 3. Enum Definitions

## ticket_status

```sql
CREATE TYPE ticket_status AS ENUM (
    'OPEN',
    'IN_PROGRESS',
    'COMPLETED',
    'FAILED'
);
```

---

## ticket_priority

```sql
CREATE TYPE ticket_priority AS ENUM (
    'LOW',
    'MEDIUM',
    'HIGH',
    'CRITICAL'
);
```

---

## ticket_source

```sql
CREATE TYPE ticket_source AS ENUM (
    'VOICE'
);
```

---

## workflow_status

```sql
CREATE TYPE workflow_status AS ENUM (
    'PENDING',
    'RUNNING',
    'COMPLETED',
    'FAILED'
);
```

---

## workflow_event_type

```sql
CREATE TYPE workflow_event_type AS ENUM (
    'WORKFLOW_STARTED',
    'EMAIL_SENDING',
    'EMAIL_SENT',
    'WORKFLOW_COMPLETED',
    'WORKFLOW_FAILED'
);
```

---

# 4. Users Table

## Purpose

Stores authenticated users from Google OAuth.

## Table

```sql
CREATE TABLE users (

    id UUID PRIMARY KEY,

    email VARCHAR(255) NOT NULL UNIQUE,

    full_name VARCHAR(255) NOT NULL,

    avatar_url TEXT,

    provider VARCHAR(50) NOT NULL DEFAULT 'GOOGLE',

    provider_user_id VARCHAR(255) NOT NULL UNIQUE,

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

---

## Indexes

```sql
CREATE UNIQUE INDEX idx_users_email
ON users(email);

CREATE UNIQUE INDEX idx_users_provider_user
ON users(provider_user_id);
```

---

# 5. Tickets Table

## Purpose

Stores AI-generated tickets.

## Table

```sql
CREATE TABLE tickets (

    id UUID PRIMARY KEY,

    ticket_number VARCHAR(20) NOT NULL UNIQUE,

    user_id UUID NOT NULL
        REFERENCES users(id),

    title VARCHAR(200) NOT NULL,

    description TEXT NOT NULL,

    priority ticket_priority NOT NULL,

    status ticket_status NOT NULL DEFAULT 'OPEN',

    source ticket_source NOT NULL DEFAULT 'VOICE',

    correlation_id UUID NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

---

## Ticket Number Sequence

```sql
CREATE SEQUENCE ticket_number_seq
START 1;
```

Generated as:

```text
TKT-000001
TKT-000002
TKT-000003
```

---

## Indexes

```sql
CREATE INDEX idx_tickets_user
ON tickets(user_id);

CREATE INDEX idx_tickets_status
ON tickets(status);

CREATE INDEX idx_tickets_priority
ON tickets(priority);

CREATE INDEX idx_tickets_created
ON tickets(created_at DESC);

CREATE INDEX idx_tickets_correlation
ON tickets(correlation_id);
```

---

## Full Text Search

Supports:

```text
ticket_number
title
description
```

```sql
ALTER TABLE tickets
ADD COLUMN search_vector tsvector;

CREATE INDEX idx_tickets_search
ON tickets
USING GIN(search_vector);
```

---

# 6. Voice Sessions Table

## Purpose

Stores voice session lifecycle and transcript data.

## Table

```sql
CREATE TABLE voice_sessions (

    id UUID PRIMARY KEY,

    user_id UUID NOT NULL
        REFERENCES users(id),

    ticket_id UUID
        REFERENCES tickets(id),

    livekit_room_id VARCHAR(255) NOT NULL,

    transcript TEXT,

    status VARCHAR(50) NOT NULL,

    correlation_id UUID NOT NULL,

    started_at TIMESTAMP NOT NULL DEFAULT NOW(),

    completed_at TIMESTAMP,

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

---

## Status Values

```text
CREATED
CONNECTED
RECORDING
PROCESSING
COMPLETED
FAILED
TIMEOUT
```

---

## Constraints

```sql
ALTER TABLE voice_sessions
ADD CONSTRAINT chk_transcript_length
CHECK (
    transcript IS NULL
    OR LENGTH(transcript) <= 5000
);
```

---

## Indexes

```sql
CREATE INDEX idx_voice_sessions_user
ON voice_sessions(user_id);

CREATE INDEX idx_voice_sessions_ticket
ON voice_sessions(ticket_id);

CREATE INDEX idx_voice_sessions_created
ON voice_sessions(created_at DESC);
```

---

# 7. Workflow Runs Table

## Purpose

Tracks workflow executions triggered from tickets.

## Table

```sql
CREATE TABLE workflow_runs (

    id UUID PRIMARY KEY,

    ticket_id UUID NOT NULL
        REFERENCES tickets(id),

    status workflow_status NOT NULL,

    trigger_event VARCHAR(100) NOT NULL,

    retry_count INTEGER NOT NULL DEFAULT 0,

    output_payload JSONB,

    error_payload JSONB,

    correlation_id UUID NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

---

## Indexes

```sql
CREATE INDEX idx_workflow_ticket
ON workflow_runs(ticket_id);

CREATE INDEX idx_workflow_status
ON workflow_runs(status);

CREATE INDEX idx_workflow_created
ON workflow_runs(created_at DESC);
```

---

# 8. Workflow Events Table

## Purpose

Stores workflow execution history and dashboard timeline.

## Table

```sql
CREATE TABLE workflow_events (

    id UUID PRIMARY KEY,

    workflow_run_id UUID NOT NULL
        REFERENCES workflow_runs(id),

    event_type workflow_event_type NOT NULL,

    message TEXT,

    event_timestamp TIMESTAMP NOT NULL,

    correlation_id UUID NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

---

## Indexes

```sql
CREATE INDEX idx_workflow_events_run
ON workflow_events(workflow_run_id);

CREATE INDEX idx_workflow_events_timestamp
ON workflow_events(event_timestamp DESC);
```

---

# 9. Idempotency Keys Table

## Purpose

Prevents duplicate resource creation.

Required by:

```text
POST /voice/sessions
POST /voice/sessions/{session_id}/complete
```

as defined in the API contract. 

---

## Table

```sql
CREATE TABLE idempotency_keys (

    id UUID PRIMARY KEY,

    user_id UUID NOT NULL
        REFERENCES users(id),

    endpoint VARCHAR(255) NOT NULL,

    idempotency_key UUID NOT NULL,

    response_payload JSONB NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    expires_at TIMESTAMP NOT NULL
);
```

---

## Unique Constraint

```sql
ALTER TABLE idempotency_keys
ADD CONSTRAINT uq_idempotency
UNIQUE (
    user_id,
    endpoint,
    idempotency_key
);
```

---

## Retention

```text
24 Hours
```

---

# 10. Audit & Correlation Strategy

Every business table stores:

```text
correlation_id
created_at
updated_at
```

Correlation IDs enable tracing across:

```text
Browser
FastAPI
LiveKit
Pipecat
Ticket Creation
Workflow Runs
n8n
Callbacks
WebSocket Events
```

as mandated by the API contract. 

---

# 11. Cascading Rules

## Users

```text
User deletion disabled
```

No cascade.

---

## Tickets

```text
Ticket deletion disabled
```

Historical record.

---

## Workflow Runs

```text
Never deleted automatically
```

Required for audit trail.

---

## Workflow Events

```text
Retained indefinitely
```

For dashboard history.

---

# 12. Expected Database Size (Portfolio Scale)

| Table            | Expected Records       |
| ---------------- | ---------------------- |
| users            | < 5,000                |
| tickets          | < 100,000              |
| voice_sessions   | < 100,000              |
| workflow_runs    | < 100,000              |
| workflow_events  | < 500,000              |
| idempotency_keys | Rolling 24-hour window |

---

# 13. Migration Order

```text
1. Enum Types
2. users
3. ticket_number_seq
4. tickets
5. voice_sessions
6. workflow_runs
7. workflow_events
8. idempotency_keys
9. indexes
10. search_vector trigger
```

This design is fully aligned with the API contract and resolves the inconsistencies currently present in the PRD database section, especially the missing `ticket_number`, `avatar_url`, `workflow_events`, `correlation_id`, and `idempotency_keys` structures.  
