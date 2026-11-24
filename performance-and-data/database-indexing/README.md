# Database Indexing

## 1. Skenario Sistem Nyata

Pada tabel `orders` yang menampung 50.000.000 (50 juta) baris data transaksi pada platform e-commerce, pelanggan membuka halaman riwayat transaksi untuk melihat 10 pesanan terakhir mereka:

```sql
SELECT id, total_amount, status, created_at
FROM orders
WHERE user_id = 482910
ORDER BY created_at DESC
LIMIT 10;
```

Jika kolom `user_id` dan `created_at` **tidak memiliki indeks**:

-   Database PostgreSQL terpaksa membaca seluruh 50 juta baris data dari disk ke memori satu per satu (_Sequential Scan / Full Table Scan_).
-   Setiap satu query membutuhkan waktu eksekusi 4,2 detik dan menguras ratusan megabyte Disk I/O.
-   Ketika 100 pengguna membuka halaman riwayat transaksi secara bersamaan, database langsung mencapai 100% CPU utilization, antrean connection pool habis, dan seluruh backend mengalami _cascading timeout_.

**Solusi dengan Database Indexing (B-Tree Composite Index):**
Dengan menambahkan indeks komposit:

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_created ON orders (user_id, created_at DESC);
```

Database membuat struktur data pohon seimbang (_B-Tree_) terpisah. Alih-alih memeriksa 50 juta baris, PostgreSQL hanya menelusuri 3 hingga 4 lompatan node dari _Root_ ke _Leaf_ untuk menemukan data spesifik pengguna tersebut. Waktu eksekusi anjlok dari **4.200 milidetik menjadi 0,6 milidetik** (peningkatan efisiensi lebih dari 7.000 kali lipat!).

## 2. Apa Itu dan Buat Apa?

Database Indexing adalah teknik membuat struktur data penolong terpisah yang memetakan nilai kolom tertentu ke lokasi fisik baris data di storage disk (_pointer tuple_).

Fungsi utamanya adalah mengubah kompleksitas pencarian data dari linear $O(N)$ menjadi logaritmik $O(\log N)$:

-   **B-Tree (Balanced Tree)**: Jenis indeks default dan paling universal. Sangat optimal untuk operator kesetaraan (`=`), perbandingan jangkauan (`<`, `<=`, `>`, `>=`, `BETWEEN`), dan pengurutan (`ORDER BY`).
-   **Composite Index**: Indeks yang menggabungkan dua atau lebih kolom sekaligus (misal `user_id` + `created_at`). Urutan kolom dari kiri ke kanan (_leading column_) sangat menentukan apakah indeks dapat digunakan atau tidak.
-   **Hash Index**: Hanya mendukung operasi kesetaraan (`=`) dengan kecepatan $O(1)$, namun tidak mendukung query rentang (_range scan_).
-   **GIN / GiST Index**: Indeks khusus untuk pencarian teks penuh (_Full-Text Search_), array, atau data geospasial (GIS).

## 3. Kenapa Harus Pakai Ini ?

Pada arsitektur backend berkecepatan tinggi, database relasional adalah sumber daya bersama (_shared bottleneck_):

1. **Mengurangi Durasi Hold Koneksi**: Query yang selesai dalam 1 milidetik langsung mengembalikan koneksi ke pool, sehingga 50 koneksi database mampu melayani ribuan RPS tanpa antrean.
2. **Menyelamatkan Disk I/O & Memory Cache**: Indeks meminimalkan jumlah blok data yang harus dimuat dari disk ke memori buffer (_shared buffers_).
3. **Mencegah Table Locking**: Query `UPDATE` atau `DELETE` yang menggunakan indeks hanya mengunci baris spesifik (_row-level lock_), bukan mengunci seluruh tabel.

## 4. Cara Kerjanya (Diagram Alur B-Tree Index)

```
                            +--------------------+
                            |     Root Node      |
                            |   [Keys: 100, 500] |
                            +--------------------+
                                 /    |     \
       +------------------------+     |      +------------------------+
       |                              |                               |
       v                              v                               v
+------------------+         +------------------+            +------------------+
|   Branch Node    |         |   Branch Node    |            |   Branch Node    |
|  [Keys: 20, 50]  |         | [Keys: 200, 350] |            | [Keys: 650, 800] |
+------------------+         +------------------+            +------------------+
   /      |     \                 /      |     \                  /      |     \
  v       v      v               v       v      v                v       v      v
