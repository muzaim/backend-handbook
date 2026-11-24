# Kong API Gateway

## 1. Simple Analogy & Real-World Scenario

Imagine a busy international airport:

Passengers arriving from all over the world (incoming HTTP requests from mobile apps and web frontends) cannot simply stroll directly into the city (internal backend microservices such as *Order Service*, *Payment Service*, or *User Service*). Before entering, all passengers must funnel through a centralized immigration and customs checkpoint:
1. **Passport & Visa Verification (Authentication & Authorization)**: Officers verify whether the traveler holds valid identity credentials.
2. **Security Scanners (Security & WAF)**: Ensuring no illicit contraband, weapons, or dangerous exploits (SQLi, DDoS, payload tampering) enter.
3. **Queue Sorting & Gate Routing (Traffic Routing & Load Balancing)**: Directing diplomat/priority travelers versus general economy passengers to the designated transit terminal.
4. **Immigration Logbooks (Logging & Metrics)**: Officials record the timestamp and volume of entries per minute.

If every office building in the city were forced to erect its own customs checkpoint and X-ray scanners, security costs would skyrocket and compliance standards would fragment. **Kong Gateway** serves as that standardized, high-performance international checkpoint for your entire backend microservices architecture.

---

## 2. Definition & Core Concepts

**Kong Gateway** is an open-source, cloud-native API Gateway and Ingress controller engineered on top of **NGINX** and the **OpenResty (LuaJIT)** engine. Kong acts as an intelligent intermediary reverse proxy sitting at the boundary of your infrastructure to handle cross-cutting platform concerns.

Kong operates around five fundamental entities:
1. **Service**: An abstraction representing your upstream microservice (e.g., `order-service` listening at `http://order-backend:8080`).
2. **Route**: Matching rules for incoming client requests (e.g., matching the path prefix `/api/v1/orders` or host header `api.store.com`). A Route maps to a specific Service.
3. **Upstream & Target**: Kong's built-in Layer-7 Load Balancer that distributes incoming traffic across multiple target host IP/port tuples (e.g., `10.0.1.5:8080`, `10.0.1.6:8080`) with active/passive health checks.
4. **Plugin**: Pluggable modules injected dynamically into the request/response lifecycle at global, service, route, or consumer scopes (e.g., `jwt`, `rate-limiting`, `cors`, `proxy-cache`, or `prometheus`).
5. **Consumer**: A client entity (a mobile app user, internal microservice, or external B2B partner) associated with credentials such as API keys, OAuth2 client secrets, or JWT public keys.

### Deployment Modes: DB-less (Declarative) vs PostgreSQL-backed
- **PostgreSQL Mode**: Stores routing rules, consumers, and plugin states inside a centralized PostgreSQL database. Dynamic administration occurs via Kong's Admin REST API (`curl -X POST http://localhost:8001/services`).
- **DB-less Mode (Declarative YAML)**: Loads its entire topology and plugin rules into memory from a single static YAML file (`kong.yaml`). This pattern is the gold standard for **GitOps**, CI/CD automation, and Kubernetes Ingress Controller (KIC) topologies because it eliminates external database dependencies and boots instantaneously.

---

## 3. Why It Matters at Production Scale

In high-throughput microservice ecosystems across scaling startups and enterprises:

1. **Elimination of Cross-Cutting Code Duplication**: Backend engineers writing Go or Java microservices no longer need to implement JWT validation, CORS headers, or rate-limiting middleware inside each service. Application code remains strictly dedicated to business logic.
2. **Centralized Security Enforcement & Zero Blast Radius**: When migrating signing algorithms (e.g., from HS256 to RS256) or rotating cryptographic keys, updates are applied to the gateway plugin in seconds without redeploying 50 backend services.
3. **Extreme NGINX/OpenResty Throughput**: Kong handles hundreds of thousands of requests per second (RPS) with sub-millisecond gateway latency overhead, drastically outperforming JVM-based or interpreted API gateways.
4. **Vast Plugin Ecosystem & Polyglot Extensibility**: Beyond dozens of battle-tested bundled plugins, custom plugins can be engineered in Lua, Go, Python, or JavaScript.

---

## 4. How It Works & Lifecycle Flow (ASCII)

The end-to-end request lifecycle traveling through Kong Gateway to internal microservices:

```
[ Mobile / Web Client ]
          |
          | 1. HTTP Request (GET /api/v1/orders, Header: Authorization: Bearer <JWT>)
          v
+-----------------------------------------------------------------------------------+
| KONG GATEWAY (Listening on :8000)                                                 |
|                                                                                   |
|  [ Router Engine ]                                                                |
|    |-- Match Path `/api/v1/orders` -> Route: "orders-route"                       |
|    |-- Resolve to Service: "order-service"                                        |
|    v                                                                              |
|  [ Plugin Execution Chain (Pre-Proxy Phase) ]                                     |
|    |-- 1. Plugin CORS (Validate incoming origin header)                           |
|    |-- 2. Plugin Rate Limiting (Check Redis quota: 100 req/min)                   |
|    |-- 3. Plugin JWT (Verify signature & expiry, decode claims)                   |
|    |-- 4. Plugin Request Transformer (Inject internal header: `X-User-ID: 12345`) |
|    v                                                                              |
|  [ Upstream Load Balancer (Round Robin) ]                                         |
|    |-- Forward to healthy backend target: `http://10.0.1.15:8080`                 |
+-----------------------------------------------------------------------------------+
          |
          | 2. Proxied Request (Header: X-User-ID: 12345)
          v
