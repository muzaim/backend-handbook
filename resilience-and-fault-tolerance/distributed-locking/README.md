# Distributed Locking (Concurrency & Race Condition Guard)

## 1. Skenario Sistem Nyata

Pada sebuah platform penjualan tiket konser musik internasional, terdapat 1 kursi eksklusif terakhir dengan nomor **VIP-A1**. Tepat saat penjualan dibuka pada pukul 12:00:00.000, sebanyak 500 pengguna menekan tombol **"Beli & Bayar Sekarang"** pada milidetik yang persis sama.

Backend aplikasi dideploy di Kubernetes dengan 20 pod instance Go yang berjalan paralel di balik sebuah Load Balancer:

Jika backend hanya mengandalkan penguncian lokal di memori (*In-Memory Mutex* via `sync.Mutex` di Go):
- Mutex lokal hanya sanggup mengunci *goroutine* di dalam **satu proses mesin yang sama**.
- Setiap pod dari 20 instance tersebut memiliki 1 pemenang lokalnya masing-masing. Akibatnya, ada 20 transaksi pembayaran yang sukses untuk kursi VIP-A1 yang sama!
- Terjadi bencana bisnis fatal: **Overselling / Double-Booking**. 19 pembeli yang sudah membayar uang jutaan rupiah tidak mendapatkan kursi di hari konser.

Jika dipaksakan mengunci baris database relasional (`SELECT ... FOR UPDATE`):
- 500 transaksi bersamaan berebut mengunci 1 baris yang sama di PostgreSQL, memicu *lock contention* parah, antrean koneksi habis, dan deadlock yang menenggelamkan database utama.

**Solusi dengan Distributed Locking (Redis / etcd):**
Sebelum memproses transaksi, pod aplikasi Go wajib memperebutkan satu kunci global terpusat di Redis menggunakan instruksi atomik:
```
SET lock:seat:VIP-A1 "uuid-request-9921" NX PX 5000
```
- Flag `NX` (Not Exists) dan `PX 5000` (Kedaluwarsa otomatis 5 detik) memastikan bahwa **hanya 1 dari 500 request yang berhasil memperoleh kunci** dalam waktu sub-milidetik.
- Sebanyak 499 request lainnya langsung ditolak seketika (*fail-fast*) dengan pesan: *"Kursi sedang diproses oleh pembeli lain"*.
- Pod pemenang menyelesaikan proses pembayaran di database, lalu melepas kunci secara aman menggunakan script Lua.

## 2. Apa Itu dan Buat Apa?

Distributed Locking adalah mekanisme sinkronisasi konkurensi yang menjamin sifat *Mutual Exclusion* (hanya satu proses yang boleh mengeksekusi *critical section* pada satu waktu) di lingkungan sistem terdistribusi yang berjalan di banyak server fisik, VM, atau container berbeda.

Karakteristik utama:
1. **Mutual Exclusion (Eksklusivitas Mutlak)**: Pada satu waktu, hanya satu client di seluruh cluster yang dapat memegang lock untuk sebuah resource ID.
2. **Deadlock Free (Bebas Kebuntuan)**: Setiap lock wajib memiliki masa kedaluwarsa otomatis (*TTL / Lease Time*). Jika server pemegang lock tiba-tiba mati atau crash akibat pemadaman listrik, kunci akan terbuka sendiri secara otomatis setelah TTL habis.
3. **Fault Tolerance**: Penyimpan lock (seperti Redis Cluster atau etcd) harus tetap hidup dan konsisten meskipun ada node yang mengalami kegagalan jaringan.

## 3. Kenapa Harus Pakai Ini di High-Throughput?

Dalam skala sistem tinggi dengan penskalaan horizontal (banyak instance pod):
1. **Mencegah Bencana Double-Spend & Overselling**: Menjamin stok barang, saldo uang, atau nomor kursi tidak pernah diklaim lebih dari satu pihak.
2. **Melindungi Database dari Concurrency Lock Storms**: Menyaring ribuan request perebutan resource di layer in-memory (Redis) berkecepatan mikrodetik, sehingga database hanya memproses 1 request pemenang.
3. **Mencegah Eksekusi Job Ganda (*Scheduled Task Overlap*)**: Mencegah cron job yang berjalan di 5 server mengeksekusi pengiriman email tagihan yang sama sebanyak 5 kali ke pengguna.

