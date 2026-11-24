# Event-Driven Architecture (EDA)

## 1. Skenario Sistem Nyata

Pada sistem transportasi online (_ride-hailing_), saat penumpang memesan dan pengemudi menerima pesanan, terjadi sebuah peristiwa bisnis (_state change_) yang tidak dapat diubah: **`RideAssigned`**.

Peristiwa ini memicu serangkaian tindakan otonom di berbagai subsistem backend:

-   `Navigation Service` menghitung rute optimal dan mengalirkan panduan jalan ke ponsel pengemudi.
-   `Pricing & Payment Service` mengunci saldo dompet digital atau pre-authorization kartu kredit penumpang.
-   `Safety Monitoring Service` mulai memantau koordinat GPS secara real-time untuk mendeteksi deviasi rute darurat.
-   `Notification Service` mengirimkan pesan push ke ponsel penumpang yang berisi nomor plat kendaraan dan nama pengemudi.

Penumpang dan pengemudi tidak pernah memanggil API billing atau safety monitoring secara langsung. Masing-masing microservice bereaksi secara mandiri terhadap aliran fakta bisnis (_immutable event stream_).

Jika sistem ini dibangun menggunakan arsitektur tradisional berbasis REST sinkron (_Request-Driven_):

-   Service Pemesanan harus memanggil 4 service lainnya secara berantai (`Ride Svc -> Nav Svc -> Payment Svc -> Safety Svc`).
-   Jika `Safety Service` mengalami latensi atau kehabisan memori, seluruh proses penjemputan penumpang akan macet dan gagal (_deep synchronous coupling_).

**Solusi dengan Event-Driven Architecture:**
Sistem melepaskan ketergantungan panggilan langsung. Service hanya mempublikasikan event ke event bus terdistribusi (misal: Apache Kafka). Setiap service consumer membaca stream event tersebut secara mandiri dan memperbarui _read model_ lokalnya tanpa memblokir alur pemesanan utama.

## 2. Apa Itu dan Buat Apa?

Event-Driven Architecture (Arsitektur Berbasis Event) adalah paradigma desain perangkat lunak di mana aliran eksekusi sistem digerakkan oleh **pembentukan (_production_), deteksi (_detection_), dan konsumsi (_consumption_) peristiwa bisnis (_Events_)**.

### Perbedaan Mendasar: Command vs Event

-   **Command (Perintah)**: Tindakan yang _diminta_ kepada target spesifik. Bersifat imperatif dan bisa ditolak.  
    _Contoh:_ `ChargeCreditCard(amount: $50)` -> Bisa gagal jika kartu ditolak.
-   **Event (Peristiwa)**: Fakta sejarah tentang sesuatu yang _telah terjadi_ di masa lalu. Bersifat objektif, tidak dapat diubah (_immutable_), dan tidak peduli siapa yang mendengarkan.  
    _Contoh:_ `OrderPlaced`, `PaymentSucceeded`, `UserAccountRegistered`.

Komponen utama dalam EDA:

-   **Event Producer**: Service yang mendeteksi perubahan status bisnis dan menerbitkan fakta tersebut ke event stream.
-   **Event Broker / Stream**: Saluran terdistribusi yang menyimpan dan mengalirkan event (misal: Apache Kafka, AWS Kinesis).
-   **Event Consumer**: Service-service independen yang bereaksi terhadap event tersebut untuk menjalankan tugasnya sendiri.

## 3. Kenapa Harus Pakai Ini ?

Dalam arsitektur tradisional berbasis REST/RPC (_Request-Driven Architecture_), service saling memanggil secara berantai (_deep synchronous call trees_):
`API Gateway -> Order Svc -> Payment Svc -> Inventory Svc -> Notification Svc`.

Jika satu service di tengah rantai mengalami lonjakan trafik atau latensi, seluruh rantai di depannya akan macet (_tight coupling_).

Dalam Event-Driven Architecture:

-   Seluruh komunikasi bersifat **asynchronous dan non-blocking**.
-   Setiap service memiliki salinan data lokal (_Read Model_) yang diproyeksikan dari riwayat event, sehingga service dapat melayani query pengguna tanpa perlu memanggil service lain melalui jaringan (_zero inter-service synchronous network hops_).
-   Sistem dapat diskalakan secara masif ke jutaan event per detik tanpa saling mengunci resource database.

## 4. Cara Kerjanya (Diagram Alur)

