# Event-Driven Architecture (EDA)

## 1. Skenario Sistem Nyata

Pada sistem transportasi online (*ride-hailing*), saat penumpang memesan dan pengemudi menerima pesanan, terjadi sebuah peristiwa bisnis (*state change*) yang tidak dapat diubah: **`RideAssigned`**.

Peristiwa ini memicu serangkaian tindakan otonom di berbagai subsistem backend:
- `Navigation Service` menghitung rute optimal dan mengalirkan panduan jalan ke ponsel pengemudi.
- `Pricing & Payment Service` mengunci saldo dompet digital atau pre-authorization kartu kredit penumpang.
- `Safety Monitoring Service` mulai memantau koordinat GPS secara real-time untuk mendeteksi deviasi rute darurat.
- `Notification Service` mengirimkan pesan push ke ponsel penumpang yang berisi nomor plat kendaraan dan nama pengemudi.

Penumpang dan pengemudi tidak pernah memanggil API billing atau safety monitoring secara langsung. Masing-masing microservice bereaksi secara mandiri terhadap aliran fakta bisnis (*immutable event stream*).

Jika sistem ini dibangun menggunakan arsitektur tradisional berbasis REST sinkron (*Request-Driven*):
- Service Pemesanan harus memanggil 4 service lainnya secara berantai (`Ride Svc -> Nav Svc -> Payment Svc -> Safety Svc`).
- Jika `Safety Service` mengalami latensi atau kehabisan memori, seluruh proses penjemputan penumpang akan macet dan gagal (*deep synchronous coupling*).

**Solusi dengan Event-Driven Architecture:**
Sistem melepaskan ketergantungan panggilan langsung. Service hanya mempublikasikan event ke event bus terdistribusi (misal: Apache Kafka). Setiap service consumer membaca stream event tersebut secara mandiri dan memperbarui *read model* lokalnya tanpa memblokir alur pemesanan utama.

## 2. Apa Itu dan Buat Apa?

Event-Driven Architecture (Arsitektur Berbasis Event) adalah paradigma desain perangkat lunak di mana aliran eksekusi sistem digerakkan oleh **pembentukan (*production*), deteksi (*detection*), dan konsumsi (*consumption*) peristiwa bisnis (*Events*)**.

### Perbedaan Mendasar: Command vs Event
- **Command (Perintah)**: Tindakan yang *diminta* kepada target spesifik. Bersifat imperatif dan bisa ditolak.  
  *Contoh:* `ChargeCreditCard(amount: $50)` -> Bisa gagal jika kartu ditolak.
- **Event (Peristiwa)**: Fakta sejarah tentang sesuatu yang *telah terjadi* di masa lalu. Bersifat objektif, tidak dapat diubah (*immutable*), dan tidak peduli siapa yang mendengarkan.  
  *Contoh:* `OrderPlaced`, `PaymentSucceeded`, `UserAccountRegistered`.

Komponen utama dalam EDA:
- **Event Producer**: Service yang mendeteksi perubahan status bisnis dan menerbitkan fakta tersebut ke event stream.
- **Event Broker / Stream**: Saluran terdistribusi yang menyimpan dan mengalirkan event (misal: Apache Kafka, AWS Kinesis).
- **Event Consumer**: Service-service independen yang bereaksi terhadap event tersebut untuk menjalankan tugasnya sendiri.

## 3. Kenapa Harus Pakai Ini di High-Throughput?

Dalam arsitektur tradisional berbasis REST/RPC (*Request-Driven Architecture*), service saling memanggil secara berantai (*deep synchronous call trees*):
`API Gateway -> Order Svc -> Payment Svc -> Inventory Svc -> Notification Svc`.

Jika satu service di tengah rantai mengalami lonjakan trafik atau latensi, seluruh rantai di depannya akan macet (*tight coupling*).

Dalam Event-Driven Architecture:
- Seluruh komunikasi bersifat **asynchronous dan non-blocking**.
- Setiap service memiliki salinan data lokal (*Read Model*) yang diproyeksikan dari riwayat event, sehingga service dapat melayani query pengguna tanpa perlu memanggil service lain melalui jaringan (*zero inter-service synchronous network hops*).
- Sistem dapat diskalakan secara masif ke jutaan event per detik tanpa saling mengunci resource database.

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

