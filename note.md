<div align="center">
  <p style="color: red; font-size: 1.4em; font-weight: bold; border: 2px solid red; padding: 14px; border-radius: 6px;">
    🚫 RESTRICTED — DO NOT COPY 🚫<br/>
    <span style="font-size: 0.85em; font-weight: normal;">
      This document is for personal study purposes only.<br/>
      Copying, reproducing, sharing, or distributing any content from this material is strictly prohibited.<br/><br/>
      ⚠️ <strong>Any unauthorized use of this content will result in immediate legal action.</strong><br/>
      <span style="font-size: 0.8em;">The author reserves all rights and will pursue copyright infringement claims to the fullest extent of the law.</span>
    </span>
  </p>
</div>

# Docker Study Notes — Nginx Reverse Proxy & Load Balancing

> Detailed study notes on Nginx as a reverse proxy: what a reverse proxy is, why it matters, how it works internally, its core benefits (load balancing, security, SSL termination, routing, fault tolerance), and a full hands-on capstone project using Docker Compose that spins up three Node.js app instances behind an Nginx load balancer.

---

## Table of Contents

1. [What Is a Reverse Proxy?](#1-what-is-a-reverse-proxy)
2. [What Is a Reverse Proxy Used For?](#2-what-is-a-reverse-proxy-used-for)
3. [How Does a Reverse Proxy Work?](#3-how-does-a-reverse-proxy-work)
4. [Types of Reverse Proxies](#4-types-of-reverse-proxies)
5. [Examples of Reverse Proxies](#5-examples-of-reverse-proxies)
6. [Reverse Proxy Benefits](#6-reverse-proxy-benefits)
7. [Nginx as a Reverse Proxy](#7-nginx-as-a-reverse-proxy)
8. [Load Balancing Algorithms in Nginx](#8-load-balancing-algorithms-in-nginx)
9. [Hands-on: Nginx Load Balancer with Docker Compose](#9-hands-on-nginx-load-balancer-with-docker-compose)
10. [Project File Walkthrough](#10-project-file-walkthrough)
11. [Recap](#11-recap)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. What Is a Reverse Proxy?

A **reverse proxy** is a server that sits **in front of one or more backend servers** and forwards client requests to them on their behalf. The client never communicates with the backend directly — it only ever talks to the proxy.

```
Without a Reverse Proxy:

  Client ──────────────────────────► Backend Server (exposed directly)
  (knows the backend's IP/port)


With a Reverse Proxy:

  Client ──────► Reverse Proxy ──────► Backend Server (hidden)
  (only knows the proxy's address)
```

### Forward Proxy vs Reverse Proxy

These two are often confused. The key difference is **who the proxy represents**:

|                 | Forward Proxy                     | Reverse Proxy                   |
| --------------- | --------------------------------- | ------------------------------- |
| **Represents**  | The **client**                    | The **server**                  |
| **Hides**       | Client identity from the internet | Backend servers from the client |
| **Typical use** | VPNs, bypassing geo-blocks        | Load balancing, SSL termination |
| **Direction**   | Client → Proxy → Internet         | Internet → Proxy → Server       |
| **Example**     | Corporate web filter              | Nginx in front of your API      |

> Think of a reverse proxy like a **receptionist** at a company — callers never know which employee handles their request. The receptionist (proxy) routes the call internally.

---

## 2. What Is a Reverse Proxy Used For?

Reverse proxies are used in virtually every production web application. Here are the core use cases:

| Use Case                    | Why It's Needed                                                                        |
| --------------------------- | -------------------------------------------------------------------------------------- |
| **Load Balancing**          | Distribute traffic across multiple server instances so no single server is overwhelmed |
| **Security**                | Hide backend server addresses, block malicious requests, enforce rate limiting         |
| **Single Entry Point**      | One public domain/IP routes to dozens of internal microservices                        |
| **HTTPS / SSL Termination** | Handle TLS certificates once at the proxy; backend servers use plain HTTP internally   |
| **Performance**             | Cache static assets, compress responses — reduce load on backends                      |
| **Routing**                 | `/api/` → service A, `/auth/` → service B, `/static/` → CDN                            |
| **Scaling**                 | Add more backend instances without changing DNS or client configuration                |
| **Fault Tolerance**         | If one backend goes down, the proxy routes traffic to healthy instances                |

---

## 3. How Does a Reverse Proxy Work?

### The Request Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  1. Client sends HTTP request to https://myapp.com/api/users        │
│                                                                     │
│  2. DNS resolves myapp.com → Nginx's IP address                     │
│                                                                     │
│  3. Nginx receives the request on port 80 / 443                     │
│                                                                     │
│  4. Nginx inspects the path, headers, and load-balancing policy     │
│                                                                     │
│  5. Nginx forwards the request to a selected backend:               │
│       http://app1:3000/api/users  (or app2 or app3...)              │
│                                                                     │
│  6. Backend processes the request and returns a response            │
│                                                                     │
│  7. Nginx receives the backend response                             │
│                                                                     │
│  8. Nginx forwards the response back to the client                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### What the Client Sees vs What Really Happens

```
Client's perspective:
  GET https://myapp.com/  →  Response: "Hello from App-2"
  (thinks it's talking to one server)

What actually happened:
  Client → Nginx (port 80) → app2:3000 (hidden, internal)
                           ← response ←
  Nginx → Client
```

The client never knows there are three backend servers. It always sees a single address.

### Key Headers Nginx Adds

When proxying, Nginx adds headers so the backend knows real client information:

| Header              | Value                | Purpose                                       |
| ------------------- | -------------------- | --------------------------------------------- |
| `X-Real-IP`         | Client's actual IP   | Backend logging & security                    |
| `X-Forwarded-For`   | Client IP chain      | Identifies original requester through proxies |
| `X-Forwarded-Proto` | `https` or `http`    | Lets backend know the original protocol       |
| `Host`              | Original host header | Backend knows which domain was requested      |

---

## 4. Types of Reverse Proxies

```
┌─────────────────────────────────────────────────────────────────────┐
│                    REVERSE PROXY TYPES                              │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Layer 7 (Application-Level)                                │   │
│  │  Reads HTTP content: URL, headers, cookies                  │   │
│  │  Can route by path, host, query params                      │   │
│  │  Examples: Nginx, HAProxy (HTTP mode), Traefik              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Layer 4 (Transport-Level)                                  │   │
│  │  Routes by IP + TCP/UDP port only — no HTTP awareness       │   │
│  │  Faster, but cannot inspect request content                 │   │
│  │  Examples: HAProxy (TCP mode), AWS NLB                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Cloud / CDN Proxies                                        │   │
│  │  Globally distributed edge nodes                           │   │
│  │  DDoS protection, caching at the edge                      │   │
│  │  Examples: Cloudflare, AWS CloudFront, Fastly              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

| Type           | Routing Based On      | Content Aware | Example Use Case               |
| -------------- | --------------------- | ------------- | ------------------------------ |
| Layer 7 (HTTP) | URL, headers, cookies | ✅ Yes        | Microservice path routing      |
| Layer 4 (TCP)  | IP + port only        | ❌ No         | Database proxying, raw TCP     |
| CDN / Edge     | Geographic proximity  | ✅ Yes        | Global static asset delivery   |
| API Gateway    | API routes + auth     | ✅ Yes        | Rate limiting + auth + routing |

---

## 5. Examples of Reverse Proxies

| Tool           | Type          | Strengths                                            | Common Use                                |
| -------------- | ------------- | ---------------------------------------------------- | ----------------------------------------- |
| **Nginx**      | Layer 7       | High performance, simple config, static file serving | Web servers, load balancers, API gateways |
| **HAProxy**    | Layer 4 & 7   | Extreme throughput, detailed health checks           | TCP/HTTP load balancing at scale          |
| **Traefik**    | Layer 7       | Auto-discovers Docker/Kubernetes services            | Cloud-native, dynamic configs             |
| **Caddy**      | Layer 7       | Automatic HTTPS by default                           | Developer-friendly web server             |
| **AWS ALB**    | Layer 7       | Managed, integrates with AWS                         | AWS-hosted applications                   |
| **Cloudflare** | CDN + Layer 7 | DDoS, edge caching, WAF                              | Public-facing websites                    |
| **Kong**       | API Gateway   | Plugins for auth, rate-limiting                      | API management                            |

> **This project uses Nginx** — the most widely used open-source reverse proxy/load balancer.

---

## 6. Reverse Proxy Benefits

### 1. Load Balancing

Distributes incoming requests across multiple backend server instances, preventing any single server from becoming a bottleneck.

```
Before Load Balancing:

  All Traffic ──────────────────────────► Single Server
                                           (overwhelmed, crashes under load)

After Load Balancing:

  Traffic ──► Nginx ──► App-1 (33% of requests)
                    ──► App-2 (33% of requests)
                    ──► App-3 (34% of requests)
```

### 2. Security — Hides Backend Servers

Clients only ever see the proxy's address. Your actual application servers are completely hidden on an internal network.

```
Public Internet:
  Knows only: nginx → 203.0.113.5:80

Internal Network (private):
  app1:3000  ← never exposed publicly
  app2:3000  ← never exposed publicly
  app3:3000  ← never exposed publicly
```

Additional security benefits:

- Nginx can block requests by IP, User-Agent, or request rate
- Web Application Firewall (WAF) rules can be applied at the proxy
- DDoS mitigation at the edge before requests hit your app

### 3. Single Entry Point — One Domain for All Services

Without a reverse proxy, each service needs its own port or domain. With Nginx, you get clean URL routing:

```
Without Nginx (ugly, impractical for users):
  http://myapp.com:3001  → User Service
  http://myapp.com:3002  → Order Service
  http://myapp.com:3003  → Auth Service

With Nginx (clean, professional):
  http://myapp.com/api/users   → User Service (port 3001 internally)
  http://myapp.com/api/orders  → Order Service (port 3002 internally)
  http://myapp.com/auth        → Auth Service (port 3003 internally)
```

### 4. HTTPS / SSL Termination

Instead of configuring TLS certificates on every backend server, you handle SSL once at Nginx. Backend servers can use plain HTTP internally.

```
Client ──[HTTPS/TLS]──► Nginx ──[HTTP, internal]──► app1:3000
                         │
                   SSL cert lives here only
                   (one cert to manage, not N)
```

```nginx
# Nginx handles TLS — backend servers don't need certificates
server {
    listen 443 ssl;
    ssl_certificate     /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://backend;  # plain HTTP to the app
    }
}
```

### 5. Better Performance — Caching & Compression

Nginx can serve cached responses for repeated requests and compress response bodies, dramatically reducing bandwidth and backend load.

```
Request 1: Client → Nginx → Backend (cache miss) → response stored in cache
Request 2: Client → Nginx → Cache Hit ✅ (backend not touched)
Request 3: Client → Nginx → Cache Hit ✅ (backend not touched)
```

```nginx
# Enable gzip compression
gzip on;
gzip_types text/plain application/json application/javascript text/css;

# Enable proxy caching
proxy_cache_path /tmp/nginx_cache levels=1:2 keys_zone=my_cache:10m;
```

### 6. Routing to Different Services

Nginx can route traffic to entirely different backend services based on URL path, subdomain, or request headers.

```nginx
# Path-based routing
location /api/ {
    proxy_pass http://api-service:3000;
}

location /auth/ {
    proxy_pass http://auth-service:4000;
}

location /static/ {
    root /var/www/html;  # serve files directly from disk
}

# Subdomain-based routing
server {
    server_name api.myapp.com;
    location / { proxy_pass http://api-service:3000; }
}

server {
    server_name admin.myapp.com;
    location / { proxy_pass http://admin-service:5000; }
}
```

### 7. Easy Scaling of Services

Add or remove backend instances without changing anything the client sees. Just update Nginx's upstream block and reload — zero downtime.

```nginx
# Scale from 2 to 4 instances — client sees no change
upstream backend {
    server app1:3000;
    server app2:3000;
    server app3:3000;  # ← added
    server app4:3000;  # ← added
}
```

```bash
# Hot-reload Nginx config with zero downtime
nginx -s reload
```

### 8. Fault Tolerance — Handles Server Failure

Nginx automatically detects unhealthy backends and stops sending traffic to them. When a backend recovers, Nginx resumes routing to it.

```
Normal:
  Requests → app1 ✅ / app2 ✅ / app3 ✅

app2 crashes:
  Nginx detects connection failure to app2
  Requests → app1 ✅ / app3 ✅ (app2 skipped automatically)

app2 recovers:
  Requests → app1 ✅ / app2 ✅ / app3 ✅ (automatically rejoins)
```

```nginx
upstream backend {
    server app1:3000 max_fails=3 fail_timeout=30s;
    server app2:3000 max_fails=3 fail_timeout=30s;
    server app3:3000 max_fails=3 fail_timeout=30s;
}
```

---

## 7. Nginx as a Reverse Proxy

### What Is Nginx?

Nginx (pronounced "engine-x") is an open-source, high-performance HTTP server, reverse proxy, and load balancer. Originally built to solve the **C10K problem** (handling 10,000 concurrent connections), it uses an **asynchronous, event-driven architecture** — unlike Apache's thread-per-request model.

```
Apache (thread-per-request):           Nginx (event-driven):

  Request 1 → Thread 1 (10 MB RAM)      Request 1 ─┐
  Request 2 → Thread 2 (10 MB RAM)      Request 2 ─┤─► Single Worker Process
  Request 3 → Thread 3 (10 MB RAM)      Request 3 ─┘   (handles all via event loop)
  ...1000 requests = 10 GB RAM          ...1000 requests = very little RAM
```

### Core Nginx Configuration Structure

```
/etc/nginx/
├── nginx.conf          ← main config file (global settings)
├── conf.d/             ← additional config files (included automatically)
│   └── default.conf
└── sites-enabled/      ← virtual host configs (on Debian/Ubuntu)
```

### The `nginx.conf` File — Key Blocks

```nginx
# Global settings (applies to the entire Nginx process)
worker_processes auto;

# Event handling settings
events {
    worker_connections 1024;  # max simultaneous connections per worker
}

# HTTP server block
http {

    # Upstream: defines a pool of backend servers
    upstream backend {
        server app1:3000;
        server app2:3000;
        server app3:3000;
    }

    # Server: defines a virtual host (one domain / port)
    server {
        listen 80;
        server_name myapp.com;

        # Location: matches URL paths and defines what to do
        location / {
            proxy_pass http://backend;  # forward to the upstream pool
        }
    }
}
```

---

## 8. Load Balancing Algorithms in Nginx

Nginx supports several strategies for distributing requests across backend servers:

### Round Robin (Default)

Requests are sent to each server in turn, in order. This is what this project uses.

```
Request 1 → app1
Request 2 → app2
Request 3 → app3
Request 4 → app1  (cycles back)
Request 5 → app2
...
```

```nginx
upstream backend {
    # No directive needed — round robin is the default
    server app1:3000;
    server app2:3000;
    server app3:3000;
}
```

### Least Connections

Routes each request to the backend currently handling the fewest active connections.

```nginx
upstream backend {
    least_conn;
    server app1:3000;
    server app2:3000;
    server app3:3000;
}
```

Best for: requests with highly variable processing time (some fast, some slow).

### IP Hash

Consistently routes all requests from the same client IP to the same backend server. Useful for session persistence (sticky sessions).

```nginx
upstream backend {
    ip_hash;
    server app1:3000;
    server app2:3000;
    server app3:3000;
}
```

Best for: stateful applications where a user must always hit the same server.

### Weighted Round Robin

Some servers receive more traffic than others, proportional to their weight.

```nginx
upstream backend {
    server app1:3000 weight=3;   # gets 3x as many requests
    server app2:3000 weight=1;
    server app3:3000 weight=1;
}
```

Best for: mixed hardware — route more to powerful servers.

### Algorithm Comparison

| Algorithm             | Config Directive          | Best For                              |
| --------------------- | ------------------------- | ------------------------------------- |
| **Round Robin**       | _(default, no directive)_ | Uniform requests, identical servers   |
| **Least Connections** | `least_conn;`             | Variable request processing time      |
| **IP Hash**           | `ip_hash;`                | Session persistence (sticky sessions) |
| **Weighted**          | `weight=N`                | Servers with different capacities     |

---

## 9. Hands-on: Nginx Load Balancer with Docker Compose

### Project Goal

Run **three identical Node.js Express app instances** (`App-1`, `App-2`, `App-3`) behind a single Nginx reverse proxy. Nginx load-balances requests across all three using round-robin.

```
                        ┌──────────────────────────┐
                        │        Docker Network     │
                        │                          │
  curl localhost:8080   │  ┌──────────────────────┐│
         │              │  │  Nginx (port 80)      ││
         ▼              │  │  exposed on 8080      ││
  Host:8080 ─────────►  │  └──────────┬───────────┘│
                        │             │            │
                        │    ┌────────┼────────┐   │
                        │    ▼        ▼        ▼   │
                        │  app1     app2     app3   │
                        │ :3000    :3000    :3000   │
                        └──────────────────────────┘
```

### Step 1 — Project Structure

```
l2-M-59/
├── docker-compose.yml       ← orchestrates all services
├── nginx/
│   └── nginx.conf           ← Nginx load balancer config
└── app/
    ├── Dockerfile           ← builds the Node.js app image
    ├── package.json
    └── server.js            ← Express app that reports its instance name
```

### Step 2 — The Node.js App (`app/server.js`)

Each instance reads its name from an environment variable and includes it in every response. This lets us **verify which instance handled each request**.

```javascript
const express = require("express");
const os = require("os");

const app = express();
const PORT = 3000;

// Read instance name from Docker Compose environment variable
const INSTANCE = process.env.INSTANCE_NAME || os.hostname();

app.get("/", (req, res) => {
  res.send(`Hello World from ${INSTANCE}`);
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT} - ${INSTANCE}`);
});
```

### Step 3 — The Dockerfile (`app/Dockerfile`)

```dockerfile
FROM node:18

WORKDIR /app

COPY package*.json ./
RUN npm install          # installs Express

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

### Step 4 — Nginx Configuration (`nginx/nginx.conf`)

```nginx
events {}              # required block (even if empty)

http {

    # Define the load-balanced pool of backend servers
    upstream backend {
        server app1:3000;   # Docker Compose service name resolves to container IP
        server app2:3000;
        server app3:3000;
    }

    server {
        listen 80;          # Nginx listens on port 80 inside the container

        location / {
            proxy_pass http://backend;  # forward all requests to the pool
        }
    }
}
```

**Key points:**

- `upstream backend` — defines the pool; round-robin by default
- `server app1:3000` — `app1` is resolved via Docker's internal DNS (same as container-to-container networking)
- `proxy_pass http://backend` — every request to `/` is forwarded to one of the pool members

### Step 5 — Docker Compose (`docker-compose.yml`)

```yaml
version: "3.9"

services:
  # Three identical app instances — only INSTANCE_NAME differs
  app1:
    build: ./app
    environment:
      - INSTANCE_NAME=App-1

  app2:
    build: ./app
    environment:
      - INSTANCE_NAME=App-2

  app3:
    build: ./app
    environment:
      - INSTANCE_NAME=App-3

  nginx:
    image: nginx:latest
    ports:
      - "8080:80" # host port 8080 → container port 80
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf # mount our custom config
    depends_on:
      - app1
      - app2
      - app3 # Nginx starts only after all app instances are up
```

**Important notes:**

- `app1`, `app2`, `app3` have **no `ports:` mapping** — they are intentionally not exposed to the host. Only Nginx is exposed.
- Docker Compose automatically creates a shared network, so `app1`, `app2`, `app3` are resolvable by name inside Nginx's container.

### Step 6 — Build and Run

```bash
# Build images and start all containers
docker compose up --build

# Or run in detached mode (background)
docker compose up --build -d
```

### Step 7 — Test the Load Balancer

```bash
# Send a single request
curl http://localhost:8080/
# Response: "Hello World from App-1"

# Send multiple requests — watch round-robin in action
curl http://localhost:8080/    # → Hello World from App-1
curl http://localhost:8080/    # → Hello World from App-2
curl http://localhost:8080/    # → Hello World from App-3
curl http://localhost:8080/    # → Hello World from App-1  (cycles back)
```

Open `http://localhost:8080` in your browser and keep refreshing — you'll see the instance name cycle through all three.

### What Happens Inside Docker

```
1. Browser/curl: GET http://localhost:8080/
         │
         │  host port 8080 → nginx container port 80
         ▼
2. Nginx receives request
         │
         │  upstream backend: picks next server (round-robin)
         │  this turn: app2:3000
         ▼
3. Nginx: proxy_pass http://app2:3000/
         │
         │  Docker DNS resolves "app2" → 172.x.x.x
         ▼
4. app2 container receives request
         │
         │  res.send("Hello World from App-2")
         ▼
5. Response flows back: app2 → Nginx → Host → Browser

6. Next request: Nginx picks app3:3000 (round-robin continues)
```

### Step 8 — Useful Commands

```bash
# View logs from all services
docker compose logs -f

# View logs from Nginx only
docker compose logs -f nginx

# View logs from app1 only
docker compose logs -f app1

# Stop all containers
docker compose down

# Stop and remove volumes
docker compose down -v

# Rebuild a specific service
docker compose up --build app1

# Scale app instances on-the-fly (without editing docker-compose.yml)
docker compose up --scale app1=2
```

---

## 10. Project File Walkthrough

### How All Files Connect

```
docker-compose.yml
  ├── builds app1, app2, app3 from ./app/Dockerfile
  │     └── app/server.js  ← Express app reporting INSTANCE_NAME
  │
  └── runs nginx:latest
        └── mounts nginx/nginx.conf into container
              └── upstream backend → routes to app1:3000, app2:3000, app3:3000
                    (resolved via Docker Compose's internal DNS)
```

### Why No `ports:` on App Services?

```yaml
# app1, app2, app3 — intentionally NOT exposed:
app1:
  build: ./app
  environment:
    - INSTANCE_NAME=App-1
  # NO ports: mapping here

# Nginx IS exposed:
nginx:
  ports:
    - "8080:80"
```

This is the correct security posture for a reverse proxy setup:

- Only the proxy is reachable from outside the Docker network
- Backend servers are on a private internal network
- Clients cannot bypass the proxy to hit backends directly

### Docker Compose Network

Docker Compose automatically creates a **default bridge network** named `<project>_default`. All services join it automatically, which is why Nginx can reach `app1:3000` by name — Docker's internal DNS resolves service names to container IPs.

```
Docker Compose Network (l2-m-59_default):
  ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
  │    app1    │   │    app2    │   │    app3    │   │   nginx    │
  │ 172.x.0.2  │   │ 172.x.0.3  │   │ 172.x.0.4  │   │ 172.x.0.5  │
  └────────────┘   └────────────┘   └────────────┘   └────────────┘
        ← all on the same virtual network, resolvable by service name →
```

---

## 11. Recap

### The Reverse Proxy Mental Model

```
┌────────────────────────────────────────────────────────────────────┐
│  WITHOUT Reverse Proxy                                             │
│                                                                    │
│  Client → app1:3001 (exposed)                                      │
│  Client → app2:3002 (exposed)                                      │
│  Client → app3:3003 (exposed)                                      │
│  Each service needs its own port, cert, and firewall rule          │
├────────────────────────────────────────────────────────────────────┤
│  WITH Nginx Reverse Proxy                                          │
│                                                                    │
│  Client → nginx:80 → app1:3000 (hidden)                            │
│                    → app2:3000 (hidden)   ← round-robin            │
│                    → app3:3000 (hidden)                            │
│  One entry point, one cert, one firewall rule. Backends invisible. │
└────────────────────────────────────────────────────────────────────┘
```

### Eight Benefits — Quick Summary

| #   | Benefit                     | What It Solves                       |
| --- | --------------------------- | ------------------------------------ |
| 1   | **Load Balancing**          | No single server overloaded          |
| 2   | **Security**                | Backend IPs never exposed            |
| 3   | **Single Entry Point**      | One domain for all services          |
| 4   | **HTTPS / SSL Termination** | TLS handled once, not per-service    |
| 5   | **Better Performance**      | Caching + compression at the edge    |
| 6   | **Routing**                 | Path/subdomain → different services  |
| 7   | **Easy Scaling**            | Add instances without client changes |
| 8   | **Fault Tolerance**         | Auto-skips unhealthy backends        |

---

## 12. Quick Reference Cheat Sheet

### Docker Compose Commands

| Command                        | What It Does                              |
| ------------------------------ | ----------------------------------------- |
| `docker compose up --build`    | Build images and start all services       |
| `docker compose up --build -d` | Same, but in background (detached)        |
| `docker compose down`          | Stop and remove containers                |
| `docker compose logs -f`       | Stream logs from all services             |
| `docker compose logs -f nginx` | Stream logs from Nginx only               |
| `docker compose ps`            | List running services                     |
| `docker compose restart nginx` | Restart Nginx (e.g., after config change) |

---

### Nginx `nginx.conf` Quick Reference

```nginx
events {}

http {

    # --- Load Balancing Algorithms ---
    upstream backend {
        # Round Robin (default — no directive needed)
        server app1:3000;
        server app2:3000;
        server app3:3000;

        # Least Connections:     least_conn;
        # IP Hash:               ip_hash;
        # Weighted:              server app1:3000 weight=3;
        # Health check:          server app1:3000 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 80;
        server_name _;          # _ matches any hostname

        location / {
            proxy_pass         http://backend;

            # Pass real client info to backend:
            proxy_set_header   Host              $host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
        }
    }
}
```

---

### Load Balancing Algorithm Cheat Sheet

| Algorithm         | Directive     | When to Use                     |
| ----------------- | ------------- | ------------------------------- |
| Round Robin       | _(default)_   | Equal servers, uniform requests |
| Least Connections | `least_conn;` | Variable request times          |
| IP Hash           | `ip_hash;`    | Session persistence required    |
| Weighted          | `weight=N`    | Mixed server capacities         |

---

### Reverse Proxy Pattern Cheat Sheet

| Pattern           | Config Approach                | Use Case               |
| ----------------- | ------------------------------ | ---------------------- |
| Load Balancing    | `upstream` + `proxy_pass`      | Scale horizontally     |
| Path Routing      | Multiple `location` blocks     | Microservices          |
| SSL Termination   | `listen 443 ssl` + cert paths  | HTTPS on one place     |
| Static Files      | `root` directive in `location` | Serve assets from disk |
| Subdomain Routing | Multiple `server` blocks       | Multi-tenant apps      |

---

### Development Workflow for This Project

```bash
# Build and start everything
docker compose up --build

# Test round-robin load balancing
for i in 1 2 3 4 5 6; do curl http://localhost:8080/; echo; done

# Expected output:
# Hello World from App-1
# Hello World from App-2
# Hello World from App-3
# Hello World from App-1
# Hello World from App-2
# Hello World from App-3

# Stop everything
docker compose down
```

---

## Load Balancing — Key Concepts

Load balancing is the process of distributing incoming network traffic across multiple servers so that no single server bears too much demand.

### Why Load Balancing Matters

1. **Distributes traffic across multiple servers** — Incoming requests are spread evenly (or intelligently) across a pool of backend servers, so no single instance handles everything alone.

2. **Prevents server overload** — When traffic spikes, the load balancer ensures each server only receives a manageable share of requests, avoiding crashes or slowdowns caused by resource exhaustion.

3. **Improves performance** — Requests are routed to the least-busy or nearest server, reducing response times and keeping the application feeling fast for every user.

4. **Ensures high availability** — If one server goes down, the load balancer automatically stops sending traffic to it and redirects requests to the remaining healthy servers — no downtime for end users.

5. **Allows horizontal scaling** — New server instances can be added to the pool at any time (without changing the application or DNS) to handle growing traffic, making scale-out simple and non-disruptive.

6. **Increases system reliability** — By eliminating single points of failure and spreading risk across multiple nodes, the overall system becomes more resilient to hardware failures, software bugs, and unexpected traffic surges.

```
Without Load Balancing:          With Load Balancing:

  Client ──► Single Server         Client ──► Load Balancer ──► Server 1
             (bottleneck,                                    ──► Server 2
              single point                                   ──► Server 3
              of failure)                    (resilient, scalable, fast)
```

---

<div align="center">
  <p style="font-size: 1.1em; font-weight: bold;">Habibur Rahman Zihad</p>
  <p style="color: gray; font-size: 0.95em;">Full-Stack Developer</p>
  <p style="color: gray; font-size: 0.85em;">© 2026 All Rights Reserved</p>
</div>
