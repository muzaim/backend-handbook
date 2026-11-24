# Zero-Downtime Database Migration (Expand and Contract Pattern)

## 1. Skenario Sistem Nyata

Pada sebuah platform finansial dengan tabel `users` yang menampung 40.000.000 (40 juta) baris data aktif, tim produk meminta perubahan skema: kolom lama `name` (string tunggal) harus dipecah menjadi dua kolom terpisah: `first_name` dan `last_name`.

Jika perubahan ini dieksekusi secara naif langsung di basis data produksi:

```sql
-- EKSEKUSI NAIF YANG FATAL DI PRODUCTION:
ALTER TABLE users
    ADD COLUMN first_name VARCHAR(100) NOT NULL,
    ADD COLUMN last_name VARCHAR(100) NOT NULL,
    DROP COLUMN name;
```

-   Mesin PostgreSQL / MySQL terpaksa mengambil kunci eksklusif penuh (_AccessExclusiveLock_) pada tabel `users`.
-   Selama 25 menit proses penulisan ulang tabel di disk berlangsung, **seluruh operasi baca (`SELECT`) dan tulis (`INSERT/UPDATE`) pada tabel `users` diblokir total**.
-   Puluhan ribu koneksi pengguna yang mencoba login atau checkout mengalami _timeout_, antrean connection pool habis, dan seluruh sistem backend tumbang seketika.
-   Lebih buruk lagi: pod aplikasi Go versi lama yang masih berjalan di Kubernetes langsung _crash_ massal karena kode mereka masih mencari kolom `name` yang sudah dihapus secara mendadak (_Breaking Change_)!

**Solusi dengan Pola Expand and Contract:**
Perubahan skema dibagi menjadi 4 tahapan bertahap (_multi-phase deployment_) yang menjamin aplikasi versi lama dan versi baru dapat berjalan berdampingan tanpa mengunci tabel dan **tanpa downtime satu detik pun**.

## 2. Apa Itu dan Buat Apa?

Zero-Downtime Database Migration adalah metodologi rekayasa skema basis data yang memungkinkan perubahan struktur tabel (DDL - Data Definition Language) pada lingkungan produksi bervolume tinggi tanpa menghentikan ketersediaan sistem (_zero service disruption_).

Pola standar industri yang digunakan adalah **Expand and Contract Pattern** (juga dikenal sebagai _Parallel Run_):

1. **Fase 1 (Expand)**: Menambahkan kolom/tabel baru tanpa menghapus kolom lama. Kolom baru harus bersifat _nullable_ agar tidak memicu penguncian tabel.
2. **Fase 2 (Dual-Write di Kode Aplikasi)**: Aplikasi diperbarui agar menulis data ke kolom lama DAN kolom baru secara bersamaan.
3. **Fase 3 (Backfill Data Historis)**: Menjalankan background worker untuk menyalin data lama ke kolom baru dalam potongan batch kecil (_chunked throttling_) tanpa membebani CPU database.
4. **Fase 4 (Contract)**: Kode aplikasi dialihkan murni membaca dari kolom baru. Setelah aman dan teruji, kolom lama dihapus dari database.

## 3. Kenapa Harus Dipahami ?

Dalam arsitektur microservices dan continuous delivery (CI/CD):

1. **SLA Ketersediaan 99.99%**: Platform modern tidak lagi mengizinkan pemadaman sistem (_maintenance window_) hanya untuk menjalankan migrasi database.
2. **Kompatibilitas Maju-Mundur (_Backward & Forward Compatibility_)**: Dalam deployment rolling update di Kubernetes, pod versi $N$ (lama) dan pod versi $N+1$ (baru) akan berjalan bersamaan selama beberapa menit. Skema database wajib mendukung kedua versi kode tersebut secara simultan.
3. **Mencegah Bencana Kunci Tabel (_Table Lock Deadlock_)**: Menjalankan migrasi besar secara bertahap menjaga latensi query pengguna tetap di level sub-milidetik.

## 4. Cara Kerjanya (Diagram Alur 4 Fase Expand and Contract)

```
[FASE 1: EXPAND (Database DDL)]
Tambahkan kolom baru (nullable):
users: [ id | name | first_name (NULL) | last_name (NULL) ]
(Operasi instan di PostgreSQL, zero table lock!)
                         |
                         v
[FASE 2: DUAL WRITE (Kode Go Deployment)]
Aplikasi menulis ke KEDUA kolom secara bersamaan:
Incoming Write: "Budi Santoso"
  --> UPDATE users SET name = 'Budi Santoso',
                       first_name = 'Budi', last_name = 'Santoso';
(Aplikasi lama tetap bisa membaca 'name', aplikasi baru membaca 'first_name')
                         |
                         v
[FASE 3: BACKFILL (Background Worker Go)]
Salin 40 juta data historis secara bertahap (5,000 baris/batch):
WHERE first_name IS NULL LIMIT 5000;
(CPU database tetap stabil 20%, tidak ada lonjakan beban)
                         |
                         v
[FASE 4: CONTRACT (Pembersihan)]
1. Ubah kode Go: Hanya membaca dan menulis ke first_name & last_name.
2. Hapus kolom usang: ALTER TABLE users DROP COLUMN name;
(Selesai dengan 100% Uptime!)
```

