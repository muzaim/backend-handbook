# Apache Kafka

## 1. Skenario Sistem Nyata

Pada platform ride-hailing dan delivery berskala raksasa sekelas Uber atau Grab, ratusan ribu event GPS dan transaksi diproduksi setiap detik: Setiap detik ada:

-   500.000 pengemudi yang memancarkan koordinat GPS via ponsel mereka.
-   100.000 transaksi pembayaran, order makanan, dan promo diskon yang berjalan bersamaan.
-   5 sistem hilir yang membutuhkan aliran data ini secara serentak: tim _Fraud Detection_ (mendeteksi order fiktif secara real-time), tim _Pricing Surge Engine_ (menghitung tarif dinamis per zona), tim _Driver Matching_, sistem _Data Warehouse/Analytics_, dan _Customer Notification Service_.

Jika arsitektur ini dibangun menggunakan HTTP REST API atau Message Broker antrean tradisional (seperti RabbitMQ dengan antrean point-to-point):

-   Saat sistem Data Warehouse mengalami perlambatan pemrosesan data (_lagging_), seluruh antrean pesan akan menumpuk di RAM broker hingga memori jebol (_OOM Crash_).
-   Pada RabbitMQ standar, begitu sebuah pesan dibaca dan di-ack oleh satu consumer, pesan tersebut dihapus dari antrean. Sangat sulit bagi 5 sistem hilir yang independen untuk membaca ulang (_replay_) data masa lalu jika terjadi bug.
-   Throughput pesan tercekik di angka puluhan ribu pesan per detik karena overhead komputasi pengelolaan status per pesan di broker.

**Solusi dengan Apache Kafka:**
Semua event GPS dan transaksi ditulis ke Kafka sebagai **Append-Only Commit Log** yang didistribusikan ke dalam beberapa partisi (_partitions_). Kafka tidak menghapus pesan saat dibaca; pesan disimpan secara persisten di disk berkecepatan tinggi selama 7 hari. Kelima sistem hilir membaca data secara independen menggunakan **Consumer Groups** dengan offset masing-masing. Throughput melonjak hingga **jutaan event per detik** dengan latensi beberapa milidetik berkat pemanfaatan OS Page Cache dan optimasi Zero-Copy!

## 2. Apa Itu dan Buat Apa?

**Apache Kafka** adalah platform _distributed event streaming_ terdistribusi yang dirancang untuk menangani aliran data (_event pipelines_), analisis streaming, dan integrasi data skala sangat besar dengan throughput ekstrem, latensi rendah, serta toleransi kesalahan yang tangguh.

### Konsep Kunci Kafka:

1. **Event / Message**: Rekaman data yang terdiri dari Key, Value, Timestamp, dan Headers opsional.
2. **Topic**: Kategori atau feed log tempat event dipublikasikan (analog dengan nama tabel di database atau folder).
3. **Partition**: Setiap Topic dipecah menjadi beberapa Partisi log independen untuk memungkinkan skalabilitas horizontal dan pemrosesan paralel lintas mesin broker.
4. **Offset**: Nomor urut identitas unik integer yang diberikan ke setiap pesan di dalam sebuah partisi. Offset bersifat monotonik naik (_immutable sequence_).
5. **Producer**: Aplikasi yang mempublikasikan (_write_) event ke Kafka topic.
6. **Consumer & Consumer Group**: Aplikasi yang membaca (_subscribe_) event dari topic. Konsumen yang berada dalam Consumer Group yang sama akan membagi partisi secara merata untuk pemrosesan paralel (_load balancing consumer_).

## 3. Kenapa Harus Pakai Ini ?

### 1. Sequential Disk I/O yang Sangat Cepat

Banyak developer mengira disk drive itu lambat. Padahal yang lambat adalah _Random Disk Access_. Untuk **Sequential Disk Write (Append-Only Log)**, kecepatan tulis disk modern (bahkan HDD konvensional) bisa mencapai 600 MB/s—menyamai performa akses memori acak. Kafka hanya menambahkan data ke akhir file log (_append-only_), menjadikannya sangat cepat.

### 2. Zero-Copy Network Transfer (`sendfile` Linux Syscall)

Pada server web biasa, mengirim data dari disk ke network melewati 4 langkah penyalinan memori (Disk -> OS Page Cache -> Application Buffer -> Socket Buffer -> NIC).  
Kafka menggunakan _syscall_ OS Linux **`sendfile()`** (Zero-Copy): data dari OS Page Cache langsung disalin ke Network Interface Card (NIC) tanpa pernah mampir ke memori JVM Kafka! Ini menghemat beban CPU dan memory bandwidth secara masif.

