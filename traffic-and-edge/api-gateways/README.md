# API Gateway

## 1. Skenario Sistem Nyata

Dalam arsitektur e-commerce berbasis microservices skala besar, sistem dipecah menjadi puluhan service backend terpisah di jaringan privat:

-   `Auth Service` (port 4001)
-   `Product Catalog Service` (port 4002)
-   `Order Service` (port 4003)
-   `Payment Service` (port 4004)
-   `Notification Service` (port 4005)

Jika sistem ini langsung diakses oleh aplikasi mobile (iOS dan Android):

-   Aplikasi mobile harus menyimpan dan mengetahui 5 URL/IP berbeda. Jika struktur internal service dipecah atau port berganti, aplikasi mobile harus di-update dan di-submit ulang ke App Store/Play Store.
-   Setiap service harus mengimplementasikan logika verifikasi JWT, CORS, rate limiting, dan SSL certificate sendiri-sendiri secara berulang (_code duplication_).
-   Untuk menampilkan satu layar checkout belanja, aplikasi mobile harus melakukan 4 panggilan HTTP terpisah melalui jaringan seluler yang lambat (_network chattiness_).

**Solusi dengan API Gateway (misal: Kong / KrakenD):**
Kamu menempatkan satu API Gateway di alamat publik `https://api.toko.com`. Aplikasi mobile hanya perlu berbicara ke satu endpoint ini. Gateway memverifikasi JWT token pengguna sekali saja di depan, menyaring rate limit per client, mengarahkan `/api/v1/orders` ke internal `http://order-service:4003`, dan bahkan dapat menggabungkan (_aggregate_) respon dari beberapa service menjadi satu JSON response ringkas ke mobile app.

## 2. Apa Itu dan Buat Apa?

API Gateway adalah komponen arsitektur yang bertindak sebagai pintu gerbang tunggal (_single point of entry_) untuk seluruh client (aplikasi mobile, web SPA, partner pihak ketiga) yang ingin mengakses sistem backend berbasis microservices.

Jika reverse proxy lebih berfokus pada layer jaringan (TCP/HTTP, SSL, caching statis), API Gateway beroperasi pada layer aplikasi dan logika bisnis API:

-   **Routing & Path Rewriting**: Memetakan `/users/*` ke User Service dan `/orders/*` ke Order Service.
-   **Autentikasi & Otorisasi Terpusat**: Memverifikasi JWT (JSON Web Token) atau API Key di gateway, sehingga service di belakangnya tidak perlu berulang kali mengecek token.
-   **Request Aggregation (BFF - Backend for Frontend)**: Menggabungkan data dari 3 microservices berbeda dalam satu kali panggilan HTTP dari mobile app.
-   **Rate Limiting per Pengguna/Tier**: Membatasi kuota request berdasarkan akun pelanggan (Free vs Premium).

Contoh software populer: Kong, Apache APISIX, KrakenD, Spring Cloud Gateway, AWS API Gateway.

## 3. Kenapa Harus Pakai Ini ?

Tanpa API Gateway, aplikasi mobile atau frontend web harus memanggil puluhan alamat microservice secara langsung melalui internet:

-   **Latensi jaringan seluler**: Melakukan 10 panggilan HTTP serial dari smartphone di jaringan 4G/5G akan membuat UI aplikasi terasa sangat lambat (_network round-trip penalty_).
-   **Kerentanan keamanan**: Seluruh microservice internal harus memiliki IP publik dan membuka port ke internet.
-   **Duplikasi kode cross-cutting**: Setiap tim developer harus menulis ulang kode verifikasi token JWT, logging request, dan CORS di setiap service.

Dengan API Gateway, client cukup melakukan satu panggilan cepat, dan gateway (yang berada di jaringan privat berkecepatan tinggi) yang melakukan orkestrasi internal dengan latensi sub-milidetik.

## 4. Cara Kerjanya (Diagram Alur)

