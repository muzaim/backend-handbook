# CDN & Edge Caching (Content Delivery Network)

## 1. Real-World System Scenario

In global e-commerce platforms and social applications where the authoritative origin backend is hosted in a regional cloud data center, international users traverse thousands of network miles. A shopper in Frankfurt, Germany, navigates to your popular product landing page, which loads a 500 KB JavaScript bundle, 100 KB CSS stylesheet, 3 MB of high-resolution product imagery, and a catalog recommendations API.

Without a Content Delivery Network (CDN):
- Network packets must traverse over 11,000 kilometers of transcontinental undersea fiber optic cables.
- Each round-trip time (RTT) between Frankfurt and Jakarta takes between **180 ms and 250 ms**.
- Because TCP connection setup and TLS 1.3 handshakes require multiple round trips, the Time to First Byte (TTFB) spikes to **1.2 seconds**, pushing total page load time up to **3 to 5 seconds**!
- When millions of shoppers worldwide access a promotion simultaneously, the origin server in Jakarta suffers severe bandwidth saturation, CPU throttling, and cascading crashes.

**Solution with CDN & Edge Caching:**
Static assets (images, JS, CSS) and cacheable API responses are replicated and stored across hundreds of distributed Edge Points of Presence (PoPs) worldwide (such as Cloudflare, AWS CloudFront, or Fastly). When the shopper in Frankfurt visits the site, their HTTP request is intercepted by an Edge CDN server located right inside Frankfurt. RTT plummets from 200 ms to **5 ms**, TTFB drops to **15 ms**, and **95% of egress bandwidth traffic** never touches the Jakarta origin server at all!

## 2. What Is It and What Is It For?

A **Content Delivery Network (CDN)** is a geographically distributed network of proxy servers deployed as close as possible to end users (*the edge of the network*). Its core mission is to accelerate content delivery for both static and cacheable dynamic resources while shielding origin application servers from massive global traffic spikes.

### Core CDN Components:
1. **Origin Server**: The authoritative central host running the primary business logic and databases (e.g., Kubernetes clusters on AWS or GCP).
2. **Edge PoP (Point of Presence)**: Local edge data centers located in hundreds of metropolitan hubs housing caching proxy fleets.
3. **Anycast DNS / BGP Routing**: Network routing protocols that announce the same IP address globally and steer client requests to the topologically nearest PoP with minimal network hops.
4. **Edge Compute / Edge Workers**: Serverless execution environments (V8 isolates / WebAssembly) running lightweight request logic directly at the edge before hitting the origin (e.g., JWT authentication, header rewriting, A/B testing splits, dynamic image optimization).

## 3. Why Use It in High-Throughput Systems?

### 1. Massive Origin Offloading
In high-throughput web applications, serving immutable static files and cacheable read queries through origin application servers wastes scarce CPU cycles and database connection pools. A CDN acts as an impenetrable shield: with a 95% *Cache Hit Ratio*, an origin cluster sized for 5,000 RPS can effortlessly support **100,000 RPS equivalent**, because 95,000 requests are resolved at the edge layer.

### 2. Overcoming the Speed of Light Constraint
The propagation speed of photons through fiber optic cable is bounded by the laws of physics (~200,000 km/s in silica glass). Routing clients across the globe to a centralized data center introduces unavoidable latency overhead. Moving data caches into the exact metro areas where clients reside is the only mechanism to achieve single-digit millisecond response times globally.

### 3. Edge DDoS Mitigation
Tier-1 CDN provider edge networks boast aggregate network capacity exceeding 100+ Terabits per second (Tbps). Volumetric Distributed Denial of Service (DDoS) attacks (SYN floods, UDP amplifications, HTTP floods) are absorbed, scrubbed, and filtered at the network perimeter before penetrating the private origin VPC.

## 4. How It Works (CDN & Edge Caching Data Flow)

