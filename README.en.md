# High-Throughput Backend Handbook

Welcome to the **High-Throughput Backend Handbook**. This repository is designed as a comprehensive, practical, and junior-friendly guide to mastering core concepts of large-scale backend engineering and high-throughput distributed systems.

Here, we cover not just theory, but real-world industrial scenarios, why a technique is mandatory during severe traffic surges, architectural trade-offs (pros and cons), clean ASCII flow diagrams, production-grade Go (Golang) implementation snippets, when to use vs when to avoid guidelines, key observability metrics, real-world case studies, and common production gotchas.

> Read the Indonesian version: [Versi Bahasa Indonesia](README.md)

---

## Curriculum Roadmap (10 Architectural Layers - 49 Topics)

All curriculum modules are categorized into 10 primary directories reflecting the lifecycle of a request from system scaling foundations, external ingress, security/identity, data acceleration, system resilience, asynchronous communication, operational delivery, high-performance protocols, concrete tooling, to production observability:

```
+-------------------------------------------------------------------------+
| architecture-and-scaling/ (System Design & Scaling Foundations)         |
| 1. Horizontal Scaling    2. Vertical Scaling     3. Monolith vs Micro   |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| traffic-and-edge/ (Ingress Gateway, CORS, & Traffic Control)            |
| 4. Reverse Proxies       5. API Gateways         6. Load Balancing      |
| 7. Rate Limiting         8. CORS                 9. BFF Pattern         |
| 10. API Versioning                                                      |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| security-and-identity/ (Distributed Authentication & Access Security)   |
| 11. JWT Authentication & Token Lifecycle                                |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| performance-and-data/ (Acceleration, Migration, & Fast Data Access)     |
| 12. Caching              13. Database Indexing   14. SQL vs NoSQL       |
| 15. Database Pagination  16. Zero-Downtime DB Migrations                |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| resilience-and-fault-tolerance/ (Reliability & Fault Handling)          |
| 17. Timeouts             18. Retries             19. Exp. Backoff       |
| 20. Circuit Breakers     21. Idempotency         22. Dist. Locking      |
| 23. Payment Idempotency                                                 |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| async-and-distributed-systems/ (Large-Scale Communication)               |
| 24. Message Queues       25. Pub/Sub             26. Event-Driven Arch  |
| 27. Dist. Transactions   28. Saga Pattern                               |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| infrastructure-and-delivery/ (Operational Foundations & Delivery)       |
| 29. Service Discovery    30. CI/CD                                      |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| protocols-and-realtime/ (High-Performance Protocols & Real-Time)        |
| 31. gRPC & Protobuf      32. WebSockets & SSE    33. CDN & Edge Caching |
| 34. Webhooks                                                            |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| tools-and-technologies/ (Concrete High-Throughput Software)             |
| 35. Docker               36. Kubernetes          37. Redis              |
| 38. Apache Kafka         39. PostgreSQL          40. NGINX              |
| 41. Jenkins              42. RabbitMQ            43. Elasticsearch      |
| 44. PgBouncer            45. HAProxy                                    |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| monitoring-and-observability/ (Production Monitoring & Reliability)     |
| 46. Prometheus           47. Grafana             48. OpenTelemetry      |
| 49. Centralized Logging                                                 |
+-------------------------------------------------------------------------+
```

---

## Complete Topic Directory

Every folder contains bilingual documentation: `README.md` (Indonesian) and `README.en.md` (English). All backend implementation examples are written in **Go (Golang)** or verified production configuration formats.

### 1. Architecture & Scaling (`architecture-and-scaling/`)
- [Horizontal Scaling](architecture-and-scaling/horizontal-scaling/README.en.md) - Elastic capacity expansion by adding stateless worker nodes in parallel.
- [Vertical Scaling](architecture-and-scaling/vertical-scaling/README.en.md) - Enhancing compute throughput (CPU, RAM, NVMe) on a single physical host.
- [Monolith, Modular Monolith, and Microservices](architecture-and-scaling/monolith-vs-microservices/README.en.md) - Architectural design guide comparing monolithic simplicity, modular in-memory boundaries, and distributed microservices with Conway's law and real-world trade-offs.

