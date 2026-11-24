# Kamus Backend High-Throughput

Selamat datang di **Kamus Backend High-Throughput**. Repositori ini dirancang khusus sebagai panduan komprehensif, praktis, dan ramah developer junior untuk memahami konsep-konsep inti arsitektur backend berskala besar (*high-throughput distributed systems*).

Di sini, kita tidak hanya membahas teori, tetapi juga skenario sistem nyata, alasan mengapa sebuah teknik dibutuhkan saat trafik meledak, trade-offs (kelebihan dan kekurangan), diagram alur ASCII yang bersih, cuplikan logika inti dalam bahasa Go (Golang) dan konfigurasi produksi teruji, panduan kapan harus pakai vs kapan jangan pakai, metrik kunci observabilitas, studi kasus industri nyata, serta jebakan umum (*gotchas*) di lingkungan produksi.

> Baca versi bahasa Inggris: [English Version](README.en.md)

---

## Peta Kurikulum (9 Kategori Arsitektur - 40 Topik)

Seluruh materi di dalam repositori ini dikelompokkan ke dalam 9 folder kategori utama untuk mencerminkan perjalanan sebuah request dari fondasi penskalaan, layer terluar, pemrosesan data, fondasi delivery, perangkat lunak konkret, hingga pemantauan sistem produksi:

```
+-------------------------------------------------------------------------+
| architecture-and-scaling/ (Fondasi Desain & Penskalaan Sistem)          |
| 1. Horizontal Scaling    2. Vertical Scaling                            |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| traffic-and-edge/ (Gerbang Utama & Pengatur Lalu Lintas)                |
| 3. Reverse Proxies       4. API Gateways                                |
| 5. Load Balancing        6. Rate Limiting                               |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| performance-and-data/ (Akselerasi & Akses Data Cepat)                   |
| 7. Caching               8. Database Indexing    9. SQL vs NoSQL        |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| resilience-and-fault-tolerance/ (Ketahanan & Penanganan Masalah)        |
| 10. Timeouts             11. Retries             12. Exp. Backoff       |
| 13. Circuit Breakers     14. Idempotency                                |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| async-and-distributed-systems/ (Komunikasi Skala Besar)                 |
| 15. Message Queues       16. Pub/Sub             17. Event-Driven Arch  |
| 18. Dist. Transactions   19. Saga Pattern                               |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| infrastructure-and-delivery/ (Fondasi Operasional & Deployment)         |
| 20. Service Discovery    21. CI/CD                                      |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| protocols-and-realtime/ (Protokol Berperforma Tinggi & Real-Time)       |
| 22. gRPC & Protobuf      23. WebSockets & SSE    24. CDN & Edge Caching |
| 25. Webhooks                                                            |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| tools-and-technologies/ (Perangkat Lunak Konkret High-Throughput)       |
| 26. Docker               27. Kubernetes          28. Redis              |
| 29. Apache Kafka         30. PostgreSQL          31. NGINX              |
| 32. Jenkins              33. RabbitMQ            34. Elasticsearch      |
| 35. PgBouncer            36. HAProxy                                    |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| monitoring-and-observability/ (Pemantauan & Keandalan Sistem Produksi)  |
| 37. Prometheus           38. Grafana             39. OpenTelemetry      |
| 40. Centralized Logging                                                 |
+-------------------------------------------------------------------------+
```

---

## Daftar Topik Lengkap

Setiap folder memiliki dokumentasi dwibahasa: `README.md` (Bahasa Indonesia) dan `README.en.md` (English). Seluruh cuplikan logika aplikasi menggunakan bahasa **Go (Golang)** dan konfigurasi terstandar.

### 1. Architecture & Scaling (`architecture-and-scaling/`)
- [Horizontal Scaling](architecture-and-scaling/horizontal-scaling/README.md) - Penskalaan elastis dengan menambah jumlah node server tanpa status (*stateless*).
- [Vertical Scaling](architecture-and-scaling/vertical-scaling/README.md) - Peningkatan kapasitas daya komputasi (CPU, RAM, NVMe) pada satu mesin tunggal.

