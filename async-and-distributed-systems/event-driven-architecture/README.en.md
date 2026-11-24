# Event-Driven Architecture (EDA)

## 1. Real-World System Scenario

Consider a ride-hailing mobility platform (such as Uber or Lyft). When a passenger confirms a pickup and a nearby driver accepts the trip, an objective, immutable business occurrence transpires: **`RideAssigned`**.

This single domain event triggers autonomous operations across multiple independent backend microservices:
- The `Navigation Service` computes an optimal traffic route and streams turn-by-turn vectors to the driver's mobile device.
- The `Billing & Payment Service` authorizes and holds the passenger's payment card pre-authorization hold.
- The `Safety & Telemetry Service` activates real-time GPS stream monitoring to detect route deviations or accidents.
- The `Notification Service` pushes a mobile alert to the rider with vehicle license plate details and ETA.

Neither the rider nor the driver directly invokes billing, safety, or navigation internal endpoints. Subsystems independently react to the publication of immutable domain event facts.

If this platform were designed using traditional synchronous request-driven REST/gRPC architectures:
- The Ride Booking API would execute a deep, brittle synchronous chain: `Ride Svc -> Nav Svc -> Billing Svc -> Safety Svc`.
- If the `Safety Service` suffers an internal lockup or network latency, the entire booking dispatch pipeline stalls and fails (*deep synchronous coupling*).

**The Event-Driven Architecture Solution:**
Systems decouple through distributed event logs (e.g., Apache Kafka). Microservices publish state changes as events, and consumers process those streams asynchronously, maintaining localized materialized views (*read projections*) without blocking core customer transactional flows.

## 2. What Is It and What Is It For?

Event-Driven Architecture (EDA) is a distributed software design paradigm where system execution, integration, and communication are orchestrated by the **production, detection, and consumption of immutable domain events**.

### Critical Distinction: Command vs. Event
- **Command**: A targeted request asking a specific service to execute an action. It represents intent, is imperative, and may be rejected.  
  *Example:* `DebitUserWallet(amount: $50)` -> Can fail due to insufficient balance.
- **Event**: A historical, immutable statement of fact that has *already occurred*. It is broadcasted indiscriminately and cannot be rejected or retroactively revoked.  
  *Example:* `OrderCreated`, `PaymentSettled`, `UserAccountVerified`.

Key Architectural Roles:
- **Event Producer**: Emits state transitions to an event log as they occur in its local domain.
- **Event Broker / Stream**: A durable, distributed streaming bus that persists and replicates event logs (e.g., Apache Kafka, AWS Kinesis).
- **Event Consumer**: Independent microservices that subscribe to event streams and autonomously execute their own local business logic.

## 3. Why Use It in High-Throughput Systems?

In classical request-driven systems (REST/gRPC), services communicate via deep, synchronous call graphs:
`API Gateway -> Order Svc -> Payment Svc -> Inventory Svc -> Notification Svc`.

If a single downstream service midway through the chain experiences latency, connection pools exhaust and the entire call tree stalls (*tight temporal coupling*).

In an Event-Driven Architecture:
- Communication is strictly **asynchronous and non-blocking**.
- Services maintain localized materialized views (*Read Projections*) updated from event streams, enabling them to fulfill queries locally without making outbound network RPC calls across internal services (*zero inter-service synchronous network hops*).
- Throughput scales linearly across partitions to handle millions of transactions per second without global database locking.

## 4. How It Works (Data Flow Diagram)

```
[ Domain: E-Commerce Order Lifecycle ]

+--------------------+
|   Order Service    | (Producer)
+--------------------+
          |
          | 1. Commit Local PostgreSQL Transaction
          | 2. Emit Event: "OrderPlaced"
          v
+-------------------------------------------------------------+
|                     Event Stream (Kafka)                    |
|                                                             |
|  [ Event: OrderPlaced (ID: #401, Items: [A, B], Total: $80) ]|
+-------------------------------------------------------------+
          |                            |
          | Consume Event              | Consume Event
          v                            v
+--------------------+       +--------------------+
|   Payment Service  |       | Notification Svc   |
+--------------------+       +--------------------+
| - Deduct Balance   |       | - Dispatch SMS     |
| - Emit New Event:  |       |   and Email Alerts |
|   "PaymentSettled" |       |                    |
+--------------------+       +--------------------+
```

## 5. Core Implementation Logic (Transactional Outbox di Go)

Implementation of the **Transactional Outbox Pattern** in Go untuk mencegah *dual-write loss* antara database dan message broker di Go:

