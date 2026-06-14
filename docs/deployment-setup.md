# OpsPilot AI

# Production Deployment Setup

Version: 1.0

Source References:

* PRD v1.0
* API Contract v1.0
* Database Design v1.0
* Sequence Diagrams v1.0
* Local Development Setup v1.0

---

# 1. Purpose

This document defines the complete production deployment architecture for OpsPilot AI.

Goals:

* Single VM deployment
* Fully containerized platform
* Automated CI/CD
* GitHub Actions deployment
* Oracle Cloud Free Tier hosting
* HTTPS everywhere
* Production-grade operational practices
* Minimal operational cost

Deployment target:

```text
Git Push
   ↓
GitHub Actions
   ↓
GHCR Images
   ↓
Oracle VM
   ↓
Automatic Deployment
```

---

# 2. Final Production Architecture

```text
Internet
    │
    ▼

┌──────────────────────────────┐
│            Nginx             │
│         SSL Gateway          │
└─────────────┬────────────────┘
              │
 ┌────────────┼────────────┬─────────────┐
 │            │            │             │
 ▼            ▼            ▼             ▼

Frontend    Backend      LiveKit      Certbot
Next.js     FastAPI

               │
      ┌────────┼────────┐
      │        │        │
      ▼        ▼        ▼

 PostgreSQL  Pipecat   n8n
```

---

# 3. Infrastructure Decisions

## Hosting

```text
Oracle Cloud Free Tier
```

---

## Compute

```text
VM.Standard.A1.Flex
```

Resources:

```text
4 OCPU
24 GB RAM
```

---

## Operating System

```text
Ubuntu 22.04 LTS
```

---

## Environment Strategy

Single environment only.

```text
main
 ↓
production
```

---

# 4. Domain Architecture

## Root Domain

```text
akirasi.in
```

---

## Production Subdomains

### Frontend

```text
opspilot.akirasi.in
```

---

### Backend

```text
api.opspilot.akirasi.in
```

---

### LiveKit

```text
livekit.opspilot.akirasi.in
```

---

## Internal Services

Not publicly exposed:

```text
postgres
pipecat
n8n
```

---

# 5. DNS Configuration

Create:

```dns
A    opspilot.akirasi.in         <oracle_vm_ip>

A    api.opspilot.akirasi.in     <oracle_vm_ip>

A    livekit.opspilot.akirasi.in <oracle_vm_ip>
```

---

# 6. Oracle Network Security

## Public Ports

Allow only:

```text
80/tcp
443/tcp
22/tcp
```

---

## Internal Only

Blocked from internet:

```text
5432
5678
7880
8000
9000
3000
```

Container networking handles internal communication.

---

# 7. Production Repository Structure

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
│   ├── certbot/
│   ├── postgres/
│   ├── livekit/
│   └── scripts/
│
├── docs/
│
├── docker-compose.prod.yml
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
│
└── README.md
```

---

# 8. Production Containers

## frontend

Purpose:

```text
Next.js Production Build
```

Internal Port:

```text
3000
```

---

## backend

Purpose:

```text
FastAPI API
```

Internal Port:

```text
8000
```

---

## postgres

Version:

```text
PostgreSQL 16
```

Internal Port:

```text
5432
```

Persistent Volume:

```text
postgres_data
```

---

## livekit

Purpose:

```text
Realtime Audio Transport
```

Internal Port:

```text
7880
```

Persistent Volume:

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

* LiveKit participant
* Deepgram STT
* Gemini extraction
* Callback delivery

Internal Port:

```text
9000
```

Stateless.

---

## n8n

Purpose:

```text
Workflow Automation
```

Internal Port:

```text
5678
```

Persistent Volume:

```text
n8n_data
```

---

## nginx

Purpose:

```text
TLS Termination
Reverse Proxy
WebSocket Proxy
LiveKit Proxy
```

Public Ports:

```text
80
443
```

---

# 9. OAuth Configuration

## Callback URL

Google OAuth Callback:

```text
https://api.opspilot.akirasi.in/api/v1/auth/callback/google
```

Backend handles:

```text
Code Exchange
User Lookup
JWT Creation
Cookie Creation
Redirect
```

Frontend never handles OAuth secrets.

---

# 10. SSL Strategy

## Provider

```text
Let's Encrypt
```

---

## Tool

```text
Certbot
```

---

## Certificates

Issued for:

```text
opspilot.akirasi.in

