# API Versioning & Deprecation Strategies

## 1. Simple Analogy & Real-World Scenario

When a government revenue department updates its annual tax return from Form 2024 to Form 2026 with revised tax bracket fields, it cannot unilaterally reject and destroy every Form 2024 currently in transit through the postal service. The revenue authority declares a formal transition window: Form 2024 remains legally accepted and processed until an established deadline (*Sunset Date*), while taxpayers are proactively instructed to migrate filings to Form 2026.

In backend systems engineering, **API Versioning & Deprecation** is the disciplined practice of evolving data contracts smoothly without disrupting active clients in production. This practice is paramount in **Mobile App Ecosystems (Android/iOS)** and **Third-Party Partner APIs**, where backend developers lack the authority to force millions of remote clients to update their software overnight.

---

## 2. Definition & Core Concepts

**API Versioning** involves running new API schema modifications side-by-side with preceding releases. When an older API version reaches its planned end-of-life, the **API Deprecation** lifecycle provides formal, machine-readable notification windows to consumers before endpoints are permanently deactivated (*Sunset / Decommission*).

```
+--------------------------------------------------------------------------+
|                            API VERSION LIFECYCLE                         |
+--------------------------------------------------------------------------+
|                                                                          |
|   [ACTIVE / GA]   --> API operates normally with full SLA support        |
|         |                                                                |
|         v                                                                |
|   [DEPRECATED]    --> New version released; legacy version operates      |
|         |             with automated Sunset and Deprecation headers      |
|         v                                                                |
|   [SUNSET / RETIRED]  Endpoint permanently decommissioned (410 / 404)    |
|                                                                          |
+--------------------------------------------------------------------------+
```

### Three Prevailing API Versioning Strategies

| Strategy | Example Syntax | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **1. URI Path** | `GET /api/v1/orders`<br>`GET /api/v2/orders` | Explicit, easy to route at edge proxies (NGINX/Envoy), highly cacheable across CDNs. | Deviates from strict REST theory (URI specifies version rather than resource identity). |
| **2. Custom Header** | `GET /api/orders`<br>`X-API-Version: 2` | Clean and elegant URIs adhering to RESTful principles. | Harder to cache across CDNs (requires `Vary: X-API-Version`), cannot be tested directly in browser address bars. |
| **3. Content Negotiation** | `Accept: application/vnd.company.v2+json` | Canonical RFC hypermedia standard. | High cognitive friction for API consumers, complex cURL/testing workflows. |

---

## 3. Why It Is Critical at Scale

1. **The Mobile Application Update Lag**:  
   Unlike web applications where frontends and backends can deploy synchronously, **20-30% of Android and iOS mobile users never enable automatic app updates**! Introducing breaking payload modifications (such as splitting `full_name` into `first_name` and `last_name`) on existing endpoints triggers immediate fatal exceptions (*NullPointerExceptions*) across hundreds of thousands of installed devices.
2. **Enterprise Partner SLAs (B2B Integrations)**:  
   Financial and payment gateways (such as Stripe or Adyen) integrate with thousands of third-party enterprise platforms. Mutating response schemas without versioning breaks partner clearing systems, exposing platforms to catastrophic legal breaches of uptime SLAs.
3. **Machine-Readable Decommissioning (IETF Standards)**:  
   Standardizing API retirement relies on formal IETF RFC HTTP headers:
   - **`Deprecation: true`** or **`Deprecation: @<timestamp>`**: Formally indicates deprecation status.
   - **`Sunset: <HTTP-date>`** (RFC 8594): The immutable timestamp when the endpoint will be severed.
   - **`Link: <url>; rel="sunset"`**: Direct URL pointing to developer migration documentation.

---

## 4. How It Works & Architecture Flow (ASCII)

```
[Legacy Client (v1)]               [Modern Client (v2)]
        |                                  |
        | GET /api/v1/products             | GET /api/v2/products
        v                                  v
+-------------------------------------------------------------+
|                      API GATEWAY / ROUTER                   |
|                                                             |
| - Version Resolution: Path /v1/ vs /v2/                     |
| - Header Injection: Deprecation & Sunset on v1 ingress      |
+------------------------------+------------------------------+
                               |
            +------------------+------------------+
            |                                     |
            v                                     v
+-----------------------+             +-----------------------+
|  Handler V1 (Legacy)  |             |  Handler V2 (Modern)  |
|  - Deprecation Header |             |  - Evolved Schema     |
|  - Sunset: 31-Dec-2026|             |  - Optimized Latency  |
+-----------------------+             +-----------------------+
```

