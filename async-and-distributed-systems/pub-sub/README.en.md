# Publish / Subscribe (Pub/Sub)

## 1. Real-World System Scenario

Consider an e-commerce microservices platform when a customer successfully completes an order payment (`OrderPaid`).

Four discrete backend domains require immediate awareness of this state change:
1. The `Inventory Service` must deduct physical stock counts in the warehouse database.
2. The `Notification Service` must dispatch an email receipt and a WhatsApp push notification.
3. The `Logistics Service` must issue a fulfillment manifest to third-party courier dispatchers.
4. The `Analytics / BI Service` must record promotional conversion metrics into the data lake.

If this architecture relies on synchronous point-to-point HTTP service calls:
- The `Order Service` must make 4 sequential network requests across the internal cloud (*tight temporal coupling*).
- If the `Notification Service` encounters a 4-second latency spike, the user's checkout session freezes for 4 seconds.
- If data engineering introduces a 5th service (e.g., *Fraud Audit Service*), the `Order Service` team must modify and redeploy their production application codebase.

**The Pub/Sub Solution (e.g., Apache Kafka / Google Cloud Pub/Sub):**
The `Order Service` acts purely as a *Publisher*. Upon payment confirmation, it emits a single event payload to the broker: `Topic: order-paid` in under 2 milliseconds. The broker automatically clones and dispatches (*fan-out*) the event across all registered *Subscribers* in parallel, asynchronously, and with complete failure isolation. New subscriber services can be onboarded anytime without changing a single line of code in the publisher.

## 2. What Is It and What Is It For?

Pub/Sub (Publish/Subscribe) is an asynchronous messaging pattern where message senders (**Publishers**) do not target specific receiving applications. Instead, publishers categorize emitted messages into designated channels called **Topics**.

Receiving services (**Subscribers**) register their interest in specific topics. Whenever a new message lands on that topic, the messaging broker **replicates and dispatches the payload to every registered subscriber (*Fan-Out Architecture*)**.

### Critical Comparison: Message Queue vs. Pub/Sub
- **Message Queue (1-to-1 Competing Consumers)**: A message enters a queue and is pulled by **EXACTLY ONE** available worker. Once acknowledged, it vanishes from the system.
- **Pub/Sub (1-to-Many Fan-Out)**: A message lands on a topic and is cloned across **ALL** registered subscribers. If five distinct services subscribe, all five receive their own independent copy.

Popular technologies: Apache Kafka, Google Cloud Pub/Sub, AWS SNS, NATS, Redis Pub/Sub, RabbitMQ (via Fanout Exchanges).

## 3. Why Use It in High-Throughput Systems?

Consider what must happen when a user finalizes a payment on an e-commerce platform (`OrderPaid` event):
1. The Inventory Service must deduct physical warehouse stock.
2. The Notification Service must dispatch an email receipt and mobile push alert.
3. The Logistics Service must generate an automated carrier shipping manifest.
4. The Analytics/BI Data Engine must record marketing conversion attribution.

**Without Pub/Sub (Synchronous HTTP Coupling):**
The Order Service must execute 4 sequential external HTTP calls over the network. If the Notification Service suffers a 4-second latency spike, checkout latency explodes by 4 seconds. If marketing decides to deploy a 5th service (e.g., Fraud Auditing), the Order Service codebase must be edited, retested, and redeployed.

**With Pub/Sub:**
The Order Service emits a single lightweight event to the broker: `Topic: order-paid` in under 1 millisecond and immediately acknowledges the user. All downstream microservices consume that broadcast in parallel, decoupled and completely isolated from each other!

## 4. How It Works (Data Flow Diagram - Fan-Out)

```
                            +-----------------+
                            |  Order Service  | (Publisher)
                            +-----------------+
                                     |
                1. Publish Event:    |
                   Topic: "order-paid"
                   Payload: { order_id: 88, amount: 250000 }
                                     v
                 +---------------------------------------+
                 |            Pub/Sub Broker             |
                 |         [ Topic: order-paid ]         |
                 +---------------------------------------+
                     /               |               \
        Clone to Sub 1/    Clone to Sub 2 |     \ Clone to Sub 3
                     /               |               \
                    v                v                v
            +---------------+ +---------------+ +---------------+
            | Inventory Svc | | Notif Svc     | | Analytics Svc |
            | (Deduct Stock)| | (Send Email)  | | (Track Metric)|
            +---------------+ +---------------+ +---------------+
```

## 5. Core Implementation Logic (Pub/Sub di Go)

Implementation of publisher event and subscriber event streaming in Go dan Subscriber loop berbasis topik di Go:

```go
package main

import (
	"context"
	"fmt"

	"github.com/redis/go-redis/v9"
)

var ctx = context.Background()

// PublishOrderEvent mempublikasikan event satu-ke-banyak (fan-out)
func PublishOrderEvent(rdb *redis.Client, topic string, orderJSON string) error {
	return rdb.Publish(ctx, topic, orderJSON).Err()
}

// SubscribeOrderEvents mengonsumsi aliran event secara independen
func SubscribeOrderEvents(rdb *redis.Client, topic string, consumerName string) {
	pubsub := rdb.Subscribe(ctx, topic)
	ch := pubsub.Channel()

	for msg := range ch {
		fmt.Printf("[%s] Menerima event baru: %s
", consumerName, msg.Payload)
		// Proses event sesuai kebutuhan masing-masing subscriber (Notification / Analytics)
	}
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Radical Loose Coupling**: Publishers possess zero knowledge of downstream subscriber topologies, network addresses, or deployment lifecycles.
- **Zero-Friction Extensibility**: Adding new business capabilities (e.g., an audit logging service) requires zero code modifications to existing publisher services—simply subscribe to the existing topic stream.
- **Hyperscale Event Throughput**: Clustered event brokers like Apache Kafka handle millions of partitioned messages per second with sustained throughput.

### Cons
- **Loss of Centralized Process Visibility**: Decentralized event flows make understanding global business state transitions difficult without distributed tracing diagrams.
- **Egress Network Multiplication**: Replicating 1 inbound message to 15 internal subscribers multiplies internal network I/O fifteen-fold on broker network cards.
- **Message Volatility on Ephemeral Brokers**: Simple in-memory pub/sub brokers (like raw Redis Pub/Sub) operate on a fire-and-forget basis; if a subscriber is offline for 2 seconds during deployment, messages are lost permanently.

## 7. When to Use vs When Not to Use

### When to Use
- Broadcasting state change events (*Domain Events*) across distinct bounded contexts in microservices.
- Telemetry ingestion, real-time security SIEM audit logging, and data lake pipelines.
- Multi-user interactive push streams (collaborative canvas edits, real-time sports score delivery, WebSockets fan-outs).

### When Not to Use
- Tight Request-Response workflows requiring immediate return values (e.g., login authentication requiring immediate token generation).
- Single-consumer queue processing where tasks must be executed strictly once across a worker pool (use a Point-to-Point Message Queue instead).

## 8. Key Metrics & Observability

Telemetry metrics to monitor Pub/Sub cluster performance and consumer health on Grafana or Datadog:

- **`kafka_consumergroup_lag` / `pubsub_subscription_backlog`**: Unread event count per partition for each subscribed consumer group.
- **`topic_publish_rate_messages_per_sec`**: Ingestion velocity per topic across producers.
- **`consumergroup_rebalance_rate`**: Frequency of consumer partition rebalancing events; spikes indicate pod flapping or consumer timeouts.
- **`bytes_in_per_sec` & `bytes_out_per_sec`**: Network card I/O throughput to identify broker saturation thresholds.
- **`undelivered_messages_dropped_total`**: Rate of dropped events caused by slow consumer timeouts or buffer overflows.

## 9. Real-World Case Studies

### Decoupling Trip-Completion Fan-Out Events in Ride-Hailing Platforms

- **The Engineering Problem**:
  On an on-demand mobility platform operating millions of daily bookings, the primary trip orchestration service (`trip-service`) executed synchronous HTTP calls to 6 downstream microservices whenever a driver completed a ride (`TripCompleted`): wallet balance deduction (`billing-service`), driver payout incentives (`driver-incentive`), fraud anomaly scoring (`fraud-service`), loyalty tier points (`loyalty-service`), passenger rating push alerts (`push-notification`), and data lake stream ingestors. Severe outages occurred when `loyalty-service` suffered a database slowdown, pushing response latencies to 5,000ms. The driver's smartphone app froze for 5 seconds waiting on checkout completion, preventing them from accepting new rides, and thread pools across `trip-service` collapsed. Additionally, whenever business teams wished to introduce a new downstream listener (e.g., carbon offset auditing), core dispatch developers were forced to modify and redeploy critical trip execution code (*tight coupling*).

- **The Architectural Solution**:
  1. **Event Fan-Out via Kafka Pub/Sub**: Synchronous HTTP dependencies were replaced with an event broker. Upon trip completion, `trip-service` publishes a single lightweight `TripCompletedEvent` onto an Apache Kafka topic `trips.lifecycle` in under 1.5ms and immediately frees the driver to accept adjacent bookings.
  2. **Isolated Consumer Groups**: All 6 downstream services were segregated into autonomous *Consumer Groups* pulling from the topic partitions independently. Consumer read velocities were completely decoupled: lag on `loyalty-service` had zero effect on the ultra-low-latency processing of `billing-service`.
  3. **Zero-Downtime Extensibility**: When product engineers released the carbon emission tracker service, they deployed a new consumer group (`carbon-audit-group`) subscribing to `trips.lifecycle` without requiring code modifications or restarts within `trip-service`.

- **The Engineering Outcome**:
  - Driver ride completion latency dropped by 96%, plummeting from 5,200ms to 180ms.
  - Outages or maintenance deployments on loyalty or analytics pipelines produced a **0% impact** on core passenger payment collection or driver availability.
  - Engineering velocity soared as distinct domain squads deployed new event consumers autonomously without coordination bottlenecks.

## 10. Interesting Facts & Production Gotchas

- **The Slow Subscriber Catastrophe**: What happens if one subscriber experiences severe thread lockups while the publisher continues emitting 50,000 events per second? In RAM-buffered brokers, memory allocated to that lagging subscriber balloons until the broker exhausts its heap and suffers an **Out-Of-Memory (OOM) crash**, destroying healthy subscribers along with it!  
  *Prevention:* Utilize distributed log systems like **Apache Kafka**, where messages reside as persistent files on disk and each subscriber manages its own read offset cursor independently without consuming broker RAM.
- **The Duplicate Metric Trap**: When a subscriber crashes and resumes consumption from an uncommitted offset, it will inevitably re-process historical events. Without **Idempotency** baked into downstream handlers, data analytics systems will silently record duplicate financial metrics.

## 11. Key Takeaways

1. **One-to-Many Decoupling**: Publishers emit events without knowledge of downstream consumer identities, eliminating point-to-point network couplings between microservices.
2. **Isolated Processing Rates**: Each consumer group tracks its own partition offset cursor, ensuring that lagging downstream subscribers do not stall fast consumers.
3. **Frictionless Extensibility**: New microservices, analytics pipelines, or notification channels can subscribe to existing event streams without altering or redeploying publishing systems.
4. **Enforce Consumer Idempotency**: Pub/Sub systems guarantee at-least-once delivery; consumers must deduplicate events using message IDs to prevent double counting in analytics and billing.
