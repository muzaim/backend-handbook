# Grafana

## 1. Skenario Sistem Nyata

Pada saat event promo gajian (_Payday Sale_) dengan volume 100.000 pengguna aktif per detik, tim backend engineer, SRE (_Site Reliability Engineer_), dan pimpinan engineering harus memantau kesehatan sistem secara _real-time_. Di dalam sistem ini ada 20 microservices yang menggunakan database PostgreSQL, cache Redis, message broker Kafka, dan cluster Kubernetes di multi-region.

Jika pemantauan dilakukan secara terpisah tanpa satu panel dashboard terpadu:

-   Tim database harus membuka CLI terminal untuk melihat koneksi Postgres, tim infra membuka Kubernetes dashboard, tim aplikasi mengecek terminal logs, dan tim networking mengecek AWS CloudWatch.
-   Ketika ada insiden penurunan transaksi pembayaran, butuh waktu 30 menit hanya untuk mengoordinasikan tim dan mencari tahu di layer mana masalah bermula (_siloed monitoring blind spots_).
-   Manajemen bisnis tidak memiliki visibilitas langsung mengenai performa sistem terhadap metrik bisnis (seperti total GMV transaksi per menit atau tingkat keberhasilan checkout).

**Solusi dengan Grafana:**
Grafana bertindak sebagai lapisan visualisasi dan analisis terpadu (_unified observability frontend_). Dalam satu layar dashboard _Single Pane of Glass_, Grafana mengagregasi metrik dari Prometheus, log dari Loki, dan distributed trace dari Jaeger/Tempo. Hanya dengan sekali lirik di layar monitoring Command Center, engineer langsung melihat korelasi visual: lonjakan latensi P99 di `Order Service` bertepatan dengan kenaikan _Active Connection_ di PostgreSQL dan kenaikan _Consumer Lag_ di Kafka, memangkas waktu investigasi akar masalah (_MTTR - Mean Time to Resolution_) dari 30 menit menjadi hanya 2 menit!

## 2. Apa Itu dan Buat Apa?

**Grafana** adalah platform visualisasi data, analisis interaktif, dan pembuatan dashboard operasional sumber terbuka yang paling populer di dunia. Grafana mengubah data numerik time-series, log, dan trace dari ratusan sumber data (_datasources_) menjadi grafik, bagan, peta panas (_heatmaps_), dan panel metrik yang mudah dipahami.

### Kemampuan Kunci Grafana:

1. **Multi-Datasource Agnostic**: Menggabungkan data dari berbagai sumber dalam satu panel dashboard yang sama (Prometheus, PostgreSQL, MySQL, Elasticsearch, InfluxDB, CloudWatch, Datadog).
2. **Visualisasi Interaktif & Variabel Dinamis**: Menyediakan panel Time-Series, Gauge, Bar Chart, Heatmap, dan Node Graph dengan filter dropdown dinamis (misal memilih `$cluster`, `$namespace`, atau `$service`).
3. **Unified Alerting Engine**: Mengevaluasi aturan ambang batas alert lintas sumber data dan mengirimkan notifikasi ke Slack, PagerDuty, Discord, Telegram, atau Webhook.
4. **Dashboard as Code (Provisioning)**: Mendukung pengelolaan dashboard dan datasource dalam bentuk file deklaratif JSON/YAML yang dapat disimpan di Git (_GitOps_).

## 3. Kenapa Harus Pakai Ini ?

### 1. Korelasi Visual Lintas Layer (The "Single Pane of Glass")

Di sistem terdistribusi dengan puluhan service, masalah jarang terjadi secara terisolasi. Penurunan throughput aplikasi hampir selalu berkaitan dengan saturasi resource di layer bawah. Grafana memungkinkan penumpukan grafik (_overlaying_): menyandingkan grafik throughput aplikasi Go dengan utilisasi CPU Kubernetes pod dan disk I/O database pada sumbu waktu yang sama persis.

### 2. Deteksi Anomali Lewat Heatmap Latensi

Grafik garis biasa sering gagal menunjukkan distribusi latensi yang timpang. Grafana menyediakan panel **Heatmap** yang memvisualisasikan seluruh sebaran latensi request secara mendalam, memudahkan engineer mendeteksi keberadaan _outlier requests_ atau anomali _bimodal latency_ dalam sekejap.

### 3. Templating & Otomatisasi Skalabilitas

