# High-Throughput Backend Handbook

Welcome to the **High-Throughput Backend Handbook**. This repository is an educational, practical, and junior-friendly reference for mastering core concepts in high-throughput distributed backend architectures.

Rather than presenting dry theory, each guide provides real-world system scenarios, technical explanations of why a concept is needed under heavy load, trade-offs (pros and cons), clean ASCII diagrams, core implementation logic in Go (Golang), decision frameworks (when to use vs when not to use), key observability metrics, real-world industry case studies, and production gotchas.

> Read the Indonesian version: [Versi Bahasa Indonesia](README.md)

---

## Curriculum Roadmap (8 Architectural Categories)

All topics in this repository are categorized into 8 primary architectural folders, mapping directly to the journey of an inbound request from the network boundary to persistent data, infrastructure, software tools, and production observability:

```
+-------------------------------------------------------------------------+
| traffic-and-edge/ (Network Perimeters & Ingress Traffic)                |
| 1. Reverse Proxies       2. API Gateways                                |
| 3. Load Balancing        4. Rate Limiting                               |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| performance-and-data/ (High-Speed Data Acceleration)                    |
| 5. Caching                                                              |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| resilience-and-fault-tolerance/ (Failure Isolation & Stability)         |
| 6. Timeouts              7. Retries                                     |
| 8. Exponential Backoff   9. Circuit Breakers                            |
| 10. Idempotency                                                         |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| async-and-distributed-systems/ (Large-Scale Asynchronous Decoupling)    |
| 11. Message Queues       12. Pub/Sub                                    |
| 13. Event-Driven Arch    14. Distributed Transactions                   |
| 15. Saga Pattern                                                        |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| infrastructure-and-delivery/ (Operational Foundations & Release)        |
| 16. Service Discovery    17. CI/CD                                      |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| protocols-and-realtime/ (High-Performance Protocols & Real-Time)        |
| 18. gRPC & Protobuf      19. WebSockets & SSE                           |
| 20. CDN & Edge Caching                                                  |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| tools-and-technologies/ (Concrete High-Throughput Software Tools)       |
| 21. Docker               22. Kubernetes                                 |
| 23. Redis                24. Apache Kafka                               |
| 25. PostgreSQL           26. NGINX                                      |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| monitoring-and-observability/ (Telemetry, Metrics & Incident Response)  |
| 27. Prometheus           28. Grafana                                    |
| 29. OpenTelemetry        30. Centralized Logging                        |
+-------------------------------------------------------------------------+
```

---

## Complete Topic Index

Each topic folder includes bilingual documentation: `README.md` (Indonesian) and `README.en.md` (English). All backend implementation logic is written in **Go (Golang)**.

### 1. Traffic & Edge (`traffic-and-edge/`)
- [Reverse Proxies](traffic-and-edge/reverse-proxies/README.en.md) - Shielding and mediating between clients and upstream backend servers.
- [API Gateways](traffic-and-edge/api-gateways/README.en.md) - Single entry point for microservices routing, authentication, and orchestration.
- [Load Balancing](traffic-and-edge/load-balancing/README.en.md) - Distributing traffic across multiple server instances to prevent overload.
- [Rate Limiting](traffic-and-edge/rate-limiting/README.en.md) - Controlling traffic throughput to prevent service abuse and system exhaustion.

### 2. Performance & Data Access (`performance-and-data/`)
- [Caching](performance-and-data/caching/README.en.md) - High-speed temporary data storage to eliminate redundant database trips.

### 3. Resilience & Fault Tolerance (`resilience-and-fault-tolerance/`)
- [Timeouts](resilience-and-fault-tolerance/timeouts/README.en.md) - Enforcing deadlines to stop unresponsive requests from hogging threads and sockets.
- [Retries](resilience-and-fault-tolerance/retries/README.en.md) - Automatically re-attempting failed operations to overcome transient network glitches.
- [Exponential Backoff](resilience-and-fault-tolerance/exponential-backoff/README.en.md) - Progressively backing off retry intervals with randomized jitter.
- [Circuit Breakers](resilience-and-fault-tolerance/circuit-breakers/README.en.md) - Halting incoming requests to failing downstream services to give them breathing room.
- [Idempotency](resilience-and-fault-tolerance/idempotency/README.en.md) - Ensuring duplicate operations produce the exact same final system state.

