# CQRS (Command Query Responsibility Segregation)

## 1. Analogi Sederhana & Skenario Nyata

Kalian sedang mengembangkan fitur **"Halaman Riwayat Pesanan & Laporan Transaksi"** di sebuah platform e-commerce berbasis arsitektur Microservices:

Sistem kalian menerapkan prinsip *Database-per-Service*:
- **User Service** memiliki database PostgreSQL sendiri (`users_db`) yang menyimpan profil, email, nomor telepon, dan alamat pembeli.
- **Order Service** memiliki database sendiri (`orders_db`) yang menyimpan ID pesanan, total bayar, dan status pengiriman.
- **Inventory Service** memiliki database sendiri (`inventory_db`) yang menyimpan nama produk, SKU, dan foto thumbnail.

Ketika pengguna membuka aplikasi mobile di halaman **"Daftar Riwayat Pesanan Saya"**, layar harus menampilkan:
*"Pesanan #INV-9021 oleh Budi Santoso (0812345678) — Beli 2x Mechanical Keyboard (Varian Black) — Total Rp 1.500.000 — Status: Terkirim"*.

Masalah besar muncul di level rekayasa backend:
1. **Tidak Ada SQL JOIN Lintas Database**: Karena database fisik terpisah di server berbeda, kalian **tidak bisa** menjalankan query SQL `JOIN users` atau `JOIN products`.
2. **Jebakan REST/gRPC N+1 Query**: Jika Order Service mencoba melakukan panggilan gRPC berulang-ulang ke User Service dan Inventory Service untuk setiap 20 baris pesanan di halaman riwayat, latensi halaman melambat menjadi 3–5 detik (*network overhead* masif).
3. **Konflik Kebutuhan Database**: Database Order dioptimalkan untuk transaksi tulis cepat (*Write Model* yang ternormalisasi 3NF dengan lock ketat), sedangkan halaman riwayat pesanan butuh query baca cepat (*Read Model* yang terdenormalisasi, kaya filter pencarian, dan siap saji).

Solusi arsitektur modern untuk masalah ini adalah menerapkan **CQRS (Command Query Responsibility Segregation)**:
- **Sisi Tulis (Command Side)**: Ketika Budi membuat pesanan (`POST /orders`), request ditangani oleh Write Model di PostgreSQL untuk memvalidasi aturan bisnis dan memotong saldo. Setelah sukses, sistem mem-publish domain event `OrderCreated` ke message broker (Kafka).
- **Sisi Baca (Query Side)**: Sebuah worker *Projection* mendengarkan event tersebut, mengambil data nama pembeli dan nama produk, lalu menyimpannya dalam bentuk dokumen JSON terdenormalisasi yang sudah matang (*pre-joined*) ke Read Model (misalnya Elasticsearch, MongoDB, atau PostgreSQL Read Table).
- Ketika Budi membuka halaman riwayat (`GET /orders/history`), aplikasi langsung membaca data siap saji dari Read Model dengan $O(1)$ lookup tanpa perlu SQL JOIN dan tanpa perlu memanggil User Service lagi!

---

## 2. Definisi & Konsep Inti

**CQRS (Command Query Responsibility Segregation)** adalah pola arsitektur yang secara tegas memisahkan model data dan alur pemrosesan antara operasi yang mengubah data (**Command**) dan operasi yang hanya membaca data (**Query**).

Konsep ini pertama kali dipopulerkan oleh Greg Young dan berakar dari prinsip CQS (*Command Query Separation*) ciptaan Bertrand Meyer:
1. **Command (Write Model)**:
   - Operasi yang melakukan mutasi status sistem (`POST`, `PUT`, `DELETE`).
   - Fokus pada validasi aturan bisnis yang ketat, integritas transaksional (ACID), dan biasanya menggunakan skema relasional yang ternormalisasi (*3rd Normal Form*).
   - Command tidak mengembalikan data entitas besar, melainkan hanya status keberhasilan atau ID resource yang dibuat.
