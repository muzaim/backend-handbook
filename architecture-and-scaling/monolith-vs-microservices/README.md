# Monolith, Modular Monolith, dan Microservices

## 1. Analogi Sederhana & Skenario Nyata

Kalian sedang mengelola sebuah restoran:

1. **Traditional Monolith (Dapur Terbuka Satu Ruangan Tanpa Sekat)**: Semua koki, kasir, dan pelayan bekerja di satu ruangan yang sama. Semua orang memakai kulkas yang sama dan saling teriak langsung saat ada pesanan baru. Sangat cepat dan murah saat pelanggan masih puluhan per hari. Namun, ketika restoran mulai mempekerjakan 50 koki, mereka mulai saling senggol, bumbu saling tertukar, dan jika kompor gas meledak, seluruh restoran langsung ditutup total.
2. **Modular Monolith (Dapur Satu Gedung dengan Ruangan & Sekat Tertata)**: Semua koki tetap bekerja di dalam satu gedung restoran yang sama (satu izin sewa, satu listrik), tetapi gedungnya memiliki dinding pembatas yang rapi: ada *Station Pastry*, *Station Grill*, dan *Station Minuman*. Masing-masing memiliki lemari bahan sendiri dan hanya boleh berkomunikasi lewat jendela pesanan resmi (*interface*). Jika bagian *Grill* kehabisan garam, mereka tidak boleh sembarangan mencomot garam dari *Pastry*.
3. **Microservices (Warung Terpisah di Berbagai Sudut Kota)**: Dapur roti ada di Jakarta Selatan, dapur grill ada di Jakarta Barat, dan kasir ada di Jakarta Pusat. Setiap cabang memiliki bangunan, tagihan listrik, kulkas, dan staf sendiri. Jika ada pesanan paket komplit, kasir harus memesan kurir ojek online (koneksi jaringan HTTP/gRPC) untuk mengantarkan roti ke dapur grill. Kelebihannya: jika dapur roti kebakaran, dapur grill tetap bisa melayani steak. Kekurangannya: biaya sewa naik 10x lipat, waktu saji melambat karena macet di jalan (*network latency*), dan mengoordinasikan pesanan menjadi mimpi buruk logistik.

Di dunia industri, perdebatan arsitektur bukan tentang "mana yang paling modern", melainkan tentang **menyelaraskan ukuran tim, kecepatan iterasi, kompleksitas domain bisnis, dan biaya operasional**.

---

## 2. Definisi & Konsep Inti

Berikut adalah ketiga model arsitektur perangkat lunak backend:

### A. Traditional Monolith
Seluruh kapabilitas bisnis (Auth, Order, Catalog, Billing, Notification) ditulis dalam satu codebase tunggal, dikompilasi menjadi satu binary/artefak deployment, dan mengakses satu database bersama tanpa batas modul yang tegas.
- **Karakteristik**: Semua pemanggilan fungsi bersifat in-memory ($< 1\,\mu\text{s}$), deployment sederhana (`docker run myapp`), tetapi rawan berubah menjadi *"Big Ball of Mud"* akibat ketergantungan kode yang saling silang (*tight coupling*).

### B. Modular Monolith
Seluruh kapabilitas bisnis tetap berada di dalam satu codebase dan di-deploy sebagai **satu unit runtime tunggal**, namun kode dipecah ke dalam modul-modul domain yang terisolasi secara ketat (*bounded contexts*):
- **Boundary Enforced**: Modul `Order` tidak boleh langsung mengakses tabel database milik modul `Billing` atau mengimpor struct internal `Billing`. Komunikasi antar modul hanya boleh melalui antarmuka publik Go (`interface`) atau in-memory domain events.
- **Karakteristik**: Mempertahankan keunggulan performa in-memory call dan kesederhanaan deployment monolit, namun memiliki kerapian domain dan kemudahan migrasi seperti microservices jika kelak perlu dipecah.

