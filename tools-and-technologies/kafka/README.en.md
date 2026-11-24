# Apache Kafka

## 1. Real-World System Scenario

In large-scale mobility and on-demand platforms (like Uber, Lyft, or Grab), hundreds of thousands of GPS updates and transactions stream continuously every second: Every single second:
- 500,000 active drivers stream their live GPS coordinates from mobile handsets.
- 100,000 concurrent payment authorizations, ride bookings, and discount redemptions execute across the platform.
- 5 independent downstream engineering systems require access to this real-time event firehose simultaneously: the *Fraud Detection Engine* (flagging fake rides on the fly), the *Dynamic Surge Pricing Engine* (calculating block-level pricing multipliers), the *Driver Dispatch Matching Service*, the *Data Lake / Analytics Pipeline*, and the *Customer Notification Service*.

If this architecture were designed using traditional HTTP REST endpoints or standard message queues (such as RabbitMQ point-to-point queues):
- When the Data Lake ingestion jobs experience temporary processing slowdowns (*consumer lag*), unconsumed messages pile up inside the broker's RAM until it suffers fatal *Out-Of-Memory (OOM) crashes*.
- In conventional queues, once a message is acknowledged by one worker, it is permanently deleted from the queue. Supporting multiple independent consumers reading historical data requires complex fan-out duplicates.
- Message throughput bottlenecks at tens of thousands of messages per second due to the CPU overhead of per-message acknowledgment tracking inside broker memory.

**Solution with Apache Kafka:**
All GPS updates and transaction events are appended to Kafka as a distributed, durable **Append-Only Commit Log** partitioned across multiple disks. Kafka does not delete messages when read; events are retained reliably on high-speed disk storage for days (e.g., 7 days). All 5 downstream systems ingest the stream independently using dedicated **Consumer Groups** tracking their own read offsets. Throughput skyrockets to **millions of events per second** with single-digit millisecond latency, enabled by the OS Page Cache and Linux Zero-Copy optimizations!

## 2. What Is It and What Is It For?

**Apache Kafka** is a distributed event streaming platform built for high-throughput, low-latency data pipelines, real-time streaming analytics, and cross-service data integration with fault-tolerant durability.

### Core Concepts:
1. **Event / Message**: An immutable record containing a Key, Value, Timestamp, and optional metadata Headers.
2. **Topic**: A logical category or feed to which events are published (analogous to a database table name or directory).
3. **Partition**: Topics are divided into multiple independent Partitions distributed across cluster brokers, enabling horizontal scalability and parallel processing.
4. **Offset**: A strictly increasing, sequential integer identifier assigned to each message within a partition, serving as an immutable record coordinate.
5. **Producer**: Client applications that publish (*write*) events to Kafka topics.
6. **Consumer & Consumer Group**: Client applications that subscribe to and process topic events. Consumers sharing a common `group.id` automatically divide topic partitions among themselves for parallel load-balanced consumption.

## 3. Why Use It in High-Throughput Systems?

### 1. High-Performance Sequential Disk I/O
A widespread engineering myth is that disk storage is inherently slow. The actual performance bottleneck is *Random Disk Access*. For **Sequential Disk Writes (Append-Only Logs)**, modern NVMe and enterprise rotational disks achieve sustained write speeds of 600+ MB/s—rivaling random memory access speeds. Kafka writes strictly sequentially to the tail of partition segment files.

### 2. Zero-Copy Network Transfer (`sendfile` Linux Syscall)
Conventional application servers transfer disk files over sockets via 4 memory-copy cycles (Disk -> OS Page Cache -> JVM Application Memory -> Socket Buffer -> Network Card).  
Kafka utilizes the Linux kernel **`sendfile()`** system call (Zero-Copy): data is transferred directly from the OS Page Cache into the Network Interface Card (NIC) buffer, completely bypassing Kafka's JVM memory footprint. This eliminates garbage collection pauses and conserves memory bus bandwidth.

### 3. Log Retention & Deterministic Replayability
Messages in Kafka are not purged upon consumption. They are retained according to time-based retention (e.g., 7 days) or size-based policies. If the Fraud Detection team discovers an algorithm bug introduced yesterday afternoon, they can simply rewind their consumer group offset back 24 hours and **replay the exact stream of production events** from historical logs!

## 4. How It Works (Topic, Partition, and Consumer Group Architecture)

