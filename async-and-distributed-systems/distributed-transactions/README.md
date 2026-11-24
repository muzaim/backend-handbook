# Distributed Transactions

## 1. Skenario Sistem Nyata

Pada sistem perbankan digital atau dompet elektronik (_e-wallet_), transaksi transfer dana menuntut mutasi saldo dari rekening Nasabah A ke rekening Nasabah B yang berada pada database terpisah.

Pada arsitektur monolitik dengan satu database tunggal, operasi ini sangat mudah karena dilindungi transaksi ACID lokal:

```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 5000000 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 5000000 WHERE id = 'B';
COMMIT;
```

Jika server mati sebelum commit, database engine otomatis membatalkan (_rollback_) kedua baris secara atomik. Uang dijamin tidak pernah hilang.

Namun, pada sistem skala besar modern dengan arsitektur microservices atau database sharding:

-   Rekening Nasabah A berada di **`Customer Account DB` (PostgreSQL di Shard 1)**.
-   Rekening Nasabah B berada di **`Merchant / Partner Ledger DB` (MySQL di Shard 2)**.
-   Keduanya adalah mesin database fisik terpisah yang berada di server berbeda dan hanya terhubung melalui jaringan jaringan TCP/IP.

Tantangan Kritis:
Jika Shard 1 sukses memotong Rp 5.000.000 dari Nasabah A, tetapi koneksi jaringan ke Shard 2 tiba-tiba putus atau server Shard 2 kehabisan memori tepat sebelum saldo Nasabah B bertambah:

-   Saldo Nasabah A berkurang, tetapi saldo Nasabah B tidak pernah masuk.
-   Transaksi SQL lokal tidak bisa melakukan rollback ke database di server lain yang berbeda! Uang nasabah lenyap di tengah jalan.

Menjamin konsistensi data yang atomik (_All-or-Nothing_) lintas database dan server terpisah jaringan inilah masalah fundamental yang diselesaikan oleh **Distributed Transactions**.

## 2. Apa Itu dan Buat Apa?

Distributed Transaction (Transaksi Terdistribusi) adalah rangkaian operasi bisnis yang melibatkan dua atau lebih penyimpanan data terpisah (misalnya: dua database PostgreSQL berbeda, atau satu database MySQL dan satu broker Apache Kafka) yang harus diselesaikan dengan jaminan konsistensi data yang utuh.

Dalam database tunggal (_monolith_), kita menikmati garansi **ACID**:

-   **Atomicity**: Semua operasi sukses, atau semua dibatalkan (_All-or-Nothing_ via `BEGIN ... COMMIT / ROLLBACK`).
-   **Consistency**: Aturan integritas data selalu valid.
-   **Isolation**: Transaksi konkuren tidak saling mengganggu.
-   **Durability**: Data yang di-commit tersimpan permanen di disk.

Namun, ketika arsitektur dipecah menjadi **Microservices (Database-per-Service)**, transaksi SQL `BEGIN ... COMMIT` lokal tidak bisa lagi menjangkau database service lain.

## 3. Kenapa Harus Dipahami ?

Banyak developer mengira mereka bisa menerapkan protokol transaksi klasik seperti **Two-Phase Commit (2PC)** di sistem skala besar. Ini adalah kesalahan fatal.

### Teorema CAP dan Realitas Jaringan:

Dalam sistem terdistribusi, teorema CAP menyatakan bahwa kita tidak bisa memiliki ketiga hal ini sekaligus:

-   **C** (Consistency - Konsistensi data 100% instan di semua node)
-   **A** (Availability - Sistem selalu bisa menerima request tanpa gagal)
-   **P** (Partition Tolerance - Sistem tetap hidup meski jaringan internet antar server putus)

Karena partisi jaringan di cloud (_Partition_) adalah fakta tak terhindarkan, kita dipaksa memilih antara **C atau A**.

Jika memilih Consistency ketat (ACID Terdistribusi / 2PC): setiap kali ada 1 server downstream lambat atau kabel jaringan putus, seluruh sistem di dunia akan mengunci (_blocking locks_) dan menolak request pelanggan. Throughput langsung anjlok dari 10.000 req/s ke 5 req/s!

Oleh karena itu, sistem high-throughput beralih dari model **ACID** ke model **BASE**:

-   **Basically Available**: Sistem selalu melayani request.
-   **Soft State**: Kondisi data bisa berubah seiring waktu tanpa campur tangan pengguna.
-   **Eventual Consistency**: Data di semua database akhirnya akan konsisten setelah beberapa detik.

## 4. Cara Kerjanya: Two-Phase Commit (2PC) & Kelemahannya

