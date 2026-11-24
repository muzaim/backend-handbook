# Idempotency

## 1. Real-World System Scenario
On digital payment gateways, a mobile app user taps the "Pay Now" button to submit a $50 order payment (`POST /api/v1/payments`).
On digital payment gateways, a mobile app user taps the "Pay Now" button to submit a $50 order payment (`POST /api/v1/payments`).

The backend Payment Service receives the request, successfully debits \$50 from the user's banking ledger at second 1.8, and writes the transaction to PostgreSQL. However, right before the `HTTP 200 OK` receipt payload reaches the customer's phone, the user's cellular connection experiences a momentary network drop.

The smartphone client encounters a *Read Timeout*, assumes the payment attempt failed, and the mobile SDK's automated retry policy re-submits the identical `POST /api/v1/payments` request 3 seconds later:
- Without Idempotency: The backend treats the second request as a brand-new transaction. The customer is **debited twice (\$100)**, producing duplicate order fulfillments, accounting errors, and compliance chargebacks.

**The Idempotency Key Solution:**
The mobile client generates and attaches a unique transaction header to mutating requests: `Idempotency-Key: "uuid-pay-8821"`. When the retry request arrives carrying the identical key, the backend checks Redis/PostgreSQL, detects that the transaction has already committed with status `COMPLETED`, halts duplicate billing, and safely replays the cached original receipt. The customer's account is debited strictly once.

## 2. What Is It and What Is It For?

Idempotency is an algebraic and software property stating that executing an operation multiple times with the exact same parameters **leaves the system in the exact same state and returns the exact same result** as the initial invocation.

