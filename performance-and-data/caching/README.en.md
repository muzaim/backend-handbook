# Caching

## 1. Real-World System Scenario
In high-traffic e-commerce platforms, the landing page displays a critical real-time widget: **"Top 50 Best-Selling Products Today"**. During peak lunch hours, this page is loaded by 25,000 active concurrent shoppers every second.
In high-traffic e-commerce platforms, the landing page displays a critical real-time widget: **"Top 50 Best-Selling Products Today"**. During peak lunch hours, this page is loaded by 25,000 active concurrent shoppers every second.

To generate this list of 50 products, the backend must execute an intensive analytical SQL query spanning table joins and aggregation:
`SELECT p.id, p.title, p.price, SUM(oi.qty) FROM products p JOIN order_items oi ON p.id = oi.product_id GROUP BY p.id ORDER BY SUM(oi.qty) DESC LIMIT 50;`

If every single user HTTP request executes this query directly against a relational database (PostgreSQL / MySQL):
- The database storage engine performs physical disk I/O and heavy grouping math 25,000 times per second for data that yields the exact same output!
- In seconds, database CPU hits 100%, connection pools exhaust, query response times explode from 15 milliseconds to 8 seconds, and the entire backend platform experiences a cascading outage.

**The Caching Solution (e.g., Redis / Memcached In-Memory):**
When the first request arrives, the backend calculates the query once and serializes the result into JSON stored in Redis RAM under key `home:top_50_products` with a 5-minute Time-To-Live (*TTL*). All subsequent 25,000 requests per second are served directly out of in-memory RAM in sub-millisecond latency (< 0.5 ms) without touching relational database disk storage. The database only receives 1 query refresh every 5 minutes.

## 2. What Is It and What Is It For?

Caching is the engineering practice of storing copies of frequently accessed data in a fast, temporary storage tier (typically RAM / In-Memory), so subsequent requests avoid querying slower persistent storage (Disk I/O, relational databases, or remote web services).

Core advantages:
- **Sub-millisecond Latencies**: Fetching data from RAM (Redis/Memcached) takes 0.1 to 1 ms, compared to 5 to 50 ms for an indexed SQL disk query.
- **Database Offloading**: Absorbs 80% to 95% of repetitive read volume away from core databases.
- **Computational Cost Reduction**: Expensive aggregations, template renders, and algorithmic calculations are executed once and reused millions of times.

### Common Caching Patterns:
- **Cache-Aside (Lazy Loading)**: The application checks the Cache first. On a *Cache Miss*, it queries the Database, writes the retrieved record into Cache with a TTL, and returns the response. This is the most prevalent pattern.
- **Write-Through**: The application writes synchronously to both the Cache and the Database concurrently on every write operation.
- **Write-Behind (Write-Back)**: The application writes strictly to Cache and acknowledges instantly; an asynchronous background worker batches updates to persistent storage seconds later (delivers extreme write speed, but risks data loss during ungraceful restarts).

Popular tools: Redis, Memcached, Dragonfly, Hazelcast, in-process memory stores (Go sync.Map, Caffeine, Guava).

## 3. Why Use It in High-Throughput Systems?

In backend engineering, the relational database (PostgreSQL, MySQL) is almost universally the **first architectural bottleneck** to collapse during unexpected traffic surges (e.g., Black Friday sales, ticket drops).

A single high-spec Postgres node typically maxes out at 2,000 to 5,000 complex queries per second before connection pool exhaustion and disk IOPS limits trigger cascading latency spikes. In stark contrast, a modest single-node Redis server comfortably handles 100,000+ operations per second because its memory access overhead is orders of magnitude lower.

Without caching, even moderate traffic spikes to product catalog pages will bring down your primary database, taking the entire business offline.

## 4. How It Works (Data Flow Diagram - Cache Aside)

```
                            +-------------+
                            |   Client    |
                            +-------------+
                                   |
                         1. GET /products/42
                                   v
                         +-------------------+
                         |  Backend Service  |
                         +-------------------+
                                   |
                     2. Query Key "product:42"
                                   v
                         +-------------------+
                         |   Redis (Cache)   |
                         +-------------------+
                                   |
                 +-----------------+-----------------+
                 |                                   |
          [ Cache HIT ]                       [ Cache MISS ]
                 |                                   |
                 v                                   v
        3a. Return Data                     3b. Query Database
         (Latency <1ms)                              |
                 |                                   v
                 |                          +-----------------+
                 |                          |  PostgreSQL DB  |
                 |                          +-----------------+
                 |                                   |
                 |                          4b. Populate Cache
                 |                              (Set TTL + Jitter)
                 v                                   v
       +-----------------------------------------------+
       |          Return Response to Client            |
       +-----------------------------------------------+
```

## 5. Core Implementation Logic (Cache-Aside dengan Anti-Stampede Singleflight di Go)

Berikut implementasi pola Cache-Aside yang dilengkapi dengan penguncian mutex terpadu (*Singleflight*) di Go untuk mencegah *Cache Stampede*:

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
	"golang.org/x/sync/singleflight"
)

var (
	sfGroup singleflight.Group
	ctx     = context.Background()
)

