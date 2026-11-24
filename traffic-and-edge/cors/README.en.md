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

- **Stripe & PayPal Checkout**: Provides embeddable iframe payment components (*Stripe Elements*). Stripe dynamic CORS policies securely permit payment forms embedded across millions of independent merchant sites without exposing raw credit card numbers to the merchant's host domain.
- **GitHub API**: Operates an open public CORS configuration (`Access-Control-Allow-Origin: *`) across public endpoints (`/users`, `/repos`) to enable global open-source developer web extensions and integrations.

## 10. Production Gotchas

1. **Combining `Access-Control-Allow-Origin: *` with Credentials**: Setting wildcards while expecting the browser to transmit cookies or auth credentials. Browsers reject the response immediately if `Access-Control-Allow-Credentials: true` is paired with `*`!  
   *Mitigation*: Validate the incoming `Origin` against a whitelist and echo back that specific origin in headers.
2. **Omitting `Access-Control-Max-Age`**: Without this header, browsers fire an `OPTIONS` call before *every single* `POST/PUT/DELETE`. At 50,000 RPS, your backend receives an unnecessary 100,000 RPS.  
   *Mitigation*: Set `Access-Control-Max-Age: 86400` (1 day).
3. **Assuming CORS Replaces API Authentication**: Leaving APIs unauthenticated under the illusion that "CORS blocks unauthorized callers". Attackers can hit endpoints directly using cURL or Python in seconds.

## 11. Quick Summary

CORS is a browser security protocol regulating cross-origin resource sharing. For high-throughput systems, the key is implementing dynamic origin whitelisting, responding immediately to `OPTIONS` with HTTP 204 No Content, and setting `Access-Control-Max-Age` caching to eliminate redundant preflight network overhead.
