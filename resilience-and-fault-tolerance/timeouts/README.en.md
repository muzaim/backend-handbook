# Timeouts

## 1. Real-World System Scenario
In distributed microservices architectures, when a customer confirms an order payment, the `Order Service` executes an outbound HTTP network call to a third-party `Bank Payment Gateway` to charge a credit card.
In distributed microservices architectures, when a customer confirms an order payment, the `Order Service` executes an outbound HTTP network call to a third-party `Bank Payment Gateway` to charge a credit card.

If the backend HTTP client omits an explicit timeout deadline (defaulting to the operating system socket timeout of up to several hours):
- When the remote banking provider suffers silent packet loss or internal deadlocks, the TCP connection socket hangs unresolved.
- Every incoming checkout request claims and locks an active application worker thread / goroutine in the `Order Service`.
- Under moderate traffic, all 200 server worker threads become completely exhausted within 30 seconds (*thread pool exhaustion*).
- The catastrophic result: `Order Service` crashes entirely, refusing all incoming traffic—including unrelated read-only endpoints like order history—triggering a platform-wide cascading failure.

**The Timeout Solution (e.g., Connect Timeout 500ms, Read Timeout 2.5s):**
You configure an immutable 2.5-second deadline using `context.WithTimeout` on the outbound HTTP client. If the bank fails to respond within 2.5 seconds, the socket is immediately severed, the worker thread is returned to the execution pool, and the application quickly serves a graceful fallback error prompting the user to select an alternative payment method.

## 2. What Is It and What Is It For?

A Timeout is a deterministic deadline enforced by a caller waiting for an I/O or network operation to complete (such as an outbound HTTP call to a microservice, a database query, or reading a filesystem stream).

If the requested operation fails to respond before the clock expires, the runtime **abruptly aborts the connection**, raises an explicit error (e.g., `Context Deadline Exceeded`, `SocketTimeoutException`), and immediately releases thread and socket resources.

Essential timeout layers in backend systems:
- **Connection Timeout**: The maximum window allowed to complete the initial TCP 3-way handshake with the target host (typically aggressive: 500ms to 2 seconds).
- **Read/Socket Timeout**: The maximum allowable gap between received packets of data once the connection is established (e.g., 2 to 5 seconds).
- **Write Timeout**: The maximum duration permitted to flush an outbound request payload across the socket buffer.
- **Request/Global Deadline**: An absolute end-to-end deadline enforced from client ingestion to final response dispatch.

## 3. Why Use It in High-Throughput Systems?

In distributed backend architectures, the most destructive failure mode is not a downstream service that hard-crashes instantly—it is a downstream service that **degrades and responds excruciatingly slowly**.

When an upstream microservice takes 60 seconds to answer a single call:
- Each incoming client request holds 1 server worker thread/goroutine, 1 socket file descriptor, and associated stack memory for a full 60 seconds.
- At 500 requests per second, your service accumulates 5,000 completely frozen threads within a mere 10-second window.
- The operating system exhausts file descriptors, connection pools dry up, garbage collection chokes, and your service collapses in a catastrophic **cascading failure**.

By enforcing strict timeouts (e.g., 2 seconds), execution fails fast, leaked resources return to the connection pool, and the server continues processing healthy requests smoothly.

## 4. How It Works (Data Flow Diagram)

```
[ Without Timeouts: Indefinitely Hung Threads ]
Client -----> [ Service A (Locked Thread) ] ===== Waits 60s =====> [ Service B (Lagging) ]
Client -----> [ Service A (Locked Thread) ] ===== Waits 60s =====> [ Service B (Lagging) ]
(Thousands of threads accumulate until Service A crashes from Out-Of-Memory)


[ With Timeouts: Fail-Fast & Resource Reclamation ]
Client -----> [ Service A ]
                    |
                    | 1. Invoke Service B (Timeout Budget: 2 Seconds)
                    v
              [ Timer Running: 0s ... 1s ... 2s ]
                    |
                    +-- Service B has not responded after 2.0s!
                    |
                    v
              [ TIMEOUT TRIGGERED ]
                    |
                    | 2. Sever Socket, Release Worker Thread
                    | 3. Return Fallback / HTTP 504 Gateway Timeout
                    v
Client <----- [ Instant Response: "Service Temporarily Busy" ]
```

## 5. Core Implementation Logic (Konfigurasi Timeout HTTP Client di Go)

Below is the proper tiered timeout implementation in Go timeout bertingkat yang benar pada HTTP Client backend menggunakan Go Context:

