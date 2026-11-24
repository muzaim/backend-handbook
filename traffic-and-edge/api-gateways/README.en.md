# API Gateway

## 1. Real-World System Scenario
In microservices-based e-commerce architectures, platform services are decomposed into discrete backend instances inside a private VPC:
In microservices-based e-commerce architectures, platform services are decomposed into discrete backend instances inside a private VPC:
- `Auth Service` (port 4001)
- `Product Catalog Service` (port 4002)
- `Order Service` (port 4003)
- `Payment Service` (port 4004)
- `Notification Service` (port 4005)

If external mobile clients (iOS and Android) interact with these services directly:
- The mobile app must hardcode and maintain 5 different hostnames and IP addresses. If a service is refactored, split, or relocated to a new port, every mobile client must receive a forced app update through the App Store and Google Play.
- Every individual backend service must re-implement JWT validation, CORS headers, SSL handling, and rate limiting logic independently (*redundant code duplication*).
- Rendering a single checkout screen requires the mobile device to make 4 separate HTTP round-trips over latency-prone cellular towers (*excessive network chattiness*).

**The API Gateway Solution (e.g., Kong / KrakenD):**
You position an API Gateway at the public domain `https://api.store.com`. Mobile apps interface exclusively with this unified boundary. The gateway inspects and verifies JWT bearer tokens once at the perimeter, checks consumer rate limits, routes `/api/v1/orders` internally to `http://order-service:4003`, and aggregates responses across multiple downstream services into a single lightweight JSON payload for the client.

## 2. What Is It and What Is It For?

An API Gateway is an architectural component that serves as the single point of entry for all external clients (mobile apps, web SPAs, third-party partners) interacting with a microservices-based backend system.

While a reverse proxy primarily operates at the network and transport layers (TCP/HTTP, TLS, static file caching), an API Gateway focuses on application-level routing and API business policy:
- **Routing & Path Rewriting**: Routing `/users/*` requests to the User Service and `/orders/*` to the Order Service.
- **Centralized Authentication & Authorization**: Validating JWT tokens or API keys at the boundary so downstream services can trust verified headers.
- **Request Aggregation (Backend for Frontend / BFF)**: Combining responses from multiple internal microservices into a single consolidated JSON response for mobile clients.
- **Granular Rate Limiting & Quota Management**: Enforcing tier-based rate limits (e.g., Free vs Enterprise API usage).

Popular software: Kong, Apache APISIX, KrakenD, Spring Cloud Gateway, AWS API Gateway.

## 3. Why Use It in High-Throughput Systems?

Without an API Gateway, mobile or frontend clients must issue individual network calls to dozens of internal services directly across the internet:
- **Mobile Network Latency Penalties**: Executing multiple sequential HTTP roundtrips over cellular networks results in poor user experiences and high battery consumption.
- **Expanded Attack Surface**: Every internal service must expose a public IP and open ingress firewall ports.
- **Cross-Cutting Code Duplication**: Every engineering team is forced to re-implement authentication, CORS handling, request logging, and tracing boilerplate.

With an API Gateway, external clients make one rapid call, and the gateway orchestrates low-latency internal calls inside a gigabit private network or VPC mesh.

## 4. How It Works (Data Flow Diagram)

```
+--------------------+
| Mobile / Web Apps  |
+--------------------+
          |
          | 1. Single API Request (e.g. GET /dashboard + JWT)
          v
+---------------------------------------+
|             API Gateway               |
|                                       |
| - Verify Auth / JWT Token             |
| - Check Rate Limit Quota              |
| - Route & Aggregate Responses         |
+---------------------------------------+
      |               |               |
      | 2a. HTTP/gRPC | 2b. HTTP/gRPC | 2c. HTTP/gRPC
      v               v               v
+------------+  +------------+  +------------+
| User Svc   |  | Order Svc  |  | Payment Svc|
+------------+  +------------+  +------------+
```

## 5. Core Implementation Logic (Middleware Gateway di Go)

Below is the execution pipeline middleware implementation in Go middleware yang dijalankan API Gateway pada setiap request menggunakan Go:

