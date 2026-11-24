# Idempotency

## 1. Skenario Sistem Nyata

Pada sistem pembayaran digital berskala produksi, pengguna menekan tombol "Bayar Sekarang" untuk pesanan senilai Rp 500.000 (`POST /api/v1/payments`).

Backend Payment Service menerima request tersebut, berhasil mendebet saldo rekening pengguna di bank pada detik ke-1.8, dan mencatat transaksi ke database. Namun, tepat sebelum respon `HTTP 200 OK` terkirim balik ke ponsel pengguna, sinyal 4G di ponsel pengguna terputus sesaat (*network drop*).

Ponsel pengguna mengalami *Read Timeout*, menganggap pembayaran gagal, dan sistem retry otomatis aplikasi mobile langsung mengirim ulang request `POST /api/v1/payments` yang sama persis beberapa detik kemudian:
- Tanpa Idempotency: Backend memperlakukan request kedua sebagai transaksi baru. Akibatnya, saldo pengguna **terpotong dua kali (Rp 1.000.000)** dan tercipta dua tagihan ganda (*double charge*). Ini adalah mimpi buruk kepatuhan regulasi finansial dan memicu sengketa pengembalian dana (*chargeback dispute*).

**Solusi dengan Mekanisme Idempotency Key:**
Aplikasi client menyertakan header unik pada setiap transaksi mutasi: `Idempotency-Key: "uuid-pay-8821"`. Saat request retry kedua masuk membawa key yang sama, backend memeriksa database/Redis, mendeteksi bahwa transaksi dengan key tersebut sudah berstatus `COMPLETED`, langsung menolak pendebetan saldo kedua, dan dengan aman mengembalikan salinan struk bukti bayar transaksi pertama. Saldo pengguna dijamin terpotong tepat satu kali (*strictly once*).

## 2. Apa Itu dan Buat Apa?

Idempotency (Idempotensi) adalah sifat sebuah operasi API di mana mengeksekusi operasi tersebut berkali-kali dengan parameter yang sama **akan menghasilkan kondisi akhir sistem (*state*) dan respon yang sama persis** seperti saat operasi pertama kali dijalankan.

Dalam spesifikasi standar protokol HTTP:
- **`GET`, `HEAD`, `OPTIONS`**: Bersifat *Safe* dan *Idempotent* (hanya membaca data).
- **`PUT`**: Bersifat *Idempotent* (mengganti seluruh resource; mengupdate alamat user menjadi "Jl. Mawar No. 10" sepuluh kali menghasilkan alamat yang sama persis).
- **`DELETE`**: Bersifat *Idempotent* (menghapus user ID 42 sepuluh kali; data tetap terhapus dari sistem).
- **`POST`**: Secara default **TIDAK Idempotent** (membuat resource baru; memanggil `POST /orders` tiga kali berpotensi membuat 3 order belanjaan berbeda).

Untuk membuat endpoint `POST` menjadi idempotent, kita menggunakan mekanisme **Idempotency Key**.

## 3. Kenapa Harus Pakai Ini di High-Throughput?

Di sistem backend skala besar, kita wajib mengimplementasikan **Timeouts** dan **Retries**. Namun, kombinasi timeouts dan retries membawa bahaya mematikan:

Bayangkan skenario belanja online:
1. Client menekan tombol "Bayar Rp 500.000". Request masuk ke Payment Service.
2. Payment Service sukses memotong saldo bank pengguna di detik ke-1.9.
3. Tiba-tiba kabel jaringan putus sesaat sebelum respon HTTP 200 terkirim balik ke client.
4. Client mengalami **Read Timeout (3 detik)** dan menganggap transaksi gagal.
5. Fitur retry otomatis di client mengirim ulang request "Bayar Rp 500.000".

Tanpa Idempotency: Saldo pengguna **terpotong dua kali (Rp 1.000.000)** dan tercipta dua invoice duplikat! Ini adalah mimpi buruk kepatuhan hukum (*regulatory disaster*) bagi perusahaan fintech dan e-commerce.