```
[ Global Client ]                      [ Edge Server (PoP) ]                   [ Origin Server ]
(e.g., Frankfurt)                      (CDN Frankfurt Edge)                   (Jakarta Data Center)
       |                                          |                                      |
       |----- 1. GET /product/banner.webp ------->|                                      |
       |                                          |-- 2. Check Local Cache?              |
       |                                          |   [ CACHE MISS ]                     |
       |                                          |                                      |
       |                                          |----- 3. Fetch from Origin ---------->|
       |                                          |<---- 4. 200 OK + Cache-Control ------|
       |                                          |                                      |
       |                                          |-- 5. Store in Edge RAM/SSD           |
       |<---- 6. Return Data (TTFB: 180ms) -------|                                      |
       |                                          |                                      |
       | === SUBSEQUENT CLIENTS FROM SAME REGION =====================================   |
       |                                          |                                      |
       |----- 7. GET /product/banner.webp ------->|                                      |
       |                                          |-- 8. Check Local Cache?              |
       |                                          |   [ CACHE HIT! ]                     |
       |<---- 9. Return Instantly (TTFB: 12ms!) --| (Zero Origin Round-Trip Needed!)    |
```

## 5. Core Implementation Logic

### Origin Server HTTP `Cache-Control` Directives (Go / Node.js)

Origin backend handlers instruct CDN edge proxies how long content may be cached and how revalidation should occur using standard HTTP headers:

```go
func handleCatalogAPI(w http.ResponseWriter, r *http.Request) {
    data := getPopularCatalog()

    // 1. max-age=60: Browser clients store cache for 60 seconds
    // 2. s-maxage=3600: CDN Edge proxies retain cache for 1 hour (3600 seconds)
    // 3. stale-while-revalidate=300: If expired, edge serves stale cache instantly
    //    while revalidating asynchronously in background with origin
    w.Header().Set("Cache-Control", "public, max-age=60, s-maxage=3600, stale-while-revalidate=300")
    w.Header().Set("Content-Type", "application/json")
    w.Write(data)
}
```

### Edge Worker Script (Cloudflare Worker / V8 Edge Isolate)

Example edge execution intercepting requests before reaching origin:

```javascript
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    const cache = caches.default;

    // Check edge cache match first
    let response = await cache.match(request);
    if (response) {
      // Inject observability tracing header
      let newHeaders = new Headers(response.headers);
      newHeaders.set("X-Edge-Status", "HIT-LOCAL-EDGE");
      return new Response(response.body, { headers: newHeaders });
    }

    // On Cache Miss, fetch from origin server
    response = await fetch(request);

    // Asynchronously commit response copy to Edge Cache
    if (response.status === 200) {
      ctx.waitUntil(cache.put(request, response.clone()));
    }

    return response;
  }
};
```

## 6. Trade-offs (Pros and Cons)

### Pros
- **Dramatically Lower Latency**: Global TTFB drops from hundreds of milliseconds to single-digit or low double-digit milliseconds.
- **Origin Bandwidth Savings**: Massive reductions in outbound egress data transfer charges from cloud providers (AWS Data Transfer Out / GCP Egress).
- **High Availability & Resilience**: In the event of temporary origin restarts or upstream brownouts, edge nodes continue serving stale cached content to users using `stale-if-error`.

### Cons
- **Cache Invalidation Complexity**: *"There are only two hard things in Computer Science: cache invalidation and naming things."* Purging stale prices or emergency security fixes across 300+ PoPs worldwide requires propagation time (from sub-second to several minutes).
- **Multi-PoP Cache Stampede Risk**: When a viral new media file or software release is requested simultaneously across 50 international cities on a cold cache, every edge PoP could concurrently fetch the origin server simultaneously if *Origin Shielding* is not configured.
- **Incremental Service Costs**: While reducing cloud egress bills, enterprise CDN tiers with dynamic route optimization, edge compute runtimes, and real-time log ingestion incur distinct infrastructure line items.

## 7. When to Use vs When Not to Use

### When to Use Edge CDNs
- All static web assets: WebP/AVIF images, fonts, video streaming media segments (HLS/DASH), JS/CSS bundles.
- High-throughput read-heavy APIs with public, semi-static payloads: Public product listings, news feeds, leaderboard rankings, and site config metadata updated periodically.
- Applications serving a geographically dispersed, international user base.

### When NOT to Cache at Edge CDNs
- Endpoints serving authenticated, user-specific private payloads (`GET /api/v1/user/profile`, wallet balances, credit card records)—caching private user data at public edge caches risks catastrophic data leak incidents between users!
- State-mutating requests (POST, PUT, DELETE, PATCH).
- Low-latency real-time market data that mutates sub-second (e.g., active stock exchange order book updates).