```
[ Domain: E-Commerce Order Lifecycle ]

+--------------------+
|   Order Service    | (Producer)
+--------------------+
          |
          | 1. Simpan Transaksi Lokal ke PostgreSQL
          | 2. Emit Event: "OrderPlaced"
          v
+-------------------------------------------------------------+
|                     Event Stream (Kafka)                    |
|                                                             |
|  [ Event: OrderPlaced (ID: #401, Items: [A, B], Total: $80) ]|
+-------------------------------------------------------------+
          |                            |
          | Konsumsi Event             | Konsumsi Event
          v                            v
+--------------------+       +--------------------+
|   Payment Service  |       | Notification Svc   |
+--------------------+       +--------------------+
| - Potong Saldo     |       | - Kirim SMS & WA   |
| - Emit Event Baru: |       |   ke Pelanggan     |
|   "PaymentSettled" |       |                    |
+--------------------+       +--------------------+
```

## 5. Cuplikan Logika Inti (Transactional Outbox di Go)

Implementasi pola **Transactional Outbox Pattern** untuk mencegah _dual-write loss_ antara database dan message broker di Go:

```go
package main

import (
	"database/sql"
	"encoding/json"
	"time"

	"github.com/google/uuid"
)

type OrderEvent struct {
	OrderID   string    `json:"order_id"`
	UserID    string    `json:"user_id"`
	Amount    float64   `json:"amount"`
	CreatedAt time.Time `json:"created_at"`
}

// CreateOrderWithOutbox menyimpan entitas dan outbox event dalam SATU transaksi database lokal
func CreateOrderWithOutbox(db *sql.DB, userID string, amount float64) (string, error) {
	tx, err := db.Begin()
	if err != nil {
		return "", err
	}
	defer tx.Rollback()

	orderID := uuid.New().String()

	// 1. Simpan order ke tabel orders
	_, err = tx.Exec("INSERT INTO orders (id, user_id, amount, status) VALUES ($1, $2, $3, $4)",
		orderID, userID, amount, "PENDING")
	if err != nil {
		return "", err
	}

	// 2. Simpan event ke tabel outbox (Transactional Outbox Pattern)
	eventPayload, _ := json.Marshal(OrderEvent{
		OrderID:   orderID,
		UserID:    userID,
		Amount:    amount,
		CreatedAt: time.Now(),
	})

	_, err = tx.Exec("INSERT INTO outbox_events (id, aggregate_type, aggregate_id, payload, processed) VALUES ($1, $2, $3, $4, $5)",
		uuid.New().String(), "Order", orderID, eventPayload, false)
	if err != nil {
		return "", err
	}

	// Commit transaksi: kedua tabel tersimpan secara atomik tanpa risiko dual-write loss
	return orderID, tx.Commit()
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Skalabilitas & Throughput Maksimal**: Komponen produser dan konsumen berjalan dengan kecepatan independen tanpa saling memblokir.
-   **Extreme Fault Tolerance**: Jika Notification Service mati selama 2 jam, Order Service tetap berjalan normal. Ketika Notification Service hidup kembali, ia tinggal membaca tumpukan event yang tertinggal dari Kafka.
-   **Auditability Sempurna**: Riwayat event berfungsi sebagai catatan sejarah lengkap (_audit log_) yang mencatat setiap perubahan status sistem sejak hari pertama.

### Kekurangan

-   **Eventual Consistency**: Sistem tidak konsisten secara instan. Data saldo di layar pengguna mungkin butuh 1-2 detik untuk sinkron dengan database backend.
-   **Kompleksitas Schema Evolution**: Jika struktur event JSON berubah (misal menghapus field atau mengubah tipe data), kamu berisiko merusak seluruh consumer lama (_breaking contract_).
-   **Sulitnya Tracing & Debugging**: Melacak akar masalah saat sebuah transaksi gagal membutuhkan sistem observabilitas terdistribusi yang matang (_Correlation ID_ di setiap event).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Sistem terdistribusi skala besar dengan banyak tim dan domain bisnis berbeda.
-   Platform analitik data real-time, IoT telemetry, dan pipeline deteksi penipuan (_fraud detection_).
-   Aplikasi yang menerapkan pola **CQRS (Command Query Responsibility Segregation)** dan **Event Sourcing**.

### Kapan Jangan Pakai

-   Aplikasi CRUD sederhana / internal admin dashboard yang hanya memiliki sedikit tabel database.
-   Arsitektur monolith kecil dengan tim developer terbatas di mana kompleksitas broker dan eventual consistency justru akan memperlambat rilis produk.

## 8. Metrik Kunci & Observabilitas

Untuk memonitor kesehatan arsitektur berbasis event pada Grafana / Datadog:

-   **`outbox_unprocessed_records_count`**: Jumlah record di tabel outbox yang belum terkirim ke broker; jika menumpuk, proses background publisher mengalami masalah.
-   **`event_end_to_end_latency_ms`**: Durasi total sejak event diterbitkan di producer hingga selesai dieksekusi di consumer terjauh.
-   **`schema_registry_validation_errors_total`**: Lonjakan error ketidakcocokan skema event (Avro / Protobuf / JSON Schema) saat producer merilis versi payload baru.
-   **`consumer_dead_letter_events_total`**: Jumlah event yang tidak dapat diproses oleh consumer dan dialihkan ke DLQ.
-   **`cdc_replication_lag_seconds`**: Waktu tunda pembacaan log database oleh Change Data Capture engine (misal Debezium).

## 9. Studi Kasus Nyata di Industri

### Resolusi Dual-Write dan Koordinasi Order-to-Fulfillment pada E-Commerce Skala Besar

- **Tantangan Rekayasa Riil (The Problem)**: Arsitektur monolit e-commerce yang melayani 35.000 pesanan per menit awalnya menggunakan rantai pemanggilan HTTP sinkron: Order Service memanggil Payment Gateway, lalu Inventory Service, lalu Notification Service. Ketika Inventory Service mengalami latensi tinggi (5 detik) atau downtime sesaat, request pemesanan pengguna timed out, mengakibatkan pembayaran terpotong tanpa stok berkurang atau order gagal tercatat di database (inkonsistensi status dual-write).
- **Implementasi Solusi Industri (The Architectural Solution)**: Rekayasa ulang arsitektur menjadi *Event-Driven* dengan menerapkan *Transactional Outbox Pattern*. Saat user membuat order, Order Service menulis data pesanan dan event `OrderCreated` ke PostgreSQL dalam satu transaksi ACID lokal atomik. Debezium CDC (Change Data Capture) membaca WAL log PostgreSQL dan mem-publish event tersebut ke Apache Kafka topic berpartisi secara asynchronous. Layanan Inventory, Payment, dan Notification mengonsumsi event ini secara independen dengan checkpointing offset dan tabel deduplikasi idempotensi.
- **Dampak & Hasil Sistem (The Outcome)**: Latensi p99 endpoint pembuatan pesanan terpangkas drastis dari 4.800 ms menjadi 120 ms. Kegagalan cascade di downstream service tereliminasi sepenuhnya (zero cascading failure), dan anomali dual-write order/payment turun hingga 0%. Downtime parsial pada Notification Service tidak lagi menggagalkan checkout pembeli.

## 10. Fakta Menarik & Gotchas Production: The Dual-Write Problem

Ini adalah jebakan maut paling terkenal saat berpindah ke arsitektur event-driven: **The Dual-Write Problem**.

Banyak developer pemula menulis kode seperti ini:

```go
// 1. Simpan ke Database
err := db.Save(order)

