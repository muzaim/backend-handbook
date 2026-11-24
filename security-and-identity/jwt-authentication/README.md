# JWT & Token-Based Authentication

## 1. Skenario Sistem Nyata

Pada sebuah platform super-app dengan 3.000.000 (3 juta) pengguna aktif harian dan 20 microservices terpisah (Order, Payment, User, Notification, Logistics, dll):

Jika sistem autentikasi mengandalkan **Stateful Session Tradisional (Cookie + Server Session)**:

-   Setiap kali pengguna berpindah halaman atau mengklik tombol, setiap microservice terpaksa melakukan panggilan jaringan (_remote call_) ke satu database session pusat (Redis atau PostgreSQL) untuk menjawab pertanyaan: _"Apakah session ID ini valid dan milik siapa?"_.
-   Pada trafik puncak 60.000 RPS, database session pusat menerima 60.000 query verifikasi per detik murni hanya untuk memeriksa login. Ketika database session mengalami _network latency_, seluruh 20 microservices ikut melambat dan lumpuh (_Single Point of Failure_).

**Solusi dengan JWT (JSON Web Token):**
Sistem beralih ke autentikasi berbasis token mandiri (_self-contained stateless token_):

1. Saat pengguna login, Auth Service memverifikasi password dan menerbitkan sebuah JWT string yang ditandatangani secara kriptografis (_cryptographic signature_) menggunakan algoritma HMAC-SHA256 atau RSA.
2. Token memuat data non-sensitif (seperti `user_id`, `role`, dan masa kedaluwarsa `exp`) langsung di dalam payload JSON-nya.
3. Seluruh 20 microservices downstream dapat memverifikasi keaslian token secara independen dan instan **tanpa perlu memanggil database session sama sekali**. Setiap service cukup memverifikasi tanda tangan kriptografis token menggunakan public key / shared secret yang di-cache di memori lokal service.

Namun, muncul tantangan industri berikutnya: _Jika JWT tidak tersimpan di database, bagaimana cara membatalkan token (force logout) jika akun pengguna dicuri atau hak aksesnya dicabut?_
Jawabannya adalah pola standar industri: **Short-Lived Access Token (15 menit)** dikombinasikan dengan **Refresh Token Rotation (7 hari)**.

## 2. Apa Itu dan Buat Apa?

JSON Web Token (JWT - RFC 7519) adalah standar terbuka yang mendefinisikan format kompak dan mandiri (_self-contained_) untuk mentransmisikan informasi identitas antar pihak secara aman dalam bentuk objek JSON.

Struktur JWT terdiri dari 3 bagian yang dipisahkan oleh tanda titik (`.`):

```
header.payload.signature
```

1. **Header**: Memuat jenis token (`JWT`) dan algoritma penandatanganan (misal `HS256` atau `RS256`).
2. **Payload (Claims)**: Memuat pernyataan identitas pengguna, seperti `sub` (User ID), `role`, `iat` (issued at), dan `exp` (expiration time). **PENTING: Payload hanya di-encode dengan Base64, bukan dienkripsi! Siapa pun bisa membaca isinya.**
3. **Signature (Tanda Tangan)**: Hash kriptografis dari gabungan Header + Payload + Secret Key server. Mencegah pihak luar memalsukan isi payload (misal mengubah role dari `user` menjadi `admin`).

## 3. Kenapa Harus Pakai Ini ?

Dalam sistem backend berskala masif:

1. **Mengeliminasi Beban I/O Database Autentikasi**: Pengecekan autentikasi terjadi murni melalui kalkulasi matematika CPU lokal di setiap server microservices.
2. **Mendukung Penskalaan Horizontal Sempurna**: Server backend bersifat _stateless_; pengguna yang login di Server A dapat mengirim request berikutnya ke Server B atau Server Z tanpa perlu sinkronisasi memori sesi antar server.
3. **Standar Lintas Domain (Cross-Domain & Mobile Friendly)**: Tidak terikat pada domain cookie browser, sehingga bekerja mulus di aplikasi mobile (iOS/Android), desktop, dan integrasi API pihak ketiga.

## 4. Cara Kerjanya (Pola Access Token & Refresh Token Rotation)

```
[Client / Mobile App]               [API Gateway / Service]            [Auth Service & Redis]
         |                                     |                                 |
         | 1. POST /login (email + password) ----------------------------------> |
         |                                     |                           (Validasi DB)
         | <-------------------------------------------------------------------- |
         |    Kembalikan:                      |                                 |
         |    * Access Token (Exp: 15 Menit)   |                                 |
         |    * Refresh Token (Exp: 7 Hari)    |                                 |
         |                                     |                                 |
         | 2. GET /orders (Bearer AccessToken) |                                 |
         | ----------------------------------> |                                 |
         |                               [Verifikasi Signature                   |
         |                                Secara Lokal via CPU]                  |
         | <---------------------------------- |                                 |
         |    HTTP 200 OK (Data Orders)        |                                 |
         |                                     |                                 |
         | ... 15 Menit Kemudian (Access Token Expired) ...                      |
         |                                     |                                 |
         | 3. GET /orders (Bearer AccessToken) |                                 |
         | ----------------------------------> |                                 |
         | <---------------------------------- |                                 |
         |    HTTP 401 Unauthorized (Token Exp)|                                 |
         |                                     |                                 |
         | 4. POST /auth/refresh (Send Refresh Token) -------------------------> |
         |                                     |                           [Cek Refresh Token
         |                                     |                            di Redis / DB]
         | <-------------------------------------------------------------------- |
         |    Kembalikan: Access Token Baru + Refresh Token Baru (Rotasi!)       |
```

