# Circuit Breakers

## 1. Real-World System Scenario
In e-commerce checkout platforms processing thousands of requests per minute, the `Checkout Service` executes outbound HTTP calls to an external `Fraud Detection Service` to evaluate account risk scores.
In e-commerce checkout platforms processing thousands of requests per minute, the `Checkout Service` executes outbound HTTP calls to an external `Fraud Detection Service` to evaluate account risk scores.

Suddenly, the Fraud Detection Service encounters severe memory leaks and locks up: every incoming request hangs until hitting a hard 5-second timeout.

If your architecture lacks circuit-breaking protection:
- Thousands of concurrent checkout requests stall for 5 seconds each.
- Operating system socket descriptors and HTTP worker thread pools on the `Checkout Service` are rapidly depleted awaiting responses.
- Catastrophe ensues: the core `Checkout Service` crashes completely, taking down the entire e-commerce storefront just because a single ancillary support service suffered an outage (*cascading failure*).

**The Circuit Breaker Solution (e.g., Netflix Hystrix / Resilience4j):**
You wrap the remote fraud check in a Circuit Breaker. When the breaker detects that 50% of outbound calls over the last 10 seconds have timed out or failed, it automatically trips to the **OPEN state**. Subsequent checkout requests completely bypass the broken network dependency (< 1 ms fail-fast) and trigger an immediate *Graceful Fallback* (e.g., allow small transactions through while queueing an asynchronous fraud review). The core checkout pipeline remains 100% operational for customers.

## 2. What Is It and What Is It For?

A Circuit Breaker is an architectural resilience pattern designed to detect failures in remote dependencies and **instantly sever outbound traffic** when error rates breach defined thresholds.

Instead of allowing incoming threads to stall awaiting timeouts from an already broken service, the circuit breaker immediately short-circuits execution, returning a sub-millisecond *Fail-Fast* error or a designated *Graceful Fallback*.

### The Three States (*State Machine*):
1. **CLOSED (Normal Operation)**:
   - Request traffic flows unobstructed to the downstream dependency.
   - The circuit breaker continuously collects rolling metrics: total requests, success rates, and timeout ratios.
   - If the error percentage breaches a threshold (e.g., > 50% failures over a 10-second rolling window), the circuit trips to **OPEN**.

2. **OPEN (Tripped / Short-Circuited)**:
   - The connection is severed. Incoming requests **ARE NOT forwarded to the downstream dependency**.
   - The caller returns immediate fail-fast exceptions or fallback payloads (0ms network delay).
   - The downstream service is completely shielded from incoming pressure, allowing it to recover.
   - A cooldown timer (*Sleep Window*, e.g., 30 seconds) begins ticking.

3. **HALF-OPEN (Canary Probe Trial)**:
   - Once the cooldown timer expires, the breaker admits a small trial sample of traffic (e.g., 5 to 10 canary calls) through to the downstream host.
   - If all trial requests succeed, the breaker concludes the downstream service is healthy and transitions to **CLOSED**.
   - If any trial request fails, the breaker trips immediately back to **OPEN**, resetting the cooldown timer.

Popular libraries: Resilience4j (Java), Sony/gobreaker (Go), Polly (.NET), Opossum (Node.js).

## 3. Why Use It in High-Throughput Systems?

Consider an e-commerce platform during an active Flash Sale:
The Order Service calls a Recommendation Service to populate "Customers Also Bought" items. Suddenly, the Recommendation Service encounters an out-of-memory crash and stops responding.

Without a Circuit Breaker:
- Every single customer attempting to check out is forced to wait for a 3-second timeout on the Recommendation API.
- Thousands of worker threads in the Order Service lock up waiting for recommendations that will never arrive.
- Eventually, the Order Service exhausts its thread pool and crashes. A mission-critical capability (revenue checkout) was destroyed by a non-essential supplementary widget.

With a Circuit Breaker:
- After 20 consecutive timeout errors, the breaker trips to **OPEN**.
- The next 10,000 checkout requests bypass the Recommendation Service entirely.
- The Order Service instantly renders an empty recommendation array `[]` and the customer completes their checkout in under 50 milliseconds!

## 4. How It Works (State Machine Flow Diagram)

```
       +-------------------------------------------------------+
       |                                                       |
       |  Request Succeeds (Healthy Metric Window)             |
       v                                                       |
+--------------+        Failure Threshold Exceeded             |
|    CLOSED    | -------------------------------------\        |
| (Normal Ops) |                                      |        |
+--------------+                                      v        |
       ^                                       +--------------+
       |                                       |     OPEN     |
       |                                       | (Fail-Fast / |
       |                                       |   Fallback)  |
       |                                       +--------------+
       | Canary Calls Succeed                         |
       |                                              | Cooldown Expires
       |                                              | (e.g. 30 seconds)
       |               +---------------+              |
       \-------------- |   HALF-OPEN   | <------------/
                       | (Trial Probe) |
                       +---------------+
                               |
                               | Canary Call Fails
                               v
                        (Re-trips to OPEN)
```