```
TOPIC: "driver-locations" (Divided across 3 Partitions)

[ Producer: Driver Handsets ]
             |
     (Stream GPS Updates)
             |
             v
+-------------------------------------------------------------+
| KAFKA BROKER CLUSTER                                        |
|                                                             |
| Partition 0:  [Offset 0] [Offset 1] [Offset 2] [Offset 3]-> |
| Partition 1:  [Offset 0] [Offset 1] [Offset 2] -----------> |
| Partition 2:  [Offset 0] [Offset 1] [Offset 2] [Offset 3]-> |
+-------------------------------------------------------------+
             |                                   |
             v                                   v
[ Consumer Group: "fraud-service" ]   [ Consumer Group: "pricing-service" ]
- Consumer A reads Partition 0 & 1    - Consumer X reads Partition 0
- Consumer B reads Partition 2        - Consumer Y reads Partition 1
                                      - Consumer Z reads Partition 2
(Both consumer fleets ingest the exact same stream completely independently!)
```

## 5. Core Implementation Logic

### 1. Producer in Go with Partition Key

Menentukan `Key` memastikan bahwa seluruh event milik satu pengemudi selalu masuk ke partisi yang sama (*ordering guarantee per key*):

```go
package main

import (
	"context"
	"encoding/json"
	"time"

	"github.com/segmentio/kafka-go"
)

type LocationEvent struct {
	DriverID  string    `json:"driver_id"`
	Latitude  float64   `json:"latitude"`
	Longitude float64   `json:"longitude"`
	Timestamp time.Time `json:"timestamp"`
}

// EmitDriverLocation mengirimkan event GPS ke Kafka dengan Partition Key
func EmitDriverLocation(ctx context.Context, writer *kafka.Writer, driverID string, lat, lng float64) error {
	payload, _ := json.Marshal(LocationEvent{
		DriverID:  driverID,
		Latitude:  lat,
		Longitude: lng,
		Timestamp: time.Now(),
	})

	// Partition Key 'driverID' menjamin seluruh event driver ini masuk ke partisi yang sama
	return writer.WriteMessages(ctx, kafka.Message{
		Key:   []byte(driverID),
		Value: payload,
	})
}
```

### 2. Consumer Group in Go with Manual Offset Commit

```go
func processEvents(ctx context.Context, reader *kafka.Reader) {
    for {
        m, err := reader.FetchMessage(ctx)
        if err != nil {
            break
        }

        // 1. Eksekusi logika bisnis
        err = handleBusinessLogic(m.Value)
        if err != nil {
            log.Printf("Gagal memproses pesan: %v", err)
            continue
        }

        // 2. Commit offset manual HANYA setelah bisnis logika sukses (At-Least-Once Delivery)
        if err := reader.CommitMessages(ctx, m); err != nil {
            log.Printf("Gagal commit offset: %v", err)
        }
    }
}
```

## 6. Trade-offs (Pros and Cons)

### Pros
- **Enormous Throughput**: Sustains millions of events per second per cluster with single-digit millisecond latency.
- **True Architectural Decoupling**: Upstream event producers and downstream consumers are isolated in compute capacity; slow consumers never stall producers.
- **High Durability & Replayability**: Persistently committed to distributed disk storage with tunable replication factors.

### Cons
- **Operational Complexity**: Operating Kafka clusters in production demands specialized infrastructure skills (ZooKeeper vs KRaft metadata consensus, JVM garbage collection tuning, partition sizing, and disk I/O provisioning).
- **Partition-Bound Ordering**: Kafka strictly guarantees message ordering **within a single partition**, never globally across an entire topic.
- **Consumer Rebalance Latency**: Consumer group membership changes trigger partition rebalances that can temporarily stall stream processing.

## 7. When to Use vs When Not to Use

### When to Use Kafka
- Event-driven microservices architectures where multiple services ingest the same operational events.
- High-volume data ingestion pipelines (telemetry feeds, clickstream web metrics, centralized log pipelines).
- Audit logging, Change Data Capture (CDC pipelines via Debezium), and real-time stream analytics (Apache Flink / Spark Streaming).

### When NOT to Use Kafka
- Simple background job queues where each task is processed by a single worker and promptly deleted (use RabbitMQ, Redis Streams, or AWS SQS instead).
- Scenarios requiring complex dynamic routing logic based on dynamic message headers (standard AMQP features).
- Small development teams without dedicated DevOps/SRE capacity to manage distributed cluster infrastructure.

