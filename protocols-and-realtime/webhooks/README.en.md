# Webhooks (HTTP Callbacks)

## 1. Real-World Scenario

In a SaaS platform serving 100,000 active businesses, the application integrates an external **Payment Gateway** (e.g., Stripe or Adyen) to handle automated monthly subscription invoices.

If the application backend tracks payment completion via traditional **Polling**:
- The backend must continuously execute background jobs making polling requests every 10 seconds:  
  `GET https://api.gateway.com/v1/invoices/inv_99812/status` for tens of thousands of outstanding pending invoices.
- This introduces millions of redundant API calls daily, rapidly exhausts external API rate limits, and wastes compute bandwidth handling 99% of responses that still report `"PENDING"`.

**Solution with Webhooks (Reverse API / Event Push):**
The application registers a secure public endpoint: `POST https://api.saas.com/webhooks/payments`.
Instead of the client platform repeatedly polling, the Payment Gateway proactively calls the backend endpoint via HTTP POST immediately when a customer settles their invoice at the bank:

```json
{
  "event": "invoice.payment_succeeded",
  "data": {
    "invoice_id": "inv_99812",
    "amount": 250000,
    "paid_at": 1732440000
  }
}
```
The customer's subscription status activates instantaneously in milliseconds without wasting network bandwidth or server thread capacity.

## 2. What Is It and What Is It For?

A Webhook is an architectural pattern for event-driven inter-system communication where a producer system delivers data payloads to a consumer system via an HTTP POST request immediately upon the occurrence of a business event. It is commonly described as a **Reverse API** or **HTTP Callback**.

Core characteristics:
- **Push vs Pull**: Replaces inefficient, high-latency polling with immediate event push notifications.
- **Standardized Payloads**: Transmitted as structured JSON payloads accompanied by metadata headers (such as event IDs, timestamps, and cryptographic signatures).
- **Asynchronous & Cross-Organizational**: Represents the global standard for communication between independent systems and SaaS platforms (e.g., GitHub, Stripe, Shopify, Slack, Twilio).

## 3. Why Is It Needed at High Throughput?

At high scale, webhooks unlock massive architectural efficiencies:
1. **Eliminates Redundant Polling Overhead**: Reduces up to 95% of wasted network round-trips, liberating socket connections and database thread pools.
2. **Near-Zero Notification Latency**: Downstream systems receive real-time updates within milliseconds of event completion on third-party platforms.
3. **Decoupled Architecture**: External systems do not require visibility into internal databases; they merely post standardized JSON payloads to negotiated endpoints.

## 4. How It Works (Webhook Flow Diagram)

```
+-------------------+                          +-------------------+
|  Payment Gateway  |                          |  Backend System   |
|  (Event Producer) |                          | (Webhook Consumer)|
+-------------------+                          +-------------------+
          |                                              |
    [Event Occurs]                                       |
  (Payment Succeeded)                                    |
          |                                              |
 1. Generate Signature                                   |
    (HMAC-SHA256 Secret)                                 |
          |                                              |
 2. HTTP POST /webhooks/payments ----------------------> |
    Headers: X-Signature, X-Timestamp                    |
    Payload: {"event": "payment.success"}                |
          |                                       3. Verify Signature
          |                                          & Timestamp
          |                                              |
          |                                       4. Immediate 200 OK
          | <------------------------------------    (< 200 ms)
          |                                              |
          |                                       5. Enqueue Payload to
          |                                          Internal Queue
          |                                          (Redis / RabbitMQ)
          |                                              |
          |                                              v
          |                                   [Background Go Worker]
          |                                   (Update Database State)
```

## 5. Core Implementation Logic (Webhook Receiver with HMAC Verification in Go)

In production, webhook receiver endpoints must authenticate incoming requests cryptographically (*HMAC SHA-256*) to verify payload integrity and return an HTTP 200 OK immediately prior to background processing:

```go
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"io"
	"log"
	"net/http"
	"time"
)

const webhookSecret = "whsec_super_secret_signing_key_production"

type WebhookPayload struct {
	EventID   string `json:"event_id"`
	EventType string `json:"event_type"`
	Data      any    `json:"data"`
}

// verifySignature validates that the payload has not been tampered with in transit (MitM)
func verifySignature(payload []byte, signatureHeader string, secret string) bool {
	mac := hmac.New(sha256.New, []byte(secret))
	mac.Write(payload)
	expectedSignature := hex.EncodeToString(mac.Sum(nil))

	// Use hmac.Equal to prevent timing attacks
	return hmac.Equal([]byte(signatureHeader), []byte(expectedSignature))
}

func WebhookHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	// 1. Read full request body
	body, err := io.ReadAll(r.Body)
	if err != nil {
		http.Error(w, "Failed reading request payload", http.StatusBadRequest)
		return
	}
	defer r.Body.Close()

	// 2. Validate cryptographic HMAC signature from request header
	signature := r.Header.Get("X-Hub-Signature-256")
	if !verifySignature(body, signature, webhookSecret) {
		log.Println("[SECURITY WARNING] Invalid webhook signature detected!")
		http.Error(w, "Unauthorized: invalid signature", http.StatusUnauthorized)
		return
	}

	// 3. Parse JSON event payload
	var event WebhookPayload
	if err := json.Unmarshal(body, &event); err != nil {
		http.Error(w, "Malformed JSON payload", http.StatusBadRequest)
		return
	}

	// 4. CRITICAL PATTERN: Acknowledge receipt immediately (< 200 ms)
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`{"status": "accepted"}`))

	// 5. Offload intensive business processing to an asynchronous worker
	go func(evt WebhookPayload) {
		processWebhookAsync(evt)
	}(event)
}

func processWebhookAsync(evt WebhookPayload) {
	log.Printf("[Worker] Processing event %s (ID: %s) asynchronously", evt.EventType, evt.EventID)
	// Execute database operations, account state updates, or email dispatches here
}
```