## 5. Cuplikan Logika Inti (JWT Generator & Middleware Verifier di Go)

Berikut implementasi produksi menggunakan library resmi `github.com/golang-jwt/jwt/v5` di Go:

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"net/http"
	"strings"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

var jwtSecretKey = []byte("super_secret_signing_key_production_2026")

// CustomClaims mendefinisikan isi payload JWT
type CustomClaims struct {
	UserID string `json:"user_id"`
	Role   string `json:"role"`
	jwt.RegisteredClaims
}

// GenerateAccessToken membuat token berdurasi singkat (15 menit)
func GenerateAccessToken(userID, role string) (string, error) {
	claims := CustomClaims{
		UserID: userID,
		Role:   role,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
			Issuer:    "company-auth-service",
		},
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(jwtSecretKey)
}

// JWTMiddleware memvalidasi token secara offline tanpa menyentuh database
func JWTMiddleware(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		authHeader := r.Header.Get("Authorization")
		if authHeader == "" || !strings.HasPrefix(authHeader, "Bearer ") {
			http.Error(w, "Unauthorized: missing bearer token", http.StatusUnauthorized)
			return
		}

		tokenString := strings.TrimPrefix(authHeader, "Bearer ")

		// Parse dan verifikasi tanda tangan token secara komputasi CPU
		token, err := jwt.ParseWithClaims(tokenString, &CustomClaims{}, func(t *jwt.Token) (interface{}, error) {
			// Cegah serangan Algoritma "none"
			if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
			}
			return jwtSecretKey, nil
		})

		if err != nil || !token.Valid {
			http.Error(w, "Unauthorized: invalid or expired token", http.StatusUnauthorized)
			return
		}

		claims, ok := token.Claims.(*CustomClaims)
		if !ok {
			http.Error(w, "Unauthorized: invalid claims", http.StatusUnauthorized)
			return
		}

		// Teruskan konteks UserID ke handler bisnis berikutnya
		ctx := context.WithValue(r.Context(), "user_id", claims.UserID)
		ctx = context.WithValue(ctx, "user_role", claims.Role)
		next(w, r.WithContext(ctx))
	}
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Stateless & Performa Tinggi**: Tidak ada beban query database sesi di setiap request HTTP.
-   **Decoupled Architecture**: Layanan microservices terpisah dapat memvalidasi token tanpa bergantung pada ketersediaan Auth Service.
-   **Portabilitas Lintas Platform**: Sangat bersahabat dengan arsitektur SPA (React/Vue/Next.js) dan mobile apps.

### Kekurangan

-   **Tidak Bisa Dibatalkan Seketika (_Revocation Problem_)**: Jika token dicuri, penyerang tetap bisa mengakses sistem hingga durasi token habis (`exp`).
-   **Ukuran Header HTTP Lebih Besar**: String JWT berukuran 300 hingga 1.000 byte (jauh lebih besar dibanding Cookie Session 32 byte), menambah sedikit bandwidth pada setiap request.
-   **Data Payload Bersifat Publik**: Siapa pun yang memegang token dapat men-decode payload JSON; data rahasia tidak boleh ditaruh di dalam JWT.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Arsitektur microservices terdistribusi dengan lalu lintas tinggi di mana memusatkan validasi sesi ke satu database akan menjadi bottleneck.
-   RESTful API publik atau integrasi aplikasi mobile (iOS/Android) yang membutuhkan otentikasi stateless.
-   Sistem Single Sign-On (SSO) lintas domain aplikasi.

### Kapan Jangan Pakai

-   Aplikasi monolitik tradisional sederhana yang hanya melayani pengguna web browser (menggunakan standard HTTP-Only Cookie Session jauh lebih aman dan mudah di-revoke).
-   Sistem perbankan dengan regulasi yang mewajibkan kemampuan memutuskan koneksi pengguna secara instan di milidetik yang sama saat transaksi anomali terdeteksi.

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`jwt_verification_failures_total` (Counter)**: Jumlah token tidak valid atau tanda tangan salah. Lonjakan metrik ini mengindikasikan adanya serangan brute force atau pergantian secret key.
-   **`jwt_expired_tokens_total` (Counter)**: Jumlah token yang ditolak karena kedaluwarsa.
-   **`refresh_token_reuse_detected` (Counter)**: Mendeteksi indikasi pencurian token (_token theft_) ketika sebuah Refresh Token yang sudah usang dicoba digunakan kembali.