### 3. Log Retention & Message Replayability

Pesan di Kafka tidak hilang saat dibaca. Pesan disimpan berdasarkan batas waktu (_time-based retention_, misal 7 hari) atau batas ukuran (_size-based retention_). Jika tim Fraud mendeteksi ada bug pada algoritma mereka kemarin sore, mereka cukup me-reset consumer offset ke 24 jam yang lalu dan memproses ulang (_replay_) seluruh aliran transaksi dari masa lalu!

## 4. Cara Kerjanya (Arsitektur Topic, Partition, dan Consumer Group)

```
TOPIC: "driver-locations" (Terbagi ke 3 Partisi)

[ Producer: Driver App ]
          |
     (Kirim GPS)
          |
          v
+-------------------------------------------------------------+
| KAFKA BROKER CLUSTER                                        |
|                                                             |
| Partisi 0:  [Offset 0] [Offset 1] [Offset 2] [Offset 3] --> |
| Partisi 1:  [Offset 0] [Offset 1] [Offset 2] --------->     |
| Partisi 2:  [Offset 0] [Offset 1] [Offset 2] [Offset 3] --> |
+-------------------------------------------------------------+
          |                                   |
          v                                   v
[ Consumer Group: "fraud-service" ]   [ Consumer Group: "pricing-service" ]
- Consumer A membaca Partisi 0 & 1    - Consumer X membaca Partisi 0
- Consumer B membaca Partisi 2        - Consumer Y membaca Partisi 1
                                      - Consumer Z membaca Partisi 2
(Kedua grup membaca data yang sama secara independen tanpa saling mengganggu!)
```

## 5. Cuplikan Logika Inti

### 1. Producer di Go dengan Partition Key

Menentukan `Key` memastikan bahwa seluruh event milik satu pengemudi selalu masuk ke partisi yang sama (_ordering guarantee per key_):

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

### 2. Consumer Group di Go dengan Manual Offset Commit

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

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Throughput Raksasa**: Mampu menelan jutaan event per detik per cluster dengan latensi transfer rendah (1-5 ms).
-   **Decoupling Sempurna**: Produsen dan banyak konsumen tidak saling bergantung dalam hal kecepatan konsumsi data (_backpressure immune_).
-   **Durabilitas & Replayability**: Data tersimpan aman di disk terdistribusi dan dapat dibaca ulang kapan saja.

### Kekurangan

-   **Kompleksitas Operasional Tinggi**: Menjalankan cluster Kafka production membutuhkan pemahaman arsitektur mendalam (Zookeeper / KRaft metadata mode, tuning JVM garbage collection, partisi, dan replikasi).
-   **Jaminan Urutan Hanya per Partisi**: Kafka HANYA menjamin urutan pesan di dalam partisi yang sama, BUKAN di seluruh topic secara global.
-   **Consumer Rebalance Storm**: Jika ada satu consumer lambat atau mati, Kafka akan memicu proses _rebalance_ partisi yang dapat menghentikan konsumsi sejenak (_stop-the-world rebalance pause_).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Pakai Kafka

-   Event-driven microservices architecture dengan banyak downstream consumer yang membutuhkan event yang sama.
-   Data ingestion pipelines bervolume raksasa (IoT telemetry, activity clickstream, log aggregation).
-   Audit trail, Change Data Capture (CDC via Debezium), dan pemrosesan stream analitik (Apache Flink/Spark).

### Kapan Jangan Pakai Kafka

-   Antrean tugas sederhana di mana setiap task hanya perlu dikerjakan oleh satu worker lalu segera dihapus (cukup gunakan RabbitMQ, Redis Streams, atau AWS SQS).
-   Kebutuhan routing pesan yang sangat kompleks (misal AMQP Topic routing berdasarkan header dinamis).
-   Tim kecil dengan beban trafik rendah yang tidak memiliki sumber daya untuk memelihara operasional cluster Kafka.

## 8. Metrik Kunci & Observabilitas

Metrik wajib di dashboard Grafana saat memonitor Kafka cluster:

-   **`consumer_lag` (Partisi & Grup)**: Selisih antara offset pesan terbaru di broker dengan offset yang sudah diproses oleh consumer. Kenaikan _lag_ menandakan consumer lambat atau macet!
-   **`under_replicated_partitions` (URP)**: Jumlah partisi yang kehilangan salinan replikanya (harus selalu 0; angka > 0 menandakan ada broker node yang mati atau disk failure).
-   **`bytes_in_per_sec` & `bytes_out_per_sec`**: Volume throughput bandwidth jaringan yang masuk dan keluar dari broker.
-   **`request_queue_time_ms`**: Waktu yang dihabiskan request di antrean broker sebelum diproses oleh thread worker.