```go
package main

import (
	"database/sql"
	"encoding/json"
	"time"

	"github.com/google/uuid"
)

type OrderEvent struct {
	OrderID   string    `json:"order_id"`
	UserID    string    `json:"user_id"`
	Amount    float64   `json:"amount"`
	CreatedAt time.Time `json:"created_at"`
}

// CreateOrderWithOutbox menyimpan entitas dan outbox event dalam SATU transaksi database lokal
func CreateOrderWithOutbox(db *sql.DB, userID string, amount float64) (string, error) {
	tx, err := db.Begin()
	if err != nil {
		return "", err
	}
	defer tx.Rollback()

	orderID := uuid.New().String()

	// 1. Simpan order ke tabel orders
	_, err = tx.Exec("INSERT INTO orders (id, user_id, amount, status) VALUES ($1, $2, $3, $4)",
		orderID, userID, amount, "PENDING")
	if err != nil {
		return "", err
	}

	// 2. Simpan event ke tabel outbox (Transactional Outbox Pattern)
	eventPayload, _ := json.Marshal(OrderEvent{
		OrderID:   orderID,
		UserID:    userID,
		Amount:    amount,
		CreatedAt: time.Now(),
	})

	_, err = tx.Exec("INSERT INTO outbox_events (id, aggregate_type, aggregate_id, payload, processed) VALUES ($1, $2, $3, $4, $5)",
		uuid.New().String(), "Order", orderID, eventPayload, false)
	if err != nil {
		return "", err
	}

	// Commit transaksi: kedua tabel tersimpan secara atomik tanpa risiko dual-write loss
	return orderID, tx.Commit()
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Extreme Decoupling & Scalability**: Publishers and consumers scale, deploy, and execute at disparate velocities with zero synchronous dependency.
- **Superior Failure Isolation**: If the Notification Service goes offline for 3 hours, order creation continues uninterrupted. When notifications come back online, they seamlessly resume processing queued historical events.
- **Complete Audit Lineage**: An event stream forms a permanent, tamper-proof record of every state transition that ever occurred in the enterprise.

### Cons
- **Eventual Consistency Complexities**: Read models drift momentarily behind writes. User interfaces must account for brief asynchronous lag (100ms - 2s).
- **Schema Evolution Fragility**: Modifying event payloads (removing properties or mutating types) can silently break downstream consumers across disparate teams.
- **Distributed Observability Challenges**: Debugging a distributed transaction across asynchronous event hops mandates rigorous end-to-end distributed tracing (*Correlation IDs*).

## 7. When to Use vs When Not to Use

### When to Use
- Complex distributed platforms with multiple autonomous engineering teams and bounded contexts.
- High-velocity stream processing, IoT sensor fleets, and real-time fraud scoring engines.
- Architectures leveraging **CQRS (Command Query Responsibility Segregation)** and **Event Sourcing**.

### When Not to Use
- Standard CRUD applications and simple back-office internal admin portals.
- Small engineering teams where managing Kafka clusters and eventual consistency introduces unjustified overhead (*accidental complexity*).

## 8. Key Metrics & Observability

Crucial telemetry metrics to monitor event-driven platforms on Grafana or Datadog:

- **`outbox_unprocessed_records_count`**: Number of uncommitted outbox entries awaiting dispatch; backlog spikes indicate background worker stalls.
- **`event_end_to_end_latency_ms`**: Total duration from producer event emission to downstream consumer completion.
- **`schema_registry_validation_errors_total`**: Count of rejected events due to schema incompatibility (Avro / Protobuf / JSON Schema).
- **`consumer_dead_letter_events_total`**: Volume of unprocessable events routed to consumer dead-letter topics.
- **`cdc_replication_lag_seconds`**: Streaming delay for Change Data Capture connectors reading database write-ahead logs.

## 9. Real-World Case Studies

- **Netflix Media Processing Pipeline**: Netflix operates a globally distributed event-driven engine for video transcoding, subtitle creation, and thumbnail rendering. Loading a new movie title into the system triggers dozens of asynchronous worker workflows in parallel without tight inter-service coupling.
- **DoorDash Real-Time Driver Dispatch**: DoorDash streams real-time courier GPS coordinates and restaurant preparation events. Driver dispatch models subscribe to these streams to match nearby couriers to orders just as cooking finishes, minimizing customer wait times.

## 10. Interesting Facts & Production Gotchas: The Dual-Write Problem

The most notorious architectural trap when transitioning to event-driven architectures is **The Dual-Write Problem**:

Junior engineers often write naive code like this:
```go
// 1. Write to database
err := db.Save(order)

// 2. Publish event to broker
err := kafka.Publish("OrderPlaced", order)
```

**The Disaster:**
If line 1 succeeds, but the application server crashes or loses network connectivity to Kafka before line 2 runs, **the database has committed the order, but Kafka never emits the event!** The order is orphaned: money is never collected, and the warehouse never ships the product.

Conversely, if line 2 executes first, Kafka emits an event even if the database transaction subsequently aborts and rolls back.

**The Production Solution: Transactional Outbox Pattern:**  
Write both the domain entity AND the outbound event payload into a dedicated `outbox` table within the **SAME ATOMIC LOCAL DATABASE TRANSACTION**. Then, an asynchronous Change Data Capture (CDC) engine (such as Debezium reading Postgres WAL logs) guarantees 100% reliable, at-least-once delivery to the Kafka cluster!