api.opspilot.akirasi.in

livekit.opspilot.akirasi.in
```

---

## Renewal

Cron:

```bash
0 2 * * * certbot renew --quiet
```

---

# 11. Docker Network

Single network:

```yaml
networks:
  opspilot-network:
```

---

## Internal DNS

```text
frontend
backend
postgres
livekit
pipecat
n8n
```

No localhost references.

---

# 12. Production Environment Variables

## Backend

```env
DATABASE_URL=postgresql+psycopg://postgres:<password>@postgres:5432/opspilot

JWT_SECRET=<secure_random>

JWT_ALGORITHM=HS256

GOOGLE_CLIENT_ID=

GOOGLE_CLIENT_SECRET=

LIVEKIT_API_KEY=

LIVEKIT_API_SECRET=

PIPECAT_URL=http://pipecat:9000

N8N_WEBHOOK_URL=http://n8n:5678/webhook/ticket-created

INTERNAL_API_KEY=<secure_random>

LLM_PROVIDER=GEMINI

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
DEEPGRAM_API_KEY=

LLM_PROVIDER=GEMINI

GEMINI_API_KEY=

INTERNAL_API_KEY=
```

---

## Frontend

```env
NEXT_PUBLIC_API_URL=https://api.opspilot.akirasi.in/api/v1

NEXT_PUBLIC_LIVEKIT_URL=https://livekit.opspilot.akirasi.in
```

---

# 13. GitHub Container Registry Strategy

## Images

Frontend

```text
ghcr.io/<user>/opspilot-frontend
```

---

Backend

```text
ghcr.io/<user>/opspilot-backend
```

---

Pipecat

```text
ghcr.io/<user>/opspilot-pipecat
```

---

## Tags

```text
latest

<commit_sha>
```

---

# 14. CI Pipeline

File:

```text
.github/workflows/ci.yml
```

---

## Stage 1

Frontend Validation

```bash
npm ci

npm run lint

npm run build
```

---

## Stage 2

Backend Validation

```bash
ruff check .

pytest
```

---

## Stage 3

Security Scan

```bash
trivy fs .
```

---

## Stage 4

Container Build

```bash
frontend

backend

pipecat
```

---

## Stage 5

Push To GHCR

Push:

```text
frontend

backend

pipecat
```

---

# 15. CD Pipeline

File:

```text
.github/workflows/deploy.yml
```

Trigger:

```yaml
on:
  push:
    branches:
      - main
```

---

## Deployment Flow

```text
Push Main
     ↓
CI Success
     ↓
Build Images
     ↓
Push GHCR
     ↓
SSH Oracle VM
     ↓
Pull Images
     ↓
Restart Containers
     ↓
Health Check
     ↓
Deployment Complete
```

---

# 16. Deployment User

Create:

```bash
sudo adduser deploy
```

Grant:

```bash
docker
```

group access.

Used only by GitHub Actions.

---

# 17. GitHub Secrets

## Oracle

```text
ORACLE_HOST

ORACLE_USER

ORACLE_SSH_KEY
```

---

## Registry

```text
GHCR_USERNAME

GHCR_TOKEN
```

---

## Application

```text
DATABASE_URL

JWT_SECRET

GOOGLE_CLIENT_ID

GOOGLE_CLIENT_SECRET

LIVEKIT_API_KEY

LIVEKIT_API_SECRET

DEEPGRAM_API_KEY

GEMINI_API_KEY

SMTP_USERNAME

SMTP_PASSWORD

INTERNAL_API_KEY
```

---

# 18. Deployment Script

```bash
cd /opt/opspilot