In the standard HTTP specification:
- **`GET`, `HEAD`, `OPTIONS`**: Inherently *Safe* and *Idempotent* (read-only operations).
- **`PUT`**: Strictly *Idempotent* (full resource replacement; setting a user's address to "123 Elm St" ten times leaves the record identical).
- **`DELETE`**: Strictly *Idempotent* (deleting user ID 42 ten times leaves the user deleted).
- **`POST`**: By specification **NON-Idempotent** (creating resources; calling `POST /orders` three times creates 3 distinct purchase orders).

To make mutating `POST` endpoints resilient, distributed systems introduce an **Idempotency Key**.

## 3. Why Use It in High-Throughput Systems?

In production distributed computing, we are mandated to implement **Timeouts** and **Retries**. However, combining timeouts with retries creates a catastrophic vulnerability:

Consider an e-commerce checkout flow:
1. A client submits a payment request: "Charge $100".
2. The Payment Service processes the transaction and writes to the ledger at second 1.9.
3. A momentary data center packet drop severs the connection right before the HTTP 200 response reaches the client.
4. The client encounters a **Read Timeout (3 seconds)** and assumes the payment failed.
5. An automated client retry policy immediately re-submits: "Charge $100".

Without Idempotency: The customer's credit card is **charged twice ($200)**, producing duplicate invoices and triggering chargebacks and regulatory compliance audits.

With Idempotency: The Payment Service inspects the client's unique *Idempotency Key*, detects that the transaction has already committed, aborts duplicate charging, and safely replays the cached original receipt.

## 4. How It Works (Data Flow Diagram - Idempotency Key)

```
Client                                                  Payment Backend & Database
  |                                                                 |
  | 1. POST /payments                                               |
  |    Header: Idempotency-Key: "uuid-1234-abcd"                    |
  |---------------------------------------------------------------->|
  |                                                                 |
  |                                                  [ Check Key in DB / Redis ]
  |                                                  Key Absent -> Acquire Lock
  |                                                  Execute Debit Transaction
  |                                                  Persist Result Record to DB
  |                                                  Cache Serialized Response
  |                                                                 |
  |               X (TCP Connection Drops In Transit!)              |
  | < - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - |
  |                                                                 |
  | [ Client Timeout -> Auto Retry Fired! ]                         |
  |                                                                 |
  | 2. POST /payments (RETRY)                                       |
  |    Header: Idempotency-Key: "uuid-1234-abcd"                    |
  |---------------------------------------------------------------->|
  |                                                                 |
  |                                                  [ Check Key in DB / Redis ]
  |                                                  Key ALREADY EXISTS!
  |                                                  Status: COMPLETED
  |                                                  DO NOT Debit Account Again!
  |                                                  Retrieve Cached Response
  |                                                                 |
  | 3. HTTP 200 OK (Original Receipt Payload)                       |
  | <---------------------------------------------------------------|
  v                                                                 v
[ Transactionally Safe: Customer Charged Exactly Once ]
```

## 5. Core Implementation Logic (Idempotency Pattern di Go)

Below is the standard application-level idempotency implementation in Go di layer aplikasi menggunakan pola Atomic Lock dan Payload Hashing di Go:

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

var ctx = context.Background()

// ProcessPaymentWithIdempotency mencegah double-charge menggunakan Redis SetNX
func ProcessPaymentWithIdempotency(rdb *redis.Client, idempotencyKey string, payload []byte) (string, error) {
	// 1. Buat hash payload untuk mendeteksi perubahan parameter request
	hasher := sha256.New()
	hasher.Write(payload)
	payloadHash := hex.EncodeToString(hasher.Sum(nil))

	lockKey := fmt.Sprintf("idempotency:%s", idempotencyKey)

	// 2. SetNX atomik sebagai distributed lock dengan TTL 120 detik
	acquired, err := rdb.SetNX(ctx, lockKey, "IN_PROGRESS:"+payloadHash, 120*time.Second).Result()
	if err != nil {
		return "", err
	}

	if !acquired {
		val, _ := rdb.Get(ctx, lockKey).Result()
		if val == "IN_PROGRESS:"+payloadHash {
			return "Request sedang diproses bersamaan, silakan tunggu", nil
		}
		return "Transaksi duplikat telah berhasil diproses sebelumnya", nil
	}

	// 3. Eksekusi mutasi saldo di core payment gateway...
	paymentID := "PAY-987654"

	// 4. Update status idempotensi menjadi COMPLETED selama 24 jam
	rdb.Set(ctx, lockKey, "COMPLETED:"+paymentID, 24*time.Hour)
	return paymentID, nil
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Financial Correctness**: Completely eliminates *Double-Spending*, *Duplicate Order Ingestion*, and *Inventory Over-allocation*.
- **Unlocks Safe Network Retries**: Allows distributed clients to aggressively retry transient failures without risking data corruption.

### Cons
- **Storage Overhead**: Systems must persist idempotency records, timestamps, and serialized response payloads in a fast database or Redis cluster with retention TTLs (e.g., 24 to 48 hours).
- **Concurrency Locks (*In-Flight Conflicts*)**: If an identical retry request arrives while the original transaction is still actively calculating, the system must coordinate distributed locking (returning HTTP 409 Conflict or blocking until resolution).

## 7. When to Use vs When Not to Use

### When to Use
- All financial, debit, transfer, and billing endpoints.
- Order creation, checkout submission, and inventory decrement APIs.
- Webhook ingestion endpoints (payment providers like Stripe, Adyen, and PayPal often deliver duplicate webhook events).

### When Not to Use
- Read-only retrieval endpoints (`GET /catalog`, `GET /users/me`).
- Append-only high-volume logging or telemetry streams where minor duplication has zero business impact (e.g., website button click tracking).

## 8. Key Metrics & Observability

Key telemetry signals to monitor idempotency health on Grafana or Datadog dashboards:

- **`idempotency_requests_total`**: Rate of incoming requests carrying an idempotency key.
- **`idempotency_cache_hit_total`**: Volume of duplicate requests successfully short-circuited and served from cache.
- **`idempotency_in_flight_conflicts_total`**: Rate of concurrent duplicate submissions triggering HTTP 409 conflicts.
- **`idempotency_payload_mismatch_errors_total`**: Spike indicates client bug or malicious payload tampering with old keys (HTTP 422).
- **`idempotency_store_latency_ms`**: Read and write latency against the Redis or database persistence layer.

## 9. Real-World Case Studies

- **Stripe API Idempotency Specification**: Stripe established the modern web standard for API payment processing using `Idempotency-Key: <UUID>`. When a client's network connection drops during a credit card charge, Stripe SDKs automatically retry with the identical key. Stripe's edge servers ensure the customer is never double-billed, validating that incoming parameters match the stored hash.
- **Shopify Order Ingestion**: During viral flash sales and Black Friday, millions of buyers click the "Place Order" button repeatedly if payment gateways take multiple seconds. Shopify leverages checkout token keys at the database transaction layer to absorb duplicate submissions without issuing multiple fulfillment orders.

## 10. Interesting Facts & Production Gotchas

- **The Database Unique Constraint Is Your Last Line of Defense**: Many junior engineers attempt idempotency solely using in-memory Redis checks: `GET key`, if null then execute. Under extreme concurrency, two identical requests hit two different server nodes at the exact same microsecond, both pass the Redis read check, and duplicate records commit (*distributed race condition*). The ultimate unyielding guard is an explicit **Database Unique Constraint** (e.g., a `UNIQUE` index on `idempotency_key` in PostgreSQL). The ACID storage engine guarantees only one insert succeeds.
- **Payload Hash Mismatch Detection**: What happens if a client mistakenly (or maliciously) re-uses an *Idempotency-Key* (`uuid-123`) but sends a completely different payload (Request 1 paid $10; Request 2 attempts to pay $1,000)? Robust enterprise APIs compute a cryptographic SHA-256 hash of the request payload body and store it with the key. If the key matches but the hash differs, the API immediately halts with **`HTTP 422 Unprocessable Entity: Idempotency key payload mismatch`**.
