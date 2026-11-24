# Prometheus

## 1. Skenario Sistem Nyata

Pada kluster microservices e-commerce yang melayani 80.000 request per detik di 150 pod Kubernetes, `Checkout Service` tiba-tiba mengalami degradasi performa: latensi rata-rata melonjak dari 50 ms menjadi 2.8 detik, dan sebagian transaksi mulai gagal dengan status `504 Gateway Timeout`.

Jika sistem tidak memiliki pengumpulan metrik berbasis *time-series*:
- Engineer harus membuka satu per satu terminal pod menggunakan `kubectl logs` di 150 pod berbeda, mencoba mencari pola error di antara jutaan baris teks log yang lewat setiap detiknya.
- Sifat log teks yang sangat besar membuat pencarian (*grep*) membutuhkan waktu puluhan menit, sementara setiap detik keterlambatan menyebabkan kerugian finansial puluhan juta rupiah.
- Manajemen tidak tahu apakah masalahnya ada di kehabisan database connection pool, CPU throttling, saturasi antrean request, atau error rate dari downstream service eksternal.

**Solusi dengan Prometheus:**
Prometheus bertindak sebagai *Time Series Database (TSDB)* dan *metrics scraper*. Setiap pod aplikasi mengekspos endpoint HTTP `/metrics` berisi counter transaksi, histogram latensi, dan gauge koneksi aktif. Setiap 15 detik, server Prometheus secara proaktif menarik (*pull*) metrik dari seluruh 150 pod. Hanya dalam hitungan detik setelah lonjakan latensi terjadi, Prometheus mengevaluasi aturan alert PromQL (`histogram_quantile(0.99, ...)`), mendeteksi bahwa P99 latency melampaui SLA 500 ms, dan seketika menembakkan alert ke tim on-call beserta grafik akar penyebabnya!

## 2. Apa Itu dan Buat Apa?

**Prometheus** adalah platform pemantauan (*monitoring*) dan sistem peringatan (*alerting*) sumber terbuka yang awalnya dikembangkan di SoundCloud dan kini menjadi proyek lulusan utama (*graduated project*) di bawah naungan Cloud Native Computing Foundation (CNCF).

### Karakteristik Kunci:
1. **Model Data Time-Series Multidimensi**: Setiap metrik diidentifikasi oleh nama metrik dan pasangan label key-value (`http_requests_total{method="POST", handler="/checkout", status="500"}`).
2. **Bahasa Query PromQL (Prometheus Query Language)**: Bahasa query ekspresif yang sangat kuat untuk mengagregasi, menghitung persentil (P50, P90, P99), menghitung tingkat laju perubahan per detik (`rate()`), dan memicu alert.
3. **Model Pengumpulan Data Pull-Based (Scraping)**: Berbeda dengan sistem push tradisional, server Prometheus secara proaktif menarik metrik dari target aplikasi melalui HTTP GET `/metrics` secara periodik (*scrape interval*).
4. **Service Discovery Dinamis**: Terintegrasi secara native dengan Kubernetes API, AWS EC2, Consul, atau DNS untuk secara otomatis menemukan pod baru yang baru dibuat (*auto-discovery*).

## 3. Kenapa Harus Pakai Ini di High-Throughput?

### 1. Desain Pull-Model Mencegah Server Monitoring Jebol
Pada sistem *push-model* tradisional (seperti Graphite atau StatsD), ketika terjadi lonjakan trafik 500.000 RPS, setiap pod aplikasi akan membombardir server monitoring dengan ratusan ribu paket UDP/TCP per detik. Server monitoring akan kehabisan memori dan tumbang tepat di saat sistem paling membutuhkannya.  
Dengan **Pull-Model** Prometheus, beban ingestion data stabil dan dapat diprediksi: server Prometheus mengatur ritme penarikannya sendiri (misal setiap 15 detik), berapa pun lonjakan RPS yang sedang dialami aplikasi.