```go
package main

import (
	"bytes"
	"context"
	"net/http"
	"time"
)

// CallDownstreamWithTimeout menerapkan timeout ketat pada level HTTP Client dan Context
func CallDownstreamWithTimeout(ctx context.Context, payload []byte) (*http.Response, error) {
	// 1. Batas timeout operasi keseluruhan via Context
	reqCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
	defer cancel()

	req, err := http.NewRequestWithContext(reqCtx, http.MethodPost, "http://payment-service:8080/charge", bytes.NewBuffer(payload))
	if err != nil {
		return nil, err
	}
	req.Header.Set("Content-Type", "application/json")

	// 2. Client HTTP dengan timeout koneksi bertingkat
	client := &http.Client{
		Timeout: 2500 * time.Millisecond,
		Transport: &http.Transport{
			ResponseHeaderTimeout: 1500 * time.Millisecond,
			TLSHandshakeTimeout:   1000 * time.Millisecond,
		},
	}

	return client.Do(req)
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Fail-Fast Predictability**: Prevents user interfaces and backend threads from stalling indefinitely.
- **Domino Cascade Prevention**: Quarantines localized latency anomalies before they propagate upstream.
- **SLA Enforcement**: Guarantees that public API endpoints honor contractual latency budgets.

### Cons
- **False Negatives**: Operations that were genuinely on the verge of successful completion downstream are aborted prematurely by the impatient caller.
- **Phantom State (Ghost Writes)**: A client marks a transaction failed due to a timeout, yet the downstream database successfully commits the row 500ms later, triggering distributed state drift.

## 7. When to Use vs When Not to Use

### When to Use
- **Every outbound network I/O call without exception**: Third-party SaaS APIs, SQL database queries, Redis calls, and internal gRPC/HTTP endpoints.
- Edge reverse proxies and API gateways (`proxy_read_timeout` in NGINX).

### When Not to Use (or requires detached long-running paradigms)
- Heavy asynchronous batch operations (e.g., generating a 5GB monthly financial audit report that legitimately takes 20 minutes). Never execute long batch workloads over synchronous HTTP; offload them to background Message Queues!

## 8. Key Metrics & Observability (Monitoring Dashboard)

- **`http_client_timeout_errors_total` (Counter)**: Rate of timeout occurrences per downstream dependency.
- **`request_duration_seconds` (p95/p99 Histogram)**: When p99 latency approaches the timeout boundary (e.g., creeping to 2.3s on a 2.5s deadline), a service degradation incident is imminent.
- **`thread_pool_active_workers` (Gauge)**: Monitors worker pool saturation caused by in-flight pending socket waits.

## 9. Real-World Case Studies

### Defeating Hanging Sockets & Goroutine Starvation in Payment Gateway Integrations

- **The Engineering Problem**:
  An e-commerce transaction platform authorized credit card charges through a third-party payment aggregator. Go backend services communicated using standard `http.DefaultClient` instances configured without explicit timeouts (`Timeout: 0`). During an upstream datacenter disruption, the external payment service deadlocked: TCP handshakes completed, but the server never returned response bytes or closed sockets (*hanging socket lockup*). Over 90 seconds, 4,500 checkout requests arrived. Each request spawned a Go worker goroutine that blocked indefinitely on network I/O (`IO Wait`). Active goroutines spiked from 150 to over 4,500, memory allocations exploded, and the Linux kernel initiated repeated `OOMKilled` terminations of the core checkout service.

- **The Architectural Solution**:
  1. **Granular Transport & Context Deadlines**: Platform engineers replaced the standard client with an explicitly bounded HTTP transport:
     ```go
     transport := &http.Transport{
         DialContext: (&net.Dialer{Timeout: 1 * time.Second}).DialContext,
         TLSHandshakeTimeout: 1 * time.Second,
         ResponseHeaderTimeout: 2 * time.Second,
     }
     client := &http.Client{Transport: transport}
     ```
  2. **End-to-End Context Propagation**: Every incoming transaction was bounded with `context.WithTimeout(ctx, 3500*time.Millisecond)`. This context was passed to downstream SQL transactions and network calls, ensuring all resources abort if the deadline expires.
  3. **Deterministic Fast Failure**: When the 3.5-second deadline expired, the goroutine immediately recycled into the pool, TCP sockets closed, and the application emitted a `PENDING_VERIFICATION` transaction state without leaving customers blocked on frozen checkout pages.

- **The Engineering Outcome**:
  - Active goroutine concurrency remained tightly bounded under 300, and `OOMKilled` incidents were completely eliminated during external provider freezes.
  - Deterministic fail-fast behavior was established: stalled calls aborted at precisely 3,500ms, preserving compute resources for healthy transaction traffic.

## 10. Interesting Facts & Production Gotchas

- **The Danger of Default Infinite Timeouts**: In many popular programming languages (such as the default `http.Client` in Go or Python's default `requests`), the standard network timeout is **ZERO (meaning INFINITE / No Timeout)**! If a remote server hangs, your application will hang forever until the Linux kernel kills the process. Never deploy default HTTP clients to production.
- **Timeout Inversion (The Upside-Down Timeout Disaster)**: This is one of the most widespread microservice anti-patterns:  
  *Example:* Web Browser timeout = 3s. API Gateway timeout = 5s. Core Microservice timeout = 10s.  
  *The Consequence:* At second 3, the user's browser abandons the connection. However, your API Gateway and internal microservices continue burning CPU cycles, disk I/O, and database connections until second 10, computing an answer that has already been discarded.  
  *Golden Rule:* **Timeouts must cascade downwards—the deepest layer must have the shortest timeout!**

## 11. Key Takeaways

1. **Never Trust Default Timeouts**: Runtimes such as Go's `http.DefaultClient` default to zero (infinite wait); always define explicit timeouts across all outbound network calls.
2. **Propagate Context Deadlines**: Propagate `context.Context` cancellation budgets through database and RPC layers to abort orphaned calculations before wasting CPU cycles.
3. **Respect Timeout Hierarchies**: Deep downstream services must enforce shorter timeouts than ingress layers to avoid wasteful *Timeout Inversions*.
4. **Isolate Third-Party Failures**: Never permit an external vendor's latency degradation to exhaust internal thread pools and take down your core application.
