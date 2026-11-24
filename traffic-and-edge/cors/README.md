# CORS (Cross-Origin Resource Sharing) & Preflight

## 1. Skenario Sistem Nyata

Pada sebuah arsitektur modern yang memisahkan frontend dan backend secara independen:
- Aplikasi web frontend (React / Next.js) di-hosting pada domain: `https://app.tokokita.com` (atau `http://localhost:3000` di lingkungan lokal).
- Layanan API backend (Go) berjalan pada domain server terpisah: `https://api.tokokita.com` (atau `http://localhost:8080`).

Ketika pengguna mengklik tombol **"Bayar Pesanan"**, JavaScript di browser mengeksekusi request:
```javascript
fetch("https://api.tokokita.com/v1/checkout", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer eyJhbGciOi..."
  },
  body: JSON.stringify({ order_id: 101 })
});
```

Tiba-tiba di console browser muncul pesan error merah yang sangat terkenal:
```text
Access to fetch at 'https://api.tokokita.com/v1/checkout' from origin 'https://app.tokokita.com' 
has been blocked by CORS policy: Response to preflight request doesn't pass access control check: 
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

Apa yang sebenarnya terjadi di balik layar?
1. Browser modern menerapkan aturan keamanan ketat **Same-Origin Policy (SOP)** untuk mencegah situs jahat (misal `https://attacker-phishing.com`) membaca data sensitif dari bank atau toko online yang sedang dibuka pengguna di tab sebelah.
2. Karena request menggunakan header kustom (`Authorization` dan `Content-Type: application/json`), browser secara otomatis menembakkan request pemeriksaan awal (*Preflight Request*) menggunakan metode **`OPTIONS`** sebelum mengirimkan request `POST` yang sesungguhnya.
3. Karena backend Go belum dikonfigurasi untuk menyetujui origin `https://app.tokokita.com`, browser menolak menampilkan respon ke JavaScript, dan proses pembayaran gagal total!

Jika tidak dipahami dengan benar di sistem bertrafik tinggi, ketiadaan konfigurasi *Preflight Caching* dapat menyebabkan **volume request ke backend Go berlipat ganda 2x (100% trafik ekstra)** hanya untuk melayani request `OPTIONS` yang berulang-ulang!

## 2. Apa Itu dan Buat Apa?

Cross-Origin Resource Sharing (CORS) adalah mekanisme keamanan standar berbasis header HTTP yang memungkinkan browser mengizinkan atau menolak aplikasi web yang berjalan di satu *origin* mengakses sumber daya dari *origin* yang berbeda.

Sebuah *Origin* didefinisikan oleh tiga komponen: **Protokol + Domain + Port**.
Dua URL dianggap berbeda origin jika salah satu dari ketiganya berbeda:
- `http://api.com` vs `https://api.com` *(Beda protokol: HTTP vs HTTPS)*
- `https://app.com` vs `https://api.com` *(Beda subdomain)*
- `http://localhost:3000` vs `http://localhost:8080` *(Beda port)*

### Jenis Request CORS:
1. **Simple Request**: Request `GET`, `HEAD`, atau `POST` standar dengan header sederhana (tanpa header kustom). Browser langsung mengirim request tanpa pemeriksaan awal.
2. **Preflight Request**: Request yang menggunakan metode berbahaya (`PUT`, `DELETE`, `PATCH`) atau header kustom (`Authorization`, `Content-Type: application/json`). Browser wajib mengirim request probe `OPTIONS` terlebih dahulu untuk meminta izin ke backend.

## 3. Kenapa Harus Dipahami di High-Throughput?

1. **Mengurangi 50% Trafik yang Sia-Sia (*Preflight Caching*)**: Dengan menyetel header `Access-Control-Max-Age: 86400` (cache 24 jam), browser hanya mengirim 1 kali request `OPTIONS` sehari per endpoint, menghemat jutaan request handshake ke server backend Go.
2. **Mencegah Kebocoran Data Sesi (*Credential Theft*)**: Mengetahui cara mengatur `Access-Control-Allow-Credentials: true` secara aman dengan whitelist origin dinamis tanpa membuka celah wildcard berbahaya.
3. **Membongkar Mitos Keamanan**: Memahami bahwa **CORS BUKAN tameng keamanan backend terhadap hacker**. Peretas yang menggunakan cURL, Postman, atau bot Python dapat menembak backend secara langsung tanpa terpengaruh aturan CORS (karena CORS murni ditegakkan oleh Web Browser client, bukan server!).

## 4. Cara Kerjanya (Diagram Alur Preflight Request & Caching)

