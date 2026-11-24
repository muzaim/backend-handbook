# Distributed Transactions

## 1. Real-World System Scenario
In digital banking and fintech architectures, an inter-account fund transfer transaction requires debiting Account A and crediting Account B residing across physically isolated database clusters.
In digital banking and fintech architectures, an inter-account fund transfer transaction requires debiting Account A and crediting Account B residing across physically isolated database clusters.

In a traditional monolithic architecture with a single centralized database, this operation is trivial because it is wrapped inside a local relational ACID transaction:
```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 1000 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 1000 WHERE id = 'B';
COMMIT;
```
If the server crashes prior to commit, the database engine automatically rolls back both balance modifications atomically. Zero funds are ever lost.

However, in modern enterprise architectures leveraging microservices or distributed database sharding:
- Customer Account A resides in the **`User Account DB` (PostgreSQL on Shard 1)**.
- Customer Account B resides in the **`Merchant Ledger DB` (MySQL on Shard 2)**.
- They are physically separated storage clusters running on distinct hardware, linked only over TCP/IP networks.

The Catastrophic Vulnerability:
If Shard 1 successfully deducts \$1,000 from Account A, but an intermittent network partition drops packets or Shard 2 runs out of memory right before Account B can be credited:
- Account A's money is gone, but Account B never receives it.
- A local SQL transaction cannot roll back modifications committed inside a completely separate remote database instance! The money has vanished in transit.

Enforcing atomic, all-or-nothing data consistency across physically detached storage nodes over imperfect networks is the central challenge solved by **Distributed Transactions**.

## 2. What Is It and What Is It For?

A Distributed Transaction is a sequence of discrete operations spanning two or more decoupled persistent storage systems (e.g., two distinct PostgreSQL databases, or a MySQL database and an Apache Kafka event log) that must collectively satisfy global consistency guarantees.

Within a single centralized monolithic database, applications benefit from local **ACID** guarantees:
- **Atomicity**: All writes commit or everything rolls back (*All-or-Nothing* via `BEGIN ... COMMIT / ROLLBACK`).
- **Consistency**: Database schema constraints and foreign keys are never violated.
- **Isolation**: Concurrent transactions cannot observe partial writes.
- **Durability**: Committed data is safely written to write-ahead logs on disk.

However, once an engineering architecture transitions to **Microservices (Database-per-Service)**, standard local database ACID transactions cannot traverse network boundaries.

## 3. Why Must It Be Understood in High-Throughput Systems?

A common mistake made by transitioning engineers is attempting to enforce classical synchronous multi-database coordination—such as **Two-Phase Commit (2PC)**—at scale. This introduces severe availability bottlenecks.

### The CAP Theorem and Physical Network Realities:
In distributed environments, the CAP Theorem dictates that a system can guarantee at most two of the following properties simultaneously:
- **C** (Consistency: Every client reads identical, latest data instantly).
- **A** (Availability: Every non-failing node returns a successful response without error).
- **P** (Partition Tolerance: The system functions despite dropped network packets).

Because cross-datacenter and cloud network partitions are an inevitable physical reality, distributed systems must trade off between **Consistency or Availability**.

Choosing strict distributed ACID (2PC): whenever an internal network cable hiccups or a downstream replica pauses for garbage collection, all participating databases freeze their thread pools with pessimistic locks, rejecting user traffic globally. System throughput plummets from 10,000 req/s to a complete standstill.

Consequently, high-throughput architectures discard strict distributed ACID in favor of the **BASE** paradigm:
- **Basically Available**: The platform remains available to users at all times.
- **Soft State**: System state may drift temporarily without explicit user interaction.
- **Eventual Consistency**: All distributed data stores reach mathematical alignment after asynchronous replication catches up.

## 4. How It Works: Two-Phase Commit (2PC) & Its Flaws

Two-Phase Commit (2PC) is the canonical computer science protocol for coordinating multi-node transactions:

```
[ Phase 1: Prepare (Voting Phase) ]
Coordinator ------> Node A (Order DB):   "Can you commit order?" -> Node A: "VOTED YES (Locks Row!)"
Coordinator ------> Node B (Payment DB): "Can you debit user?"   -> Node B: "VOTED YES (Locks Balance!)"

[ Phase 2: Commit (Execution Phase) ]
Coordinator observes unanimous YES votes:
Coordinator ------> Node A: "COMMIT!" -> Node A commits and unlocks.
Coordinator ------> Node B: "COMMIT!" -> Node B commits and unlocks.
```

```
[ The 2PC Catastrophe: Node B Crashes or Network Partitions ]
Coordinator ------> Node A: "COMMIT!" -> Node A commits.
Coordinator ------> Node B: PACKET LOSS / UNRESPONSIVE!
(Node A committed, but Node B's status is unknown.
 Database locks on Node A & Node B hang indefinitely. System throughput collapses!)
```

## 5. Core Implementation Logic (Two-Phase Commit 2PC di Go)

