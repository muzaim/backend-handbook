# Redis (Remote Dictionary Server)

## 1. Real-World System Scenario

In high-traffic flash sale systems on tier-1 e-commerce platforms, 500 units of inventory are contested by **150,000 concurrent shoppers** in the first few seconds of launch. There are 500 discounted flagship smartphones available, and **150,000 eager shoppers click the checkout button simultaneously** within the first 3 seconds of promo launch.

If you rely on a traditional relational database (such as PostgreSQL or MySQL) to handle inventory checks and stock decrement:
- Every checkout request issues a `SELECT stock FROM products WHERE id = 1` followed by an `UPDATE products SET stock = stock - 1`.
- Hundreds of thousands of concurrent database connections bombard the storage engine, triggering *disk I/O saturation*, severe *row-level lock contention*, and cascading *connection pool exhaustion*. The database server crashes abruptly, and catastrophic *overselling* occurs (stock count turns negative to -50).

**Solution with Redis:**
The remaining inventory counter is stored directly in Redis RAM as an integer key: `stock:iphone15 = 500`. When the tsunami of requests arrives, the backend executes the atomic Redis command **`DECR stock:iphone15`**. Because all operations run in memory on an atomic *single-threaded event loop* without disk locks, Redis effortlessly chews through **100,000+ operations per second** with sub-millisecond latency (0.2 ms). The 501st user instantly receives 0 or a negative return value and is safely turned away in single-digit milliseconds, protecting the primary relational database from ever feeling the shockwave!

## 2. What Is It and What Is It For?

**Redis (Remote Dictionary Server)** is an open-source, ultra-fast in-memory data store used as a database, cache, message broker, and streaming engine.

### Core Characteristics:
1. **Pure In-Memory Storage**: All primary dataset operations occur in RAM, resulting in average read and write response times well under 1 millisecond.
2. **Rich Data Structures**: Beyond plain key-value strings, Redis natively supports:
   - **Strings**: Text, numeric counters, raw binaries (up to 512 MB).
   - **Hashes**: Field-value mappings (ideal for user profiles or shopping cart records).
   - **Lists**: Linked lists maintaining insertion order (great for FIFO/LIFO job queues).
   - **Sets & Sorted Sets (ZSet)**: Collections of unique members scored by floating-point values (perfect for gaming leaderboards, ranking, and sliding window rate limiters).
   - **Bitmaps & HyperLogLog**: Probabilistic data structures offering extraordinary memory efficiency (tracking millions of daily active users using mere kilobytes of RAM).
3. **Configurable Disk Persistence**: While primarily memory-resident, Redis provides optional persistence mechanisms:
   - **RDB (Snapshotting)**: Point-in-time binary disk snapshots written at configurable intervals.
   - **AOF (Append Only File)**: Logs every write command sequentially for near-zero data loss recovery.

## 3. Why Use It in High-Throughput Systems?

### 1. Pure RAM Access Speed (Sub-Millisecond Latency)
Fetching data from RAM takes ~100 nanoseconds, compared to 10-50 microseconds for NVMe SSD reads and milliseconds for rotational disks. The raw throughput of RAM makes Redis the ultimate architectural shock absorber against brutal traffic spikes.

### 2. Single-Threaded Event Loop (Zero Locking Overhead)
Core Redis commands run on a single thread powered by non-blocking I/O multiplexing (*epoll* on Linux). Because execution is single-threaded, **there are no race conditions, deadlocks, or thread context-switching penalties**. Operations like `INCR`, `HSET`, and `LPUSH` are natively atomic without requiring application-level mutexes.

### 3. Distributed High Availability (Replication, Sentinel & Cluster)
Redis supports asynchronous master-replica replication, automated failover through Redis Sentinel, and horizontal data sharding across up to 1,000 nodes using Redis Cluster for multi-terabyte datasets.

## 4. How It Works (Redis Single-Threaded Event Loop Architecture)

