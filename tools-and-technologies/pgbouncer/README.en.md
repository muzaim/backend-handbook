# PgBouncer (PostgreSQL Connection Pooler)

## 1. Real-World Scenario

In an e-commerce microservices platform orchestrated on Kubernetes, the Horizontal Pod Autoscaler (HPA) scales out application backend replicas from 15 pods to 120 pods during a sudden flash sale traffic spike.

Each Go application pod configures standard internal connection pooling:
```go
db.SetMaxOpenConns(25)
```

If these 120 pods establish direct network links to the primary PostgreSQL database:
- The total concurrent connection pool attempting to open against PostgreSQL reaches:  
  $120 \text{ pods} \times 25 \text{ connections} = 3,000 \text{ TCP connections}$!
- **PostgreSQL's Architectural Bottleneck**: PostgreSQL operates on a process-based concurrency model rather than threads. Every single incoming client connection forks a dedicated OS process (`postgres: user db [idle]`), consuming between 5 to 10 MB of resident memory.
- Maintaining 3,000 concurrent physical connections consumes 20 to 30 GB of RAM purely on mostly idle connection sockets. Database CPU utilization hits 100% simply handling Linux kernel context switching across thousands of competing processes.
- Eventually, the database server collapses, rejecting all incoming client queries with:  
  `FATAL: sorry, too many clients already`. Payment and order checkout pipelines across the entire site abruptly fail.

**Solution with PgBouncer:**
The platform infrastructure team deploys **PgBouncer** as a lightweight reverse connection pooling proxy fronting PostgreSQL. All 120 application pods (3,000 client sockets) connect directly to PgBouncer. Under **Transaction Pooling** mode, PgBouncer assigns physical PostgreSQL connections exclusively for the active duration of an executing SQL transaction (typically 1 to 5 milliseconds). Once the transaction commits, that physical link is immediately returned to serve requests from adjacent pods.

The result: **3,000 client application connections are comfortably sustained by just 60 physical PostgreSQL server connections!** Database CPU usage plummets from 100% down to 25%, and connection starvation is permanently resolved.

## 2. What Is It and What Is It For?

PgBouncer is a lightweight, high-performance open-source connection pooler engineered specifically for the PostgreSQL database protocol.

Core characteristics:
- **Lightweight Event-Loop Architecture**: Implemented in C using `libevent`. PgBouncer manages tens of thousands of idle client sockets consuming merely a few megabytes of memory.
- **Three Pooling Modes**:
  1. **Session Pooling**: Retains 1 server connection for the entire duration of the client connection lifecycle. Released only upon client disconnect (most compatible, but least efficient).
  2. **Transaction Pooling (The High-Throughput Standard)**: Server connections are bound only for the active lifecycle of a SQL transaction (`BEGIN ... COMMIT`). Highly optimal for stateless web services and microservices.
  3. **Statement Pooling**: Server connections are released immediately upon the completion of a single SQL statement (does not permit multi-statement transactions).

## 3. Why Is It Needed at High Throughput?

In modern architectures dominated by Kubernetes container autoscaling and Serverless functions:
1. **Protects PostgreSQL Against Connection Starvation**: Prevents runaway horizontal replica expansion from overwhelming database processes.
2. **Minimizes TCP & TLS Handshake Latencies**: Applications maintain fast, local socket links to PgBouncer, while long-haul persistent connections to the upstream database remain continuously open.
3. **Maximizes PostgreSQL Core Performance**: PostgreSQL functions with peak efficiency when active concurrent worker counts roughly align with $2 \times \text{CPU Core Count}$. Routing 50 tightly packed connections is far faster than managing 2,000 fragmented idle processes.

## 4. How It Works (Transaction Pooling Flow Diagram)

```
 [100+ Go Backend Pods in Kubernetes]
 (Each allocating 25-50 client pool sockets)
 +--------+  +--------+         +--------+
 | Pod #1 |  | Pod #2 |  . . .  |Pod #100|
 +--------+  +--------+         +--------+
      \          |          /
       \         |         /
        v        v        v
  [3,000 Client Virtual Connections]
                 |
                 v
 +-----------------------------------------------+
 |                   PgBouncer                   |
 |        (Lightweight libevent proxy)           |
 |                                               |
 |  * 2,940 Client Sockets: IDLE (Awaiting Query)|
 |  * 60 Client Sockets: ACTIVE (In Transaction) |
 +-----------------------------------------------+
                 |
                 | Transaction Pooling (Lease Link Only During Active SQL)
                 v
   [60 Physical Server Connections]
                 |
                 v
 +-----------------------------------------------+
 |               PostgreSQL Server               |
 |         (Dedicated Master Instance)           |
 |                                               |
 |  * Exactly 60 Active Linux OS Processes       |
 |  * Zero Context Switching Thrashing           |
 |  * Healthy CPU: 20-30%, Abundant Free RAM     |
 +-----------------------------------------------+
```

## 5. Core Configuration & Go Integration

### Production `pgbouncer.ini` (Transaction Pooling)

```ini
[databases]
; Route client database queries to internal PostgreSQL master host
mydb = host=10.0.1.50 port=5432 dbname=mydb_production auth_user=postgres

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

; PRIMARY POOLING MODE: transaction (return server link upon commit)
pool_mode = transaction

; Concurrency limits
max_client_conn = 10000     ; Accept up to 10,000 incoming client connections
default_pool_size = 50      ; Max 50 physical connections to PostgreSQL per user/DB
min_pool_size = 10          ; Maintain 10 warm persistent connections
reserve_pool_size = 5       ; Emergency overflow buffer
max_db_connections = 100    ; Absolute ceiling of open server sockets
```