---

## 5. Production Code Implementation (Go)

Below is a production-grade **API Versioning Middleware** in Go that handles path-based version routing while automatically injecting IETF RFC 8594 `Deprecation` and `Sunset` headers on deprecated routes:

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"
)

// DTO Version 1 (Legacy): Combines user identity into a single full_name string
type UserResponseV1 struct {
	ID       string `json:"id"`
	FullName string `json:"full_name"`
	Email    string `json:"email"`
}

// DTO Version 2 (Modern): Splits names into explicit first and last name attributes
type UserResponseV2 struct {
	ID        string `json:"id"`
	FirstName string `json:"first_name"`
	LastName  string `json:"last_name"`
	Email     string `json:"email"`
}

// Middleware injecting standard IETF Deprecation and Sunset headers
func DeprecationNoticeMiddleware(sunsetDate time.Time, docURL string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// RFC 8594 Sunset Header and Draft Deprecation standard
			w.Header().Set("Deprecation", "true")
			w.Header().Set("Sunset", sunsetDate.UTC().Format(http.TimeFormat))
			w.Header().Set("Link", fmt.Sprintf("<%s>; rel=\"sunset\"", docURL))

			next.ServeHTTP(w, r)
		})
	}
}

// Handler V1 (Deprecated)
func GetUserV1(w http.ResponseWriter, r *http.Request) {
	resp := UserResponseV1{
		ID:       "USR-101",
		FullName: "Syafri Surya",
		Email:    "syafri@example.com",
	}

	w.Header().Set("Content-Type", "application/json")
	_ = json.NewEncoder(w).Encode(resp)
}

// Handler V2 (Active / Modern)
func GetUserV2(w http.ResponseWriter, r *http.Request) {
	resp := UserResponseV2{
		ID:        "USR-101",
		FirstName: "Syafri",
		LastName:  "Surya",
		Email:     "syafri@example.com",
	}

	w.Header().Set("Content-Type", "application/json")
	_ = json.NewEncoder(w).Encode(resp)
}

