# CORS (Cross-Origin Resource Sharing) & Preflight

## 1. Real-World Scenario

In a decoupled architecture separating frontend web clients from backend microservices:
- The Single-Page Application (SPA, built with React or Next.js) is served from: `https://app.ourstore.com` (or `http://localhost:3000` in local development).
- The Go backend API operates from a dedicated domain: `https://api.ourstore.com` (or `http://localhost:8080`).

When an authenticated user clicks **"Pay Order"**, browser-side JavaScript initiates an asynchronous network call:
```javascript
fetch("https://api.ourstore.com/v1/checkout", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer eyJhbGciOi..."
  },
  body: JSON.stringify({ order_id: 101 })
});
```

Instantly, the browser developer console turns red with an infamous error:
```text
Access to fetch at 'https://api.ourstore.com/v1/checkout' from origin 'https://app.ourstore.com' 
has been blocked by CORS policy: Response to preflight request doesn't pass access control check: 
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

What actually occurred under the hood?
1. Web browsers enforce a fundamental defensive barrier known as the **Same-Origin Policy (SOP)** to prevent malicious websites (e.g., `https://attacker-phishing.com`) from making unauthorized background requests to bank or e-commerce accounts active in adjacent tabs.
2. Because the request contains custom headers (`Authorization` and `Content-Type: application/json`), the browser automatically emits a probe request (*Preflight Request*) using the **`OPTIONS`** HTTP method before transmitting the actual `POST` payload.
3. Because the Go backend lacked headers authorizing the origin `https://app.ourstore.com`, the browser suppressed the response, preventing client JavaScript from reading it and failing the user's checkout entirely.

In high-throughput environments, misconfigured or missing *Preflight Caching* can inadvertently **double total backend ingress traffic (100% redundant load)** purely answering repetitive `OPTIONS` handshakes!

## 2. What Is It and What Is It For?

Cross-Origin Resource Sharing (CORS) is an HTTP header-based security mechanism implemented natively by web browsers allowing or restricting web applications running at one origin from interacting with resources at an adjacent origin.

An *Origin* is strictly defined by three tuples: **Protocol + Domain + Port**.
Two endpoints represent distinct origins if any tuple diverges:
- `http://api.com` vs `https://api.com` *(Protocol mismatch: HTTP vs HTTPS)*
- `https://app.com` vs `https://api.com` *(Subdomain divergence)*
- `http://localhost:3000` vs `http://localhost:8080` *(Port divergence)*

### Two Classes of CORS Invocations:
1. **Simple Requests**: Standard `GET`, `HEAD`, or `POST` calls utilizing basic browser headers. Dispatched immediately without preliminary probing.
2. **Preflight Requests**: Calls using mutating verbs (`PUT`, `DELETE`, `PATCH`) or custom headers (`Authorization`, `Content-Type: application/json`). The browser mandates dispatching an exploratory `OPTIONS` query first to secure backend consent.

## 3. Why Is It Needed at High Throughput?

1. **Halves Wasted Ingress Traffic (*Preflight Caching*)**: By emitting `Access-Control-Max-Age: 86400` (24-hour cache), browsers perform exactly 1 `OPTIONS` probe daily per endpoint, saving millions of redundant round-trips to backend Go pods.
2. **Prevents Malicious Credential Hijacking**: Configures `Access-Control-Allow-Credentials: true` with dynamic origin whitelisting without opening dangerous wildcard security holes.
3. **Dispelling Security Myths**: Understanding that **CORS is NOT a server-side firewall**. Malicious attackers using cURL, Postman, or automated Python scripts can query backend endpoints directly unimpeded by CORS (since CORS is exclusively enforced by client web browsers!).

## 4. How It Works (Preflight Request & Caching Flow)

