# Saga Pattern

## 1. Real-World System Scenario

Consider an e-commerce checkout flow distributed across 3 autonomous microservices, each with its own isolated persistent database (*Database-per-Service*):
1. **`Order Service`** creates an order record in state `PENDING` in a PostgreSQL database.
2. **`Payment Service`** debits \$75 from the user's digital wallet in a MySQL database.
3. **`Inventory Service`** decrements warehouse stock counts in a dedicated inventory database.

Step 1 succeeds (order initialized). Step 2 succeeds (\$75 successfully debited). However, at Step 3, the `Inventory Service` fails and returns an error: **The requested item was just bought out by another shopper (*Out of Stock*)!**

The Architectural Problem:
Because each microservice maintains its own private database, the `Payment Service` has already committed its local SQL transaction in MySQL. Standard relational SQL engines cannot issue a remote rollback across networks to another physical server. Without a recovery mechanism, the customer has lost \$75 while receiving zero goods!

**The Saga Pattern Solution:**
A Saga decomposes the distributed transaction into a sequence of local transactions paired with corresponding **Compensating Transactions**:
- The failure in `Inventory Service` automatically activates a reverse compensating workflow.
- The system invokes compensation on the `Payment Service`: `RefundPayment(order_id)` to credit \$75 back to the user's wallet.
- The system updates `Order Service` with status `CANCELLED: OUT_OF_STOCK`.
Every participating database returns to an eventual consistent state without ever acquiring slow, distributed locks across the network.

## 2. What Is It and What Is It For?

The Saga Pattern is an architectural pattern for managing distributed business transactions across multiple microservices without relying on slow, blocking distributed locking protocols like Two-Phase Commit (2PC).

Inside a Saga:
- A high-level distributed business transaction is decomposed into a **sequence of localized transactions (*Local Transactions*)** executed inside individual microservices.
- Each service executes its own work in its own isolated database using standard, fast local SQL commits, subsequently triggering the next step via events or message queues.
- If any intermediate step fails (e.g., credit limit exceeded or warehouse inventory unavailable), the Saga initiates a sequence of **Compensating Transactions** that undo the committed changes in reverse chronological order.

## 3. Two Implementation Approaches: Choreography vs Orchestration

```
[ 1. Saga Choreography (Event-Driven / Decentralized) ]

Order Svc ----(Event: OrderCreated)----> Payment Svc
                                              |
                                     (Event: PaymentSuccess)
                                              |
                                              v
Inventory Svc <-------------------------------+
  (Deduct Stock)
```
- *Characteristics*: Decentralized. Participating services listen to domain events and autonomously execute their next step without a centralized orchestrator.
- *Best for*: Simple linear transaction flows involving 2 to 4 microservices. Beyond that, tracing circular event dependencies becomes difficult (*event spaghetti*).

```
[ 2. Saga Orchestration (Centralized State Machine) ]

                    +-------------------------+
                    |    Order Orchestrator   |
                    |     (State Machine)     |
                    +-------------------------+
                      |          ^          |
           1. Command |          | 3. Book  |
              Payment | 2. Paid  |    Stock | 4. Stock Out!
                      v          |          v
                +-------------+  |    +---------------+
                | Payment Svc | -/    | Inventory Svc |
                +-------------+       +---------------+
                      ^
                      | 5. "COMPENSATE: Refund Payment!"
                      +---------------------------------
```
- *Characteristics*: A dedicated orchestrator service manages the transaction lifecycle: issuing commands to services, handling timeouts, recording state transitions, and triggering compensation workflows when errors occur.
- *Best for*: Complex enterprise business flows featuring numerous branches, approvals, and error states (5+ services).

## 4. How It Works (Execution and Compensation Diagram)

```
Forward Steps:
[ OrderCreated ] ---> [ PaymentCharged ] ---> [ InventoryReserved ] (FAILED!)
                                                      |
                                                      v
Rollback (Compensating Steps):                        |
[ OrderCancelled ] <--- [ PaymentRefunded ] <---------+
```

## 5. Core Implementation Logic (Saga Orchestrator di Go)

Implementation of the Saga Orchestrator pattern in Go dengan eksekusi forward dan kompensasi otomatis saat salah satu langkah gagal di Go:

```go
package main

import (
	"context"
	"fmt"
)

type SagaStep struct {
	Name       string
	Execute    func(ctx context.Context) error
	Compensate func(ctx context.Context) error
}

// ExecuteSaga menjalankan orkestrasi langkah lokal dan kompensasi otomatis jika terjadi kegagalan
func ExecuteSaga(ctx context.Context, steps []SagaStep) error {
	var executedSteps []SagaStep

	for _, step := range steps {
		err := step.Execute(ctx)
		if err != nil {
			fmt.Printf("Langkah %s gagal: %v. Memulai proses kompensasi balik...
", step.Name, err)
			// Jalankan kompensasi dengan urutan mundur (LIFO)
			for i := len(executedSteps) - 1; i >= 0; i-- {
				_ = executedSteps[i].Compensate(ctx)
			}
			return fmt.Errorf("saga gagal pada langkah %s: %w", step.Name, err)
		}
		executedSteps = append(executedSteps, step)
	}

	return nil
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Hyperscale Transactional Throughput**: Scales horizontally without global database lock contention.
- **Resilient Failure Recovery**: Systematically unwinds partial business transactions through automated compensating logic.
- **Service Decoupling (in Choreography)**: Services interact purely through domain events without hard-coded network dependencies.

### Cons
- **Lack of Read Isolation**: Because local transactions commit immediately before the global Saga completes, concurrent users can observe intermediate, uncommitted states (*dirty reads* at the business layer).
- **Compensating Logic Complexity**: Engineering teams must write and maintain compensating logic for every forward action (e.g., if you code `ChargeWallet`, you must code and test `RefundWallet`).
- **Strict Idempotency Prerequisite**: All compensating actions must be strictly idempotent, as network retries may deliver compensation triggers multiple times.

## 7. When to Use vs When Not to Use

### When to Use
- Multi-step distributed transactions spanning multiple microservice bounded contexts (e-commerce checkout, travel booking pipelines, financial loan approvals).
- High-throughput systems where Two-Phase Commit (2PC) locks would cause intolerable connection pool exhaustion.

### When Not to Use
- Monolithic architectures (leverage standard local relational ACID transactions `BEGIN ... COMMIT`, which are simpler and atomic).
- Systems demanding absolute, uncompromised transactional isolation where concurrent actors must never observe intermediate states (e.g., core double-entry general ledgers in banking).

## 8. Key Metrics & Observability

Observability metrics to track Saga flow health and compensation rates on Grafana or Datadog:

- **`saga_execution_duration_seconds`**: Total latency from initial transaction step through complete saga terminal state.
- **`saga_compensation_rate`**: Percentage of sagas that initiate backward compensation; spikes alert on third-party outage or stock depletion.
- **`saga_failed_step_total`**: Counter of step-specific aborts (e.g., identifying whether payment or inventory triggers failures).
- **`saga_stuck_zombies_total`**: Number of stalled sagas where compensation failed repeatedly, flagging urgent manual on-call triage.
- **`saga_active_instances_count`**: Current count of active in-flight saga workflows.

## 9. Real-World Case Studies

- **Uber Trip Lifecycle via Cadence / Temporal**: Uber orchestrates millions of daily rides using Saga-based workflow engines (Cadence/Temporal). Matching drivers, credit reservations, and journey tracking are modeled as distributed state machines that trigger compensation if either rider or driver cancels.
- **Amazon Multi-Service Checkout**: Amazon executes checkout pipelines via series of localized commits: applying promotional credits, charging card tokens, and notifying logistics centers. If credit authorization fails, compensation restores promotional balance without globally locking customer tables.

## 10. Interesting Facts & Production Gotchas

- **Semantic Rollback Is Not a Database Undo**: In a SQL engine, a rollback restores byte blocks to their previous state. In a Saga, compensation is an explicit forward-moving business action. For instance, you cannot un-send a welcome email that already arrived in a customer's inbox; the compensation is dispatching a follow-up email: *"Apologies, your registration was canceled due to capacity limits"*.
- **The Pivot Transaction**: In every well-architected Saga, there is a pivotal step known as the **Pivot Transaction**. Once the pivot transaction commits successfully, the Saga is guaranteed to never roll back; it will aggressively retry forward until all subsequent steps succeed. Correctly identifying your pivot step (typically payment capture) eliminates the need to author complex compensations for downstream operations.
