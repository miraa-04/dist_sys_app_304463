# 🚖 Campus Ride — Distributed Computing Group Project

> \*****STIJK2124 Distributed Computing | Universiti Utara Malaysia*****\
> A real-time campus transportation platform built to demonstrate a full distributed systems stack — from local development to cloud deployment.

\---

## 📋 Table of Contents

1. [Architecture Diagram](#1-architecture-diagram)
2. [Prerequisites](#2-prerequisites)
3. [Setup Instructions (Parts 1–7)](#3-setup-instructions)
4. [Environment Variables Reference](#4-environment-variables-reference)
5. [Kubernetes Manifests](#5-kubernetes-manifests)
6. [Wireshark Filter Reference](#6-wireshark-filter-reference)
7. [Live Railway URLs \& Endpoints](#7-live-railway-urls--endpoints)
8. [Troubleshooting](#8-troubleshooting)
9. [Team Contributions](#9-team-contributions)

\---

## 1\. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                 CAMPUS RIDE — SYSTEM ARCHITECTURE                │
└─────────────────────────────────────────────────────────────────┘

  USER BROWSER
      │  HTTPS (443)
      ▼
┌──────────────────────┐     ┌─────────────────────────────────────┐
│  TIER 1: FRONTEND    │     │        SUPABASE (BaaS Cloud)         │
│  Next.js + TypeScript│────▶│  Auth (JWT) │ Realtime │ RLS        │
│  Port: 3000          │     │  Region: Singapore (ap-southeast-1) │
└────────┬─────────────┘     └─────────────────────────────────────┘
         │  HTTP REST (port 8000)
         ▼
┌──────────────────────┐
│  TIER 2: BACKEND     │
│  FastAPI (Python)    │
│  Uvicorn ASGI        │
│  Port: 8000          │
└────────┬─────────────┘
         │  TCP PostgreSQL Wire Protocol (port 5432)
         ▼
┌──────────────────────┐
│  TIER 3: DATABASE    │
│  PostgreSQL 15       │
│  Port: 5432          │
└──────────────────────┘

───────────────── INFRASTRUCTURE LAYERS ─────────────────────────

  LOCAL DEV           DOCKER                KUBERNETES          CLOUD
  ─────────────       ──────────────────    ──────────────────  ──────────────
  python server.py    docker-compose up     kubectl apply -f    railway up
  npm run dev         campus-frontend       k8s/                HTTPS auto-TLS
                      campus-backend        HPA: 2-10 pods      GitOps deploy
                      campus-postgres       CoreDNS discovery   Live metrics

───────────────── DOCKER NETWORK (dist_sys_app_default) ──────────

  Container           IP Address      Port    Role
  ──────────────────  ──────────────  ──────  ─────────────────
  campus-frontend     172.18.0.4      3000    Next.js frontend
  campus-postgres     172.18.0.2      5432    PostgreSQL database
  Gateway             172.18.0.1      —       Docker bridge
  Subnet              172.18.0.0/16   —       Bridge network
```

\---

## 2\. Prerequisites

|Tool|Version Used|Installation|
|-|-|-|
|**Git**|Latest|https://git-scm.com/downloads|
|**Node.js**|v25.9.0|https://nodejs.org/en/download|
|**Python**|3.11+|https://www.python.org/downloads|
|**Docker Desktop**|4.25+|https://www.docker.com/products/docker-desktop|
|**kubectl**|1.28+|https://kubernetes.io/docs/tasks/tools|
|**Minikube**|Latest|https://minikube.sigs.k8s.io/docs/start|
|**Railway CLI**|Latest|`npm install -g @railway/cli`|
|**Artillery**|2.0.32|`npm install -g artillery`|
|**Wireshark**|4.6.6|https://www.wireshark.org/download.html|
|**Npcap**|Latest|https://npcap.com (install during Wireshark setup)|

> Windows users: Install Npcap when prompted during Wireshark installation. This enables the Adapter for loopback traffic capture interface needed for Part 6.

\---

## 3\. Setup Instructions

### Part 1 — Local Development

```bash
# Clone the repository
git clone https://github.com/your-org/dist_sys_app.git
cd dist_sys_app

# Backend setup (FastAPI on port 8000)
cd backend
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python server.py

# Frontend setup (open a new terminal)
cd frontend
npm install
npm run dev
```

Verify locally:

* Backend health check → http://127.0.0.1:8000/health returns `{"status":"ok","database":"connected"}`
* Backend Swagger UI → http://127.0.0.1:8000/docs
* Frontend → http://127.0.0.1:3000

\---

### Part 2 — Supabase Authentication

```bash
# 1. Create project at https://supabase.com
#    Region: Southeast Asia (Singapore)

# 2. Copy credentials to frontend/.env.local
NEXT_PUBLIC_SUPABASE_URL=https://your-project-id.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...

# 3. Run migration in Supabase SQL Editor
#    File: supabase/migrations/001_university_ride.sql
```

\---

### Part 3 — Docker Containerisation

```bash
# Build and start all containers
docker-compose build
docker-compose up -d

# Verify containers are running
docker ps
# Expected: campus-frontend (3000), campus-postgres (5432)

# Check container network
docker network inspect dist_sys_app_default

# View logs
docker-compose logs -f

# Stop containers
docker-compose down
```

Our Docker network uses subnet **172.18.0.0/16**:

* `campus-postgres` → **172.18.0.2**
* `campus-frontend` → **172.18.0.4**
* Gateway → **172.18.0.1**

\---

### Part 4 — Kubernetes Orchestration

```bash
# Start Minikube
minikube start

# Verify cluster
kubectl cluster-info
kubectl get nodes

# Deploy all manifests
kubectl apply -f k8s/

# Watch pods start up
kubectl get pods -w

# Check HPA status
kubectl get hpa

# Access via port-forward
kubectl port-forward svc/campus-frontend-service 3000:3000
```

\---

### Part 5 — Railway Cloud Deployment

```bash
# Login and initialise
railway login
railway init

# Deploy services
railway up

# Get live URLs
railway domain
```

After deploying, add your Railway frontend URL to:

1. `backend/app/main.py` → `allow_origins` list (fixes CORS)
2. Supabase Dashboard → Authentication → URL Configuration → Redirect URLs

**Live URLs:**

* Frontend: https://refreshing-connection-production-83d9.up.railway.app
* Backend: https://distsysapp-production.up.railway.app

\---

### Part 6 — Wireshark Network Analysis

```bash
# Make sure Docker containers are running
docker ps

# Exercise 6.1 — TCP handshake and termination
# Wireshark filter: tcp.flags.syn == 1
curl https://distsysapp-production.up.railway.app/health

# Exercise 6.2 — Full layer expansion
# Wireshark filter: tls  →  click Packet 10 (Client Hello)
curl https://distsysapp-production.up.railway.app/api/v1/students/

# Exercise 6.3 — HTTP POST with JSON
curl -X POST https://distsysapp-production.up.railway.app/api/v1/students/ -H "Content-Type: application/json" -d "{\"name\":\"Ali\",\"matric_id\":\"S001\",\"email\":\"ali@uni.edu\",\"lat\":6.45,\"lng\":100.50}"
# Returns: {"name":"Ali","matric_id":"S001","email":"ali@uni.edu","lat":6.45,"lng":100.5,"id":1}

# Exercise 6.4 — TCP termination
# Wireshark filter: tcp.flags.fin == 1

# Exercise 6.5 — Docker container IPs
docker network inspect dist_sys_app_default
docker inspect campus-frontend | findstr IPAddress
docker inspect campus-postgres | findstr IPAddress

# Exercise 6.6 — TLS on Railway
# Wireshark filter: tls  →  look for Client Hello / Server Hello
curl https://distsysapp-production.up.railway.app/health
```

Capture interface: **Adapter for loopback traffic capture** (Exercises 6.1–6.4, 6.6)
Docker interface: **vEthernet (WSL)** (Exercise 6.5)

\---

### Part 7 — Artillery Load Testing

```bash
# Install Artillery
npm install -g artillery
# Installed version: Artillery 2.0.32, Node.js v25.9.0

# Run load test (3 minutes total)
cd C:\dist_sys_app
artillery run load-test.yml

# Generate HTML report
artillery run load-test.yml --output report.json
artillery report report.json
start report.json.html
```

**Load test results (actual):**

|Metric|Value|
|-|-|
|Total requests sent|6,600|
|Total failures|0|
|Overall request rate|37 req/s|
|Min response time|240ms|
|Max response time|1,296ms|
|Mean response time|317.1ms|
|Median response time|320.6ms|
|p95 response time|368.8ms|
|p99 response time|528.6ms|
|Test duration|3 minutes 0 seconds|

Railway CPU spike was clearly visible at 2:27 AM during Phase 2 (50 req/s).
Network ingress spiked to ~1MB. Error rate stayed at **0.0%** throughout.

\---

## 4\. Environment Variables Reference

|Variable|Service|Description|Where to Get|
|-|-|-|-|
|`NEXT_PUBLIC_SUPABASE_URL`|Frontend|Supabase project REST URL|Supabase → Settings → API → Project URL|
|`NEXT_PUBLIC_SUPABASE_ANON_KEY`|Frontend|Public anon key|Supabase → Settings → API → anon public|
|`DATABASE_URL`|Backend|PostgreSQL connection string|Format: `postgresql+asyncpg://user:pass@host:5432/dbname`|
|`SUPABASE_SERVICE_ROLE_KEY`|Backend|Admin key for server-side auth|Supabase → Settings → API → service\_role|
|`SECRET_KEY`|Backend|JWT signing secret|Generate: `openssl rand -hex 32`|
|`CORS_ORIGINS`|Backend|Allowed frontend URLs|Railway frontend URL + http://localhost:3000|
|`NEXT_PUBLIC_API_URL`|Frontend|Backend base URL|Local: `http://localhost:8000` / Prod: Railway backend URL|

> ⚠️ Never commit `.env` or `.env.local` files to GitHub. Use `.env.example` with placeholder values only.

\---

## 5\. Kubernetes Manifests

All manifests are in the `k8s/` folder:

|File|Kind|Description|
|-|-|-|
|`k8s/configmap.yaml`|ConfigMap|Non-sensitive config: backend host, port, node environment|
|`k8s/secret.yaml`|Secret|Sensitive credentials base64-encoded: Supabase URL, anon key, database URL|
|`k8s/postgres.yaml`|Deployment + Service|PostgreSQL 15, ClusterIP service, port 5432, internal only|
|`k8s/backend.yaml`|Deployment + Service|FastAPI, 2 replicas, liveness + readiness probes on `/health`|
|`k8s/frontend.yaml`|Deployment + Service|Next.js, 2 replicas, ClusterIP, port 3000|
|`k8s/ingress.yaml`|Ingress|NGINX: `/` → frontend, `/api` → backend|
|`k8s/hpa.yaml`|HorizontalPodAutoscaler|Backend: min 2 pods, max 10 pods, CPU threshold 70%|

```bash
# Apply everything at once
kubectl apply -f k8s/

# Check HPA is working
kubectl get hpa
kubectl describe hpa campus-backend-hpa
```

\---

## 6\. Wireshark Filter Reference

|Filter|Captures|Exercise|
|-|-|-|
|`tcp.flags.syn == 1`|TCP SYN packets — new connections|6.1|
|`tls`|All TLS handshake and encrypted records|6.2, 6.6|
|`http.request.method == "POST"`|HTTP POST requests|6.3|
|`tcp.flags.fin == 1`|TCP FIN packets — connection teardown|6.4|
|`ip.src == 172.18.0.0/16 or ip.dst == 172.18.0.0/16`|Docker container traffic|6.5|
|`tcp`|All TCP packets|General|
|`ip`|All IP packets|General|

**Capture interfaces used:**

|Interface|Used For|
|-|-|
|Adapter for loopback traffic capture|Exercises 6.1, 6.2, 6.3, 6.4, 6.6|
|vEthernet (WSL / Hyper-V firewall)|Exercise 6.5 — Docker container traffic|

**Key packets observed:**

* Packet 10 → TLSv1.3 Client Hello (274 bytes) — used for Exercise 6.2 full layer expansion
* Packet 71 → TLSv1.3 Application Data (1788 bytes) — POST request for Exercise 6.3
* Packets 994+ → TCP \[FIN, ACK] pairs — Exercise 6.4 termination sequence

\---

## 7\. Live Railway URLs \& Endpoints

|Service|URL|
|-|-|
|**Frontend (Live)**|https://refreshing-connection-production-83d9.up.railway.app|
|**Backend (Live)**|https://distsysapp-production.up.railway.app|
|**API Docs (Swagger)**|https://distsysapp-production.up.railway.app/docs|
|**Health Check**|https://distsysapp-production.up.railway.app/health|

### API Endpoints

|Method|Endpoint|Description|Tested With|
|-|-|-|-|
|`GET`|`/health`|Health probe — returns `{"status":"ok","database":"connected"}`|curl + Artillery|
|`GET`|`/api/v1/students/`|List all students|curl + Artillery|
|`POST`|`/api/v1/students/`|Create student — returns `{"id":1,...}`|curl Exercise 6.3|
|`GET`|`/api/v1/students/{matric_id}`|Get student by matric number|Swagger|
|`GET`|`/api/v1/drivers/`|List all drivers|Artillery|
|`POST`|`/api/v1/drivers/`|Register a driver|Swagger|
|`GET`|`/api/v1/taxis/`|List taxis with location|Artillery|
|`GET`|`/api/v1/buses/`|List campus buses|Swagger|
|`GET`|`/api/v1/bicycles/`|List available bicycles|Swagger|

\---

## 8\. Troubleshooting

### Issue 1: `curl http://127.0.0.1:8000/health` fails — "Could not connect to server"

**Symptom:** `curl: (7) Failed to connect to 127.0.0.1 port 8000`

**Cause:** The backend container is not running locally. In our project the FastAPI backend is deployed on Railway, not in docker-compose.

**Solution:** Use the Railway backend URL instead:

```bash
curl https://distsysapp-production.up.railway.app/health
# Should return: {"status":"ok","database":"connected"}
```

\---

### Issue 2: `docker network inspect campus-ride_default` — network not found

**Symptom:** `Error response from daemon: network campus-ride_default not found`

**Cause:** Our docker-compose project is named `dist_sys_app`, so the network is named `dist_sys_app_default`.

**Solution:**

```bash
docker network ls
docker network inspect dist_sys_app_default
```

\---

### Issue 3: Wireshark shows no packets on loopback

**Symptom:** Capture starts but 0 packets appear after running curl.

**Solution:**

* Make sure **Npcap** is installed (not WinPcap)
* Select **"Adapter for loopback traffic capture"** not "Local Area Connection"
* Run Wireshark as **Administrator**
* Use display filter `tls` instead of `http` since Railway uses HTTPS

\---

### Issue 4: Artillery — `'artillery' is not recognized`

**Symptom:** `'artillery' is not recognized as an internal or external command`

**Solution:**

```bash
npm install -g artillery
artillery version
# Should show: Artillery: 2.0.32
```

\---

### Issue 5: Railway CORS error in browser

**Symptom:** Browser console shows `Access to fetch blocked by CORS policy`

**Solution:** Add your Railway frontend URL to `allow_origins` in `backend/app/main.py`:

```python
allow_origins = [
    "http://localhost:3000",
    "https://refreshing-connection-production-83d9.up.railway.app"
]
```

Then redeploy: `railway up`

\---

### Issue 6: Supabase login redirect fails on Railway

**Symptom:** Login redirects to wrong URL or shows redirect_uri mismatch.

**Solution:**

1. Supabase Dashboard → Authentication → URL Configuration
2. Set **Site URL**: `https://refreshing-connection-production-83d9.up.railway.app`
3. Add **Redirect URL**: `https://refreshing-connection-production-83d9.up.railway.app`

\---

## 9\. Team Contributions

|Member|Matric No.|Role|Parts|Contributions|
|-|-|-|-|-|
|**\[NUR AFRINA ATIQAH BINTI SHAHARUL ANUAR]**|\[304317]|Project Lead \& DevOps|3, 4, 5|Kubernetes manifests (k8s/), Railway deployment, CORS config, HPA setup, Minikube|
|**\[NORAZMIRAH BINTI SULIMIN ]**|\[304008]|Backend Developer|6, 7|FastAPI endpoints, docker-compose, PostgreSQL schema, Docker backend|
|**\[PUTRI NUR AMIRAH BINTI MOHD FITRY DAUD]**|\[304463]|Frontend Developer|1, 2, 8|Next.js frontend, Supabase JWT auth integration, UI components|

\---

## 📁 Project Structure

```
dist_sys_app/
├── backend/
│   ├── app/
│   │   └── main.py              # FastAPI app, CORS config, all endpoints
│   ├── Dockerfile
│   └── requirements.txt
├── frontend/
│   ├── src/
│   ├── Dockerfile
│   └── package.json
├── k8s/
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── postgres.yaml
│   ├── backend.yaml
│   ├── frontend.yaml
│   ├── ingress.yaml
│   └── hpa.yaml
├── supabase/
│   └── migrations/
│       └── 001_university_ride.sql
├── screenshots/                 # All evidence screenshots
├── docker-compose.yml
├── load-test.yml                # Artillery: Phase 1 (10 req/s) + Phase 2 (50 req/s)
├── .env.example
├── .gitignore
└── README.md
```

\---

*STIJK2124 Distributed Computing — Campus Ride Group Project | Universiti Utara Malaysia | Session A252 2025/2026*



