# Modular Monolith Architecture

## 1. Analogi Sederhana & Skenario Nyata

Bayangkan sebuah gedung perkantoran satu atap yang dirancang modern:

Gedung tersebut ditempati oleh berbagai divisi: **Divisi Finance**, **Divisi Sales**, dan **Divisi Logistik**. 
- Semua divisi bekerja di dalam gedung yang sama (satu izin sewa, satu genset listrik, satu alamat fisik).
- Namun, masing-masing divisi memiliki **ruangan kantor sendiri yang tertutup rapat dengan pintu akses kartu kunci**. 
- Staf Sales dilarang keras masuk menyelinap ke brankas dokumen Finance untuk mengambil faktur secara diam-diam. Jika Sales membutuhkan data faktur, mereka wajib mendatangi loket resmi (*receptionist / interface*) Finance dan meminta dokumen tersebut sesuai prosedur.

Jika seluruh gedung tidak disekat (seperti open-space berantakan), dokumen akan berserakan dan rahasia bocor (*Traditional Monolith tanpa batas*). Sebaliknya, jika ketiga divisi tersebut dipaksa menyewa gedung di tiga kota berbeda, biaya operasional membengkak dan komunikasi melambat drastis (*Microservices prematur*). 

**Modular Monolith** adalah titik temu ideal: **Satu proses deployment tunggal, namun dengan dinding pembatas domain bisnis yang ditegakkan secara ketat di dalam kode**.

---

## 2. Definisi & Konsep Inti

**Modular Monolith Architecture** adalah pola arsitektur perangkat lunak di mana aplikasi tetap dikompilasi, dijalankan, dan di-deploy sebagai **satu unit runtime tunggal**, namun struktur kode internalnya dipecah menjadi modul-modul independen yang terisolasi berdasarkan prinsip *Bounded Context* (Domain-Driven Design / DDD).

Aturan emas (*core rules*) dalam Modular Monolith:
1. **Strict Interface Boundaries**: Setiap modul hanya mengekspos *public interface* (antarmuka publik) dan DTO (Data Transfer Object) minimal. Implementasi internal, struct entitas, dan logika bisnis bersifat privat (*unexported*).
2. **Database Schema Isolation**: Modul `Order` tidak boleh melakukan `JOIN` SQL ke tabel milik modul `Payment`. Setiap modul memiliki tabel logis sendiri (misalnya schema `orders.*` vs `payments.*`), dan pertukaran data lintas modul wajib melalui API internal modul tersebut.
3. **In-Memory Domain Events**: Komunikasi asinkron antar-modul tidak membutuhkan broker Kafka eksternal yang rumit; cukup menggunakan *in-memory event dispatcher* atau Go channels.
4. **Compile-Time Linter Enforcement**: Batas modul ditegakkan oleh compiler (misal pencegahan *import cycle* di Go) atau alat linter arsitektur statis (seperti `depguard` di Go atau `packwerk` di Ruby) agar developer tidak bisa melanggar batas secara tidak sengaja.

---

## 3. Mengapa Sangat Penting di Skala Produksi

Modular Monolith kini menjadi arsitektur pilihan utama bagi startup skala menengah hingga *scale-up* teknologi modern:

1. **Jembatan Aman Menuju Skala Besar**: Menghindari kekacauan kode (*spaghetti code*) tanpa harus membayar "pajak" sistem terdistribusi (seperti *network latency*, *circuit breakers*, dan *eventual consistency*).
2. **Refactoring Bebas Risiko**: Jika batas domain antar-modul salah dirancang, memperbaikinya di dalam satu codebase hanya membutuhkan refactor compiler Go (dalam hitungan jam). Pada microservices, salah merancang batas layanan bisa memakan waktu berbulan-bulan untuk migrasi API dan database terpisah.
3. **Peluang Ekstraksi ke Microservices Tanpa Sakit**: Jika kelak salah satu modul (misal modul Transcoding atau Chat) traffic-nya meledak dan butuh resource komputasi GPU mandiri, modul tersebut sudah memiliki batas yang rapi sehingga bisa "dipotong" menjadi microservice independen dalam hitungan hari.
4. **Efisiensi Finansial & Operasional Tinggi**: Menikmati kecepatan eksekusi panggilan in-memory ($< 1\,\mu\text{s}$) dan kemudahan deployment tunggal dengan biaya infrastruktur cloud yang jauh lebih hemat.