## 5. Core Implementation Logic (State Machine Circuit Breaker di Go)

Below is the state machine implementation in Go Circuit Breaker di Go:

```go
package main

import (
	"errors"
	"sync"
	"time"
)

type State int

const (
	StateClosed State = iota
	StateOpen
	StateHalfOpen
)

type CircuitBreaker struct {
	mu               sync.Mutex
	state            State
	failureCount     int
	failureThreshold int
	recoveryTimeout  time.Duration
	lastFailureTime  time.Time
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
	cb.mu.Lock()
	now := time.Now()

	// Jika OPEN dan waktu recovery sudah lewat, beralih ke HALF-OPEN untuk uji coba
	if cb.state == StateOpen && now.Sub(cb.lastFailureTime) > cb.recoveryTimeout {
		cb.state = StateHalfOpen
	}

	if cb.state == StateOpen {
		cb.mu.Unlock()
		return errors.New("circuit breaker is OPEN (permintaan ditolak otomatis)")
	}
	cb.mu.Unlock()

	err := fn()

	cb.mu.Lock()
	defer cb.mu.Unlock()

	if err != nil {
		cb.failureCount++
		cb.lastFailureTime = time.Now()
		if cb.failureCount >= cb.failureThreshold || cb.state == StateHalfOpen {
			cb.state = StateOpen
		}
		return err
	}

	// Jika sukses di status HALF-OPEN, pulihkan sirkuit ke CLOSED
	if cb.state == StateHalfOpen {
		cb.state = StateClosed
		cb.failureCount = 0
	}
	return nil
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Cascading Failure Prevention**: Quarantines localized dependencies before they bring down the entire platform.
- **Superior Fail-Fast Characteristics**: Instantly reclaims CPU, RAM, and network socket threads when downstream dependencies fail.
- **Graceful Degradation**: Protects core transactional revenue streams by decoupling non-essential features under stress.

### Cons
- **Tuning Complexity**: Establishing correct thresholds requires deep operational profiling (how many errors justify tripping? how long should cooldowns last?).
- **Fallback Logic Overhead**: Requires developers to craft and test intelligent fallback responses (e.g., cached stale data vs default empty models).

## 7. When to Use vs When Not to Use

### When to Use
- All synchronous inter-service RPC / HTTP communication across microservices.
- Outbound third-party SaaS integrations (Payment Gateways, Logistics Trackers, SMS Providers).
- Remote caching clusters and secondary database connections.

### When Not to Use
- In-process internal function calls within the same application memory space.
- Asynchronous message consumers listening on event queues (the message queue already natively handles backpressure and dead-lettering).

## 8. Key Metrics & Observability (Monitoring Dashboard)

- **`circuit_breaker_state` (Gauge)**: Discrete enumeration (0=Closed, 1=Half-Open, 2=Open). High-severity alert fires if a state remains 2 (Open) for over 60 seconds.
- **`circuit_breaker_trip_total` (Counter)**: Tally of how frequently the circuit trips open.
- **`fallback_invocations_total` (Counter)**: Volume of calls diverted to graceful fallback branches.

## 9. Real-World Case Studies (Tech Giants)

- **Netflix (Hystrix)**: Netflix engineered **Hystrix** in 2011 following a major outage where a latency spike in an auxiliary user metadata service triggered Tomcat thread starvation, completely halting video playback globally. Hystrix pioneered wrapping all remote dependencies in isolated thread pools and circuit-breaker state machines.
- **FinTech Payment Routers**: Global payment processors wrap banking APIs in circuit breakers. If a major bank's acquiring gateway degrades, the circuit breaker opens and the router automatically steers user credit card authorizations to alternative clearing partner banks without dropping checkout conversion rates.

## 10. Interesting Facts & Production Gotchas

- **Absolute Count vs Error Percentage Trap**: Never configure a breaker based purely on *absolute error counts* (e.g., "trip if 10 errors occur"). Under 10,000 requests per second, 10 errors represents an insignificant 0.1% blip, yet your breaker will trip and sever traffic for thousands of healthy users! Always evaluate a **Minimum Request Volume** (e.g., minimum 50 calls in 10s) alongside an **Error Rate Percentage** (e.g., > 50% error rate).
- **The Silent Outage Danger of Fallbacks**: Engineering teams often write fallbacks so seamless (e.g., returning default empty arrays) that nobody notices the downstream service has been dead for two days because no alerts fired. Always wire circuit breaker state transitions directly into your monitoring telemetry (Prometheus, Datadog) to alert on-call teams immediately when a breaker trips.
