# Zero-Downtime Database Migration (Expand and Contract Pattern)

## 1. Real-World Scenario

In a production fintech platform managing a `users` table with 40,000,000 (40 million) active records, product requirements mandate a schema alteration: splitting a legacy monolithic `name` column into two discrete attributes: `first_name` and `last_name`.

If an engineer executes this change naively directly on the production database:
```sql
-- DANGEROUS NAIVE PRODUCTION DDL:
ALTER TABLE users 
    ADD COLUMN first_name VARCHAR(100) NOT NULL,
    ADD COLUMN last_name VARCHAR(100) NOT NULL,
    DROP COLUMN name;
```
- PostgreSQL / MySQL must acquire an exclusive table lock (*AccessExclusiveLock*) on the `users` table.
- For the entire 25 minutes required to rewrite the 40-million row table on disk, **all incoming reads (`SELECT`) and writes (`INSERT/UPDATE`) across the entire table are completely blocked**.
- Tens of thousands of active client login and checkout requests timeout, database connection pools exhaust, and customer transactions fail en masse.
- Worse yet: active Kubernetes pods running legacy Go code crash instantly because their queries still expect the suddenly deleted `name` column (*Breaking Change*).

**Solution with the Expand and Contract Pattern:**
The migration is divided into four distinct phases (*multi-phase deployment*), ensuring that legacy application versions and new application releases run concurrently without locking tables and **with zero customer downtime**.

## 2. What Is It and What Is It For?

Zero-Downtime Database Migration is a schema evolution methodology allowing structural table changes (DDL - Data Definition Language) to execute on live, high-throughput production datastores without service interruptions.

The enterprise gold standard is the **Expand and Contract Pattern** (also termed *Parallel Run*):
1. **Phase 1 (Expand)**: Introduce new tables or nullable columns alongside legacy structures without dropping existing attributes.
2. **Phase 2 (Dual-Write in Application Code)**: Update backend code to write simultaneously to both legacy and new schema targets.
3. **Phase 3 (Backfill Historical Records)**: Run background workers to copy legacy data into the new columns in small, throttled batches without saturating database IOPS or CPU.
4. **Phase 4 (Contract)**: Transition application reads and writes exclusively to the new columns. Once verified, drop the legacy columns during an off-peak cleanup cycle.

## 3. Why Is It Needed at High Throughput?

In modern microservices and continuous deployment (CI/CD) environments:
1. **99.99% Availability SLAs**: Modern customer applications cannot afford scheduled maintenance downtime windows simply to execute database schema alterations.
2. **Backward & Forward Compatibility**: During Kubernetes rolling updates, pods running version $N$ (legacy) and version $N+1$ (new) coexist for several minutes. The database schema must satisfy both code versions simultaneously.
3. **Prevents Table Lock Deadlocks**: Deconstructing massive schema migrations into staged increments preserves sub-millisecond query latencies.

## 4. How It Works (Expand and Contract 4-Phase Flow)

```
[PHASE 1: EXPAND (Database DDL)]
Provision new nullable columns:
users: [ id | name | first_name (NULL) | last_name (NULL) ]
(Instantaneous metadata update in PostgreSQL; zero table locks!)
                         |
                         v
[PHASE 2: DUAL WRITE (Go Application Release)]
Application writes to BOTH targets simultaneously:
Incoming Write: "Budi Santoso"
  --> UPDATE users SET name = 'Budi Santoso', 
                       first_name = 'Budi', last_name = 'Santoso';
(Legacy pods safely read 'name'; new pods safely read 'first_name')
                         |
                         v
[PHASE 3: BACKFILL (Background Go Worker)]
Copy 40 million legacy records progressively (5,000 rows/batch):
WHERE first_name IS NULL LIMIT 5000;
(Database CPU remains stable at 20%; zero replication lag storms)
                         |
                         v
[PHASE 4: CONTRACT (Cleanup Phase)]
1. Update Go codebase: Route 100% of reads and writes to first_name & last_name.
2. Drop legacy column: ALTER TABLE users DROP COLUMN name;
(Completed with 100% System Uptime!)
```

