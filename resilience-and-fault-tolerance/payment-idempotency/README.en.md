# Idempotency Key & Payment Deduplication (Transactional Boundary)

## 1. Simple Analogy & Real-World Scenario

You are at a convenience store cashier and swipe your debit card on the payment terminal. Right after you press **Enter PIN**, the terminal screen suddenly freezes on a loading spinner because the store's cellular connection momentarily dropped. The cashier assumes the payment did not go through and asks you to tap your card a second time.

If the issuing bank lacks an **Idempotency Key & Deduplication** architecture, your checking account would immediately be charged twice for the exact same grocery bag. In modern banking infrastructure, however, the initial request was tagged with a unique transaction reference number (*RRN - Retrieval Reference Number* or *Idempotency Key*). When the second transaction hits the bank with the exact same identifier, the system detects the duplicate, prevents a second deduction, and replays the original receipt that was already approved.

In backend engineering, this exact failure mode occurs thousands of times daily: mobile users abruptly transition between Wi-Fi and LTE mid-checkout, HTTP client libraries trigger automatic retries upon receiving transient socket timeouts, or impatient users aggressively double-click the "Pay Now" button.

---

## 2. Definition & Core Concepts

**Idempotency** is the mathematical and architectural property of an API operation where executing it once or multiple times with identical arguments produces the exact same server-side state and system outcome.

Within HTTP specifications:
- `GET`, `PUT`, and `DELETE` are inherently idempotent.
- `POST` and `PATCH` are **non-idempotent by default** (creating or mutating new system resources on every invocation).

To make mutating payment endpoints (`POST /v1/payments/charges`) impervious to duplicate charges, tech companies employ the **Idempotency Key Pattern**:
1. **Idempotency-Key Header**: The client generates a unique UUIDv4 on the client/frontend prior to initiating the network call and attaches it via HTTP headers: `Idempotency-Key: 9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d`.
2. **Payload Fingerprinting (SHA-256)**: The backend calculates a cryptographic hash of the raw request payload alongside the path and authenticated User ID to ensure the client is not reusing the same key with conflicting arguments (*payload tampering*).
3. **Distributed State Machine**: The key transitions through a strict lifecycle:
   - `STARTED` / `IN_PROGRESS`: Key is acquired via atomic distributed locking (e.g. Redis `SETNX`) while the database transaction runs.
   - `SUCCESS`: The database transaction commits, external banking rails succeed, and an immutable response snapshot (HTTP status + response body) is cached.
   - `FAILED`: Transaction cleanly rolls back or card is declined; the lock is released or marked failed so the client can safely retry with adjusted parameters.
4. **Cached Response Replay**: If an incoming request matches an existing key whose status is `SUCCESS`, the backend **bypasses core domain processing completely** and replays the cached snapshot verbatim.

---

## 3. Why It Matters at Production Scale

Across high-throughput fintech, payment gateway, and e-commerce platforms handling tens of thousands of transactions per second (TPS):

1. **Zero Double-Charges & Financial Loss Prevention**: Without deduplication, transient connection drops trigger automatic retries that charge credit cards or digital wallets multiple times, resulting in bank fines, dispute processing fees, and customer churn.
2. **Protection Against Duplicate Disbursements**: For driver payouts, seller remittances, or payroll transfers, an unchecked duplicate transfers real capital out of company banking reserves without programmatic recovery guarantees.
3. **Preventing Inventory Double-Allocation**: Prevents single buyers from inadvertently booking multiple hotel suites, flight seats, or flash-sale units during concurrent checkout retries.
4. **Solving the Distributed Two Generals' Problem**: In distributed networking, a client never knows whether a timeout occurred on the outbound flight, during database processing, or on the return leg. The idempotency key deterministically reconciles this uncertainty server-side.

---

## 4. How It Works & Sequence Flow (ASCII)

Here is the transactional lifecycle of an Idempotency Key within a payment service:

```
Client (App/Web)                API Gateway / Backend Service              Redis Lock / PostgreSQL
      |                                       |                                       |
      |-- 1. POST /charges ------------------>|                                       |
      |   (Header: Idempotency-Key: <UUID>)   |                                       |
      |   (Body: {"amount": 50000})           |-- 2. Compute SHA-256(Body) ---------->|
      |                                       |                                       |
      |                                       |-- 3. Atomic SETNX key:lock (TTL 60s)->|
      |                                       |<-- Lock Acquired (Success) -----------|
      |                                       |                                       |
      |                                       |-- 4. Check DB for existing key ------>|
      |                                       |<-- Not found (Brand New Request) -----|
      |                                       |                                       |
      |                                       |-- 5. INSERT (key, hash, 'STARTED')--->|
      |                                       |                                       |
      |                                       |-- 6. Execute Payment & DB Debit ----->|
      |                                       |-- 7. UPDATE (key, 'SUCCESS', resp)--->|
      |                                       |-- 8. Release Redis Lock ------------->|
      |                                       |                                       |
      |<-- 9. HTTP 201 Created ---------------|                                       |
      |    (Body: {"charge_id": "ch_123"})    |                                       |
      |                                       |                                       |
   ======================= DUPLICATE SCENARIO (NETWORK RETRY) =========================
      |                                       |                                       |
      |-- 10. POST /charges (RETRY) --------->|                                       |
      |    (Same Idempotency-Key & Payload)   |-- 11. Compute SHA-256(Body) --------->|
      |                                       |                                       |
      |                                       |-- 12. Check DB / Cache for key ------>|
      |                                       |<-- Found: Status='SUCCESS', resp_json-|
      |                                       |                                       |
      |<-- 13. HTTP 201 Created (REPLAY) -----| (NO PAYMENT PROCESSED, ZERO DOUBLE CHARGE)
      |    (Cached Body: {"charge_id":...})   |                                       |
      |                                       |                                       |
   ======================= HIGH-CONCURRENCY SCENARIO (RACE CONDITION) =================
      |                                       |                                       |
      |-- 14. POST /charges (Rapid Click) --->|                                       |
      |    (Concurrent duplicate request)     |-- 15. Atomic SETNX key:lock --------->|
      |                                       |<-- Lock Failed: Already Locked -------|
      |                                       |                                       |
      |<-- 16. HTTP 409 Conflict -------------| (Request in-flight, please wait)      |
```

---

## 5. Concrete Implementation Example (Go)

The following example illustrates a production-ready payment handler in Go featuring:
- Extraction and validation of the `Idempotency-Key` header.
- Request payload fingerprinting via `SHA-256`.
- Non-blocking atomic distributed locking simulating Redis `SET NX EX`.
- State tracking and cached response replaying.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"sync"
	"time"
)

// IdempotencyRecord represents an immutable transaction snapshot in persistence.
type IdempotencyRecord struct {
	Key          string    `json:"key"`
	RequestHash  string    `json:"request_hash"`
	Status       string    `json:"status"` // "STARTED", "SUCCESS", "FAILED"
	ResponseCode int       `json:"response_code"`
	ResponseBody []byte    `json:"response_body"`
	CreatedAt    time.Time `json:"created_at"`
}

// MockStorage simulates combined PostgreSQL persistence and Redis distributed locking.
type MockStorage struct {
	mu      sync.Mutex
	locks   map[string]time.Time
	records map[string]*IdempotencyRecord
}

var store = &MockStorage{
	locks:   make(map[string]time.Time),
	records: make(map[string]*IdempotencyRecord),
}

// AcquireLock simulates redis.SetNX(ctx, "lock:"+key, "1", 60*time.Second)
func (s *MockStorage) AcquireLock(key string, ttl time.Duration) bool {
	s.mu.Lock()
	defer s.mu.Unlock()

	exp, exists := s.locks[key]
	if exists && time.Now().Before(exp) {
		return false // Concurrently locked by an in-flight worker
	}
	s.locks[key] = time.Now().Add(ttl)
	return true
}

func (s *MockStorage) ReleaseLock(key string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	delete(s.locks, key)
}

func (s *MockStorage) GetRecord(key string) (*IdempotencyRecord, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()
	rec, exists := store.records[key]
	return rec, exists
}

func (s *MockStorage) SaveRecord(rec *IdempotencyRecord) {
	s.mu.Lock()
	defer s.mu.Unlock()
	store.records[rec.Key] = rec
}

type PaymentRequest struct {
	AccountID string  `json:"account_id"`
	Amount    float64 `json:"amount"`
	Currency  string  `json:"currency"`
}

type PaymentResponse struct {
	TransactionID string `json:"transaction_id"`
	Status        string `json:"status"`
	Message       string `json:"message"`
}