+------------------------------------+
| Internal Microservice (Go Backend) |
|   - Zero JWT verification overhead |
|   - Reads trusted `X-User-ID`      |
|   - Executes core database query   |
+------------------------------------+
          |
          | 3. HTTP 200 OK (JSON Response Payload)
          v
+-----------------------------------------------------------------------------------+
| KONG GATEWAY (Post-Proxy Phase)                                                   |
|    |-- Plugin Response Transformer (Strip sensitive debug headers)                |
|    |-- Plugin Prometheus (Record request latency & status code histogram)         |
+-----------------------------------------------------------------------------------+
          |
          | 4. HTTP 200 OK (Sanitized Response)
          v
[ Mobile / Web Client ]
```

---

## 5. Concrete Production Example: DB-less Declarative Setup

Below is a production-grade declarative DB-less configuration for Kong Gateway leveraging `kong.yaml` alongside a containerized `docker-compose.yml`.

### A. Declarative Configuration File (`kong.yaml`)

This configuration registers an upstream backend cluster, maps a route to `/api/v1/orders`, and applies **Rate Limiting** and **API Key Authentication**:

```yaml
_format_version: "3.0"

# 1. Define Upstream Load Balancer
upstreams:
  - name: order_upstream
    algorithm: round-robin
    targets:
      - target: order-backend-1:8080
        weight: 100
      - target: order-backend-2:8080
        weight: 100

# 2. Define Backend Services & Ingress Routes
services:
  - name: order-service
    url: http://order_upstream
    routes:
      - name: order-route
        paths:
          - /api/v1/orders
        strip_path: false
        methods:
          - GET
          - POST
    # 3. Attach Plugins at Service Scope
    plugins:
      # Rate Limiting: Max 100 requests per minute per Consumer / IP
      - name: rate-limiting
        config:
          minute: 100
          policy: local # Use 'redis' for multi-node Kong clusters
          fault_tolerant: true
          hide_client_headers: false

      # Key Authentication: Mandate valid 'apikey' header
      - name: key-auth
        config:
          key_names:
            - apikey
          hide_credentials: true

# 4. Registered Consumer Credentials
consumers:
  - username: mobile-app-client
    keyauth_credentials:
      - key: sec_mobile_app_prod_99812
```

### B. Deployment via `docker-compose.yml`

```yaml
version: '3.8'

services:
  kong:
    image: kong:3.5-alpine
    container_name: kong-gateway
    environment:
      KONG_DATABASE: "off"                      # Activate DB-less mode
      KONG_DECLARATIVE_CONFIG: /etc/kong/kong.yaml
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001           # Admin API (Read-only in DB-less)
      KONG_PROXY_LISTEN: 0.0.0.0:8000           # Public Ingress Proxy Port
    volumes:
      - ./kong.yaml:/etc/kong/kong.yaml:ro
    ports:
      - "8000:8000" # Public traffic ingress
      - "8001:8001" # Admin inspection
    restart: unless-stopped
