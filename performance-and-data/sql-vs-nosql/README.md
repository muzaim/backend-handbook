# SQL vs NoSQL

## 1. Skenario Sistem Nyata

Pada sebuah platform super-app pemesanan transportasi dan pembayaran digital, sistem menangani dua jenis beban kerja yang memiliki karakteristik sangat bertolak belakang:

1. **Layanan Dompet & Saldo (*Wallet & Ledger Service*)**:
   - Menangani mutasi saldo, transfer antar bank, dan checkout pesanan.
   - Persyaratan utama: **Integritas data mutlak (ACID)**. Saldo pengirim harus berkurang tepat bersamaan dengan saldo penerima bertambah dalam satu transaksi atomik. Kegagalan data, *phantom read*, atau desinkronisasi saldo tidak boleh terjadi sama sekali.
   - Pilihan arsitektur: **SQL / Relational Database (PostgreSQL)**.

2. **Layanan Pelacakan Posisi Pengemudi (*Driver GPS Location Service*)**:
   - 300.000 mitra pengemudi mengirimkan koordinat latitude dan longitude setiap 2 detik (150.000 write operations per detik).
   - Data koordinat bersifat temporal: posisi 10 detik lalu sudah tidak berharga. Yang dibutuhkan adalah kemampuan menyerap lonjakan penulisan ekstrem (*write-heavy ingestion*) dan pembacaan posisi terakhir secara terdistribusi.
   - Pilihan arsitektur: **NoSQL (Redis In-Memory Key-Value / Apache Cassandra / ScyllaDB)**.

Alih-alih memaksakan satu database untuk semua kebutuhan, arsitektur modern berskala tinggi menerapkan pola **Polyglot Persistence**: menggunakan SQL untuk data transaksional relasional, dan NoSQL untuk data berskala besar dengan skema fleksibel atau kebutuhan write throughput ekstrem.

## 2. Apa Itu dan Buat Apa?

Perbandingan antara SQL (Relational) dan NoSQL (Non-Relational) berpusat pada bagaimana data distrukturkan, disimpan, dan dijamin konsistensinya:

- **SQL (Relational Database Management System - RDBMS)**:
  - Menyimpan data dalam bentuk tabel relasional dengan baris dan kolom yang terstruktur kaku (*Strict Schema*).
  - Mengandalkan bahasa SQL terstandardisasi dengan kemampuan relasi antar tabel melalui `JOIN` dan *Foreign Keys*.
  - Menjamin prinsip **ACID** (*Atomicity, Consistency, Isolation, Durability*).
  - Contoh: PostgreSQL, MySQL, MariaDB, Oracle, CockroachDB.

- **NoSQL (Not Only SQL)**:
  - Dirancang tanpa skema kaku (*Schemaless* / *Flexible Schema*) dan dioptimalkan untuk penskalaan horizontal (*Scale-Out*).
  - Mengorbankan sebagian garansi ACID demi ketersediaan dan performa (*BASE: Basically Available, Soft state, Eventual consistency*).
  - Terdiri dari 4 keluarga utama:
    1. **Document Store**: Menyimpan data sebagai dokumen JSON/BSON (misal MongoDB, Couchbase).
    2. **Key-Value Store**: Pemetaan pasangan kunci-nilai berkecepatan ultra tinggi (misal Redis, AWS DynamoDB).
    3. **Wide-Column Store**: Menyimpan data berdasarkan kolom untuk throughput tulis analitik raksasa (misal Apache Cassandra, ScyllaDB).
    4. **Graph Database**: Menyimpan node dan edge untuk relasi jaringan sosial atau fraud detection (misal Neo4j).

## 3. Kenapa Harus Dipahami di High-Throughput?

Di level arsitektur backend senior, pemilihan jenis database menentukan batasan penskalaan sistem:
1. **Teorema CAP (Consistency, Availability, Partition Tolerance)**: Sistem terdistribusi hanya bisa menjamin maksimal dua dari tiga sifat tersebut. SQL cenderung memilih CA/CP, sedangkan NoSQL sering kali memilih AP/CP untuk menjamin ketersediaan tinggi saat jaringan antar data center terputus.
2. **Bottleneck Skalabilitas Penulisan (Write Scaling)**: Database SQL tradisional sulit di-scale secara horizontal untuk operasi write tanpa kompleksitas sharding manual. NoSQL dirancang sejak awal untuk auto-partitioning dan distributed clustering.
3. **Mencegah Anti-Pattern Polyglot**: Memahami trade-offs mencegah tim membuang waktu memaksakan PostgreSQL menyimpan miliaran data log time-series atau memaksakan MongoDB mengelola ledger perbankan yang rapuh tanpa transaksi multi-dokumen ketat.

