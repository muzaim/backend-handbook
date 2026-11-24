# API Versioning & Deprecation Strategies

## 1. Analogi Sederhana & Skenario Nyata

Ketika instansi perpajakan memperbarui formulir pelaporan pajak tahunan dari Formulir Versi 2024 ke Formulir Versi 2026 dengan beberapa format kolom baru, mereka tidak bisa seketika menolak dan membakar seluruh formulir versi 2024 yang sedang dalam perjalanan pos dari jutaan warga. Kantor pajak memberlakukan kebijakan masa transisi resmi: Formulir Versi 2024 tetap diproses dan dinyatakan sah hingga batas waktu tertentu (*Sunset Date*), sembari mengumumkan instruksi transisi agar seluruh warga secara bertahap berpindah ke Formulir Versi 2026.

Dalam rekayasa perangkat lunak backend, **API Versioning & Deprecation** adalah disiplin mengelola evolusi kontrak data antarmuka (API) secara terkontrol tanpa merusak aplikasi klien yang sedang aktif berjalan di produksi. Hal ini sangat krusial terutama pada ekosistem **Aplikasi Mobile (Android/iOS)** dan **Partner API Pihak Ketiga**, di mana backend developer tidak memiliki kendali langsung untuk memaksa jutaan pengguna memperbarui aplikasi mereka dalam semalam.

---

## 2. Definisi & Konsep Inti

**API Versioning** adalah teknik merilis perubahan pada struktur atau perilaku API secara berdampingan (*side-by-side*) dengan versi sebelumnya. Ketika versi lama diputuskan untuk dipensiunkan, proses **API Deprecation** dijalankan untuk memberi sinyal peringatan formal kepada pengguna API sebelum endpoint tersebut benar-benar dinonaktifkan secara permanen (*Sunset / Decommission*).

```
+--------------------------------------------------------------------------+
|                        SIKLUS HIDUP VERSI API                            |
+--------------------------------------------------------------------------+
|                                                                          |
|   [ACTIVE / GA]   --> API beroperasi normal dengan dukungan penuh        |
|         |                                                                |
|         v                                                                |
|   [DEPRECATED]    --> Versi baru dirilis; versi lama masih berfungsi     |
|         |             tetapi menyertakan header Sunset & Deprecation     |
|         v                                                                |
|   [SUNSET / RETIRED]  Endpoint dimatikan permanen (HTTP 410 Gone / 404)  |
|                                                                          |
+--------------------------------------------------------------------------+
```

### Tiga Strategi Utama API Versioning di Industri

| Strategi | Contoh Sintaks | Kelebihan | Kekurangan |
| :--- | :--- | :--- | :--- |
| **1. URI Path** | `GET /api/v1/orders`<br>`GET /api/v2/orders` | Sangat eksplisit, mudah di-route oleh reverse proxy (NGINX/Envoy), dan sangat ramah terhadap cache CDN. | Melanggar prinsip REST teoritis murni (URI merepresentasikan versi, bukan sumber daya). |
| **2. Custom Header** | `GET /api/orders`<br>`X-API-Version: 2` | URI tetap bersih dan elegan. Mematuhi standar arsitektur REST. | Sulit di-cache oleh CDN publik (membutuhkan header `Vary`), sulit diuji langsung dari browser URL bar. |
| **3. Content Negotiation** | `Accept: application/vnd.company.v2+json` | Standar resmi RFC RESTful murni (*Hypermedia standard*). | Sangat kompleks bagi developer konsumen dan perkakas pengujian API (cURL, Postman). |

---

## 3. Mengapa Sangat Penting di Skala Produksi?

1. **Jebakan Versi Aplikasi Mobile (The Mobile Lag)**:  
   Pada aplikasi web, developer bisa memperbarui frontend dan backend secara bersamaan dalam satu deployment. Namun pada aplikasi Android dan iOS di Google Play Store atau Apple App Store, **20-30% pengguna tidak pernah mengaktifkan auto-update aplikasi**! Jika kamu melakukan *breaking change* pada endpoint yang sama (misal menghapus field `full_name` menjadi `first_name` dan `last_name`), aplikasi mobile versi lama di HP ratusan ribu pengguna akan mengalami *crash* seketika (*NullPointerException*).