```
[Browser: https://app.com]                          [Backend Go: https://api.com]
          |                                                       |
          | 1. [Preflight Check]: OPTIONS /v1/checkout ---------> |
          |    Headers:                                           |
          |    Origin: https://app.com                            |
          |    Access-Control-Request-Method: POST                |
          |    Access-Control-Request-Headers: Authorization      |
          |                                                       |
          | <---------------------------------------------------- |
          | 2. Balas HTTP 204 No Content:                         |
          |    Access-Control-Allow-Origin: https://app.com       |
          |    Access-Control-Allow-Methods: POST, GET, OPTIONS   |
          |    Access-Control-Allow-Headers: Authorization        |
          |    Access-Control-Max-Age: 86400 (Cache 24 Jam!)      |
          |                                                       |
  [Browser Memeriksa Izin: OK!]                                   |
          |                                                       |
          | 3. [Actual Request]: POST /v1/checkout -------------> |
          |    Headers: Authorization: Bearer token...            |
          |                                                       |
          | <---------------------------------------------------- |
          | 4. Balas HTTP 200 OK (Data Transaksi Selesai)         |
```

## 5. Cuplikan Logika Inti (High-Performance CORS Middleware di Go)

Berikut implementasi middleware CORS berkinerja tinggi di Go tanpa dependensi pihak ketiga yang mendukung whitelist dinamis, preflight caching, dan penanganan HTTP 204:

```go
package main

import (
	"net/http"
)

// allowedOrigins mendefinisikan domain frontend resmi yang diizinkan mengakses API
var allowedOrigins = map[string]bool{
	"https://app.tokokita.com": true,
	"https://admin.tokokita.com": true,
	"http://localhost:3000":     true, // Lingkungan development
}

func CORSMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		origin := r.Header.Get("Origin")

		// 1. Periksa apakah origin client terdaftar di whitelist
		if allowedOrigins[origin] {
			w.Header().Set("Access-Control-Allow-Origin", origin)
			w.Header().Set("Access-Control-Allow-Credentials", "true")
		}

		// 2. Tangani Preflight Request (OPTIONS)
		if r.Method == http.MethodOptions {
			w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, PATCH, OPTIONS")
			w.Header().Set("Access-Control-Allow-Headers", "Authorization, Content-Type, X-Request-ID")
			
			// PENTING UNTUK HIGH-THROUGHPUT: Cache hasil preflight selama 24 jam (86400 detik)
			// Menghindari 50% request tambahan yang membebani server!
			w.Header().Set("Access-Control-Max-Age", "86400")

			// Langsung balas 204 No Content tanpa meneruskan ke handler bisnis
			w.WriteHeader(http.StatusNoContent)
			return
		}

		// 3. Teruskan request aktual (GET, POST, dll) ke handler berikutnya
		next.ServeHTTP(w, r)
	})
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Melindungi Pengguna dari Eksploitasi Antar-Situs**: Mencegah website berbahaya menembak API privat korban menggunakan cookie sesi yang tersimpan di browser.
- **Memungkinkan Pemisahan Total Frontend & Backend**: Tim web dapat mendeploy aplikasi di CDN/Vercel secara terpisah dari cluster backend Kubernetes.
- **Granular Access Control**: Server backend memiliki kendali mutlak untuk menentukan origin, metode HTTP, dan header apa saja yang diizinkan.

### Kekurangan
- **Overhead Latensi Jaringan (*Double Round-Trip*)**: Preflight `OPTIONS` menambahkan 1 round-trip ekstra sebelum request aktual dieksekusi jika caching tidak dikonfigurasi dengan benar.
- **Sering Menyebabkan Frustrasi Developer**: Salah satu sumber kebingungan nomor 1 bagi developer pemula saat integrasi lokal.
- **Bukan Solusi Keamanan Server-Side**: Hanya efektif di lingkungan browser web; tidak melindungi API dari serangan bot scraping atau script backend eksternal.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Dikonfigurasi
- API backend publik yang melayani aplikasi Single Page Application (SPA) di domain atau port yang berbeda.
- Backend microservices yang diakses oleh dashboard admin web internal pada domain khusus.
- Layanan multi-tenant yang menyediakan widget JavaScript untuk disematkan di website pihak ketiga.

### Kapan Tidak Perlu (CORS Tidak Berlaku)
- Komunikasi antar backend microservices (*Server-to-Server communication* via gRPC atau HTTP).
- Aplikasi monolitik tradisional di mana frontend HTML dirender langsung oleh server backend pada origin yang sama (*Server-Side Rendering* di domain yang sama).
- Mobile App murni (Flutter, React Native, Swift, Kotlin) yang tidak berjalan di dalam WebView browser.

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

- **`http_requests_total{method="OPTIONS"}` vs Actual Methods (Counter)**: Membandingkan rasio request OPTIONS dibanding request bisnis. Jika rasio mendekati 1:1, berarti header `Access-Control-Max-Age` gagal di-cache oleh browser pengguna.
- **`cors_origin_rejected_total` (Counter)**: Jumlah request lintas-domain yang ditolak karena tidak ada dalam whitelist. Lonjakan metrik ini dapat menandakan upaya phishing atau domain baru yang lupa didaftarkan.

## 9. Studi Kasus Industri Nyata

### Optimasi Preflight OPTIONS & Multi-Tenant Ingress pada Platform E-Commerce

- **Tantangan Rekayasa Riil (The Problem)**:
  Sebuah platform SaaS e-commerce mengelola ribuan toko online independen dengan domain kustom (`brand-a.com`, `brand-b.id`) yang seluruhnya memanggil API checkout terpusat di `api.commerce.com`. Developer awalnya menyetel `Access-Control-Allow-Origin: *`. Ketika sistem beralih menggunakan cookie sesi terenkripsi (`HttpOnly; SameSite=None; Secure`), seluruh browser pelanggan langsung memblokir transaksi checkout karena standar keamanan browser melarang pengiriman kredensial jika origin disetel wildcard `*`. Sebagai solusinya, tim memasang middleware yang memverifikasi domain toko ke database PostgreSQL pada setiap request `OPTIONS`. Hasilnya fatal: volume trafik ke database melonjak dua kali lipat akibat badai request `OPTIONS` (50.000 RPS), latensi checkout membengkak menjadi 650ms, dan koneksi database kolaps.

- **Implementasi Solusi Industri (The Architectural Solution)**:
  1. **Edge-Level Dynamic Origin Reflection**: Logika verifikasi CORS dipindahkan dari aplikasi backend ke reverse proxy edge (Envoy / Cloudflare Workers). Whitelist domain toko yang aktif di-cache ke dalam shared memory berkecepatan tinggi (*in-memory lookup* dalam 0.05ms) tanpa menyentuh database.
  2. **Aggressive Preflight Caching**: Edge proxy menyertakan header `Access-Control-Max-Age: 86400` (24 jam) pada respon preflight, sehingga browser pelanggan hanya melakukan request `OPTIONS` satu kali sehari per domain toko.
  3. **Immediate 204 No Content Termination**: Request preflight `OPTIONS` langsung dijawab di layer proxy dengan status `204 No Content` dan header CORS yang sesuai, membebaskan container backend Go dari keharusan memproses request preflight.
  4. **Strict Header Whitelisting**: Hanya header bisnis yang dibutuhkan (`Authorization`, `Content-Type`, `X-Idempotency-Key`) yang dimasukkan ke `Access-Control-Allow-Headers`.

- **Dampak & Hasil Sistem (The Engineering Outcome)**:
  - Volume request `OPTIONS` ke cluster backend Go dan database PostgreSQL anjlok hingga 97%.
  - Latensi checkout sisi klien (*initial interaction latency*) terpangkas drastis dari 650ms menjadi hanya 28ms.
  - Seluruh toko online merchant dapat melayani transaksi pembayaran berbasis cookie dengan aman tanpa melanggar kepatuhan Same-Origin Policy browser.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Jebakan `Access-Control-Allow-Origin: *` Bersama Credentials**: Menyetel wildcard `*` padahal aplikasi membutuhkan pengiriman cookie atau kredensial autentikasi. Standar browser akan menolak request tersebut secara otomatis jika `Access-Control-Allow-Credentials: true` digabungkan dengan wildcard `*`!  
   *Solusi*: Periksa header `Origin` request secara dinamis, lalu pantulkan origin tersebut di respon jika terdaftar dalam whitelist.
2. **Lupa Mengatur `Access-Control-Max-Age`**: Tanpa header ini, browser akan mengirimkan request `OPTIONS` sebelum *setiap satu* pemanggilan API `POST/PUT/DELETE`. Di traffic 50.000 RPS, backend Anda sebenarnya menerima 100.000 RPS!  
   *Solusi*: Pasang `Access-Control-Max-Age: 86400` (1 hari) atau nilai maksimal yang didukung browser.
3. **Mengira CORS Melindungi Backend dari Serangan Hacker**: Membiarkan endpoint API tanpa proteksi autentikasi karena merasa "sudah aman dibatasi oleh CORS". Hacker dapat menembak endpoint tersebut menggunakan Python atau cURL dalam 1 detik tanpa terpengaruh CORS sama sekali!

## 11. Rangkuman Singkat

1. **Proteksi Sisi Browser**: CORS adalah mekanisme keamanan browser web untuk mencegah situs jahat menyalahgunakan hak akses pengguna, bukan pengganti autentikasi server.
2. **Hindari Wildcard untuk Kredensial**: Jangan gabungkan `Access-Control-Allow-Origin: *` dengan `Allow-Credentials: true`; pantulkan origin yang valid dari whitelist dinamis.
3. **Wajib Pasang Preflight Caching**: Sertakan header `Access-Control-Max-Age` (misal 24 jam) untuk menghentikan pemborosan jaringan akibat request OPTIONS ganda.
4. **Selesaikan di Edge Proxy**: Terminasi preflight OPTIONS langsung di layer API Gateway atau reverse proxy dengan respons `204 No Content` tanpa membebani container aplikasi.