```go
package main

import (
	"net/http"
	"net/http/httputil"
	"net/url"
)

// GatewayMiddlewarePipeline menjalankan autentikasi terpusat dan perutean microservice
func GatewayMiddlewarePipeline(targetServiceURL string) http.Handler {
	target, _ := url.Parse(targetServiceURL)
	proxy := httputil.NewSingleHostReverseProxy(target)

	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// 1. Validasi Autentikasi JWT Terpusat
		token := r.Header.Get("Authorization")
		if token == "" || !isValidJWT(token) {
			http.Error(w, `{"error": "Unauthorized"}`, http.StatusUnauthorized)
			return
		}

		// 2. Cek Kuota Rate Limiting per IP
		if !rateLimiterAllow(r.RemoteAddr) {
			http.Error(w, `{"error": "Rate limit exceeded"}`, http.StatusTooManyRequests)
			return
		}

		// 3. Inject Context & Teruskan ke Microservice Internal
		r.Header.Set("X-User-ID", extractUserID(token))
		proxy.ServeHTTP(w, r)
	})
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Simplified Client Integration**: Clients only track a single clean API hostname and endpoint schema.
- **Centralized Cross-Cutting Policies**: Security, telemetry, rate limiting, and CORS are configured and audited centrally.
- **Protocol Translation Flexibility**: Can accept public HTTP/JSON requests and convert them to high-speed internal binary gRPC calls.

### Cons
- **Potential Performance Bottleneck**: All traffic funnels through the gateway; misconfiguration degrades the entire platform.
- **Cascading Single Point of Failure (SPOF)**: Gateway failures immediately blind all client applications.
- **The "Smart Gateway" Trap**: Teams are often tempted to inject heavy business domain logic into the gateway, mutating it into an unmaintainable distributed monolith.

## 7. When to Use vs When Not to Use

### When to Use
- Microservices architectures with multiple discrete business domains.
- Multi-client applications where mobile, web, and IoT require distinct payload shapes and rate limits.
- Public developer APIs requiring per-client API key metering, throttling, and monetization.

### When Not to Use
- Standard monolithic architectures (a standard reverse proxy like NGINX is much simpler and faster).
- Small engineering teams with moderate traffic where operating a dedicated gateway cluster is unnecessary overhead (*premature optimization*).

## 8. Key Metrics & Observability (Monitoring Dashboard)

- **`gateway_internal_latency_seconds` (p95/p99)**: The duration spent purely inside gateway middleware logic (excluding upstream wait time). Target: < 5ms.
- **`auth_verification_failures_total` (Counter)**: Authentication rejections (signals brute-force floods or stale key tokens).
- **`upstream_service_errors_total{service="order-svc"}` (Counter)**: Tracking error distributions per isolated downstream microservice.
- **`request_body_size_bytes` (Histogram)**: Monitoring ingress payload allocations to protect worker memory buffers.

## 9. Real-World Case Studies (Tech Giants)

- **Netflix (Zuul & Zuul 2)**: Pioneered API Gateways in AWS cloud infrastructure to manage billions of daily API requests from hundreds of smart TVs, game consoles, and mobile clients. Zuul handles dynamic routing, A/B canary testing, and edge cryptographic validation.
- **Kong (Born from Mashape)**: Kong was engineered by API marketplace Mashape to process millions of developer API transactions per second. Built atop OpenResty (NGINX + Lua), Kong uses modular C/Lua plugins to enforce low-latency security and traffic policies at sub-millisecond overhead.

## 10. Interesting Facts & Production Gotchas

- **The Smart Gateway Anti-pattern**: The most common production mistake is adding database queries or complex transactional logic inside gateway plugins. Gateways should remain lightweight traffic dispatchers and security checkpoints, never domain executors.
- **Cryptographic JWT Bottlenecks**: At high scale (e.g., 50,000+ requests/sec), validating asymmetric RSA/ECDSA cryptographic signatures on every inbound request can consume over 70% of the gateway's CPU cores. High-throughput architectures mitigate this by caching verified token states in shared memory for short intervals (e.g., 10-30 seconds).