### C. Microservices
Sistem dipecah menjadi kumpulan service independen berukuran kecil yang masing-masing memiliki proses runtime terpisah, repositori sendiri (atau mono-repo terisolasi), pipeline CI/CD sendiri, dan **database sendiri (*Database-per-Service*)**.
- **Karakteristik**: Komunikasi wajib melalui jaringan (REST, gRPC, atau message broker Kafka/RabbitMQ). Mengizinkan penskalaan independen dan otonomi tim skala besar (ratusan engineer), namun membawa kompleksitas sistem terdistribusi (latency, *network partitions*, *eventual consistency*, dan *distributed tracing*).

---

## 3. Mengapa Sangat Penting di Skala Produksi

Kesalahan memilih arsitektur adalah salah satu penyebab kebangkrutan startup teknologi:

1. **Premature Microservices Trap**: Banyak startup tahap awal dengan 5-10 engineer langsung mengadopsi puluhan microservices karena tergiur teknologi Netflix atau Uber. Akibatnya, 70% waktu engineer habis mengurus Kubernetes, Helm charts, Envoy proxy, dan distributed tracing alih-alih merilis fitur produk (*business velocity drops*).
2. **Conway's Law**: Arsitektur sistem mencerminkan struktur komunikasi organisasi. Microservices dirancang untuk perusahaan dengan puluhan tim independen (tiap tim 6-8 orang) agar mereka tidak bentrok saat deploy. Jika tim hanya 5 orang, microservices justru menciptakan friksi komunikasi.
3. **Biaya Infrastruktur & Overhead Jaringan**: Pemanggilan fungsi antar-modul di memori membutuhkan waktu ~10 nanodetik. Begitu diubah menjadi microservices via HTTP/JSON atau gRPC, latensi melonjak menjadi 2–20 milidetik (naik 100.000 kali lipat).

---

## 4. Cara Kerja & Diagram Alur (ASCII)

Perbandingan struktural antara Monolith, Modular Monolith, dan Microservices:

```
+-----------------------------------------------------------------------------+
| 1. TRADITIONAL MONOLITH (Tightly Coupled)                                   |
|                                                                             |
|   [ HTTP Requests ]                                                         |
|          |                                                                  |
|          v                                                                  |
|   +---------------------------------------------------------------------+   |
|   | Single App Process (Go Binary)                                      |   |
|   |   package auth  <---------> package order <---------> package pay   |   |
|   |         \                       |                       /           |   |
|   +----------\----------------------|----------------------/------------+   |
|               v                     v                     v                 |
|             +-----------------------------------------------+               |
|             |        Single Shared Relational Database      |               |
|             +-----------------------------------------------+               |
+-----------------------------------------------------------------------------+

+-----------------------------------------------------------------------------+
| 2. MODULAR MONOLITH (Strict In-Memory Boundaries)                           |
|                                                                             |
|   [ HTTP Requests ]                                                         |
|          |                                                                  |
|          v                                                                  |
|   +---------------------------------------------------------------------+   |
|   | Single App Process (Go Binary)                                      |   |
|   |   +-----------------+   Go Interface   +-----------------+          |   |
|   |   |  Module: Order  | ---------------> | Module: Payment |          |   |
|   |   | (Private Domain)| < - - - - - - -  | (Private Domain)|          |   |
|   |   +-----------------+  In-Memory Event +-----------------+          |   |
|   |            |                                    |                       |   |
|   +------------|------------------------------------|-------------------+   |
|                v                                    v                       |
|   +---------------------------+       +-----------------------------+       |
|   | DB Schema: orders_*       |       | DB Schema: payments_*       |       |
|   +---------------------------+       +-----------------------------+       |
|   (Isolated Tables / Logical Schemas within the same DB Instance)           |
+-----------------------------------------------------------------------------+

+-----------------------------------------------------------------------------+
| 3. MICROSERVICES (Independent Processes & Polyglot Storage)                |
|                                                                             |
|   [ HTTP Requests ]                                                         |
|          |                                                                  |
|          v                                                                  |
|   [ API Gateway ]                                                           |
|       |                     Network Call (gRPC / HTTP)                      |
|       +------------------------------------+                                |
|       |                                    |                                |
|       v                                    v                                |
|   +---------------------+              +---------------------+              |
|   | Order Service (Pod) | -----------> | Payment Serv. (Pod) |              |
|   +---------------------+              +---------------------+              |
|              |                                    |                         |
|              v                                    v                         |
|   +---------------------+              +---------------------+              |
|   | Order Database (PG) |              | Payment DB (MySQL)  |              |
|   +---------------------+              +---------------------+              |
+-----------------------------------------------------------------------------+
```

