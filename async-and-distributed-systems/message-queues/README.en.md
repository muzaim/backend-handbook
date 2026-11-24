# Message Queues

## 1. Real-World System Scenario
On high-volume media streaming platforms, users upload 1.5 GB raw video recordings through the backend ingestion API (`POST /api/v1/videos/upload`).
On high-volume media streaming platforms, users upload 1.5 GB raw video recordings through the backend ingestion API (`POST /api/v1/videos/upload`).

Once the file payload is staged, the backend must process the video: transcode and compress it into 4 different streaming resolutions (1080p, 720p, 480p, 360p) and generate 10 preview thumbnails. This heavy CPU/GPU computation takes approximately 3 to 5 minutes to complete.

If this intensive computation executes synchronously within the incoming HTTP request lifecycle:
- The web server holds open the client's HTTP socket connection for a full 5 minutes.
- Mobile and web clients encounter gateway timeouts, and server worker threads remain tied up waiting on long-running encoding binaries.
- The web API quickly exhausts its available thread pool, collapsing under incoming traffic (*thread starvation*).

**The Message Queue Solution (e.g., RabbitMQ / AWS SQS / Celery):**
When the raw video upload finishes writing to cloud object storage (S3), the API server merely publishes a lightweight job envelope (`{"job_id": 901, "video_id": "vid-882", "s3_url": "..."}`) into a Message Queue, and immediately returns `HTTP 202 Accepted: Video Processing` to the client in under 150 milliseconds. The HTTP server threads are instantly freed to handle subsequent traffic. Behind the scenes, an isolated fleet of background *Worker Consumers* continuously pulls tasks from the queue and encodes videos asynchronously at a controlled, sustainable pace.

## 2. What Is It and What Is It For?

A Message Queue is an architectural infrastructure component that provides temporary, ordered (typically *FIFO - First-In, First-Out*) message storage to enable asynchronous, decoupled communication between discrete software services.

Its fundamental topology is the **Point-to-Point (Competing Consumers)** pattern:
- One or more Producers enqueue work payloads (*messages or jobs*).
- Any single message placed into the queue is **delivered to and processed by EXACTLY ONE consumer instance**.
- Upon finishing the task, the consumer sends an acknowledgment signal (**ACK**), permanently removing the message. If the consumer crashes midway, the queue returns the message to the backlog for another worker to pick up.

Popular technologies: RabbitMQ, AWS SQS, Apache ActiveMQ, Redis Streams/BullMQ, Celery.

## 3. Why Use It in High-Throughput Systems?

### 1. Traffic Smoothing (*Peak Shaving / Load Leveling*)
Consider an e-commerce platform slammed by 50,000 checkout submissions per second during a midnight flash sale, while downstream inventory and banking payment connections can only absorb 2,000 writes per second.
- **Without a Message Queue**: 48,000 requests encounter timeouts, database CPU reaches 100%, connections pool out, and orders fail en masse.
- **With a Message Queue**: 50,000 order intents are safely enqueued in memory/disk in 5 milliseconds. A pool of background workers processes them continuously at a stable 2,000 writes per second until the queue drains. Zero downtime.

### 2. Decoupling Latency-Heavy Workloads
When a user uploads a video file, the HTTP API should never hang a client socket for 10 minutes while encoding 1080p, 720p, and 480p streams. The API quickly enqueues a transcoding job, returns `HTTP 202 Accepted: Video Processing`, and frees web server threads instantly.

## 4. How It Works (Data Flow Diagram)

```
+---------------+
|  Web API Svc  | (Producer)
+---------------+
        |
        | 1. Enqueue Job: { order_id: 101, user: "andi" }
        v
+-------------------------------------------------------+
|                     Message Queue                     |
|                                                       |
|  [ Job 103 ] -> [ Job 102 ] -> [ Job 101 ]            |
+-------------------------------------------------------+
       |                           |
       | 2a. Claim Job 101         | 2b. Claim Job 102
       v                           v
+---------------+           +---------------+
|   Worker 1    |           |   Worker 2    | (Consumer Pool)
| (Processing)  |           | (Processing)  |
+---------------+           +---------------+
       |                           |
       | 3a. ACK (Complete!)       | 3b. ACK (Complete!)
       v                           v
   (Delete Job 101)            (Delete Job 102)
```

## 5. Core Implementation Logic (Consumer Loop & DLQ di Go)

Implementation of the consumer worker loop with ACK and DLQ in Go dengan penanganan ACK, retry bertahap, dan pemindahan ke Dead Letter Queue (DLQ) di Go:

```go
package main

import (
	"context"
	"log"
	"time"
)

type Message struct {
	ID        string
	Payload   []byte
	Attempt   int
}

// StartQueueWorker menjalankan loop konsumsi pesan dengan retry dan DLQ
func StartQueueWorker(ctx context.Context, queue <-chan Message, dlq chan<- Message) {
	for {
		select {
		case msg := <-queue:
			err := processVideoEncoding(msg.Payload)
			if err != nil {
				msg.Attempt++
				log.Printf("Gagal memproses pesan %s (percobaan ke-%d): %v", msg.ID, msg.Attempt, err)

				if msg.Attempt >= 3 {
					log.Printf("Pesan %s dialihkan ke Dead Letter Queue (DLQ)", msg.ID)
					dlq <- msg
				} else {
					// Masukkan kembali ke antrean utama untuk dicoba lagi
					go func(m Message) {
						time.Sleep(2 * time.Second)
						queue <- m
					}(msg)
				}
				continue
			}

			// Sukses diproses (ACK)
			log.Printf("Pesan %s berhasil diproses", msg.ID)

		case <-ctx.Done():
			return
		}
	}
}
```

## 6. Pros and Cons (Trade-offs)

### Pros
- **Fault Isolation & Resilience**: If all downstream worker instances crash simultaneously, unconsumed messages remain safely persisted on disk. When workers reboot, consumption resumes with zero data loss.
- **Elastic Horizontal Scalability**: Scale worker fleets dynamically from 2 to 100 instances during backlog surges without touching API gateways or producers.
- **Domain Decoupling**: Ingestion APIs require zero operational knowledge of downstream database architectures or programming runtimes.

### Cons
- **Eventual Consistency Latency**: Mutation effects are not instant. End-users must tolerate an asynchronous state transition (e.g., waiting a few seconds for order status to flip to "Confirmed").
- **Infrastructure Operational Burden**: Requires provisioning, clustering, disk monitoring, and maintenance of dedicated message broker clusters.
- **Distributed Observability Complexity**: Tracing request lineages across detached asynchronous boundaries requires distributed tracing tooling (OpenTelemetry, trace IDs).

## 7. When to Use vs When Not to Use

### When to Use
- Resource-intensive background workloads: Bulk email dispatches, push notification fan-outs, PDF generation, media transcoding, AI batch inferences.
- Buffering high-volume checkout transaction spikes (*protective shock absorbers*).
- Asynchronous service-to-service communication where callers do not require synchronous returns.

### When Not to Use
- Strictly synchronous transactions requiring immediate interactive answers (e.g., password credential validation; users cannot wait 60 seconds to find out their password was mistyped).
- Low-volume applications where a simple database status column (`status: PENDING`) with periodic polling is sufficient (*avoid accidental infrastructure bloat*).

## 8. Key Metrics & Observability

Essential telemetry metrics to track message queue health on Grafana or Datadog dashboards:

- **`queue_messages_visible` / `queue_depth`**: Count of messages currently queued and waiting for worker dispatch.
- **`queue_consumer_lag_seconds`**: Elapsed time between message ingestion and consumer execution start.
- **`queue_messages_unacknowledged`**: Number of messages currently in flight with workers awaiting ACK/NACK.
- **`dlq_messages_total`**: Count of failed messages directed to the Dead Letter Queue, alerting on bad payloads or downstream outages.
- **`consumer_throughput_messages_per_sec`**: Rate of completed consumer tasks compared to producer ingest rate.

## 9. Real-World Case Studies

- **DoorDash Delivery Dispatching**: DoorDash uses RabbitMQ and Celery to manage thousands of incoming restaurant orders per second. When an order is placed, it is enqueued into dispatch queues. Dedicated worker pools compute optimal courier routes and delivery ETAs asynchronously without degrading customer mobile app responsiveness.
- **Uber Driver Webhook Ingestion**: Uber leverages AWS SQS queues to absorb millions of driver location updates and ride event webhooks. The queue buffers rush hour burst traffic, decoupling mobile ingestion from heavy database persistence routines.

## 10. Interesting Facts & Production Gotchas

- **The Poison Pill Nightmare**: What happens if an unhandled edge case or corrupted payload enters the queue? Worker 1 reads the message, panics, and crashes ungracefully. The broker notices the dead consumer and re-delivers the message. Worker 2 claims it and instantly crashes too! This cascade continues until every worker in your fleet is dead.  
  *Mandatory Solution:* Implement a **DLQ (Dead-Letter Queue)**. If a message fails processing consecutively across a retry threshold (e.g., 3 attempts), the broker automatically quarantines the poisonous payload into a DLQ for developer post-mortem inspection.
- **Monitoring Consumer Lag**: The most crucial telemetry metric for message queue health is **Consumer Lag** (the delta between message production rate and message consumption rate). If consumer lag spikes upward continuously, it is an urgent alert that your worker pool is bottlenecked and requires auto-scaling.
