 # OpsPilot AI

# Local Development Setup

Version: 1.0

Source References:

* PRD v1.0
* API Contract v1.0
* Database Design v1.0
* Sequence Diagrams v1.0

---

# 1. Purpose

This document defines the complete local development environment for OpsPilot AI.

Goals:

* Fully dockerized development environment
* One-command startup
* Hot reload for frontend and backend
* Local HTTPS support
* Production-like architecture
* Consistent onboarding experience

The local environment must mirror production architecture wherever practical.

---

# 2. Local Architecture

## Development Stack

```text
┌─────────────────────────────────────────────┐
│                 Browser                      │
└─────────────────────┬───────────────────────┘
                      │
                      │ HTTPS
                      ▼
┌─────────────────────────────────────────────┐
│                  Nginx                       │
└───────┬─────────────┬─────────────┬──────────┘
        │             │             │
        ▼             ▼             ▼

Frontend       FastAPI Backend     LiveKit
Next.js

                    │
                    │
                    ▼

               PostgreSQL

                    │
                    ▼

                 Pipecat

                    │
                    ▼

                   n8n
```

---

# 3. Services

## frontend

Purpose:

```text
Next.js Application
```

Responsibilities:

* Google Login
* Dashboard
* Voice Controls
* Ticket Display
* Workflow Display
* WebSocket Client

Port:

```text
3000
```

Internal DNS:

```text
frontend
```

---

## backend

Purpose:

```text
FastAPI Application
```

Responsibilities:

* Authentication
* Authorization
* Ticket APIs
* Workflow APIs
* WebSocket
* LiveKit Integration
* Pipecat Integration
* n8n Integration

Port:

```text
8000
```

Internal DNS:

```text
backend
```

---

## postgres

Purpose:

```text
System of Record
```

Version:

```text
PostgreSQL 16
```

Port:

```text
5432
```

Volume:

```text
postgres_data
```

---

## livekit

Purpose:

```text
Realtime Audio Transport
```

Port:

```text
7880
```

Volume:

```text
livekit_data
```

---

## pipecat

Purpose:

```text
AI Processing Service
```

Responsibilities:

* Deepgram STT
* Title Extraction
* Description Extraction
* Priority Extraction
* Confidence Generation

Port:

```text
9000
```

Volume:

```text
None
```

Pipecat is stateless.

---

## n8n

Purpose:

```text
Workflow Automation
```

Responsibilities:

* Email Notification
* Workflow Execution
* Callback Delivery

Port:

```text
5678
```

Volume:

```text
n8n_data
```

---

## nginx

Purpose:

```text
Local HTTPS Gateway
```

Responsibilities:

* TLS Termination
* Reverse Proxy
* Local Domain Routing

Ports:

```text
80
443
```

---

# 4. Local Domains

## Hosts File

Add:

```text
127.0.0.1 opspilot.local
127.0.0.1 api.opspilot.local
127.0.0.1 livekit.opspilot.local
127.0.0.1 n8n.opspilot.local
```

---

## URLs

Frontend

```text
https://opspilot.local
```

Backend

```text
https://api.opspilot.local
```

LiveKit

```text
https://livekit.opspilot.local
```

n8n

```text
https://n8n.opspilot.local
```

---

# 5. Local HTTPS

## Tool

```text
mkcert
```

---

## Install

Mac

```bash
brew install mkcert

brew install nss
```

---

## Initialize

```bash
mkcert -install
```

---

## Generate Certificates

```bash
mkdir -p infrastructure/nginx/certs

mkcert \
opspilot.local \
api.opspilot.local \
livekit.opspilot.local \
n8n.opspilot.local
```

---

# 6. Repository Structure

```text
opspilot-ai-lite/

├── frontend/
│
├── backend/
│
├── pipecat/
│
├── infrastructure/
│   ├── nginx/
│   ├── postgres/
│   ├── livekit/
│   └── n8n/
│
├── docs/
│
├── docker-compose.yml
│
├── .env
│
└── README.md
```

---

# 7. Docker Network

Single bridge network.

```yaml
networks:
  opspilot-network:
```

---

## Internal Service Discovery

```text
frontend → backend

backend → postgres

backend → livekit

backend → pipecat

backend → n8n
```

No localhost references allowed between containers.

---

# 8. Environment Variables

## Root .env

```env
COMPOSE_PROJECT_NAME=opspilot
```

---

## Backend

```env
DATABASE_URL=postgresql+psycopg://postgres:postgres@postgres:5432/opspilot

JWT_SECRET=change-me
JWT_ALGORITHM=HS256

GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

LIVEKIT_API_KEY=
LIVEKIT_API_SECRET=

PIPECAT_URL=http://pipecat:9000

N8N_WEBHOOK_URL=http://n8n:5678/webhook/ticket-created

INTERNAL_API_KEY=

LLM_PROVIDER=GEMINI

OPENAI_API_KEY=
GEMINI_API_KEY=

SUPPORT_EMAIL=

SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=
SMTP_PASSWORD=
```