Implementation of the Two-Phase Commit (2PC) coordinator in Go (2PC) dengan mekanisme prepare, voting quorum, commit/abort, dan penanganan timeout di Go:

```go
package main

import (
	"context"
	"errors"
	"sync"
	"time"
)

type Participant interface {
	Prepare(ctx context.Context, txID string) error
	Commit(ctx context.Context, txID string) error
	Rollback(ctx context.Context, txID string) error
}

type TwoPhaseCoordinator struct {
	participants []Participant
	timeout      time.Duration
}

// Execute menjalankan protokol 2PC (Phase 1: Voting, Phase 2: Decision)
func (c *TwoPhaseCoordinator) Execute(ctx context.Context, txID string) error {
	reqCtx, cancel := context.WithTimeout(ctx, c.timeout)
	defer cancel()

	// Fase 1: Prepare (Voting Quorum)
	var wg sync.WaitGroup
	errCh := make(chan error, len(c.participants))

	for _, p := range c.participants {
		wg.Add(1)
		go func(part Participant) {
			defer wg.Done()
			if err := part.Prepare(reqCtx, txID); err != nil {
				errCh <- err
			}
		}(p)
	}
	wg.Wait()
	close(errCh)

	// Jika ada 1 partisipan yang menolak / gagal voting, batalkan semua (Rollback)
	if len(errCh) > 0 {
		for _, p := range c.participants {
			_ = p.Rollback(context.Background(), txID)
		}
		return errors.New("transaksi 2PC dibatalkan: salah satu partisipan gagal prepare")
	}

	// Fase 2: Commit global
	for _, p := range c.participants {
		_ = p.Commit(context.Background(), txID)
	}
	return nil
}
```

## 6. Pros and Cons (Trade-offs)

### Pros (Distributed ACID / 2PC Approach)
- **Strong Consistency Guarantees**: Eliminates eventual consistency lag. Financial state reflects 100% precision instantaneously.
- **Simpler Developer Mental Model**: Developers can reason about data using familiar relational ACID abstractions.

### Cons
- **Pessimistic Blocking**: Participating database engines must acquire exclusive row locks across network roundtrips, causing latency explosions.
- **Coordinator Single Point of Failure (SPOF)**: If the coordinator process crashes midway through Phase 2, participants remain stranded in an *in-doubt* state, unwilling to release database locks.
- **Anti-Scalability**: Throughput degrades exponentially as the number of participating microservices increases.

## 7. When to Use vs When Not to Use

### When to Use (2PC / XA Transactions)
- Inside the internal consensus engines of distributed SQL databases operating low-latency intra-cluster protocols (Google Spanner, CockroachDB, YugabyteDB).
- Low-volume, heavily regulated inter-bank settlement ledgers where immediate synchronous auditing supersedes availability concerns.

### When Not to Use (Avoid 2PC Across Microservices!)
- Microservice boundaries communicating across cloud regions or public networks.
- Consumer-scale e-commerce, mobility, streaming, and retail platforms handling concurrent traffic spikes. (Use the **Saga Pattern** instead!).

## 8. Key Metrics & Observability

Observability metrics to detect distributed transaction latency and deadlock risks on Grafana or Datadog:

- **`distributed_tx_duration_seconds`**: Total latency from transaction start through multi-node commit completion.
- **`distributed_tx_in_doubt_total`**: Count of orphaned transactions stalled because a participant crashed during Phase 2.
- **`distributed_tx_abort_rate`**: Percentage of transactions rejected during the prepare phase due to timeouts or contention.
- **`database_lock_hold_time_ms`**: Duration database records remain locked pending remote network confirmations.
- **`coordinator_heartbeat_failures_total`**: Frequency of connection interruptions between coordinator and worker nodes.

## 9. Real-World Case Studies

- **Google Spanner TrueTime Distributed Transactions**: Google built Spanner to provide global distributed ACID without prolonged blocking locks by integrating atomic clocks and GPS receivers (*TrueTime API*). Spanner commits cross-region distributed transactions with strict serializability guarantees.
- **CockroachDB Raft Consensus**: Rather than relying on fragile centralized 2PC coordinators, CockroachDB divides relational data into ranges governed by distributed Raft consensus groups, providing transactional consistency with built-in high availability.

## 10. Interesting Facts & Production Gotchas

- **Distributed Deadlocks**: Service A locks Account #1 in Database A and requests a lock on Account #2 in Database B. Concurrently, Service B locks Account #2 in Database B and requests a lock on Account #1 in Database A. Detecting distributed cyclical deadlocks across disparate database engines is extremely difficult without aggressive global deadline timeouts.
- **Real-World Commerce Is Not 2PC**: Modern internet giants like Amazon, Uber, and Shopify **DO NOT** use synchronous two-phase commit for customer checkouts. Real-world commerce operates on asynchronous business compensation: ingest the order immediately, collect payment asynchronously, and if payment fails 30 seconds later, trigger a compensating workflow that cancels the order and notifies the customer.
