# Redis (Remote Dictionary Server)

## 1. Skenario Sistem Nyata

Pada fitur **Flash Sale Kilat 12.12** platform e-commerce besar, sebanyak 500 unit produk diskon diperebutkan oleh **150.000 pengguna secara serentak** dalam hitungan detik pertama. Ada 500 unit iPhone diskon 90% yang diperebutkan oleh **150.000 pengguna secara serentak** dalam 3 detik pertama setelah promo dibuka.

Jika kamu mengandalkan database relasional tradisional (seperti PostgreSQL atau MySQL) untuk menangani pengecekan sisa stok dan pemesanan:

-   Setiap klik checkout pengguna mengeksekusi `SELECT stock FROM products WHERE id = 1` lalu `UPDATE products SET stock = stock - 1`.
-   Ratusan ribu koneksi konkuren membombardir database, memicu _disk I/O saturation_, _row-level locking contention_, dan akhirnya _connection pool exhaustion_. Database crash seketika, server down, dan terjadi bencana _overselling_ (stok minus menjadi -50).

**Solusi dengan Redis:**
Jumlah sisa stok disimpan di memori RAM Redis sebagai integer key: `stock:iphone15 = 500`. Ketika serbuan request masuk, backend langsung mengeksekusi perintah atomik Redis **`DECR stock:iphone15`**. Karena seluruh data berada di RAM dan dieksekusi secara atomik _single-threaded_ tanpa lock disk, Redis sanggup memproses **100.000+ operasi per detik** dengan latensi sub-milidetik (0.2 ms). Pengguna ke-501 langsung mendapatkan angka 0 atau negatif dan ditolak detik itu juga dengan aman, tanpa menyentuh disk database utama sama sekali!

## 2. Apa Itu dan Buat Apa?

**Redis (Remote Dictionary Server)** adalah _in-memory data store_ sumber terbuka berkecepatan tinggi yang dapat difungsikan sebagai database, cache, message broker, dan streaming engine.

### Karakteristik Kunci:

1. **Penyimpanan Murni di Memori (RAM)**: Semua data primer berada di RAM komputer, menghasilkan waktu baca dan tulis rata-rata di bawah 1 milidetik.
2. **Koleksi Struktur Data Kaya**: Tidak hanya menyimpan string key-value sederhana, Redis mendukung:
    - **Strings**: Teks, angka, biner (hingga 512 MB).
    - **Hashes**: Representasi objek/field (seperti profil user atau cart).
    - **Lists**: Antrean ordered elemen (cocok untuk queue FIFO/LIFO).
    - **Sets & Sorted Sets (ZSet)**: Himpunan unik bernilai skor (sempurna untuk leaderboard, ranking, dan sliding window rate limiter).
    - **Bitmaps & HyperLogLog**: Penghitungan unik memori ultra-hemat (jutaan DAU dihitung dalam hitungan Kilobyte).
3. **Persistensi Data ke Disk**: Walaupun berbasis RAM, Redis memiliki mekanisme persistensi:
    - **RDB (Snapshotting)**: Menyalin seluruh snapshot memori ke disk secara berkala.
    - **AOF (Append Only File)**: Mencatat setiap log operasi mutasi data ke disk untuk pemulihan instan saat server crash.

## 3. Kenapa Harus Pakai Ini ?

### 1. Kecepatan Eksekusi Berbasis RAM (_Sub-Millisecond Latency_)

Membaca data dari RAM membutuhkan waktu sekitar 100 nanodetik, sedangkan membaca dari SSD NVMe membutuhkan 10-50 mikrodetik, dan HDD butuh milidetik. Kecepatan murni RAM membuat Redis menjadi peredam kejut (_shock absorber_) terbaik di hadapan trafik meledak.

### 2. Arsitektur Single-Threaded Event Loop (Non-Blocking I/O)

Redis inti menggunakan arsitektur single-threaded yang ditenagai oleh I/O multiplexing (_epoll_ di Linux). Karena single-threaded, **tidak pernah ada race condition, deadlock, atau overhead thread context-switching**. Semua operasi manipulasi data (seperti `INCR`, `HSET`, `LPUSH`) bersifat atomik secara bawaan.

### 3. Distribusi dan Ketersediaan Tinggi (_Replication & Sentinel/Cluster_)

Redis mendukung replikasi master-replica asinkron, failover otomatis menggunakan Redis Sentinel, dan sharding horizontal hingga 1.000 node menggunakan Redis Cluster untuk kapasitas memori Terabyte.

## 4. Cara Kerjanya (Arsitektur Single-Threaded Event Loop Redis)

