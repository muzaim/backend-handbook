# Idempotency

## 1. Skenario Sistem Nyata

Pada sistem pembayaran digital berskala produksi, pengguna menekan tombol "Bayar Sekarang" untuk pesanan senilai Rp 500.000 (`POST /api/v1/payments`).

Backend Payment Service menerima request tersebut, berhasil mendebet saldo rekening pengguna di bank pada detik ke-1.8, dan mencatat transaksi ke database. Namun, tepat sebelum respon `HTTP 200 OK` terkirim balik ke ponsel pengguna, sinyal 4G di ponsel pengguna terputus sesaat (_network drop_).

Ponsel pengguna mengalami _Read Timeout_, menganggap pembayaran gagal, dan sistem retry otomatis aplikasi mobile langsung mengirim ulang request `POST /api/v1/payments` yang sama persis beberapa detik kemudian:

-   Tanpa Idempotency: Backend memperlakukan request kedua sebagai transaksi baru. Akibatnya, saldo pengguna **terpotong dua kali (Rp 1.000.000)** dan tercipta dua tagihan ganda (_double charge_). Ini adalah mimpi buruk kepatuhan regulasi finansial dan memicu sengketa pengembalian dana (_chargeback dispute_).

**Solusi dengan Mekanisme Idempotency Key:**
Aplikasi client menyertakan header unik pada setiap transaksi mutasi: `Idempotency-Key: "uuid-pay-8821"`. Saat request retry kedua masuk membawa key yang sama, backend memeriksa database/Redis, mendeteksi bahwa transaksi dengan key tersebut sudah berstatus `COMPLETED`, langsung menolak pendebetan saldo kedua, dan dengan aman mengembalikan salinan struk bukti bayar transaksi pertama. Saldo pengguna dijamin terpotong tepat satu kali (_strictly once_).

## 2. Apa Itu dan Buat Apa?

Idempotency (Idempotensi) adalah sifat sebuah operasi API di mana mengeksekusi operasi tersebut berkali-kali dengan parameter yang sama **akan menghasilkan kondisi akhir sistem (_state_) dan respon yang sama persis** seperti saat operasi pertama kali dijalankan.

Dalam spesifikasi standar protokol HTTP:

-   **`GET`, `HEAD`, `OPTIONS`**: Bersifat _Safe_ dan _Idempotent_ (hanya membaca data).
-   **`PUT`**: Bersifat _Idempotent_ (mengganti seluruh resource; mengupdate alamat user menjadi "Jl. Mawar No. 10" sepuluh kali menghasilkan alamat yang sama persis).
-   **`DELETE`**: Bersifat _Idempotent_ (menghapus user ID 42 sepuluh kali; data tetap terhapus dari sistem).
-   **`POST`**: Secara default **TIDAK Idempotent** (membuat resource baru; memanggil `POST /orders` tiga kali berpotensi membuat 3 order belanjaan berbeda).

Untuk membuat endpoint `POST` menjadi idempotent, kita menggunakan mekanisme **Idempotency Key**.

## 3. Kenapa Harus Pakai Ini ?

Di sistem backend skala besar, kita wajib mengimplementasikan **Timeouts** dan **Retries**. Namun, kombinasi timeouts dan retries membawa bahaya mematikan:

Bayangkan skenario belanja online:

1. Client menekan tombol "Bayar Rp 500.000". Request masuk ke Payment Service.
2. Payment Service sukses memotong saldo bank pengguna di detik ke-1.9.
3. Tiba-tiba kabel jaringan putus sesaat sebelum respon HTTP 200 terkirim balik ke client.
4. Client mengalami **Read Timeout (3 detik)** dan menganggap transaksi gagal.
5. Fitur retry otomatis di client mengirim ulang request "Bayar Rp 500.000".

Tanpa Idempotency: Saldo pengguna **terpotong dua kali (Rp 1.000.000)** dan tercipta dua invoice duplikat! Ini adalah mimpi buruk kepatuhan hukum (_regulatory disaster_) bagi perusahaan fintech dan e-commerce.

Dengan Idempotency: Payment Service mendeteksi bahwa request kedua membawa _Idempotency Key_ yang sama, menolak eksekusi ulang, dan langsung mengembalikan bukti pembayaran pertama.

## 4. Cara Kerjanya (Diagram Alur - Idempotency Key)

```
Client                                                  Payment Backend & Database
  |                                                                 |
  | 1. POST /payments                                               |
  |    Header: Idempotency-Key: "uuid-1234-abcd"                    |
  |---------------------------------------------------------------->|
  |                                                                 |
  |                                                  [ Cek Key di Database/Redis ]
  |                                                  Key Belum Ada -> Kunci (*Lock*)
  |                                                  Eksekusi Pemotongan Saldo
  |                                                  Simpan Hasil Transaksi ke DB
  |                                                  Simpan Response Body ke Cache
  |                                                                 |
  |               X (Koneksi Putus di Jalan!)                       |
  | < - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - |
  |                                                                 |
  | [ Client Timeout -> Auto Retry! ]                               |
  |                                                                 |
  | 2. POST /payments (RETRY)                                       |
  |    Header: Idempotency-Key: "uuid-1234-abcd"                    |
  |---------------------------------------------------------------->|
  |                                                                 |
  |                                                  [ Cek Key di Database/Redis ]
  |                                                  Key SUDAH ADA!
  |                                                  Status: COMPLETED
  |                                                  JANGAN Eksekusi Saldo Lagi!
  |                                                  Ambil Cached Response
  |                                                                 |
  | 3. HTTP 200 OK (Data Transaksi Pertama)                         |
  | <---------------------------------------------------------------|
  v                                                                 v
[ Transaksi Aman: Saldo Hanya Terpotong Sekali ]
```

