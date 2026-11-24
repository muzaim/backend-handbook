# Load Balancing

## 1. Real-World System Scenario
In online concert ticketing platforms and e-commerce flash sales subject to sudden surges of 40,000 incoming HTTP requests per second the moment booking opens:
In online concert ticketing platforms and e-commerce flash sales subject to sudden surges of 40,000 incoming HTTP requests per second the moment booking opens:

If all incoming client traffic targets a single standalone server instance (e.g., an 8-core CPU, 16GB RAM virtual machine):
- The server caps out around 2,000 requests per second before TCP socket tables exhaust, CPU hits 100%, and application memory crashes (*OOM crash*).
- The remaining 38,000 customers face connection timeouts, `HTTP 502 Bad Gateway` errors, and aborted checkout sessions.
- The single server represents a fatal **Single Point of Failure (SPOF)**. If hardware fails at midnight, the entire business experiences complete downtime.

**The Load Balancer Solution (e.g., HAProxy / AWS ALB / NGINX):**
You replicate your application into 20 identical containerized instances (`Backend 1` through `Backend 20`) within a private network. A Load Balancer is positioned at the public entrypoint with a single virtual IP. When 40,000 requests hit per second, the Load Balancer divides the traffic evenly (e.g., ~2,000 req/s each) using algorithms like Round-Robin or Least Connections. If `Backend 7` crashes, the Load Balancer detects the failure via automated *health checks* within seconds and redirects traffic to the 19 surviving instances with *zero downtime*.

## 2. What Is It and What Is It For?

Load Balancing is the technique of distributing incoming network traffic or computational workload evenly across a cluster of backend servers (commonly referred to as a *server farm* or *upstream pool*).

Its primary objectives:
- **Preventing server exhaustion**: Ensuring no single server instance is overwhelmed while neighboring machines sit idle.
- **High Availability & Fault Isolation**: Performing automated, periodic *Health Checks*. If an instance crashes or becomes unresponsive, the balancer immediately strips it from the routing pool.
- **Horizontal Scaling (*Scale-Out*)**: Allowing operators to seamlessly add new compute instances to meet surging demand without service interruption.

### Common Load Balancing Algorithms:
- **Round Robin**: Distributing requests sequentially in circular order (1, 2, 3, 1, 2, 3...).
- **Least Connections**: Forwarding incoming connections to the instance with the fewest active connections (ideal for workloads with variable request durations).
- **IP Hash / Consistent Hashing**: Routing requests deterministically based on client IP hashes (useful for sticky affinity when required).
- **Weighted Round Robin**: Allocating proportional traffic weights based on underlying machine capacity (e.g., 16-core instances receive double the share of 8-core instances).

Popular tools: NGINX, HAProxy, AWS ALB/NLB, Cloudflare, F5 BIG-IP.

## 3. Why Use It in High-Throughput Systems?

No matter how powerful a single server is (even a 128-core machine with 512GB of RAM), physical hardware limits inevitably trigger CPU bottlenecks, thermal throttling, and TCP socket exhaustion under 100,000+ requests per second.

Furthermore, relying on a single server guarantees catastrophic downtime during routine software updates, operating system patching, or unforeseen hardware failures.

Load balancing unlocks resilient horizontal scale: 20 commodity instances working in concert deliver far greater throughput at lower cost. If two instances fail unexpectedly overnight, the system absorbs the loss with zero user-facing impact.

## 4. How It Works (Data Flow Diagram)

```
                       +-------------------+
                       |  Incoming Traffic |
                       | (10,000 req/sec)  |
                       +-------------------+
                                 |
                                 v
                     +-----------------------+
                     |     Load Balancer     |
                     |                       |
                     | Periodic Health Check |
                     | Algorithm: Round Robin|
                     +-----------------------+
                       /         |         \
         3,333 req/s  /          |          \  3,333 req/s
                     /  3,333    |           \
                    v   req/s    v            v
             +----------+  +----------+  +----------+
             | Server A |  | Server B |  | Server C |
             | (Healthy)|  | (Healthy)|  | (Healthy)|
             +----------+  +----------+  +----------+
```

If Server B suddenly fails health check verification:

```
                     +-----------------------+
                     |     Load Balancer     |
                     |  [Server B UNHEALTHY] |
                     +-----------------------+
                       /                   \
         5,000 req/s  /                     \  5,000 req/s
                     v                       v
             +----------+                 +----------+
             | Server A |  [X] Server B   | Server C |
             | (Healthy)|   (Isolated)    | (Healthy)|
             +----------+                 +----------+
```

## 5. Core Implementation Logic (Algoritma Round Robin di Go)

Below is the production implementation of the algorithm in Go Round Robin dengan seleksi node sehat (*health-aware*) di Go:

```go
package main

import (
	"errors"
	"sync"
)

type ServerNode struct {
	Host    string
	Healthy bool
}

type RoundRobinLoadBalancer struct {
	mu      sync.Mutex
	servers []*ServerNode
	index   int
}

func NewRoundRobinLoadBalancer(servers []*ServerNode) *RoundRobinLoadBalancer {
	return &RoundRobinLoadBalancer{
		servers: servers,
	}
}

// GetNextServer memilih server berikutnya yang berada dalam kondisi sehat (Healthy)
func (lb *RoundRobinLoadBalancer) GetNextServer() (*ServerNode, error) {
	lb.mu.Lock()
	defer lb.mu.Unlock()

	totalServers := len(lb.servers)
	for i := 0; i < totalServers; i++ {
		selected := lb.servers[lb.index]
		lb.index = (lb.index + 1) % totalServers

		if selected.Healthy {
			return selected, nil
		}
	}

	return nil, errors.New("seluruh upstream server sedang down / unhealthy")
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Zero-Downtime Deployments**: Facilitates rolling restarts and progressive updates without severing active user traffic.
- **Fault Tolerance**: Automatically isolates degraded upstream instances in real-time.
- **Cost-Effective Scalability**: Scaling horizontally with elastic compute nodes is significantly cheaper than procuring enterprise mainframe hardware.

### Cons
- **Session State Challenges (*The Sticky Session Dilemma*)**: Backends must be strictly stateless. If login sessions reside in local memory, consecutive client requests routed to different nodes trigger unexpected logouts. Mitigation: store sessions in a distributed Redis store or utilize stateless JWTs.
- **Layer 4 vs Layer 7 Trade-offs**: Choosing between L4 balancing (packet-level TCP routing, blazing fast but payload-blind) and L7 balancing (HTTP header/cookie aware, intelligent routing at higher CPU overhead).

## 7. When to Use vs When Not to Use

### When to Use
- Any production workload demanding high availability (*99.9%+ uptime SLA*).
- When incoming traffic volume surpasses the computational capacity of a single physical instance.
- Implementing modern release methodologies such as *Canary Releases* or *Blue-Green Deployments*.

### When Not to Use
- Internal hobby tools or early prototype MVPs where occasional maintenance windows have zero financial consequence.
- Legacy monolithic systems architected around local in-memory files and non-distributable state (refactoring is required first).

## 8. Key Metrics & Observability (Monitoring Dashboard)

- **`healthy_host_count` vs `unhealthy_host_count` (Gauge)**: Real-time ratio of operational instances versus targets failing active health checks.
- **`backend_connection_errors_total` (Counter)**: Dropped connection attempts into upstream worker pools.
- **`target_request_distribution` (Counter)**: Verifying traffic divides proportionally; uneven distribution flags broken hashing or sticky-session traps.
- **`surge_queue_depth` (Gauge)**: Pending client connections queued in the balancer memory while upstream instances are running at 100% thread saturation.

## 9. Real-World Case Studies

### Mitigating Hotspot Nodes & Achieving Zero-Downtime Releases in Payment Gateways

- **The Engineering Problem**:
  A financial payment processing platform deployed 20 identical Go backend workers behind a basic Round-Robin Layer 7 balancer. Transaction execution profiles varied significantly: balance inquiry endpoints resolved in 5ms, while settlement reconciliation queries required upwards of 1,500ms. Blind Round-Robin scheduling continuously directed incoming requests to nodes already overwhelmed with long-running reconciliation queries. Three worker nodes suffered goroutine pool exhaustion, CPU loads pegged at 100%, and incoming user checkouts assigned to those nodes failed abruptly with HTTP 504 Gateway Timeouts.

- **The Architectural Solution**:
  1. **Dynamic Peak-EWMA Scheduling**: The load balancer was reconfigured from Round-Robin to *Peak Exponentially Weighted Moving Average* (Peak-EWMA), dynamically prioritizing backend targets maintaining the lowest active connection depth and lowest moving latency history.
  2. **Intelligent Deep Health Probes**: Backend `/healthz` endpoints were upgraded from simple static `200 OK` pings to dynamic status checks assessing active goroutine count and database connection pool availability. When a node’s goroutine count surpassed 5,000, it signaled degraded capacity, prompting the balancer to pause routing new traffic until the backlog cleared.
  3. **Graceful Connection Draining**: During rolling Kubernetes deployments, the ingress load balancer enforced a 30-second `deregistration_delay`, ensuring existing in-flight financial authorizations finished cleanly before worker pods were decommissioned.

- **The Engineering Outcome**:
  - HTTP 504 Gateway Timeouts were eradicated down to 0.00%.
  - Fleet-wide p99 latency dropped by 62% due to the complete elimination of overloaded hotspot nodes.
  - Development teams gained the ability to execute rolling production releases 20 times per day without dropping a single active customer transaction.

## 10. Interesting Facts & Production Gotchas

- **Health Check Flapping Cascades**: Hyper-aggressive health check configurations (e.g., 100ms timeout with 2 consecutive failures triggering removal) can prove lethal. If a server pauses briefly for Garbage Collection, the balancer marks it dead and dumps its traffic onto neighboring nodes. The remaining nodes buckle under the sudden surge, triggering a catastrophic domino failure across the entire cluster.
- **DNS Round Robin Is Not True Load Balancing**: Many developers initially attempt load balancing by binding multiple A-records in DNS. DNS servers possess zero awareness of node health, and aggressive client-side DNS caching prevents instantaneous failover when a target crashes.

## 11. Key Takeaways

1. **Traffic Equilibrium & Hotspot Prevention**: Ensures traffic distributes evenly across healthy server pools so no single machine collapses under localized queue buildup.
2. **Select Algorithms Matching Workload Realities**: Adopt *Least Connections* or *Peak-EWMA* for heterogeneous processing workloads; avoid static Round-Robin when execution durations vary widely.
3. **Adaptive Health Probing**: Health checks must gauge authentic service dependencies without being so hair-trigger sensitive that they induce cascading flapping failures.
4. **Connection Draining is Essential**: Always enforce connection draining grace periods during deployments to allow in-flight customer transactions to finish safely.