## 9. Studi Kasus Nyata di Industri

### Pipa Penyerapan 500.000 Event/Detik Telemetri Finansial dan Deteksi Penipuan Real-Time

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah bank digital internasional memproses ratusan ribu transaksi pembayaran, otorisasi kartu, dan sinyal geolokasi perangkat setiap detik. Sistem lama menggunakan antrean pesan point-to-point konvensional (RabbitMQ). Ketika volume event melonjak ke 500.000 event/detik saat event promo, antrean RAM broker jenuh, node mengalami memory pressure OOM, dan pesan mulai ter-drop. Selain itu, beberapa departemen (Fraud Detection, Audit Ledger, Push Notification, dan Data Lake) memerlukan data transaksi yang sama secara bersamaan, sehingga sistem lama terpaksa menduplikasi 4 kali pengiriman pesan yang membebani jaringan origin.
- **Implementasi Solusi Industri (The Architectural Solution)**: Tim membangun tulang punggung streaming berbasis kluster *Apache Kafka* (mode KRaft) dengan topic terpartisi berdasarkan `account_id` untuk menjamin urutan pesan yang teratur per akun. Parameter produser disetel dengan `acks=all`, kompresi Snappy, dan `idempotent=true` untuk mencegah duplikasi pesan. Layanan Fraud Detection membaca stream via Kafka Streams dengan latensi rendah (< 10 ms), sementara consumer group Audit Ledger dan Data Lake ingestion (CDC via Debezium & Snowflake connector) membaca stream yang sama secara independen tanpa saling berebut resource atau menghapus data log yang persisten di disk selama 14 hari retention.
- **Dampak & Hasil Sistem (The Outcome)**: Throughput penyerapan event melonjak hingga 850.000 event/detik dengan latensi end-to-end P99 di bawah 15 ms. Sistem deteksi fraud mampu memblokir transaksi mencurigakan dalam tempo 40 milidetik sejak kartu digesek. Zero message drop tercatat saat lonjakan trafik, dan insiden backpressure teratasi karena Kafka menyimpan log secara sekuensial di NVMe disk secara aman tanpa membebani memori RAM broker.

## 10. Fakta Menarik & Gotchas Production

-   **The Consumer Rebalance Storm (Stop-the-World Production Freeze)**:  
    Jika konsumer pod memproses satu pesan terlalu lama melebihi ambang batas `max.poll.interval.ms` (default 5 menit), Kafka Coordinator mengira pod tersebut mati! Akibatnya, Kafka mencabut hak partisi dan memicu **Consumer Group Rebalance**. Saat rebalance terjadi, seluruh pemrosesan di grup tersebut berhenti total! Jika konsumer berikutnya juga lambat, rebalance akan berulang terus tanpa henti (_infinite rebalance loop_).  
    _Solusi Wajib:_ Turunkan ukuran batch pesan (`max.poll.records`), pisahkan komputasi berat ke worker pool asynchronous, atau naikkan `max.poll.interval.ms`.
-   **Partition Count is a One-Way Street**:  
    Kamu bisa menambah jumlah partisi di sebuah topic, tetapi kamu **TIDAK BISA mengurangi jumlah partisi** yang sudah dibuat tanpa menghapus dan membuat ulang topic dari awal! Selain itu, menambah partisi akan merusak hashing key partisi sebelumnya (`hash(key) % partition_count`).  
    _Aturan Wajib:_ Rencanakan jumlah partisi dengan matang sejak awal berdasarkan target throughput dan jumlah consumer concurrency.

## 11. Rangkuman Singkat

- **Distributed Commit Log Abadi**: Kafka bukan sekadar message queue; Kafka adalah log terdistribusi yang menyimpan aliran data secara persisten dan berurutan di disk, memungkinkan pembacaan ulang historis (*replayability*).
- **Skalabilitas Horisontal via Partisi**: Throughput Kafka diskalakan dengan membagi topic ke banyak partisi; urutan pesan hanya dijamin di dalam satu partisi yang sama menggunakan penentuan partition key yang konsisten.
- **Banyak Konsumen Tanpa Duplikasi Beban**: Beberapa consumer group independen dapat membaca topic yang sama dengan kecepatan masing-masing tanpa saling memblokir atau menghapus pesan log.
- **Waspadai Consumer Rebalance Storm**: Pastikan pemrosesan batch event di aplikasi tidak melebihi `max.poll.interval.ms` agar broker tidak mencurigai consumer mati dan memicu rebalance massal yang menghentikan alur konsumsi data.
