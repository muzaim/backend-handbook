# Service Discovery

## 1. Real-World System Scenario
In dynamic cloud microservices architectures, the `Order Service` must communicate with an internal `Payment Service` over private HTTP sockets without hardcoded IP configurations.
In dynamic cloud microservices architectures, the `Order Service` must communicate with an internal `Payment Service` over private HTTP sockets without hardcoded IP configurations.

In modern containerized cloud computing (such as Kubernetes or AWS ECS), backend pod instances are strictly **Ephemeral**:
- Every time developers roll out an updated image tag or autoscalers adjust capacity, old pods are terminated and replaced with brand-new pods allocated random private IP subnets (e.g., mutating from `10.244.1.24` to `10.244.4.91`).
- If you hardcode a static network address inside configuration files: `PAYMENT_API_URL=http://10.244.1.24:8080`, the moment a rolling update occurs, the `Order Service` instantly fails with endless `Connection Refused` errors.
- If you scale the payment service from 3 to 30 pod replicas during a traffic surge, the operation yields zero benefit because the caller is blind to the network coordinates of the 27 newly spawned instances!

**The Service Discovery Solution (e.g., Kubernetes CoreDNS / HashiCorp Consul):**
Whenever a payment pod boots, it automatically registers its private IP address and readiness status with a centralized **Service Registry**. The `Order Service` never needs to know dynamic IP coordinates; it simply addresses a stable logical DNS identity: `http://payment-service:8080`. The Service Discovery layer dynamically and transparently resolves that name to currently healthy, live pod IPs in real time.

## 2. What Is It and What Is It For?

Service Discovery is the automated networking mechanism in distributed architectures that enables software components to dynamically detect the network locations (**IP Addresses and Port Numbers**) of upstream dependencies without hardcoding brittle static IP configurations.

Service Discovery rests upon three primary pillars:
1. **Service Registry**: A highly consistent distributed key-value database acting as the authoritative central directory of all active service instances and their health statuses (Consul, Eureka, etcd, ZooKeeper, Kubernetes CoreDNS).
2. **Service Registration**: The lifecycle routine where a newly booted container instance registers its private IP, listening port, and metadata with the Registry, sending periodic *Heartbeats*. Upon graceful shutdown, it deregisters.
3. **Service Lookup**: The runtime query where Service A discovers the current healthy IP addresses belonging to Service B before initiating network communication.

## 3. Two Implementation Approaches: Client-Side vs Server-Side

```
[ 1. Client-Side Discovery (e.g. Netflix Eureka / Consul SDK) ]

+-----------------+                 +-----------------------+
|   Order Svc     | --1. Query IP-> |   Service Registry    |
| (Client-Side)   | <-2. IP List--- | (Consul / Eureka)     |
+-----------------+                 +-----------------------+
        |
        | 3. Client selects an IP directly via Client-Side Balancer
        v
+-----------------+
| Payment Svc #2  |
| (10.0.1.45:8080)|
+-----------------+
```

```
[ 2. Server-Side Discovery (e.g. Kubernetes Services / AWS ALB) ]

+-----------------+
|   Order Svc     |
+-----------------+
        |
        | 1. Invoke stable virtual DNS: "http://payment-service"
        v
+---------------------------------------+
|         Internal Proxy / Balancer     |
|  (K8s Service / Kube-Proxy / Ingress) | <--- Continuously synced to Registry
+---------------------------------------+
        |
        | 2. Proxy forwards packet to a healthy target pod
        v
+-----------------+
| Payment Pod #2  |
+-----------------+
```

## 4. Why Use It in High-Throughput Systems?

In modern containerized cloud computing (Kubernetes), server instances are strictly **Ephemeral**:
- Pods are constantly terminated, rescheduled, and multiplied during autoscaling, node rotations, or failure recovery.
- A container pod might live for only 4 hours, and upon recreating, it receives a completely different internal IP address (e.g., rotating from `10.244.1.15` to `10.244.2.89`).

If you hardcode static IP addresses in application config files:
- The moment a pod restarts, dependent services fail with immediate `Connection Refused` exceptions.
- Scaling your worker fleet from 5 to 50 nodes during a traffic spike achieves zero benefit because existing callers are blind to the existence of the 45 new instances!

With Service Discovery, dynamic network topology mutations (scaling up, scaling down, rolling deployments) propagate globally across the cluster in milliseconds.

## 5. Core Implementation Logic (Client-Side Service Discovery di Go)

Implementation of client-side discovery with in-memory caching in Go dengan in-memory caching dan health checking di Go:

```go
package main

import (
	"fmt"
	"net/http"
	"sync"
	"time"
)

type ServiceDiscoveryClient struct {
	mu          sync.RWMutex
	serviceName string
	instances   []string
	lastUpdated time.Time
}

// GetInstance mengembalikan endpoint sehat dari cache in-memory lokal
func (sd *ServiceDiscoveryClient) GetInstance() (string, error) {
	sd.mu.RLock()
	if time.Since(sd.lastUpdated) < 10*time.Second && len(sd.instances) > 0 {
		inst := sd.instances[0]
		sd.mu.RUnlock()
		return inst, nil
	}
	sd.mu.RUnlock()

	// Refresh daftar endpoint dari Service Registry (misal Consul / etcd)
	sd.mu.Lock()
	defer sd.mu.Unlock()
	instances, err := fetchFromRegistry(sd.serviceName)
	if err != nil {
		if len(sd.instances) > 0 {
			return sd.instances[0], nil // Fallback ke cache lama
		}
		return "", err
	}
	sd.instances = instances
	sd.lastUpdated = time.Now()
	return sd.instances[0], nil
}

func fetchFromRegistry(name string) ([]string, error) {
	// Melakukan query HTTP ke Consul agent: GET /v1/health/service/:name
	_ = http.Get
	return []string{"10.0.1.20:8080", "10.0.1.21:8080"}, nil
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Zero-Touch Elastic Scalability**: Newly spawned replicas ingest production traffic the moment readiness probes pass without configuration changes.
- **Automated Fault Isolation**: Crashed nodes are culled from discovery directories within seconds, preventing traffic blackholing.
- **Clean Service Name Abstractions**: Developers target human-readable logical identities (`payment-service`) rather than tracking ephemeral IP subnets.

### Cons
- **Critical Centralized Dependency**: If the Service Registry cluster collapses and clients lack local caching, all inter-service communication across the enterprise freezes.
- **Heartbeat Floods**: Thousands of microservice containers aggressively emitting 1-second keep-alive heartbeats can saturate registry CPU and network sockets.
- **Lookup Latency Overhead**: Introduces a resolution hop prior to initiating underlying socket connections.

## 7. When to Use vs When Not to Use

### When to Use
- All distributed microservices running on elastic container infrastructure (Kubernetes, AWS ECS, Nomad, Docker Swarm).
- Dynamic autoscaling environments responding to volatile, real-time traffic surges.

### When Not to Use
- Traditional monolithic applications communicating with a single dedicated database host on a permanent static IP.
- Fixed small-scale on-premise hardware deployments where static private DNS records never change.

## 8. Key Metrics & Observability

Observability metrics to track dynamic service discovery health on Grafana or Datadog:

- **`service_registry_healthy_instances_count`**: Count of healthy nodes registered per service identity; sharp drops alert on rollout failures.
- **`service_discovery_lookup_latency_ms`**: In-memory and external network latency observed during service queries.
- **`service_registry_heartbeat_failure_rate`**: Percentage of failed keep-alive heartbeats triggering node deregulation.
- **`dns_lookup_duration_seconds` (CoreDNS)**: Internal DNS resolution latency for cluster services.
- **`service_registry_churn_rate`**: Frequency of pod registrations and unregistrations per minute; excessive churn flags flapping pods.

## 9. Real-World Case Studies

### Dynamic Routing Resolution in a 10,000-Pod Kubernetes Cluster

- **The Engineering Problem**: An on-demand logistics platform scaled 120 microservices dynamically from 1,000 to 10,000 pods during seasonal surges. Internal traffic was originally routed via centralized internal Nginx instances with static upstream blocks refreshed by cron scripts. During rapid pod termination and churn, reload race conditions caused 5% to 8% of internal RPCs to fail with `502 Bad Gateway` as requests hit deallocated container IPs (traffic blackholing).
- **The Architectural Solution**: The infrastructure was migrated to dynamic service discovery using *Kubernetes EndpointSlices* and *Envoy sidecar proxies*. Envoy instances ingested live endpoint state over gRPC Endpoint Discovery Service (EDS) streams without requiring process restarts. The team added Kubernetes container lifecycle `preStop` hooks (a 5-second sleep) to ensure Endpoint controllers purged pod IPs from the routing mesh before termination signals reached the application container, backed by aggressive readiness probes.
- **The System Outcome**: Inter-service 502 errors during deployment rollouts and autoscaling dropped from 6.5% to 0.001%. Endpoint convergence time decreased from 45 seconds to under 200 ms. Router CPU spikes caused by frequent process reloads were completely eliminated.

## 10. Interesting Facts & Production Gotchas

- **The Infamous Java JVM DNS Cache Gotcha**: In standard Oracle and OpenJDK runtimes, the default security policy caches DNS hostname resolutions with **`networkaddress.cache.ttl = -1` (meaning: CACHE FOREVER)**! When a Kubernetes pod terminates and its IP changes, Java services continue pounding the deceased IP address indefinitely, throwing millions of `ConnectException` errors. In cloud environments, always explicitly configure JVM security properties: set `networkaddress.cache.ttl` to a low value like 5 to 10 seconds!
- **Deregistration Lag & Blackholing**: When a pod shuts down ungracefully, a brief multi-second gap occurs before the Registry acknowledges the missing heartbeat and purges the entry. Upstream callers may continue routing traffic to the dead socket during this window. Always combine Service Discovery with **Graceful Shutdown Hooks (`preStop`)** and **Client-Side Retries with Short Timeouts**.

## 11. Key Takeaways

- **Container IP Ephemerality**: Container IPs in modern clouds change constantly; service discovery dynamically maps logical service names to healthy socket pools.
- **Client-Side vs Server-Side Trade-offs**: Client-side discovery (Consul/Ribbon) reduces network hops by load-balancing in-process, whereas server-side discovery (K8s Service/ALB) keeps microservice code clean and language-agnostic.
- **Prevent Traffic Blackholing**: Utilize pod termination hooks (`preStop`) and strict health checks so instances are deregistered from routing tables before accepting socket closures.
- **Tame Long-Lived DNS Caching**: Configure application HTTP client runtimes (especially the JVM) with low DNS TTL limits (3-10 seconds) to avoid persistent connection failures to dead IPs.