```
[ Ribuan Client Concurrency ]
Client 1     Client 2     Client 3
    |            |            |
    v            v            v
+------------------------------------------+
|  I/O Multiplexer (Linux epoll / kqueue)  |  <-- Menampung ribuan soket TCP aktif
+------------------------------------------+
                     |
                     v (Antrean Event Berurutan)
+------------------------------------------+
|       Single-Threaded Event Loop         |
|  - Eksekusi Perintah 1: INCR key (0.1ms) |  <-- Dieksekusi SATU PER SATU secara atomik
|  - Eksekusi Perintah 2: GET user (0.1ms) |      tanpa butuh mutex / thread lock!
|  - Eksekusi Perintah 3: ZADD rank (0.2ms)|
+------------------------------------------+
                     |
                     v
+------------------------------------------+
|         In-Memory RAM Key-Value          |
+------------------------------------------+
                     |
       (Asinkron di Background Thread)
                     v
+------------------------------------------+
|    Disk Persistence (RDB / AOF Log)      |
+------------------------------------------+
```

## 5. Cuplikan Logika Inti

### 1. Atomic Decrement untuk Flash Sale di Go

```go
func BuyFlashSaleItem(ctx context.Context, rdb *redis.Client, userID string, itemID string) (bool, error) {
	stockKey := fmt.Sprintf("stock:%s", itemID)

	// DECR dijalankan secara atomik murni di single thread Redis
	remainingStock, err := rdb.Decr(ctx, stockKey).Result()
	if err != nil {
		return false, err
	}

	if remainingStock < 0 {
		// Revert counter dan tolak pesanan
		rdb.Incr(ctx, stockKey)
		return false, nil // Sold Out
	}

	// Masukkan ke antrean pesanan
	rdb.LPush(ctx, fmt.Sprintf("orders:%s", itemID), userID)
	return true, nil
}
```

### 2. Distributed Lock Sederhana (Redlock Pattern) di Go

