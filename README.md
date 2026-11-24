# Kamus Backend High-Throughput

Selamat datang di **Kamus Backend High-Throughput**. Repositori ini dirancang khusus sebagai panduan komprehensif, praktis, dan ramah developer junior untuk memahami konsep-konsep inti arsitektur backend berskala besar (*high-throughput distributed systems*).

Di sini, kita tidak hanya membahas teori, tetapi juga skenario sistem nyata, alasan mengapa sebuah teknik dibutuhkan saat trafik meledak, trade-offs (kelebihan dan kekurangan), diagram alur ASCII yang bersih, cuplikan logika inti dalam bahasa Go (Golang), panduan kapan harus pakai vs kapan jangan pakai, metrik kunci observabilitas, studi kasus industri nyata, serta jebakan umum (*gotchas*) di lingkungan produksi.

> Baca versi bahasa Inggris: [English Version](README.en.md)

---

## Peta Kurikulum (8 Kategori Arsitektur)

Seluruh materi di dalam repositori ini dikelompokkan ke dalam 8 folder kategori utama untuk mencerminkan perjalanan sebuah request dari layer terluar, pemrosesan data, fondasi delivery, perangkat lunak konkret, hingga pemantauan sistem produksi:

```
+-------------------------------------------------------------------------+
| traffic-and-edge/ (Gerbang Utama & Pengatur Lalu Lintas)                |
| 1. Reverse Proxies       2. API Gateways                                |
| 3. Load Balancing        4. Rate Limiting                               |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| performance-and-data/ (Akselerasi & Akses Data Cepat)                   |
| 5. Caching                                                              |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| resilience-and-fault-tolerance/ (Ketahanan & Penanganan Masalah)        |
| 6. Timeouts              7. Retries                                     |
| 8. Exponential Backoff   9. Circuit Breakers                            |
| 10. Idempotency                                                         |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| async-and-distributed-systems/ (Komunikasi Skala Besar)                 |
| 11. Message Queues       12. Pub/Sub                                    |
| 13. Event-Driven Arch    14. Distributed Transactions                   |
| 15. Saga Pattern                                                        |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| infrastructure-and-delivery/ (Fondasi Operasional & Deployment)         |
| 16. Service Discovery    17. CI/CD                                      |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| protocols-and-realtime/ (Protokol Berperforma Tinggi & Real-Time)       |
| 18. gRPC & Protobuf      19. WebSockets & SSE                           |
| 20. CDN & Edge Caching                                                  |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| tools-and-technologies/ (Perangkat Lunak Konkret High-Throughput)       |
| 21. Docker               22. Kubernetes                                 |
| 23. Redis                24. Apache Kafka                               |
| 25. PostgreSQL           26. NGINX                                      |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| monitoring-and-observability/ (Pemantauan & Keandalan Sistem Produksi)  |
| 27. Prometheus           28. Grafana                                    |
| 29. OpenTelemetry        30. Centralized Logging                        |
+-------------------------------------------------------------------------+
```

---

## Daftar Topik Lengkap

Setiap folder memiliki dokumentasi dwibahasa: `README.md` (Bahasa Indonesia) dan `README.en.md` (English). Seluruh cuplikan logika aplikasi menggunakan bahasa **Go (Golang)**.

### 1. Traffic & Edge (`traffic-and-edge/`)
- [Reverse Proxies](traffic-and-edge/reverse-proxies/README.md) - Pelindung dan perantara antara client dan server internal.
- [API Gateways](traffic-and-edge/api-gateways/README.md) - Pintu masuk tunggal untuk orkestrasi, autentikasi, dan routing microservices.
- [Load Balancing](traffic-and-edge/load-balancing/README.md) - Pembagi beban trafik agar tidak ada server yang tumbang sendirian.
- [Rate Limiting](traffic-and-edge/rate-limiting/README.md) - Pembatas laju request untuk mencegah abuse dan kelebihan muatan.

### 2. Performance & Data Access (`performance-and-data/`)
- [Caching](performance-and-data/caching/README.md) - Penyimpanan data sementara berkecepatan tinggi untuk memangkas latensi database.

### 3. Resilience & Fault Tolerance (`resilience-and-fault-tolerance/`)
- [Timeouts](resilience-and-fault-tolerance/timeouts/README.md) - Batas waktu tunggu agar thread atau koneksi tidak menggantung selamanya.
- [Retries](resilience-and-fault-tolerance/retries/README.md) - Percobaan ulang otomatis untuk menghadapi kegagalan jaringan yang bersifat sementara.
- [Exponential Backoff](resilience-and-fault-tolerance/exponential-backoff/README.md) - Strategi penundaan waktu retry secara eksponensial dengan jitter.
- [Circuit Breakers](resilience-and-fault-tolerance/circuit-breakers/README.md) - Pemutus aliran request saat downstream service mulai sakit/tumbang.
- [Idempotency](resilience-and-fault-tolerance/idempotency/README.md) - Jaminan bahwa request yang diproses berulang kali menghasilkan efek yang sama persis.