```
[ Thousands of Concurrent Clients ]
Client 1     Client 2     Client 3
    |            |            |
    v            v            v
+------------------------------------------+
|  I/O Multiplexer (Linux epoll / kqueue)  |  <-- Aggregates thousands of active TCP sockets
+------------------------------------------+
                     |
                     v (Sequential Event Queue)
+------------------------------------------+
|       Single-Threaded Event Loop         |
|  - Execute Command 1: INCR key (0.1ms)   |  <-- Executed ONE-BY-ONE atomically
|  - Execute Command 2: GET user (0.1ms)   |      without thread locks or mutexes!
|  - Execute Command 3: ZADD rank (0.2ms)  |
+------------------------------------------+
                     |
                     v
+------------------------------------------+
|         In-Memory RAM Key-Value          |
+------------------------------------------+
                     |
        (Async Background Worker Thread)
                     v
+------------------------------------------+
|    Disk Persistence (RDB / AOF Log)      |
+------------------------------------------+
```

## 5. Core Implementation Logic

### 1. Atomic Decrement for Flash Sales in Go

```go
func BuyFlashSaleItem(ctx context.Context, rdb *redis.Client, userID string, itemID string) (bool, error) {
	stockKey := fmt.Sprintf("stock:%s", itemID)

	// DECR dijalankan secara atomik murni di single thread Redis
	remainingStock, err := rdb.Decr(ctx, stockKey).Result()
	if err != nil {
		return false, err
	}

	if remainingStock < 0 {
		// Revert counter dan tolak pesanan
		rdb.Incr(ctx, stockKey)
		return false, nil // Sold Out
	}

	// Masukkan ke antrean pesanan
	rdb.LPush(ctx, fmt.Sprintf("orders:%s", itemID), userID)
	return true, nil
}
```

### 2. Distributed Lock Pattern (Redlock) in Go

```go
func acquireLock(client *redis.Client, lockKey string, token string, ttl time.Duration) bool {
    // Set jika belum ada (NX) dengan masa kedaluwarsa otomatis (EX)
    success, err := client.SetNX(context.Background(), lockKey, token, ttl).Result()
    return err == nil && success
}

func releaseLock(client *redis.Client, lockKey string, token string) {
    // Lua script memastikan unlock hanya menghapus lock jika token pemilik cocok (atomik)
    luaScript := `
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
    `
    client.Eval(context.Background(), luaScript, []string{lockKey}, token)
}
```

## 6. Trade-offs (Pros and Cons)

### Pros
- **Exceptional Throughput**: Easily sustains 100,000+ queries per second (QPS) per single-core CPU instance.
- **Native Atomicity**: Eliminates complex synchronization and distributed locking logic at the application tier for counting, queues, and sets.
- **Versatile Feature Set**: Out-of-the-box support for TTL expiration, Pub/Sub messaging, geospatial queries, and pipeline batching.

### Cons
- **High Memory Cost**: Retaining hundreds of gigabytes in system RAM is significantly more expensive than persistent disk block storage.
- **Durability Constraints**: Because default disk persistence operates via asynchronous background fsync intervals, a sudden hardware crash may lose the most recent second of writes (*performance vs durability trade-off*).
- **Single-Threaded Head-of-Line Blocking**: Any command with an $O(N)$ time complexity (like `KEYS *` or sorting huge collections) blocks the entire event loop, freezing all other incoming client requests!

## 7. When to Use vs When Not to Use

### When to Use Redis
- Ultra-low latency caching of hot read data (user session state, frequent catalog metadata).
- High-frequency atomic counters and rate limiting (Token Bucket, Sliding Window algorithms).
- Real-time Leaderboards using Sorted Sets (`ZADD`, `ZRANGEBYSCORE`).
- Distributed locks coordinating cross-pod task execution to avoid duplicate processing.

### When NOT to Use Redis
- Primary storage of massive documents (>1 MB per record) or relational data requiring strict multi-table ACID transactions.
- Cold archival storage accessed rarely (inefficient use of RAM).
- As a blanket replacement for relational databases without robust replication and persistence architectures.

## 8. Key Metrics & Observability

