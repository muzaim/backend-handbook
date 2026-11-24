# Saga Pattern

## 1. Skenario Sistem Nyata

Pada alur checkout pesanan platform e-commerce yang melibatkan 3 microservice independen dengan database terpisah (*Database-per-Service*):
1. **`Order Service`** membuat record pesanan dengan status `PENDING` di database PostgreSQL.
2. **`Payment Service`** mendebet saldo dompet digital pengguna sebesar Rp 300.000 di database MySQL.
3. **`Inventory Service`** memotong kuota stok fisik barang di database warehouse.

Langkah 1 berhasil (order tercipta). Langkah 2 berhasil (saldo pengguna sukses terpotong Rp 300.000). Namun pada Langkah 3, `Inventory Service` mengembalikan error: **Stok barang mendadak habis dibeli orang lain (*Out of Stock*)!**

Dilema Arsitektur:
Karena masing-masing service memiliki database sendiri, `Payment Service` sudah terlanjur melakukan `COMMIT` lokal di MySQL miliknya. Database SQL lokal tidak memiliki kemampuan untuk membatalkan (*rollback*) data yang sudah ter-commit di server fisik lain. Tanpa mekanisme pemulihan, saldo pengguna sudah raib Rp 300.000 sementara barangnya tidak pernah ada!

**Solusi dengan Saga Pattern:**
Saga memecah transaksi terdistribusi menjadi serangkaian transaksi lokal yang dipasangkan dengan aksi pembatalan terprogram (**Transaksi Kompensasi**):
- Kegagalan di `Inventory Service` segera memicu alur kompensasi mundur otomatis.
- Sistem memanggil fungsi kompensasi ke `Payment Service`: `RefundPayment(order_id)` untuk mengembalikan saldo Rp 300.000 ke dompet pengguna.
- Sistem memperbarui status di `Order Service` menjadi `CANCELLED: OUT_OF_STOCK`.
Semua database kembali konsisten tanpa pernah menggunakan protokol penguncian (*locking*) jaringan yang lambat.

## 2. Apa Itu dan Buat Apa?

Saga Pattern adalah pola arsitektur untuk mengelola transaksi bisnis terdistribusi lintas microservices tanpa menggunakan protokol locking yang lambat seperti Two-Phase Commit (2PC).

Di dalam Saga:
- Sebuah transaksi bisnis besar dipecah menjadi **serangkaian transaksi lokal (*Local Transactions*)** di masing-masing service.
- Setiap service memperbarui database-nya sendiri dengan transaksi SQL lokal biasa yang sangat cepat, lalu memicu langkah berikutnya via event/message.
- Jika ada salah satu langkah yang gagal (misal saldo kurang atau stok habis), Saga akan mengeksekusi serangkaian **Transaksi Kompensasi (*Compensating Transactions*)** untuk membatalkan perubahan yang terlanjur di-commit oleh langkah-langkah sebelumnya secara mundur.

## 3. Dua Pendekatan Saga: Choreography vs Orchestration

```
[ 1. Saga Choreography (Berbasis Event / Tanpa Koordinator) ]

Order Svc ----(Event: OrderCreated)----> Payment Svc
                                              |
                                     (Event: PaymentSuccess)
                                              |
                                              v
Inventory Svc <-------------------------------+
  (Potong Stok)
```
- *Karakter*: Setiap service mendengarkan event dari service sebelumnya dan memutuskan langkah berikutnya sendiri.
- *Cocok untuk*: Alur transaksi sederhana (2 hingga 4 service). Jika lebih dari itu, alur menjadi rumit dipahami (*spaghetti events*).

```
[ 2. Saga Orchestration (Berbasis Orkestrator Terpusat) ]

                    +-------------------------+
                    |    Order Orchestrator   |
                    |     (State Machine)     |
                    +-------------------------+
                      |          ^          |
           1. Charge  |          | 3. Book  |
              Payment | 2. Paid  |    Stock | 4. Stock Empty!
                      v          |          v
                +-------------+  |    +---------------+
                | Payment Svc | -/    | Inventory Svc |
                +-------------+       +---------------+
                      ^
                      | 5. "KOMPENSASI: Refund Pembayaran!"
                      +---------------------------------
```
- *Karakter*: Ada satu service sentral (Orchestrator) yang menjadi otak pengendali: memanggil service A, menunggu hasil, memanggil service B, dan memanggil fungsi kompensasi/refund jika ada yang gagal.
- *Cocok untuk*: Alur transaksi kompleks dengan banyak cabang logika bisnis (5+ service).

## 4. Cara Kerjanya (Diagram Alur Eksekusi & Kompensasi)

```
Forward Steps:
[ OrderCreated ] ---> [ PaymentCharged ] ---> [ InventoryReserved ] (FAILED!)
                                                      |
                                                      v
Rollback (Compensating Steps):                        |
[ OrderCancelled ] <--- [ PaymentRefunded ] <---------+
```

## 5. Cuplikan Logika Inti (Saga Orchestrator di Go)

Implementasi pola Saga Orchestrator dengan eksekusi forward dan kompensasi otomatis saat salah satu langkah gagal di Go:

```go
package main

import (
	"context"
	"fmt"
)

type SagaStep struct {
	Name       string
	Execute    func(ctx context.Context) error
	Compensate func(ctx context.Context) error
}

// ExecuteSaga menjalankan orkestrasi langkah lokal dan kompensasi otomatis jika terjadi kegagalan
func ExecuteSaga(ctx context.Context, steps []SagaStep) error {
	var executedSteps []SagaStep

	for _, step := range steps {
		err := step.Execute(ctx)
		if err != nil {
			fmt.Printf("Langkah %s gagal: %v. Memulai proses kompensasi balik...
", step.Name, err)
			// Jalankan kompensasi dengan urutan mundur (LIFO)
			for i := len(executedSteps) - 1; i >= 0; i-- {
				_ = executedSteps[i].Compensate(ctx)
			}
			return fmt.Errorf("saga gagal pada langkah %s: %w", step.Name, err)
		}
		executedSteps = append(executedSteps, step)
	}

	return nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Throughput Ekstrem**: Skalabilitas tinggi karena hanya bergantung pada transaksi database lokal tanpa blocking lock lintas jaringan.
- **Toleransi Kegagalan Terdistribusi**: Mampu memulihkan status bisnis secara teratur melalui alur kompensasi terprogram.
- **Loose Coupling (pada Choreography)**: Service tidak perlu saling mengenal secara langsung.

### Kekurangan
- **Ketiadaan Isolasi Data (*Lack of Read Isolation*)**: Karena transaksi lokal langsung di-commit sebelum seluruh saga selesai, pengguna lain bisa melihat data sementara yang belum tentu permanen (*dirty reads* di level bisnis).
- **Kerumitan Menulis Transaksi Kompensasi**: Developer wajib menulis fungsi pembatalan untuk setiap langkah (misal: jika ada fungsi `PotongSaldo`, harus dibuatkan fungsi `RefundSaldo`).
- **Idempotency Wajib di Semua Titik**: Seluruh aksi kompensasi harus dirancang idempotent karena pesan kompensasi bisa terkirim lebih dari satu kali jika terjadi network retry.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Alur transaksi bisnis multi-step di arsitektur microservices (misal: proses checkout e-commerce, pemesanan tiket pesawat + hotel, alur pinjaman kredit finansial).
- Sistem dengan kebutuhan throughput tinggi yang tidak bisa menoleransi locking Two-Phase Commit (2PC).

### Kapan Jangan Pakai
- Arsitektur Monolith (gunakan transaksi SQL `BEGIN ... COMMIT` lokal biasa yang jauh lebih sederhana).
- Sistem yang secara mutlak menuntut *Read Isolation* tingkat tinggi di mana status sementara tidak boleh dilihat sama sekali oleh pihak luar (misal buku besar akuntansi perbankan inti).

## 8. Metrik Kunci & Observabilitas

Untuk memantau stabilitas dan rasio kompensasi alur Saga pada Grafana / Datadog:

- **`saga_execution_duration_seconds`**: Waktu total dari langkah pertama hingga langkah terakhir saga selesai.
- **`saga_compensation_rate`**: Persentase saga yang memicu kompensasi/rollback; lonjakan metrik ini mengindikasikan masalah eksternal (misal gateway pembayaran down atau stok inventaris habis).
- **`saga_failed_step_total`**: Counter error per langkah spesifik (misal kegagalan terbanyak ada di step `ReserveInventory`).
- **`saga_stuck_zombies_total`**: Jumlah saga yang menggantung karena ada langkah kompensasi yang gagal berulang kali dan membutuhkan intervensi manual.
- **`saga_active_instances_count`**: Jumlah alur saga yang sedang aktif berjalan bersamaan.

## 9. Studi Kasus Nyata di Industri

- **Uber Trip Management via Cadence / Temporal**: Uber mengelola jutaan perjalanan menggunakan orkestrator alur kerja berbasis Saga (Cadence/Temporal). Dari mencocokkan pengemudi, reservasi pembayaran, hingga pelacakan rute, setiap langkah dimodelkan sebagai state machine terdistribusi yang mengeksekusi kompensasi jika pengguna atau pengemudi membatalkan perjalanan.
- **Amazon Multi-Service Checkout**: Amazon memproses pesanan melalui rangkaian transaksi lokal: memotong voucher promosi, memverifikasi kartu kredit, dan mengirim instruksi ke robot gudang. Jika otorisasi kartu kredit ditolak, kompensasi otomatis mengembalikan voucher diskon pengguna tanpa mengunci database global.

## 10. Fakta Menarik & Gotchas Production

- **Semantic Rollback Bukan Undo Biasa**: Di database SQL, `ROLLBACK` memulihkan data persis ke kondisi awal sebelum query berjalan. Di dalam Saga, kompensasi adalah transaksi baru ke depan (*compensating forward action*). Misal: kamu tidak bisa membatalkan pengiriman email selamat datang yang terlanjur terkirim ke inbox pengguna; kompensasinya adalah mengirimkan email kedua: *"Mohon maaf, pendaftaran akun Anda dibatalkan karena kuota habis"*.
- **Pivot Transaction**: Dalam setiap Saga, selalu ada satu langkah penentu yang disebut **Pivot Transaction**. Setelah langkah pivot sukses, saga dijamin tidak akan pernah di-rollback lagi, melainkan dipaksa maju sampai selesai (*retry until success*). Mengidentifikasi mana langkah pivot (biasanya proses pembayaran) sangat penting agar kamu tidak perlu repot-repot membuat transaksi kompensasi yang mustahil.
