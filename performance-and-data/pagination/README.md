# Database Pagination (Offset vs Cursor-Based)

## 1. Skenario Sistem Nyata

Pada sebuah platform media sosial dan marketplace e-commerce dengan tabel `posts` atau `products` yang memiliki 10.000.000 (10 juta) baris data:

Jika backend menggunakan **Offset-Based Pagination** (`LIMIT ... OFFSET ...`):

```sql
SELECT id, title, created_at FROM posts ORDER BY id DESC LIMIT 20 OFFSET 500000;
```

-   Saat pengguna membuka Halaman 1 (`OFFSET 0`), query berjalan cepat (1 milidetik).
-   Namun ketika pengguna atau bot web scraper menavigasi ke halaman dalam (_deep pagination_, misal Halaman 25.000 dengan `OFFSET 500000`), mesin database PostgreSQL terpaksa membaca seluruh 500.020 baris dari storage disk, mengurutkannya di memori, lalu **membuang 500.000 baris pertama** dan hanya mengembalikan 20 baris terakhir!
-   Waktu eksekusi query melonjak dari 1 ms menjadi **8,5 detik**, menguras CPU database dan memori buffer cache secara sia-sia.
-   **Masalah Data Drift (Duplikasi Data)**: Jika ada 5 post baru dibuat saat pengguna sedang membaca Halaman 1, ketika mereka membuka Halaman 2, 5 postingan terbawah dari Halaman 1 akan terdorong ke Halaman 2 sehingga pengguna melihat postingan yang sama dua kali!

**Solusi dengan Cursor-Based Pagination (Keyset / Seek Method):**
Alih-alih melompati baris dengan `OFFSET`, aplikasi menggunakan nilai ID baris terakhir yang dilihat pengguna sebagai penunjuk posisi (_cursor_):

```sql
SELECT id, title, created_at FROM posts WHERE id < 948210 ORDER BY id DESC LIMIT 20;
```

Dengan indeks pada kolom `id`, PostgreSQL langsung melakukan _Index Seek_ melompat tepat ke baris `948210` dan mengambil 20 baris berikutnya dalam waktu **0,4 milidetik**! Tidak peduli apakah data berada di halaman ke-1 atau halaman ke-500.000, latensi query tetap konstan di sub-milidetik, dan tidak akan pernah ada data duplikat meskipun jutaan post baru terus bertambah.

## 2. Apa Itu dan Buat Apa?

Pagination adalah teknik membagi himpunan data berukuran besar menjadi potongan-potongan halaman kecil (_chunks_) agar efisien ditransmisikan melalui jaringan dan nyaman ditampilkan pada antarmuka pengguna.

Dua strategi pagination utama di industri:

1. **Offset-Based Pagination (Metode Nomor Halaman)**:
    - Menggunakan parameter `page` dan `limit` (misal `?page=5&limit=20` diterjemahkan ke `OFFSET 80 LIMIT 20`).
    - Sangat intuitif untuk navigasi UI yang memiliki nomor halaman eksplisit (1, 2, 3, 4, ... 100).
    - Kompleksitas waktu membaca database: $O(N)$ di mana $N$ adalah nilai offset. Semakin dalam halamannya, semakin lambat query-nya.
2. **Cursor-Based Pagination (Keyset / Seek Method)**:
    - Menggunakan penanda posisi unik terurut (_opaque cursor_, biasanya string Base64 dari timestamp atau ID) sebagai acuan awal pencarian halaman berikutnya (`?cursor=eyJpZCI6OTQ4MjEwfQ==&limit=20`).
    - Standar emas industri untuk _Infinite Scrolling_ (Instagram, Twitter/X, TikTok, Slack API, Stripe API).
    - Kompleksitas waktu membaca database: $O(1)$ atau $O(\log N)$ yang konstan stabil di kedalaman data mana pun.

## 3. Kenapa Harus Pakai Ini ?

Di sistem backend berskala tinggi:

1. **Mencegah Latensi Meledak pada Data Besar**: Query tidak akan mengalami degradasi performa meski tabel bertumbuh hingga ratusan juta baris data.
2. **Eliminasi Data Drift**: Menjamin integritas data saat scrolling berkelanjutan; item baru yang masuk di posisi atas tidak akan menggeser item yang sedang dibaca.
3. **Menghemat I/O dan Memori Database**: Database hanya membaca tepat 20 baris data dari disk, bukan membaca puluhan ribu baris lalu membuangnya.

## 4. Cara Kerjanya (Perbandingan Alur Offset vs Cursor)

```
[Offset-Based Pagination: O(N) Scan and Drop]
Query: OFFSET 50000 LIMIT 5
Database memindai 50.005 baris:
+-------------------------------------------------------------+---------+
|     50,000 Baris Pertama Dipindai Lalu DIBUANG DARI RAM     | 5 Baris |
+-------------------------------------------------------------+---------+
                                                                   |
                                                                   v
                                                        [Dikembalikan ke Client]
(Membuang waktu komputasi & I/O untuk 50.000 baris yang sia-sia!)

-------------------------------------------------------------------------

[Cursor-Based Pagination: O(log N) Index Direct Seek]
Query: WHERE id < 948210 ORDER BY id DESC LIMIT 5
Database memanfaatkan B-Tree Index:
                  [B-Tree Root]
                        |
                        v
               [Lompat Langsung ke 948210]
                        |
                        +---> Ambil 5 baris berikutnya secara berurutan
                        |
                        v
             [Dikembalikan ke Client: < 0.5 ms]
```

## 5. Cuplikan Logika Inti (Cursor-Based Pagination di Go)

Berikut implementasi cursor pagination di Go dengan encoding Base64 dan klausul `WHERE id < :cursor LIMIT :limit`:

```go
package main

import (
	"context"
	"database/sql"
	"encoding/base64"
	"fmt"
	"strconv"
)

type Post struct {
	ID    int64  `json:"id"`
	Title string `json:"title"`
}

type PaginatedResponse struct {
	Data       []Post  `json:"data"`
	NextCursor *string `json:"next_cursor"`
	HasMore    bool    `json:"has_more"`
}

// FetchPostsCursor mengeksekusi pagination berbasis cursor berkecepatan konstan
func FetchPostsCursor(ctx context.Context, db *sql.DB, cursorStr string, limit int) (*PaginatedResponse, error) {
	var lastID int64
	if cursorStr != "" {
		// 1. Decode opaque cursor dari Base64
		decoded, err := base64.StdEncoding.DecodeString(cursorStr)
		if err != nil {
			return nil, fmt.Errorf("cursor tidak valid: %w", err)
		}
		lastID, _ = strconv.ParseInt(string(decoded), 10, 64)
	}

	// 2. Query data dengan LIMIT + 1 untuk mengecek apakah masih ada halaman berikutnya
	query := "SELECT id, title FROM posts WHERE ($1 = 0 OR id < $1) ORDER BY id DESC LIMIT $2"
	rows, err := db.QueryContext(ctx, query, lastID, limit+1)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var posts []Post
	for rows.Next() {
		var p Post
		if err := rows.Scan(&p.ID, &p.Title); err != nil {
			return nil, err
		}
		posts = append(posts, p)
	}

	hasMore := len(posts) > limit
	if hasMore {
		// Buang item ke-(limit+1) yang hanya dipakai untuk penanda hasMore
		posts = posts[:limit]
	}

	var nextCursor *string
	if hasMore && len(posts) > 0 {
		// 3. Encode ID terakhir menjadi string cursor Base64 untuk request berikutnya
		lastItem := posts[len(posts)-1]
		encodedCursor := base64.StdEncoding.EncodeToString([]byte(strconv.FormatInt(lastItem.ID, 10)))
		nextCursor = &encodedCursor
	}

	return &PaginatedResponse{
		Data:       posts,
		NextCursor: nextCursor,
		HasMore:    hasMore,
	}, nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Offset-Based

-   **Kelebihan**: Sangat mudah diimplementasikan, mendukung navigasi lompat langsung ke sembarang nomor halaman (misal: langsung lompat dari Hal 1 ke Hal 15), mudah menampilkan total halaman (`COUNT(*)`).
-   **Kekurangan**: Performa anjlok di halaman dalam ($O(N)$), rentan data duplikat atau terlewat saat data baru masuk (_data drift_), kalkulasi `COUNT(*)` pada tabel jutaan baris sangat berat.

### Cursor-Based

-   **Kelebihan**: Performa selalu sub-milidetik konstan di kedalaman mana pun ($O(1)$), imun terhadap data duplikasi saat penambahan data baru, sangat ideal untuk infinite scroll mobile app.
-   **Kekurangan**: Tidak bisa lompat langsung ke halaman acak (tidak ada konsep "Halaman 5"), tidak bisa menampilkan total keseluruhan halaman, membutuhkan kolom penunjuk yang terurut dan unik (_sequential unique column_).

## 7. Kapan Harus Pakai Offset vs Cursor

### Pilih Offset Pagination Jika:

-   Halaman web memiliki navigasi nomor halaman tabel admin (UI tabel dashboard internal dengan tombol `[1] [2] [3] ... [Terakhir]`).
-   Ukuran total data relatif kecil (kurang dari puluhan ribu baris) di mana `OFFSET` belum menimbulkan masalah latensi.
-   Data bersifat statis dan jarang ada penambahan data baru yang memicu pergeseran indeks.

### Pilih Cursor Pagination Jika:

-   Aplikasi berbasis _Infinite Scroll_ (Feed sosial, linimasa komentar, katalog e-commerce, chat history).
-   Tabel basis data berukuran besar (ratusan ribu hingga ratusan juta baris).
-   Data bervolume tinggi yang terus-menerus bertambah secara real-time.
-   API publik modern (mengikuti standar Stripe, Slack, dan GitHub API).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`db_query_duration_seconds` per Pagination Type (Histogram)**: Membandingkan p99 latensi query pagination. Jika query offset melonjak tinggi di halaman dalam, segera migrasi ke cursor.
-   **`db_blocks_read_total` (Counter)**: Jumlah blok disk storage yang dimuat database. Offset besar akan menghasilkan lonjakan pembacaan blok yang tajam.
-   **`api_invalid_cursor_errors_total` (Counter)**: Mendeteksi klien yang mengirimkan string cursor yang rusak atau dimanipulasi.

## 9. Studi Kasus Industri Nyata

### Eliminasi Latensi Infinite Scroll & Page Drift pada Feed Media Sosial

- **Tantangan Rekayasa Riil (The Problem)**:
  Sebuah aplikasi linimasa konten berkembang pesat hingga memiliki 35 juta rekaman pada tabel `posts`. Aplikasi mobile awalnya menggunakan navigasi bertingkat berbasis *Offset Pagination* (`GET /api/feed?page=250&limit=20` yang diterjemahkan menjadi `OFFSET 5000 LIMIT 20`). Di lingkungan produksi, muncul dua bencana rekayasa:
  1. *Degradasi Latensi & Buffer Thrashing*: Saat pengguna aktif atau bot melakukan scraping hingga halaman dalam, PostgreSQL harus membaca 5.000 baris data dari disk/RAM lalu membuangnya begitu saja hanya untuk menyajikan 20 baris terakhir. Latensi query melonjak dari 8ms di halaman pertama menjadi 5.200ms di halaman ke-250, menguras kapasitas CPU database hingga 100%.
  2. *Anomali Konten Duplikat (Page Drift)*: Ketika pengguna sedang membaca halaman 2, ada 5 postingan baru diterbitkan ke linimasa. Saat pengguna menggulir ke bawah untuk memuat halaman 3 (`OFFSET 40`), seluruh data terdorong turun sejauh 5 baris, mengakibatkan 5 postingan yang sudah dibaca di halaman 2 muncul kembali di halaman 3 dan merusak pengalaman pengguna.

- **Implementasi Solusi Industri (The Architectural Solution)**:
  1. **Migrasi ke Keyset (Cursor-Based) Pagination**: Query offset dirombak total menggunakan predikat perbandingan berindeks komposit:
     ```sql
     SELECT id, created_at, content, author_id 
     FROM posts 
     WHERE (created_at, id) < ($1, $2) 
     ORDER BY created_at DESC, id DESC 
     LIMIT 20;
     ```
  2. **B-Tree Seek Optimization**: Database dipasangi indeks gabungan `CREATE INDEX idx_posts_created_id ON posts (created_at DESC, id DESC)`. Alih-alih melakukan scan dan discard, query planner PostgreSQL langsung melakukan *Index Seek* $O(1)$ melompat ke titik koordinat kursor dalam hitungan mikrodetik.
  3. **Opaque Base64 Cursor Token**: Di layer API Go, nilai `created_at` dan `id` terakhir dienkode menjadi string Base64 yang tidak mudah ditebak (`next_cursor = "ZXlK..."`) dan disematkan ke payload JSON respons.
  4. **Peniadaan `COUNT(*)`**: UI mobile diinstruksikan menghapus informasi total halaman dan hanya membaca status ketersediaan data via flag `has_more: bool`.

- **Dampak & Hasil Sistem (The Engineering Outcome)**:
  - Latensi pembacaan data di halaman awal maupun di kedalaman ribuan data menjadi seragam dan deterministik di angka 1.2ms (pemangkasan latensi 99.97%).
  - Masalah konten berulang (*page drift*) tereliminasi 100% karena posisi pembacaan terkunci pada koordinat timestamp dan ID fisik terakhir.
  - Beban I/O disk database turun 85%, memungkinkan sistem menampung lonjakan trafik pengguna aktif tanpa perlu menambah kapasitas RAM database.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Menggunakan Kolom Non-Unik Sebagai Kursor Tunggal**: Menggunakan kolom `created_at` saja sebagai kursor. Jika ada 10 transaksi dibuat pada detik/milidetik yang persis sama, kursor `WHERE created_at < :last_time` akan melompati sebagian transaksi tersebut!  
   _Solusi_: Gunakan **Composite Tie-Breaker Cursor** (`WHERE (created_at, id) < (:last_time, :last_id)`).
2. **Melakukan `SELECT COUNT(*)` di Setiap Halaman Cursor**: Menggunakan cursor pagination untuk kecepatan, tetapi di saat bersamaan menjalankan query `SELECT COUNT(*) FROM table` untuk menampilkan total item. Query `COUNT(*)` pada tabel 50 juta baris akan tetap membuat database lambat!  
   _Solusi_: Jangan tampilkan total item di infinite scroll; cukup kembalikan flag boolean `has_more`.
3. **Cursor Tidak Terlindungi (_Tampering_)**: Mengirimkan cursor mentah yang mudah ditebak pengguna. Jika pengguna memodifikasi ID kursor, mereka dapat membocorkan data yang tidak seharusnya diakses.  
   _Solusi_: Encode cursor menggunakan Base64 terenkripsi atau HMAC signature.

## 11. Rangkuman Singkat

1. **Efisiensi $O(1)$ Melalui Index Seek**: Cursor pagination mengubah pemindaian linier $O(N)$ yang membuang jutaan baris menjadi pencarian koordinat langsung pada pohon B-Tree berlatensi konstan.
2. **Wajib Tie-Breaker Unik**: Selalu pasangkan kolom waktu dengan kolom unik (`created_at, id`) untuk mencegah data terlewat jika terjadi duplikasi timestamp pada baris data.
3. **Kebal Terhadap Page Drift**: Menjamin pengguna aplikasi mobile tidak akan melihat item yang sama berulang kali saat data baru masuk secara real-time.
4. **Hilangkan `SELECT COUNT(*)`**: Pada sistem ber-throughput tinggi dengan data jutaan baris, hindari menghitung total baris dan cukup kembalikan penunjuk `next_cursor` beserta flag `has_more`.