## 4. Cara Kerjanya (Perbandingan Arsitektur)

```
+-------------------------------------------------------------------------+
|                  Perbandingan Model Relasional vs Dokumen               |
+-------------------------------------------------------------------------+

 [Model SQL / Relasional: Normalisasi & Relasi Ketat]
 +------------------+            +------------------+
 | users (Tabel)    |            | orders (Tabel)   |
 | id (PK)          |<-----------| user_id (FK)     |
 | name, email      |            | total_amount     |
 +------------------+            +------------------+
                                          |
                                          v
                                 +------------------+
                                 | order_items      |
                                 | order_id (FK)    |
                                 | product_id, qty  |
                                 +------------------+
 (Menggunakan JOIN SQL saat dibaca. Butuh ACID saat checkout).

 -------------------------------------------------------------------------

 [Model NoSQL / Dokumen: Denormalisasi Mandiri (Embedded)]
 +-----------------------------------------------------------------------+
 | Document: orders (Koleksi NoSQL)                                      |
 | {                                                                     |
 |   "_id": "ord_9921",                                                  |
 |   "user": { "id": 101, "name": "Budi", "email": "budi@mail.com" },   |
 |   "items": [                                                          |
 |     { "product_id": "p1", "name": "Kemeja", "qty": 2, "price": 50000}|
 |   ],                                                                  |
 |   "total_amount": 100000,                                             |
 |   "created_at": "2026-11-24T10:00:00Z"                                |
 | }                                                                     |
 +-----------------------------------------------------------------------+
 (Semua data terbaca dalam 1 kali lookup tanpa JOIN multi-tabel).
```

## 5. Cuplikan Logika Inti (Transaksi ACID di SQL vs Upsert Kencang di NoSQL Go)

Berikut perbandingan implementasi kode Go: transaksi perbankan dengan jaminan rollback atomik di SQL vs penulisan cepat tanpa skema kaku di NoSQL (Redis/Mongo):

