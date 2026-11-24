# PostgreSQL

## 1. Skenario Sistem Nyata

Pada backend perbankan digital (_core banking ledger_) atau platform pembayaran internasional seperti Stripe, setiap detik ribuan transaksi saldo menuntut jaminan konsistensi **ACID mutlak**. Setiap detik ada ribuan transaksi mutasi saldo antar-rekening, pembayaran QRIS, dan pencatatan audit keuangan yang wajib bergaransi **ACID mutlak** (uang tidak boleh hilang atau berlipat ganda sedetik pun).

Jika kamu menggunakan database NoSQL tanpa jaminan transaksi kaku, transaksi transfer uang antar dua rekening rentan mengalami inkonsistensi saat server mendadak restart. Namun jika kamu menggunakan database relasional tanpa pemahaman arsitektur throughput tinggi:

-   Setiap kali ada transaksi tulis (`UPDATE balances`), row database terkunci.
-   Saat ada proses lain yang ingin membaca data saldo (`SELECT balance`), pembacaan tersebut bisa terblokir oleh transaksi tulis yang sedang berlangsung (_read-write lock contention_).
-   Jika ada 500 koneksi aplikasi backend yang langsung terhubung ke database tanpa perantara, proses OS Postgres akan memakan seluruh RAM server, memicu _out of memory_, dan database tumbang di tengah lonjakan transaksi.

**Solusi dengan PostgreSQL:**
PostgreSQL dirancang dengan arsitektur **MVCC (Multi-Version Concurrency Control)**: _"Readers never block writers, and writers never block readers"_. Saat transaksi mutasi sedang berjalan, query pembacaan tetap dapat membaca versi snapshot data yang valid secara instan tanpa terblokir sama sekali! Dikombinasikan dengan arsitektur Write-Ahead Logging (WAL), streaming replication, dan connection pooler (seperti PgBouncer), PostgreSQL sanggup melayani **puluhan ribu transaksi per detik** dengan integritas data finansial tingkat tinggi.

## 2. Apa Itu dan Buat Apa?

**PostgreSQL (sering disebut Postgres)** adalah sistem manajemen database relasional objek (_Object-Relational Database Management System_ / ORDBMS) sumber terbuka tingkat enterprise yang terkenal dengan keandalan ekstrem, integritas data, kepatuhan standar SQL, dan fitur ekstensibilitasnya yang kaya.

### Fitur Utama di High-Throughput:

1. **MVCC (Multi-Version Concurrency Control)**: Mengelola konkurensi tinggi dengan menyimpan beberapa versi baris (_tuple_) sekaligus, sehingga query `SELECT` tidak pernah mengunci query `UPDATE/INSERT` dan sebaliknya.
2. **Write-Ahead Logging (WAL)**: Semua mutasi data ditulis secara sekuensial ke log disk WAL terlebih dahulu sebelum ditulis ke tabel data utama di disk. Ini memastikan integritas data (_durability_) dan pemulihan instan saat crash.
3. **Ekstensibilitas Index Canggih**: Mendukung B-Tree, GIN (Generalized Inverted Index untuk JSONB dan teks), GiST, BRIN (Block Range Index untuk time-series data miliaran baris), dan Hash index.
4. **Dukungan JSONB Native**: Performa penyimpanan dan query dokumen semi-terstruktur JSON dengan kecepatan indexing yang menyaingi database NoSQL murni seperti MongoDB.

## 3. Kenapa Harus Pakai Ini ?

### 1. Konkurensi Tanpa Lock Pembacaan (MVCC)

Di sistem high-throughput yang didominasi oleh perbandingan 80% Read dan 20% Write, lock contention adalah pembunuh performa. Di Postgres, transaksi penulisan tidak menimpa baris lama, melainkan membuat tuple versi baru dengan header transaksi (`xmin` dan `xmax`). Query pembacaan membaca versi tuple yang relevan dengan snapshot transaksinya, menghilangkan locking overhead antar-pembaca dan penulis.

### 2. BRIN Indexing untuk Tabel Miliaran Baris

