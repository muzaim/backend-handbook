# Prometheus

## 1. Real-World System Scenario

In a microservices-based e-commerce platform processing 80,000 requests per second across 150 Kubernetes pods, the `Checkout Service` abruptly suffers severe performance degradation: average latency surges from 50 ms to 2.8 seconds, and transactions begin failing with `504 Gateway Timeout` errors.

If the architecture lacks a centralized time-series metrics collection system:
- Software engineers are forced to run `kubectl logs` across 150 individual pod terminal sessions, frantically grepping for error patterns among millions of unstructured log lines streaming every second.
- Searching through text logs takes tens of minutes, while each minute of checkout downtime bleeds tens of thousands of dollars in lost revenue.
- Leadership and on-call teams cannot determine whether the root cause is database connection pool exhaustion, CPU throttling, network socket saturation, or upstream third-party gateway failures.

**Solution with Prometheus:**
Prometheus acts as an ultra-fast *Time Series Database (TSDB)* and automated *metrics scraper*. Every backend pod exposes an HTTP `/metrics` endpoint containing transaction counters, latency histograms, and active gauge measurements. Every 15 seconds, the central Prometheus server scrapes metrics from all 150 pods. Within seconds of the initial latency spike, Prometheus evaluates its PromQL alert rules (`histogram_quantile(0.99, ...)`), detects that P99 latency breached the 500 ms SLA threshold, and fires an automated alert to on-call engineers alongside root-cause graphs!

## 2. What Is It and What Is It For?

**Prometheus** is an open-source systems monitoring and alerting platform originally developed at SoundCloud and currently maintained as a graduated flagship project by the Cloud Native Computing Foundation (CNCF).

### Core Characteristics:
1. **Multi-Dimensional Time-Series Data Model**: Every metric is identified by a metric name and key-value label pairs (`http_requests_total{method="POST", handler="/checkout", status="500"}`).
2. **PromQL (Prometheus Query Language)**: An expressive, purpose-built query language for real-time aggregations, quantile calculations (P50, P90, P99), per-second rate derivates (`rate()`), and alert triggers.
3. **Pull-Based Metrics Ingestion**: Unlike traditional push systems, Prometheus actively scrapes metrics from instrumented application targets over HTTP GET `/metrics` on configured scrape intervals.
4. **Dynamic Service Discovery**: Integrates natively with the Kubernetes API, AWS EC2, Consul, and DNS to automatically discover new containers and pods as they scale up.

## 3. Why Use It in High-Throughput Systems?

### 1. Pull Model Shields the Monitoring Infrastructure
In traditional push-based monitoring (like StatsD or Graphite), sudden traffic spikes to 500,000 RPS cause hundreds of application pods to bombard the monitoring server with hundreds of thousands of UDP/TCP packets per second. The monitoring collector crashes precisely when needed most.  
With Prometheus's **Pull Architecture**, ingestion load is deterministic and rate-limited by the Prometheus server itself (e.g., pulling every 15 seconds), regardless of incoming user traffic volume.

### 2. High-Density Compression (Gorilla Algorithm)
Prometheus implements Facebook's *Gorilla* time-series compression algorithm in memory and disk storage. Each raw data sample (8-byte timestamp + 8-byte float value = 16 bytes) is compressed down to an average of **1.37 bytes per sample**! A single Prometheus instance can store billions of data points daily on modest disk allocations.

### 3. Precise SLA/SLO Enforcement via Histograms
Averages often mask severe latency anomalies (an average response time of 60 ms can hide 1% of users suffering 10-second timeouts). Prometheus **Histograms** enable mathematical calculation of true percentiles (P95, P99) across distributed clusters using PromQL.

## 4. How It Works (Prometheus Pull & Alert Pipeline)