## 8. Key Metrics & Observability

Essential indicators for CDN operational dashboards (Cloudflare Analytics, Fastly Insights, Datadog CDN):

- **`edge_cache_hit_ratio_percent`**: Percentage of requests resolved directly at the edge layer without origin forwarding (target: >90% for static media, >60% for public APIs).
- **`origin_bandwidth_offloaded_bytes`**: Total volume of egress network traffic conserved by edge proxies.
- **`edge_time_to_first_byte_p95_ms` (TTFB)**: Time elapsed between client request dispatch and receipt of the first response byte at the edge.
- **`origin_error_rate_5xx`**: Proportion of 502/503/504 gateway failures returned by origin backends to edge PoPs.
- **`cache_purge_propagation_latency_ms`**: Time required for API purge commands to invalidate cached objects across all global edge nodes.

## 9. Real-World Case Studies

### Offloading 95% of Global Flash Sale Ingress and Origin Shielding via Edge Caching

- **The Engineering Problem**: A global e-commerce marketplace ran a year-end sale that triggered a surge of 120,000 HTTP requests per second against product catalog landing pages. Origin API services and relational databases saturated under identical repetitive queries; database CPU hit 100%, and p99 response times spiked from 80 ms to over 12 seconds, rendering product listings unreachable for thousands of concurrent buyers.
- **The Architectural Solution**: The infrastructure team deployed a distributed edge caching policy across edge CDNs (Cloudflare / AWS CloudFront) enforcing `Cache-Control: public, max-age=60, s-maxage=600, stale-while-revalidate=30`. Static media assets were pinned with content-hash URLs (`immutable`). For semi-dynamic catalog data, *Origin Shield (Tiered Caching)* collapsed multi-PoP cache misses, paired with automated webhook-driven API cache purge pipelines when inventory managers mutated product prices.
- **The System Outcome**: The CDN Edge Cache Hit Ratio climbed to 96.4%, offloading over 96% of incoming traffic from origin infrastructure. Time to First Byte (TTFB) for global end-users dropped from 1,200 ms to a median of 32 ms. Backend origin CPU utilization remained under 25% throughout peak shopping hours without provisioning extra server capacity.

## 10. Interesting Facts & Production Gotchas

- **The Private Cache Header Disaster (Account Data Leak Incident)**:  
  A notorious failure mode in high-traffic architectures is improper `Cache-Control` header configuration on personalized endpoints. If an engineer forgets to specify `private` on an authenticated endpoint (e.g., `GET /api/v1/me`) and inadvertently returns `Cache-Control: public, s-maxage=3600`, the Edge CDN will store User A's profile response in its cache! Hundreds of subsequent users opening that screen will suddenly see User A's private account details, name, and email!  
  *Production Golden Rule:* Every authenticated endpoint receiving `Authorization` headers or session cookies must explicitly declare **`Cache-Control: private, no-store, no-cache`**.
- **Origin Shielding (Preventing Multi-PoP Stampedes)**:  
  If your CDN spans 350 PoPs worldwide, and a popular cache key expires simultaneously, all 350 PoP proxies may simultaneously dispatch origin requests for the exact same resource (*multi-PoP cache stampede*).  
  *Production Solution:* Always enable **Origin Shield** (Cloudflare Tiered Cache / CloudFront Origin Shield). This routes edge cache misses through a designated regional super-PoP shield layer, collapsing 350 incoming requests into a single collapsed origin request.

## 11. Key Takeaways

- **Proximity Cuts Latency**: CDNs replicate static and semi-dynamic assets across global edge Points of Presence (PoPs), terminating TLS handshakes physically close to end-users.
- **Leverage Modern Cache Headers**: Employ `s-maxage` for edge proxies and `stale-while-revalidate` to instantly return cached data while refreshing stale entries asynchronously in the background.
- **Prevent Private Data Leaks**: Never emit public cache directives on personalized endpoints; strictly mandate `Cache-Control: private, no-store` on all authenticated routes.
- **Deploy Origin Shields**: Shield origin instances against multi-PoP cache stampedes during simultaneous TTL expirations by funneling edge misses through intermediate Tiered Cache layers.
