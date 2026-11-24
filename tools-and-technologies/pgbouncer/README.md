# PgBouncer (PostgreSQL Connection Pooler)

## 1. Skenario Sistem Nyata

Pada sebuah platform microservices e-commerce yang berjalan di atas cluster Kubernetes, Horizontal Pod Autoscaler (HPA) secara otomatis menaikkan jumlah replica backend aplikasi Go dari 15 pod menjadi 120 pod saat terjadi lonjakan trafik flash sale.

Setiap pod aplikasi Go dikonfigurasi dengan standar connection pool internal:

```go
db.SetMaxOpenConns(25)
```

Jika 120 pod tersebut terhubung langsung ke satu basis data PostgreSQL utama:

-   Total koneksi konkuren yang mencoba dibuka ke PostgreSQL mencapai:  
    $120 \text{ pod} \times 25 \text{ koneksi} = 3.000 \text{ koneksi TCP}$!
-   **Masalah Arsitektur PostgreSQL**: PostgreSQL menggunakan model arsitektur berbasis proses (_process-based model_), bukan thread. Setiap 1 koneksi klien yang masuk akan membuat satu proses baru di Linux (`postgres: user db [idle]`) yang mengonsumsi 5 hingga 10 MB RAM memori.
-   Membuka 3.000 koneksi fisik membutuhkan sekitar 20 hingga 30 GB RAM hanya untuk menampung koneksi yang sebagian besar sedang menganggur (_idle_). CPU server database melonjak ke 100% hanya karena sibuk melakukan _context switching_ antar ribuan proses Linux.
-   Akhirnya database meledak dan menolak semua request baru dengan pesan error:  
    `FATAL: sorry, too many clients already`. Seluruh sistem pembayaran dan pesanan e-commerce mati total.

**Solusi dengan PgBouncer:**
Tim infrastruktur memasang **PgBouncer** sebagai reverse proxy connection pooler ringan di depan PostgreSQL. Seluruh 120 pod aplikasi Go (3.000 koneksi klien) sekarang terhubung ke PgBouncer. Dengan mode **Transaction Pooling**, PgBouncer hanya meminjamkan koneksi fisik ke PostgreSQL selama sebuah transaksi SQL sedang dieksekusi (biasanya hanya 1 hingga 5 milidetik). Begitu transaksi selesai, koneksi fisik tersebut langsung dialihkan untuk melayani request dari pod lain.

Hasilnya: **3.000 koneksi aplikasi Go cukup dilayani oleh 60 koneksi fisik ke PostgreSQL!** Penggunaan CPU database turun drastis dari 100% menjadi 25%, dan sistem tidak pernah lagi mengalami _connection exhaustion_.

## 2. Apa Itu dan Buat Apa?

PgBouncer adalah perangkat lunak _lightweight connection pooler_ open-source berkinerja tinggi yang dirancang khusus untuk basis data PostgreSQL.

Karakteristik utama:

-   **Arsitektur Event-Loop Ringan**: Dibangun menggunakan pustaka C dan `libevent`. PgBouncer mampu mengelola puluhan ribu koneksi klien yang menganggur dengan konsumsi memori hanya beberapa megabyte RAM.
-   **Tiga Mode Pooling**:
    1. **Session Pooling**: Mengalokasikan 1 koneksi server selama sesi koneksi klien aktif. Koneksi dilepas saat aplikasi memutus koneksi (paling kompatibel, tapi paling boros).
    2. **Transaction Pooling (Paling Populer di High-Throughput)**: Koneksi server hanya dialokasikan selama blok transaksi SQL berlangsung (`BEGIN ... COMMIT`). Sangat efisien untuk aplikasi stateless web/API.
    3. **Statement Pooling**: Koneksi server dilepas seketika setelah setiap satu query SQL selesai dieksekusi (tidak mendukung transaksi multi-pernyataan).

## 3. Kenapa Harus Pakai Ini ?

Dalam arsitektur backend modern berbasis microservices dan container autoscaling:

1. **Penyelamat Database dari Connection Starvation**: Mencegah lonjakan pod aplikasi melumpuhkan basis data PostgreSQL.
2. **Memangkas Latensi Handshake TCP & TLS**: Aplikasi Go membuka koneksi lokal yang cepat ke PgBouncer, sementara koneksi jarak jauh ke server database utama tetap terbuka secara persisten.
3. **Memaksimalkan Throughput CPU PostgreSQL**: PostgreSQL bekerja paling efisien saat jumlah koneksi aktif berada di kisaran $2 \times \text{Core CPU}$. Mengalirkan 50 koneksi aktif jauh lebih kencang dibanding memaksa database menangani 2.000 koneksi simultan.