+---+   +---+  +---+           +---+   +---+  +---+            +---+   +---+  +---+
| 5 |   |30 |  |80 |           |150|   |280|  |450|            |600|   |720|  |950| <-- Leaf Nodes
+---+   +---+  +---+           +---+   +---+  +---+            +---+   +---+  +---+
  |       |      |               |       |      |                |       |      |
  v       v      v               v       v      v                v       v      v
 [ Tuple Pointer ] ------------> [ Fisik Baris Data di Disk / Heap Storage Table ]
```

## 5. Cuplikan Logika Inti (Eksekusi Query Terindeks & Verifikasi EXPLAIN di Go)

Berikut implementasi eksekusi query Go yang teroptimasi indeks, disertai fungsi diagnostik untuk memeriksa _execution plan_ menggunakan `EXPLAIN (ANALYZE, BUFFERS)`:

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"strings"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

type Order struct {
	ID          int64     `json:"id"`
	TotalAmount float64   `json:"total_amount"`
	Status      string    `json:"status"`
	CreatedAt   time.Time `json:"created_at"`
}

// GetUserRecentOrders mengeksekusi query yang memanfaatkan composite index (user_id, created_at DESC)
func GetUserRecentOrders(ctx context.Context, db *sql.DB, userID int64, limit int) ([]Order, error) {
	query := `
		SELECT id, total_amount, status, created_at
		FROM orders
		WHERE user_id = $1
		ORDER BY created_at DESC
		LIMIT $2;
	`

	rows, err := db.QueryContext(ctx, query, userID, limit)
	if err != nil {
		return nil, fmt.Errorf("gagal query orders: %w", err)
	}
	defer rows.Close()

	var orders []Order
	for rows.Next() {
		var o Order
		if err := rows.Scan(&o.ID, &o.TotalAmount, &o.Status, &o.CreatedAt); err != nil {
			return nil, err
		}
		orders = append(orders, o)
	}
	return orders, nil
}

// VerifyIndexUsage mengeksekusi EXPLAIN ANALYZE untuk memastikan query menggunakan Index Scan, bukan Seq Scan
func VerifyIndexUsage(ctx context.Context, db *sql.DB, userID int64) error {
	explainQuery := `
		EXPLAIN (ANALYZE, BUFFERS)
		SELECT id, total_amount, status, created_at
		FROM orders
		WHERE user_id = $1
		ORDER BY created_at DESC
		LIMIT 10;
	`

	rows, err := db.QueryContext(ctx, explainQuery, userID)
	if err != nil {
		return err
	}
	defer rows.Close()

	var planLines []string
	var isUsingIndex bool

	for rows.Next() {
		var line string
		if err := rows.Scan(&line); err != nil {
			return err
		}
		planLines = append(planLines, line)
		if strings.Contains(line, "Index Scan") || strings.Contains(line, "Bitmap Index Scan") {
			isUsingIndex = true
		}
	}

	if !isUsingIndex {
		log.Printf("[WARNING] Query TIDAK menggunakan Index! Full Table Scan terdeteksi:\n%s", strings.Join(planLines, "\n"))
	} else {
		log.Printf("[OK] Query berhasil menggunakan Index Scan:\n%s", planLines[0])
	}
	return nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Performa Baca Eksponensial**: Memangkas waktu query dari hitungan detik menjadi sub-milidetik.
-   **Efisiensi Pengurutan (_Zero-Cost Sorting_)**: Indeks B-Tree sudah tersimpan dalam keadaan terurut, sehingga query `ORDER BY` tidak memerlukan kalkulasi sort berat di memori (_in-memory quicksort_).
-   **Mendukung Aturan Unik (_Data Integrity_)**: Menjamin keunikan nilai kolom secara atomik via `UNIQUE INDEX`.

### Kekurangan

-   **Overhead Penulisan (_Write Penalty_)**: Setiap kali operasi `INSERT`, `UPDATE`, atau `DELETE` terjadi, database harus memperbarui struktur pohon B-Tree pada seluruh indeks yang ada di tabel tersebut.
-   **Konsumsi Ruang Disk Ekstra**: Indeks pada tabel raksasa dapat memakan kapasitas disk yang setara atau bahkan lebih besar daripada ukuran tabel data itu sendiri.
-   **Beban Memori RAM**: Agar bekerja maksimal, indeks-indeks penting harus selalu tersimpan di dalam memori RAM (_buffer pool_).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Kolom yang sering dijadikan filter klausa `WHERE` dengan kardinalitas tinggi (misal `user_id`, `email`, `order_number`).
-   Kolom kunci asing (_Foreign Keys_) yang digunakan untuk relasi `JOIN`.
-   Kolom yang sering digunakan untuk pengurutan data (`ORDER BY created_at DESC`).

### Kapan Jangan Pakai

-   Kolom dengan kardinalitas sangat rendah di mana variasi nilainya hanya 2 atau 3 jenis (misal kolom `is_active` boolean atau status `gender`). Database engine biasanya lebih memilih Sequential Scan.
-   Tabel yang sangat kecil (kurang dari beberapa ratus baris data). Full scan pada satu halaman disk sering kali lebih cepat dibanding membaca indeks lalu membaca heap table.
-   Tabel log bervolume tulis ekstrem (_write-heavy ingestion table_) yang jarang dibaca secara real-time.

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`pg_stat_user_tables_idx_scan` vs `seq_scan` (Counter)**: Rasio pencarian indeks dibanding scan tabel sekuensial. Nilai sehat pada tabel transaksi produksi adalah > 95% `idx_scan`.
-   **`pg_stat_statements_mean_exec_time` (Gauge)**: Rata-rata durasi eksekusi query SQL. Query yang tiba-tiba melambat mengindikasikan query planner mengabaikan indeks.
-   **`pg_stat_user_indexes_idx_scan` (Counter)**: Mendeteksi _Unused Indexes_ (indeks dengan nilai scan 0) yang hanya membuang disk dan memperlambat write.
-   **Index Bloat Ratio (Gauge)**: Persentase ruang kosong tidak efisien pada halaman B-Tree akibat proses update/delete berulang.

## 9. Studi Kasus Industri Nyata

-   **GitHub**: Menangani ratusan juta query per menit untuk menampilkan commit, issue, dan pull request. Tim infrastruktur GitHub membagikan rekayasa optimasi indeks mendalam pada PostgreSQL/MySQL, termasuk pembersihan indeks yang tidak terpakai (_unused indexes_) yang berhasil menghemat terabyte memori RAM dan mempercepat operasi write transaksi hingga 35%.
-   **Uber**: Pada arsitektur basis data _Schemaless_ mereka di atas MySQL, Uber merancang strategi pengindeksan sekunder khusus (_secondary indexing_) yang diisolasi agar operasi update data perjalanan pengemudi tidak memicu fragmentasi dan lock contention pada B-Tree global.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Membuat Indeks Tanpa `CONCURRENTLY`**: Menjalankan perintah `CREATE INDEX` biasa pada tabel transaksi bernilai jutaan baris akan mengunci seluruh operasi tulis (_Exclusive Write Lock_). Seluruh proses checkout e-commerce akan berhenti dan gagal total selama proses indexing berjalan.  
   _Solusi_: Selalu gunakan `CREATE INDEX CONCURRENTLY` di PostgreSQL.
2. **Aturan Urutan Kolom Kiri (_Leading Column Rule_)**: Jika indeks komposit dibuat pada `(user_id, status)`, query `WHERE status = 'PENDING'` tidak akan bisa memanfaatkan indeks tersebut karena kolom pertama (`user_id`) tidak disertakan dalam filter query.
3. **Membungkus Kolom dengan Fungsi**: Query `WHERE UPPER(email) = 'USER@EXAMPLE.COM'` tidak akan bisa memakai indeks standar pada kolom `email`.  
   _Solusi_: Buat _Expression Index_ / _Functional Index_: `CREATE INDEX idx_user_lower_email ON users (LOWER(email));`.

## 11. Rangkuman Singkat

Database Indexing adalah mekanisme fundamental yang mengubah pencarian linear $O(N)$ menjadi pencarian pohon logaritmik $O(\log N)$. Pengindeksan yang tepat adalah kunci utama latensi sub-milidetik pada sistem berskala besar, dengan trade-off berupa konsumsi memori dan sedikit penalti pada kecepatan penulisan data.