Two-Phase Commit (2PC) adalah protokol klasik untuk mengkoordinasikan transaksi terdistribusi:

```
[ Phase 1: Prepare ]
Coordinator ------> Node A (Order DB):   "Bisa commit pesanan?" -> Node A: "BISA (Kunci Baris!)"
Coordinator ------> Node B (Payment DB): "Bisa potong saldo?"    -> Node B: "BISA (Kunci Saldo!)"

[ Phase 2: Commit ]
Coordinator mendapati SEMUA node menjawab BISA:
Coordinator ------> Node A: "COMMIT SEKARANG!" -> Node A: Sukses & Lepas Kunci
Coordinator ------> Node B: "COMMIT SEKARANG!" -> Node B: Sukses & Lepas Kunci
```

```
[ BENCANA 2PC: Node B Mati di Tengah Jalan ]
Coordinator ------> Node A: "COMMIT!" -> Sukses
Coordinator ------> Node B: KONEKSI PUTUS / CRASH!
(Node A sudah terlanjur commit, Node B tidak tahu harus apa.
 Baris data di Node A & B terkunci selamanya. Throughput hancur total!)
```

## 5. Cuplikan Logika Inti (Two-Phase Commit 2PC di Go)

Implementasi logika koordinator Two-Phase Commit (2PC) dengan mekanisme prepare, voting quorum, commit/abort, dan penanganan timeout di Go:

```go
package main

import (
	"context"
	"errors"
	"sync"
	"time"
)

type Participant interface {
	Prepare(ctx context.Context, txID string) error
	Commit(ctx context.Context, txID string) error
	Rollback(ctx context.Context, txID string) error
}

type TwoPhaseCoordinator struct {
	participants []Participant
	timeout      time.Duration
}

// Execute menjalankan protokol 2PC (Phase 1: Voting, Phase 2: Decision)
func (c *TwoPhaseCoordinator) Execute(ctx context.Context, txID string) error {
	reqCtx, cancel := context.WithTimeout(ctx, c.timeout)
	defer cancel()

	// Fase 1: Prepare (Voting Quorum)
	var wg sync.WaitGroup
	errCh := make(chan error, len(c.participants))

	for _, p := range c.participants {
		wg.Add(1)
		go func(part Participant) {
			defer wg.Done()
			if err := part.Prepare(reqCtx, txID); err != nil {
				errCh <- err
			}
		}(p)
	}
	wg.Wait()
	close(errCh)

	// Jika ada 1 partisipan yang menolak / gagal voting, batalkan semua (Rollback)
	if len(errCh) > 0 {
		for _, p := range c.participants {
			_ = p.Rollback(context.Background(), txID)
		}
		return errors.New("transaksi 2PC dibatalkan: salah satu partisipan gagal prepare")
	}

	// Fase 2: Commit global
	for _, p := range c.participants {
		_ = p.Commit(context.Background(), txID)
	}
	return nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan (Pendekatan ACID Terdistribusi / 2PC)

-   **Konsistensi Kuat (_Strong Consistency_)**: Tidak ada jeda _eventual consistency_. Data langsung 100% sinkron seketika.
-   **Model Mental Sederhana bagi Developer**: Menyerupai cara kerja transaksi database lokal.

### Kekurangan

-   **Blocking Protocol**: Semua database mengunci baris data (_database row locks_) selama proses voting jaringan berlangsung. Latensi meledak puluhan kali lipat.
-   **Single Point of Failure (SPOF)**: Jika Coordinator mati di tengah Phase 2, seluruh node partisipan terkatung-katung (_in-doubt state_) dan tidak berani melepas lock.
-   **Throughput Rendah**: Sangat tidak scalable untuk sistem modern yang melayani puluhan ribu transaksi per detik.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Boleh Pakai (2PC / XA Transactions)

-   Di dalam internal engine database terdistribusi modern yang menggunakan konsensus Raft/Paxos berkecepatan tinggi di satu data center (seperti Google Spanner, CockroachDB, YugabyteDB).
-   Sistem perbankan kliring antar-bank tertutup (_legacy financial networks_) dengan volume transaksi rendah tetapi regulasi hukum audit instan.

### Kapan Jangan Pakai (Hindari 2PC!)

-   Komunikasi antar microservices melalui jaringan publik atau cloud multi-region.
-   Aplikasi e-commerce, ride-hailing, streaming, dan fintech skala konsumen dengan trafik tinggi. (Sebagai gantinya: gunakan **Saga Pattern**!).

## 8. Metrik Kunci & Observabilitas

Untuk memonitor latensi dan anomali transaksi terdistribusi pada Grafana / Datadog:

-   **`distributed_tx_duration_seconds`**: Waktu total eksekusi dari inisiasi transaksi hingga fase commit selesai di semua node.
-   **`distributed_tx_in_doubt_total`**: Jumlah transaksi yang tertahan dalam status tidak pasti (_in-doubt_) karena node mati di tengah proses 2PC.
-   **`distributed_tx_abort_rate`**: Persentase transaksi yang dibatalkan pada fase Prepare akibat timeout atau lock contention.
-   **`database_lock_hold_time_ms`**: Durasi row lock ditahan di setiap partisipan database; lonjakan metrik ini langsung mematikan throughput sistem.
-   **`coordinator_heartbeat_failures_total`**: Kegagalan koneksi antara koordinator dan node partisipan.

## 9. Studi Kasus Nyata di Industri

### Kegagalan Two-Phase Commit (2PC) pada Sistem Kliring Antar-Bank dan Migrasi ke Konsistensi Eventual

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah konsorsium perbankan awalnya menerapkan protokol Two-Phase Commit (2PC / XA Transactions) yang menghubungkan Core Banking Database, Fraud Detection Database, dan Payment Switch eksternal secara sinkron. Ketika trafik transfer antar-bank memuncak di awal bulan, partisi jaringan intermiten menyebabkan koordinator transaksi mati saat prepare phase. Akibatnya, ratusan ribu row database terkunci (*distributed lock-in-doubt*), pool koneksi database exhausted (mencapai batas 2.000 max connections), dan throughput transfer anjlok hingga kegagalan berantai (*cascading outage*).
- **Implementasi Solusi Industri (The Architectural Solution)**: Rekayasa ulang sistem memensiunkan protokol 2PC sinkron lintas domain jaringan dan beralih ke transaksi terdistribusi berbasis *Eventual Consistency* dan *Reconciliation Ledger*. Transaksi transfer dipecah menjadi state machine lokal: saldo pengirim didebit secara atomik dengan status `PENDING`, lalu instruksi transfer dikirim melalui message broker dengan jaminan pesan transaksional. Sistem kliring penerima memproses secara asinkron dan mengirimkan acknowledgement konfirmasi; proses reconciliation worker berkala (setiap 30 detik) mencocokkan mutasi ledger dan menyelesaikan transaksi *in-flight*.
- **Dampak & Hasil Sistem (The Outcome)**: Throughput sistem melonjak 12x lipat (dari 450 TPS menjadi 5.500+ TPS). Ketergantungan *distributed locking* antar entitas dieliminasi sepenuhnya, menghapus risiko *lock contention* dan *pool exhaustion*. Ketersediaan sistem kliring naik dari 99,2% menjadi 99,99%, dengan zero orphan locks selama insiden partisi jaringan.

## 10. Fakta Menarik & Gotchas Production

-   **Distributed Deadlock**: Ketika Service A mengunci row di Database 1 dan menunggu Database 2, sementara Service B mengunci row di Database 2 dan menunggu Database 1. Di database monolith, engine memiliki _Deadlock Detector_ otomatis. Di sistem terdistribusi, mendeteksi distributed deadlock hampir mustahil tanpa timeout global yang agresif.
-   **Penyelesaian Transaksi di Dunia Nyata Bukan 2PC**: Banyak developer terkejut mengetahui bahwa perusahaan seperti Amazon, Uber, dan Netflix **TIDAK MENGGUNAKAN** 2PC untuk transaksi pemesanan mereka. Mereka menggunakan kompensasi bisnis: terima order dulu, jika pembayaran gagal semenit kemudian, kirim email _"Pembayaran Anda Ditolak"_ dan batalkan order secara asynchronous.

## 11. Rangkuman Singkat

- **Biaya Mahal 2PC**: Protokol 2-Phase Commit (2PC) bersifat *blocking*; latensi transaksi terdistribusi dibatasi oleh node paling lambat dan rawan kebuntuan (*in-doubt locks*) jika koordinator mati.
- **Hindari 2PC Lintas Jaringan Publik**: Jangan gunakan 2PC/XA untuk microservices lintas jaringan cloud; batasi penggunaannya hanya di internal engine database terdistribusi modern (seperti Spanner atau CockroachDB).
- **Pilih Konsistensi Bertahap**: Desain sistem bisnis skala besar dengan *eventual consistency* dan pola kompensasi (Saga) daripada memaksakan ACID kaku yang membunuh throughput.
- **Audit Ledger & Rekonsiliasi**: Selalu sediakan log rekonsiliasi (*audit ledger*) dan worker berkala untuk memverifikasi serta menyeimbangkan status data yang mengalami kegagalan di tengah jalan.