```
+--------------------+
| Mobile / Web Apps  |
+--------------------+
          |
          | 1. Single API Request (e.g. GET /dashboard + JWT)
          v
+---------------------------------------+
|             API Gateway               |
|                                       |
| - Verify Auth / JWT Token             |
| - Check Rate Limit Quota              |
| - Route & Aggregate Responses         |
+---------------------------------------+
      |               |               |
      | 2a. HTTP/gRPC | 2b. HTTP/gRPC | 2c. HTTP/gRPC
      v               v               v
+------------+  +------------+  +------------+
| User Svc   |  | Order Svc  |  | Payment Svc|
+------------+  +------------+  +------------+
```

## 5. Cuplikan Logika Inti (Middleware Gateway di Go)

Berikut alur eksekusi pipeline middleware yang dijalankan API Gateway pada setiap request menggunakan Go:

```go
package main

import (
	"net/http"
	"net/http/httputil"
	"net/url"
)

// GatewayMiddlewarePipeline menjalankan autentikasi terpusat dan perutean microservice
func GatewayMiddlewarePipeline(targetServiceURL string) http.Handler {
	target, _ := url.Parse(targetServiceURL)
	proxy := httputil.NewSingleHostReverseProxy(target)

	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// 1. Validasi Autentikasi JWT Terpusat
		token := r.Header.Get("Authorization")
		if token == "" || !isValidJWT(token) {
			http.Error(w, `{"error": "Unauthorized"}`, http.StatusUnauthorized)
			return
		}

		// 2. Cek Kuota Rate Limiting per IP
		if !rateLimiterAllow(r.RemoteAddr) {
			http.Error(w, `{"error": "Rate limit exceeded"}`, http.StatusTooManyRequests)
			return
		}

		// 3. Inject Context & Teruskan ke Microservice Internal
		r.Header.Set("X-User-ID", extractUserID(token))
		proxy.ServeHTTP(w, r)
	})
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Menyederhanakan client**: Client hanya perlu tahu satu domain API publik.
-   **Sentralisasi cross-cutting concerns**: Autentikasi, rate limit, logging, dan metrik ditangani di satu tempat.
-   **Protokol internal bebas**: Gateway bisa menerima HTTP/REST dari luar, tetapi berkomunikasi via gRPC berkecepatan tinggi ke microservice internal.

### Kekurangan

-   **Bottleneck performa**: Semua request melewati gateway; jika gateway salah konfigurasi, seluruh ekosistem melambat.
-   **Penyatuan kegagalan (Single Point of Failure)**: Gateway mati berarti seluruh aplikasi lumpuh total.
-   **Risiko logika bisnis berlebihan (Smart Gateway Trap)**: Tim sering tergoda menaruh logika bisnis kompleks di gateway hingga gateway menjadi "monolith baru".

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Arsitektur berbasis Microservices dengan banyak domain service independen.
-   Sistem yang melayani berbagai jenis client dengan kebutuhan payload berbeda (Mobile App butuh payload kecil, Web butuh data lengkap).
-   API publik yang membebankan kuota pemakaian per API Key ke pihak ketiga.

### Kapan Jangan Pakai

-   Arsitektur Monolith sederhana (cukup gunakan Reverse Proxy biasa seperti NGINX).
-   Tim kecil dengan traffic rendah di mana memelihara gateway cluster hanya menambah kerumitan infrastruktur (_premature complexity_).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`gateway_latency_seconds` (Histogram p50/p95/p99)**: Waktu pemrosesan internal gateway itu sendiri (di luar waktu tunggu upstream). Harus < 5 milidetik.
-   **`auth_verification_failures_total` (Counter)**: Lonjakan token invalid (bisa mengindikasikan serangan brute force atau issue sinkronisasi auth service).
-   **`upstream_service_errors_total{service="order-svc"}` (Counter)**: Error yang dikembalikan per masing-masing microservice hilir.
-   **`request_body_size_bytes` (Histogram)**: Memantau ukuran payload yang masuk agar tidak menghabiskan memory buffer gateway.

## 9. Studi Kasus Industri Nyata

### Konsolidasi BFF & Agregasi Request pada Super-App Mobile

- **Tantangan Rekayasa Riil (The Problem)**:
  Sebuah platform *super-app* berkembang pesat dari monolit menjadi 60+ microservices independen. Untuk menampilkan layar beranda aplikasi mobile, aplikasi klien harus menembak 9 service berbeda secara terpisah (layanan saldo dompet, notifikasi promo, driver terdekat, riwayat pesanan, profil pengguna, dll). Di jaringan seluler 4G yang tidak stabil, panggilan paralel ini memicu *connection congestion*, menghabiskan baterai pengguna, dan membuat latensi loading beranda melonjak hingga 2.500ms (akibat efek *tail latency* dari service terlambat). Selain itu, setiap tim microservice mengulang-ulang penulisan kode verifikasi JWT, CORS, dan rate limiting yang tidak seragam.

- **Implementasi Solusi Industri (The Architectural Solution)**:
  1. **Centralized API Gateway & BFF Pattern**: Diimplementasikan API Gateway terpusat (berbasis Envoy/Kong) yang bertindak sebagai gerbang tunggal di depan seluruh microservices.
  2. **Security & Identity Offloading**: Gateway memvalidasi tanda tangan kriptografi JWT, memeriksa status revokasi token di Redis, lalu menyuntikkan header identitas tepercaya (`X-User-ID`, `X-User-Role`) ke jaringan VPC internal. Microservice hilir tidak perlu lagi membebani CPU mereka dengan verifikasi kriptografi token.
  3. **Aggregated Endpoint Orchestration**: Gateway menyediakan rute khusus mobile `/api/v1/home-feed` yang memanggil 9 microservice internal secara asinkron lewat jaringan gRPC lokal super cepat, memfilter field yang tidak dibutuhkan layar smartphone, lalu mengembalikan satu payload JSON padat kepada client.

- **Dampak & Hasil Sistem (The Engineering Outcome)**:
  - Durasi rendering beranda mobile terpangkas 85%, dari 2.500ms menjadi hanya 310ms.
  - Jumlah round-trip request jaringan dari smartphone pengguna berkurang dari 9 koneksi HTTP menjadi 1 koneksi terpadu.
  - Inkonsistensi aturan rate limiting dan celah keamanan token di 60 microservice tereliminasi sepenuhnya karena kebijakan keamanan dikelola terpusat di gateway.

## 10. Fakta Menarik & Gotchas Production

-   **The Smart Gateway Anti-pattern**: Kesalahan fatal paling umum adalah menulis kode database query atau logika transaksi bisnis di dalam API Gateway. Gateway seharusnya hanya bertindak sebagai "duri pemindah rute dan satpam", bukan tempat komputasi bisnis.
-   **JWT Decoding CPU Bottleneck**: Di high-throughput (misal 50.000 req/sec), melakukan verifikasi signature kriptografi RSA/ECDSA pada setiap token JWT di gateway bisa menghabiskan 80% CPU. Solusinya: gunakan symmetric hashing yang efisien (HS256) atau cache status validitas token di memory gateway selama beberapa detik.

## 11. Rangkuman Singkat

1. **Pintu Masuk Terpadu**: Mengonsolidasikan autentikasi, otorisasi, SSL termination, dan rate limiting di satu tempat sebelum request menyentuh microservice.
2. **Efisiensi Klien Mobile**: Pola agregasi request memangkas puluhan panggilan API menjadi satu respons terpadu, menghemat kuota dan baterai pengguna.
3. **Waspadai Anti-pattern Smart Gateway**: Jangan pernah menyisipkan query database atau kalkulasi bisnis di gateway; pertahankan peran gateway sebagai perantara lalu lintas yang ramping.
4. **Perlindungan Internal**: Microservice di belakang gateway aman di jaringan privat dan dapat fokus mengeksekusi logika bisnis tanpa terbebani validasi keamanan berulang.