### Go Driver Configuration (`pgxpool` / `database/sql`) Safe for PgBouncer

Under *Transaction Pooling*, using named prepared statements in standard Go drivers can trigger `prepared statement does not exist` errors, as subsequent queries may be executed across different underlying server connections.

Configure the driver to utilize the simple protocol or bypass named statement caching:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

func ConnectViaPgBouncer() (*pgxpool.Pool, error) {
	// Connect to PgBouncer port 6432
	connString := "postgres://app_user:secret@localhost:6432/mydb?sslmode=disable"

	config, err := pgxpool.ParseConfig(connString)
	if err != nil {
		return nil, fmt.Errorf("failed parsing config: %w", err)
	}

	// CRITICAL FOR PGBOUNCER TRANSACTION POOLING:
	// Disable statement caching to prevent cross-connection collision
	config.ConnConfig.DefaultQueryExecMode = 1 // pgx.QueryExecModeSimpleProtocol

	// Local pod pool limits
	config.MaxConns = 30
	config.MinConns = 5
	config.MaxConnLifetime = 30 * time.Minute
	config.MaxConnIdleTime = 5 * time.Minute

	pool, err := pgxpool.NewWithConfig(context.Background(), config)
	if err != nil {
		return nil, fmt.Errorf("failed creating pool: %w", err)
	}

	log.Println("[Database] Connected successfully to PostgreSQL via PgBouncer (Port 6432)")
	return pool, nil
}
```

## 6. Trade-offs (Pros & Cons)

### Pros
- **Exceptional Memory Efficiency**: Reclaims dozens of gigabytes on database nodes by suppressing process sprawl.
- **Enables Fleet Elasticity**: Allows thousands of stateless container replicas to scale out dynamically without triggering `too many clients`.
- **Negligible Overhead**: Introduces sub-0.2 millisecond proxy latency with negligible CPU impact.

### Cons
- **Session-Level Feature Restrictions (in Transaction Mode)**: Features bound to persistent server sessions cannot be used:
  - `LISTEN / NOTIFY` (requires continuous dedicated session state).
  - Session-level runtime mutations (`SET TIME ZONE ...` or `SET work_mem ...`).
  - Session-scoped temporary tables.
  - Named prepared statements across transactions.
- **Additional Operational Component**: Introduces an extra architectural dependency requiring high-availability clustering and monitoring.

## 7. When to Use vs When to Avoid

### When to Use
- Kubernetes microservice or Serverless deployments where hundreds of worker instances burst concurrent database connections.
- PostgreSQL deployments experiencing elevated CPU load caused by process context switching across thousands of idle connections.
- Stateless HTTP REST or gRPC APIs performing short transactional units of work.

### When to Avoid
- Applications heavily dependent on PostgreSQL `LISTEN / NOTIFY` for real-time pub/sub (workaround: route pub/sub connections directly to port 5432).
- Batch analytics pipelines where only 2 or 3 long-running persistent database worker connections exist.
- Non-PostgreSQL datastores (PgBouncer is strictly dedicated to PostgreSQL).

## 8. Key Observability Metrics (Monitoring via `SHOW POOLS`)

PgBouncer exposes a built-in virtual administrative console (`psql -p 6432 -U pgbouncer pgbouncer`):

- **`cl_active` vs `cl_waiting` (Gauge)**: `cl_active` indicates client sockets currently executing SQL. If `cl_waiting` exceeds 0, applications are blocked waiting for available database slots.
- **`sv_active` vs `sv_idle` (Gauge)**: `sv_active` tracks active physical server connections executing queries, while `sv_idle` represents warm connections ready for dispatch.
- **`maxwait` (Gauge - Seconds)**: The maximum queuing delay experienced by any client awaiting a server connection. Exceeding 1 second requires expanding `default_pool_size`.

## 9. Real-World Case Studies

- **GitLab**: Powers large-scale production PostgreSQL fleets serving millions of git operations and CI/CD pipelines. GitLab standardizes on PgBouncer nodes ahead of every PostgreSQL master and replica to absorb connection storms originating from thousands of Sidekiq background workers.
- **Supabase & Heroku Postgres**: Managed cloud platforms deploy PgBouncer automatically in front of managed databases to ensure high-frequency ephemeral Serverless functions do not exhaust database process limits.

## 10. Production Gotchas

1. **The `prepared statement "..." already exists` Trap**: Go or Java frameworks preparing named statements across multiple queries under Transaction Pooling. The second query hits a distinct underlying server connection lacking that prepared definition, causing transaction failures.  
   *Mitigation*: Enforce *Simple Protocol* execution or disable named statement caching in driver pool settings.
2. **Blocked Connections from Lingering Transactions**: Applications opening a transaction (`BEGIN`), executing a slow external third-party HTTP call, and only then issuing `COMMIT`. The physical server connection remains held hostage, starving other queuing clients.  
   *Mitigation*: Never execute external network I/O inside database transactional blocks.
3. **Out-of-Sync `userlist.txt` Credentials**: Forgetting to mirror newly created database user credentials into PgBouncer's authentication mapping, leading to unexpected `Auth failed` rejections.

## 11. Quick Summary

PgBouncer is a lightweight, battle-tested connection pooler shielding PostgreSQL from connection exhaustion. By leveraging Transaction Pooling, it bridges thousands of elastic client microservices to a compact, highly optimized pool of physical server processes, preserving peak database throughput during extreme traffic peaks.