2. **Perjanjian SLA Mitra Bisnis (B2B Integrations)**:  
   Platform pembayaran (seperti Stripe atau Midtrans) terintegrasi dengan ribuan sistem enterprise eksternal. Mengubah struktur payload tanpa versioning akan merusak sistem perbankan mitra dan memicu tuntutan hukum pelanggaran Service Level Agreement (SLA).
3. **Standarisasi Penghentian Layanan (IETF RFC 8594)**:  
   Menghentikan dukungan API lama membutuhkan komunikasi formal berbasis mesin via HTTP header resmi:
   - **`Deprecation: true`** atau **`Deprecation: @<timestamp>`**: Menandai bahwa endpoint ini sudah usang.
   - **`Sunset: <HTTP-date>`**: Tanggal mutlak kapan endpoint ini akan dimatikan total.
   - **`Link: <url>; rel="sunset"`**: Tautan dokumentasi panduan migrasi ke versi baru.

---

## 4. Cara Kerja & Diagram Alur (ASCII)

```
[Klien Kuno (v1)]                 [Klien Modern (v2)]
        |                                  |
        | GET /api/v1/products             | GET /api/v2/products
        v                                  v
+-------------------------------------------------------------+
|                      API GATEWAY / ROUTER                   |
|                                                             |
| - Deteksi Versi: Path /v1/ vs /v2/                          |
| - Injeksi Header Deprecation untuk request v1               |
+------------------------------+------------------------------+
                               |
            +------------------+------------------+
            |                                     |
            v                                     v
+-----------------------+             +-----------------------+
|  Handler V1 (Legacy)  |             |  Handler V2 (Modern)  |
|  - Deprecation Header |             |  - Schema Baru        |
|  - Sunset: 31-Dec-2026|             |  - Throughput Cepat   |
+-----------------------+             +-----------------------+
```

---

## 5. Contoh Implementasi Nyata di Kode (Go)

Berikut adalah implementasi nyata **API Versioning Middleware** di Go yang mendukung routing berbasis URI path sekaligus secara otomatis menginjeksi header standar IETF `Deprecation` dan `Sunset` untuk versi lama:

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"strings"
	"time"
)

// DTO Versi 1 (Legacy): Menggunakan satu string field NamaLengkap
type UserResponseV1 struct {
	ID          string `json:"id"`
	NamaLengkap string `json:"nama_lengkap"`
	Email       string `json:"email"`
}

// DTO Versi 2 (Modern): Nama dipecah menjadi FirstName & LastName sesuai standar baru
type UserResponseV2 struct {
	ID        string `json:"id"`
	FirstName string `json:"first_name"`
	LastName  string `json:"last_name"`
	Email     string `json:"email"`
}

// Middleware untuk menyuntikkan header IETF Deprecation & Sunset
func DeprecationNoticeMiddleware(sunsetDate time.Time, docURL string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Standar IETF RFC 8594 Sunset Header
			w.Header().Set("Deprecation", "true")
			w.Header().Set("Sunset", sunsetDate.UTC().Format(http.TimeFormat))
			w.Header().Set("Link", fmt.Sprintf("<%s>; rel=\"sunset\"", docURL))

			next.ServeHTTP(w, r)
		})
	}
}

// Handler V1 (Deprecated)
func GetUserV1(w http.ResponseWriter, r *http.Request) {
	resp := UserResponseV1{
		ID:          "USR-101",
		NamaLengkap: "Syafri Surya",
		Email:       "syafri@example.com",
	}

	w.Header().Set("Content-Type", "application/json")
	_ = json.NewEncoder(w).Encode(resp)
}

// Handler V2 (Active / Modern)
func GetUserV2(w http.ResponseWriter, r *http.Request) {
	resp := UserResponseV2{
		ID:        "USR-101",
		FirstName: "Syafri",
		LastName:  "Surya",
		Email:     "syafri@example.com",
	}

	w.Header().Set("Content-Type", "application/json")
	_ = json.NewEncoder(w).Encode(resp)
}

