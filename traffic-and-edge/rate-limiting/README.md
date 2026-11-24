# Rate Limiting

## 1. Skenario Sistem Nyata

Pada endpoint autentikasi pengguna berskala produksi `POST /api/v1/auth/login`, Setiap kali request masuk, server harus mencari data user di database PostgreSQL dan memverifikasi password menggunakan algoritma hashing *bcrypt* atau *Argon2* (yang sengaja dirancang memakan waktu ~80 milidetik CPU per hash agar aman dari peretasan offline).

Jika endpoint ini dibiarkan terbuka tanpa proteksi:
- Seorang penyerang menggunakan skrip bot otomatis untuk melancarkan serangan *credential stuffing* atau *brute force*, menembakkan 3.000 kombinasi email dan password per detik dari satu alamat IP.
- Menjalankan 3.000 komputasi bcrypt per detik langsung melahap 100% kapasitas CPU seluruh server backend dalam sekejap.
- Akibatnya: Pengguna sah yang ingin login atau checkout belanja mengalami connection timeout, dan seluruh API backend lumpuh total (*Denial of Service*).

**Solusi dengan Rate Limiter (misal: Redis Token Bucket / Sliding Window):**
Kamu menerapkan aturan rate limiting ketat pada endpoint tersebut: maksimal 5 percobaan login per 1 menit per IP address. Request ke-1 sampai ke-5 diproses normal. Namun, begitu request ke-6 masuk di jendela waktu yang sama, Rate Limiter di layer gateway langsung menolak request tersebut dalam waktu < 1 milidetik dengan status **`HTTP 429 Too Many Requests`** dan menyertakan header `Retry-After: 60`. CPU backend dan database terlindungi seutuhnya dari kelebihan beban komputasi.

## 2. Apa Itu dan Buat Apa?

Rate Limiting adalah mekanisme kontrol lalu lintas untuk membatasi frekuensi request yang diizinkan masuk ke sebuah sistem atau API dalam jangka waktu tertentu (misalnya: maksimum 100 request per menit per IP address).

Jika batas terlampaui, sistem akan langsung menolak request berlebih tersebut dengan status HTTP standard: **`429 Too Many Requests`**.

Tujuan utamanya:
- **Proteksi dari Denial of Service (DoS/DDoS)**: Mencegah serangan bot atau script jahat yang membombardir server.
- **Mencegah Scraping & Brute Force**: Membatasi percobaan tebak password di halaman login (misal: max 5 percobaan per menit).
- **Menjamin Keadilan (Fair Usage Policy)**: Mencegah satu pengguna rakus (*noisy neighbor*) memonopoli kapasitas server hingga merugikan pengguna lain.
- **Kontrol Biaya API Pihak Ketiga**: Mencegah tagihan bengkak saat sistem memanggil API berbayar (seperti OpenAI, Twilio, atau Payment Gateway).

### Algoritma Rate Limiting Populer:
- **Token Bucket**: Token diisi ke dalam ember dengan laju konstan. Setiap request mengambil 1 token. Jika ember kosong, request ditolak. Sangat populer karena mengizinkan lonjakan trafik sesaat (*burst traffic*).
- **Leaky Bucket**: Request masuk ke penampungan dan dikeluarkan ke server dengan laju tetap (seperti air yang menetes dari ember bocor). Menghasilkan trafik keluar yang sangat mulus.
- **Fixed Window Counter**: Menghitung request dalam blok waktu tetap (misal: detik 0-59). Kelemahannya: rentan kebobolan 2x lipat kuota di batas pergantian menit (*window edge burst*).
- **Sliding Window Log / Counter**: Menghitung request berdasarkan rentang waktu dinamis ke belakang (misal 60 detik terakhir dari milidetik sekarang). Sangat akurat tetapi butuh memori lebih besar.

## 3. Kenapa Harus Pakai Ini di High-Throughput?

Di sistem skala besar, kapasitas database (khususnya *write capacity*) dan thread CPU selalu memiliki batas jenuh. 

Tanpa Rate Limiting:
- Satu script crawler eksternal yang agresif bisa menyedot 90% resource database, menyebabkan query pengguna biasa mengalami timeout.
- Bug loop tak hingga (*infinite loop*) di aplikasi mobile client bisa tanpa sengaja melakukan DDoS ke server sendiri saat rilis versi baru.

Dengan Rate Limiting di level terdepan (API Gateway atau reverse proxy), jutaan request sampah bisa langsung dipangkas dalam hitungan mikrodetik (menggunakan memori Redis) sebelum sempat menyentuh logika aplikasi atau database.

## 4. Cara Kerjanya (Diagram Alur - Token Bucket)

```
                       +-------------------+
                       |  Inbound Request  |
                       +-------------------+
                                 |
                                 v
                     +-----------------------+
                     | Rate Limiter (Memory) |
                     | [ Token Bucket: IP ]  |
                     +-----------------------+
                                 |
                     Apakah token > 0?
                                 |
                 +---------------+---------------+
                 |                               |
              [ YES ]                         [ NO ]
                 |                               |
                 v                               v
         Kurangi 1 Token                 Tolak Request
                 |                               |
                 v                               v
         +---------------+               +---------------+
         |  Teruskan ke  |               | Return HTTP   |
         | Backend Svc   |               | 429 Too Many  |
         +---------------+               | Requests      |
                                         +---------------+
```

