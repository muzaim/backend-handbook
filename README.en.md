# High-Throughput Backend Handbook

Welcome to the **High-Throughput Backend Handbook**. This repository is designed as a comprehensive, practical, and junior-friendly guide to mastering core concepts of large-scale backend engineering and high-throughput distributed systems.

Here, we cover not just theory, but real-world industrial scenarios, why a technique is mandatory during severe traffic surges, architectural trade-offs (pros and cons), clean ASCII flow diagrams, production-grade Go (Golang) implementation snippets, when to use vs when to avoid guidelines, key observability metrics, real-world case studies, and common production gotchas.

> Read the Indonesian version: [Versi Bahasa Indonesia](README.md)

---

## Curriculum Roadmap (9 Architectural Layers - 40 Topics)

All curriculum modules are categorized into 9 primary directories reflecting the lifecycle of a request from system scaling foundations, external ingress, data acceleration, operational infrastructure, concrete tooling, to production observability:

```
+-------------------------------------------------------------------------+
| architecture-and-scaling/ (System Design & Scaling Foundations)         |
| 1. Horizontal Scaling    2. Vertical Scaling                            |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| traffic-and-edge/ (Ingress Gateway & Traffic Control)                   |
| 3. Reverse Proxies       4. API Gateways                                |
| 5. Load Balancing        6. Rate Limiting                               |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| performance-and-data/ (Acceleration & Fast Data Access)                 |
| 7. Caching               8. Database Indexing    9. SQL vs NoSQL        |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| resilience-and-fault-tolerance/ (Reliability & Fault Handling)          |
| 10. Timeouts             11. Retries             12. Exp. Backoff       |
| 13. Circuit Breakers     14. Idempotency                                |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| async-and-distributed-systems/ (Large-Scale Communication)               |
| 15. Message Queues       16. Pub/Sub             17. Event-Driven Arch  |
| 18. Dist. Transactions   19. Saga Pattern                               |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| infrastructure-and-delivery/ (Operational Foundations & Delivery)       |
| 20. Service Discovery    21. CI/CD                                      |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| protocols-and-realtime/ (High-Performance Protocols & Real-Time)        |
| 22. gRPC & Protobuf      23. WebSockets & SSE    24. CDN & Edge Caching |
| 25. Webhooks                                                            |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| tools-and-technologies/ (Concrete High-Throughput Software)             |
| 26. Docker               27. Kubernetes          28. Redis              |
| 29. Apache Kafka         30. PostgreSQL          31. NGINX              |
| 32. Jenkins              33. RabbitMQ            34. Elasticsearch      |
| 35. PgBouncer            36. HAProxy                                    |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| monitoring-and-observability/ (Production Monitoring & Reliability)     |
| 37. Prometheus           38. Grafana             39. OpenTelemetry      |
| 40. Centralized Logging                                                 |
+-------------------------------------------------------------------------+
```

---

## Complete Topic Directory

Every folder contains bilingual documentation: `README.md` (Indonesian) and `README.en.md` (English). All backend implementation examples are written in **Go (Golang)** or verified production configuration formats.

### 1. Architecture & Scaling (`architecture-and-scaling/`)
- [Horizontal Scaling](architecture-and-scaling/horizontal-scaling/README.en.md) - Elastic capacity expansion by adding stateless worker nodes in parallel.
- [Vertical Scaling](architecture-and-scaling/vertical-scaling/README.en.md) - Enhancing compute throughput (CPU, RAM, NVMe) on a single physical host.

### 2. Traffic & Edge (`traffic-and-edge/`)
- [Reverse Proxies](traffic-and-edge/reverse-proxies/README.en.md) - Intermediary protecting and routing traffic to internal backend servers.
- [API Gateways](traffic-and-edge/api-gateways/README.en.md) - Single entry point orchestrating, authenticating, and routing microservice APIs.
- [Load Balancing](traffic-and-edge/load-balancing/README.en.md) - Distributing traffic across server pools to avoid hotspot failures.
- [Rate Limiting](traffic-and-edge/rate-limiting/README.en.md) - Traffic throttling to shield internal services from abuse and overloads.

### 3. Performance & Data Access (`performance-and-data/`)
- [Caching](performance-and-data/caching/README.en.md) - High-speed in-memory data caching to drastically reduce database query latency.
- [Database Indexing](performance-and-data/database-indexing/README.en.md) - Auxiliary tree structures transforming $O(N)$ table scans into $O(\log N)$ lookups.
- [SQL vs NoSQL](performance-and-data/sql-vs-nosql/README.en.md) - Comparing transactional consistency (ACID) against horizontal scale-out and flexible schemas.

### 4. Resilience & Fault Tolerance (`resilience-and-fault-tolerance/`)
- [Timeouts](resilience-and-fault-tolerance/timeouts/README.en.md) - Enforcing upper latency bounds to prevent runaway thread starvation.
- [Retries](resilience-and-fault-tolerance/retries/README.en.md) - Automated re-execution to overcome transient network and socket disruptions.
- [Exponential Backoff](resilience-and-fault-tolerance/exponential-backoff/README.en.md) - Exponentially decaying retry intervals with jitter to mitigate thundering herds.
- [Circuit Breakers](resilience-and-fault-tolerance/circuit-breakers/README.en.md) - Fast-failing traffic to failing downstream dependencies to enable recovery.
- [Idempotency](resilience-and-fault-tolerance/idempotency/README.en.md) - Guaranteeing that duplicated requests yield identical system side-effects.