```
+--------------------------------------------------------------+
| KUBERNETES CLUSTER / VIRTUAL MACHINES                        |
|                                                              |
| [ Pod App 1 ] ----> Exposes: GET /metrics (Plaintext Format) |
| [ Pod App 2 ] ----> Exposes: GET /metrics                    |
| [ Pod App 3 ] ----> Exposes: GET /metrics                    |
+--------------------------------------------------------------+
          ^
          | (Periodic Pull: Scrape Interval 15s)
          |
+--------------------------------------------------------------+
| PROMETHEUS SERVER                                            |
|                                                              |
| 1. Service Discovery: Discovers new pods via K8s API         |
| 2. Retrieval Engine: Scrapes HTTP metrics from pods          |
| 3. TSDB (Time Series DB): Writes to RAM & Disk (Gorilla)     |
| 4. PromQL Engine: Evaluates Dashboards & Alert Rules         |
+--------------------------------------------------------------+
          |                                  |
          v (PromQL Queries)                 v (Dispatches Alert Triggers)
+-------------------+              +-------------------+
|  Grafana Dashboard|              |   Alertmanager    |
|  (Visualization)  |              |  (PagerDuty/Slack)|
+-------------------+              +-------------------+
```

## 5. Core Implementation Logic

### 1. Metric Instrumentation in Go (`client_golang`)

```go
package main

import (
	"net/http"
	"strconv"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	// 1. Counter: Monotonically increasing counter for total HTTP requests
	httpRequestsTotal = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "http_requests_total",
			Help: "Total count of processed HTTP requests",
		},
		[]string{"method", "endpoint", "status"},
	)

	// 2. Histogram: Measures request latency distribution
	httpRequestDuration = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "http_request_duration_seconds",
			Help:    "HTTP request processing duration in seconds",
			Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5},
		},
		[]string{"method", "endpoint"},
	)
)

func init() {
	prometheus.MustRegister(httpRequestsTotal)
	prometheus.MustRegister(httpRequestDuration)
}

// MiddlewarePrometheus instruments latency and status codes automatically
func MiddlewarePrometheus(endpoint string, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rw := &responseWriterInterceptor{ResponseWriter: w, statusCode: http.StatusOK}

		next(rw, r)

		duration := time.Since(start).Seconds()
		httpRequestsTotal.WithLabelValues(r.Method, endpoint, strconv.Itoa(rw.statusCode)).Inc()
		httpRequestDuration.WithLabelValues(r.Method, endpoint).Observe(duration)
	}
}

func main() {
	http.HandleFunc("/api/v1/checkout", MiddlewarePrometheus("/checkout", handleCheckout))
	// Standard Prometheus scraping endpoint
	http.Handle("/metrics", promhttp.Handler())
	_ = http.ListenAndServe(":8080", nil)
}
```

### 2. Prometheus Scrape Configuration (`prometheus.yml`)

```yaml
global:
  scrape_interval: 15s     # Scrape metrics every 15 seconds
  evaluation_interval: 15s # Evaluate alert rules every 15 seconds

scrape_configs:
  - job_name: "ecommerce-backend"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_container_port_number]
        action: keep
        regex: "8080"
```

## 6. Trade-offs (Pros and Cons)

### Pros
- **High Storage Efficiency**: Ingests millions of samples per second on a single node without external database clustering.
- **Expressive PromQL Engine**: Supports real-time percentile slicing, moving averages, and growth rate regressions.
- **Cloud-Native De-Facto Standard**: Ubiquitous integration across open-source infrastructure (Kubernetes, NGINX, Redis, Kafka, PostgreSQL).

### Cons
- **Not Designed for Long-Term Archival**: Prometheus local TSDB is optimized for short-term operational retention (15-30 days). Multi-year retention requires remote storage layers like Thanos, Cortex, or VictoriaMetrics.
- **High Cardinality Sensitivity**: Unbounded label dimensions (e.g., `user_id`, `order_id`, or `email`) spawn millions of concurrent time-series in memory, triggering fatal Out-Of-Memory (OOM) crashes.
- **No Log or Trace Storage**: Strictly handles aggregate numeric time-series, not discrete event payloads or distributed call stacks.

## 7. When to Use vs When Not to Use

### When to Use Prometheus
- Monitoring server resource saturation (CPU, RAM, Disk I/O) and application Golden Signals (RPS, P99 Latency, 5xx Error Rates).
- Containerized Kubernetes environments requiring dynamic pod auto-discovery.
- Alert evaluation and dispatching to on-call teams (PagerDuty, Slack).

### When NOT to Use Prometheus
- Text log aggregation or financial audit logs (use Elasticsearch, Loki, or ClickHouse).
- Distributed tracing across microservice call chains (use OpenTelemetry and Jaeger).
- High-cardinality metadata querying (e.g., tracking metrics per individual customer UUID or IP address).