### 4. Asynchronous & Distributed Systems (`async-and-distributed-systems/`)
- [Message Queues](async-and-distributed-systems/message-queues/README.md) - Antrean pesan asynchronous berbasis point-to-point untuk peredam lonjakan trafik.
- [Pub/Sub](async-and-distributed-systems/pub-sub/README.md) - Pola distribusi pesan satu-ke-banyak (fan-out) berbasis topik.
- [Event-Driven Architecture](async-and-distributed-systems/event-driven-architecture/README.md) - Desain sistem yang bereaksi terhadap perubahan event secara decoupled.
- [Distributed Transactions](async-and-distributed-systems/distributed-transactions/README.md) - Tantangan konsistensi data lintas database terpisah (dan batasannya).
- [Saga Pattern](async-and-distributed-systems/saga-pattern/README.md) - Pola transaksi bisnis terdistribusi berbasis serangkaian transaksi lokal dan kompensasi.

### 5. Infrastructure & Delivery (`infrastructure-and-delivery/`)
- [Service Discovery](infrastructure-and-delivery/service-discovery/README.md) - Mekanisme dinamis agar service dapat saling menemukan alamat IP dan port rekannya.
- [CI/CD](infrastructure-and-delivery/ci-cd/README.md) - Otomatisasi integrasi kode, testing, dan deployment secara aman dan terukur.

### 6. Protocols & Real-Time (`protocols-and-realtime/`)
- [gRPC & Protocol Buffers](protocols-and-realtime/grpc-and-protobuf/README.md) - Protokol RPC biner berbasis HTTP/2 untuk komunikasi antar-layanan berkecepatan tinggi.
- [WebSockets & Server-Sent Events](protocols-and-realtime/websockets-and-sse/README.md) - Komunikasi data real-time dua arah dan streaming event berlatensi rendah.
- [CDN & Edge Caching](protocols-and-realtime/cdn-and-edge-caching/README.md) - Distribusi konten statis dan dynamic caching di server tepi terdekat dengan pengguna.

### 7. Tools & Technologies (`tools-and-technologies/`)
- [Docker](tools-and-technologies/docker/README.md) - Kontainerisasi aplikasi untuk lingkungan eksekusi yang konsisten dan terisolasi.
- [Kubernetes](tools-and-technologies/kubernetes/README.md) - Orkestrasi kontainer skala besar untuk self-healing, scaling, dan manajemen deployment.
- [Redis](tools-and-technologies/redis/README.md) - In-memory key-value data store untuk caching super cepat, distributed lock, dan antrean real-time.
- [Apache Kafka](tools-and-technologies/kafka/README.md) - Platform event streaming terdistribusi berbasis log untuk menampung jutaan event per detik.
- [PostgreSQL](tools-and-technologies/postgresql/README.md) - Database relasional enterprise dengan konkurensi MVCC, WAL, dan integritas data ACID mutlak.
- [NGINX](tools-and-technologies/nginx/README.md) - Web server dan asynchronous reverse proxy berkinerja tinggi untuk SSL offloading dan traffic buffering.

### 8. Monitoring & Observability (`monitoring-and-observability/`)
- [Prometheus](monitoring-and-observability/prometheus/README.md) - Sistem pemantauan metrik time-series berbasis pull-model dan evaluasi PromQL real-time.
- [Grafana](monitoring-and-observability/grafana/README.md) - Pusat visualisasi operasional (Single Pane of Glass) untuk metrik, log, dan trace.
- [OpenTelemetry](monitoring-and-observability/opentelemetry/README.md) - Standar observabilitas industri terpadu untuk distributed tracing dan context propagation.
- [Centralized Logging](monitoring-and-observability/centralized-logging/README.md) - Agregasi log terstruktur (JSON) skala besar menggunakan Grafana Loki / Elasticsearch.

---

## Panduan Belajar

1. **Ikuti urutan layer**: Disarankan membaca berurutan dari layer depan (`traffic-and-edge`), data, fault tolerance, infrastruktur, tools, hingga monitoring (`monitoring-and-observability`) agar terbangun mental model menyeluruh.
2. **Perhatikan bagian Trade-offs dan Gotchas**: Dalam arsitektur sistem, tidak ada solusi yang sempurna; yang ada hanyalah pertukaran konsekuensi (*trade-offs*).
3. **Pahami skenario sistemnya**: Setiap topik diawali dengan skenario arsitektur nyata di industri untuk memberikan konteks mengapa teknologi tersebut mutlak dibutuhkan saat trafik meledak.