### 5. Asynchronous & Distributed Systems (`async-and-distributed-systems/`)
- [Message Queues](async-and-distributed-systems/message-queues/README.en.md) - Point-to-point asynchronous messaging buffers absorbing traffic spikes.
- [Pub/Sub](async-and-distributed-systems/pub-sub/README.en.md) - Decoupled one-to-many event broadcast distribution by topic.
- [Event-Driven Architecture](async-and-distributed-systems/event-driven-architecture/README.en.md) - Systems communicating through emitted state changes and decoupled consumers.
- [Distributed Transactions](async-and-distributed-systems/distributed-transactions/README.en.md) - Data consistency challenges across distributed databases (and their limits).
- [Saga Pattern](async-and-distributed-systems/saga-pattern/README.en.md) - Managing distributed long-running business workflows through compensating actions.

### 6. Infrastructure & Delivery (`infrastructure-and-delivery/`)
- [Service Discovery](infrastructure-and-delivery/service-discovery/README.en.md) - Dynamic tracking enabling microservices to locate peer IPs and ports.
- [CI/CD](infrastructure-and-delivery/ci-cd/README.en.md) - Automated integration, testing, and progressive production delivery pipelines.

### 7. Protocols & Real-Time (`protocols-and-realtime/`)
- [gRPC & Protocol Buffers](protocols-and-realtime/grpc-and-protobuf/README.en.md) - Binary HTTP/2 RPC protocol delivering high-speed inter-service communication.
- [WebSockets & Server-Sent Events](protocols-and-realtime/websockets-and-sse/README.en.md) - Full-duplex persistent connections and low-latency server event streaming.
- [CDN & Edge Caching](protocols-and-realtime/cdn-and-edge-caching/README.en.md) - Terminating and serving static assets and dynamic caches closest to end users.
- [Webhooks](protocols-and-realtime/webhooks/README.en.md) - Event-driven HTTP push callbacks (*Reverse API*) secured with cryptographic HMAC signatures.

### 8. Tools & Technologies (`tools-and-technologies/`)
- [Docker](tools-and-technologies/docker/README.en.md) - Application containerization ensuring consistent, isolated runtime environments.
- [Kubernetes](tools-and-technologies/kubernetes/README.en.md) - Large-scale container orchestration for automated self-healing, scaling, and rollouts.
- [Redis](tools-and-technologies/redis/README.en.md) - In-memory key-value data store for low-latency caching, distributed locking, and queues.
- [Apache Kafka](tools-and-technologies/kafka/README.en.md) - Distributed append-only log streaming platform handling millions of events/sec.
- [PostgreSQL](tools-and-technologies/postgresql/README.en.md) - Enterprise relational database engine featuring MVCC concurrency and strict ACID durability.
- [NGINX](tools-and-technologies/nginx/README.en.md) - Asynchronous event-driven web server and reverse proxy for high-concurrency buffering and TLS termination.
- [Jenkins](tools-and-technologies/jenkins/README.en.md) - Self-hosted CI/CD automation server orchestrating distributed pipelines and deployment gates.
- [RabbitMQ](tools-and-technologies/rabbitmq/README.en.md) - AMQP message broker providing flexible exchange routing and per-message manual acknowledgments.
- [Elasticsearch](tools-and-technologies/elasticsearch/README.en.md) - Distributed full-text search and analytics engine powered by Inverted Indexes and BM25 scoring.
- [PgBouncer](tools-and-technologies/pgbouncer/README.en.md) - Lightweight connection pooler shielding PostgreSQL from high-concurrency connection starvation.
- [HAProxy](tools-and-technologies/haproxy/README.en.md) - High-throughput Layer 4 & Layer 7 load balancer renowned for memory efficiency and health detection.

### 9. Monitoring & Observability (`monitoring-and-observability/`)
- [Prometheus](monitoring-and-observability/prometheus/README.en.md) - Pull-based time-series metrics monitoring with real-time PromQL querying.
- [Grafana](monitoring-and-observability/grafana/README.en.md) - Unified operational dashboarding (Single Pane of Glass) across metrics, logs, and traces.
- [OpenTelemetry](monitoring-and-observability/opentelemetry/README.en.md) - Universal vendor-neutral standard for distributed tracing and context propagation.
- [Centralized Logging](monitoring-and-observability/centralized-logging/README.en.md) - Large-scale structured JSON log aggregation and search using Grafana Loki / Elasticsearch.

---

## Study Guide

1. **Follow the Layer Hierarchy**: We recommend reading sequentially starting from architectural foundations (`architecture-and-scaling`), outer edge ingress (`traffic-and-edge`), data access, resilience, asynchronous messaging, delivery, protocols, tools (`tools-and-technologies`), and finally observability (`monitoring-and-observability`).
2. **Pay Close Attention to Trade-offs & Gotchas**: In distributed architecture, there are no perfect solutions—only strategic trade-offs.
3. **Understand the Real-World Scenario**: Every topic begins with an industrial production scenario demonstrating precisely why that pattern is indispensable when traffic surges.
