# Distributed Locking (Concurrency & Race Condition Guard)

## 1. Real-World Scenario

In a global stadium ticketing platform, exactly one last seat remains available for a marquee concert: seat **VIP-A1**. At the stroke of 12:00:00.000, 500 eager fans click **"Checkout & Pay Now"** during the exact same millisecond.

The backend application is deployed across 20 horizontally scaled Go container pods behind a Cloud Load Balancer:

If the backend relies exclusively on local memory locks (*In-Memory Mutex* via `sync.Mutex` in Go):
- Local mutexes only synchronize goroutines within **one individual operating system process**.
- Each of the 20 application pods picks one local winner from its incoming queue. Consequently, 20 independent customer transactions successfully process payment for the exact same physical seat!
- A catastrophic business failure occurs: **Overselling / Double-Booking**. 19 disappointed ticket buyers are turned away at the stadium entrance.

If the backend attempts to rely on relational row-level locking (`SELECT ... FOR UPDATE`):
- 500 simultaneous database transactions fiercely compete to acquire locks on the identical row in PostgreSQL, causing extreme lock contention, connection starvation, and cascading deadlock aborts.

**Solution with Distributed Locking (Redis / etcd):**
Prior to modifying inventory state, the Go application pod must acquire a globally centralized distributed lock stored in Redis via an atomic primitive:
```
SET lock:seat:VIP-A1 "uuid-request-9921" NX PX 5000
```
- The `NX` (Not Exists) flag and `PX 5000` (5-second auto-expiration) guarantee that **exactly 1 out of the 500 requests successfully acquires the lock** within sub-milliseconds.
- The remaining 499 requests fail fast immediately (< 2 ms) with: *"Seat is currently being reserved by another buyer"*.
- The victorious pod completes the database transaction and safely releases the lock via an atomic Lua script.

## 2. What Is It and What Is It For?

Distributed Locking is a concurrency synchronization pattern guaranteeing *Mutual Exclusion* (ensuring only one execution thread across an entire distributed cluster can enter a *critical section* at any given instant) spanning multiple independent servers, virtual machines, or container nodes.

Core architectural properties:
1. **Mutual Exclusion**: At any given time, only one client across the entire fleet can hold the lock for a specified resource ID.
2. **Deadlock Freedom**: Every lock must enforce a deterministic expiration lease (*TTL / Lease Time*). If a node crashes abruptly from a power outage while holding a lock, the key automatically expires.
3. **Fault Tolerance**: The distributed lock coordinator (e.g., Redis Cluster or etcd) must remain available and consistent despite network partitions.

## 3. Why Is It Needed at High Throughput?

In large-scale distributed architectures:
1. **Prevents Double-Spend & Overselling Disasters**: Guarantees digital balances, inventory quotas, or seat reservations are never claimed multiple times concurrently.
2. **Shields Databases from Concurrency Storms**: Filters thousands of competing requests at an in-memory caching tier in microseconds, allowing only the single winning transaction to access relational storage.
3. **Prevents Duplicate Distributed Cron Executions**: Prevents recurring background jobs scheduled across 5 distinct application servers from dispatching identical billing emails five times to a customer.

## 4. How It Works (Redis Distributed Lock Flow Diagram)

```
 [500 Concurrent Pod Requests]
 (All competing for seat VIP-A1)
   |       |             |
   v       v             v
 +---------------------------------------------------------------+
 |                       Redis Cluster                           |
 |                                                               |
 | Command: SET lock:seat:VIP-A1 <unique_uuid> NX PX 5000        |
 |                                                               |
 | * 1 Winner    : Returns OK   --> [Lock ACQUIRED Successfully] |
 | * 499 Requests: Returns NIL  --> [Lock FAILED / Fast Abort]   |
 +---------------------------------------------------------------+
        |                                     |
        v                                     v
 [Winning Pod Instance]              [499 Losing Pods]
 1. Execute DB Payment & Order       Immediate HTTP 409 Conflict:
 2. Decrement Inventory State        "Seat currently being reserved"
 3. Release Lock via Lua Script:
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    end
```

## 5. Core Implementation Logic (Safe Distributed Locking in Go)

Below is a production-grade distributed lock implementation in Go using `github.com/redis/go-redis/v9`, featuring unique random UUID ownership tokens and safe lock release via an atomic Lua script:

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"

	"github.com/google/uuid"
	"github.com/redis/go-redis/v9"
)

// Lua script to atomically release the lock ONLY IF the stored value matches the caller's UUID.
// Prevents Client A from accidentally deleting Client B's lock if Client A's execution exceeded the TTL.
const unlockLuaScript = `
	if redis.call("get", KEYS[1]) == ARGV[1] then
		return redis.call("del", KEYS[1])
	else
		return 0
	end
`

type DistributedLocker struct {
	rdb *redis.Client
}

func NewLocker(rdb *redis.Client) *DistributedLocker {
	return &DistributedLocker{rdb: rdb}
}