```

### C. Client & Backend Verification

When an unauthenticated request hits the gateway:
```bash
curl -i http://localhost:8000/api/v1/orders
# Output: HTTP/1.1 401 Unauthorized
# {"message":"No API key found in request"}
```

When supplying a registered consumer key:
```bash
curl -i http://localhost:8000/api/v1/orders -H "apikey: sec_mobile_app_prod_99812"
# Output: HTTP/1.1 200 OK
# X-RateLimit-Limit-Minute: 100
# X-RateLimit-Remaining-Minute: 99
```

---

## 6. Trade-offs: Advantages and Disadvantages

### Advantages
- **Outstanding Throughput & Low Latency**: Powered by NGINX's event-driven architecture and LuaJIT, Kong offers world-class RPS performance with minimal memory consumption compared to Java/JVM-based gateways.
- **Dynamic Extensibility**: Boasts a vast ecosystem of bundled plugins that activate dynamically without restarting the underlying gateway daemon.
- **Architectural Versatility**: Supports DB-less mode for GitOps, PostgreSQL-backed mode for dynamic admin APIs, and Hybrid mode (decoupling Control Planes from edge Data Planes).

### Disadvantages
- **Lua Debugging Curve**: Developing custom plugins in Lua requires specialized understanding of the OpenResty request phases and memory lifecycle.
- **DB-less Runtime Limitations**: Certain plugins requiring dynamic state mutation during runtime (such as on-the-fly OAuth2 client creation via REST API) are constrained without redeploying the declarative YAML.
- **Distributed Rate Limiting Complexity**: Local in-memory rate limiting multiplies quotas across replicas unless explicitly tied to a shared Redis cluster.

---

## 7. When to Use vs When Not to Use

### When to Use
- **Polyglot Microservice Environments**: Your engineering squads use Go, Python, and Node.js. Enforcing standard JWT verification, TLS offloading, and rate limiting at Kong saves engineering teams from re-implementing auth middleware in three different languages.
- **Public Developer APIs & Partner Integrations**: Requiring fine-grained API Key authentication, tiered monetization quotas, and security throttling for external consumers.
- **Kubernetes Ingress Controllers**: Deploying the Kong Ingress Controller (KIC) to handle external cluster ingress dynamically via native Kubernetes Custom Resource Definitions (CRDs).

### When Not to Use
- **Simple Monolithic Deployments**: If your backend consists of a single Go binary and a PostgreSQL database, adding Kong introduces unnecessary network hops and operational friction. A simple NGINX reverse proxy is more than adequate.
- **Internal Microservice Mesh (East-West Traffic)**: If your primary requirement is mutual TLS (mTLS) and fine-grained circuit breaking between hundreds of internal microservices, dedicated service meshes (e.g., Istio or Linkerd) are better suited than an ingress API gateway.

---

## 8. Key Metrics & Observability

Kong's bundled `prometheus` plugin exports telemetry at `:8001/metrics`:

1. **Gateway Latency Overhead (`kong_latency_ms`)**: Time consumed within Kong executing plugin chains and internal routing (production goal: p99 < 2ms).
2. **Upstream Latency (`kong_upstream_latency_ms`)**: Latency consumed by your internal backend microservices to compute and return responses.
3. **HTTP Status Code Distribution (`kong_http_requests_total`)**: Monitor ratios of client error spikes (`401 Unauthorized`, `429 Too Many Requests`) versus backend server errors (`502 Bad Gateway`).
4. **Nginx Active Connections (`kong_nginx_connections_active`)**: Total concurrent TCP sockets handled across worker processes.

---

## 9. Real-World Case Study

### Scenario: Fintech Gateway Migration & Perimeter Security Hardening

#### Problem
A fast-growing fintech expanded from 3 monolithic services to 45 distributed microservices in two years. Every microservice handled token validation and rate limiting independently:
- Security vulnerabilities in one Node.js JWT library left 15 services vulnerable to exploit, whereas Go services remained unaffected.
- External banking partners frequently burst payment polling endpoints, overwhelming core database pools due to lack of global ingress rate limiting.

#### Architecture Solution
The team deployed **Kong Gateway** as their **Kubernetes Ingress Controller**:
1. Removed all token validation libraries across the 45 backend microservices.
2. Activated Kong's `jwt` and `cors` plugins at ingress. Kong validated cryptographic signatures at the perimeter and injected an immutable, trusted header `X-Consumer-Custom-ID: <user_id>` downstream.
3. Applied the `rate-limiting` plugin backed by a shared Redis cluster, capping external partners at 50 requests per second.
4. Adopted declarative GitOps via `decK`, managing all route changes through version-controlled Pull Requests.

#### Outcome / Impact
- **Zero Insecure Perimeter Endpoints**: 100% of public traffic verified at the edge; malformed or expired tokens are rejected before consuming backend CPU cycles.
- **Reduced p99 Latency**: Eliminating repetitive JWT decoding across internal hops standardized and lowered backend latency.
- **Outage Elimination**: Abrupt traffic spikes from external partners were safely throttled with HTTP `429 Too Many Requests`, keeping core database connections well within safe operating limits.

---

## 10. Production Gotchas & Best Practices

1. **DNS Caching & Stale Pod IPs in Kubernetes**: Kong (via NGINX) caches upstream DNS records aggressively. In Kubernetes environments where Pod IPs shift dynamically during deployments, Kong can mistakenly route traffic to terminated pods, causing sporadic `502 Bad Gateway` errors. Solution: Configure `dns_stale_ttl` and set low resolver TTLs (e.g., `dns_resolver = 10.96.0.10:53`, valid=5s).
2. **Prefer DB-less Mode on Kubernetes**: Running PostgreSQL to back Kong in Kubernetes adds an unnecessary point of failure. If the database crashes, Admin APIs freeze. Leverage DB-less mode using GitOps tools (`decK`) or the Kong Ingress Controller CRDs for stateless, zero-downtime operation.
3. **Beware of 'local' Rate Limiting Policies**: If you run 4 Kong replicas with a 100 req/min limit using `policy: local`, an aggressive client can actually issue up to 400 requests per minute across pods. Always set `policy: redis` when strict multi-replica quotas are required.

---

## 11. Summary

1. **Core Purpose**: Operates as a centralized entry point and reverse proxy managing authentication, rate limiting, SSL termination, and routing across microservices.
2. **High-Performance Engine**: Built on NGINX and OpenResty LuaJIT, offering sub-millisecond gateway latency and minimal memory overhead.
3. **Primary Abstractions**: Built around **Services** (backend endpoints), **Routes** (URL/Host matchers), **Upstreams** (load balancers), and **Plugins** (security, traffic, and transformation policies).
4. **Declarative DB-less Standard**: Modern cloud-native deployments utilize `kong.yaml` without external database dependencies for reliable, automated GitOps operations.
