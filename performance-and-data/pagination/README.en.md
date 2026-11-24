# Database Pagination (Offset vs Cursor-Based)

## 1. Real-World Scenario

In a social network or e-commerce marketplace platform managing a `posts` or `products` table containing 10,000,000 (10 million) active rows:

If the engineering team relies on **Offset-Based Pagination** (`LIMIT ... OFFSET ...`):
```sql
SELECT id, title, created_at FROM posts ORDER BY id DESC LIMIT 20 OFFSET 500000;
```
- When a user views Page 1 (`OFFSET 0`), queries execute instantaneously (1 millisecond).
- However, when users or web scrapers navigate to deep pages (*deep pagination*, e.g., Page 25,000 with `OFFSET 500000`), PostgreSQL is forced to scan all 500,020 rows sequentially from disk, sort them in memory, **discard the first 500,000 rows**, and return only the trailing 20 rows!
- Query latency explodes from 1 ms to **8.5 seconds**, exhausting database CPU and disk cache buffers.
- **The Data Drift Problem (Item Duplication)**: If 5 new posts are published while a user reads Page 1, navigating to Page 2 causes the bottom 5 items from Page 1 to shift onto Page 2, showing identical duplicate content to the user!

**Solution with Cursor-Based Pagination (Keyset / Seek Method):**
Instead of skipping rows via `OFFSET`, the application uses the unique sequential ID of the last viewed record as a spatial boundary (*cursor*):
```sql
SELECT id, title, created_at FROM posts WHERE id < 948210 ORDER BY id DESC LIMIT 20;
```
With an index on `id`, PostgreSQL performs an immediate *Index Seek*, jumping directly to record `948210` and reading the next 20 entries in **0.4 milliseconds**! Whether reading Page 1 or Page 500,000, latency remains fixed in sub-milliseconds, and duplicate records never appear during continuous scrolling.

## 2. What Is It and What Is It For?

Pagination is the technique of breaking large datasets into small, discrete chunks to optimize network payload serialization and provide smooth client-side user experiences.

The two prevailing industry pagination strategies:
1. **Offset-Based Pagination (Page Numbering Method)**:
   - Utilizes `page` and `limit` parameters (e.g., `?page=5&limit=20` mapped to `OFFSET 80 LIMIT 20`).
   - Highly intuitive for desktop web UIs with explicit numerical page controls (`[1] [2] [3] ... [Last]`).
   - Algorithmic time complexity: $O(N)$ where $N$ is the offset magnitude. Deeper pages trigger severe linear degradation.
2. **Cursor-Based Pagination (Keyset / Seek Method)**:
   - Employs an ordered, unique identifier (*opaque cursor*, typically a Base64-encoded timestamp or primary key) marking the pagination boundary (`?cursor=eyJpZCI6OTQ4MjEwfQ==&limit=20`).
   - The universal industry standard for *Infinite Scrolling* (Instagram, Twitter/X, TikTok, Slack, Stripe).
   - Algorithmic time complexity: Constant $O(1)$ or $O(\log N)$ regardless of database depth.

## 3. Why Is It Needed at High Throughput?

In large-scale distributed architectures:
1. **Prevents Latency Explosion on Massive Tables**: Prevents catastrophic linear slowdowns on multi-million row datasets.
2. **Eliminates Data Drift**: Guarantees deterministic reads during continuous scrolling; concurrent inserts at the top never displace adjacent page boundaries.
3. **Drastically Reduces Database I/O Footprint**: The database reads exactly 20 physical rows from disk rather than reading and discarding tens of thousands of rows.

## 4. How It Works (Offset vs Cursor Flow Comparison)

```
[Offset-Based Pagination: O(N) Scan and Drop]
Query: OFFSET 50000 LIMIT 5
Database scans 50,005 rows:
+-------------------------------------------------------------+---------+
|     First 50,000 Rows Scanned, Sorted, and DISCARDED FROM RAM| 5 Rows  |
+-------------------------------------------------------------+---------+
                                                                   |
                                                                   v
                                                        [Returned to Client]
(Wasted compute, memory, and disk I/O on 50,000 discarded records!)

-------------------------------------------------------------------------

[Cursor-Based Pagination: O(log N) Index Direct Seek]
Query: WHERE id < 948210 ORDER BY id DESC LIMIT 5
Database traverses B-Tree Index:
                  [B-Tree Root]
                        |
                        v
               [Direct Seek to 948210]
                        |
                        +---> Read next 5 indexed rows sequentially
                        |
                        v
             [Returned to Client: < 0.5 ms]
```

## 5. Core Implementation Logic (Cursor-Based Pagination in Go)

Below is an idiomatic Go implementation decoding Base64 cursor boundaries and executing direct index seeks via `WHERE id < :cursor LIMIT :limit`:

```go
package main

import (
	"context"
	"database/sql"
	"encoding/base64"
	"fmt"
	"strconv"
)

type Post struct {
	ID    int64  `json:"id"`
	Title string `json:"title"`
}

type PaginatedResponse struct {
	Data       []Post  `json:"data"`
	NextCursor *string `json:"next_cursor"`
	HasMore    bool    `json:"has_more"`
}

// FetchPostsCursor executes constant-time cursor-based pagination
func FetchPostsCursor(ctx context.Context, db *sql.DB, cursorStr string, limit int) (*PaginatedResponse, error) {
	var lastID int64
	if cursorStr != "" {
		// 1. Decode opaque Base64 cursor string
		decoded, err := base64.StdEncoding.DecodeString(cursorStr)
		if err != nil {
			return nil, fmt.Errorf("invalid cursor: %w", err)
		}
		lastID, _ = strconv.ParseInt(string(decoded), 10, 64)
	}

	// 2. Query with LIMIT + 1 to check if subsequent pages exist without extra COUNT queries
	query := "SELECT id, title FROM posts WHERE ($1 = 0 OR id < $1) ORDER BY id DESC LIMIT $2"
	rows, err := db.QueryContext(ctx, query, lastID, limit+1)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var posts []Post
	for rows.Next() {
		var p Post
		if err := rows.Scan(&p.ID, &p.Title); err != nil {
			return nil, err
		}
		posts = append(posts, p)
	}

	hasMore := len(posts) > limit
	if hasMore {
		// Trim the lookahead (limit+1)-th element
		posts = posts[:limit]
	}

	var nextCursor *string
	if hasMore && len(posts) > 0 {
		// 3. Encode trailing record ID into an opaque Base64 cursor for the subsequent request
		lastItem := posts[len(posts)-1]
		encodedCursor := base64.StdEncoding.EncodeToString([]byte(strconv.FormatInt(lastItem.ID, 10)))
		nextCursor = &encodedCursor
	}

	return &PaginatedResponse{
		Data:       posts,
		NextCursor: nextCursor,
		HasMore:    hasMore,
	}, nil
}
```

## 6. Trade-offs (Pros & Cons)

### Offset-Based
- **Pros**: Easy to implement, supports arbitrary page jumping (e.g., jump from Page 1 to Page 15), trivial total page calculation (`COUNT(*)`).
- **Cons**: Severe performance degradation on deep offsets ($O(N)$), vulnerable to missing or duplicated records during active writes (*data drift*), heavy `COUNT(*)` overhead on large tables.

### Cursor-Based
- **Pros**: Constant sub-millisecond retrieval regardless of dataset size ($O(1)$), immune to data drift on concurrent writes, native fit for mobile infinite feeds.
- **Cons**: Cannot jump to arbitrary numerical pages (no concept of "Page 8"), cannot trivially display total page count, requires unique sequential sorting keys.

## 7. When to Use Offset vs Cursor

### Choose Offset Pagination When:
- Admin dashboards requiring explicit page controls (`[1] [2] [3] ... [Last]`).
- Small to moderate dataset sizes (< 50,000 rows) where `OFFSET` introduces negligible overhead.
- Strictly static archival tables where concurrent inserts never occur.

### Choose Cursor Pagination When:
- Mobile applications or social feeds built around infinite scrolling.
- Multi-million row tables where deep pagination is common.
- High-velocity datasets receiving continuous inserts in real time.
- Public developer APIs conforming to standards established by Stripe, Slack, and GitHub.

## 8. Key Observability Metrics (Monitoring Dashboard)

- **`db_query_duration_seconds` by Pagination Strategy (Histogram)**: Evaluates p99 latency degradation as page depth increases.
- **`db_blocks_read_total` (Counter)**: Storage blocks loaded into buffer caches; high offset queries display massive read spikes.
- **`api_invalid_cursor_errors_total` (Counter)**: Monitors client integrations submitting corrupted or tampered cursor tokens.

## 9. Real-World Case Studies

- **Stripe API**: Standardized modern cursor pagination across global financial APIs. All list endpoints (`/v1/charges`, `/v1/invoices`) utilize `starting_after` and `ending_before` cursors passing object IDs, ensuring reliable and high-speed integration.
- **Twitter / X Timeline**: Uses unique 64-bit time-sorted *Snowflake IDs* as timeline cursors. As users scroll feeds, mobile clients issue `GET /timeline?max_id=184920194821` to fetch adjacent tweets instantly with zero duplicates.

## 10. Production Gotchas

1. **Using Non-Unique Columns as Sole Cursors**: Setting `created_at` as the only cursor key. If 10 rows share the exact same timestamp millisecond, a query filtering `WHERE created_at < :last_time` will inadvertently drop records!  
   *Mitigation*: Implement a **Composite Tie-Breaker Cursor** (`WHERE (created_at, id) < (:last_time, :last_id)`).
2. **Coupling `SELECT COUNT(*)` with Cursor Feeds**: Implementing fast cursor pagination while simultaneously executing full table `COUNT(*)` queries to display total counts. The `COUNT(*)` continues to saturate database CPU.  
   *Mitigation*: Omit total counts on infinite scrolling UIs; return a boolean `has_more` flag instead.
3. **Unprotected Predictable Cursors**: Exposing raw database integer IDs in URLs without obfuscation, inviting enumeration attacks.  
   *Mitigation*: Obfuscate cursor tokens using Base64 or signed HMAC strings.

## 11. Quick Summary

Offset pagination suits small, static numerical tables but suffers severe $O(N)$ degradation and data drift on large datasets. Cursor-based pagination provides the high-throughput standard, leveraging direct index seeks to guarantee constant sub-millisecond latencies across multi-million row collections.
