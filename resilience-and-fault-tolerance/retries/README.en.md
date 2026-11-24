# Retries

## 1. Real-World System Scenario
In automated logistics fulfillment pipelines, the `Order Fulfillment Service` makes outbound HTTP calls to an external `Logistics Carrier API` to generate shipping tracking waybills.
In automated logistics fulfillment pipelines, the `Order Fulfillment Service` makes outbound HTTP calls to an external `Logistics Carrier API` to generate shipping tracking waybills.

In distributed cloud networking, short-lived disruptions (*transient faults*) are an inevitable reality:
- The carrier's ingress proxy might be executing a 2-second container pod rotation, a cloud router encounters momentary packet loss, or a socket disconnects abruptly (*Connection Reset by Peer*), returning an initial `HTTP 503 Service Unavailable`.
- Without Retries: The backend assumes the failure is catastrophic and permanent. The order status stalls, shipping is halted, and operations staff are forced into manual incident resolution.
- Conversely, if the carrier returns `HTTP 400 Bad Request` (malformed postal code) or `HTTP 401 Unauthorized` (expired API token), endlessly hammering the endpoint will never succeed and merely burns compute bandwidth.

**The Configured Retries Solution:**
You configure an automated retry policy: when a request fails with an identifiable transient error (`502`, `503`, `504`, or TCP drop), the client automatically re-attempts the request up to 3 times after a short pause. The second attempt succeeds 500 milliseconds later, the tracking number is registered, and customer fulfillment proceeds without manual intervention (*self-healing API interaction*).

## 2. What Is It and What Is It For?

Retries is an architectural resilience pattern where a client or calling service automatically repeats a failed network operation, operating under the assumption that the underlying failure is temporary and recoverable (*transient faults*).

Errors that **SHOULD** be retried (*Transient Faults*):
- `503 Service Unavailable` (server is temporarily shedding load).
- `504 Gateway Timeout` or socket read timeouts.
- Raw network socket disconnects (`ECONNRESET`, packet loss during router failovers).
- Momentary database concurrency deadlocks (optimistic locking conflicts).

Errors that **MUST NEVER** be retried (*Permanent / Non-Transient Faults*):
- `400 Bad Request` (malformed JSON syntax; sending it a million times will never make it valid).
- `401 Unauthorized` or `403 Forbidden` (invalid authentication credentials or insufficient permissions).
- `404 Not Found` (the requested resource does not exist).
- `422 Unprocessable Entity` (domain validation failed, e.g., insufficient bank account balance).

## 3. Why Use It in High-Throughput Systems?

In modern cloud computing, networks are inherently unreliable (*Fallacies of Distributed Computing*).

In a massive high-throughput deployment:
- Kubernetes pods reschedule and rotate nodes constantly.
- Cloud software-defined networks occasionally recalculate routing tables.
- Upstream reverse proxies periodically reap idle keep-alive connections.

If your backend immediately surfaces a fatal user error upon encountering a 5-millisecond network blip, your system's perceived availability SLA will be unacceptable. Introducing a disciplined 1-to-2 retry policy resolves upwards of 90% of transient hiccups transparently without human intervention.

## 4. How It Works (Data Flow Diagram)

```
Client / Service A                     Service B (Target)
      |                                        |
      | 1. HTTP Request (Attempt 1)            |
      |--------------------------------------->| (Target pod restarting)
      | <--------------------------------------|
      | 2. Error: 503 / Connection Refused     |
      |                                        |
 [ Check Error Classification ]                |
   Is Transient? -> YES                        |
   Is Retry Budget Available? -> YES (1/3)     |
      |                                        |
  [ Wait Interval / Backoff Delay ]            |
      |                                        |
      | 3. HTTP Request (Attempt 2)            |
      |--------------------------------------->| (Pod is healthy again!)
      | <--------------------------------------|
      | 4. HTTP 200 OK + Payload               |
      v                                        v
[ Successful Operation ]
```

## 5. Core Implementation Logic (Smart Retry Loop di Go)

Below is the production implementation logic in Go retry cerdas dengan penyaringan status code yang tepat di Go:

```go
package main

import (
	"context"
	"errors"
	"net/http"
	"time"
)

// ExecuteWithRetry menjalankan request HTTP dengan filter status code yang tepat
func ExecuteWithRetry(ctx context.Context, reqFn func() (*http.Response, error), maxRetries int) (*http.Response, error) {
	var resp *http.Response
	var err error

	for attempt := 1; attempt <= maxRetries; attempt++ {
		resp, err = reqFn()

		// Jika sukses dan status code 2xx atau 4xx (client error), jangan di-retry
		if err == nil && resp.StatusCode < 500 {
			return resp, nil
		}

		// Hanya retry jika error jaringan (I/O timeout) atau server error 502/503/504
		if resp != nil && (resp.StatusCode == http.StatusBadGateway ||
			resp.StatusCode == http.StatusServiceUnavailable ||
			resp.StatusCode == http.StatusGatewayTimeout) {
			// Status sementara, boleh retry
		} else if err == nil {
			return resp, nil // Error 500 biasa atau status lainnya, hentikan retry
		}

		if attempt < maxRetries {
			select {
			case <-time.After(time.Duration(attempt*100) * time.Millisecond):
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		}
	}

	if err != nil {
		return nil, err
	}
	return nil, errors.New("request gagal setelah batas maksimal retry")
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Automated Self-Healing**: Dampens momentary network jitters seamlessly.
- **Enhanced Success SLA**: Converts transient network failures into successful transactions behind the scenes.

### Cons
- **The "Retry Storm" Nightmare**: If a downstream service is struggling under heavy load, and 10,000 upstream clients simultaneously fire 3 retries each, total incoming traffic instantly amplifies by 300% (from 10k to 40k req/s), driving the struggling service into complete collapse.
- **Latency Inflation**: Repeated retry cycles multiply the end-to-end response time experienced by the end-user.
- **Duplicate Execution Hazards**: Retrying an unacknowledged payment transaction can result in double-charging customer accounts.

## 7. When to Use vs When Not to Use

### When to Use
- Exclusively on **Idempotent** operations (actions that yield the exact same system state regardless of how many times they run, such as `GET`, `PUT`, `DELETE`, or `POST` protected by an *Idempotency-Key*).
- Asynchronous background worker tasks or message queue consumers.

### When Not to Use
- **Non-Idempotent** mutating operations lacking idempotency tokens (e.g., `/api/v1/checkout` or `/api/v1/charge-card`).
- When encountering standard client application validation errors (`4xx`).
- When the parent request context deadline has nearly expired.

## 8. Key Metrics & Observability (Monitoring Dashboard)

- **`http_client_retries_total` (Counter)**: Aggregate volume of retries triggered across services.
- **`retry_success_ratio` (Gauge)**: Proportion of retried calls that eventually succeed (`Successful Retries / Total Retries`). If this plunges below 10%, downstream services are dead, not transiently jittering.
- **`retry_budget_exhaustions_total` (Counter)**: Operations terminating because retry limits were reached.

## 9. Real-World Case Studies (Tech Giants)

- **Envoy Proxy (Lyft & Istio)**: Envoy enforces automated **Retry Budgets**. By default, it tracks rolling retry ratios: if retries breach 20% of normal traffic, all retries are rejected globally to prevent upstream clusters from crushing struggling downstream services.
- **Stripe**: Official Stripe client SDKs automatically execute up to 2 network retries on HTTP 500/503 errors. Because Stripe enforces unique `Idempotency-Key` headers on initial mutation attempts, client-side automated retries are transactionally secure and never duplicate customer charges.

## 10. Interesting Facts & Production Gotchas

- **The Legendary Retry Storm Disaster**: In high-throughput architectures, the primary reason a collapsed database fails to recover after restarting is the *Retry Storm*. While the DB was restarting, thousands of app servers queued up aggressive immediate retries. The moment port 5432 opens, millions of queued connections flood the database, instantly killing it again.  
  *Mandatory Rules:* Always cap retries (maximum 2 or 3 attempts), combine with **Exponential Backoff and Jitter**, and guard with a **Circuit Breaker**.
- **Retry Budgets**: Advanced distributed systems (like Google infrastructure and Envoy Proxy) enforce *Retry Budgeting*. A client tracks its retry-to-request ratio in a rolling window: if retry attempts exceed 10% of total normal traffic, all retries are immediately short-circuited to protect downstream ecosystem health.