Essential telemetry for production Redis monitoring on Grafana / Datadog:

- **`instantaneous_ops_per_sec`**: Total read/write operations processed per second across the instance.
- **`used_memory` vs `maxmemory`**: Current RAM allocation; approaching `maxmemory` indicates upcoming key evictions or imminent write rejection.
- **`keyspace_hits` & `keyspace_misses`**: Computes cache hit efficiency (`hits / (hits + misses)`).
- **`connected_clients`**: Number of active client connections (vital for detecting client connection leaks).
- **`latest_fork_usec`**: Duration taken by the OS `fork()` syscall during RDB/AOF background rewrites; slow forks induce latency spikes.

## 9. Real-World Case Studies

### High-Throughput Token Session In-Memory Caching and Atomic Sliding Window Rate Limiting

- **The Engineering Problem**: A financial services portal experienced an authentication surge of 60,000 requests per second during peak market hours. Originally, every OAuth bearer token verification and session lookup executed a synchronous SQL query against PostgreSQL. Database connection pools saturated (5,000 max connections exhausted), disk I/O hit 100%, and authorization p99 latency surged from 15 ms to 3,800 ms. Concurrently, distributed brute-force credential stuffing attacks evaded detection because logging failed attempts to disk was too slow to throttle attackers in real time.
- **The Architectural Solution**: The infrastructure team introduced a distributed Redis Cluster as the low-latency session and security throttling tier. User session payloads and token states were cached with explicit 24-hour TTLs. Rate limiting was upgraded to an atomic *Sliding Window Log* implemented via Redis Sorted Sets (`ZADD`, `ZREMRANGEBYSCORE`, and `ZCARD`) evaluated inside transactional Lua scripts. The cluster enforced a `volatile-lru` memory eviction policy with strict `maxmemory` reservations to avoid Linux OOM terminations.
- **The System Outcome**: Authorization verification p99 latency plummeted from 3,800 ms to 1.2 ms. Over 98.5% of authentication query pressure was offloaded from PostgreSQL, reducing relational database CPU usage from 95% to 18%. Credential stuffing attempts were throttled at the ingress layer in sub-milliseconds without taxing transactional storage.

## 10. Interesting Facts & Production Gotchas

- **The `KEYS *` Production Nightmare**:  
  Running the `KEYS *` command to discover keys matching a pattern is an $O(N)$ full keyspace scan. Because Redis is single-threaded, running this on a production instance with tens of millions of keys **freezes the server for tens of seconds**! All dependent backend services will experience socket timeouts, initiating catastrophic cluster-wide cascading failures!  
  *Production Golden Rule:* Ban `KEYS *` in production configuration. Always use the non-blocking, cursor-based **`SCAN`** command instead.
- **The Memory Eviction Policy Trap**:  
  By default, if Redis reaches its configured `maxmemory` ceiling, it rejects write commands with fatal `OOM command not allowed` exceptions.  
  *Production Fix:* Explicitly configure an eviction policy in `redis.conf`, such as **`allkeys-lru`** (evict least recently used keys across the entire dataset) or **`volatile-lru`** (evict only keys configured with an expiration TTL).

## 11. Key Takeaways

- **Sub-Millisecond In-Memory Throughput**: Redis stores data in RAM and operates on a non-blocking single-threaded event loop, processing hundreds of thousands of operations per second with microsecond latencies.
- **Leverage Native Rich Data Structures**: Utilize Redis beyond key-value strings: Hashes for domain models, Sets for deduplication, Sorted Sets for leaderboards and rate limiting, and Streams for lightweight messaging.
- **Ban O(N) Commands in Production**: Never issue `KEYS *` or `FLUSHALL` on production clusters, as long-running $O(N)$ operations freeze the single thread and cause cascade timeouts; always use cursor-driven `SCAN`.
- **Set Explicit Eviction Policies**: Bound Redis memory with `maxmemory` and select appropriate eviction strategies (`allkeys-lru` or `volatile-lru`) to prevent runtime crashes when memory reaches capacity.