## 9. Studi Kasus Industri Nyata

### Eliminasi Bottleneck Database Sesi pada Arsitektur 50+ Microservices

- **Tantangan Rekayasa Riil (The Problem)**:
  Sebuah platform digital berskala besar mengoperasikan 50+ microservices independen yang melayani 20 juta pengguna aktif. Sistem awalnya menggunakan mekanisme *Stateful Session*: setiap kali sebuah request masuk (mencapai total 80.000 RPS saat jam sibuk), setiap microservice harus melakukan panggilan jaringan I/O ke cluster Redis terpusat untuk memvalidasi `session_id`. Ketika kampanye promo dibuka, Redis cluster mengalami saturasi bandwidth jaringan (*network saturation*) dan CPU mencapai 98% murni hanya untuk melayani pembacaan data sesi. Lonjakan latensi Redis dari 1ms menjadi 250ms langsung melumpuhkan seluruh dependensi microservices secara berantai (*cascading failure*).

- **Implementasi Solusi Industri (The Architectural Solution)**:
  1. **Stateless JWT Migration**: Format otentikasi diubah menjadi JWT yang ditandatangani secara kriptografis (HS256/RS256). Setiap microservice internal memvalidasi integritas token secara lokal di memori CPU mereka sendiri (*zero network I/O*) tanpa pernah menyentuh Redis.
  2. **Short-Lived Access Token (15 Menit)**: Access Token dibatasi masa aktifnya hanya 15 menit untuk memitigasi risiko keamanan jika token dicuri di jaringan publik.
  3. **Refresh Token Rotation & Theft Detection**: Client menyimpan Refresh Token di `HttpOnly Cookie`. Setiap kali Access Token kedaluwarsa, client menembak `/auth/refresh` untuk mendapatkan pasangan token baru, dan token lama langsung dianulir. Jika sebuah token lama yang sudah dianulir dicoba digunakan kembali, sistem mendeteksi adanya pembajakan token (*token theft*) dan langsung mematikan seluruh sesi akun tersebut di database.
  4. **Selective Blacklisting**: Redis tidak lagi menyimpan seluruh sesi pengguna, melainkan hanya menyimpan daftar token yang di-revoke sebelum waktunya (`jti:blacklist:{id}`) dengan TTL sesuai sisa masa aktif token.

- **Dampak & Hasil Sistem (The Engineering Outcome)**:
  - Trafik pembacaan sesi ke cluster Redis anjlok drastis sebesar 99.1% (dari 80.000 RPS menjadi hanya sekitar 700 RPS untuk kebutuhan refresh token dan blacklist check).
  - Latensi verifikasi otentikasi di setiap microservice terpangkas dari 3.8ms menjadi hanya 0.03ms per request.
  - Arsitektur sistem menjadi sepenuhnya terdesentralisasi: tumbangnya Redis tidak lagi melumpuhkan verifikasi request yang membawa Access Token valid.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Menyimpan Password atau Data Rahasia di Payload**: Developer pemula mengira JWT itu dienkripsi. Padahal payload hanya di-encode dengan Base64. Siapa pun dapat membuka situs `jwt.io` dan melihat seluruh isinya dalam 1 detik!  
   _Solusi_: Jangan pernah menaruh password, nomor kartu kredit, atau data sensitif di payload JWT.
2. **Access Token Berdurasi Terlalu Panjang**: Mengatur masa aktif Access Token selama 30 hari karena malas membuat alur Refresh Token. Jika token pengguna bocor di kafe/Wi-Fi publik, akun tersebut tidak bisa diselamatkan selama sebulan penuh.  
   _Solusi_: Pasang batas Access Token maksimal 15 menit, dan gunakan Refresh Token rotasi.
3. **Kerentanan Algoritma "none" (`alg: none`)**: Penyerang mengubah header JWT menjadi `{"alg": "none"}` dan menghapus tanda tangan. Backend yang naif akan menganggap token valid tanpa verifikasi signature!  
   _Solusi_: Selalu validasi metode algoritma penandatanganan secara eksplisit di kode verifier Go.

## 11. Rangkuman Singkat

1. **Verifikasi Stateless Berkecepatan Tinggi**: Mengeliminasi network hop ke database sesi; verifikasi integritas token dieksekusi secara lokal di memori CPU dalam hitungan mikrodetik.
2. **Short-Lived Access Token adalah Kunci**: Batasi masa berlaku access token (10–15 menit) untuk mempersempit jendela bahaya jika terjadi kebocoran token.
3. **Amankan dengan Refresh Token Rotation**: Lindungi akun pengguna dengan rotasi token berkala dan deteksi penggunaan ulang (*token reuse detection*) untuk mencegah pembajakan sesi.
4. **Bukan Media Penyimpanan Rahasia**: Payload JWT hanya di-encode dengan Base64 publik; jangan pernah menaruh kredensial sensitif atau data rahasia di dalam klaim token.
