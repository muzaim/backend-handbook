# Rate Limiting

## 1. Real-World System Scenario
On public user authentication endpoints like `POST /api/v1/auth/login`, each inbound request forces the server to look up records in PostgreSQL and compute cryptographic password hashes using *bcrypt* or *Argon2* (~80 milliseconds CPU time per hash).
On public user authentication endpoints like `POST /api/v1/auth/login`, each inbound request forces the server to look up records in PostgreSQL and compute cryptographic password hashes using *bcrypt* or *Argon2* (~80 milliseconds CPU time per hash).

If this endpoint is exposed without traffic admission control:
- An attacker launches an automated credential stuffing or brute-force botnet, firing 3,000 login attempts per second against your API from a single IP subnet.
- Executing 3,000 intensive bcrypt computations per second immediately saturates 100% of all backend CPU cores in seconds.
- Legitimate users attempting to log in, browse the store, or submit orders experience connection timeouts, and the entire backend platform collapses (*Denial of Service*).

**The Rate Limiter Solution (e.g., Redis Token Bucket / Sliding Window):**
You configure a rate limiting rule at your edge gateway: a strict limit of 5 login attempts per 1-minute window per client IP. Requests 1 through 5 pass through for verification. The moment request 6 arrives within that same window, the Rate Limiter intercepts it in-memory within < 1 millisecond, immediately returning **`HTTP 429 Too Many Requests`** along with a `Retry-After: 60` response header. Database connection pools and backend CPU cores remain entirely unburdened.

## 2. What Is It and What Is It For?

Rate Limiting is a traffic control policy that caps how frequently a client or process is permitted to hit an API or resource within a specified time window (for example: a maximum of 100 requests per minute per IP address).

When a threshold is exceeded, the server promptly rejects subsequent requests with standard HTTP status: **`429 Too Many Requests`**.

Primary objectives:
- **Denial of Service (DoS/DDoS) Mitigation**: Shielding application threads and databases from automated volumetric floods.
- **Credential Stuffing & Scraping Defense**: Preventing brute-force password guessing against login endpoints (e.g., maximum 5 attempts per minute).
- **Fair Usage Policy (Noisy Neighbor Isolation)**: Ensuring that one greedy or poorly written client script cannot monopolize compute capacity to the detriment of other users.
- **Third-Party API Cost Governance**: Enforcing cost containment when wrapping expensive upstream APIs (such as OpenAI models, Twilio SMS, or payment providers).

### Popular Rate Limiting Algorithms:
- **Token Bucket**: Tokens refill into a bucket at a constant rate. Each request consumes 1 token. If the bucket is empty, requests are denied. This is the industry standard because it naturally accommodates brief bursts of legitimate traffic.
- **Leaky Bucket**: Inbound requests enter a FIFO buffer and exit at a strictly regulated drip rate (like water exiting a punctured bucket), smoothing out traffic spikes.
- **Fixed Window Counter**: Tallies requests within fixed time blocks (e.g., seconds 00 to 59). Flaw: allows double the quota during boundary transitions (*window edge burst*).
- **Sliding Window Counter / Log**: Evaluates request volume across a dynamic trailing window (e.g., the exact preceding 60 seconds from current millisecond). Highly accurate at the cost of higher memory consumption.

## 3. Why Use It in High-Throughput Systems?

At massive scale, database write capacity and thread pools represent finite, vulnerable resources.

Without Rate Limiting:
- A single aggressive third-party web scraper can saturate 95% of database I/O, causing normal interactive users to suffer severe timeouts.
- A client-side infinite loop bug in an updated mobile app release can accidentally self-DDoS the entire company infrastructure.

Enforcing rate limits at the outer perimeter (the API Gateway or reverse proxy layer) allows millions of abusive requests to be rejected in sub-millisecond time via in-memory Redis keys before touching CPU-heavy application runtimes or databases.

## 4. How It Works (Data Flow Diagram - Token Bucket)