---

## 5. Contoh Implementasi Nyata di Kode (Go)

Contoh berikut menunjukkan bagaimana membedakan **Modular Monolith (In-Process Interface & Isolation)** dengan **Microservice Client (Network Call)** pada domain Checkout yang memanggil Payment.

### Pendekatan A: Modular Monolith (Cepat, Type-Safe, In-Memory)

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
)

// --- MODUL PAYMENT (Terkapsulasi di internal package terpisah) ---

type ChargeParams struct {
	OrderID  string
	Amount   float64
	Currency string
}

type ChargeResult struct {
	PaymentID string
	Success   bool
}

// PaymentService mendefinisikan interface publik modul Payment.
// Modul Order HANYA boleh bergantung pada interface ini.
type PaymentService interface {
	ProcessCharge(ctx context.Context, params ChargeParams) (*ChargeResult, error)
}

type paymentServiceImpl struct {
	// private database handle hanya untuk tabel-tabel pembayaran
}

func NewPaymentService() PaymentService {
	return &paymentServiceImpl{}
}

func (s *paymentServiceImpl) ProcessCharge(ctx context.Context, params ChargeParams) (*ChargeResult, error) {
	if params.Amount <= 0 {
		return nil, errors.New("amount must be greater than zero")
	}
	// Eksekusi logic internal payment (misal nembak gateway bank)
	return &ChargeResult{
		PaymentID: fmt.Sprintf("pay_%s", params.OrderID),
		Success:   true,
	}, nil
}

// --- MODUL ORDER (Menggunakan Dependency Injection antar-modul) ---

type OrderModule struct {
	payment PaymentService // In-memory dependency
}

func NewOrderModule(pay PaymentService) *OrderModule {
	return &OrderModule{payment: pay}
}

func (m *OrderModule) Checkout(ctx context.Context, orderID string, total float64) error {
	log.Printf("[Modular Monolith] Processing checkout for %s via in-memory interface", orderID)

	// Pemanggilan fungsi in-memory langsung (< 100ns, zero network serialization)
	res, err := m.payment.ProcessCharge(ctx, ChargeParams{
		OrderID:  orderID,
		Amount:   total,
		Currency: "IDR",
	})
	if err != nil || !res.Success {
		return fmt.Errorf("checkout payment failed: %w", err)
	}

	log.Printf("[Modular Monolith] Order %s successfully paid with %s", orderID, res.PaymentID)
	return nil
}

func main() {
	// Inisialisasi Modular Monolith di main.go tunggal
	paySvc := NewPaymentService()
	orderMod := NewOrderModule(paySvc)

	ctx := context.Background()
	_ = orderMod.Checkout(ctx, "ord_1001", 150000)
}
```

### Pendekatan B: Microservices Boundary (Network RPC via gRPC/HTTP)

Ketika dipecah menjadi Microservices, pemanggilan in-memory di atas berubah menjadi network call yang membutuhkan network timeout, serialization (JSON/Protobuf), retry, circuit breaker, dan penanganan kegagalan jaringan:

```go
// Pada Microservices, OrderModule tidak lagi memanggil Go pointer,
// melainkan memanggil gRPC client stub melalui koneksi TCP jaringan.
type MicroserviceOrderModule struct {
	// paymentClient pb.PaymentServiceClient (gRPC network stub)
}