Dengan Idempotency: Payment Service mendeteksi bahwa request kedua membawa *Idempotency Key* yang sama, menolak eksekusi ulang, dan langsung mengembalikan bukti pembayaran pertama.

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
- **Integritas Finansial 100%**: Mencegah insiden *Double Spending*, *Duplicate Orders*, dan *Over-crediting*.
- **Aman Dikombinasikan dengan Retry**: Memungkinkan sistem jaringan melakukan percobaan ulang tanpa takut merusak data.

### Kekurangan
- **Overhead Penyimpanan**: Sistem harus menyimpan riwayat Idempotency Key beserta respon JSON-nya di database atau Redis selama durasi tertentu (misal: 24 jam).
- **Penanganan Concurrency (*In-Flight Request*)**: Jika request kedua datang saat request pertama masih dalam proses eksekusi, sistem harus menangani status *Conflict* (HTTP 409) atau menahan request hingga proses pertama selesai.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Seluruh endpoint mutasi finansial: Pembayaran, Transfer Saldo, Refund, Penarikan Dana.
- Pembuatan order dan pemotongan inventaris stok (*checkout*).
- Webhook receiver (misal menerima notifikasi status pembayaran dari Stripe/Midtrans/Xendit yang sering dikirimkan berulang kali).

### Kapan Jangan Pakai
- Endpoint pencarian atau pembacaan data biasa (`GET /users`, `GET /products`).
- Data telemetry / logging yang sifatnya append-only dan tidak kritis jika ada selisih minor (misal event tracking klik tombol analitik).

## 8. Metrik Kunci & Observabilitas

Untuk memonitor kesehatan dan efektivitas idempotensi pada dashboard (Grafana / Datadog):

- **`idempotency_requests_total`**: Total request yang masuk dengan header idempotency.
- **`idempotency_cache_hit_total`**: Jumlah request duplikat yang berhasil ditahan dan disajikan langsung dari cache response.
- **`idempotency_in_flight_conflicts_total`**: Jumlah request duplikat yang datang bersamaan saat request pertama masih berlangsung (HTTP 409).
- **`idempotency_payload_mismatch_errors_total`**: Lonjakan metrik ini mengindikasikan adanya bug client atau upaya manipulasi parameter dengan key lama (HTTP 422).
- **`idempotency_store_latency_ms`**: Latensi membaca dan menyimpan status idempotensi ke Redis / database.

## 9. Studi Kasus Nyata di Industri

- **Stripe API Idempotency Specification**: Stripe memelopori standar industri untuk pembayaran online menggunakan header `Idempotency-Key: <UUID>`. Jika koneksi client terputus saat charge kartu kredit, SDK Stripe otomatis mengulang request dengan key yang sama. Server Stripe menjamin saldo pelanggan tidak pernah terdebet dua kali dan data payload dicocokkan dengan hash tersimpan.
- **Shopify Order Ingestion**: Selama flash sale Black Friday, ribuan pembeli menekan tombol submit pesanan berulang kali saat halaman checkout terasa lambat. Arsitektur order Shopify memanfaatkan token checkout unik sebagai idempotency key di layer backend database untuk menyerap jutaan checkout tanpa membuat pesanan duplikat.

## 10. Fakta Menarik & Gotchas Production

- **The Database Unique Constraint Is Your Best Friend**: Banyak developer mencoba mengecek idempotensi hanya lewat Redis: `GET key`, jika tidak ada lalu proses. Di trafik tinggi, dua request identik bisa masuk di milidetik yang sama (*race condition*) dan keduanya lolos! Perlindungan lapisan terakhir yang paling kokoh adalah **Database Unique Constraint** (misal kolom `idempotency_key VARCHAR UNIQUE` di tabel PostgreSQL). Database engine menjamin secara atomik bahwa hanya satu record yang berhasil di-insert!
- **Payload Mismatch Attack / Bug**: Apa yang terjadi jika client mengirimkan *Idempotency-Key* yang sama (`uuid-123`), tetapi dengan payload yang berbeda (request 1 bayar Rp 50.000, request 2 bayar Rp 500.000)? API yang dirancang dengan baik akan menghitung hash SHA-256 dari payload body. Jika Key sama tetapi Payload berbeda, kembalikan error **`HTTP 422 Unprocessable Entity: Idempotency key payload mismatch`**.