func main() {
	mux := http.NewServeMux()

	// Tanggal pemadaman total versi 1: 31 Desember 2026
	v1SunsetDate := time.Date(2026, time.December, 31, 23, 59, 59, 0, time.UTC)
	migrationDoc := "https://api.example.com/docs/migrations/v1-to-v2"

	// Register V1 routes dengan middleware deprecation
	v1Handler := http.HandlerFunc(GetUserV1)
	mux.Handle("/api/v1/user", DeprecationNoticeMiddleware(v1SunsetDate, migrationDoc)(v1Handler))

	// Register V2 routes tanpa deprecation
	mux.HandleFunc("/api/v2/user", GetUserV2)

	fmt.Println("API Server berjalan di port :8080...")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Zero-Downtime Migration**: Klien lama dan baru dapat hidup berdampingan secara damai tanpa interupsi layanan.
- **Keamanan dan Prediktabilitas**: Developer klien memiliki jadwal yang pasti kapan harus memigrasikan kode sebelum batas *Sunset*.
- **Pemberitahuan Otomatis ke Mesin Klien**: Header `Sunset` memungkinkan SDK klien modern mencatat peringatan secara otomatis di log developer mereka sebelum endpoint mati.

### Kekurangan
- **Beban Pemeliharaan Ganda (*Maintenance Tax*)**: Menjaga 2 atau 3 versi API yang aktif bersamaan berarti tim backend harus mengelola bug fixes dan testing pada beberapa versi kode sekaligus.
- **Risiko Database Drift**: Jika versi 1 dan versi 2 menulis ke tabel database yang sama dengan skema berbeda, backend memerlukan adapter layer yang kompleks (*Data Layer Shims*).
- **Inersia Klien**: Tanpa tenggat waktu sunset yang tegas dan penegakan komunikasi, developer pihak ketiga cenderung menunda migrasi selamanya.

---

## 7. Kapan Harus Bikin Versi Baru vs Kapan Cukup Non-Breaking Change

### Kapan Wajib Membuat Versi Baru (Breaking Changes):
1. **Menghapus atau Mengubah Nama Field**: Menghapus field `phone` atau mengubahnya menjadi `phone_number`.
2. **Mengubah Tipe Data**: Mengubah field `amount` dari integer (`15000`) menjadi string berformat (`"Rp 15.000"`).
3. **Mengubah Alur Autentikasi**: Mengubah otentikasi dari API Key query param menjadi OAuth2 Bearer Token.
4. **Mengubah Struktur Respons Dasar**: Mengubah format array `data: [...]` menjadi pagination object `{ results: [...], next: "..." }`.

### Kapan TIDAK Perlu Versi Baru (Non-Breaking Changes):
1. **Menambahkan Field Baru**: Menambahkan field opsional baru (seperti `avatar_url`) pada payload JSON. Klien lama yang mengabaikan field tak dikenal tidak akan mengalami crash.
2. **Menambahkan Endpoint Baru**: Menambahkan `POST /api/v1/orders/cancel`.
3. **Optimasi Performa Internal**: Mengubah query database dari lambat menjadi cepat atau menambahkan caching Redis di sisi server.

---

## 8. Metrik Kunci & Observabilitas

Metrik wajib di Grafana / Prometheus untuk memantau adopsi versi dan jadwal sunset:

- **`http_requests_total{version="v1|v2", handler="..."}`**: Rasio perbandingan volume trafik antara versi lama vs versi baru.
- **`deprecated_api_requests_total{client_type="mobile_android|mobile_ios|web"}`**: Jumlah request yang masih menggunakan versi deprecated dikelompokkan berdasarkan tipe klien (sangat berguna untuk melacak sisa pengguna mobile lama).
- **`sunset_countdown_days{version="v1"}`**: Jumlah hari tersisa sebelum tanggal batas akhir pemadaman.
- **`api_client_user_agent_breakdown{version="v1"}`**: Versi aplikasi mobile (`User-Agent: MyEcomApp/3.2.1`) yang masih menembak API versi 1.

---

## 9. Studi Kasus Nyata di Industri

### Strategi Versioning Stripe: Menangani Ratusan Versi Tanpa Duplikasi Kode Controller

- **Tantangan Rekayasa Riil (The Problem)**: Stripe melayani jutaan developer pembayaran global. Sejak tahun 2011, Stripe telah merilis lebih dari 100 perubahan API. Jika Stripe menggunakan metode konvensional dengan membuat 100 controller terpisah untuk setiap versi (`v1`, `v2`, ..., `v100`), basis kode backend mereka akan runtuh akibat duplikasi ribuan baris kode logika transaksi pembayaran yang sama persis dan beban testing regresi yang mustahil dikelola.
- **Implementasi Solusi Industri (The Architectural Solution)**: Tim Stripe merancang sistem *Eventual Transformation Pipeline* berbasis tanggal rilis (`Stripe-Version: 2020-08-27`). Kode backend inti Stripe hanya mengeksekusi satu versi paling mutakhir (versi hari ini). Di layer gateway, Stripe menyisipkan *API Version Shims* deklaratif. Ketika request dari klien versi lama masuk, gateway mentransformasikan payload request lama ke format skema terbaru. Setelah logika bisnis inti selesai, lapisan transformasi mundur (*backward response transform*) mengubah format respons terbaru kembali ke bentuk JSON kuno yang diharapkan oleh klien versi lama tersebut.
- **Dampak & Hasil Sistem (The Outcome)**: Stripe berhasil mendukung lebih dari 10 tahun kompatibilitas mundur tanpa ada satupun integrasi pembayaran merchant lama yang rusak. Tim engineer Stripe hanya perlu merawat satu basis kode domain utama, sementara beban pemeliharaan versi lama terisolasi murni sebagai skrip transformasi data ringkas di lapisan gateway.

---

## 10. Fakta Menarik & Gotchas Production

1. **Pola Expand and Contract (Kunci Tanpa Downtime)**:  
   Jangan pernah langsung mengganti kolom atau field data! Selalu gunakan 3 tahap:
   - *Tahap 1 (Expand)*: Tambahkan field baru di samping field lama. Server menulis ke kedua field sekaligus (*dual-write*).
   - *Tahap 2 (Migrate)*: Perbarui seluruh klien secara bertahap untuk membaca field baru.
   - *Tahap 3 (Contract)*: Setelah metrik menunjukkan trafik pembacaan field lama mencapai 0%, hapus field lama secara aman.
2. **Mobile App Store Review Delay**:  
   Jangan pernah merilis backend breaking change bersamaan dengan submit update aplikasi ke App Store. Persetujuan aplikasi oleh tim review Apple / Google bisa memakan waktu 24 hingga 72 jam, atau bahkan ditolak! Jika kamu sudah mematikan API versi lama sebelum update aplikasi disetujui, ratusan ribu pengguna aktif akan terkatung-katung tanpa aplikasi yang berfungsi.
3. **Mekanisme Force Update (Kill Switch)**:  
   Selalu siapkan endpoint `/api/config` atau respons header `X-Minimum-App-Version: 4.0.0` sejak hari pertama aplikasi mobile dibuat. Jika terjadi celah keamanan darurat pada API v1 yang mengharuskan pemadaman seketika, aplikasi mobile dapat menampilkan dialog modal: *"Aplikasi ini sudah usang. Silakan update di Play Store untuk melanjutkan"*.

---

## 11. Rangkuman Singkat

- **Pilih Strategi Versioning Sesuai Kebutuhan**: Gunakan URI Path (`/v1/`) untuk kemudahan perutean dan kompatibilitas cache CDN maksimal; gunakan Header Versioning untuk keselarasan REST murni.
- **Pisahkan Breaking vs Non-Breaking Changes**: Menambah field baru bersifat kompatibel mundur (*non-breaking*); menghapus field atau mengubah tipe data adalah *breaking change* yang wajib menaikkan nomor versi API.
- **Patuhi Standar IETF Deprecation & Sunset**: Gunakan header resmi `Deprecation: true` dan `Sunset: <date>` untuk mengomunikasikan batas waktu pemadaman endpoint lama secara otomatis kepada mesin klien.
- **Terapkan Pola Expand & Contract**: Jangan langsung menghapus skema lama; jalankan periode transisi berdampingan (*dual-write*) hingga seluruh adopsi klien bermigrasi secara tuntas.