## 5. Cuplikan Logika Inti (Dual-Write Repository & Backfill Worker di Go)

### 1. Pola Dual-Write di Repository Go (Fase 2)

```go
package main

import (
	"context"
	"database/sql"
	"strings"
)

type User struct {
	ID        int64
	FirstName string
	LastName  string
}

// CreateUserDualWrite menulis ke kolom lama dan kolom baru secara bersamaan
func CreateUserDualWrite(ctx context.Context, db *sql.DB, user User) error {
	// Kolom lama: name gabungan
	fullName := strings.TrimSpace(user.FirstName + " " + user.LastName)

	query := `
		INSERT INTO users (name, first_name, last_name)
		VALUES ($1, $2, $3);
	`
	_, err := db.ExecContext(ctx, query, fullName, user.FirstName, user.LastName)
	return err
}
```

### 2. Script Backfill Data Historis dengan Throttling (Fase 3)

```go
package main

import (
	"context"
	"database/sql"
	"log"
	"strings"
	"time"
)

// BackfillHistoricalData menyalin data lama secara bertahap (batch chunking)
func BackfillHistoricalData(ctx context.Context, db *sql.DB, batchSize int) {
	for {
		// 1. Ambil 5.000 baris yang belum di-backfill
		rows, err := db.QueryContext(ctx, `
			SELECT id, name FROM users
			WHERE first_name IS NULL
			ORDER BY id ASC
			LIMIT $1;
		`, batchSize)
		if err != nil {
			log.Fatalf("Error query backfill: %v", err)
		}

		var count int
		for rows.Next() {
			var id int64
			var name string
			_ = rows.Scan(&id, &name)

			parts := strings.SplitN(name, " ", 2)
			firstName := parts[0]
			lastName := ""
			if len(parts) > 1 {
				lastName = parts[1]
			}

			// 2. Perbarui kolom baru per baris atau batch
			_, _ = db.ExecContext(ctx, `
				UPDATE users
				SET first_name = $1, last_name = $2
				WHERE id = $3;
			`, firstName, lastName, id)

			count++
		}
		rows.Close()

		log.Printf("[Backfill] Berhasil memperbarui %d baris...", count)

		// Jika data yang diproses kurang dari batch size, berarti seluruh data sudah selesai!
		if count < batchSize {
			log.Println("[Backfill] SELESAI! Seluruh data historis berhasil dimigrasi.")
			break
		}

		// 3. PENTING: Tidur sejenak agar CPU dan replikasi database tidak panas
		time.Sleep(100 * time.Millisecond)
	}
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Nol Downtime Pengguna (_Zero Customer Impact_)**: Layanan tetap beroperasi normal 100% selama seluruh proses migrasi skema berlangsung.
-   **Dukungan Rollback Aman**: Jika kode baru bermasalah di Fase 2, deployment dapat di-rollback seketika karena kolom lama masih ada dan tetap terupdate.
-   **Beban Database Terkendali**: Pembaruan data historis dilakukan secara terukur melalui batch throttling.

### Kekurangan

-   **Siklus Rilis Lebih Panjang**: Membutuhkan minimal 2 hingga 3 kali deployment terpisah (tidak bisa selesai dalam 1 kali merge).
-   **Overhead Kode Sementara**: Kode repository harus memelihara logika dual-write sampai seluruh data historis selesai dimigrasi.
-   **Penyimpanan Disk Tambahan Sementara**: Data tersimpan ganda di kolom lama dan kolom baru sebelum fase pembersihan dilakukan.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Tabel produksi berukuran besar (ratusan ribu hingga ratusan juta baris data).
-   Sistem berskala tinggi yang menuntut ketersediaan tanpa henti (e-commerce, perbankan, ride-hailing).
-   Perubahan skema yang bersifat _Breaking Changes_ (mengubah nama kolom, memecah/menggabungkan kolom, mengubah tipe data).

### Kapan Jangan Pakai

-   Tahap awal pengembangan produk (_early-stage prototyping_) di mana database masih berjalan di lingkungan lokal dan downtime singkat di staging tidak berdampak pada bisnis.
-   Tabel referensi statis berukuran kecil (kurang dari 1.000 baris data).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`pg_stat_activity` Lock Wait Time (Gauge)**: Memantau durasi query yang mengantre akibat penguncian skema. Jika terdeteksi lock > 2 detik, batalkan migrasi seketika!
-   **`pg_stat_replication_lag_bytes` (Gauge)**: Memantau replication lag database replica saat proses backfill berjalan. Jika lag menanjak, perlambat interval sleep backfill.
-   **`backfill_unmigrated_rows_count` (Gauge)**: Menghitung sisa baris data yang belum selesai dimigrasi (`WHERE new_col IS NULL`).

## 9. Studi Kasus Industri Nyata

### Refactoring Skema Tabel Akun 28 Juta Baris Tanpa Downtime

- **Tantangan Rekayasa Riil (The Problem)**:
  Pada sebuah platform perbankan digital, tim engineering perlu memecah kolom `full_name` menjadi `first_name` dan `last_name` pada tabel `users` yang menampung 28 juta baris data aktif. Developer pemula awalnya menjalankan script migrasi langsung pada jam sepi:
  `ALTER TABLE users DROP COLUMN full_name, ADD COLUMN first_name VARCHAR(100), ADD COLUMN last_name VARCHAR(100);`  
  Perintah ini membutuhkan kunci `AccessExclusiveLock`. Karena tabel `users` terus-menerus dibaca oleh transaksi transfer yang sedang aktif, perintah `ALTER TABLE` tertahan di antrean kunci (*lock queue*). Di PostgreSQL, `AccessExclusiveLock` yang mengantre akan memblokir seluruh query baca dan tulis berikutnya yang datang di belakangnya. Dalam 20 detik, seluruh *pool* koneksi database (500 koneksi) habis terpakai, API Gateway memuntahkan error `504 Gateway Timeout`, dan layanan perbankan lumpuh total selama 25 menit.

- **Implementasi Solusi Industri (The Architectural Solution)**:
  1. **Fase 1 (Expand - Non-Blocking DDL)**: Dibuat migrasi yang hanya menambahkan kolom baru sebagai nullable tanpa nilai default:
     ```sql
     SET lock_timeout = '2s';
     ALTER TABLE users ADD COLUMN first_name VARCHAR(100), ADD COLUMN last_name VARCHAR(100);
     ```
     Dengan `lock_timeout = 2s`, operasi DDL hanya memegang lock selama 1.5ms tanpa risiko menyandera antrean query pengguna.
  2. **Fase 2 (Dual-Writing di Kode Go)**: Backend dideploy dengan kode yang secara paralel menulis ke kolom lama dan kolom baru saat ada pengguna mendaftar atau mengedit profil.
  3. **Fase 3 (Throttled Background Backfill Worker)**: Dijalankan proses worker mandiri di Go yang memigrasikan data historis 28 juta baris secara bertahap dalam batch kecil (1.000 baris per batch dengan jeda `time.Sleep(100ms)`). Worker terus memantau metrik `pg_stat_replication_lag` agar proses backfill tidak membuat database replica tertinggal.
  4. **Fase 4 (Contract & Cleanup)**: Setelah metrik `COUNT(*) WHERE first_name IS NULL` mencapai angka 0, kode backend Go dialihkan murni membaca kolom baru. Kolom lama `full_name` kemudian dihapus secara aman di migrasi penutup.

- **Dampak & Hasil Sistem (The Engineering Outcome)**:
  - Seluruh proses migrasi 28 juta baris diselesaikan dengan downtime tepat **0 detik** (ketersediaan SLA 100%).
  - Nol transaksi transfer atau login yang gagal akibat *table locking*.
  - Menghilangkan risiko kegagalan rilis: pada Fase 2 dan 3, tim memiliki jaminan keamanan untuk melakukan rollback instan jika ditemukan bug tanpa kehilangan data baru.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Menambah Kolom dengan Default Value di PostgreSQL Versi Lama**: Di PostgreSQL < 11, menambahkan kolom `ADD COLUMN status INT DEFAULT 1` akan menulis ulang seluruh tabel secara fisik di disk (_table rewrite_) dan mengunci tabel berjam-jam!  
   _Solusi_: Di versi lama, tambahkan kolom sebagai NULL dulu, jalankan backfill bertahap, baru pasang default constraint.
2. **Backfill Tanpa Jeda (_Unthrottled Backfill Storm_)**: Menjalankan `UPDATE users SET ...` pada 40 juta baris sekaligus dalam 1 query raksasa. Database akan langsung kehabisan memori, lock tabel membengkak, dan replikasi database tertinggal puluhan gigabyte (_massive replication lag_).  
   _Solusi_: Selalu gunakan batching (500 - 5.000 baris) dengan jeda waktu `time.Sleep`.
3. **Menghapus Kolom Sebelum Seluruh Node Lama Mati**: Menjalankan Fase 4 (Drop Column) saat masih ada container pod lama yang sedang dalam proses rolling update di Kubernetes.

## 11. Rangkuman Singkat

1. **Prinsip Expand and Contract**: Pisahkan modifikasi skema destruktif menjadi penambahan kolom baru (Expand), penulisan ganda (Dual-write), pembaruan data bertahap (Backfill), dan pembersihan akhir (Contract).
2. **Kendalikan Lock Timeout**: Selalu sertakan `SET lock_timeout = '2s'` pada migrasi DDL produksi untuk mencegah antrean `AccessExclusiveLock` menenggelamkan connection pool database.
3. **Throttled Batch Backfill**: Jangan pernah menjalankan update massal jutaan baris dalam 1 query raksasa; selalu gunakan batching kecil dengan jeda istirahat agar replikasi database tidak tertinggal.
4. **Sinkronisasi Siklus Deployment**: Pastikan seluruh armada container versi lama telah selesai dimatikan (*rolling update complete*) sebelum menjalankan tahap penghapusan kolom lama.