Untuk tabel audit log atau metrik time-series yang memiliki ratusan juta baris, indeks B-Tree standar bisa memakan puluhan Gigabyte RAM. Postgres menyediakan **BRIN (Block Range Index)** yang hanya mencatat nilai minimum dan maksimum per blok disk. Ukuran indeks BRIN hanya **sekitar 1% dari B-Tree**, memungkinkan pencarian data time-series cepat dengan konsumsi RAM yang sangat minim.

### 3. Replikasi Asinkron & Sinkron (_Read Replicas_)

Postgres mendukung _Physical Streaming Replication_ berbasis WAL stream. Kamu dapat mengarahkan semua query penulisan ke Primary Node, lalu menyebarkan ribuan query pembacaan ke beberapa Read Replica Node secara transparan.

## 4. Cara Kerjanya (Arsitektur MVCC dan WAL di PostgreSQL)

```
[ Transaksi WRITE: UPDATE balance = 200 ]       [ Transaksi READ: SELECT balance ]
                    |                                           |
                    v                                           v
       +-------------------------+                 +-------------------------+
       | Tulis Log Sekuensial ke |                 | Baca Snapshot Transaksi |
       | Disk: Write-Ahead (WAL) |                 | tanpa menunggu lock!    |
       +-------------------------+                 +-------------------------+
                    |                                           |
                    v                                           |
       +---------------------------------------------+          |
       | PostgreSQL Shared Buffer Pool (RAM)         |          |
       |                                             |          |
       | Row Tuple Versi 1: [balance: 100, xmax: T2] |<---------+ (Read Snapshot Lama)
       | Row Tuple Versi 2: [balance: 200, xmin: T2] |  (Tuple Baru yang Ditulis)
       +---------------------------------------------+
```

## 5. Cuplikan Logika Inti

### 1. Transaksi Transfer Uang Atomik dengan Row-Level Lock

```sql
BEGIN;

-- Mengunci baris rekening sumber agar saldo tidak ditarik ganda secara bersamaan
SELECT balance
FROM accounts
WHERE account_id = 'ACC_001'
FOR UPDATE;

-- Validasi saldo di aplikasi (misal balance >= 50.000)
UPDATE accounts
SET balance = balance - 50000
WHERE account_id = 'ACC_001';

UPDATE accounts
SET balance = balance + 50000
WHERE account_id = 'ACC_002';

-- Catat log audit transaksi
INSERT INTO transaction_logs (from_acc, to_acc, amount, created_at)
VALUES ('ACC_001', 'ACC_002', 50000, NOW());

COMMIT;
```

### 2. Optimasi Index JSONB di PostgreSQL