## 5. Cuplikan Logika Inti (Idempotency Pattern di Go)

Implementasi standar idempotensi di layer aplikasi menggunakan pola Atomic Lock dan Payload Hashing di Go:

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

var ctx = context.Background()

// ProcessPaymentWithIdempotency mencegah double-charge menggunakan Redis SetNX
func ProcessPaymentWithIdempotency(rdb *redis.Client, idempotencyKey string, payload []byte) (string, error) {
	// 1. Buat hash payload untuk mendeteksi perubahan parameter request
	hasher := sha256.New()
	hasher.Write(payload)
	payloadHash := hex.EncodeToString(hasher.Sum(nil))

	lockKey := fmt.Sprintf("idempotency:%s", idempotencyKey)

	// 2. SetNX atomik sebagai distributed lock dengan TTL 120 detik
	acquired, err := rdb.SetNX(ctx, lockKey, "IN_PROGRESS:"+payloadHash, 120*time.Second).Result()
	if err != nil {
		return "", err
	}

	if !acquired {
		val, _ := rdb.Get(ctx, lockKey).Result()
		if val == "IN_PROGRESS:"+payloadHash {
			return "Request sedang diproses bersamaan, silakan tunggu", nil
		}
		return "Transaksi duplikat telah berhasil diproses sebelumnya", nil
	}

	// 3. Eksekusi mutasi saldo di core payment gateway...
	paymentID := "PAY-987654"

	// 4. Update status idempotensi menjadi COMPLETED selama 24 jam
	rdb.Set(ctx, lockKey, "COMPLETED:"+paymentID, 24*time.Hour)
	return paymentID, nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Integritas Finansial 100%**: Mencegah insiden _Double Spending_, _Duplicate Orders_, dan _Over-crediting_.
-   **Aman Dikombinasikan dengan Retry**: Memungkinkan sistem jaringan melakukan percobaan ulang tanpa takut merusak data.

### Kekurangan

-   **Overhead Penyimpanan**: Sistem harus menyimpan riwayat Idempotency Key beserta respon JSON-nya di database atau Redis selama durasi tertentu (misal: 24 jam).
-   **Penanganan Concurrency (_In-Flight Request_)**: Jika request kedua datang saat request pertama masih dalam proses eksekusi, sistem harus menangani status _Conflict_ (HTTP 409) atau menahan request hingga proses pertama selesai.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Seluruh endpoint mutasi finansial: Pembayaran, Transfer Saldo, Refund, Penarikan Dana.
-   Pembuatan order dan pemotongan inventaris stok (_checkout_).
-   Webhook receiver (misal menerima notifikasi status pembayaran dari Stripe/Midtrans/Xendit yang sering dikirimkan berulang kali).

### Kapan Jangan Pakai

-   Endpoint pencarian atau pembacaan data biasa (`GET /users`, `GET /products`).
-   Data telemetry / logging yang sifatnya append-only dan tidak kritis jika ada selisih minor (misal event tracking klik tombol analitik).

## 8. Metrik Kunci & Observabilitas

Untuk memonitor kesehatan dan efektivitas idempotensi pada dashboard (Grafana / Datadog):

-   **`idempotency_requests_total`**: Total request yang masuk dengan header idempotency.
-   **`idempotency_cache_hit_total`**: Jumlah request duplikat yang berhasil ditahan dan disajikan langsung dari cache response.
-   **`idempotency_in_flight_conflicts_total`**: Jumlah request duplikat yang datang bersamaan saat request pertama masih berlangsung (HTTP 409).
-   **`idempotency_payload_mismatch_errors_total`**: Lonjakan metrik ini mengindikasikan adanya bug client atau upaya manipulasi parameter dengan key lama (HTTP 422).
-   **`idempotency_store_latency_ms`**: Latensi membaca dan menyimpan status idempotensi ke Redis / database.

## 9. Studi Kasus Industri Nyata

### Pencegahan Double-Charge dan Duplicate Orders pada Checkout E-Commerce

- **Tantangan Rekayasa Riil (The Problem)**:
  Pada sebuah platform belanja daring saat kampanye promo kilat, seorang pengguna menekan tombol "Bayar Sekarang" senilai Rp 2.500.000. Karena jaringan seluler pengguna sedang mengalami lonjakan latensi, indikator pemuatan (*spinner*) di layar smartphone berputar selama 3 detik tanpa kepastian. Pengguna yang panik menekan tombol bayar 3 kali lagi secara berturut-turut. Di saat yang sama, lapisan jaringan HTTP client mobile mendeteksi *read timeout* dan secara otomatis meluncurkan 2 kali *network retry*. Tanpa pengaman idempotensi di backend, ke-5 request tersebut diproses secara paralel oleh beberapa pod Go yang berbeda: saldo rekening pengguna terpotong 3 kali lipat (mengakibatkan kerugian finansial pengguna sebesar Rp 7.500.000) dan 3 nomor pesanan duplikat terbit untuk satu barang yang sama.

- **Implementasi Solusi Industri (The Architectural Solution)**:
  1. **Client-Generated Idempotency Key**: Aplikasi klien diwajibkan menyematkan header `Idempotency-Key: <UUIDv4>` yang di-generate tepat saat pengguna menekan tombol konfirmasi untuk pertama kalinya. Nilai key ini wajib dipertahankan sama pada seluruh proses retry jaringan.
  2. **Atomic In-Flight Locking via Redis**: Begitu request tiba di middleware Go:
     - Dijalankan perintah atomik Redis: `SET idempotency:{key} STARTED NX EX 120`.
     - Jika key sudah ada dengan status `STARTED`, request paralel berikutnya langsung ditahan atau dikembalikan dengan status `HTTP 409 Conflict: Transaction In-Progress`.
  3. **Payload Fingerprint Verification (SHA-256)**: Sistem menghitung hash SHA-256 dari payload JSON request dan menyimpannya di Redis. Jika ada request masuk dengan Idempotency Key yang sama namun payload nominalnya berbeda, backend langsung menolak dengan `HTTP 422 Unprocessable Entity (Payload Mismatch)` untuk mencegah manipulasi data.
  4. **Database Unique Constraint as Hard Anchor**: Pada tabel transaksi database PostgreSQL, kolom `idempotency_key` dipasangi `UNIQUE CONSTRAINT`. Sekalipun terjadi anomali pada Redis, PostgreSQL secara fisik membatalkan proses insert duplikat melalui mekanisme penegakan integritas ACID database.
  5. **Response Replay Caching**: Begitu transaksi pembayaran sukses diproses oleh core banking, status diperbarui menjadi `COMPLETED` dan payload respon JSON disimpan di Redis selama 24 jam. Seluruh request duplikat berikutnya langsung dibalas dengan salinan respon tersebut dalam waktu 0.5ms tanpa pernah mengeksekusi pemotongan saldo ulang.

- **Dampak & Hasil Sistem (The Engineering Outcome)**:
  - Insiden pemotongan saldo ganda (*double-debit*) dan pembuatan pesanan dobel tereliminasi hingga 0 kasus mutlak.
  - Beban eksekusi pemrosesan transaksi di payment processor berkurang 28% karena klik ganda pengguna disaring langsung di layer gerbang idempotensi.
  - Transaksi checkout menjadi sepenuhnya aman dikombinasikan dengan mekanisme *automatic network retries*.

## 10. Fakta Menarik & Gotchas Production

-   **The Database Unique Constraint Is Your Best Friend**: Banyak developer mencoba mengecek idempotensi hanya lewat Redis: `GET key`, jika tidak ada lalu proses. Di trafik tinggi, dua request identik bisa masuk di milidetik yang sama (_race condition_) dan keduanya lolos! Perlindungan lapisan terakhir yang paling kokoh adalah **Database Unique Constraint** (misal kolom `idempotency_key VARCHAR UNIQUE` di tabel PostgreSQL). Database engine menjamin secara atomik bahwa hanya satu record yang berhasil di-insert!
-   **Payload Mismatch Attack / Bug**: Apa yang terjadi jika client mengirimkan _Idempotency-Key_ yang sama (`uuid-123`), tetapi dengan payload yang berbeda (request 1 bayar Rp 50.000, request 2 bayar Rp 500.000)? API yang dirancang dengan baik akan menghitung hash SHA-256 dari payload body. Jika Key sama tetapi Payload berbeda, kembalikan error **`HTTP 422 Unprocessable Entity: Idempotency key payload mismatch`**.

## 11. Rangkuman Singkat

1. **Jaminan Integritas Finansial**: Memastikan request mutasi data yang dieksekusi berulang kali menghasilkan efek samping sistem yang persis sama dengan eksekusi satu kali.
2. **Kombinasi Dua Lapisan Pertahanan**: Manfaatkan Redis *atomic lock* untuk meredam duplikasi kilat dan pasang *Database Unique Constraint* sebagai benteng integritas ACID terakhir yang kebal *race condition*.
3. **Validasi Sidik Jari Payload**: Hitung dan cocokkan hash SHA-256 dari request body untuk mendeteksi anomali jika ada client yang menyalahgunakan key lama untuk transaksi berbeda.
4. **Caching Respon Asli**: Simpan respon transaksi awal di cache terdistribusi agar panggilan berulang berikutnya langsung dijawab seketika tanpa membebani processor pembayaran inti.
