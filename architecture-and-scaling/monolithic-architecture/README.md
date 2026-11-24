# Monolithic Architecture

## 1. Analogi Sederhana & Skenario Nyata

Bayangkan kalian membuka sebuah kafe kopi kecil:

Semua aktivitas berada dalam satu ruangan terbuka: kasir menerima uang, barista menyeduh espresso, pemanggang menghangatkan roti, dan manajer mencatat pembukuan harian di buku kas yang sama di atas meja kasir. 
- Saat pesanan baru masuk, kasir cukup berteriak ke barista: *"Americano satu!"*. Tidak ada kurir, tidak ada panggilan telepon, komunikasi terjadi seketika dalam hitungan detik (*in-memory*).
- Jika kalian ingin merenovasi atau memindahkan kafe, kalian hanya perlu memindahkan satu ruangan tersebut.

Inilah gambaran **Monolithic Architecture** (Arsitektur Monolit). Semua fitur aplikasi kalian (Autentikasi, Katalog, Keranjang Belanja, Pembayaran, Notifikasi) berada dalam satu basis kode (*single codebase*), dikompilasi menjadi satu berkas binary executable tunggal, dan dijalankan dalam satu proses sistem operasi yang mengakses satu database utama.

---

## 2. Definisi & Konsep Inti

**Monolithic Architecture** adalah pola perancangan perangkat lunak tradisional di mana seluruh komponen fungsional sistem digabungkan dan di-deploy bersamaan sebagai **satu unit eksekusi tunggal (*single deployable unit*)**.

Karakteristik utama arsitektur monolit:
1. **Single Runtime Process**: Aplikasi berjalan sebagai satu proses sistem operasi (misalnya satu binary Go `app` atau kontainer Docker tunggal).
2. **In-Memory Communication**: Pemanggilan antar-fitur (misal fungsi Checkout memanggil fungsi Payment) dilakukan melalui panggilan fungsi langsung di memori (*in-memory function call*) dengan latensi sangat rendah ($< 1\,\mu\text{s}$).
3. **Shared Database & ACID Transactions**: Semua domain bisnis membaca dan menulis ke satu basis data relasional yang sama (misalnya PostgreSQL atau MySQL), memungkinkan transaksi atomik ACID yang sederhana via `BEGIN ... COMMIT`.
4. **Unified Build & Deployment Pipeline**: Seluruh kode diuji, dibuild, dan dirilis bersamaan melalui satu pipeline CI/CD tunggal.

---

## 3. Mengapa Sangat Penting di Skala Produksi

Di tengah tren microservices yang rumit, arsitektur monolit tetap menjadi pilihan utama bagi banyak perusahaan sukses (seperti Stack Overflow, GitHub awal, dan Basecamp):

1. **Kecepatan Iterasi Awal (Product-Market Fit)**: Pada fase awal startup, perubahan model bisnis terjadi setiap minggu. Pada monolit, refactoring fungsi, penggantian skema tabel, atau penambahan fitur baru bisa diselesaikan dalam hitungan menit tanpa repot mengoordinasikan breaking API changes lintas layanan.
2. **Nol Overhead Jaringan (Zero Network Latency)**: Tidak ada latensi serialisasi JSON/Protobuf, tidak ada handshake TLS, dan tidak ada kegagalan parsial jaringan (*network partitions*).
3. **Operasional Infrastruktur Minimal**: Tidak membutuhkan kluster Kubernetes yang mahal, service mesh (Istio), atau sistem distributed tracing yang rumit. Satu atau dua server virtual (VM) berkekuatan tinggi sudah cukup untuk melayani jutaan pengguna.
4. **Kemudahan Tracing & Debugging**: Saat terjadi error, *stack trace* utuh tercetak dalam satu log file. Engineer dapat dengan mudah men-debug menggunakan breakpoint lokal di IDE.

---

## 4. Cara Kerja & Diagram Alur (ASCII)

Struktur internal dan alur eksekusi sebuah aplikasi Monolitik:

```
[ Web Browser / Mobile Client ]
              |
              | 1. HTTP Request (POST /checkout)
              v
+-----------------------------------------------------------------------+
| MONOLITHIC SERVER PROCESS (Single Go Binary / Single OS Process)     |
|                                                                       |
|  [ HTTP Router / Handler ]                                            |
|        |                                                              |
|        | 2. Direct Function Call (In-Memory, ~10 nanoseconds)         |
|        v                                                              |
|  [ Order Service Domain ]                                             |
|        |                                                              |
|        | 3. Direct Function Call (In-Memory, zero network hop)        |
|        v                                                              |
|  [ Payment Service Domain ]                                           |
|        |                                                              |
|        | 4. Direct Function Call (In-Memory)                          |
|        v                                                              |
|  [ Inventory & Notification Domain ]                                  |
|        |                                                              |
|        +--- 5. Single Database Transaction (ACID) ----------------+   |
|                                                                   |   |
+-------------------------------------------------------------------|---+
                                                                    v
                                             +-------------------------------+
                                             | Single PostgreSQL Database    |
                                             |   (orders, users, payments)   |
                                             +-------------------------------+
```