// AcquireLock attempts to secure an exclusive distributed lock with an automated expiration TTL
func (l *DistributedLocker) AcquireLock(ctx context.Context, resourceKey string, ttl time.Duration) (string, error) {
	lockKey := fmt.Sprintf("lock:%s", resourceKey)
	lockValue := uuid.New().String() // Unique ownership identifier

	// SET lockKey lockValue NX PX ttl (Single Atomic Instruction)
	success, err := l.rdb.SetNX(ctx, lockKey, lockValue, ttl).Result()
	if err != nil {
		return "", fmt.Errorf("failed contacting redis coordinator: %w", err)
	}

	if !success {
		return "", errors.New("failed acquiring lock: resource currently held by another worker")
	}

	return lockValue, nil
}

// ReleaseLock safely relinquishes the lock using an atomic Lua script
func (l *DistributedLocker) ReleaseLock(ctx context.Context, resourceKey string, lockValue string) error {
	lockKey := fmt.Sprintf("lock:%s", resourceKey)

	// Evaluate Lua script atomically inside Redis
	result, err := l.rdb.Eval(ctx, unlockLuaScript, []string{lockKey}, lockValue).Result()
	if err != nil {
		return fmt.Errorf("failed executing release script: %w", err)
	}

	if result.(int64) == 0 {
		return errors.New("lock already expired or owned by an adjacent worker")
	}

	return nil
}
```

## 6. Trade-offs (Pros & Cons)

### Pros
- **Enforces Strict Data Integrity**: Eliminates duplicate processing and dirty race conditions across hundreds of compute nodes.
- **Microsecond Latency Performance**: Redis `SET NX` primitives resolve in sub-millisecond execution times.
- **Deadlock-Resilient Self-Healing**: Automated TTL leases ensure stale or abandoned locks recover automatically after worker terminations.

### Cons
- **Infrastructure Dependency**: Outages on the distributed lock coordinator (Redis) stall critical transactional business paths.
- **The TTL Expiration Dilemma (*Clock Drift & GC Pauses*)**: If a Go worker encounters a prolonged Garbage Collection pause exceeding the TTL, the lock may expire prematurely while the critical section is still executing.
- **Multi-Master Consensus Overhead (Redlock)**: Achieving fault tolerance across multiple Redis masters requires implementing the complex Redlock consensus protocol.

## 7. When to Use vs When to Avoid

### When to Use
- High-concurrency operations competing over scarce physical or digital inventory (concert ticketing, flash sales, limited voucher claims).
- Financial ledger transactions where double-spending must be prevented.
- Distributed recurring scheduled tasks (*cron jobs*) where only a single node must execute the workload.

### When to Avoid
- Read-only queries (`GET` requests) that mutate no internal state.
- Simple single-instance monoliths where Go's native in-memory `sync.Mutex` is vastly simpler and faster.
- Workloads where concurrency collisions can be handled natively by relational database constraints (`UNIQUE KEY`).

## 8. Key Observability Metrics (Monitoring Dashboard)

- **`distributed_lock_acquisition_failures_total` (Counter)**: Rate of rejected lock requests; expected to spike during promotional flash sale bursts.
- **`distributed_lock_hold_duration_seconds` (Histogram)**: Quantifies how long workers retain locks. Sustained approaches to TTL limits require tuning.
- **`distributed_lock_expired_before_release_total` (Counter)**: Critical alarm metric detecting instances where locks expired prior to explicit worker release.

## 9. Real-World Case Studies

- **Ticketmaster**: Relies on high-throughput distributed locking to reserve seats for 5-minute checkout windows during stadium stadium events, preventing concurrent buyers from checking out identical seat allocations.
- **Shopee / Tokopedia**: During midnight flash sales where millions of users contest 10 discounted units, in-memory distributed locks in Redis absorb 99.9% of concurrent contention, ensuring only verified winners are directed to database checkout queues.

## 10. Production Gotchas

1. **Releasing Another Client's Lock Without Lua**: Client A acquires a lock with a 3-second TTL. Due to database slowdowns, Client A takes 5 seconds. At second 3, the lock expires and is acquired by Client B. At second 5, Client A finishes and issues a blind `DEL lock:key`, inadvertently deleting Client B's valid lock!  
   *Mitigation*: Always attach random UUID tokens and verify ownership via atomic Lua scripts.
2. **Omitting Expiration Leases (Permanent Deadlocks)**: Creating locks without `PX/EX` TTL parameters. If the worker encounters an unhandled panic before unlocking, the resource locks permanently.
3. **Neglecting Fencing Tokens**: For systems demanding strict linearizability, pass incrementing monotonically increasing *Fencing Tokens* to ensure downstream datastores reject delayed writes from stale lock holders.

## 11. Quick Summary

Distributed Locking coordinates concurrency across multi-node architectures to prevent data corruption. By pairing atomic in-memory primitives (`SET NX PX`), random UUID ownership tokens, and Lua script releases, distributed locking shields critical business transactions from the catastrophic effects of race conditions and overselling.