```sql
-- Membuat tabel event log fleksibel dengan kolom JSONB
CREATE TABLE user_activity_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Membuat GIN Index agar query pencarian properti JSON bersarang menjadi secepat B-Tree
CREATE INDEX idx_user_activity_payload ON user_activity_logs USING GIN (payload jsonb_path_ops);

-- Query berkecepatan tinggi memanfaatkan index GIN:
SELECT * FROM user_activity_logs
WHERE payload @> '{"action": "PURCHASE", "status": "COMPLETED"}';
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Integritas ACID Sempurna**: Standar baku emas industri untuk keandalan data transaksi keuangan dan data relasional penting.
-   **Dukungan Index Sangat Lengkap**: B-Tree, GIN, GiST, SP-GiST, BRIN, dan Hash index.
-   **Ekosistem Ekstensi Luar Biasa**: Didukung ekstensi seperti `TimescaleDB` (time-series), `PostGIS` (geospasial), dan `pgvector` (vector database untuk AI).

### Kekurangan

-   **Tabel Bloat & Kebutuhan VACUUM**: Karena MVCC tidak menimpa baris lama, baris lama yang sudah usang (_dead tuples_) tertinggal di disk dan memakan tempat (_bloat_). Memerlukan proses latar belakang `AUTOVACUUM` yang memakan disk I/O.
-   **Model Koneksi Process-Based (Fork OS)**: Berbeda dengan MySQL atau Redis yang multithreaded, setiap koneksi client baru di Postgres membuat satu proses OS baru (`fork()`). Menghubungkan 1.000 client langsung dapat melumpuhkan RAM server jika tidak menggunakan connection pooler.
-   **Skalabilitas Write Terbatas**: Secara default, PostgreSQL berskala secara _vertical_ untuk write. Sharding penulisan lintas node membutuhkan arsitektur terdistribusi terpisah (seperti Citus / YugabyteDB).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Pakai PostgreSQL

-   Sistem dengan kebutuhan integritas transaksi finansial tinggi (billing, ledger, inventory, e-commerce checkout).
-   Kebutuhan schema data relasional kompleks dengan foreign keys, joins, dan constraints.
-   Aplikasi yang membutuhkan kombinasi data relasional SQL dan data dokumen JSONB fleksibel dalam satu database terpadu.

### Kapan Jangan Pakai PostgreSQL Sebagai Solusi Utama

-   Aliran data ingestion streaming dengan volume ratusan ribu write mentah per detik tanpa kebutuhan relasi (lebih cocok Kafka / Cassandra / ClickHouse).
-   Data cache murni yang membutuhkan latensi sub-milidetik (gunakan Redis).
-   Pencarian teks berskala multi-terabyte (gunakan Elasticsearch / OpenSearch).

## 8. Metrik Kunci & Observabilitas

Metrik wajib di Grafana / Datadog untuk PostgreSQL:

-   **`pg_stat_activity` (Active Connections vs Idle)**: Jumlah koneksi yang sedang aktif menjalankan query vs yang idle.
-   **`cache_hit_ratio` (Buffer Pool Hit)**: Persentase pembacaan data yang terlayani dari memori Shared Buffer RAM tanpa menyentuh disk (target: > 99%).
-   **`dead_tuple_count` & `autovacuum_activity`**: Jumlah baris mati di tabel; lonjakan dead tuples menandakan autovacuum terlambat membersihkan disk bloat.
-   **`replication_lag_bytes` / `lag_seconds`**: Jarak ketertinggalan data replica dari node primary.
-   **`pg_stat_statements_mean_exec_time`**: Rata-rata waktu eksekusi query untuk mendeteksi _slow queries_.

## 9. Studi Kasus Nyata di Industri

-   **Stripe Financial Infrastructure**: Stripe mengandalkan PostgreSQL sebagai sumber kebenaran data (_source of truth_) untuk memproses ratusan miliar dolar volume pembayaran global per tahun, memanfaatkan transaksi ACID ketat dan replikasi multi-region terisolasi.
-   **Instagram Early Architecture**: Instagram membangun seluruh sistem backend awalnya di atas kluster PostgreSQL yang di-shard secara horizontal berdasarkan `user_id` untuk menangani ratusan juta foto dan interaksi sosial.

## 10. Fakta Menarik & Gotchas Production

-   **The Connection Exhaustion Trap (Wajib Menggunakan PgBouncer)**:  
    Setiap koneksi ke Postgres mengalokasikan sekitar **5 MB hingga 10 MB memori RAM** untuk proses backend di Linux. Jika kamu mendeploy 50 pod microservices dan masing-masing membuka connection pool 30 koneksi, totalnya menjadi **1.500 koneksi langsung ke Postgres**! RAM database akan habis seketika, dan CPU akan sibuk melakukan context switching antar-proses OS.  
    _Aturan Wajib:_ Jangan pernah biarkan aplikasi terhubung langsung ke Postgres di production. Selalu pasang **PgBouncer** di depan Postgres untuk melakukan _Connection Pooling (Transaction Pooling)_, sehingga 5.000 koneksi aplikasi dapat dilayani secara efisien oleh hanya 50-100 koneksi fisik database!
-   **The Transaction ID Wraparound Disaster**:  
    PostgreSQL menggunakan integer 32-bit untuk nomor identitas transaksi (`xmin`). Angka ini memiliki batas maksimal sekitar 4 miliar transaksi (`2^32`). Jika sebuah database aktif sangat sibuk dan proses vacuum tidak pernah membersihkan counter transaksi lama, Postgres akan mencapai batas _wraparound_ dan **secara paksa mengunci database ke mode READ-ONLY** untuk mencegah rusaknya data masa lalu!  
    _Solusi Wajib:_ Pastikan daemon `autovacuum` berjalan sehat dan pantau metrik `datfrozenxid` di monitoring.