### 2. Kompresi Time-Series Ekstrem (Gorilla Compression)
Prometheus menggunakan algoritma kompresi memori dan disk berbasis *Facebook Gorilla algorithm*. Setiap sampel data titik waktu (timestamp 8-byte + float value 8-byte = 16 bytes) dikompresi menjadi rata-rata hanya **1.37 bytes per sampel** di disk! Ini memungkinkan satu instance Prometheus menyimpan miliaran titik metrik per hari dengan konsumsi disk dan RAM yang sangat efisien.

### 3. Evaluasi SLA/SLO yang Akurat via Histogram
Rata-rata (*Average Latency*) sering menipu engineer (misal rata-rata 60 ms bisa menyembunyikan 1% user yang menunggu 10 detik). Prometheus menyediakan tipe metrik **Histogram** yang memungkinkan kalkulasi matematis persentil sejati (P95, P99) di tingkat kluster menggunakan PromQL.

## 4. Cara Kerjanya (Arsitektur Prometheus Pull & Alert Pipeline)

```
+--------------------------------------------------------------+
| KUBERNETES CLUSTER / VIRTUAL MACHINES                        |
|                                                              |
| [ Pod App 1 ] ----> Ekspos: GET /metrics (Format Teks)       |
| [ Pod App 2 ] ----> Ekspos: GET /metrics                     |
| [ Pod App 3 ] ----> Ekspos: GET /metrics                     |
+--------------------------------------------------------------+
          ^
          | (Tarik Data Periodik: Scrape Interval 15 detik)
          |
+--------------------------------------------------------------+
| PROMETHEUS SERVER                                            |
|                                                              |
| 1. Service Discovery: Deteksi Pod baru via K8s API           |
| 2. Retrieval Engine: Scrape HTTP data dari Pod               |
| 3. TSDB (Time Series DB): Simpan ke RAM & Disk (Gorilla)     |
| 4. PromQL Engine: Evaluasi Query & Alert Rules               |
+--------------------------------------------------------------+
          |                                  |
          v (Query PromQL)                   v (Kirim Alert Trigger)
+-------------------+              +-------------------+
|  Grafana Dashboard|              |   Alertmanager    |
|  (Visualisasi)    |              |  (PagerDuty/Slack)|
+-------------------+              +-------------------+
```

## 5. Cuplikan Logika Inti

### 1. Instrumentasi Metrik di Aplikasi Go (`client_golang`)

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
	// 1. Counter: Menghitung total request masuk (monotonik naik)
	httpRequestsTotal = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "http_requests_total",
			Help: "Total jumlah HTTP request yang diproses",
		},
		[]string{"method", "endpoint", "status"},
	)

	// 2. Histogram: Mengukur distribusi durasi latensi request
	httpRequestDuration = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "http_request_duration_seconds",
			Help:    "Durasi pemrosesan HTTP request dalam detik",
			Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5},
		},
		[]string{"method", "endpoint"},
	)
)

func init() {
	prometheus.MustRegister(httpRequestsTotal)
	prometheus.MustRegister(httpRequestDuration)
}

// MiddlewarePrometheus mencatat latensi dan status code secara otomatis
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
	// Endpoint resmi scraping Prometheus
	http.Handle("/metrics", promhttp.Handler())
	_ = http.ListenAndServe(":8080", nil)
}
```

### 2. Konfigurasi Scrape Prometheus (`prometheus.yml`)

```yaml
global:
  scrape_interval: 15s     # Tarik metrik setiap 15 detik
  evaluation_interval: 15s # Evaluasi alert rules setiap 15 detik

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

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Efisiensi Penyimpanan Sangat Tinggi**: Mampu memproses jutaan sampel metrik per detik pada satu server tanpa cluster database eksternal.
- **Bahasa Query PromQL Sangat Ekspresif**: Memungkinkan kalkulasi kuantil matematis, turunan derivatif, dan regresi tren trafik secara real-time.
- **Standar Terbuka Ekosistem Cloud-Native**: Hampir semua tool open-source (Kubernetes, NGINX, Redis, Kafka, PostgreSQL) memiliki exporter resmi Prometheus.