type Product struct {
	ID    int64   `json:"id"`
	Name  string  `json:"name"`
	Price float64 `json:"price"`
}

// GetProductDetails menerapkan pola Cache-Aside dengan Singleflight (Anti-Stampede)
func GetProductDetails(rdb *redis.Client, productID int64) (*Product, error) {
	cacheKey := fmt.Sprintf("product:%d", productID)

	// 1. Coba baca dari Cache Redis
	val, err := rdb.Get(ctx, cacheKey).Result()
	if err == nil {
		var p Product
		_ = json.Unmarshal([]byte(val), &p)
		return &p, nil // Cache Hit
	}

	// 2. Cache Miss: Gunakan Singleflight agar dari 10.000 request konkuren,
	//    hanya 1 request yang memanggil database, sisanya menunggu hasilnya
	v, err, _ := sfGroup.Do(cacheKey, func() (interface{}, error) {
		product, dbErr := fetchProductFromDatabase(productID)
		if dbErr != nil {
			return nil, dbErr
		}

		// Simpan hasil ke Redis dengan TTL 10 menit
		data, _ := json.Marshal(product)
		rdb.Set(ctx, cacheKey, data, 10*time.Minute)
		return product, nil
	})

	if err != nil {
		return nil, err
	}
	return v.(*Product), nil
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Throughput Multiplier**: Scales read-heavy services to millions of active users on minimal database hardware.
- **Exceptional User Latency**: UI and API response times drop from tens of milliseconds to imperceptible sub-milliseconds.
- **Database Protective Cushion**: Acts as an unyielding shock absorber during sudden viral events.

### Cons
- **Stale Data Risks**: Data in cache may drift from persistent database state if writes occur without proper eviction.
- **High Memory Costs**: RAM costs substantially more per gigabyte than NVMe SSD persistent storage.
- **Invalidation Complexity**: As the famous computer science proverb states: *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil Karlton.

## 7. When to Use vs When Not to Use

### When to Use
- Workloads with high read-to-write ratios (*Read-Heavy*, e.g., 90% Read / 10% Write: product details, user public profiles, news articles).
- Static or slowly changing application configuration parameters.
- User session tokens and authorization context.

### When Not to Use
- Mission-critical financial ledgers where stale reads can result in double-spending (e.g., bank account balances, pending stock trades).
- Ephemeral data that is never read more than once (wasting expensive RAM on one-off lookups).
- Rapidly mutating real-time counters where cache eviction overhead outweighs read utility.

## 8. Key Metrics & Observability (Monitoring Dashboard)

- **`cache_hit_ratio` (Gauge / Percentage)**: Calculated as `Hits / (Hits + Misses) * 100%`. Healthy production threshold is **> 85%**. Sub-70% ratios signal that the persistent database is at severe risk of saturation.
- **`evicted_keys_total` (Counter)**: Rate at which keys are forcibly ejected by the cache replacement algorithm (LRU/LFU) due to memory constraints. Spikes mandate an immediate RAM vertical scale-up.
- **`used_memory` vs `maxmemory` (Gauge)**: Memory capacity utilization. Set high-priority alerts at 80% saturation.
- **`connected_clients` (Gauge)**: Monitored to detect leaking connection pools across distributed application nodes.

## 9. Real-World Case Studies (Tech Giants)

- **Meta (Facebook)**: Operates the largest distributed **Memcached** fleet on earth, serving billions of operations per second. Meta published seminal research detailing how they resolve *Cache Stampedes* and *Stale Replication Invalidation* using a custom **Lease Mechanism** built directly into the Memcached server daemon.
- **Twitter / X**: Relies on massive distributed Redis clusters to materialize in-memory home timelines for every active user. When high-follower accounts tweet, fan-out services asynchronously inject the tweet reference into millions of cached timeline lists in RAM for instant feed retrieval.

## 10. Interesting Facts & Production Gotchas

The three legendary production monsters of caching:

1. **Cache Avalanche**: Occurs when thousands of cached entries are configured with identical TTL durations (e.g., all set to expire after exactly 3,600 seconds). At second 3,601, every single key expires simultaneously. Hundreds of thousands of concurrent requests face instant *Cache Misses* and hammer the database together, causing total database blackout.  
   *Mitigation*: Introduce randomized **Jitter** to your TTLs (e.g., 60 minutes ± a random variance between 1 to 5 minutes).

2. **Cache Stampede (Thundering Herd)**: Occurs when a single hyper-popular key (e.g., the homepage hero banner) expires. In the exact millisecond between expiration and regeneration, hundreds of concurrent application threads detect a *Miss* and independently execute the heavy database query simultaneously.  
   *Mitigation*: Implement **Singleflight / Mutex Locking** so only one worker is granted permission to query the database and repopulate the cache while other threads wait for the result.

3. **Cache Penetration**: Malicious actors deliberately query non-existent keys (e.g., `/products/-999999`). Because the key never exists in the database, the cache is never populated, and every single attack request penetrates straight to disk I/O.  
   *Mitigation*: Cache empty/null markers with short expiration times, or position an in-memory **Bloom Filter** upstream of the cache.