## 8. Key Metrics & Observability

Critical Prometheus / Grafana metrics for Kafka production monitoring:

- **`kafka_consumergroup_lag`**: Distance between the latest partition log-end offset and consumer committed offset; rising lag indicates overloaded or stalled consumers.
- **`UnderReplicatedPartitions` (URP)**: Number of partitions lacking required in-sync replicas (must strictly equal 0; positive numbers signal broker outages or storage failures).
- **`BytesInPerSec` & `BytesOutPerSec`**: Ingress and egress network bandwidth consumed across broker nodes.
- **`RequestHandlerAvgIdlePercent`**: Percentage of time broker I/O request handler threads are idle (below 20% indicates broker CPU exhaustion).

## 9. Real-World Case Studies

### High-Throughput Ingestion of 500,000 Events/Sec for Real-Time Financial Fraud Detection

- **The Engineering Problem**: An international digital bank processed hundreds of thousands of payment card authorizations, login telemetry signals, and merchant geolocations per second. The legacy architecture used traditional point-to-point queues (RabbitMQ). When traffic surged to 500,000 events/second during market open hours, broker RAM saturated, nodes crashed under memory pressure, and messages were dropped. Furthermore, multiple downstream systems (Fraud Detection, Audit Ledger, Real-Time Notifications, and the Big Data Lake) all required access to the exact same stream, forcing fragile fan-out replication that quadrupled network overhead.
- **The Architectural Solution**: The data engineering team deployed an enterprise *Apache Kafka* cluster (KRaft metadata mode) with partitioned event streams keyed by `account_id` to guarantee per-account message ordering. Producers enabled idempotent publishing (`idempotence=true`), Snappy compression, and `acks=all`. The Fraud Detection service ingested real-time streams with Kafka Streams in under 10 ms, while independent Consumer Groups for Audit Ledger and Data Lake replication (via Debezium CDC and cloud connectors) consumed from identical topic offsets independently without mutating the underlying 14-day persistent append-only logs.
- **The System Outcome**: Ingestion capacity scaled past 850,000 events/second with p99 end-to-end transport latency under 15 ms. Fraud evaluation latency dropped to under 40 ms from card swipe. Message loss during promotional surges dropped to 0%, and broker memory exhaustion incidents were eliminated by relying on sequential, zero-copy kernel disk writes rather than RAM-bound queues.

## 10. Interesting Facts & Production Gotchas

- **The Consumer Rebalance Storm**:  
  If a consumer instance takes longer than `max.poll.interval.ms` (default: 5 minutes) to process a batch of records, the Kafka Group Coordinator assumes the consumer has died! It forcibly revokes the consumer's partition assignments and triggers a **Consumer Group Rebalance**. During a rebalance, all consumers in the group pause processing. If subsequent consumers also exceed processing thresholds, the cluster spirals into an *infinite rebalance freeze loop*!  
  *Production Fix:* Reduce batch processing sizes (`max.poll.records`), offload expensive computations to background worker pools, or increase `max.poll.interval.ms`.
- **Partition Count is a One-Way Street**:  
  You can dynamically scale out by adding partitions to a Kafka topic, but **you CANNOT reduce partition counts** once created without deleting and recreating the entire topic! Furthermore, altering partition counts changes the hashing distribution (`hash(key) % partition_count`), breaking ordering guarantees for subsequent records of existing keys.  
  *Production Golden Rule:* Size partition counts thoughtfully upfront based on target peak throughput and consumer concurrency limits.

## 11. Key Takeaways

- **Distributed Append-Only Commit Log**: Kafka organizes data as an immutable, persistent disk log; consumers track their own offset positions, enabling deterministic event replayability.
- **Horizontal Scalability via Partitions**: Topics scale by sharding across partitions; strict message ordering is guaranteed strictly within individual partitions keyed consistently.
- **Independent Competing Consumer Groups**: Multiple consumer groups subscribe to identical topics without mutual interference or message deletion, freeing downstream systems to process at independent speeds.
- **Mitigate Consumer Rebalance Freezes**: Keep consumer batch execution times strictly within `max.poll.interval.ms` to prevent coordinator heartbeats from triggering catastrophic cluster-wide partition rebalances.