### 2. Traffic & Edge (`traffic-and-edge/`)
- [Reverse Proxies](traffic-and-edge/reverse-proxies/README.en.md) - Intermediary protecting and routing traffic to internal backend servers.
- [API Gateways](traffic-and-edge/api-gateways/README.en.md) - Single entry point orchestrating, authenticating, and routing microservice APIs.
- [Load Balancing](traffic-and-edge/load-balancing/README.en.md) - Distributing traffic across server pools to avoid hotspot failures.
- [Rate Limiting](traffic-and-edge/rate-limiting/README.en.md) - Traffic throttling to shield internal services from abuse and overloads.
- [CORS (Cross-Origin Resource Sharing)](traffic-and-edge/cors/README.en.md) - Browser cross-origin security mechanism handling preflight OPTIONS, Access-Control headers, and preflight caching.
- [Backend for Frontend (BFF)](traffic-and-edge/bff-pattern/README.en.md) - Client-tailored gateway aggregation pattern that minimizes chatty round-trips and payload overfetching across mobile, web, and external platforms.
- [API Versioning & Deprecation](traffic-and-edge/api-versioning/README.en.md) - Evolution and contract management strategies across URI, Header, and Query parameters with standardized IETF Deprecation and Sunset governance.

### 3. Security & Identity (`security-and-identity/`)
- [JWT Authentication](security-and-identity/jwt-authentication/README.en.md) - Stateless scalable token authentication with cryptographic verification, refresh token rotation, and distributed blacklisting.

### 4. Performance & Data Access (`performance-and-data/`)
- [Caching](performance-and-data/caching/README.en.md) - High-speed in-memory data caching to drastically reduce database query latency.
- [Database Indexing](performance-and-data/database-indexing/README.en.md) - Auxiliary tree structures transforming $O(N)$ table scans into $O(\log N)$ lookups.
- [SQL vs NoSQL](performance-and-data/sql-vs-nosql/README.en.md) - Comparing transactional consistency (ACID) against horizontal scale-out and flexible schemas.
- [Database Pagination](performance-and-data/pagination/README.en.md) - Paginating large result sets comparing Offset ($O(N)$) vs Keyset/Cursor-based pagination ($O(1)$) to eliminate buffer thrashing.
- [Zero-Downtime DB Migrations](performance-and-data/zero-downtime-db-migrations/README.en.md) - Safe, zero-downtime database schema evolutions using the Expand and Contract (Parallel Run) pattern.

### 5. Resilience & Fault Tolerance (`resilience-and-fault-tolerance/`)
- [Timeouts](resilience-and-fault-tolerance/timeouts/README.en.md) - Enforcing upper latency bounds to prevent runaway thread starvation.
- [Retries](resilience-and-fault-tolerance/retries/README.en.md) - Automatic operational re-execution to overcome transient network glitches.
- [Exponential Backoff](resilience-and-fault-tolerance/exponential-backoff/README.en.md) - Exponential delay progression with randomized jitter to avert thundering herds.
- [Circuit Breakers](resilience-and-fault-tolerance/circuit-breakers/README.en.md) - Automatic circuit interruption protecting failing downstream services.
- [Idempotency](resilience-and-fault-tolerance/idempotency/README.en.md) - Mathematical guarantee that repeated identical requests produce uniform state changes.
- [Distributed Locking](resilience-and-fault-tolerance/distributed-locking/README.en.md) - Multi-instance concurrency synchronization using atomic Redis `SET NX PX` and safe identifier-matching Lua releases.
- [Payment Idempotency & Deduplication](resilience-and-fault-tolerance/payment-idempotency/README.en.md) - Financial transaction deduplication pattern preventing double charges via `Idempotency-Key` headers, SHA-256 payload fingerprinting, and cached response replays.

### 6. Asynchronous & Distributed Systems (`async-and-distributed-systems/`)
- [Message Queues](async-and-distributed-systems/message-queues/README.en.md) - Point-to-point asynchronous messaging acting as shock absorbers during traffic spikes.
- [Pub/Sub](async-and-distributed-systems/pub-sub/README.en.md) - Decoupled one-to-many event fan-out distribution.
- [Event-Driven Architecture](async-and-distributed-systems/event-driven-architecture/README.en.md) - Decentralized, event-responsive decoupled system topologies.
- [Distributed Transactions](async-and-distributed-systems/distributed-transactions/README.en.md) - Coordinating consistency across isolated databases (and its trade-offs).
- [Saga Pattern](async-and-distributed-systems/saga-pattern/README.en.md) - Distributed transactions executed via sequential local transactions and compensating rollbacks.