docker compose -f docker-compose.prod.yml pull

docker compose -f docker-compose.prod.yml up -d

docker image prune -f
```

---

# 19. Nginx Routing

## Frontend

```nginx
server {

    server_name opspilot.akirasi.in;

    location / {

        proxy_pass http://frontend:3000;

    }

}
```

---

## Backend

```nginx
server {

    server_name api.opspilot.akirasi.in;

    location / {

        proxy_pass http://backend:8000;

    }

}
```

---

## LiveKit

```nginx
server {

    server_name livekit.opspilot.akirasi.in;

    location / {

        proxy_pass http://livekit:7880;

        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;

        proxy_set_header Connection "upgrade";

    }

}
```

---

# 20. Persistent Volumes

```yaml
volumes:

  postgres_data:

  n8n_data:

  livekit_data:
```

---

# 21. Backup Strategy

## Scope

Only PostgreSQL.

---

## Frequency

```text
Daily
```

---

## Backup Directory

```text
/opt/backups
```

---

## Backup Script

```bash
#!/bin/bash

DATE=$(date +%F)

docker exec postgres \
pg_dump -U postgres opspilot \
> /opt/backups/opspilot-$DATE.sql
```

---

## Retention

```bash
find /opt/backups \
-type f \
-mtime +7 \
-delete
```

---

# 22. Recovery Objectives

## RPO

```text
24 Hours
```

---

## RTO

```text
30 Minutes
```

---

# 23. Health Verification

## Frontend

```text
https://opspilot.akirasi.in
```

Loads successfully.

---

## Backend

```text
https://api.opspilot.akirasi.in/health
```

Returns healthy.

---

## OAuth

```text
Google Login Successful
```

---

## LiveKit

```text
Room Creation Successful
```

---

## Pipecat

```text
Transcript Processing Successful
```

---

## Ticket Creation

```text
Ticket Persisted
```

---

## Workflow

```text
Email Delivered
```

---

# 24. Rollback Procedure

Identify previous image:

```bash
docker images
```

---

Update image tag.

```yaml
image: ghcr.io/...:<previous_sha>
```

---

Redeploy:

```bash
docker compose -f docker-compose.prod.yml up -d
```

---

Verify:

```text
Health Endpoint

OAuth

Voice Flow

Workflow Flow
```

---

# 25. Disaster Recovery

## Database Failure

Restore latest backup.

```bash
psql -U postgres opspilot < backup.sql
```

---

## VM Failure

Provision new VM.

Restore:

```text
Git Repository

Docker Compose

Environment File

Database Backup
```

Redeploy.

---

# 26. Production Verification Checklist

```text
✓ Oracle VM Provisioned

✓ Docker Installed

✓ Nginx Installed

✓ Certbot Installed

✓ DNS Configured

✓ SSL Active

✓ GHCR Access Working

✓ PostgreSQL Running

✓ LiveKit Running

✓ Pipecat Running

✓ n8n Running

✓ Backend Running

✓ Frontend Running

✓ OAuth Working

✓ Voice Session Created

✓ Transcript Generated

✓ Ticket Created

✓ Workflow Triggered

✓ Email Delivered

✓ Health Endpoint Healthy

✓ Backup Script Scheduled

✓ GitHub Actions Deploying

✓ End-To-End Voice-To-Ticket Flow Operational
```

---

# 27. Operations Definition Of Done

Infrastructure is considered production ready when:

```text
✓ Git Push Deploys Automatically

✓ HTTPS Enabled Everywhere

✓ OAuth Authentication Operational

✓ LiveKit Accessible Through Nginx

✓ Pipecat Joins LiveKit Rooms

✓ Ticket Creation Operational

✓ Workflow Automation Operational

✓ Daily Backups Scheduled

✓ Rollback Procedure Verified

✓ End-ToEnd Demo Flow Successful
```

The resulting architecture stays aligned with the PRD, API contract, database design, sequence diagrams, and local development setup while keeping the entire platform deployable on a single Oracle Free Tier VM with minimal operational cost.