```go
func acquireLock(client *redis.Client, lockKey string, token string, ttl time.Duration) bool {
    // Set jika belum ada (NX) dengan masa kedaluwarsa otomatis (EX)
    success, err := client.SetNX(context.Background(), lockKey, token, ttl).Result()
    return err == nil && success
}

func releaseLock(client *redis.Client, lockKey string, token string) {
    // Lua script memastikan unlock hanya menghapus lock jika token pemilik cocok (atomik)
    luaScript := `
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
    `
    client.Eval(context.Background(), luaScript, []string{lockKey}, token)
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Throughput Ekstrem**: Mampu melayani lebih dari 100.000 Request Per Second (RPS) per instance core tunggal.
-   **Operasi Atomik**: Menghilangkan kebutuhan locking rumit di layer aplikasi untuk skenario increment, decrement, dan queueing.
-   **Kaya Fitur Bawaan**: Mendukung TTL (_Time-To-Live_), Pub/Sub, Geospatial indexing, HyperLogLog, dan transaksi pipeline.

### Kekurangan

-   **Biaya Memori RAM Mahal**: Menyimpan data berukuran puluhan gigabyte di RAM jauh lebih mahal daripada menyimpannya di SSD NVMe disk.
-   **Data Loss Risk (Durabilitas Terbatas)**: Karena default persistensi AOF/RDB bersifat _fsync asinkron_ per detik, server yang mendadak mati dapat kehilangan data 1 detik terakhir (_trade-off performa vs durabilitas_).
-   **Single-Threaded Bottleneck**: Perintah yang memakan kompleksitas waktu $O(N)$ (seperti `KEYS *` atau sorting dataset besar) akan memblokir (_freeze_) seluruh server Redis dan membekukan semua request client lainnya!

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Pakai Redis

-   Caching data yang sering dibaca dengan latensi sub-milidetik (sesi login user, metadata produk).
-   Counter dan Rate Limiting atomik (Token Bucket / Sliding Window).
-   Real-time Leaderboard menggunakan Sorted Sets (`ZADD`, `ZRANGEBYSCORE`).
-   Distributed Lock untuk mengoordinasikan eksekusi tugas agar tidak dijalankan ganda oleh pod berbeda.

### Kapan Jangan Pakai Redis

-   Penyimpanan primer dokumen besar (> 1 MB per record) atau data relasional kompleks yang membutuhkan ACID transaksi kaku.
-   Data warehouse / cold storage yang jarang diakses (pemborosan kapasitas RAM).
-   Sebagai pengganti database relasional utama tanpa strategi persistensi dan backup yang solid.

## 8. Metrik Kunci & Observabilitas

Metrik wajib di Grafana / Datadog saat memantau kluster Redis produksi:

-   **`instantaneous_ops_per_sec`**: Jumlah operasi baca/tulis yang sedang diproses per detik.
-   **`used_memory` vs `maxmemory`**: Konsumsi RAM saat ini; jika menyentuh `maxmemory`, Redis akan mulai melakukan _eviction_ atau menolak request write (`OOM error`).
-   **`keyspace_hits` & `keyspace_misses`**: Digunakan untuk mengukur efektivitas cache hit ratio (`hits / (hits + misses)`).
-   **`connected_clients`**: Jumlah koneksi soket client aktif (waspadai connection leak dari aplikasi).
-   **`latest_fork_usec`**: Durasi waktu yang dibutuhkan Redis untuk proses `fork()` saat membuat background snapshot RDB; fork yang lambat dapat membekukan latensi request sesaat.

## 9. Studi Kasus Nyata di Industri

### Caching Multi-Layer dan Rate Limiting Atomik pada Sistem Autentikasi Berskala Tinggi

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah portal fintech mengalami ledakan trafik login dan verifikasi token OAuth sebanyak 60.000 request per detik. Awalnya, setiap validasi bearer token dan sesi user dieksekusi dengan query `SELECT` langsung ke database relasional PostgreSQL. Database mengalami kehabisan connection pool (5.000 koneksi jenuh), disk I/O menyentuh 100%, dan latensi p99 endpoint otorisasi melonjak dari 15 ms menjadi 3.800 ms. Selain itu, upaya brute-force login tidak dapat dibendung karena pencatatan percobaan gagal di database disk terlalu lambat.
- **Implementasi Solusi Industri (The Architectural Solution)**: Tim mengadopsi Redis Cluster terdistribusi sebagai session store in-memory dan layer rate limiting. Sesi login dan payload klaim token di-cache dalam key-value dengan TTL presisi 24 jam. Verifikasi kuota login menerapkan algoritma *Sliding Window Rate Limiter* menggunakan struktur data Redis Sorted Set (`ZREMRANGEBYSCORE` dan `ZADD`) yang dieksekusi secara atomik via script Lua. Kebijakan pembersihan memori disetel ke `volatile-lru` dengan alokasi `maxmemory` terkontrol guna mencegah memori OOM.
- **Dampak & Hasil Sistem (The Outcome)**: Latensi verifikasi otorisasi p99 terpangkas dari 3.800 ms menjadi 1,2 ms (performa sub-milidetik). Sebanyak 98,5% beban query otentikasi terangkat dari database PostgreSQL, menurunkan utilisasi CPU database dari 95% menjadi 18%. Serangan brute-force berhasil dicegat seketika di layer gateway melalui atomic sliding window limiter tanpa menambah beban database relasional.

## 10. Fakta Menarik & Gotchas Production

-   **The `KEYS *` Production Nightmare**:  
    Di Redis, menjalankan perintah `KEYS *` untuk mencari key yang cocok dengan pola tertentu adalah operasi $O(N)$ yang memeriksa seluruh miliaran key di memori. Karena Redis single-threaded, perintah ini akan **mengunci (_freeze_) server Redis selama berpuluh-puluh detik**! Semua request aplikasi backend lainnya akan timeout seketika dan memicu cascading failure global!  
    _Aturan Wajib:_ Haramkan `KEYS *` di production. Selalu gunakan **`SCAN`** yang berjalan secara inkremental tanpa memblokir event loop.
-   **Redis Maxmemory Eviction Policy Trap**:  
    Secara default, jika kapasitas memori `maxmemory` habis, Redis akan menolak semua operasi write dengan error `OOM command not allowed`.  
    _Solusi Wajib:_ Konfigurasikan kebijakan pembersihan memori (_eviction policy_) yang tepat di `redis.conf`, seperti **`allkeys-lru`** (hapus key yang paling jarang digunakan saat RAM penuh) atau **`volatile-lru`** (hanya hapus key yang memiliki batas waktu TTL).

## 11. Rangkuman Singkat

- **In-Memory Berkecepatan Sub-Milidetik**: Redis menyimpan seluruh dataset di RAM dengan arsitektur single-threaded event loop, menghasilkan operasi baca/tulis ratusan ribu TPS dengan latensi di bawah 1 milidetik.
- **Pilih Struktur Data yang Tepat**: Manfaatkan kapabilitas bawaan Redis selain string biasa: Hashes untuk objek, Sets untuk keunikan, Sorted Sets untuk leaderboard & rate limiting, dan HyperLogLog untuk penghitungan kardinalitas unik.
- **Haramkan Perintah O(N) di Production**: Jangan pernah menjalankan `KEYS *` atau `FLUSHALL` di cluster produksi karena akan membekukan server single-thread; selalu gunakan perintah kursor non-blocking seperti `SCAN`.
- **Konfigurasikan Maxmemory dan Eviction Policy**: Tetapkan batas `maxmemory` dan pasang eviction policy (seperti `allkeys-lru` atau `volatile-lru`) agar Redis tidak melempar error OOM saat kapasitas RAM server menyentuh batas.