---

## 5. Contoh Implementasi Nyata di Kode (Go)

Contoh berikut menunjukkan bagaimana sebuah aplikasi Monolit Go menangani alur Checkout dan Pembayaran dalam satu proses dengan pemanggilan fungsi in-memory dan transaksi database lokal:

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"net/http"
	"time"
)

// MonolithApp menggabungkan seluruh komponen bisnis dalam satu struct tunggal.
type MonolithApp struct {
	db *sql.DB
}

func (app *MonolithApp) ProcessPayment(ctx context.Context, tx *sql.Tx, orderID string, amount float64) error {
	// Pemanggilan fungsi in-memory langsung tanpa koneksi jaringan HTTP/gRPC
	log.Printf("[Monolith-Payment] Charging %.2f for order %s", amount, orderID)
	
	// Eksekusi potong saldo dalam transaksi database yang sama
	_, err := tx.ExecContext(ctx, "UPDATE wallets SET balance = balance - $1 WHERE user_id = $2", amount, "user_1")
	return err
}

func (app *MonolithApp) CheckoutHandler(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
	defer cancel()

	// 1. Memulai transaksi ACID database tunggal yang mencakup order & payment
	tx, err := app.db.BeginTx(ctx, nil)
	if err != nil {
		http.Error(w, "Database error", http.StatusInternalServerError)
		return
	}
	defer tx.Rollback()

	orderID := "ord_9901"
	amount := 250000.0

	// 2. Simpan order
	_, err = tx.ExecContext(ctx, "INSERT INTO orders (id, amount, status) VALUES ($1, $2, 'PENDING')", orderID, amount)
	if err != nil {
		http.Error(w, "Failed to create order", http.StatusInternalServerError)
		return
	}

	// 3. Panggil domain payment secara in-memory
	if err := app.ProcessPayment(ctx, tx, orderID, amount); err != nil {
		http.Error(w, "Payment failed", http.StatusBadRequest)
		return
	}

	// 4. Commit transaksi sekaligus (Atomik: kedua operasi sukses atau gagal total)
	if err := tx.Commit(); err != nil {
		http.Error(w, "Commit failed", http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusCreated)
	fmt.Fprintf(w, `{"status":"SUCCESS","order_id":"%s"}`, orderID)
}

func main() {
	// Inisialisasi aplikasi monolit
	app := &MonolithApp{}
	http.HandleFunc("/api/checkout", app.CheckoutHandler)

	log.Println("Monolith server running on :8080...")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Kesederhanaan Pengembangan (Simplicity)**: Sangat mudah dijalankan di laptop lokal developer (`go run main.go`). Tidak perlu menjalankan 15 docker container hanya untuk mengetes satu fitur.
- **Performa Komunikasi Maksimal**: Tidak ada latensi jaringan antar-modul; transmisi data terjadi di level CPU register dan RAM.
- **Integritas Transaksional (ACID)**: Menjamin konsistensi data kuat (*strong consistency*) menggunakan transaksi database relasional standar tanpa perlu pola rumit seperti Saga atau Two-Phase Commit (2PC).
- **Biaya Operasional Rendah**: Membutuhkan sedikit server, memangkas tagihan cloud, dan mengurangi kebutuhan tim DevOps/SRE khusus.

### Kekurangan
- **Kerapuhan Terhadap Bug Kritis (Single Point of Failure)**: Satu bug fatal (misalnya *nil-pointer dereference* yang tidak di-recover atau kebocoran memori di modul notifikasi) dapat meruntuhkan seluruh proses server, mematikan fitur login dan checkout sekaligus.
- **Antrean Rilis (Release Bottleneck)**: Jika puluhan developer bekerja di repositori yang sama, mereka harus menunggu antrean testing CI/CD dan koordinasi rilis yang panjang.
- **Skalabilitas Tidak Efisien (Uniform Scaling)**: Jika modul transcoding video membutuhkan CPU 100%, kalian harus menduplikasi seluruh monolit ke server baru, termasuk modul-modul lain yang sebenarnya tidak membutuhkan resource tambahan.

---

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- **Tahap Awal Startup (Pre-Seed s.d. Seri A)**: Fokus utama adalah menemukan kecocokan produk dengan pasar (*Product-Market Fit*) secepat mungkin.
- **Tim Pengembang Berukuran Kecil (< 20 Engineer)**: Semua engineer dapat berkomunikasi langsung tanpa hambatan birokrasi organisasi.
- **Aplikasi Bisnis dengan Integritas Data Ketat**: Sistem perbankan inti, akuntansi, atau platform e-commerce awal yang membutuhkan jaminan ACID mutlak.

### Kapan Jangan Pakai
- **Organisasi Raksasa (> 150-200 Engineer)**: Tim mulai saling bertabrakan (*merge conflict* masif, antrean rilis berhari-hari).
- **Beban Kerja Asimetris Ekstrem**: Sistem memiliki satu modul komputasi intensif (misal AI machine learning inference atau image rendering) yang memerlukan kluster GPU khusus.

---

## 8. Metrik Kunci & Observabilitas

1. **Process Heap & Memory Consumption (`go_memstats_alloc_bytes`)**: Memantau kebocoran memori dalam proses aplikasi monolit.
2. **Goroutine Count (`go_goroutines`)**: Memastikan tidak ada goroutine menggantung (*goroutine leak*) yang memakan resource server secara perlahan.
3. **Database Connection Pool Saturation (`db_open_connections`)**: Karena seluruh fitur berbagi satu database pool, pantau utilisasi koneksi agar tidak terjadi *connection starvation*.
4. **Mean Time to Deploy (MTTD)**: Waktu yang dibutuhkan untuk menjalankan build pipeline monolit dari commit hingga live di server.

---

## 9. Studi Kasus Nyata di Industri

### Skenario: Stack Overflow — Melayani Ratusan Juta Pengguna dengan Monolit

#### Problem
Stack Overflow adalah salah satu website dengan lalu lintas terpadat di dunia, melayani lebih dari 100 juta pengunjung unik bulanan dan miliaran *page views*. Banyak pakar arsitektur menyarankan mereka memecah sistem menjadi ratusan microservices agar mampu menangani lonjakan trafik raksasa tersebut.

#### Solusi Arsitektur
Stack Overflow secara sadar memilih **tetap mempertahankan arsitektur monolitik**:
- Aplikasi inti ditulis sebagai satu monolit C# / .NET yang di-deploy ke segelintir server fisik bare-metal (hanya sekitar 9 server web on-premise).
- Mengoptimalkan efisiensi kode, in-memory caching, dan query SQL PostgreSQL/SQL Server performan tinggi.

#### Hasil / Dampak
- **Infrastruktur Super Efisien**: Melayani miliaran request bulanan hanya dengan segelintir server fisik, dengan utilisasi CPU rata-rata di bawah 15%.
- **Biaya Operasional Rendah**: Biaya server Stack Overflow hanya sepersekian kecil dibanding perusahaan teknologi berukuran setara yang menggunakan ratusan pod microservices di cloud.
- **Kecepatan Respons Ekstrem**: Latensi render halaman stabil di bawah 20 milidetik karena tidak ada *network hops* internal antarlayanan.

---

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan "Big Ball of Mud"**: Masalah utama monolit bukanlah arsitekturnya, melainkan disiplin kode. Tanpa pembatasan modul yang tegas, kode antar-folder akan saling panggil secara serampangan hingga menjadi kode spageti yang mustahil di-refactor.
2. **Horizontal Scaling Monolit Sangat Mudah**: Banyak junior developer salah mengira monolit tidak bisa di-*scale*. Faktanya, monolit *stateless* sangat mudah di-scale: cukup letakkan NGINX/HAProxy di depan, lalu jalankan 10 instance server monolit di belakangnya.
3. **Bahaya Single Shared DB Pool**: Jika fitur ekspor laporan bulanan yang lambat menghabiskan 50 koneksi database yang tersedia di pool monolit, pengguna yang ingin login di beranda akan mengalami timeout karena tidak kebagian koneksi database.

---

## 11. Rangkuman Singkat

1. **Definisi Inti**: Seluruh fitur aplikasi dikompilasi dan di-deploy bersamaan sebagai satu unit runtime tunggal yang mengakses satu database utama.
2. **Keunggulan Utama**: Kesederhanaan operasional, biaya cloud murah, performa fungsi in-memory instan ($< 1\,\mu\text{s}$), dan integritas transaksi ACID mutlak.
3. **Tantangan Terbesar**: Kerapuhan terhadap crash fatal (*single point of failure*) dan antrean rilis panjang saat tim membesar (> 50-100 engineer).
4. **Prinsip Praktis**: Monolit adalah arsitektur default terbaik untuk memulai proyek baru. Jangan terburu-buru memecah sistem sebelum organisasi atau domain bisnis benar-benar menuntutnya.