### 4. Asynchronous & Distributed Systems (`async-and-distributed-systems/`)
- [Message Queues](async-and-distributed-systems/message-queues/README.en.md) - Point-to-point asynchronous messaging for traffic smoothing and decoupling.
- [Pub/Sub](async-and-distributed-systems/pub-sub/README.en.md) - One-to-many event broadcasting pattern using topic subscriptions.
- [Event-Driven Architecture](async-and-distributed-systems/event-driven-architecture/README.en.md) - Building decoupled reactive systems responding to state events.
- [Distributed Transactions](async-and-distributed-systems/distributed-transactions/README.en.md) - Understanding ACID limitations across isolated databases.
- [Saga Pattern](async-and-distributed-systems/saga-pattern/README.en.md) - Coordinating cross-service business transactions through local transactions and compensation.

### 5. Infrastructure & Delivery (`infrastructure-and-delivery/`)
- [Service Discovery](infrastructure-and-delivery/service-discovery/README.en.md) - Dynamic tracking of dynamic container IP addresses and ports.
- [CI/CD](infrastructure-and-delivery/ci-cd/README.en.md) - Automated integration, testing, and progressive release pipelines.

### 6. Protocols & Real-Time (`protocols-and-realtime/`)
- [gRPC & Protocol Buffers](protocols-and-realtime/grpc-and-protobuf/README.en.md) - Binary RPC communication over HTTP/2 for high-throughput inter-service traffic.
- [WebSockets & Server-Sent Events](protocols-and-realtime/websockets-and-sse/README.en.md) - Bidirectional real-time channels and low-latency server-sent event streams.
- [CDN & Edge Caching](protocols-and-realtime/cdn-and-edge-caching/README.en.md) - Globally distributed edge caching and static asset acceleration.

### 7. Tools & Technologies (`tools-and-technologies/`)
- [Docker](tools-and-technologies/docker/README.en.md) - Containerizing applications for portable, isolated, and reproducible execution environments.
- [Kubernetes](tools-and-technologies/kubernetes/README.en.md) - Container orchestration platform for self-healing, scaling, and rolling deployments.
- [Redis](tools-and-technologies/redis/README.en.md) - Ultra-fast in-memory key-value data store for caching, distributed locking, and real-time queues.
- [Apache Kafka](tools-and-technologies/kafka/README.en.md) - Distributed commit log event streaming platform capable of handling millions of events per second.
- [PostgreSQL](tools-and-technologies/postgresql/README.en.md) - Enterprise relational database engine featuring MVCC concurrency, WAL durability, and ACID guarantees.
- [NGINX](tools-and-technologies/nginx/README.en.md) - High-performance asynchronous reverse proxy, web server, and SSL termination engine.

### 8. Monitoring & Observability (`monitoring-and-observability/`)
- [Prometheus](monitoring-and-observability/prometheus/README.en.md) - Pull-based time-series metrics monitoring system with real-time PromQL evaluation.
- [Grafana](monitoring-and-observability/grafana/README.en.md) - Operational visualization command center (Single Pane of Glass) for metrics, logs, and traces.
- [OpenTelemetry](monitoring-and-observability/opentelemetry/README.en.md) - Industry-standard observability framework for distributed tracing and context propagation.
- [Centralized Logging](monitoring-and-observability/centralized-logging/README.en.md) - High-throughput structured JSON log aggregation using Grafana Loki / Elasticsearch.

---

## Learning Recommendations

1. **Follow the category path**: Progressing sequentially from `traffic-and-edge` through `tools-and-technologies` and `monitoring-and-observability` builds an end-to-end mental model.
2. **Review Trade-offs and Gotchas**: Every architectural solution introduces operational complexity; there are no silver bullets.
3. **Anchor in real-world scenarios**: Each guide begins with an industry-grade system scenario to contextualize why the technology is mandatory under extreme traffic load.
