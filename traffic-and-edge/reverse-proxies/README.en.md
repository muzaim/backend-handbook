# Reverse Proxy

## 1. Real-World System Scenario
In modern web application architectures, backend runtime services written in Go or Node.js operate on internal ports (like `3000`) in private network subnets serving thousands of concurrent users.
In modern web application architectures, backend runtime services written in Go or Node.js operate on internal ports (like `3000`) in private network subnets serving thousands of concurrent users.

If you expose this application server directly to the public internet:
- Every client performs HTTPS encryption and decryption (*TLS handshakes*) directly against your application runtime, wasting 30% to 40% of precious CPU cycles on cryptographic math rather than executing business logic.
- Static assets (product images, CSS, JavaScript bundles) must be served directly by backend application threads, exhausting connection pools and worker routines.
- The raw IP address of your internal database/application host is exposed to public port scans and direct DDoS attacks.

**The Reverse Proxy Solution (e.g., NGINX / Envoy):**
You place an edge reverse proxy on public ports `80/443`. The reverse proxy terminates incoming SSL/TLS certificates at the network perimeter, caches and serves static assets instantly from disk, applies Gzip/Brotli compression, and forwards clean plain HTTP requests internally to `http://127.0.0.1:3000`. The true internal backend topology remains completely isolated in private VPC subnets.

## 2. What Is It and What Is It For?

A reverse proxy is an intermediary server positioned in front of one or more backend web servers. Its core job is to intercept incoming client requests from the public internet and safely forward them to the appropriate internal server inside a private network.

Key responsibilities:
- **Masking backend infrastructure**: The actual IP addresses of internal servers are never revealed to the public internet.
- **SSL/TLS Termination**: Handling cryptography and certificate handshakes at the edge, allowing internal upstream servers to communicate over lightweight plain HTTP.
- **Payload Compression**: Compressing outbound HTTP responses (Gzip/Brotli) to conserve egress bandwidth.
- **Edge Shielding**: Filtering malformed requests, port scans, and rudimentary attacks before application logic is reached.

Popular software: NGINX, HAProxy, Envoy, Traefik, Caddy.

## 3. Why Use It in High-Throughput Systems?

Under tens of thousands of requests per second, application runtime CPUs (Node.js, Go, Python, Java) should not be wasted on heavy cryptographic TLS handshakes and gzip compression.

Exposing application instances directly to the public web causes major operational hazards:
- Slow client connections (such as Slowloris attacks) saturate application worker pools and consume precious heap memory.
- Rolling restarts or scaling operations disrupt active clients because clients cache the raw server IPs.

With a dedicated reverse proxy written in C or C++, network I/O is handled by an asynchronous event loop, freeing your application servers to focus exclusively on business execution.

## 4. How It Works (Data Flow Diagram)

```
+-------------+
|   Client    | (Public Internet)
+-------------+
       |
       | 1. HTTPS Request (Port 443)
       v
+-------------------------------+
|         Reverse Proxy         |
|  (NGINX / Envoy / HAProxy)    |
|                               |
| - SSL/TLS Decryption          |
| - Header Injection (X-Forward)|
| - Connection Pooling          |
+-------------------------------+
       |
       | 2. Plain HTTP Internal (Port 8080/8081)
       +-----------------------+
       |                       |
       v                       v
+---------------+       +---------------+
| Backend App 1 |       | Backend App 2 |
| (Private IP)  |       | (Private IP)  |
+---------------+       +---------------+
```

## 5. Core Implementation Logic (Declarative & Forwarding Pattern)

Below is an authentic declarative upstream forwarding configuration (NGINX syntax):