## 4. Cara Kerjanya (Diagram Alur Transaction Pooling)

```
 [100+ Pod Aplikasi Go di Kubernetes]
 (Masing-masing membuka 25-50 koneksi klien)
 +--------+  +--------+         +--------+
 | Pod #1 |  | Pod #2 |  . . .  |Pod #100|
 +--------+  +--------+         +--------+
      \          |          /
       \         |         /
        v        v        v
  [3,000 Client Virtual Connections]
                 |
                 v
 +-----------------------------------------------+
 |                   PgBouncer                   |
 |        (Lightweight libevent proxy)           |
 |                                               |
 |  * 2,940 Client Conns: IDLE (Menunggu Query)  |
 |  * 60 Client Conns: ACTIVE (Sedang Transaksi) |
 +-----------------------------------------------+
                 |
                 | Transaction Pooling (Pinjam Koneksi Hanya Saat SQL Aktif)
                 v
   [60 Physical Server Connections]
                 |
                 v
 +-----------------------------------------------+
 |               PostgreSQL Server               |
 |         (Dedicated Master Instance)           |
 |                                               |
 |  * Hanya 60 Proses OS Linux Aktif             |
 |  * Zero Context Switching Thrashing           |
 |  * CPU Sehat: 20-30%, RAM Sangat Lega         |
 +-----------------------------------------------+
```

## 5. Cuplikan Konfigurasi & Integrasi Go

### Konfigurasi Produksi `pgbouncer.ini` (Transaction Pooling)

```ini
[databases]
; Memetakan database aplikasi ke host PostgreSQL utama
mydb = host=10.0.1.50 port=5432 dbname=mydb_production auth_user=postgres

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

; MODE POOLING UTAMA: transaction (koneksi server dilepas setelah commit)
pool_mode = transaction

; Batas kapasitas koneksi
max_client_conn = 10000     ; Mampu menampung hingga 10.000 koneksi dari pod Go
default_pool_size = 50      ; Maksimal 50 koneksi fisik asli ke PostgreSQL per DB/user
min_pool_size = 10          ; Pertahankan 10 koneksi selalu terbuka (hangat)
reserve_pool_size = 5       ; Tambahan darurat saat antrean menumpuk
max_db_connections = 100    ; Batas mutlak koneksi ke PostgreSQL
```

### Cuplikan Driver Go (`pgxpool` / `database/sql`) yang Aman untuk PgBouncer

Dalam mode _Transaction Pooling_, penggunaan _named prepared statements_ di driver Go standar bisa menimbulkan error `prepared statement does not exist` karena query berikutnya bisa dijalankan di koneksi fisik PostgreSQL yang berbeda.

Gunakan protokol `simple_protocol` atau _statement cache disabled_:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

func ConnectViaPgBouncer() (*pgxpool.Pool, error) {
	// Hubungkan ke port 6432 milik PgBouncer
	connString := "postgres://app_user:secret@localhost:6432/mydb?sslmode=disable"

	config, err := pgxpool.ParseConfig(connString)
	if err != nil {
		return nil, fmt.Errorf("gagal parse config: %w", err)
	}

	// PENTING UNTUK PGBOUNCER TRANSACTION POOLING:
	// Nonaktifkan automatic statement caching agar tidak bentrok antar koneksi fisik server
	config.ConnConfig.DefaultQueryExecMode = 1 // pgx.QueryExecModeSimpleProtocol

	// Atur pool lokal di dalam pod aplikasi
	config.MaxConns = 30
	config.MinConns = 5
	config.MaxConnLifetime = 30 * time.Minute
	config.MaxConnIdleTime = 5 * time.Minute

	pool, err := pgxpool.NewWithConfig(context.Background(), config)
	if err != nil {
		return nil, fmt.Errorf("gagal inisialisasi pool: %w", err)
	}

	log.Println("[Database] Berhasil terhubung ke PostgreSQL melalui PgBouncer (Port 6432)")
	return pool, nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Efisiensi Memori Ekstrem**: Menghemat gigabyte RAM pada server PostgreSQL dengan menekan jumlah proses OS ke level minimal.
-   **Dukungan Skalabilitas Ribuan Klien**: Memungkinkan ribuan container microservices terhubung secara simultan tanpa takut error `too many clients`.
-   **Ringan & Cepat**: Berjalan dengan konsumsi CPU dan latensi proxy di bawah 0.2 milidetik.

### Kekurangan

-   **Keterbatasan Fitur pada Transaction Pooling**: Fitur-fitur yang terikat pada tingkat sesi koneksi (_Session State_) tidak dapat digunakan, seperti:
    -   `LISTEN / NOTIFY` (membutuhkan sesi persisten).
    -   Variabel sesi dinamis (`SET TIME ZONE ...` atau `SET work_mem ...`).
    -   Tabel sementara (_Temporary Tables_) berbasis sesi.
    -   _Named Prepared Statements_ (harus dinonaktifkan di driver aplikasi).
-   **Titik Kegagalan Tambahan (_Extra Hop_)**: Menambah satu komponen jaringan yang harus dipantau dan dijamin ketersediaannya (_High Availability_).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Arsitektur berbasis Kubernetes / Serverless di mana puluhan hingga ratusan pod container dapat melakukan scaling out dan membuka ribuan koneksi bersamaan ke PostgreSQL.
-   Server PostgreSQL yang sering mengalami lonjakan CPU tinggi akibat _context switching_ ribuan koneksi idle.
-   Aplikasi web stateless berbasis HTTP REST API atau gRPC.

### Kapan Jangan Pakai

-   Aplikasi yang sangat bergantung pada fitur `LISTEN / NOTIFY` PostgreSQL untuk event real-time (solusi: buat koneksi khusus bypass langsung ke port 5432 untuk listener tersebut).
-   Beban kerja komputasi analitik batch tunggal di mana hanya ada 2 atau 3 aplikasi yang terhubung secara konstan.
-   Database selain PostgreSQL (PgBouncer eksklusif hanya untuk protokol PostgreSQL).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard via `SHOW POOLS`)