```go
package main

import (
	"context"
	"database/sql"
	"encoding/json"
	"errors"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

// 1. PENDEKATAN SQL (PostgreSQL): Transaksi Transfer Uang dengan Garansi ACID
func TransferBalanceSQL(ctx context.Context, db *sql.DB, fromUserID, toUserID int64, amount float64) error {
	// Memulai blok transaksi terisolasi
	tx, err := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelReadCommitted})
	if err != nil {
		return err
	}
	defer tx.Rollback() // Rollback otomatis jika terjadi error atau panic di tengah jalan

	// Kurangi saldo pengirim (dengan validasi saldo cukup)
	res, err := tx.ExecContext(ctx, "UPDATE wallets SET balance = balance - $1 WHERE user_id = $2 AND balance >= $1", amount, fromUserID)
	if err != nil {
		return err
	}
	rowsAffected, _ := res.RowsAffected()
	if rowsAffected == 0 {
		return errors.New("saldo tidak mencukupi atau user tidak ditemukan")
	}

	// Tambahkan saldo penerima
	if _, err := tx.ExecContext(ctx, "UPDATE wallets SET balance = balance + $1 WHERE user_id = $2", amount, toUserID); err != nil {
		return err
	}

	// Commit atomik kedua perubahan saldo
	return tx.Commit()
}

// 2. PENDEKATAN NoSQL (Redis): Ingestion Koordinat GPS Real-Time Berkecepatan Tinggi
type DriverLocation struct {
	DriverID  int64     `json:"driver_id"`
	Latitude  float64   `json:"latitude"`
	Longitude float64   `json:"longitude"`
	UpdatedAt time.Time `json:"updated_at"`
}

func UpdateDriverLocationNoSQL(ctx context.Context, rdb *redis.Client, loc DriverLocation) error {
	key := fmt.Sprintf("driver:loc:%d", loc.DriverID)
	data, err := json.Marshal(loc)
	if err != nil {
		return err
	}

	// Tulis koordinat ke RAM dengan TTL 60 detik (kecepatan < 0.5 ms per write)
	return rdb.Set(ctx, key, data, 60*time.Second).Err()
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### SQL (Relational)
- **Kelebihan**: Integritas data terjamin via ACID, skema kaku mencegah anomali data kotor, query analitik relasional fleksibel dengan `JOIN`, mature tooling & ekosistem puluhan tahun.
- **Kekurangan**: Sulit di-scale horizontal untuk write throughput masif, migrasi skema tabel raksasa membutuhkan waktu dan perencanaan matang, performa menurun drastis jika query melibatkan terlalu banyak JOIN kompleks.

### NoSQL (Non-Relational)
- **Kelebihan**: Penskalaan horizontal bawaan (*Auto-sharding*), throughput tulis sangat tinggi, struktur data fleksibel dan mudah beradaptasi dengan perubahan skema tanpa migrasi DDL yang lama.
- **Kekurangan**: Tidak ada dukungan relasi bawaan (tidak ada native foreign keys/JOIN efisien), konsistensi data sering kali *eventual* (risiko data stale), rawan duplikasi data (*denormalization storage overhead*).

## 7. Kapan Harus Pakai SQL vs NoSQL

### Pilih SQL Jika:
- Aplikasi Anda adalah sistem finansial, pembayaran, e-commerce checkout, atau ERP/CRM perusahaan yang menuntut akurasi 100% dan integritas referensial.
- Pola query data belum pasti dan membutuhkan kemampuan agregasi data kompleks lintas domain bisnis yang saling berelasi.
- Skala data masih dalam batas ratusan gigabyte hingga beberapa terabyte di mana satu server PostgreSQL modern masih sangat mumpuni.

### Pilih NoSQL Jika:
- Volume penulisan data sangat masif dan konstan (ratusan ribu write per detik: IoT sensor telemetry, log event clickstream, geolokasi real-time).
- Data memiliki struktur dinamis tanpa bentuk seragam (misal: katalog produk e-commerce dengan ratusan variasi atribut spesifikasi).
- Aplikasi membutuhkan penyimpanan key-value dengan latensi sub-milidetik (session cache, shopping cart sementara).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

- **SQL - `database_deadlocks_total` (Counter)**: Jumlah deadlock transaksi SQL akibat perebutan baris secara simultan.
- **SQL - `active_transactions_duration_seconds` (Gauge)**: Memantau transaksi mengambang (*long-running transactions*) yang menahan tabel.
- **NoSQL - `replication_lag_seconds` (Gauge)**: Jarak waktu keterlambatan sinkronisasi data antar node replica NoSQL.
- **NoSQL - `read_repair_requests_total` (Counter)**: Jumlah operasi perbaikan data yang tidak konsisten saat dibaca di cluster Cassandra/DynamoDB.

## 9. Studi Kasus Industri Nyata

- **Amazon**: Merancang basis data NoSQL internal legendaris bernama **DynamoDB** setelah menyadari bahwa 70% query database layanan ritel mereka sebenarnya hanya berupa pencarian key-value sederhana yang tidak memerlukan relational JOIN RDBMS yang mahal dan lambat.
- **Uber**: Mengombinasikan PostgreSQL, MySQL (Schemaless), dan Redis/Cassandra. Data transaksi keuangan perjalanan tetap berada di RDBMS berintegritas tinggi, sedangkan riwayat koordinat penjemputan dan telemetry GPS dialirkan ke datastore NoSQL terdistribusi.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Ilusi ACID pada NoSQL**: Mengira bahwa NoSQL modern bisa menangani transaksi multi-dokumen sekuat RDBMS. Meskipun beberapa NoSQL menambahkan fitur transaksi, biaya latensi jaringan dan risiko *write conflicts* pada cluster terdistribusi sangat tinggi.
2. **PostgreSQL JSONB Menggantikan NoSQL**: Sering kali developer beralih ke MongoDB hanya demi menyimpan JSON dinamis. Padahal PostgreSQL memiliki tipe data `JSONB` yang dilengkapi indeks GIN berkecepatan tinggi, memungkinkan kombinasi tabel relasional dan data dokumen tanpa perlu mengoperasikan database NoSQL terpisah.
3. **Application-Level JOIN Nightmare di NoSQL**: Melakukan denormalisasi berlebihan di NoSQL hingga saat butuh laporan agregasi, backend terpaksa melakukan puluhan query HTTP terpisah dan menggabungkannya secara manual di memori kode Go (*in-memory stitching*).

## 11. Rangkuman Singkat

SQL unggul dalam integritas data transaksional (ACID) dan relasi query yang presisi, sedangkan NoSQL unggul dalam elastisitas penskalaan horizontal dan throughput penulisan bervolume ekstrem. Keputusan terbaik dalam arsitektur backend modern bukanlah memilih salah satu secara eksklusif, melainkan mengombinasikan keduanya (*Polyglot Persistence*) sesuai kebutuhan spesifik domain bisnis.