---

## 4. Cara Kerja & Diagram Alur (ASCII)

Struktur isolasi modul dan pertukaran data di dalam Modular Monolith:

```
+---------------------------------------------------------------------------------+
| MODULAR MONOLITH RUNTIME (Single Go Process)                                    |
|                                                                                 |
|  [ Public Ingress / HTTP Handlers ]                                             |
|          |                                                                      |
|          |-- 1. Call Order API                                                  |
|          v                                                                      |
|  +-------------------------------------+                                        |
|  | Module: Order (internal/order)      |                                        |
|  |   - Private domain logic            |                                        |
|  |   - Public Interface: OrderService  |                                        |
|  +-------------------------------------+                                        |
|          |                                                                      |
|          |-- 2. In-Memory Interface Call (Type-Safe, <100ns)                    |
|          v                                                                      |
|  +-------------------------------------+                                        |
|  | Module: Payment (internal/payment)  |                                        |
|  |   - Private domain logic            |                                        |
|  |   - Public Interface: PaymentService|                                        |
|  +-------------------------------------+                                        |
|          |                                                                      |
|          |-- 3. Publish In-Memory Event ("PaymentSuccessEvent")                 |
|          v                                                                      |
|  [ In-Memory Domain Event Bus (Go Channels / Sync Event Dispatcher) ]           |
|          |                                                                      |
|          +--------------------------------------+                               |
|                                                 v                               |
|                              +--------------------------------------+           |
|                              | Module: Notification (internal/notif)|           |
|                              |   - Listens to PaymentSuccessEvent   |           |
|                              |   - Sends email asynchronously       |           |
|                              +--------------------------------------+           |
+---------------------------------------------------------------------------------+
                                       |
                                       v
         +-------------------------------------------------------------+
         | Single Relational Database Instance (PostgreSQL)            |
         |  - Logical Schema 1: `orders.*` (Managed ONLY by Order)     |
         |  - Logical Schema 2: `payments.*` (Managed ONLY by Payment) |
         |  - NO DIRECT CROSS-SCHEMA SQL JOINS ALLOWED!                |
         +-------------------------------------------------------------+
```

---

## 5. Contoh Implementasi Nyata di Kode (Go)

Contoh berikut menunjukkan bagaimana membagi paket Go menjadi modul mandiri dengan kontrak interface publik dan isolasi domain logic:

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
	"sync"
	"time"
)

// ============================================================================
// 1. KONTRAK PUBLIK MODUL PAYMENT (package internal/payment/contract)
// ============================================================================

type ChargeRequest struct {
	OrderID  string
	Amount   float64
	Currency string
}

type ChargeResponse struct {
	PaymentID string
	Status    string
}

// PaymentModuleInterface adalah SATU-SATUNYA gerbang yang boleh diakses modul lain.
type PaymentModuleInterface interface {
	Charge(ctx context.Context, req ChargeRequest) (*ChargeResponse, error)
}

// ============================================================================
// 2. IMPLEMENTASI PRIVAT MODUL PAYMENT (package internal/payment)
// ============================================================================

type paymentModule struct {
	// private database connection / private tables
}

func NewPaymentModule() PaymentModuleInterface {
	return &paymentModule{}
}

func (p *paymentModule) Charge(ctx context.Context, req ChargeRequest) (*ChargeResponse, error) {
	if req.Amount <= 0 {
		return nil, errors.New("invalid charge amount")
	}
	// Eksekusi logic internal payment
	return &ChargeResponse{
		PaymentID: fmt.Sprintf("pay_%s_%d", req.OrderID, time.Now().Unix()),
		Status:    "SETTLED",
	}, nil
}

// ============================================================================
// 3. MODUL ORDER (package internal/order)
// Hanya bergantung pada interface publik modul Payment, bukan implementasinya.
// ============================================================================

type OrderModule struct {
	paymentClient PaymentModuleInterface
	events        chan string
}