## 5. Core Implementation Logic (Dual-Write Repository & Backfill Worker in Go)

### 1. Dual-Write Repository Implementation in Go (Phase 2)

```go
package main

import (
	"context"
	"database/sql"
	"strings"
)

type User struct {
	ID        int64
	FirstName string
	LastName  string
}

// CreateUserDualWrite writes to both legacy and modernized schema targets simultaneously
func CreateUserDualWrite(ctx context.Context, db *sql.DB, user User) error {
	// Synthesize legacy column representation
	fullName := strings.TrimSpace(user.FirstName + " " + user.LastName)

	query := `
		INSERT INTO users (name, first_name, last_name)
		VALUES ($1, $2, $3);
	`
	_, err := db.ExecContext(ctx, query, fullName, user.FirstName, user.LastName)
	return err
}
```

### 2. Historical Data Backfill Worker with Throttling in Go (Phase 3)

```go
package main

import (
	"context"
	"database/sql"
	"log"
	"strings"
	"time"
)

// BackfillHistoricalData iterates through legacy rows in controlled batches
func BackfillHistoricalData(ctx context.Context, db *sql.DB, batchSize int) {
	for {
		// 1. Fetch unmigrated records using a deterministic sequential index
		rows, err := db.QueryContext(ctx, `
			SELECT id, name FROM users 
			WHERE first_name IS NULL 
			ORDER BY id ASC 
			LIMIT $1;
		`, batchSize)
		if err != nil {
			log.Fatalf("Error during backfill query: %v", err)
		}

		var count int
		for rows.Next() {
			var id int64
			var name string
			_ = rows.Scan(&id, &name)

			parts := strings.SplitN(name, " ", 2)
			firstName := parts[0]
			lastName := ""
			if len(parts) > 1 {
				lastName = parts[1]
			}

			// 2. Populate modernized schema attributes
			_, _ = db.ExecContext(ctx, `
				UPDATE users 
				SET first_name = $1, last_name = $2 
				WHERE id = $3;
			`, firstName, lastName, id)

			count++
		}
		rows.Close()

		log.Printf("[Backfill] Processed %d records successfully...", count)

		// Termination condition: processed batch is smaller than requested batch size
		if count < batchSize {
			log.Println("[Backfill] COMPLETE: All historical records migrated.")
			break
		}

		// 3. CRITICAL: Throttle execution to avoid saturating database CPU and replica lag
		time.Sleep(100 * time.Millisecond)
	}
}
```

## 6. Trade-offs (Pros & Cons)

### Pros
- **Zero Customer Downtime**: System availability remains at 100% throughout large-scale schema reorganizations.
- **Trivial Rollbacks**: If regressions appear during Phase 2, deployments can roll back instantly because legacy columns remain intact and updated.
- **Controlled System Impact**: Backfill workers operate through throttled batching without starving production connection pools.

### Cons
- **Extended Release Cycles**: Mandates 2 to 3 separate deployment cycles instead of a single atomic release.
- **Temporary Code Complexity**: Data access layers must temporarily maintain dual-write mappings.
- **Transient Storage Overhead**: Dual-storing attributes temporarily inflates disk storage footprints.

## 7. When to Use vs When to Avoid

### When to Use
- Production tables containing hundreds of thousands to hundreds of millions of records.
- High-availability systems where maintenance windows are strictly prohibited.
- Breaking structural schema modifications (column splits, column merges, datatype alterations).

### When to Avoid
- Early-stage greenfield prototyping where local or staging environments tolerate brief maintenance restarts.
- Minor metadata tables containing under 1,000 records where instant DDL execution causes no perceptible locking.

## 8. Key Observability Metrics (Monitoring Dashboard)

- **`pg_stat_activity` Lock Wait Time (Gauge)**: Monitors statement queuing caused by DDL lock conflicts. Abort operations immediately if lock waits exceed 2 seconds.
- **`pg_stat_replication_lag_bytes` (Gauge)**: Tracks read replica lag during backfilling. Throttle batch intervals if replica lag spikes.
- **`backfill_unmigrated_rows_count` (Gauge)**: Quantifies remaining unmigrated records (`WHERE new_col IS NULL`).