func main() {
	mux := http.NewServeMux()

	// Target decommissioning date for v1: December 31, 2026
	v1SunsetDate := time.Date(2026, time.December, 31, 23, 59, 59, 0, time.UTC)
	migrationDoc := "https://api.example.com/docs/migrations/v1-to-v2"

	// Register V1 routes wrapped with the deprecation notice middleware
	v1Handler := http.HandlerFunc(GetUserV1)
	mux.Handle("/api/v1/user", DeprecationNoticeMiddleware(v1SunsetDate, migrationDoc)(v1Handler))

	// Register V2 routes (active standard)
	mux.HandleFunc("/api/v2/user", GetUserV2)

	fmt.Println("API Server listening on port :8080...")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## 6. Trade-offs: Pros and Cons

### Pros
- **Zero-Downtime System Evolution**: Client applications migrate gracefully at their own pace without unexpected service interruptions.
- **Predictable Decommissioning**: Clients receive clear, verifiable calendar deadlines before old interfaces terminate.
- **Automated Developer Telemetry**: Standardized headers allow client SDKs to surface automated warnings directly in development consoles.

### Cons
- **The Maintenance Tax**: Concurrently maintaining multiple API versions forces backend teams to support, test, and patch multiple runtime code paths.
- **Relational Schema Drift**: When multiple API versions write to identical database tables using disparate field layouts, backends require complex translation shims.
- **Client Inertia**: Absent firm sunset dates, consumers will delay migrations indefinitely.

---

## 7. When to Version vs. When to Make Non-Breaking Changes

### When to Introduce a New API Version (Breaking Changes):
1. **Removing or Renaming Attributes**: Deleting `phone` or renaming it to `phone_number`.
2. **Mutating Data Types**: Changing an `amount` field from an integer (`15000`) to a formatted currency string (`"USD 150.00"`).
3. **Altering Authentication Schemas**: Migrating from API Key query parameters to OAuth2 Bearer Tokens.
4. **Structural Response Modifications**: Altering an array response `data: [...]` into a paginated object `{ items: [...], cursor: "..." }`.

### When NOT to Version (Non-Breaking Enhancements):
1. **Adding Optional Fields**: Appending optional attributes (like `avatar_url`) to existing JSON payloads. Properly written clients simply ignore unrecognized fields.
2. **Adding New Endpoints**: Introducing `POST /api/v1/orders/refund`.
3. **Internal Performance Tuning**: Optimizing SQL queries or implementing Redis caching on server backends.

---

## 8. Key Metrics & Observability

Critical Prometheus / Grafana metrics to monitor API adoption and track sunset schedules:

- **`http_requests_total{version="v1|v2", handler="..."}`**: Traffic distribution between legacy and modern API versions.
- **`deprecated_api_requests_total{client_type="mobile_android|mobile_ios|web"}`**: Ingress volume hitting deprecated routes partitioned by client runtime.
- **`sunset_countdown_days{version="v1"}`**: Time remaining until hard decommissioning deadlines.
- **`api_client_user_agent_breakdown{version="v1"}`**: Specific mobile build signatures (`User-Agent: ShoppingApp/2.4.1`) relying on legacy contracts.

---

## 9. Real-World Case Studies

### Stripe API Evolution: Hundreds of Versions Maintained via Transformation Pipelines

- **The Engineering Problem**: Stripe provides critical financial infrastructure to millions of merchants globally. Since 2011, Stripe has released hundreds of API revisions. Had Stripe adopted traditional approaches by creating distinct controller classes for every version (`v1`, `v2`, ..., `v100`), codebase complexity would have ballooned uncontrollably, causing unmanageable regression testing burdens.
- **The Architectural Solution**: Stripe engineered an *Eventual Transformation Pipeline* keyed by date stamps (`Stripe-Version: 2020-08-27`). Core backend services execute exclusively against the single, latest domain schema. At the API gateway tier, declarative version shims intercept incoming legacy requests, transforming outdated payloads forward into current domain models. Following business execution, reverse transformation shims convert modern responses back into the precise JSON structure expected by that caller's historical version.
- **The System Outcome**: Stripe has maintained backward compatibility for over a decade without breaking a single merchant integration. Core engineering teams maintain only one unified production codebase, while legacy compatibility is isolated to lightweight transformation shims.

---

## 10. Interesting Facts & Production Gotchas

1. **The Expand and Contract Pattern**:  
   Never execute sudden in-place schema changes. Always execute across three phases:
   - *Phase 1 (Expand)*: Deploy new fields alongside legacy fields, dual-writing to both.
   - *Phase 2 (Migrate)*: Progressively migrate consumers to read from the new attributes.
   - *Phase 3 (Contract)*: Once telemetry confirms zero traffic on legacy fields, safely decommission the old schema.
2. **Mobile App Store Review Latency**:  
   Never deploy breaking backend changes concurrently with mobile app store submissions. App Store approval can take 24 to 72 hours—or be rejected entirely! Decommissioning an old API before the approved binary reaches end users leaves mobile customers stranded.
3. **The Mobile Force-Update Kill Switch**:  
   Always include a dynamic configuration endpoint or header (`X-Minimum-App-Version: 4.0.0`) from day one. If a critical vulnerability requires severing legacy endpoints immediately, mobile clients can render a modal prompting users to update via the app store.

---

## 11. Key Takeaways

- **Select the Right Strategy**: Choose URI Path (`/v1/`) for routing simplicity and CDN caching efficiency; adopt Header-based versioning for strict REST purity.
- **Distinguish Breaking from Non-Breaking**: Adding optional fields is backward-compatible; renaming fields or mutating data types represents a breaking change requiring a new API version.
- **Adhere to IETF Standards**: Leverage standard `Deprecation: true` and `Sunset: <date>` HTTP headers to communicate endpoint expiration dates automatically to client runtimes.
- **Execute Expand and Contract Transitions**: Phase schema migrations progressively with temporary dual-writes until client telemetry verifies complete adoption.