## 4. Cara Kerjanya (Diagram Alur Distributed Lock di Redis)

```
 [500 Concurrent Pod Requests]
 (Semua berebut kursi VIP-A1)
   |       |             |
   v       v             v
 +---------------------------------------------------------------+
 |                       Redis Cluster                           |
 |                                                               |
 | Command: SET lock:seat:VIP-A1 <unique_uuid> NX PX 5000        |
 |                                                               |
 | * 1 Pemenang  : Return OK   --> [Lock BERHASIL Diperoleh!]    |
 | * 499 Request : Return NIL  --> [Lock GAGAL / Tolak Seketika] |
 +---------------------------------------------------------------+
        |                                     |
        v                                     v
 [Pod Pemenang]                      [499 Pod Lainnya]
 1. Eksekusi Pembayaran di DB        Langsung balas HTTP 409 Conflict:
 2. Kurangi Stok Tiket               "Kursi sedang dipesan orang lain"
 3. Hapus Lock via Script Lua:
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    end
```

## 5. Cuplikan Logika Inti (Safe Distributed Lock di Go)

Berikut implementasi distributed lock atomik di Go menggunakan library `github.com/redis/go-redis/v9`, lengkap dengan identifier UUID acak dan pelepasan kunci yang aman menggunakan script Lua:

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"

	"github.com/google/uuid"
	"github.com/redis/go-redis/v9"
)

// Script Lua untuk melepaskan kunci secara atomik HANYA JIKA UUID-nya cocok
// Mencegah client A menghapus kunci milik client B jika proses client A memakan waktu melebihi TTL
const unlockLuaScript = `
	if redis.call("get", KEYS[1]) == ARGV[1] then
		return redis.call("del", KEYS[1])
	else
		return 0
	end
`

type DistributedLocker struct {
	rdb *redis.Client
}

func NewLocker(rdb *redis.Client) *DistributedLocker {
	return &DistributedLocker{rdb: rdb}
}

// AcquireLock mencoba mengambil lock dengan durasi TTL tertentu
func (l *DistributedLocker) AcquireLock(ctx context.Context, resourceKey string, ttl time.Duration) (string, error) {
	lockKey := fmt.Sprintf("lock:%s", resourceKey)
	lockValue := uuid.New().String() // Token unik pemilik lock

	// SET lockKey lockValue NX PX ttl (Atomic Instruction)
	success, err := l.rdb.SetNX(ctx, lockKey, lockValue, ttl).Result()
	if err != nil {
		return "", fmt.Errorf("gagal menghubungi redis: %w", err)
	}

	if !success {
		return "", errors.New("gagal memperoleh lock: resource sedang dikunci proses lain")
	}

	return lockValue, nil
}