```nginx
upstream backend_cluster {
    server 10.0.1.10:8080 max_fails=3 fail_timeout=10s;
    server 10.0.1.11:8080 max_fails=3 fail_timeout=10s;
    keepalive 64; # Connection pooling to internal upstreams
}

server {
    listen 443 ssl http2;
    server_name api.backendhandbook.com;

    ssl_certificate     /etc/ssl/certs/bundle.crt;
    ssl_certificate_key /etc/ssl/private/api.key;

    location / {
        proxy_pass http://backend_cluster;
        proxy_http_version 1.1;
        proxy_set_header Connection ""; # Required for HTTP keep-alive pooling
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 2s;
        proxy_read_timeout 5s;
    }
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Security hardening**: Internal hosts remain entirely isolated in private subnets or VPCs.
- **Resource offloading**: Shifts encryption, buffer management, and compression away from business services.
- **Deployment agility**: Upstream instances can be rotated, upgraded, or restarted without modifying public DNS records.

### Cons
- **Single Point of Failure (SPOF)**: Without redundant clustering (e.g., VRRP/Keepalived or cloud edge load balancers), proxy downtime kills the application.
- **Network hop latency**: Adds an extra network hop (typically sub-millisecond, but measurable at high scale).
- **Configuration overhead**: Requires ongoing maintenance of upstream pools, timeouts, buffer allocations, and SSL certificates.

## 7. When to Use vs When Not to Use

### When to Use
- Every production web service serving traffic over the public internet.
- Architectures requiring centralized HTTPS certificate renewal and cipher suite management.
- Routing multiple internal services under a unified domain path (e.g., `/api` routed to Go, `/static` served directly).

### When Not to Use
- Bare local developer machines where setup complexity offers zero production fidelity benefit.
- Purely internal point-to-point inter-service calls already managed by an internal sidecar service mesh.

## 8. Key Metrics & Observability (Monitoring Dashboard)

When monitoring reverse proxies in Datadog or Prometheus/Grafana:
- **`active_connections` (Gauge)**: Total concurrent open TCP client connections.
- **`upstream_response_time_seconds` (p95/p99 Histogram)**: Time spent by upstream backends generating responses. Spikes point to application-tier bottlenecks.
- **`upstream_connect_time_seconds` (Histogram)**: Time taken to establish a TCP socket with upstream nodes. Over 1 second indicates socket pool exhaustion or saturated internal routing.
- **`http_status_codes_total{status=~"5xx"}` (Counter)**: Rapid spikes in HTTP 502 (Bad Gateway) or 504 (Gateway Timeout) immediately flag crashing backends.

## 9. Real-World Case Studies

### Mitigating Slowloris & Offloading TLS in High-Volume Banking Ingress

- **The Engineering Problem**:
  A digital banking core platform originally exposed dozens of Go application pods directly to the public Internet over port 443. Mobile clients operating on erratic or throttled mobile networks transmitted JSON request payloads very slowly (taking 2 to 4 seconds to complete a single HTTP body upload). Go worker goroutines and open TCP sockets remained starved and blocked waiting on client buffers, mimicking Slowloris connection starvation. Concurrently, repeated cryptographic TLS handshakes (RSA/ECDSA asymmetric key negotiations) consumed upwards of 40% of compute CPU cycles across backend instances.

- **The Architectural Solution**:
  1. **Edge Reverse Proxy Layer**: Platform engineers deployed an autoscaling NGINX/Envoy cluster into public subnets (*DMZ*) acting as the sole ingress point ahead of strictly private backend subnets.
  2. **TLS Offloading & Keep-Alive Pooling**: HTTPS was terminated entirely at the proxy edge using hardware crypto acceleration. Traffic between the proxy and upstream Go pods traversed private VPC links utilizing persistent HTTP keep-alive connection pools.
  3. **Aggressive Request Buffering**: The reverse proxy was configured to asynchronously buffer incoming client request bodies into RAM. Only when the final byte was safely received did the proxy dispatch the complete request to upstream Go handlers over sub-millisecond local LAN connections.

- **The Engineering Outcome**:
  - Application server CPU utilization dropped from 82% to 38% immediately after eliminating TLS computational overhead.
  - Active backend socket hold duration shrank from an average of 3,200ms to just 18ms per transaction.
  - Upstream backend IPs remained strictly unaddressable from public networks, shutting down direct scanning and port-probing vulnerabilities.

## 10. Interesting Facts & Production Gotchas

- **The X-Forwarded-For Pitfall**: Without proper proxy header configuration, backend logs will report the reverse proxy's IP for every single user request. Always configure `X-Forwarded-For` and `X-Real-IP`, and ensure your application trusts only designated reverse proxies to prevent spoofing.
- **Upstream Keep-Alive Neglect**: A common beginner mistake is omitting HTTP keep-alive pools between the reverse proxy and upstream instances. The proxy then tears down and re-establishes TCP connections on every single request, quickly resulting in ephemeral port exhaustion (*TIME_WAIT storms*) during heavy traffic surges.

## 11. Key Takeaways

1. **Private Network Shield**: Shields internal service topology and IP addressing from direct public Internet exposure.
2. **Computational Offloading**: Handles resource-intensive SSL/TLS negotiations, response compression, and buffering, freeing backends to focus on core domain logic.
3. **Protection from Slow Clients**: Request buffering insulates application worker pools from network latency stalls caused by slow client links.
4. **Mandatory Persistent Keep-Alives**: Maintain persistent TCP connection pools between the reverse proxy and upstream backends to eliminate ephemeral port exhaustion (`TIME_WAIT`).