// PaymentChargeHandler handles payment mutations guarded by an idempotency boundary.
func PaymentChargeHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Method Not Allowed", http.StatusMethodNotAllowed)
		return
	}

	idempotencyKey := r.Header.Get("Idempotency-Key")
	if idempotencyKey == "" {
		http.Error(w, `{"error":"Missing required header: Idempotency-Key"}`, http.StatusBadRequest)
		return
	}

	// 1. Read and compute SHA-256 fingerprint to detect payload tampering
	bodyBytes, err := io.ReadAll(r.Body)
	if err != nil {
		http.Error(w, `{"error":"Unable to read request body"}`, http.StatusBadRequest)
		return
	}
	r.Body = io.NopCloser(bytes.NewBuffer(bodyBytes)) // Rewind stream

	hasher := sha256.New()
	hasher.Write(bodyBytes)
	currentHash := hex.EncodeToString(hasher.Sum(nil))

	// 2. Check if a prior execution record already exists
	if existing, found := store.GetRecord(idempotencyKey); found {
		// Detect if the key was reused with conflicting parameters
		if existing.RequestHash != currentHash {
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusUnprocessableEntity)
			w.Write([]byte(`{"error":"Idempotency-Key reused with different request payload"}`))
			return
		}

		// Check if a concurrent request is currently in-flight
		if existing.Status == "STARTED" {
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusConflict)
			w.Write([]byte(`{"error":"A transaction with this Idempotency-Key is currently in progress"}`))
			return
		}

		// CACHED REPLAY: Replay original response snapshot
		w.Header().Set("Content-Type", "application/json")
		w.Header().Set("X-Cache-Lookup", "HIT - Idempotent Replay")
		w.WriteHeader(existing.ResponseCode)
		w.Write(existing.ResponseBody)
		return
	}

	// 3. Acquire non-blocking distributed lock to prevent concurrent races
	lockKey := "idemp_lock:" + idempotencyKey
	if !store.AcquireLock(lockKey, 30*time.Second) {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusConflict)
		w.Write([]byte(`{"error":"Concurrent request locked, please retry shortly"}`))
		return
	}
	defer store.ReleaseLock(lockKey)

	// Persist initial state as STARTED
	initRecord := &IdempotencyRecord{
		Key:         idempotencyKey,
		RequestHash: currentHash,
		Status:      "STARTED",
		CreatedAt:   time.Now(),
	}
	store.SaveRecord(initRecord)

	// 4. Execute core domain payment logic
	var payReq PaymentRequest
	if err := json.Unmarshal(bodyBytes, &payReq); err != nil {
		http.Error(w, `{"error":"Invalid JSON payload"}`, http.StatusBadRequest)
		return
	}

	txID := fmt.Sprintf("tx_charge_%d", time.Now().UnixNano())
	payResp := PaymentResponse{
		TransactionID: txID,
		Status:        "SETTLED",
		Message:       fmt.Sprintf("Successfully charged %.2f %s", payReq.Amount, payReq.Currency),
	}

	respBytes, _ := json.Marshal(payResp)

	// 5. Update transaction state to SUCCESS and store immutable response snapshot
	initRecord.Status = "SUCCESS"
	initRecord.ResponseCode = http.StatusCreated
	initRecord.ResponseBody = respBytes
	store.SaveRecord(initRecord)

	// 6. Return response to client
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	w.Write(respBytes)
}