### 2. Traffic & Edge (`traffic-and-edge/`)
- [Reverse Proxies](traffic-and-edge/reverse-proxies/README.md) - Pelindung dan perantara antara client dan server internal.
- [API Gateways](traffic-and-edge/api-gateways/README.md) - Pintu masuk tunggal untuk orkestrasi, autentikasi, dan routing microservices.
- [Load Balancing](traffic-and-edge/load-balancing/README.md) - Pembagi beban trafik agar tidak ada server yang tumbang sendirian.
- [Rate Limiting](traffic-and-edge/rate-limiting/README.md) - Pembatas laju request untuk mencegah abuse dan kelebihan muatan.

### 3. Performance & Data Access (`performance-and-data/`)
- [Caching](performance-and-data/caching/README.md) - Penyimpanan data sementara berkecepatan tinggi untuk memangkas latensi database.
- [Database Indexing](performance-and-data/database-indexing/README.md) - Struktur data pohon penolong untuk pencarian cepat $O(\log N)$ dan query optimization.
- [SQL vs NoSQL](performance-and-data/sql-vs-nosql/README.md) - Perbandingan integritas transaksional (ACID) versus skalabilitas horizontal dan skema fleksibel.

### 4. Resilience & Fault Tolerance (`resilience-and-fault-tolerance/`)
- [Timeouts](resilience-and-fault-tolerance/timeouts/README.md) - Batas waktu tunggu agar thread atau koneksi tidak menggantung selamanya.
- [Retries](resilience-and-fault-tolerance/retries/README.md) - Percobaan ulang otomatis untuk menghadapi kegagalan jaringan yang bersifat sementara.
- [Exponential Backoff](resilience-and-fault-tolerance/exponential-backoff/README.md) - Strategi penundaan waktu retry secara eksponensial dengan jitter.
- [Circuit Breakers](resilience-and-fault-tolerance/circuit-breakers/README.md) - Pemutus aliran request saat downstream service mulai sakit/tumbang.
- [Idempotency](resilience-and-fault-tolerance/idempotency/README.md) - Jaminan bahwa request yang diproses berulang kali menghasilkan efek yang sama persis.

### 5. Asynchronous & Distributed Systems (`async-and-distributed-systems/`)
- [Message Queues](async-and-distributed-systems/message-queues/README.md) - Antrean pesan asynchronous berbasis point-to-point untuk peredam lonjakan trafik.
- [Pub/Sub](async-and-distributed-systems/pub-sub/README.md) - Pola distribusi pesan satu-ke-banyak (fan-out) berbasis topik.
- [Event-Driven Architecture](async-and-distributed-systems/event-driven-architecture/README.md) - Desain sistem yang bereaksi terhadap perubahan event secara decoupled.
- [Distributed Transactions](async-and-distributed-systems/distributed-transactions/README.md) - Tantangan konsistensi data lintas database terpisah (dan batasannya).
- [Saga Pattern](async-and-distributed-systems/saga-pattern/README.md) - Pola transaksi bisnis terdistribusi berbasis serangkaian transaksi lokal dan kompensasi.

### 6. Infrastructure & Delivery (`infrastructure-and-delivery/`)
- [Service Discovery](infrastructure-and-delivery/service-discovery/README.md) - Mekanisme dinamis agar service dapat saling menemukan alamat IP dan port rekannya.
- [CI/CD](infrastructure-and-delivery/ci-cd/README.md) - Otomatisasi integrasi kode, testing, dan deployment secara aman dan terukur.

### 7. Protocols & Real-Time (`protocols-and-realtime/`)
- [gRPC & Protocol Buffers](protocols-and-realtime/grpc-and-protobuf/README.md) - Protokol RPC biner berbasis HTTP/2 untuk komunikasi antar-layanan berkecepatan tinggi.
- [WebSockets & Server-Sent Events](protocols-and-realtime/websockets-and-sse/README.md) - Komunikasi data real-time dua arah dan streaming event berlatensi rendah.
- [CDN & Edge Caching](protocols-and-realtime/cdn-and-edge-caching/README.md) - Distribusi konten statis dan dynamic caching di server tepi terdekat dengan pengguna.
- [Webhooks](protocols-and-realtime/webhooks/README.md) - Pola HTTP push event proaktif (*Reverse API*) dengan verifikasi keamanan tanda tangan HMAC.