```
[Browser: https://app.com]                          [Backend Go: https://api.com]
          |                                                       |
          | 1. [Preflight Check]: OPTIONS /v1/checkout ---------> |
          |    Headers:                                           |
          |    Origin: https://app.com                            |
          |    Access-Control-Request-Method: POST                |
          |    Access-Control-Request-Headers: Authorization      |
          |                                                       |
          | <---------------------------------------------------- |
          | 2. Returns HTTP 204 No Content:                       |
          |    Access-Control-Allow-Origin: https://app.com       |
          |    Access-Control-Allow-Methods: POST, GET, OPTIONS   |
          |    Access-Control-Allow-Headers: Authorization        |
          |    Access-Control-Max-Age: 86400 (Cached 24 Hours!)   |
          |                                                       |
  [Browser Validates Permission: PASSED]                          |
          |                                                       |
          | 3. [Actual Request]: POST /v1/checkout -------------> |
          |    Headers: Authorization: Bearer token...            |
          |                                                       |
          | <---------------------------------------------------- |
          | 4. Returns HTTP 200 OK (Checkout State Mutated)       |
```

## 5. Core Implementation Logic (High-Performance CORS Middleware in Go)

Below is an idiomatic, dependency-free Go CORS middleware implementing dynamic whitelisting, preflight caching, and immediate HTTP 204 terminations:

```go
package main

import (
	"net/http"
)

// allowedOrigins defines the whitelist of authorized web applications
var allowedOrigins = map[string]bool{
	"https://app.ourstore.com":   true,
	"https://admin.ourstore.com": true,
	"http://localhost:3000":     true, // Development environment
}

func CORSMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		origin := r.Header.Get("Origin")

		// 1. Verify caller origin against authorized whitelist
		if allowedOrigins[origin] {
			w.Header().Set("Access-Control-Allow-Origin", origin)
			w.Header().Set("Access-Control-Allow-Credentials", "true")
		}

		// 2. Intercept and handle Preflight Requests (OPTIONS)
		if r.Method == http.MethodOptions {
			w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, PATCH, OPTIONS")
			w.Header().Set("Access-Control-Allow-Headers", "Authorization, Content-Type, X-Request-ID")
			
			// CRITICAL FOR HIGH-THROUGHPUT: Cache preflight permission for 24 hours (86,400 seconds)
			// Cuts redundant probe queries by 50%!
			w.Header().Set("Access-Control-Max-Age", "86400")

			// Fast-terminate with 204 No Content without invoking downstream business handlers
			w.WriteHeader(http.StatusNoContent)
			return
		}

		// 3. Delegate actual functional requests (GET, POST, etc.) down the pipeline
		next.ServeHTTP(w, r)
	})
}
```

## 6. Trade-offs (Pros & Cons)

### Pros
- **Protects Users from Cross-Site Exploitation**: Prevents rogue sites from triggering malicious background transactions using ambient stored browser cookies.
- **Enables Clean Frontend/Backend Decoupling**: Frontend static apps can be globally distributed on edge CDNs (Vercel, Cloudflare Pages) completely separated from Kubernetes API clusters.
- **Granular Control**: Grants backends fine-grained governance over permissible methods, headers, and client origins.

### Cons
- **Network Latency Overhead (*Double Round-Trip*)**: Preflight `OPTIONS` calls introduce an extra network hop unless properly mitigated with aggressive browser caching.
- **Frequent Developer Friction**: A leading cause of local integration frustration among novice frontend and backend engineers.
- **Client-Side Enforcement Only**: Completely ineffective against automated scrapers, cURL, or server-side bots that ignore browser standards.

## 7. When to Use vs When to Avoid

### When to Configure
- Public or private API gateways serving browser-based Single Page Applications on distinct domains.
- Microservice backends accessed by internal web administration consoles hosted on isolated subdomains.
- Embeddable multi-tenant JavaScript widgets deployed across third-party websites.

### When to Avoid (CORS Does Not Apply)
- Direct server-to-server microservice RPCs (gRPC or internal HTTP within a private Kubernetes VPC).
- Monolithic web applications where HTML pages are rendered directly by the backend from the same origin (*Server-Side Rendering*).
- Native mobile applications (iOS Swift, Android Kotlin, Flutter) not hosted within browser WebViews.

## 8. Key Observability Metrics (Monitoring Dashboard)