## 5. Cuplikan Logika Inti (Token Bucket di Go dengan Redis)

Berikut adalah logika algoritma Token Bucket yang dieksekusi secara atomik menggunakan Go dan Redis:

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

var ctx = context.Background()

// IsRequestAllowed memeriksa kuota Token Bucket secara atomik dengan Redis
func IsRequestAllowed(rdb *redis.Client, clientID string, maxCapacity int64, refillRatePerSec float64) bool {
	key := fmt.Sprintf("ratelimit:%s", clientID)
	now := time.Now().Unix()

	// Gunakan Redis Hashes untuk menyimpan [tokens, last_updated]
	vals, err := rdb.HMGet(ctx, key, "tokens", "last_updated").Result()
	var tokens float64 = float64(maxCapacity)
	var lastUpdated int64 = now

	if err == nil && vals[0] != nil && vals[1] != nil {
		fmt.Sscanf(vals[0].(string), "%f", &tokens)
		fmt.Sscanf(vals[1].(string), "%d", &lastUpdated)

		// 1. Refill token berdasarkan selisih waktu
		elapsed := now - lastUpdated
		tokens = tokens + (float64(elapsed) * refillRatePerSec)
		if tokens > float64(maxCapacity) {
			tokens = float64(maxCapacity)
		}
	}

	// 2. Konsumsi token jika tersedia
	if tokens >= 1.0 {
		tokens -= 1.0
		rdb.HSet(ctx, key, "tokens", tokens, "last_updated", now)
		rdb.Expire(ctx, key, 1*time.Hour)
		return true // Request diizinkan lewat
	}

	return false // 429 Too Many Requests (Rate limit terlampaui)
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Stabilitas sistem terjamin**: Menjaga server tetap berada di bawah batas kapasitas aman.
- **Biaya operasional terkendali**: Menghindari pemborosan server autoscaling akibat serangan bot.
- **Keamanan meningkat**: Menangkal serangan brute-force credential stuffing secara efektif.

### Kekurangan
- **False Positives**: Pengguna sah yang berada di balik jaringan kantor besar (satu public NAT IP) bisa terkena blokir bersamaan jika rate limit hanya berbasis IP.
- **Overhead State Terdistribusi**: Menyimpan counter rate limit di Redis cluster menambahkan latensi jaringan mikro (sekitar 1-2 ms per request).
- **Kompleksitas Client**: Client harus menangani response HTTP 429 dan membaca header `Retry-After`.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Seluruh endpoint API publik (terutama yang mahal secara komputasi: endpoint pencarian, ekspor data, upload file).
- Endpoint autentikasi sensitif: `/login`, `/register`, `/forgot-password`, `/send-otp`.
- API komersial berbasis kuota langganan berbayar.

### Kapan Jangan Pakai
- Komunikasi internal private microservice yang sudah terpercaya dan memiliki kontrol kuota tersendiri (kecuali antar domain besar yang berbeda tim).
- Health check endpoint internal yang diakses load balancer (jangan sampai load balancer terblokir saat memeriksa kesehatan server!).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

- **`rate_limit_hits_total` (Counter)**: Jumlah request yang berhasil melewati rate limiter.
- **`rate_limit_exceeded_total` (Counter)**: Jumlah request yang diblokir dengan HTTP 429. Lonjakan tajam menandakan serangan bot atau bug infinite loop di client.
- **`rate_limit_token_depth` (Gauge)**: Rata-rata persentase sisa token per client (mendeteksi seberapa dekat ekosistem dari batas saturasi).
- **`rate_limiter_latency_microseconds` (Histogram)**: Overhead evaluasi key di Redis. Wajib di bawah 1 milidetik.

## 9. Studi Kasus Nyata di Industri

- **Stripe**: Menerapkan arsitektur 4-layer rate limiter yang sangat canggih untuk melindungi API finansial mereka: *Request Rate Limiter* (mencegah lonjakan dasar), *Allocation Rate Limiter* (kuota berbasis tier akun), *Concurrent Request Limiter* (membatasi request paralel yang sedang aktif), dan *Fleet Usage Limiter* (penyisihan beban saat terjadi insiden downstream).
- **Cloudflare**: Menjalankan rate limiting di ratusan server Edge menggunakan algoritma Sliding Window counter. Cloudflare mampu mendeteksi dan meredam miliaran request serangan Layer 7 DDoS dalam hitungan detik sebelum serangan sempat menyentuh infrastruktur origin pelanggan.

## 10. Fakta Menarik & Gotchas Production

- **The Standard HTTP Headers**: Rate limiter yang baik selalu menyertakan 3 header respon standar:
  - `X-RateLimit-Limit`: Jumlah maksimum request yang diizinkan dalam satu jendela waktu.
  - `X-RateLimit-Remaining`: Sisa kuota request yang masih bisa digunakan.
  - `Retry-After`: Jumlah detik yang harus ditunggu client sebelum boleh mencoba kembali.
- **Race Condition di Redis**: Kesalahan umum developer junior adalah mengimplementasikan rate limiter dengan pola: `GET count`, cek jika `< limit`, lalu `INCR count`. Di kondisi high throughput, jeda waktu antara `GET` dan `INCR` menyebabkan ribuan request lolos secara serentak (*race condition*). Solusinya: gunakan **Redis Lua Script** agar operasi pembacaan dan pemotongan token dieksekusi secara atomik (*atomic transaction*).