func main() {
	http.HandleFunc("/v1/payments/charges", PaymentChargeHandler)
	log.Println("Payment Gateway running on :8080...")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 6. Trade-offs: Advantages and Disadvantages

### Advantages
- **Deterministic Financial Consistency**: Completely mitigates double-spending and duplicate card billing across distributed systems.
- **Safe Automated Retries**: Downstream mobile apps and services can apply aggressive exponential backoff retries without corrupting database ledgers.
- **Network Partition Tolerance**: Transparently bridges edge network flakiness between clients and gateway layers.

### Disadvantages
- **Latency Overhead**: Every transaction incurs round-trip latency to check distributed locks (Redis) and query database records before executing business logic.
- **Storage and Purging Costs**: High-throughput payment systems generate millions of idempotency keys daily, demanding automated TTL partitioning and background pruning (typically 24–48 hour retention).
- **Failure State Nuance**: System designers must strictly discern whether an infrastructure failure (e.g. transient DB timeout) should be recorded as a permanent `FAILED` state or immediately unlocked for retries.

---

## 7. When to Use vs When Not to Use

### When to Use
- **Financial Mutations**: Top-up, card debit/charge, bank transfer, disbursement, and refund operations.
- **Checkout & Inventory Allocations**: E-commerce cart checkouts, flight/seat reservations, and digital order creation.
- **Outbound Webhook Dispatching**: Delivering webhooks to external third-party partners to prevent repeated event processing.

### When Not to Use
- **Pure Read Operations**: `GET`, `HEAD`, and `OPTIONS` requests that are naturally idempotent. Performing lock lookups on reads adds unnecessary compute drag.
- **High-Frequency Telemetry Ingestion**: Logging or GPS telemetry pings arriving every 500ms where subsequent packets supersede older frames.
- **Absolute State Overwrites**: Simple profile updates like `PUT /users/avatar` where the latest write safely overwrites previous values without cumulative financial impacts.

---

## 8. Key Metrics & Observability

Monitor these operational signals in production:

1. **Idempotency Hit Rate (`idempotency_cache_hits_total`)**: Percentage of requests served directly from the cached response replay. Spikes typically indicate client auto-retry loops or mobile network degradation.
2. **Hash Mismatch Counter (`idempotency_hash_mismatch_total`)**: Count of requests reusing existing keys with altered payload bodies. Alerts engineers to client-side payload tampering or UUID reuse bugs.
3. **Lock Contention Rate (`idempotency_lock_conflicts_total`)**: Rate of `409 Conflict` responses returned due to concurrent in-flight requests. High rates suggest missing frontend UI debouncing.
4. **Idempotency Query Latency (p99)**: Round-trip lookup overhead for cache and database state checks (target: < 5ms).

---

## 9. Real-World Case Study

### Scenario: Stripe Payment API & Mobile Network Flakiness

#### Problem
Stripe processes billions of dollars across global merchants. On unpredictable 3G/4G cellular connections in developing markets, mobile clients frequently experienced network timeouts mid-payment. If merchants or SDKs triggered automatic retries after a socket disconnect when the transaction had already cleared upstream card networks, buyers suffered double charges. This triggered customer panic, merchant fines, and surge in credit card dispute fees.

#### Architecture Solution
Stripe engineered a globally distributed **Idempotency Layer**:
1. All client SDKs enforce or automatically generate an `Idempotency-Key` UUIDv4 header before initiating TCP/TLS handshakes.
2. The ingress gateway computes an SHA-256 fingerprint encompassing endpoint path, method, client account ID, and serialized request body.
3. Idempotency records are maintained across a low-latency distributed store with atomic concurrency locks. If a matching key with a completed status arrives, the stored response body and status code are replayed without engaging downstream payment processors.
4. Records expire automatically after a 24-hour TTL window.

#### Outcome / Impact
- **Zero Double Charges**: Eradicated 100% of duplicate card charges caused by network disconnects and client retry loops worldwide.
- **Developer Confidence**: Allowed client developers to safely configure aggressive auto-retry policies with exponential backoff on flaky cellular networks.
- **Industry Standardization**: Stripe's `Idempotency-Key` header pattern became the basis for official IETF draft RFC specifications for financial HTTP APIs.

---

## 10. Production Gotchas & Best Practices

1. **JSON Key Ordering in Hash Fingerprinting**: JSON keys are un-ordered (`{"amount":100, "user":"A"}` vs `{"user":"A", "amount":100}`). If middleware parses and re-encodes payloads, non-deterministic key ordering alters the SHA-256 hash! Always hash the raw byte stream as received, or use canonical JSON formatting before hashing.
2. **Zombie "In-Progress" Locks**: If a backend instance crashes (OOM or Kubernetes pod kill) while a transaction is marked `STARTED`, the key remains locked indefinitely. Always set lease TTLs on Redis locks and implement background sweepers to reconcile stuck transactions against banking records.
3. **Never Cache Transient 5xx Errors**: If a database replica fails transiently with a 500/503 error, **do not** cache this error as a completed idempotency snapshot. Release the lock to allow subsequent client retries to succeed.
4. **Always Scope Keys to Tenant/User**: Enforce compound unique constraints in persistence (`UNIQUE(merchant_id, idempotency_key)`). Relying solely on raw client-supplied UUIDs allows malicious actors to squat on keys belonging to other tenants.

---

## 11. Summary

1. **Primary Purpose**: Prevents double charges, duplicate bookings, and financial inconsistencies when clients retry requests over unstable networks.
2. **Core Mechanism**: Combines client-provided `Idempotency-Key` headers (UUIDv4), `SHA-256` payload fingerprinting, and a transactional state machine (`STARTED`, `SUCCESS`, `FAILED`).
3. **Response Replay**: Subsequent duplicate requests with matching payloads bypass business logic and immediately replay cached responses (*cached response replay*).
4. **Concurrency Safety**: Employs non-blocking distributed locking (e.g., Redis `SET NX EX`) to reject concurrent identical requests with `409 Conflict`.