### 8. Tools & Technologies (`tools-and-technologies/`)
- [Docker](tools-and-technologies/docker/README.md) - Kontainerisasi aplikasi untuk lingkungan eksekusi yang konsisten dan terisolasi.
- [Kubernetes](tools-and-technologies/kubernetes/README.md) - Orkestrasi kontainer skala besar untuk self-healing, scaling, dan manajemen deployment.
- [Redis](tools-and-technologies/redis/README.md) - In-memory key-value data store untuk caching super cepat, distributed lock, dan antrean real-time.
- [Apache Kafka](tools-and-technologies/kafka/README.md) - Platform event streaming terdistribusi berbasis log untuk menampung jutaan event per detik.
- [PostgreSQL](tools-and-technologies/postgresql/README.md) - Database relasional enterprise dengan konkurensi MVCC, WAL, dan integritas data ACID mutlak.
- [NGINX](tools-and-technologies/nginx/README.md) - Web server dan asynchronous reverse proxy berkinerja tinggi untuk SSL offloading dan traffic buffering.
- [Jenkins](tools-and-technologies/jenkins/README.md) - Server otomatisasi CI/CD mandiri (*self-hosted*) untuk orkestrasi pipeline pengujian dan rilis kode.
- [RabbitMQ](tools-and-technologies/rabbitmq/README.md) - Message broker AMQP dengan routing fleksibel (Exchanges & Queues) dan konfirmasi per-pesan.
- [Elasticsearch](tools-and-technologies/elasticsearch/README.md) - Mesin pencari teks dan analitik terdistribusi berbasis Inverted Index dan scoring BM25.
- [PgBouncer](tools-and-technologies/pgbouncer/README.md) - Reverse connection pooler ringan penyelamat PostgreSQL dari kehabisan koneksi (*connection exhaustion*).
- [HAProxy](tools-and-technologies/haproxy/README.md) - Load balancer Layer 4 & 7 berkinerja tinggi dengan efisiensi memori ekstrem dan health check presisi.

### 9. Monitoring & Observability (`monitoring-and-observability/`)
- [Prometheus](monitoring-and-observability/prometheus/README.md) - Sistem pemantauan metrik time-series berbasis pull-model dan evaluasi PromQL real-time.
- [Grafana](monitoring-and-observability/grafana/README.md) - Pusat visualisasi operasional (Single Pane of Glass) untuk metrik, log, dan trace.
- [OpenTelemetry](monitoring-and-observability/opentelemetry/README.md) - Standar observabilitas industri terpadu untuk distributed tracing dan context propagation.
- [Centralized Logging](monitoring-and-observability/centralized-logging/README.md) - Agregasi log terstruktur (JSON) skala besar menggunakan Grafana Loki / Elasticsearch.

---

## Panduan Belajar

1. **Ikuti urutan layer**: Disarankan membaca berurutan dari layer fondasi penskalaan (`architecture-and-scaling`), gerbang luar (`traffic-and-edge`), data, fault tolerance, sistem asinkron, infrastruktur, protokol, perangkat lunak konkret (`tools-and-technologies`), hingga monitoring (`monitoring-and-observability`) agar terbangun mental model menyeluruh.
2. **Perhatikan bagian Trade-offs dan Gotchas**: Dalam arsitektur sistem, tidak ada solusi yang sempurna; yang ada hanyalah pertukaran konsekuensi (*trade-offs*).
3. **Pahami skenario sistemnya**: Setiap topik diawali dengan skenario arsitektur nyata di industri untuk memberikan konteks mengapa teknologi tersebut mutlak dibutuhkan saat trafik meledak.