// ReleaseLock melepaskan lock menggunakan script Lua
func (l *DistributedLocker) ReleaseLock(ctx context.Context, resourceKey string, lockValue string) error {
	lockKey := fmt.Sprintf("lock:%s", resourceKey)

	// Jalankan script Lua secara atomik di Redis
	result, err := l.rdb.Eval(ctx, unlockLuaScript, []string{lockKey}, lockValue).Result()
	if err != nil {
		return fmt.Errorf("gagal melepaskan lock: %w", err)
	}

	if result.(int64) == 0 {
		return errors.New("lock sudah kedaluwarsa atau bukan milik Anda")
	}

	return nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Menjamin Integritas Mutlak**: Mencegah duplikasi transaksi dan inkonsistensi data lintas ratusan node server.
- **Performa Sangat Cepat**: Operasi `SET NX` di Redis diselesaikan dalam waktu mikrodetik (< 1 ms).
- **Perlindungan Terhadap Kegagalan (*Self-Healing*)**: Fitur TTL memastikan tidak ada kunci yang menggantung selamanya jika worker mati mendadak.

### Kekurangan
- **Ketergantungan Ekstra pada Datastore**: Kegagalan cluster Redis dapat menghambat seluruh alur transaksi bisnis.
- **Tantangan Durasi TTL (*Clock Drift & GC Pause*)**: Jika eksekusi kode Go mengalami jeda *Garbage Collection* yang panjang melebihi durasi TTL, lock bisa kedaluwarsa sebelum pekerjaannya selesai.
- **Kompleksitas Konsensus Multi-Master (Algoritma Redlock)**: Membutuhkan kuorum quorum master Redis jika menginginkan perlindungan partisi jaringan tingkat tinggi.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Alur transaksi kritis bervolume tinggi yang rentan terhadap perebutan resource terbatas (pembelian tiket konser, flash sale barang langka, klaim kupon promo unik).
- Operasi dompet digital dan transaksi finansial yang tidak mentolerir *double-spending*.
- Penjadwalan pekerjaan berkala (*Distributed Cron Job*) yang hanya boleh dieksekusi oleh 1 worker terpilih.

### Kapan Jangan Pakai
- Operasi baca data yang bersifat idempotem (`GET` request) yang tidak memodifikasi status data.
- Aplikasi monolitik sederhana yang hanya berjalan di 1 instance server tunggal (cukup gunakan `sync.Mutex` lokal Go).
- Beban kerja di mana toleransi data bentrok dapat diselesaikan secara alami melalui constraint unik database (`UNIQUE CONSTRAINT`).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

- **`distributed_lock_acquisition_failures_total` (Counter)**: Jumlah penolakan akibat perebutan kunci. Lonjakan metrik ini normal saat flash sale berlangsung.
- **`distributed_lock_hold_duration_seconds` (Histogram)**: Mengukur berapa lama worker menahan lock sebelum melepaskannya. Jika mendekati batas TTL, durasi TTL harus dievaluasi.
- **`distributed_lock_expired_before_release_total` (Counter)**: Mendeteksi kejadian berbahaya di mana kunci kedaluwarsa sebelum worker selesai mengeksekusi tugasnya.

## 9. Studi Kasus Industri Nyata

- **Ticketmaster**: Menggunakan distributed lock berkecepatan tinggi untuk menahan hak reservasi kursi selama 5 menit saat pengguna memilih nomor kursi di denah stadion, memastikan jutaan penonton tidak pernah berebut kursi yang sama.
- **Tokopedia / Shopee**: Pada event Flash Sale tengah malam, jutaan pengguna berebut 10 unit smartphone diskon 90%. Distributed locking di layer Redis menyaring 99.9% request dalam hitungan milidetik sehingga hanya 10 pembeli tercepat yang dialirkan ke antrean pembayaran database.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Menghapus Lock Milik Client Lain Tanpa Script Lua**: Client A mengambil lock dengan TTL 3 detik. Karena database lambat, proses Client A memakan waktu 5 detik. Di detik ke-3, lock kedaluwarsa dan diambil oleh Client B. Di detik ke-5, Client A selesai dan memanggil perintah `DEL lock:key` biasa—sehingga menghapus lock milik Client B yang sah!  
   *Solusi*: Selalu sertakan token UUID unik dan gunakan script Lua untuk memverifikasi kepemilikan sebelum menghapus.
2. **Tidak Mengatur TTL (Kunci Abadi / Deadlock)**: Menyetel lock tanpa parameter `PX/EX`. Jika pod Go tiba-tiba mengalami crash OOM di baris berikutnya sebelum sempat melepas lock, resource tersebut akan terkunci selamanya dan tidak pernah bisa diakses lagi oleh siapa pun!
3. **Mengabaikan Fencing Token**: Untuk sistem yang sangat sensitif, gunakan nomor urut yang selalu naik (*Fencing Token*) agar database menolak penulisan dari worker yang prosesnya terlambat (*stale write*).

## 11. Rangkuman Singkat

Distributed Locking adalah penjaga integritas konkurensi data di arsitektur multi-server berskala besar. Dengan mengombinasikan instruksi atomik Redis (`SET NX PX`), token kepemilikan unik (UUID), dan pelepasan kunci berbasis script Lua, distributed locking menjamin bahwa transaksi kritis terbebas dari malapetaka race condition dan double-booking.