### 7. Infrastructure & Delivery (`infrastructure-and-delivery/`)
- [Service Discovery](infrastructure-and-delivery/service-discovery/README.en.md) - Dynamic IP and port resolution in ephemeral microservice environments.
- [CI/CD](infrastructure-and-delivery/ci-cd/README.en.md) - Automated integration, testing, and continuous deployment validation pipelines.

### 8. Protocols & Real-Time (`protocols-and-realtime/`)
- [gRPC & Protocol Buffers](protocols-and-realtime/grpc-and-protobuf/README.en.md) - High-performance binary RPC protocol over HTTP/2 for service-to-service communication.
- [WebSockets & Server-Sent Events](protocols-and-realtime/websockets-and-sse/README.en.md) - Full-duplex persistent communication and low-latency server-to-client event streaming.
- [CDN & Edge Caching](protocols-and-realtime/cdn-and-edge-caching/README.en.md) - Edge content delivery and dynamic caching closest to end users.
- [Webhooks](protocols-and-realtime/webhooks/README.en.md) - Proactive HTTP event dispatching (*Reverse API*) with HMAC SHA256 cryptographic verification.

### 9. Tools & Technologies (`tools-and-technologies/`)
- [Docker](tools-and-technologies/docker/README.en.md) - Application containerization providing immutable runtime isolation across environments.
- [Kubernetes](tools-and-technologies/kubernetes/README.en.md) - Container orchestration at scale providing automated scheduling, self-healing, and rolling deployments.
- [Redis](tools-and-technologies/redis/README.en.md) - In-memory key-value engine powering low-latency caches, distributed locks, and real-time streaming buffers.
- [Apache Kafka](tools-and-technologies/kafka/README.en.md) - Distributed append-only commit-log streaming platform handling millions of events per second.
- [PostgreSQL](tools-and-technologies/postgresql/README.en.md) - Enterprise-grade relational database engine leveraging MVCC, WAL, and strict ACID transaction guarantees.
- [NGINX](tools-and-technologies/nginx/README.en.md) - Asynchronous event-driven web server and reverse proxy handling high concurrent connection buffering and TLS termination.
- [Jenkins](tools-and-technologies/jenkins/README.en.md) - Battle-tested self-hosted automation server for orchestrating scalable CI/CD build and delivery pipelines.
- [RabbitMQ](tools-and-technologies/rabbitmq/README.en.md) - Flexible AMQP message broker supporting complex routing topologies (exchanges/queues) and per-message acknowledgments.
- [Elasticsearch](tools-and-technologies/elasticsearch/README.en.md) - Distributed inverted-index search and analytics engine for full-text querying and BM25 relevance scoring.
- [PgBouncer](tools-and-technologies/pgbouncer/README.en.md) - Lightweight connection pooler shielding PostgreSQL from connection limits and fork overhead.
- [HAProxy](tools-and-technologies/haproxy/README.en.md) - High-performance Layer 4 and Layer 7 load balancer known for deterministic low latency and precise health checking.

### 10. Monitoring & Observability (`monitoring-and-observability/`)
- [Prometheus](monitoring-and-observability/prometheus/README.en.md) - Time-series metrics monitoring platform utilizing a pull model and PromQL expressions.
- [Grafana](monitoring-and-observability/grafana/README.en.md) - Central visualization dashboard unifying metrics, logs, and distributed traces.
- [OpenTelemetry](monitoring-and-observability/opentelemetry/README.en.md) - Vendor-neutral observability framework for distributed tracing and context propagation.
- [Centralized Logging](monitoring-and-observability/centralized-logging/README.en.md) - High-volume structured log aggregation (JSON) using Grafana Loki / Elasticsearch stacks.

---

## Study Guide

1. **Follow the Layer Hierarchy**: It is recommended to read sequentially from foundation scaling (`architecture-and-scaling`), entry gateways (`traffic-and-edge`), identity (`security-and-identity`), data layer (`performance-and-data`), fault resilience (`resilience-and-fault-tolerance`), async messaging (`async-and-distributed-systems`), operational infrastructure (`infrastructure-and-delivery`), protocols (`protocols-and-realtime`), concrete tooling (`tools-and-technologies`), to observability (`monitoring-and-observability`) to form an integrated mental model.
2. **Scrutinize Trade-offs and Gotchas**: In distributed systems, there are no silver bullets—only sets of trade-offs.
3. **Internalize System Scenarios**: Every module starts with real-world industry architecture context explaining why the technology is mandatory during high-throughput stress.
