# Caching

## 1. Skenario Sistem Nyata

Pada halaman beranda aplikasi e-commerce bervolume tinggi, widget **"Top 50 Produk Terlaris Hari Ini"** diakses secara intensif oleh 25.000 pengguna aktif per detik saat jam sibuk.

Untuk menghasilkan daftar 50 produk tersebut, backend harus menjalankan query SQL analitik yang melibatkan multi-table JOIN dan agregasi:
`SELECT p.id, p.title, p.price, SUM(oi.qty) FROM products p JOIN order_items oi ON p.id = oi.product_id GROUP BY p.id ORDER BY SUM(oi.qty) DESC LIMIT 50;`

Jika setiap request pengguna dieksekusi langsung ke database relasional (PostgreSQL / MySQL):

-   Database terpaksa melakukan pembacaan disk storage (_Disk I/O_) dan kalkulasi agregasi data 25.000 kali per detik untuk data yang hasilnya sebenarnya sama persis!
-   Dalam hitungan detik, kapasitas CPU database menyentuh 100%, connection pool habis, latensi query meledak dari 15 milidetik menjadi 8 detik, dan seluruh sistem backend tumbang.

**Solusi dengan Caching (misal: Redis / Memcached In-Memory):**
Saat request pertama datang, backend menghitung query database tersebut satu kali, lalu menyimpan hasilnya dalam bentuk JSON di memori RAM Redis dengan key `home:top_50_products` dan masa kedaluwarsa (_TTL_) 5 menit. Seluruh 25.000 request per detik berikutnya dilayani langsung dari RAM Redis dalam waktu sub-milidetik (< 0.5 ms) tanpa menyentuh database sama sekali. Database hanya menerima 1 query pembaruan setiap 5 menit.

## 2. Apa Itu dan Buat Apa?

Caching adalah teknik menyimpan salinan data yang sering diakses di media penyimpanan sementara yang sangat cepat (biasanya RAM / In-Memory), sehingga request berikutnya tidak perlu lagi membaca data dari sumber aslinya yang lambat (Disk I/O atau Database relasional).

Keuntungan utama:

-   **Latensi super rendah**: Mengakses data di RAM (Redis/Memcached) hanya butuh waktu sub-milidetik (0.1 - 1 ms), dibandingkan query SQL ke disk SSD/HDD (5 - 50 ms).
-   **Mengurangi beban Database**: Memangkas 80-95% query berulang ke database utama.
-   **Efisiensi biaya komputasi**: Hasil kalkulasi data yang berat tidak perlu dihitung ulang setiap detik.

### Pola Caching Populer:

-   **Cache-Aside (Lazy Loading)**: Aplikasi mencari data di Cache dulu. Jika tidak ada (_Cache Miss_), aplikasi mengambil dari Database, lalu menyimpannya ke Cache untuk request berikutnya. Ini adalah pola yang paling umum.
-   **Write-Through**: Setiap ada data baru atau update, aplikasi menulis ke Cache dan Database secara bersamaan.
-   **Write-Behind (Write-Back)**: Aplikasi hanya menulis ke Cache seketika, lalu proses background akan menyimpan data tersebut ke Database secara asynchronous beberapa detik kemudian (sangat cepat untuk write-heavy, tapi ada risiko data hilang jika server mati).

Contoh software: Redis, Memcached, Dragonfly, Hazelcast, local memory (Go sync.Map, Guava, Caffeine).

## 3. Kenapa Harus Pakai Ini ?

Dalam sistem backend, Database relasional (PostgreSQL, MySQL) hampir selalu menjadi **bottleneck pertama** saat trafik meledak (misal saat promo tanggal kembar atau Flash Sale).

Sebuah database Postgres tunggal mungkin hanya sanggup melayani 2.000 hingga 5.000 query kompleks per detik sebelum CPU 100% dan connection pool habis. Sebaliknya, satu instance Redis sederhana sanggup melayani 100.000 request per detik dengan mudah karena berjalan murni di RAM.

Tanpa caching, lonjakan trafik sekecil apa pun pada halaman utama (home page e-commerce) akan langsung menenggelamkan database utama dan melumpuhkan seluruh aplikasi.

## 4. Cara Kerjanya (Diagram Alur - Cache Aside)

```
                            +-------------+
                            |   Client    |
                            +-------------+
                                   |
                         1. GET /products/42
                                   v
                         +-------------------+
                         |  Backend Service  |
                         +-------------------+
                                   |
                     2. Cek Key "product:42"
                                   v
                         +-------------------+
                         |   Redis (Cache)   |
                         +-------------------+
                                   |
                 +-----------------+-----------------+
                 |                                   |
          [ Cache HIT ]                       [ Cache MISS ]
                 |                                   |
                 v                                   v
        3a. Return Data                     3b. Query Database
         (Latency <1ms)                              |
                 |                                   v
                 |                          +-----------------+
                 |                          |  PostgreSQL DB  |
                 |                          +-----------------+
                 |                                   |
                 |                          4b. Simpan ke Cache
                 |                              (dengan TTL + Jitter)
                 v                                   v
       +-----------------------------------------------+
       |         Kirimkan Response ke Client           |
       +-----------------------------------------------+
```

## 5. Cuplikan Logika Inti (Cache-Aside dengan Anti-Stampede Singleflight di Go)

Berikut implementasi pola Cache-Aside yang dilengkapi dengan penguncian mutex terpadu (_Singleflight_) di Go untuk mencegah _Cache Stampede_:

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
	"golang.org/x/sync/singleflight"
)

var (
	sfGroup singleflight.Group
	ctx     = context.Background()
)

type Product struct {
	ID    int64   `json:"id"`
	Name  string  `json:"name"`
	Price float64 `json:"price"`
}

