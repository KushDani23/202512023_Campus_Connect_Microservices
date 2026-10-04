# CampusConnect — Microservices & API Gateway

> **Web Services & Service Oriented Architecture (SOA) — Lab 6 & 7**
> **Student ID:** 202512023 | **Course:** Web Services & SOA

[![Live](https://img.shields.io/badge/Status-Live%20on%20Render-brightgreen)](https://campusconnect-api-gateway-c37d.onrender.com/health)
[![Node.js](https://img.shields.io/badge/Node.js-Express-339933)](https://nodejs.org/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED)](https://www.docker.com/)
[![MongoDB](https://img.shields.io/badge/Database-MongoDB%20Atlas-47A248)](https://www.mongodb.com/atlas)
[![Render](https://img.shields.io/badge/Cloud-Render.com-46E3B7)](https://render.com)

---

## Live Cloud Endpoints

| Service | URL | Status |
|---|---|---|
| **API Gateway** | https://campusconnect-api-gateway-c37d.onrender.com | Live |
| User Service | https://campusconnect-user-service-csdw.onrender.com | Live |
| Product Service | https://campusconnect-product-service-f43o.onrender.com | Live |
| Order Service | https://campusconnect-order-service-gdji.onrender.com | Live |

> Free-tier instances spin down after inactivity. The first request may take ~30-50 seconds to cold-start.

---

## Table of Contents

1. [What This Lab Covers](#1-what-this-lab-covers)
2. [Tech Stack](#2-tech-stack)
3. [Architecture Overview](#3-architecture-overview)
4. [How the API Gateway Works](#4-how-the-api-gateway-works)
5. [Service Discovery — Config-Based Approach](#5-service-discovery--config-based-approach)
6. [API Endpoint Reference](#6-api-endpoint-reference)
7. [Cloud Deployment (Render + MongoDB Atlas)](#7-cloud-deployment-render--mongodb-atlas)
8. [Running Locally with Docker Compose](#8-running-locally-with-docker-compose)
9. [Testing the Gateway](#9-testing-the-gateway)
10. [Discussion & Analysis](#10-discussion--analysis)
11. [Project Structure](#11-project-structure)

---

## 1. What This Lab Covers

This lab builds and deploys a production-ready **API Gateway** on top of three independent microservices, converting the architecture into a fully cloud-hosted system.

**Lab 6 — Microservices Decomposition:**
- Broke a monolithic backend into three autonomous services: **User**, **Product**, and **Order**
- Each service owns its own database (`user_db`, `product_db`, `order_db`)
- Services communicate over an isolated Docker bridge network

**Lab 7 — API Gateway + Cloud Deployment:**
- Built a **unified reverse proxy gateway** (port 8080) as the single public entry point
- Implemented **path-based routing**: `/users/*` to User Service, `/products/*` to Product Service, `/orders/*` to Order Service
- Added **centralized logging**, **health monitoring**, and **fault-tolerant error handling** at the gateway level
- Deployed all four containers to **Render.com** cloud using Docker + a Blueprint `render.yaml`
- Connected each service to **MongoDB Atlas** for persistent cloud storage

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 18 (Alpine) |
| Framework | Express.js |
| Gateway Proxy | http-proxy-middleware |
| Database | MongoDB (via Mongoose) |
| Cloud DB | MongoDB Atlas (M0 Free Tier) |
| Containerization | Docker + Docker Compose |
| Cloud Hosting | Render.com (Free Web Services) |
| API Testing | Postman / PowerShell scripts |

---

## 3. Architecture Overview

```
                     CLIENT / POSTMAN
                  (Browser, Mobile, REST Tool)
                           |
                           | HTTPS
                           v
         +------------------------------------------+
         |          API GATEWAY  :8080              |
         |  +------------------------------------+  |
         |  |  Path Router (http-proxy-middleware|  |
         |  |  Request Logger                   |  |
         |  |  Health Check Handler (/health)   |  |
         |  |  Error Interceptor (502/503)      |  |
         |  +------------------------------------+  |
         +------+-------------+----------+----------+
                |             |          |
    ============|=============|==========|===========
    ||   INTERNAL DOCKER NETWORK (campus-network)  ||
    ||   (Services NOT exposed to public internet) ||
    ============|=============|==========|===========
                |             |          |
       /users/* |  /products/*|  /orders/|
                v             v          v
      +--------------+ +-----------+ +-----------+
      | user-service | |product-svc| |order-svc  |
      |    :3001     | |   :3002   | |   :3003   |
      |  CRUD Users  | |  Products | |  Orders   |
      +------+-------+ +-----+-----+ +-----+-----+
             |               |             |
             |    Inter-Service REST Calls |
             |<----------------------------+
             |  GET /users/:id (order validation)
             |
             v
    +---------------------------------------------------+
    |          MongoDB Atlas (Cloud)                    |
    |   user_db        product_db        order_db       |
    +---------------------------------------------------+
```

**Key Design Decision:** Only the API Gateway has a publicly mapped port. The three microservices use Docker's `expose` (not `ports`), making them reachable only within the private `campus-network` bridge.

---

## 4. How the API Gateway Works

The gateway (`api-gateway/`) is a standalone Express.js application with four responsibilities:

### 4.1 Reverse Proxy Routing

Uses `http-proxy-middleware` to forward requests based on URL prefix:

```js
// Requests to /users/**    => forwarded to USER_SERVICE_URL
// Requests to /products/** => forwarded to PRODUCT_SERVICE_URL
// Requests to /orders/**   => forwarded to ORDER_SERVICE_URL
```

### 4.2 Structured Request Logging

Every proxied request is logged with: timestamp, method, path, target service, status code, and response latency (ms).

### 4.3 Health Checks

- `GET /health` — returns gateway uptime and registered service URLs
- `GET /health?probe=true` — actively pings all three downstream services and reports their live status

### 4.4 Centralized Error Handling

If a downstream service is unreachable (ECONNREFUSED / ETIMEDOUT), the gateway returns a clean JSON response:

```json
{
  "success": false,
  "error": "Service Unavailable",
  "statusCode": 503,
  "service": "User Service",
  "code": "ECONNREFUSED",
  "message": "The downstream microservice is currently offline or unreachable.",
  "requestedPath": "/users",
  "timestamp": "2026-10-04T07:16:00.000Z"
}
```

---

## 5. Service Discovery — Config-Based Approach

The gateway discovers downstream services entirely through **environment variables**, not hardcoded URLs:

```env
USER_SERVICE_URL=http://user-service:3001
PRODUCT_SERVICE_URL=http://product-service:3002
ORDER_SERVICE_URL=http://order-service:3003
```

This is **Configuration-Based (Static) Service Discovery**. The routing table is built at startup from these variables.

### Static vs. Dynamic Discovery

| Criterion | Config-Based (This Lab) | Dynamic (Consul / K8s DNS) |
|---|---|---|
| Setup Complexity | Minimal - just env vars | High - needs dedicated registry |
| Auto Scaling | No - fixed endpoints | Yes - instances self-register |
| Health Monitoring | Passive (on-request) | Active heartbeats |
| Failure Recovery | Manual redeploy | Automatic rerouting |
| Best Fit | Docker Compose / small stacks | Kubernetes / large deployments |

---

## 6. API Endpoint Reference

All requests go through the **API Gateway**. Replace `BASE_URL` with:
- **Local:** `http://localhost:8080`
- **Cloud:** `https://campusconnect-api-gateway-c37d.onrender.com`

### Gateway

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Gateway status + registered service URLs |
| `GET` | `/health?probe=true` | Live probe all downstream services |
| `GET` | `/` | Route map overview |

### Users (`/users`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/users` | List all users |
| `GET` | `/users/:id` | Get user by ID |
| `POST` | `/users` | Create new user |
| `PUT` | `/users/:id` | Update user |
| `DELETE` | `/users/:id` | Delete user |

### Products (`/products`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/products` | List all products |
| `GET` | `/products/:id` | Get product by ID |
| `POST` | `/products` | Add product |
| `PUT` | `/products/:id` | Update product |
| `DELETE` | `/products/:id` | Delete product |

### Orders (`/orders`)

| Method | Path | Description |
|---|---|---|
| `POST` | `/orders` | Create order (validates user + product via inter-service calls) |
| `GET` | `/orders` | List all orders |
| `GET` | `/orders/:id` | Get order by ID |

---

## 7. Cloud Deployment (Render + MongoDB Atlas)

### How Render Resolves Inter-Service URLs

The `render.yaml` uses `fromService.property: host` to auto-inject sibling service hostnames:

```yaml
services:
  - type: web
    name: campusconnect-api-gateway
    env: docker
    dockerfilePath: ./api-gateway/Dockerfile
    envVars:
      - key: USER_SERVICE_URL
        fromService:
          name: campusconnect-user-service
          property: host
      - key: PRODUCT_SERVICE_URL
        fromService:
          name: campusconnect-product-service
          property: host
      - key: ORDER_SERVICE_URL
        fromService:
          name: campusconnect-order-service
          property: host
```

### MongoDB Atlas — MONGO_URI per Service

| Service | Database |
|---|---|
| user-service | `mongodb+srv://...@cluster.../user_db` |
| product-service | `mongodb+srv://...@cluster.../product_db` |
| order-service | `mongodb+srv://...@cluster.../order_db` |

These are set manually in the Render Dashboard under each service's Environment tab.

---

## 8. Running Locally with Docker Compose

### Prerequisites
- Docker Desktop installed and running
- No processes occupying ports `8080` or `27017`

### Start the Stack

```powershell
# From the project root
docker compose up --build -d

# Check all containers are running
docker compose ps

# Tail gateway logs
docker compose logs -f api-gateway
```

### Expected Containers

```
NAME               STATUS    PORTS
api-gateway        Up        0.0.0.0:8080->8080/tcp
user-service       Up        (internal: 3001)
product-service    Up        (internal: 3002)
order-service      Up        (internal: 3003)
mongodb            Up        0.0.0.0:27017->27017/tcp
```

### Stop

```powershell
docker compose down

# Also wipe data volumes:
docker compose down -v
```

---

## 9. Testing the Gateway

### Health Check

```powershell
# Local
curl http://localhost:8080/health

# Cloud
curl https://campusconnect-api-gateway-c37d.onrender.com/health
```

### PowerShell Automated Test Suite

```powershell
# Local testing
.\test_gateway.ps1

# Cloud testing
.\test_gateway.ps1 -GatewayUrl "https://campusconnect-api-gateway-c37d.onrender.com"
```

Validates: health endpoint, all CRUD routes, inter-service order validation, network isolation, and fault tolerance (503 on service kill + auto-recovery).

### Sample curl Commands

```powershell
# Create a user
curl -X POST http://localhost:8080/users `
  -H "Content-Type: application/json" `
  -d '{"name":"Kush Dani","email":"kush@campus.in","role":"student"}'

# Add a product
curl -X POST http://localhost:8080/products `
  -H "Content-Type: application/json" `
  -d '{"name":"Laptop","price":45000,"category":"Electronics","stock":10}'

# Place an order (validates user + product internally)
curl -X POST http://localhost:8080/orders `
  -H "Content-Type: application/json" `
  -d '{"userId":"<user_id>","productId":"<product_id>","quantity":1}'

# Verify isolation - this should FAIL (connection refused)
curl http://localhost:3001/users
```

### Postman Collection

```
postman/CampusConnect_Lab7_API_Gateway.postman_collection.json
```

Set collection variable `gateway_url`:
- Local: `http://localhost:8080`
- Cloud: `https://campusconnect-api-gateway-c37d.onrender.com`

---

## 10. Discussion & Analysis

### Part A: Why an API Gateway over Direct Client-to-Service Communication?

Allowing clients to call microservices directly creates tight coupling and significant operational risk:

**1. Topology Hiding**
Clients interact with one stable URL. The internal layout (IPs, ports, service names) is completely opaque — enabling backend migrations and scaling without breaking consumers.

**2. Reduced Attack Surface**
Only port 8080 is internet-facing. Ports 3001-3003 live exclusively inside the Docker bridge network. A compromised microservice cannot be directly probed or exploited from outside.

**3. Single Place for Cross-Cutting Concerns**
Structured logging, CORS, rate limiting, SSL termination, and auth — written once at the gateway, not duplicated across each service in different languages.

**4. Graceful Fault Isolation**
When a service crashes, the gateway intercepts the raw ECONNREFUSED and returns a clean `503 Service Unavailable` JSON response instead of exposing raw TCP failures to clients.

**5. Composability**
Gateways can aggregate data from multiple services into one response (Backend for Frontend pattern) or translate between protocols (REST to gRPC).

### Part B: Config-Based vs. Dynamic Service Discovery

This lab implements **static, environment variable-based discovery** — the appropriate choice for a fixed-topology Docker Compose / Render deployment.

Dynamic registries (Consul, Netflix Eureka, Kubernetes DNS) are justified when:
- Containers have ephemeral, dynamically assigned IPs (Kubernetes pods)
- Services auto-scale horizontally to unknown instance counts
- Zero-downtime canary/blue-green deployments require live traffic weight shifting
- Services span multiple regions or clouds

For our 4-service, fixed-hostname stack (Docker DNS resolves `user-service`, `product-service`, `order-service` by container name), static discovery is not a limitation — it is the pragmatic, zero-overhead right tool for the job.

---

## 11. Project Structure

```
202512023_Campus_Connect_Microservices/
|
+-- api-gateway/                   # Express.js reverse proxy gateway
|   +-- src/
|   |   +-- app.js                 # Main gateway app + proxy mounts
|   |   +-- config/services.js     # Service registry (env-var driven)
|   |   +-- middleware/
|   |   |   +-- logger.js          # Structured request logger
|   |   |   +-- errorHandler.js    # 502/503 fault interceptor
|   |   +-- routes/health.js       # Health check + live probe endpoint
|   +-- Dockerfile
|   +-- package.json
|
+-- user-service/                  # User CRUD microservice (:3001)
+-- product-service/               # Product catalog microservice (:3002)
+-- order-service/                 # Order orchestration microservice (:3003)
|                                  # (calls user-service + product-service internally)
+-- postman/                       # Postman collection
+-- compose.yaml                   # Docker Compose (local dev)
+-- render.yaml                    # Render Blueprint (cloud deployment)
+-- test_gateway.ps1               # Automated gateway tests (PowerShell)
+-- test_microservices.ps1         # Individual service tests
+-- .env.example                   # Environment variable template
+-- README.md                      # This file
```

---

## 12. Key Learnings

- Microservices decomposition reduces blast radius — a failed Product Service does not crash User or Order services
- The API Gateway pattern enforces a clean boundary between public-facing API and internal service topology
- Docker network isolation (`expose` vs `ports`) is a simple but powerful security mechanism
- Config-based service discovery with Docker Compose DNS is elegant and requires zero extra tooling
- Cloud deployment with Render + MongoDB Atlas proves a multi-service architecture can run on free-tier infrastructure with minimal manual setup

---

*Lab 6 & 7 | Web Services & SOA | Student ID: 202512023*
*GitHub: https://github.com/KushDani23/202512023_Campus_Connect_Microservices*