### Kekurangan
- **Bukan untuk Penyimpanan Jangka Panjang (Long-Term Storage)**: TSDB lokal Prometheus dirancang untuk retensi data jangka pendek (15-30 hari). Untuk retensi bulanan/tahunan, membutuhkan integrasi remote storage seperti Thanos, Cortex, atau VictoriaMetrics.
- **Masalah High Cardinality**: Menambahkan label dengan nilai unik tak terbatas (seperti `user_id`, `order_id`, atau `email`) akan meledakkan jumlah time-series di RAM dan membuat Prometheus crash (*OOM Crash*).
- **Tidak Memantau Log atau Traces**: Prometheus murni dirancang untuk metrik numerik agregat, bukan rekaman event individual atau pelacakan alur stack trace.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Pakai Prometheus
- Memantau kesehatan sistem, resource hardware (CPU, Memory, Disk I/O), dan performa aplikasi (RPS, Latency P99, Error Rate 5xx).
- Sistem berbasis kontainer dan Kubernetes yang membutuhkan auto-discovery pod dinamis.
- Evaluasi Service Level Objectives (SLO) dan penembakan alert real-time ke tim on-call (PagerDuty/Slack).

### Kapan Jangan Pakai Prometheus
- Pencatatan log pesan teks individual atau audit trail keuangan (gunakan Elasticsearch, Loki, atau ClickHouse).
- Kebutuhan tracing alur pemanggilan fungsi antar-service (gunakan OpenTelemetry dan Jaeger).
- Data metrik dengan dimensi ber-kardinalitas tinggi (misal metrik dengan label IP pengguna individual atau UUID unik per transaksi).

## 8. Metrik Kunci & Observabilitas

Metrik internal untuk memantau kesehatan server Prometheus itu sendiri (*self-monitoring*):

- **`prometheus_tsdb_head_series`**: Jumlah total time-series aktif yang sedang disimpan di memori RAM kepala TSDB (waspadai ledakan kardinalitas).
- **`prometheus_target_scrape_pool_targets`**: Jumlah endpoint aplikasi yang berhasil ditemukan dan sedang aktif di-scrape.
- **`prometheus_target_scrapes_sample_out_of_order_total`**: Jumlah data sampel yang ditolak karena timestamp tidak berurutan.
- **`prometheus_engine_query_duration_seconds`**: Waktu yang dibutuhkan engine PromQL untuk menyelesaikan evaluasi query dashboard atau alert.

## 9. Studi Kasus Nyata di Industri

- **SoundCloud (Tempat Lahirnya Prometheus)**: SoundCloud menciptakan Prometheus untuk menggantikan sistem monitoring monolitik lama yang sering tumbang saat trafik streaming lagu memuncak. Dengan model pull dan PromQL, SoundCloud mampu memantau ribuan microservices secara presisi.
- **Kubernetes Core Monitoring**: Seluruh arsitektur internal Kubernetes (API Server, Kubelet, CoreDNS, etcd) mengekspos metrik secara native dalam format Prometheus.

## 10. Fakta Menarik & Gotchas Production

- **The High Cardinality Explosion (Penyebab #1 Prometheus Down)**:  
  Kesalahan paling mematikan developer pemula adalah menempelkan ID unik sebagai label metrik, misalnya: `http_requests_total{user_id="123456", order_id="ABC-789"}`. Jika ada 1 juta pengguna, Prometheus akan membuat **1 juta entri time-series baru di memori RAM**! Server Prometheus akan langsung kehabisan memori dan mati seketika (*OOMKilled*).  
  *Aturan Wajib:* Label metrik hanya boleh bernilai diskret terbatas (*low cardinality*, misal: `method="GET"`, `status="200"`, `region="ap-southeast-3"`). Jangan pernah memasukkan ID unik ke dalam label Prometheus!
- **PromQL `rate()` vs `irate()` Trap**:  
  Fungsi `rate()` menghitung rata-rata kenaikan per detik dalam jendela waktu tertentu (misal `rate(http_requests_total[5m])`), sedangkan `irate()` hanya menghitung selisih antara 2 titik data terakhir. Menggunakan `irate()` untuk membuat aturan Alerting adalah kesalahan fatal karena grafik akan bergerigi liar (*noisy spikes*) dan memicu alarm palsu (*false alert fatigue*) di tengah malam!  
  *Solusi Wajib:* Selalu gunakan `rate()` untuk evaluasi alert rules dan SLA.