// GetProductDetails menerapkan pola Cache-Aside dengan Singleflight (Anti-Stampede)
func GetProductDetails(rdb *redis.Client, productID int64) (*Product, error) {
	cacheKey := fmt.Sprintf("product:%d", productID)

	// 1. Coba baca dari Cache Redis
	val, err := rdb.Get(ctx, cacheKey).Result()
	if err == nil {
		var p Product
		_ = json.Unmarshal([]byte(val), &p)
		return &p, nil // Cache Hit
	}

	// 2. Cache Miss: Gunakan Singleflight agar dari 10.000 request konkuren,
	//    hanya 1 request yang memanggil database, sisanya menunggu hasilnya
	v, err, _ := sfGroup.Do(cacheKey, func() (interface{}, error) {
		product, dbErr := fetchProductFromDatabase(productID)
		if dbErr != nil {
			return nil, dbErr
		}

		// Simpan hasil ke Redis dengan TTL 10 menit
		data, _ := json.Marshal(product)
		rdb.Set(ctx, cacheKey, data, 10*time.Minute)
		return product, nil
	})

	if err != nil {
		return nil, err
	}
	return v.(*Product), nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Peningkatan Throughput Drastis**: Aplikasi dapat melayani jutaan pengguna dengan resource server database minimal.
-   **User Experience Superior**: Halaman aplikasi memuat data hampir seketika.
-   **Perlindungan Database**: Berfungsi sebagai tameng pelindung saat ada lonjakan trafik mendadak.

### Kekurangan

-   **Data Basi (_Stale Data_)**: Data di cache bisa berbeda dengan database jika data di database berubah tetapi cache belum dibersihkan.
-   **Biaya Memori (RAM)**: Harga memori RAM jauh lebih mahal per gigabyte dibanding penyimpanan SSD.
-   **Kompleksitas Invalidation**: "Ada dua hal tersulit dalam ilmu komputer: _Cache Invalidation_ dan _Naming Things_." (Phil Karlton).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Data yang rasio pembacaannya jauh lebih tinggi daripada penulisan (_Read-Heavy_, misalnya 90% Read dan 10% Write: katalog produk, profil publik pengguna, artikel berita).
-   Data konfigurasi sistem atau static metadata yang jarang sekali berubah.
-   Session login pengguna (_user session tokens_).

### Kapan Jangan Pakai

-   Data yang sifatnya _real-time_ kritis dan wajib 100% akurat setiap detik (misal: saldo rekening bank atau eksekusi order trading saham).
-   Data yang sangat jarang dibaca kembali (menyimpan data yang cuma dibaca sekali seumur hidup hanya akan membuang memori RAM).
-   Data yang sangat cepat berubah (setiap milidetik berubah); biaya mengupdate cache akan lebih mahal daripada manfaat pembacaannya.

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`cache_hit_ratio` (Gauge / Persentase)**: `Hits / (Hits + Misses) * 100%`. Standar sehat production adalah **> 85%**. Jika turun ke < 70%, database akan mulai kepanasan.
-   **`evicted_keys_total` (Counter)**: Jumlah key yang dibuang paksa oleh Redis karena RAM penuh (_maxmemory policy_). Jika angka ini naik tajam, alokasi RAM cache harus segera diperbesar.
-   **`used_memory` vs `maxmemory` (Gauge)**: Kapasitas pemakaian RAM instance cache. Pasang alert jika menyentuh 80%.
-   **`connected_clients` (Gauge)**: Memantau kebocoran koneksi pool ke Redis.

## 9. Studi Kasus Nyata di Industri

-   **Meta (Facebook)**: Mengoperasikan kluster **Memcached** terbesar di dunia yang menangani miliaran request per detik. Facebook merilis paper arsitektur legendaris tentang bagaimana mereka menangani masalah _Cache Stampede_ dan _Stale Data_ menggunakan teknik **Leases** (token sewa) di level server Memcached untuk menjamin konsistensi data profil dan social graph.
-   **Twitter / X**: Menggunakan arsitektur in-memory Redis cluster raksasa untuk menyimpan timeline tweet setiap pengguna aktif. Saat akun selebriti dengan jutaan followers mem-posting tweet baru, fan-out service Twitter langsung menyuntikkan tweet ID tersebut ke cache timeline followers secara bertahap.

## 10. Fakta Menarik & Gotchas Production

Tiga monster cache yang paling sering menenggelamkan sistem production:

1. **Cache Avalanche (Longsor Cache)**: Terjadi ketika ribuan data di cache dipasang dengan durasi kedaluwarsa (_TTL - Time To Live_) yang persis sama (misal semua diset 60 menit). Tepat di menit ke-60, semua key tersebut kedaluwarsa secara serentak. Ratusan ribu request tiba-tiba langsung menghantam database secara bersamaan hingga database tumbang.  
   _Solusi_: Tambahkan waktu acak (_Jitter_) pada TTL (misal: 60 menit + acak 1 hingga 5 menit).

2. **Cache Stampede / Thundering Herd**: Terjadi ketika satu data yang sangat populer (misal banner promo flash sale iPhone) kedaluwarsa di cache. Dalam hitungan 1 detik, ribuan request yang datang mendapati _Cache Miss_ dan semuanya serempak mengeksekusi query berat yang sama ke database.  
   _Solusi_: Gunakan teknik _Mutex Locking / Single Flight_ (hanya 1 worker yang boleh query database untuk merefresh cache, worker lain menunggu).

3. **Cache Penetration**: Penyerang sengaja mencari data dengan ID yang tidak pernah ada di database (misal `/products/-9999`). Cache selalu _Miss_, dan database selalu dicari padahal hasilnya nihil.  
   _Solusi_: Simpan nilai `null` di cache dengan TTL pendek, atau gunakan data struktur **Bloom Filter** di depan cache.