- **`http_requests_total{method="OPTIONS"}` vs Business Methods (Counter)**: Monitors the ratio of preflights to actual requests. Approaching a 1:1 ratio indicates failing `Access-Control-Max-Age` cache retention.
- **`cors_origin_rejected_total` (Counter)**: Tracks requests rejected from non-whitelisted domains. Spikes indicate configuration regressions or active phishing attempts.

## 9. Real-World Case Studies

### Preflight OPTIONS Storms & Multi-Tenant Ingress in E-Commerce SaaS

- **The Engineering Problem**:
  A multi-tenant e-commerce platform hosted thousands of independent merchant store fronts on custom domains (`merchant-a.com`, `merchant-b.io`) all querying a consolidated checkout backend at `api.platform.com`. Developers originally configured `Access-Control-Allow-Origin: *`. When the architecture shifted to secure, encrypted authentication cookies (`HttpOnly; SameSite=None; Secure`), client web browsers uniformly rejected checkout requests because browser specifications strictly forbid credentialed requests when origins are set to wildcards (`*`). Attempting a naive fix, engineers introduced backend middleware that executed a PostgreSQL query on every single incoming preflight `OPTIONS` request to validate whether the calling merchant was legitimate. This triggered severe database connection pool saturation under 50,000 RPS of preflight traffic, pushing checkout latencies past 650ms.

- **The Architectural Solution**:
  1. **Edge Dynamic Origin Reflection**: The CORS evaluation pipeline was relocated from Go application handlers into the edge reverse proxy layer (Envoy / Cloudflare Workers). Active merchant domains were loaded into an in-memory lookup cache (resolving in under 0.05ms) without touching primary databases.
  2. **Aggressive Preflight Caching**: Edge proxies injected the header `Access-Control-Max-Age: 86400` (24 hours) on all preflight responses, instructing shopper browsers to cache preflight clearance for an entire day per merchant origin.
  3. **Immediate Edge 204 Termination**: Preflight `OPTIONS` calls were terminated directly at the ingress proxy with `204 No Content`, offloading backend Go compute workers entirely.
  4. **Strict Allowed Headers**: Whitelisted exclusively necessary business headers (`Authorization`, `Content-Type`, `X-Idempotency-Key`).

- **The Engineering Outcome**:
  - Inbound `OPTIONS` request volume to Go backend application pods dropped by 97%.
  - User-perceived client-side initial interaction latency plummeted from 650ms to 28ms.
  - Multi-tenant merchants operated cookie-backed authenticated checkout flows securely without violating browser Same-Origin Policy mandates.

## 10. Production Gotchas

1. **Combining `Access-Control-Allow-Origin: *` with Credentials**: Setting wildcards while expecting the browser to transmit cookies or auth credentials. Browsers reject the response immediately if `Access-Control-Allow-Credentials: true` is paired with `*`!  
   *Mitigation*: Validate the incoming `Origin` against a whitelist and echo back that specific origin in headers.
2. **Omitting `Access-Control-Max-Age`**: Without this header, browsers fire an `OPTIONS` call before *every single* `POST/PUT/DELETE`. At 50,000 RPS, your backend receives an unnecessary 100,000 RPS.  
   *Mitigation*: Set `Access-Control-Max-Age: 86400` (1 day).
3. **Assuming CORS Replaces API Authentication**: Leaving APIs unauthenticated under the illusion that "CORS blocks unauthorized callers". Attackers can hit endpoints directly using cURL or Python in seconds.

## 11. Key Takeaways

1. **Browser-Enforced Security**: CORS is a browser-level security barrier preventing rogue web applications from exploiting ambient client credentials, not a server-side authentication system.
2. **Never Pair Wildcards with Credentials**: Do not combine `Access-Control-Allow-Origin: *` with `Allow-Credentials: true`; dynamically mirror verified origins from an authorized whitelist.
3. **Always Enforce Preflight Caching**: Include `Access-Control-Max-Age` (e.g. 24 hours) to eliminate redundant, expensive preflight roundtrips across user sessions.
4. **Terminate Preflights at the Ingress Edge**: Complete `OPTIONS` handshakes directly at your API Gateway or reverse proxy with `204 No Content` to insulate backend application workers from preflight overhead.