func (m *MicroserviceOrderModule) CheckoutRPC(ctx context.Context, orderID string, total float64) error {
	// 1. Marshaling struct ke Protobuf/JSON
	// 2. Transmisi byte melalui koneksi HTTP/2 atau TCP
	// 3. Wajib menyertakan Timeout & Tracing context
	// 4. Penanganan network partition (504 Gateway Timeout, ECONNREFUSED)
	return nil
}
```

---

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Dimensi | Traditional Monolith | Modular Monolith | Microservices |
|---|---|---|---|
| **Kecepatan Deploy (Awal)** | Sangat Cepat (1 Binary) | Sangat Cepat (1 Binary) | Lambat (Banyak CI/CD & Image) |
| **Latensi Komunikasi** | Ultra-rendah ($< 1\,\mu\text{s}$) | Ultra-rendah ($< 1\,\mu\text{s}$) | Tinggi (2–20 ms via Network) |
| **Integritas Data** | ACID Transaksional Mudah | ACID Transaksional Terkontrol | Rumit (Saga / Eventual Consistency) |
| **Isolasi Domain & Kode** | Buruk (Kerap Spageti) | Sangat Bagus (Compiler Enforced) | Sangat Bagus (Fisik Terpisah) |
| **Skalabilitas Organisasi** | Terbatas (< 30 engineer) | Menengah-Tinggi (30–150 engineer) | Sangat Tinggi (> 200 engineer) |
| **Biaya Cloud & Server** | Minimal / Rendah | Minimal / Rendah | Tinggi (Banyak Pod, RAM, & Gateway) |
| **Toleransi Kegagalan** | Rendah (Crash = Mati Semua) | Menengah (Bisa isolasi worker pool)| Tinggi (Fault Isolation per Pod) |

---

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Memilih Modular Monolith
- **Startup Tahap Awal hingga Menengah (Seed s.d. Seri B)**: Product-Market Fit (PMF) masih sering berubah. Modular Monolith memungkinkan refactoring domain dengan mudah hanya melalui compiler refactor tanpa pusing breaking changes antarlayanan.
- **Ukuran Tim Engineering < 50 Orang**: Koordinasi tim masih bisa diselesaikan dalam satu atau beberapa squad tanpa memerlukan divisi platform infrastructure khusus.
- **Biaya Server Harus Efisien**: Menghindari pemborosan cluster Kubernetes besar untuk aplikasi yang traffic-nya belum mencapai jutaan request per detik.

### Kapan Memilih Microservices
- **Organisasi Skala Besar (> 150-200 Engineer)**: Puluhan tim fitur terhambat antrean rilis jika deploy bersamaan di satu repository monolit.
- **Karakteristik Komputasi Sangat Berbeda**: Bagian *Video Transcoding* butuh GPU dan memory besar, sedangkan service *Text Messaging* butuh I/O non-blocking ringan. Memisahkan keduanya menghemat biaya infrastruktur auto-scaling.
- **Kebutuhan Otonomi Teknologi (Polyglot)**: Tim AI menggunakan Python, tim Core Ledger menggunakan Go, dan tim Web Gateway menggunakan Node.js.

### Kapan Memilih Traditional Monolith
- **Aplikasi MVP, Internal Tool, atau Hackathon**: Dibangun oleh 1-2 orang dalam waktu beberapa minggu untuk validasi ide secepat kilat.

---

## 8. Metrik Kunci & Observabilitas

1. **Deployment Lead Time & Frequency (DORA Metrics)**: Seberapa cepat kode dari commit masuk ke production. Pada monolit yang buruk, angka ini bisa berhari-hari; pada modular monolit atau microservices yang matang, angka ini dalam hitungan menit.
2. **Cross-Service / Cross-Module Latency**: Memantau waktu yang dihabiskan untuk komunikasi antar-fitur. Lonjakan p99 pada microservices biasanya disebabkan oleh rantai RPC yang terlalu dalam (*deep nested network hops*).
3. **Cloud Infrastructure Cost per Monthly Active User (MAU)**: Rasio biaya server terhadap pengguna aktif. Jika beralih ke microservices membuat biaya server naik 5x tanpa kenaikan traffic, arsitektur mengalami *over-engineering*.
4. **Boundary Violation Count**: Menggunakan linter statis (seperti `depguard` di Go atau `packwerk` di Ruby) untuk mendeteksi apakah modul A mengimpor modul B secara terlarang.

---

## 9. Studi Kasus Nyata di Industri

### Skenario 1: Shopify — Menjaga Skala Raksasa dengan Modular Monolith
- **Problem**: Shopify memproses miliaran dolar transaksi e-commerce global (Black Friday/Cyber Monday). Awalnya mereka menggunakan monolit Ruby on Rails raksasa. Seiring pertumbuhan ratusan engineer, basis kode menjadi rumit dan rapuh (*tightly coupled*). Namun, Shopify menolak memecah menjadi ratusan microservices karena tidak mau menanggung biaya latensi jaringan dan kompleksitas transaksi database e-commerce.
- **Solusi**: Shopify merancang pendekatan **Modular Monolith**. Mereka membangun *tooling open-source* (`Packwerk`) untuk menegakkan batas-batas modul domain secara ketat saat kompilasi. Modul Inventory dilarang membaca tabel Orders secara langsung.
- **Hasil**: Shopify tetap mempertahankan satu monolit performan tinggi yang mampu menangani jutaan request per detik di Black Friday dengan tim ribuan engineer tanpa menderita overhead sistem terdistribusi.

### Skenario 2: Segment — Konsolidasi Kembali dari Microservices ke Monolith
- **Problem**: Platform data pelanggan Segment awalnya memecah pipeline integrasi API mereka menjadi lebih dari 140 microservices terpisah. Setiap ada update library bersama, engineer harus merilis 140 kali. Biaya operasional dan overhead antrian pesan membebani tim kecil mereka.
- **Solusi**: Segment memutuskan mengonsolidasi 140 microservices tersebut kembali menjadi **satu single monolith**.
- **Hasil**: Waktu rilis berkurang drastis, latensi rata-rata membaik, dan tim engineering dapat kembali fokus pada fitur pelanggan alih-alih merawat infrastruktur service yang terpecah-pecah.

---

## 10. Fakta Menarik & Gotchas Production

1. **The "Distributed Monolith" Anti-Pattern**: Ini adalah skenario terburuk dalam arsitektur software. Anda memiliki 30 microservices di server berbeda, tetapi semuanya membaca dan menulis ke **satu database PostgreSQL yang sama**! Anda mendapatkan seluruh keburukan microservices (network latency, deployment rumit) ditambah seluruh keburukan monolit (keterikatan skema DB dan single point of failure).
2. **Shared Libraries Are Dangerous in Microservices**: Jika Anda membuat library internal `common-utils` dan semua 20 service mengimpornya, setiap kali ada bugfix di library tersebut, Anda harus me-rebuild dan men-deploy ulang ke-20 service secara bersamaan. Ini adalah monolit yang menyamar.
3. **Circular Dependencies di Modular Monolith**: Tanpa linter arsitektur, Modul A akan memanggil Modul B, dan Modul B memanggil Modul A. Di Go, ini akan ditolak langsung oleh compiler (`import cycle not allowed`), yang merupakan berkah pelindung alami bahasa Go.

---

## 11. Rangkuman Singkat

1. **Tradisional Monolith vs Modular Monolith**: Keduanya di-deploy sebagai satu proses tunggal, namun Modular Monolith menerapkan batas domain modul dan akses database yang terisolasi ketat.
2. **Kapan Microservices Dibutuhkan**: Bukan karena traffic tinggi semata, melainkan karena skala organisasi tim (> 150 engineer) yang membutuhkan otonomi rilis mandiri.
3. **Bahaya Premature Microservices**: Menambah latensi jaringan (dari mikrodetik ke milidetik), biaya server, dan risiko inkonsistensi transaksi data terdistribusi.
4. **Prinsip Utama**: Mulailah dari **Modular Monolith**. Rancang batas modul yang rapi (*clean boundaries*). Pecah satu modul menjadi microservice terpisah HANYA jika modul tersebut memang memiliki kebutuhan scaling atau tim yang benar-benar berbeda.