## 6. Trade-offs (Pros & Cons)

### Pros
- **Maximum Network Efficiency**: Eliminates redundant polling calls that return no state updates.
- **Real-Time Data Propagation**: Updates arrive at the exact moment events finalize upstream.
- **Universal Industry Standard**: Supported natively by major enterprise developer APIs worldwide.

### Cons
- **Delivery Failure Vulnerability**: If the receiver service experiences temporary downtime, events may be lost unless the producer incorporates robust retry mechanisms.
- **Mandatory Idempotency Handling**: Producers adhere to *At-Least-Once Delivery* guarantees, meaning transient network hiccups can cause identical payloads to be delivered multiple times.
- **Public Attack Surface**: Receiver endpoints must be publicly exposed to the internet, demanding strict security against DDoS and replay attacks.

## 7. When to Use vs When to Avoid

### When to Use
- Asynchronous cross-organizational integrations spanning different infrastructure boundaries (Payment Gateways, courier tracking, communication APIs, CI/CD pipelines).
- State notification updates with unpredictable timing patterns (e.g., when a buyer completes payment via an ATM).
- Public event streaming where direct access to internal message brokers cannot be exposed.

### When to Avoid
- High-velocity internal microservice communication within a shared private Kubernetes VPC (internal messaging systems like Kafka, RabbitMQ, or gRPC provide superior performance and isolation).
- Synchronous workflows where end users in web browsers wait on blocking request-response cycles.
- High-frequency telemetry streams (e.g., 50,000 events/second); individual HTTP connection overhead will induce significant TCP handshake contention.

## 8. Key Observability Metrics (Monitoring Dashboard)

- **`webhook_delivery_success_rate` (Gauge / Percentage)**: Proportion of webhook requests successfully returning 2xx HTTP codes. Production target exceeds **99.9%**.
- **`webhook_response_latency_ms` (Histogram)**: Time taken by consumer handlers to acknowledge payloads. Must stay under 500 ms to avoid producer-side connection timeouts.
- **`webhook_retries_total` (Counter)**: Volume of delivery retries triggered by network disruptions or upstream 5xx errors.
- **`webhook_signature_failures_total` (Counter)**: Spikes in signature mismatches signal potential replay attacks, unauthorized probing, or expired signing secrets.

## 9. Real-World Case Studies

### Ingesting Payment Gateway Notification Spikes and Defeating Replay Attacks

- **The Engineering Problem**: An e-commerce platform received hundreds of thousands of asynchronous payment notification webhooks during a promotional campaign. The original handler executed heavy processing synchronously: verifying status, locking order records, decrementing inventory, and invoking third-party invoicing APIs before writing the HTTP response. When invoicing APIs slowed down, handlers exceeded the payment gateway's strict 5-second connection timeout. The gateway interpreted timeouts as delivery failures and dispatched aggressive retry storms, triggering severe database lock contention and duplicate credit issuance across thousands of customer wallets.
- **The Architectural Solution**: The webhook ingress architecture was rebuilt into a non-blocking asynchronous pipeline. Upon receiving a POST request, the handler cryptographically verified HMAC-SHA256 signatures, validated timestamp freshness within a 5-minute threshold to prevent replay attacks, checked event uniqueness via atomic Redis `SET NX EX` locks, and immediately returned `HTTP 200 OK` within 50 ms. Payload processing was offloaded to background message queues (RabbitMQ/SQS) with dead-letter queue (DLQ) isolation.
- **The System Outcome**: Webhook HTTP response latencies plummeted from an average of 4,200 ms to 28 ms, eliminating sender-side timeouts and subsequent retry avalanches. Double-crediting and duplicate fulfillment rates dropped to exactly 0%. The webhook ingestion pipeline effortlessly absorbed bursts of 15,000 incoming webhooks/sec without degrading core relational database operations.

## 10. Production Gotchas

1. **Synchronous Heavy Processing Before HTTP 200 OK**: Executing slow database transactions or third-party API calls directly within the webhook HTTP handler. Sender platforms enforce short timeouts (typically 5 seconds); exceeding this causes senders to mark requests failed and aggressively retransmit identical payloads.  
   *Mitigation*: Verify the signature, stage the payload in a queue, return HTTP 200 OK immediately, and process asynchronously.
2. **Neglecting Idempotency**: Receiving identical `payment.succeeded` events twice due to network retries. Without checking an idempotency store (e.g., a `processed_events` table), accounts risk double-crediting balances.
3. **Ingesting Webhooks Without Signature Verification**: Accepting unauthenticated HTTP POST payloads without checking HMAC signatures or timestamp freshness. Attackers can easily spoof requests to `/webhooks/payments` and fraudulently activate paid services.

## 11. Key Takeaways

- **Acknowledge HTTP 200 OK Instantly**: Never execute slow database queries or remote calls in the main webhook thread; verify the signature, enqueue the payload to a worker queue, and respond with HTTP 200 in under 200 ms.
- **Enforce Cryptographic HMAC Verification**: Protect exposed public ingress endpoints by authenticating request payloads using shared HMAC-SHA256 secret tokens.
- **Thwart Replay Attacks via Timestamps**: Inspect incoming timestamp headers and reject requests with timestamps drifting more than 5 minutes from server clock time.
- **Guarantee Absolute Idempotency**: Webhook providers enforce at-least-once delivery; record and verify unique `event_id` keys in persistent storage to safely discard duplicate retry payloads.