Saat kluster Kubernetes kamu melakukan auto-scaling dari 10 pod menjadi 200 pod, kamu tidak perlu membuat 200 dashboard manual. Dengan fitur **Dashboard Variables** dan **Panel Repeat**, Grafana secara otomatis membuat panel pemantauan baru untuk setiap pod baru yang terdeteksi.

## 4. Cara Kerjanya (Arsitektur Multi-Datasource Grafana)

```
[ PENGGUNA: SRE / Backend Engineer / Management ]
                        |
                        v
         +-----------------------------+
         |     GRAFANA SERVER (UI)     |
         |  - Interactive Dashboards   |
         |  - Unified Alerting Engine  |
         |  - Role-Based Access (RBAC) |
         +-----------------------------+
                        |
       (Query Paralel Berdasarkan Panel)
                        |
       +----------------+----------------+
       |                |                |
       v                v                v
+--------------+ +--------------+ +--------------+
|  Prometheus  | | Grafana Loki | |  PostgreSQL  |
|  (Metrics)   | | (App Logs)   | | (Biz Data)   |
+--------------+ +--------------+ +--------------+
```

## 5. Cuplikan Konfigurasi Inti

### 1. Deklarasi Otomatis Datasource Prometheus (`datasources.yml`)

```yaml
apiVersion: 1

datasources:
    - name: Prometheus-Production
      type: prometheus
      access: proxy
      url: http://prometheus-server.monitoring.svc.cluster.local:9090
      isDefault: true
      editable: false
      jsonData:
          httpMethod: POST
          timeInterval: 15s
```

### 2. Cuplikan Query Panel PromQL untuk Dashboard Grafana

Query yang umum digunakan pada panel dashboard microservices:

```promql
# Panel 1: Throughput Request per Detik (RPS) per Endpoint
sum by (endpoint) (rate(http_requests_total{status=~"2.."}[1m]))

# Panel 2: Latensi P99 (SLA Compliance)
histogram_quantile(0.99, sum by (le, endpoint) (rate(http_request_duration_seconds_bucket[5m])))

# Panel 3: Rasio Error 5xx (Error Rate %)
(
  sum(rate(http_requests_total{status=~"5.."}[1m]))
  /
  sum(rate(http_requests_total[1m]))
) * 100
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Fleksibilitas Tanpa Tanding**: Tidak terkunci pada satu vendor database monitoring tertentu (_zero vendor lock-in_).
-   **Komunitas & Ekosistem Luas**: Tersedia ribuan template dashboard siap pakai dari komunitas (untuk Node Exporter, Kubernetes, Redis, NGINX, PostgreSQL).
-   **Kolaborasi Efisien**: Fitur snapshot dashboard dan anotasi waktu mempermudah tim melakukan post-mortem insiden produksi.

### Kekurangan

-   **Bukan Mesin Penyimpan Data**: Grafana murni layer presentasi; Grafana tidak menyimpan data time-series itu sendiri. Performa dashboard sangat bergantung pada kecepatan database di belakangnya (Prometheus / Elasticsearch).
-   **Risiko Query Berlebih (Query Overload)**: Dashboard publik dengan refresh rate 1 detik yang dibuka oleh ratusan developer dapat membebani database Prometheus hingga crash (_accidental DDoS on TSDB_).
-   **Kompleksitas Manajemen Akses di Skala Besar**: Fitur RBAC tingkat lanjut dan integrasi audit log tim enterprise hanya tersedia di lisensi berbayar Grafana Enterprise.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Pakai Grafana

-   Pusat komando pemantauan operasional (_Single Pane of Glass_) untuk tim engineering dan SRE.
-   Visualisasi metrik throughput, latensi P95/P99, error rate, dan saturasi resource secara real-time.
-   Menampilkan metrik bisnis penting di layar TV kantor (NOC room dashboard).

### Kapan Jangan Mengandalkan Grafana Saja

-   Analisis deep stack trace kode aplikasi saat debugging error logika lokal (lebih tepat menggunakan APM / Jaeger).
-   Sebagai database analisis analitik data besar (_data warehouse reporting_) yang membutuhkan query ad-hoc kompleks berjam-jam (lebih cocok Tableau / Metabase di atas BigQuery / ClickHouse).

## 8. Metrik Kunci & Observabilitas

Metrik untuk memantau performa instance Grafana itu sendiri:

-   **`grafana_http_request_duration_seconds`**: Waktu respons internal API Grafana saat merender halaman.
-   **`grafana_api_dataproxy_request_all_milliseconds`**: Durasi waktu tunggu query proxy antara Grafana dan datasource eksternal (mendeteksi datasource yang lemot).
-   **`grafana_stat_active_users`**: Jumlah pengguna aktif yang sedang membuka dashboard.

## 9. Studi Kasus Nyata di Industri

### Unifikasi Visualisasi Observabilitas (Metrik, Log, & Traces) pada Dashboard Incident Response

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah institusi e-commerce multinasional mengalami fragmentasi observabilitas yang parah saat insiden outages: tim infrastruktur melihat metrik di Prometheus, tim backend mengecek log teks manual di server via SSH atau Kibana terpisah, dan tim reliabilitas melacak jejak di Jaeger. Ketika sistem checkout melambat saat flash sale, para on-call engineer membutuhkan waktu 45 menit hanya untuk menghubungkan ID pesanan yang gagal dengan pod server yang bermasalah karena harus berganti-ganti 4 tool antarmuka yang berbeda (_context-switching fatigue_).
- **Implementasi Solusi Industri (The Architectural Solution)**: Tim membangun *Single Pane of Glass* observabilitas terpadu menggunakan *Grafana Enterprise*. Tiga pilar observabilitas diintegrasikan dalam satu dashboard insiden: panel visualisasi metrik latensi PromQL (Prometheus), panel penelusuran log real-time berbasis LogQL (Grafana Loki), dan visualisasi alur stack trace terdistribusi (Grafana Tempo). Fitur *Data Links* dikonfigurasi sehingga engineer yang mengklik lonjakan p99 di grafik metrik dapat langsung melompat ke trace ID yang relevan di Tempo, lalu melihat baris log tepat saat transaksi error terjadi di Loki hanya dengan 1 klik. Dashboard distandarisasi sebagai kode (*Dashboard-as-Code*) menggunakan Terraform.
- **Dampak & Hasil Sistem (The Outcome)**: Mean Time to Resolve (MTTR) saat insiden produksi terpangkas drastis dari 45 menit menjadi rata-rata 4 menit. Beban kognitif on-call engineer berkurang secara signifikan karena seluruh korelasi data telemetri tersaji dalam satu layar terpadu, dan utilisasi resource datasource monitoring hemat 30% berkat fitur Query Caching di Grafana.

## 10. Fakta Menarik & Gotchas Production

-   **The 1-Second Auto-Refresh Disaster**:  
    Banyak tim engineering memasang dashboard Grafana di TV ruang kerja dengan interval _Auto-Refresh_ disetel ke **1 detik** (`1s`). Jika dashboard tersebut memiliki 30 panel dengan query `histogram_quantile` yang berat, satu layar tersebut mengeksekusi **1.800 query PromQL berat per menit** ke Prometheus! Ini setara dengan melancarkan serangan denial-of-service ke server monitoring sendiri!  
    _Aturan Wajib:_ Batasi auto-refresh dashboard di lingkungan produksi minimal **`30s`** atau **`1m`**.
-   **Panel Query Caching (Menyelamatkan TSDB)**:  
    Di Grafana modern, aktifkan fitur **Query Caching** pada datasource. Jika ada 10 engineer yang membuka dashboard incident response yang sama secara bersamaan, Grafana hanya akan mengeksekusi 1 query ke Prometheus dan menyajikan hasilnya dari cache memori ke 9 engineer lainnya.

## 11. Rangkuman Singkat

- **Single Pane of Glass Terpadu**: Grafana menyatukan berbagai sumber data (Prometheus, Loki, Tempo, PostgreSQL, Elasticsearch) ke dalam satu antarmuka visualisasi yang interaktif dan fleksibel.
- **Korelasikan Metrik ke Log dan Traces**: Manfaatkan fitur Data Links di Grafana untuk melompat dari lonjakan grafik metrik langsung ke baris log yang error dan distributed trace ID terkait dalam satu klik.
- **Waspadai Interval Auto-Refresh TV**: Jangan setel auto-refresh panel ke angka ekstrem (1-5 detik) pada monitor display kantor; tetapkan batas aman minimal 30-60 detik untuk mencegah DoS terhadap datasource monitoring.
- **Terapkan Dashboard as Code**: Kelola definisi dashboard menggunakan format JSON atau generator kode (Terraform/Grafonnet) di repositori Git agar perubahan visualisasi dapat direview dan dilacak versinya.