PgBouncer menyediakan antarmuka virtual console administratif (`psql -p 6432 -U pgbouncer pgbouncer`):

-   **`cl_active` vs `cl_waiting` (Gauge)**: `cl_active` adalah jumlah klien yang sedang aktif mengeksekusi query. Jika `cl_waiting` mulai bernilai lebih dari 0, berarti klien sedang mengantre menunggu jatah koneksi server!
-   **`sv_active` vs `sv_idle` (Gauge)**: `sv_active` adalah jumlah koneksi fisik asli ke PostgreSQL yang sedang bekerja. `sv_idle` adalah koneksi siap pakai yang sedang menganggur.
-   **`maxwait` (Gauge - Detik)**: Berapa lama waktu terlama yang dihabiskan oleh klien di dalam antrean untuk mendapatkan koneksi server. Jika > 1 detik, ukuran pool harus dinaikkan.

## 9. Studi Kasus Industri Nyata

-   **GitLab**: Mengoperasikan armada cluster PostgreSQL raksasa untuk melayani jutaan repositori dan pipeline CI/CD. GitLab menempatkan PgBouncer di setiap node database untuk menahan jutaan koneksi konkuren dari worker background Sidekiq dan aplikasi Rails.
-   **Supabase & Heroku Postgres**: Platform Backend-as-a-Service terkemuka ini menyertakan PgBouncer secara default untuk setiap instance database penggunanya agar fungsi-fungsi _Serverless Lambda_ yang menyala ribuan kali per detik tidak menenggelamkan PostgreSQL.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Jebakan `prepared statement "..." already exists`**: Driver Go atau Java ORM mencoba membuat prepared statement dengan nama tertentu di mode Transaction Pooling. Query kedua dikirim ke koneksi fisik lain yang belum memiliki statement tersebut, memicu error fatal.  
   _Solusi_: Gunakan _Simple Protocol_ atau anonymized prepared statements pada konfigurasi driver Go/Java.
2. **Koneksi `cl_waiting` Macet Karena Transaksi Tergantung**: Klien membuka transaksi (`BEGIN`) lalu melakukan panggilan HTTP eksternal yang lambat sebelum melakukan `COMMIT`. Koneksi fisik PgBouncer akan tertahan (_held hostage_), membuat klien lain mengantre panjang.  
   _Solusi_: Jangan pernah memanggil API luar atau I/O lambat di dalam blok transaksi database.
3. **Mengabaikan File `userlist.txt`**: Lupa menyinkronkan password user database baru ke file kredensial PgBouncer saat ada penambahan user di PostgreSQL, mengakibatkan error `Auth failed`.

## 11. Rangkuman Singkat

PgBouncer adalah tameng pelindung dan connection pooler ringan yang menyelamatkan PostgreSQL dari kehabisan koneksi (_connection exhaustion_). Melalui mode Transaction Pooling, PgBouncer memungkinkan ribuan replica microservices terhubung secara stabil ke puluhan koneksi fisik database, menjaga performa CPU dan memori PostgreSQL tetap optimal di skala trafik tertinggi.