---

## Pipecat

```env
STT_PROVIDER=DEEPGRAM

DEEPGRAM_API_KEY=

LLM_PROVIDER=GEMINI

OPENAI_API_KEY=
GEMINI_API_KEY=

OLLAMA_URL=http://host.docker.internal:11434
```

---

## Frontend

```env
NEXT_PUBLIC_API_URL=https://api.opspilot.local/api/v1

NEXT_PUBLIC_LIVEKIT_URL=https://livekit.opspilot.local
```

---

# 9. LLM Provider Strategy

Supported providers:

```text
OPENAI
GEMINI
OLLAMA
```

---

## Switching Provider

```env
LLM_PROVIDER=OPENAI
```

or

```env
LLM_PROVIDER=GEMINI
```

or

```env
LLM_PROVIDER=OLLAMA
```

No code changes required.

---

# 10. Email Strategy

## Local Development

Uses Gmail App Password.

```env
SMTP_HOST=smtp.gmail.com

SMTP_PORT=587

SMTP_USERNAME=yourgmail@gmail.com

SMTP_PASSWORD=xxxxxxxxxxxxxxxx
```

---

## n8n Workflow

```text
Ticket Created
       ↓
Send Gmail Email
       ↓
Call Backend Callback
```

---

# 11. Hot Reload Configuration

## Frontend

Container mounts source code.

```yaml
volumes:
  - ./frontend:/app
```

Run:

```bash
npm run dev
```

---

## Backend

Container mounts source code.

```yaml
volumes:
  - ./backend:/app
```

Run:

```bash
uvicorn app.main:app \
--host 0.0.0.0 \
--port 8000 \
--reload
```

---

## Pipecat

Container mounts source code.

```yaml
volumes:
  - ./pipecat:/app
```

Run:

```bash
python main.py
```

---

# 12. Database Migrations

## Tool

```text
Alembic
```

---

## Create Migration

```bash
docker compose exec backend \
alembic revision --autogenerate \
-m "create tickets table"
```

---

## Apply Migration

```bash
docker compose exec backend \
alembic upgrade head
```

---

## Current Database State

Verify:

```bash
docker compose exec backend \
alembic current
```

---

# 13. Startup Sequence

Order:

```text
postgres
      ↓
backend
      ↓
livekit
      ↓
pipecat
      ↓
n8n
      ↓
frontend
      ↓
nginx
```

---

# 14. First Time Setup

## Step 1

Clone repository.

```bash
git clone <repo>
```

---

## Step 2

Create environment file.

```bash
cp .env.example .env
```

---

## Step 3

Generate local certificates.

```bash
mkcert -install
```

---

## Step 4

Start services.

```bash
docker compose up -d
```

---

## Step 5

Run migrations.

```bash
docker compose exec backend \
alembic upgrade head
```

---

## Step 6

Open application.

```text
https://opspilot.local
```

---

# 15. Verification Checklist

Frontend:

```text
https://opspilot.local
```

Loads successfully.

---

Backend:

```text
https://api.opspilot.local/health
```

Returns:

```json
{
  "status": "healthy",
  "services": {
    "database": "healthy"
  },
  "timestamp": "2026-06-14T10:00:00Z"
}
```

---

Database:

```bash
docker compose exec postgres psql
```

Connection succeeds.

---

LiveKit:

```text
https://livekit.opspilot.local
```

Reachable.

---

n8n:

```text
https://n8n.opspilot.local
```

Reachable.

---

Google OAuth:

```text
Login succeeds
```

---

Voice Session:

```text
Session created
```

---

Pipecat:

```text
Extraction succeeds
```

---

Ticket Creation:

```text
Ticket stored
```

---

Workflow:

```text
Email delivered
```

---

# 16. Common Commands

## Start

```bash
docker compose up -d
```

---

## Stop

```bash
docker compose down
```

---

## Restart

```bash
docker compose restart
```

---

## Logs

All services:

```bash
docker compose logs -f
```

---

Backend:

```bash
docker compose logs -f backend
```

---

Pipecat:

```bash
docker compose logs -f pipecat
```

---

n8n:

```bash
docker compose logs -f n8n
```

---

## Rebuild

```bash
docker compose up -d --build
```

---

# 17. Development Definition of Ready

The local environment is considered ready when:

```text
✓ Docker Compose Starts Successfully

✓ PostgreSQL Connected

✓ Alembic Migrations Applied

✓ Frontend Hot Reload Works

✓ Backend Hot Reload Works

✓ Google OAuth Works

✓ LiveKit Session Creation Works

✓ Deepgram Transcription Works

✓ Pipecat Extraction Works

✓ Ticket Creation Works

✓ n8n Workflow Executes

✓ Gmail Email Delivered

✓ HTTPS Enabled

✓ End-to-End Voice To Ticket Flow Operational
```