```
                       +-------------------+
                       |  Inbound Request  |
                       +-------------------+
                                 |
                                 v
                     +-----------------------+
                     | Rate Limiter (Memory) |
                     | [ Token Bucket: IP ]  |
                     +-----------------------+
                                 |
                        Are tokens > 0?
                                 |
                 +---------------+---------------+
                 |                               |
              [ YES ]                         [ NO ]
                 |                               |
                 v                               v
          Consume 1 Token                 Reject Request
                 |                               |
                 v                               v
         +---------------+               +---------------+
         |  Forward to   |               | Return HTTP   |
         |  Backend Svc  |               | 429 Too Many  |
         +---------------+               | Requests      |
                                         +---------------+
```

## 5. Core Implementation Logic (Token Bucket di Go dengan Redis)

Below is the core implementation Token Bucket yang dieksekusi secara atomik menggunakan Go dan Redis:

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

var ctx = context.Background()

// IsRequestAllowed memeriksa kuota Token Bucket secara atomik dengan Redis
func IsRequestAllowed(rdb *redis.Client, clientID string, maxCapacity int64, refillRatePerSec float64) bool {
	key := fmt.Sprintf("ratelimit:%s", clientID)
	now := time.Now().Unix()

	// Gunakan Redis Hashes untuk menyimpan [tokens, last_updated]
	vals, err := rdb.HMGet(ctx, key, "tokens", "last_updated").Result()
	var tokens float64 = float64(maxCapacity)
	var lastUpdated int64 = now

	if err == nil && vals[0] != nil && vals[1] != nil {
		fmt.Sscanf(vals[0].(string), "%f", &tokens)
		fmt.Sscanf(vals[1].(string), "%d", &lastUpdated)

		// 1. Refill token berdasarkan selisih waktu
		elapsed := now - lastUpdated
		tokens = tokens + (float64(elapsed) * refillRatePerSec)
		if tokens > float64(maxCapacity) {
			tokens = float64(maxCapacity)
		}
	}

	// 2. Konsumsi token jika tersedia
	if tokens >= 1.0 {
		tokens -= 1.0
		rdb.HSet(ctx, key, "tokens", tokens, "last_updated", now)
		rdb.Expire(ctx, key, 1*time.Hour)
		return true // Request diizinkan lewat
	}

	return false // 429 Too Many Requests (Rate limit terlampaui)
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Predictable Service Availability**: Guarantees backend load remains safely within provisioned capacity boundaries.
- **Infrastructure Cost Control**: Prevents runaway cloud auto-scaling bills caused by bot attacks or crawlers.
- **Security Hardening**: Neutralizes automated account takeover attempts and API scrapers.

### Cons
- **NAT / Shared IP Collisions**: Legitimate corporate users behind an enterprise proxy (sharing a single public outbound IP) can accidentally get rate limited together.
- **Distributed State Latency**: Synchronizing rate limit counters across an external Redis cluster incurs a small network hop (1–2 ms per request).
- **Client Handling Complexity**: Clients must implement defensive handling for HTTP 429 codes and parse `Retry-After` headers.

## 7. When to Use vs When Not to Use

### When to Use
- All public-facing REST, GraphQL, or RPC endpoints (especially expensive operations: search queries, report exports, file uploads).
- Sensitive authentication routes: `/login`, `/signup`, `/reset-password`, `/send-otp`.
- Tiered commercial APIs with monetization paywalls.

### When Not to Use
- Trusted private internal microservice calls where service-to-service contracts and circuit breakers already manage capacity.
- Internal infrastructure health check endpoints pinged by load balancers (blocking a health check probe will cause the balancer to mistakenly mark healthy servers dead!).

## 8. Key Metrics & Observability (Monitoring Dashboard)

- **`rate_limit_hits_total` (Counter)**: Ingress volume successfully serviced within allocated quotas.
- **`rate_limit_exceeded_total` (Counter)**: Count of rejected HTTP 429 events. Spikes signal brute-force campaigns, scrapers, or client polling bugs.
- **`rate_limit_token_depth` (Gauge)**: Average percentage of available tokens across active clients (identifies impending capacity exhaustion).
- **`rate_limiter_evaluation_latency` (Histogram)**: Overhead incurred querying distributed Redis counters. Target: < 1ms.

## 9. Real-World Case Studies

### Defeating Credential Stuffing & Scraping Floods in E-Commerce

- **The Engineering Problem**:
  An international retail e-commerce platform experienced an aggressive distributed credential stuffing attack against its `/api/v1/auth/login` endpoint coupled with scraping on its `/api/v1/search` endpoint. Over 40,000 requests per second were proxied through rotating residential IP proxies. Application server CPU utilization immediately hit 100% due to repeated execution of computationally heavy `bcrypt` cryptographic password hashing routines. Concurrently, expensive wildcard SQL text queries flooded PostgreSQL buffer pools, crashing legitimate customer checkouts with cascading gateway timeouts.

- **The Architectural Solution**:
  1. **Multi-Tier Rate Limiting Architecture**: Architecture engineers integrated a multi-level defense at the API Gateway backed by Redis Cluster:
     - *Ingress IP Limiter (Sliding Window Log)*: Throttles cumulative requests per external IP block to a maximum of 120 req/min, slicing through brute volumetric scrapers.
     - *Sensitive Endpoint Guard (Atomic Token Bucket via Redis Lua Script)*: The `/login` endpoint enforces a strict quota of 5 attempts per minute per account identity key (`login:attempt:{username}`), executed inside an atomic Lua script to eliminate concurrent race condition bypasses.
  2. **Standardized 429 Contract**: When token limits exhaust, the edge layer immediately responds with `HTTP 429 Too Many Requests` in under 0.8ms without touching upstream Go pods, packaging standard `Retry-After: 60` and `X-RateLimit-Remaining: 0` headers.
  3. **Escalating Penalty Windows**: Clients repeatedly bombarding endpoints past quota depletion are automatically subjected to exponential quarantine multipliers applied directly to Redis key TTL timers.

- **The Engineering Outcome**:
  - Backend application CPU load dropped from 100% to a steady 28% after dropping 94% of malicious bot traffic at the gateway edge.
  - Database IOPS pressure fell by 70%, recovering customer p99 transaction latencies from 8,000ms down to 45ms.
  - Credential stuffing attempts were completely neutralized without blocking legitimate shoppers sharing identical public NAT gateways.

## 10. Interesting Facts & Production Gotchas

- **Standard Rate Limit Response Headers**: Well-designed APIs always communicate rate limit status transparently:
  - `X-RateLimit-Limit`: Maximum requests permitted within the evaluation window.
  - `X-RateLimit-Remaining`: Remaining request quota in the current window.
  - `Retry-After`: The number of seconds the client must pause before retrying.
- **The Redis Race Condition Trap**: A classic junior developer trap is implementing rate limiting using naive multi-step logic: `GET count`, check if `< limit`, then `INCR count`. Under high concurrency, hundreds of requests execute the `GET` check simultaneously before any increment occurs, bypassing the quota entirely. The production-ready solution is using an **atomic Redis Lua Script**, guaranteeing atomicity in a single roundtrip.

## 11. Key Takeaways

1. **System Resource Guard**: Shields downstream databases and worker goroutines from fatal traffic spikes, DoS floods, and client-side retry storms.
2. **Mandatory Atomicity via Redis Lua**: Never decouple state lookup and counter increments; always execute rate checks within atomic Lua scripts to prevent concurrent race conditions.
3. **Endpoint-Specific Tiering**: Apply strict, aggressive quotas on computationally expensive operations (like bcrypt hashing or full-text queries) compared to lightweight read routes.
4. **Transparent Client Signaling**: Always emit `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `Retry-After` headers alongside `HTTP 429` statuses so well-behaved clients can back off cleanly.