2. **Query (Read Model)**:
   - Operasi yang hanya membaca data (`GET`).
   - Tidak pernah mengubah status sistem sama sekali (*zero side-effects*).
   - Menggunakan skema data terdenormalisasi (*denormalized views*) yang dioptimalkan khusus untuk kebutuhan antarmuka pengguna (UI), misalnya tabel read-only, cache Redis, atau index Elasticsearch.
3. **Projection Engine (Asynchronous Synchronizer)**:
   - Mekanisme yang menyinkronkan perubahan dari Write Model ke Read Model.
   - Biasanya bekerja secara asinkron berbasis event (*Event-Driven Architecture*) melalui Kafka atau RabbitMQ.
4. **Eventual Consistency**:
   - Karena sinkronisasi Write ke Read Model membutuhkan waktu beberapa milidetik (*lag propagation*), sistem di sisi pembacaan menganut konsistensi bertahap (*eventual consistency*).

---

## 3. Mengapa Sangat Penting di Skala Produksi

Pada skala puluhan ribu request per detik di industri teknologi modern:

1. **Rasio Baca vs Tulis yang Asimetris**: Pada sebagian besar aplikasi web (e-commerce, media sosial, streaming), rasio perbandingan baca dan tulis bisa mencapai **100:1** atau bahkan **1000:1** (1.000 user membaca katalog/feed untuk setiap 1 user yang melakukan pembelian). CQRS memungkinkan kalian menskalakan cluster Read Model secara independen tanpa membebani Write Database.
2. **Mengeliminasi Slow Database JOINs**: Query pelaporan analitik atau dashboard pencarian yang membutuhkan JOIN 8 tabel relasional yang memakan CPU database bisa digantikan dengan single document lookup di Read Model.
3. **Penyatuan Data Lintas Microservices**: Memungkinkan pembuatan *View Model* terpadu (seperti Halaman Detail Transaksi) yang menggabungkan data dari User Service, Payment Service, dan Inventory Service tanpa melanggar prinsip otonomi *Database-per-Service*.
4. **Fleksibilitas Mesin Penyimpanan (Polyglot Persistence)**: Write Model bisa menggunakan PostgreSQL (untuk transaksi ACID keuangan), sementara Read Model menggunakan Elasticsearch (untuk full-text search) atau Redis (untuk key-value lookup sub-milidetik).

---

## 4. Cara Kerja & Diagram Alur (ASCII)

Arsitektur siklus hidup CQRS dengan Event Projection:

```
                          COMMAND / WRITE PATH
[ Client: Checkout ] 
        |
        | 1. POST /orders (Command: CreateOrder)
        v
+-------------------------------------------------------------+
| COMMAND SERVICE (Write API)                                 |
|  - Validasi business logic                                  |
|  - Eksekusi transaksi tulis atomik (ACID)                   |
+-------------------------------------------------------------+
        |
        |-- 2. INSERT into `orders_write_db` (PostgreSQL 3NF)
        |
        +-- 3. Publish Event: `OrderPlacedEvent`
        v
+-------------------------------------------------------------+
| MESSAGE BROKER (Apache Kafka / RabbitMQ)                    |
+-------------------------------------------------------------+
        |
        | 4. Consume Event secara Asinkron
        v
+-------------------------------------------------------------+
| PROJECTION WORKER                                           |
|  - Ambil data snapshot user dari User Service / Cache       |
|  - Transformasi menjadi Denormalized View Model             |
|  - 5. Upsert ke Read Model                                 |
+-------------------------------------------------------------+
        |
        v
+-------------------------------------------------------------+
| READ MODEL DATABASE (Elasticsearch / Redis / Read Table)    |
|  Schema: `orders_summary_read` (Pre-joined, No SQL JOIN)    |
+-------------------------------------------------------------+
        ^
        | 6. SELECT / GET (Sub-millisecond query)
        |
+-------------------------------------------------------------+
| QUERY SERVICE (Read API)                                    |
+-------------------------------------------------------------+
        ^
        | 7. GET /api/v1/orders/my-history
[ Client: Mobile Dashboard ]
                           QUERY / READ PATH
```

---

## 5. Contoh Implementasi Nyata di Kode (Go)

Contoh berikut menunjukkan pemisahan konkret antara **Command Model (Write)** dan **Query Model (Read)** di Go, dihubungkan dengan handler proyeksi in-memory/asinkron:

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"log"
	"net/http"
	"sync"
	"time"
)