// 2. Publish ke Kafka
err := kafka.Publish("OrderPlaced", order)
```

**Petaka Terjadi:**
Jika baris 1 sukses (order tersimpan di database), tetapi server mendadak mati atau koneksi ke Kafka putus sebelum baris 2 dieksekusi, maka **Database punya data order, tetapi Kafka tidak pernah menerima event-nya!** Akibatnya: uang pengguna tidak pernah dipotong dan barang tidak pernah dikirim!

Sebaliknya, jika baris 2 ditaruh di awal, Kafka menerima event tetapi transaksi database gagal (_rollback_).

**Solusi Standar Industri: Transactional Outbox Pattern:**  
Simpan data order dan data event ke dalam tabel `outbox` di dalam **SATU TRANSAKSI DATABASE LOKAL YANG ATOMIK** (PostgreSQL Transaction). Kemudian, gunakan proses background terpisah (seperti Debezium / CDC - Change Data Capture) untuk membaca tabel outbox dan mem-publish-nya ke Kafka dengan jaminan 100% konsisten!

## 11. Rangkuman Singkat

- **Loose Temporal Coupling**: Arsitektur event-driven melepaskan ketergantungan waktu antara produsen dan konsumen; produsen memancarkan fakta (event) tanpa memedulikan kapan atau bagaimana downstream mengeksekusinya.
- **Wajib Mitigasi Dual-Write**: Jangan pernah memanggil `db.Save()` dan `broker.Publish()` secara sekuensial tanpa proteksi; gunakan *Transactional Outbox Pattern* dengan CDC (Debezium) untuk menjamin atomisitas.
- **Konsistensi Bertahap (Eventual Consistency)**: Terima bahwa data lintas sistem tidak akan instan identik; rancang alur bisnis dan antarmuka pengguna yang adaptif terhadap latensi propagasi event.
- **Desain Idempotensi di Konsumen**: Jaminan pengiriman broker hampir selalu *at-least-once*; setiap consumer wajib mengimplementasikan pengecekan ID event (deduplikasi) agar tidak memproses ulang event yang sama.