## 9. Real-World Case Studies

### Refactoring a 28-Million Row User Schema with Zero Downtime

- **The Engineering Problem**:
  At a digital banking platform, engineers were tasked with decomposing a legacy `full_name` column into separate `first_name` and `last_name` fields across a 28-million row `users` table. An engineer executed a naive migration script during off-peak hours:
  `ALTER TABLE users DROP COLUMN full_name, ADD COLUMN first_name VARCHAR(100), ADD COLUMN last_name VARCHAR(100);`  
  This operation demanded an `AccessExclusiveLock`. Because active fund transfer transactions were continually acquiring lightweight `RowShareLocks` on the table, the `ALTER TABLE` statement was forced into PostgreSQL's lock queue. In PostgreSQL, a queued `AccessExclusiveLock` blocks every subsequent read and write query arriving behind it. Within 20 seconds, the 500-slot application connection pool was exhausted, API gateways returned `504 Gateway Timeout` errors, and the core banking platform experienced a catastrophic 25-minute outage.

- **The Architectural Solution**:
  1. **Phase 1 (Expand - Non-Blocking DDL)**: Engineers deployed a non-blocking migration adding the new fields as nullable without default values:
     ```sql
     SET lock_timeout = '2s';
     ALTER TABLE users ADD COLUMN first_name VARCHAR(100), ADD COLUMN last_name VARCHAR(100);
     ```
     With `lock_timeout = '2s'`, the DDL acquired its lock in 1.5ms without endangering the transactional queue.
  2. **Phase 2 (Dual-Writing Go Application Deployment)**: Backend code was deployed to write concurrently to both old and new columns whenever users registered or modified account data.
  3. **Phase 3 (Throttled Background Backfill Worker)**: A standalone Go worker script backfilled the 28 million historical records in discrete chunks (1,000 rows per batch accompanied by a mandatory `time.Sleep(100ms)` pause). The worker monitored `pg_stat_replication_lag` to prevent write amplification from degrading read replicas.
  4. **Phase 4 (Contract & Cleanup)**: Once metrics verified that `COUNT(*) WHERE first_name IS NULL` was 0, Go services were updated to read exclusively from the new columns, and the obsolete `full_name` column was safely dropped.

- **The Engineering Outcome**:
  - The entire 28-million row schema evolution was completed with precisely **0 seconds of downtime** (100% SLA availability).
  - Zero banking transactions or authentications failed due to database table locks.
  - Deployment risk was eliminated: Phases 2 and 3 allowed immediate rollback without loss of newly written customer data.

## 10. Production Gotchas

1. **Adding Columns with Non-Constant Defaults on Older PostgreSQL**: On PostgreSQL versions prior to 11, running `ADD COLUMN status INT DEFAULT 1` forced an entire physical table rewrite on disk, taking multi-hour table locks!  
   *Mitigation*: On legacy engines, add columns as NULL, backfill progressively, and only then apply default constraints.
2. **Unthrottled Batching Storms**: Executing a single unmetered `UPDATE users SET ...` across 40 million records. Database memory exhausts, exclusive locks trigger, and read replicas drop out of sync by dozens of gigabytes.  
   *Mitigation*: Always enforce chunked batches (500 to 5,000 rows) paired with explicit `time.Sleep` intervals.
3. **Dropping Columns While Old Pods Live**: Executing Phase 4 (Drop Column) while earlier application pods are still completing Kubernetes rolling update cycles.

## 11. Key Takeaways

1. **Expand and Contract Discipline**: Break breaking schema alterations into distinct phases: additive expansion (Expand), dual-writing, throttled backfills, and final contraction (Contract).
2. **Strict Lock Timeouts**: Always enforce `SET lock_timeout = '2s'` on production DDL migrations to prevent pending `AccessExclusiveLocks` from starving application connection pools.
3. **Throttled Batch Processing**: Never execute monolithic multi-million row `UPDATE` statements; always process migrations in discrete batches with explicit cooldown intervals.
4. **Coordinate Deployment Lifecycles**: Ensure earlier container generations have completely terminated before executing destructive column drop operations.