// ============================================================================
// 1. COMMAND SIDE (WRITE MODEL): Fokus pada Integritas Transaksi & Validasi
// ============================================================================

type CreateOrderCommand struct {
	OrderID  string  `json:"order_id"`
	UserID   string  `json:"user_id"`
	Amount   float64 `json:"amount"`
	ItemName string  `json:"item_name"`
}

type OrderWriteModel struct {
	mu     sync.Mutex
	orders map[string]CreateOrderCommand // Simulasi tabel relasional Write (3NF)
}

var writeDB = &OrderWriteModel{orders: make(map[string]CreateOrderCommand)}

// Event yang dipublish saat status berhasil dimutasi
type OrderPlacedEvent struct {
	OrderID   string    `json:"order_id"`
	UserID    string    `json:"user_id"`
	Amount    float64   `json:"amount"`
	ItemName  string    `json:"item_name"`
	Timestamp time.Time `json:"timestamp"`
}

// Simulasi Message Bus (misal Kafka)
var eventBus = make(chan OrderPlacedEvent, 100)

func HandleCreateOrder(w http.ResponseWriter, r *http.Request) {
	var cmd CreateOrderCommand
	if err := json.NewDecoder(r.Body).Decode(&cmd); err != nil {
		http.Error(w, "Invalid JSON", http.StatusBadRequest)
		return
	}

	// Validasi aturan bisnis di sisi Write
	if cmd.Amount <= 0 {
		http.Error(w, "Amount must be positive", http.StatusBadRequest)
		return
	}

	writeDB.mu.Lock()
	writeDB.orders[cmd.OrderID] = cmd
	writeDB.mu.Unlock()

	// Publish event ke broker untuk memperbarui Read Model
	eventBus <- OrderPlacedEvent{
		OrderID:   cmd.OrderID,
		UserID:    cmd.UserID,
		Amount:    cmd.Amount,
		ItemName:  cmd.ItemName,
		Timestamp: time.Now(),
	}

	w.WriteHeader(http.StatusAccepted)
	w.Write([]byte(`{"status":"COMMAND_ACCEPTED"}`))
}

// ============================================================================
// 2. QUERY SIDE (READ MODEL): Denormalized View (Pre-Joined, Siap Saji)
// ============================================================================

// OrderDetailView adalah model baca yang sudah menggabungkan data user & produk
type OrderDetailView struct {
	OrderID      string  `json:"order_id"`
	CustomerName string  `json:"customer_name"` // Didapat dari User Service
	CustomerTier string  `json:"customer_tier"` // Didapat dari Loyalty Service
	ItemSummary  string  `json:"item_summary"`
	TotalAmount  float64 `json:"total_amount"`
	FormattedDate string `json:"formatted_date"`
}

type OrderReadStore struct {
	mu    sync.RWMutex
	views map[string]OrderDetailView // Simulasi Elasticsearch / Redis Read Cache
}

var readStore = &OrderReadStore{views: make(map[string]OrderDetailView)}

// Worker Projection yang mendengarkan event dan menyusun Read Model
func StartProjectionWorker() {
	go func() {
		for event := range eventBus {
			// Simulasi lookup profil user sekali saja saat event tiba
			userName := "Budi Santoso"
			userTier := "Gold VIP"

			// Bentuk dokumen denormalisasi siap saji
			view := OrderDetailView{
				OrderID:      event.OrderID,
				CustomerName: userName,
				CustomerTier: userTier,
				ItemSummary:  event.ItemName,
				TotalAmount:  event.Amount,
				FormattedDate: event.Timestamp.Format("02 Jan 2006 15:04"),
			}

			readStore.mu.Lock()
			readStore.views[event.OrderID] = view
			readStore.mu.Unlock()
			log.Printf("[CQRS-Projection] Updated Read Model for order: %s", event.OrderID)
		}
	}()
}