func NewOrderModule(pay PaymentModuleInterface, events chan string) *OrderModule {
	return &OrderModule{
		paymentClient: pay,
		events:        events,
	}
}

func (o *OrderModule) CreateOrderAndPay(ctx context.Context, orderID string, amount float64) error {
	log.Printf("[OrderModule] Creating order %s", orderID)

	// 1. Panggil modul payment melalui interface in-memory (< 100ns)
	payResp, err := o.paymentClient.Charge(ctx, ChargeRequest{
		OrderID:  orderID,
		Amount:   amount,
		Currency: "IDR",
	})
	if err != nil {
		return fmt.Errorf("payment rejected: %w", err)
	}

	log.Printf("[OrderModule] Order %s successfully charged with PaymentID: %s", orderID, payResp.PaymentID)

	// 2. Publish in-memory domain event ke modul lain secara non-blocking
	select {
	case o.events <- fmt.Sprintf("ORDER_COMPLETED:%s", orderID):
	default:
	}

	return nil
}

// ============================================================================
// 4. MAIN BOOTSTRAPPER (Komposisi Modular Monolith)
// ============================================================================

func main() {
	eventBus := make(chan string, 100)
	var wg sync.WaitGroup

	// Background worker untuk mendengarkan domain event internal (misal notifikasi)
	wg.Add(1)
	go func() {
		defer wg.Done()
		for ev := range eventBus {
			log.Printf("[NotificationModule] Received internal domain event: %s -> Sending email...", ev)
			return // Keluar setelah simulasi 1 event
		}
	}()

	// Inisialisasi modul-modul terpisah
	paymentMod := NewPaymentModule()
	orderMod := NewOrderModule(paymentMod, eventBus)

	ctx := context.Background()
	_ = orderMod.CreateOrderAndPay(ctx, "ord_2026_id", 350000)

	close(eventBus)
	wg.Wait()
}
```

---

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Arsitektur Rapi & Bersih**: Struktur domain jelas dan terkelola berkat pembagian *bounded context* yang tegas.
- **Performa Kelas Satu**: Komunikasi antar-modul tetap secepat pemanggilan fungsi memori biasa ($< 1\,\mu\text{s}$) tanpa serialisasi jaringan.
- **Kesiapan Ekstraksi (Microservices-Ready)**: Jika satu modul perlu dipisahkan menjadi microservice independen, kodenya sudah terisolasi dan siap diekstraksi kapan saja.
- **Zero Distributed Latency**: Tidak ada *network hops* internal, *distributed deadlock*, atau latensi RPC yang memperlambat p99 response time.

### Kekurangan
- **Membutuhkan Disiplin Tim Tinggi**: Tanpa *architectural linter* otomatis, developer yang malas atau terburu-buru akan tergoda mengimpor struct internal modul lain secara langsung.
- **Shared Failure Domain**: Jika ada kebocoran memori parah atau *goroutine deadlock* di modul analytics, seluruh binary modular monolith tetap bisa mengalami crash bersamaan.
- **Teknologi Seragam (Monoglot)**: Seluruh modul harus ditulis dalam bahasa pemrograman dan runtime yang sama (misal semua tim menggunakan Go).

---

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- **Startup Pertumbuhan Cepat (Series A s.d. Series C, Tim 20–100 Engineer)**: Membutuhkan isolasi kepemilikan kode per squad tanpa kerumitan infrastruktur DevOps yang masif.
- **Sistem E-Commerce & SaaS Kompleks**: Domain bisnis luas (Inventory, Auth, Billing, Promo, CRM), tetapi masih ingin menghemat anggaran server cloud.
- **Sebelum Membangun Microservices**: Sebagai langkah awal yang wajib dilalui (*Monolith-First Strategy*) sebelum memutuskan memecah sistem.

### Kapan Jangan Pakai
- **Prototipe MVP 1–2 Minggu**: Membagi-bagi package dan interface secara ketat di awal bisa memperlambat validasi ide kilat pada hackathon atau proyek sampingan.
- **Kebutuhan Tim Terdistribusi Raksasa (> 200+ Engineer)**: Koordinasi merge conflict dan dependensi deployment satu binary mulai menjadi hambatan harian.

---

## 8. Metrik Kunci & Observabilitas

1. **Architecture Boundary Violations (`arch_violations_count`)**: Jumlah pelanggaran batas impor terlarang yang terdeteksi saat CI build (target: 0).
2. **In-Memory Event Bus Queue Depth (`internal_event_queue_size`)**: Jumlah pesan yang mengantre di kanal event internal antar-modul.
3. **Module Execution Duration Breakdown**: Menghitung berapa persen waktu request dihabiskan di dalam Modul Order vs Modul Payment.
4. **CI/CD Unit Test Execution Time per Module**: Menjalankan pengujian secara modular agar modul yang tidak berubah tidak perlu diuji ulang (*cached test runs*).

---

## 9. Studi Kasus Nyata di Industri

### Skenario: Shopify — Menjaga Skala Miliaran Dolar dengan Modular Monolith

#### Problem
Shopify memproses miliaran transaksi e-commerce global (khususnya lonjakan Black Friday/Cyber Monday). Awalnya basis kode mereka adalah monolit raksasa. Seiring bertambahnya ratusan engineer, basis kode menjadi rapuh karena modul saling mengimpor data secara serampangan (*tightly coupled*). Namun, tim arsitek menolak migrasi ke microservices karena risiko konsistensi transaksi e-commerce dan latensi jaringan.

#### Solusi Arsitektur
Shopify merancang arsitektur **Modular Monolith**:
- Membagi aplikasi ke dalam puluhan modul terisolasi (*componentization*).
- Membangun dan merilis alat open-source `Packwerk` untuk menegakkan aturan batas modul saat compile-time. Modul Inventory secara ketat dilarang menjalankan query ke tabel Orders tanpa melalui interface resmi.
- Memisahkan skema basis data secara logis di dalam satu kluster database yang sama.

#### Hasil / Dampak
- **Performa Ekstrem Tetap Terjaga**: Mampu menangani jutaan request per detik di Black Friday dengan latensi sub-detik tanpa beban sistem terdistribusi.
- **Skalabilitas Organisasi Tinggi**: Ratusan tim engineer dapat bekerja bersamaan di satu repository tanpa saling merusak domain rekan kerja.
- **Efisiensi Finansial Luar Biasa**: Menghemat biaya infrastruktur hingga puluhan juta dolar dibanding jika memecah sistem menjadi ratusan microservices.

---

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan "Modular Tapi Satu Transaksi Database Raksasa"**: Banyak engineer memisahkan kode modul, tetapi tetap menggabungkan pemanggilan database kedua modul dalam 1 transaksi `tx.Begin()` yang panjang. Akibatnya, modul menjadi terikat secara transaksional di database level. Solusi: Pisahkan transaksi database per modul dan gunakan pola event-driven internal.
2. **Compiler Go Adalah Sahabat Modular Monolith**: Go secara tegas melarang *cyclic imports* (`import cycle not allowed`). Ini memaksa arsitektur kalian bersih secara alami.
3. **Pemisahan Skema Database Nyata**: Jangan hanya memisahkan kode Go; pisahkan juga tabelnya di PostgreSQL menggunakan schema berbeda (`CREATE SCHEMA orders; CREATE SCHEMA payments;`). Cabut hak akses user database `order_user` ke schema `payments`.

---

## 11. Rangkuman Singkat

1. **Definisi Inti**: Satu proses runtime tunggal (seperti monolit biasa), tetapi memiliki batas modul (*bounded contexts*), interface privat, dan isolasi data yang sangat ketat.
2. **Keunggulan Utama**: Performa ultra-cepat in-memory ($< 1\,\mu\text{s}$), biaya cloud hemat, dan kesiapan untuk dipotong menjadi microservices kelak jika diperlukan.
3. **Kunci Sukses**: Disiplin batas domain yang ditegakkan secara otomatis via compiler dan architectural linters.
4. **Sweet Spot Industri**: Arsitektur paling direkomendasikan untuk 90% startup tahap berkembang (Series A-C) sebelum terjebak kompleksitas microservices.