Implementasi pola **Transactional Outbox Pattern** untuk mencegah *dual-write loss* antara database dan message broker di Go:

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
- **Skalabilitas & Throughput Maksimal**: Komponen produser dan konsumen berjalan dengan kecepatan independen tanpa saling memblokir.
- **Extreme Fault Tolerance**: Jika Notification Service mati selama 2 jam, Order Service tetap berjalan normal. Ketika Notification Service hidup kembali, ia tinggal membaca tumpukan event yang tertinggal dari Kafka.
- **Auditability Sempurna**: Riwayat event berfungsi sebagai catatan sejarah lengkap (*audit log*) yang mencatat setiap perubahan status sistem sejak hari pertama.

### Kekurangan
- **Eventual Consistency**: Sistem tidak konsisten secara instan. Data saldo di layar pengguna mungkin butuh 1-2 detik untuk sinkron dengan database backend.
- **Kompleksitas Schema Evolution**: Jika struktur event JSON berubah (misal menghapus field atau mengubah tipe data), kamu berisiko merusak seluruh consumer lama (*breaking contract*).
- **Sulitnya Tracing & Debugging**: Melacak akar masalah saat sebuah transaksi gagal membutuhkan sistem observabilitas terdistribusi yang matang (*Correlation ID* di setiap event).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Sistem terdistribusi skala besar dengan banyak tim dan domain bisnis berbeda.
- Platform analitik data real-time, IoT telemetry, dan pipeline deteksi penipuan (*fraud detection*).
- Aplikasi yang menerapkan pola **CQRS (Command Query Responsibility Segregation)** dan **Event Sourcing**.

### Kapan Jangan Pakai
- Aplikasi CRUD sederhana / internal admin dashboard yang hanya memiliki sedikit tabel database.
- Arsitektur monolith kecil dengan tim developer terbatas di mana kompleksitas broker dan eventual consistency justru akan memperlambat rilis produk.

## 8. Metrik Kunci & Observabilitas

Untuk memonitor kesehatan arsitektur berbasis event pada Grafana / Datadog:

- **`outbox_unprocessed_records_count`**: Jumlah record di tabel outbox yang belum terkirim ke broker; jika menumpuk, proses background publisher mengalami masalah.
- **`event_end_to_end_latency_ms`**: Durasi total sejak event diterbitkan di producer hingga selesai dieksekusi di consumer terjauh.
- **`schema_registry_validation_errors_total`**: Lonjakan error ketidakcocokan skema event (Avro / Protobuf / JSON Schema) saat producer merilis versi payload baru.
- **`consumer_dead_letter_events_total`**: Jumlah event yang tidak dapat diproses oleh consumer dan dialihkan ke DLQ.
- **`cdc_replication_lag_seconds`**: Waktu tunda pembacaan log database oleh Change Data Capture engine (misal Debezium).

## 9. Studi Kasus Nyata di Industri

- **Netflix Media Processing Pipeline**: Netflix mengelola ribuan transkoding video, pembuatan thumbnail, dan subtitle melalui arsitektur berbasis event. Ketika studio memuat file master film baru, serangkaian event independen memicu worker rendering di berbagai region cloud tanpa ada dependensi sinkron antar service.
- **DoorDash Real-Time Driver Dispatch**: DoorDash menggunakan event streams untuk mengalirkan update GPS kurir dan status pesanan restoran. Algoritma pencocokan pengemudi mendengarkan aliran event secara real-time untuk memasangkan kurir terdekat dengan restoran yang makanannya hampir selesai dimasak.

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

Sebaliknya, jika baris 2 ditaruh di awal, Kafka menerima event tetapi transaksi database gagal (*rollback*).

**Solusi Standar Industri: Transactional Outbox Pattern:**  
Simpan data order dan data event ke dalam tabel `outbox` di dalam **SATU TRANSAKSI DATABASE LOKAL YANG ATOMIK** (PostgreSQL Transaction). Kemudian, gunakan proses background terpisah (seperti Debezium / CDC - Change Data Capture) untuk membaca tabel outbox dan mem-publish-nya ke Kafka dengan jaminan 100% konsisten!