// Query Handler: Cepat, nol kalkulasi berat, nol SQL JOIN
func HandleGetOrderDetail(w http.ResponseWriter, r *http.Request) {
	orderID := r.URL.Query().Get("id")

	readStore.mu.RLock()
	view, found := readStore.views[orderID]
	readStore.mu.RUnlock()

	if !found {
		http.Error(w, `{"error":"Order not found in Read View"}`, http.StatusNotFound)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(view)
}

func main() {
	// Menjalankan Projection Worker di background
	StartProjectionWorker()

	http.HandleFunc("/api/orders/create", HandleCreateOrder) // Command Endpoint (Write)
	http.HandleFunc("/api/orders/view", HandleGetOrderDetail) // Query Endpoint (Read)

	log.Println("CQRS Server running on :8080...")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Performa Query Super Cepat**: Pembacaan tidak memerlukan kalkulasi atau SQL JOIN yang rumit karena data sudah disiapkan (*pre-calculated & denormalized*) di Read Model.
- **Penskalaan Independen (Independent Autoscaling)**: Server pembacaan (Read Service) dapat di-scale hingga 100 replika di Kubernetes untuk menyerap lonjakan traffic browsing tanpa membebani database utama.
- **Skema Khusus Sesuai Kebutuhan UI**: Read Model dapat dioptimalkan persis sesuai kebutuhan tampilan mobile app tanpa mengorbankan normalisasi tabel transaksi keuangan.
- **Isolasi Beban Kerja (Workload Isolation)**: Query analitik yang berat di sisi Read tidak akan pernah mengunci (*table lock*) transaksi checkout di sisi Write.

### Kekurangan
- **Eventual Consistency Lag**: Ada jeda waktu (beberapa milidetik s.d. detik) antara saat transaksi ditulis di Write Model hingga muncul di Read Model. Pengguna yang me-refresh halaman instan bisa mendapati data belum terupdate (*lag window*).
- **Kompleksitas Kode & Duplikasi Penyimpanan**: Memerlukan pengelolaan dua skema data, worker sinkronisasi, dan message broker. Biaya penyimpanan data bertambah karena data diduplikasi (*denormalized storage overhead*).
- **Risiko Kegagalan Proyeksi**: Jika Projection Worker mengalami bug atau antrean Kafka macet, Read Model akan menyajikan data basi (*stale view*).

---

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- **Rasio Trafik Baca Jauh Lebih Tinggi Dibanding Tulis**: Sistem di mana ribuan pengguna membaca data secara masif dibanding sedikit mutasi tulis (katalog produk, feed media sosial, riwayat transaksi).
- **Kebutuhan Menggabungkan Data Lintas Microservices**: Ketika beberapa database microservice yang terisolasi perlu disajikan bersamaan dalam satu tampilan dashboard.
- **Pencarian & Pelaporan Kompleks**: Saat data transaksi relasional di PostgreSQL perlu diekspos ke Elasticsearch untuk pencarian teks penuh dan filter multi-kolom yang rumit.

### Kapan Jangan Pakai
- **Aplikasi CRUD Sederhana**: Jika aplikasi kalian hanya sistem administrasi internal sederhana di mana beban baca dan tulis seimbang dan tabel database masih sedikit. Menerapkan CQRS di sini adalah *over-engineering* parah.
- **Operasi yang Menuntut Konsistensi Kuat (*Read-Your-Own-Writes Mandate*)**: Pada alur transfer perbankan real-time di mana saldo harus langsung terbaca akurat di milidetik yang sama tanpa toleransi *eventual consistency lag*.

---

## 8. Metrik Kunci & Observabilitas

1. **Projection Lag / Replication Lag (`cqrs_projection_lag_seconds`)**: Waktu yang dibutuhkan dari saat event dibuat di Write Model hingga berhasil disimpan di Read Model (target produksi: p99 < 500ms).
2. **Read vs Write Throughput Ratio (`cqrs_requests_total{type="read|write"}`)**: Memantau proporsi request untuk menentukan rasio autoscaling pod Query vs Command.
3. **Projection Failure / DLQ Counter (`cqrs_projection_errors_total`)**: Jumlah event mutasi yang gagal diproyeksikan ke Read Model akibat bug parsing atau database read down.
4. **Read Cache Hit Rate (`cqrs_read_store_hits_total`)**: Persentase query yang berhasil dilayani langsung dari Read Model tanpa fallback.

---

## 9. Studi Kasus Nyata di Industri

### Skenario: Uber — Halaman Riwayat Perjalanan & Pelaporan Driver

#### Problem
Uber memiliki ribuan microservices. Layanan perjalanan (*Trip Service*) memproses jutaan perjalanan harian di PostgreSQL dengan model relasional ketat. Sementara itu, jutaan pengemudi dan penumpang setiap detik membuka aplikasi untuk melihat **"Riwayat Perjalanan & Rincian Tarif"**.
- Untuk merender 1 baris riwayat, sistem butuh data dari *Driver Service* (nama & foto driver), *Vehicle Service* (plat nomor & tipe mobil), *Payment Service* (rincian kartu/cash), dan *Rating Service*.
- Jika setiap kali driver membuka tab riwayat sistem melakukan RPC panggilan ke 4 microservices tersebut, jaringan internal Uber akan kolaps karena miliaran panggilan RPC redundan.

#### Solusi Arsitektur
Uber menerapkan pola **CQRS**:
1. Sisi Command: *Trip Service* hanya fokus mencatat koordinat GPS dan status perjalanan di database operasional.
2. Saat perjalanan selesai, event `TripCompleted` disiarkan melalui Apache Kafka.
3. *Projection Pipeline* membaca event tersebut, mengambil snapshot nama driver, tipe mobil, dan tarif akhir, lalu menyusunnya menjadi satu dokumen utuh di Read Store terpusat (Docstore / Schemaless).
4. Layanan riwayat penumpang dan driver murni melakukan query baca ke Read Store tersebut secara mandiri dengan latensi rendah (< 15ms).

#### Hasil / Dampak
- **Mengeliminasi Beban Database Operasional**: 100% beban query riwayat jutaan pengguna dialihkan dari database transaksional utama.
- **Toleransi Kegagalan Tinggi**: Meskipun Driver Service atau Rating Service sedang mengalami downtime sejenak, pengguna tetap bisa melihat seluruh riwayat perjalanan masa lalu tanpa gangguan.

---

## 10. Fakta Menarik & Gotchas Production

1. **CQRS Tidak Wajib Pakai Event Sourcing**: Banyak developer mengira jika memakai CQRS, mereka wajib memakai *Event Sourcing* (menyimpan state sebagai deretan event permanen). Padahal tidak! CQRS murni hanya memisahkan jalur baca dan tulis. Kalian bisa menggunakan tabel PostgreSQL biasa untuk Write Model dan Elasticsearch untuk Read Model.
2. **Masalah "Read-Your-Own-Writes" di Frontend**: Pengguna menekan tombol "Simpan Alamat", diarahkan kembali ke halaman profil, tapi alamat baru belum muncul karena adanya *projection lag* 200ms di Read Model. Solusi: Gunakan optimisme UI di frontend, atau sertakan flag sementara untuk membaca langsung dari Write Model khusus untuk request pertama setelah mutasi.
3. **Kemudahan Re-Indexing (Replay Projections)**: Jika kelak tim produk ingin menambahkan informasi baru pada tampilan riwayat (misalnya menambahkan kolom "Status Cashback"), kalian cukup me-replay ulang event historis dari Kafka ke worker projection baru untuk membangun Read Model baru dari nol tanpa perlu migrasi skema database relasional utama.

---

## 11. Rangkuman Singkat

1. **Prinsip Dasar**: Memisahkan secara fisik antara alur pemrosesan mutasi data (**Command / Write**) dan alur pembacaan data (**Query / Read**).
2. **Penyelesai Masalah Microservices**: Menghilangkan kebutuhan SQL JOIN lintas database dengan membuat model data baca yang sudah terdenormalisasi (*pre-joined*) di sisi Query.
3. **Penyelarasan Beban Kerja**: Mengizinkan penskalaan masif pada sistem pembacaan yang memiliki rasio baca tinggi tanpa membebani database transaksional utama.
4. **Konsekuensi Utama**: Menerima konsistensi bertahap (*eventual consistency*) dan adanya *lag* sinkronisasi antara event tulis dan view baca.