## 8. Key Metrics & Observability

Internal telemetry for monitoring Prometheus server health (*self-monitoring*):

- **`prometheus_tsdb_head_series`**: Total active time-series held in memory in the TSDB head block (critical for detecting cardinality explosions).
- **`prometheus_target_scrape_pool_targets`**: Number of discovered application endpoints currently scraped.
- **`prometheus_target_scrapes_sample_out_of_order_total`**: Count of samples rejected due to out-of-order timestamps.
- **`prometheus_engine_query_duration_seconds`**: Latency required by PromQL to evaluate dashboard panels and alert rules.

## 9. Real-World Case Studies

### Real-Time Performance Degradation Detection and Multi-Cluster SLO Alerting

- **The Engineering Problem**: A digital bank operated 140 microservices across several Kubernetes clusters. System health originally relied on centralized log tailing and basic blackbox pings checking `/healthz` endpoints every minute. During a major incident, a database deadlock caused Payment Gateway p99 latencies to degrade from 150 ms to 7,800 ms, yet the shallow `/healthz` probe continued returning `200 OK`. For 40 minutes, thousands of customer checkouts silently failed with zero automated alerts until angry users flooded customer support lines.
- **The Architectural Solution**: The SRE team deployed *Prometheus* using the Prometheus Operator across clusters. Every containerized microservice exposed standardized *Golden Signals* (Latency Histograms, Traffic RPS, 5xx Error Rates, and Resource Saturation). Prometheus scraped endpoints every 15 seconds via dynamic Kubernetes pod annotations. The team engineered PromQL alerting rules tracking *Error Budget Burn Rates* and p99 latency regressions (`histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 2`). Alerts routed through Alertmanager with multi-channel escalation policies.
- **The System Outcome**: Mean Time to Detect (MTTD) degraded system states plummeted from 40 minutes to under 45 seconds. Engineering teams intercepted transient latency regressions before they cascaded into customer-impacting outages. Customer-reported payment incident tickets dropped by 92%, and platform observability coverage reached 100% across all microservices.

## 10. Interesting Facts & Production Gotchas

- **The High Cardinality Explosion (The #1 Prometheus Killer)**:  
  A common mistake by new engineers is injecting unique identifiers into metric labels, e.g.: `http_requests_total{user_id="123456", order_id="ABC-789"}`. With 1 million unique users, Prometheus creates **1 million discrete time-series in RAM**! The Prometheus process will crash with an immediate `OOMKilled` termination.  
  *Production Golden Rule:* Labels must have bounded, discrete values (*low cardinality*, e.g.: `method="GET"`, `status="200"`, `region="us-east-1"`). Never inject unique UUIDs into metric labels!
- **PromQL `rate()` vs `irate()` Trap**:  
  `rate()` calculates the per-second average rate of increase across a specified time window (e.g., `rate(http_requests_total[5m])`), whereas `irate()` evaluates only the last two data points. Using `irate()` for Alerting rules produces wildly jagged spikes, triggering false alarms (*alert fatigue*) in the middle of the night!  
  *Production Fix:* Always use `rate()` for alert rule evaluation and SLO tracking.

## 11. Key Takeaways

- **Pull-Based Metrics Collection**: Prometheus scrapes numerical telemetry from lightweight `/metrics` HTTP endpoints on an interval, discovering ephemeral container endpoints dynamically via Kubernetes SD.
- **Powerful PromQL Analytics**: Leverage PromQL for real-time statistical aggregation (per-second rates, p95/p99 histogram quantiles, and error ratios) to evaluate strict Service Level Objectives (SLOs).
- **Prevent High-Cardinality Disasters**: Never place unbounded, unique identifiers (e.g., `user_id`, `order_id`, UUIDs) in metric labels; uncontrolled label cardinality exhausts Prometheus RAM, causing fatal OOM crashes.
- **Favor `rate()` Over `irate()` for Alerts**: Utilize `rate()` over smoothed evaluation windows (e.g., 5m) for robust alerting and SLO enforcement; reserve volatile `irate()` for short-term instant visualization to prevent alert fatigue.
