# Vertical Scaling (Scale-Up)

## 1. Skenario Sistem Nyata

Pada sebuah platform perbankan digital, basis data transaksional utama (PostgreSQL) menangani pembukuan akuntansi (_general ledger_) dengan persyaratan kepatuhan finansial ketat dan isolasi transaksi level ACID tingkat tinggi.

Ketika volume transaksi harian bertumbuh pesat:

-   Seluruh indeks dan tabel data aktif (_working set_) tidak lagi muat di dalam memori RAM server yang hanya berukuran 32 GB.
-   PostgreSQL terpaksa melakukan jutaan pembacaan acak ke disk SSD (_Disk I/O thrashing_). Latensi transaksi melonjak dari 5 milidetik menjadi 450 milidetik, menyebabkan antrean pembayaran mengular dan koneksi aplikasi ke database mengalami _timeout_.
-   Melakukan pemecahan database ke banyak mesin (_database sharding_) dalam waktu singkat sangat berisiko tinggi karena membutuhkan penulisan ulang ratusan query pelaporan keuangan yang kompleks.

**Solusi dengan Vertical Scaling (Scale-Up):**
Tim infrastruktur menjadwalkan jendela pemeliharaan (_maintenance window_) selama 5 menit pada jam 02.00 dini hari untuk meningkatkan spesifikasi instance database cloud: dari 8 vCPU dan 32 GB RAM menjadi 64 vCPU, 256 GB RAM, serta storage beralih ke NVMe berkecepatan 64.000 IOPS. Begitu server menyala kembali, parameter `shared_buffers` PostgreSQL dinaikkan ke 64 GB sehingga 98% pembacaan query langsung dilayani dari RAM. Latensi query kembali anjlok ke level sub-milidetik secara instan tanpa perlu merombak satu baris pun kode arsitektur backend.

## 2. Apa Itu dan Buat Apa?

Vertical Scaling (_Scale-Up_) adalah strategi meningkatkan performa sistem dengan cara menambah daya komputasi pada satu mesin atau node yang sama.

Peningkatan resource mencakup:

-   **CPU**: Menambah jumlah core prosesor atau beralih ke generasi CPU dengan _clock speed_ lebih tinggi.
-   **Memori (RAM)**: Memperbesar kapasitas RAM agar cache in-memory dan working set database dapat ditampung seluruhnya di memori utama.
-   **Penyimpanan (Storage & IOPS)**: Beralih dari standard SSD ke Provisioned IOPS NVMe SSD untuk mempercepat throughput baca-tulis disk.
-   **Jaringan (Bandwidth NIC)**: Meningkatkan kartu jaringan virtual dari 1 Gbps menjadi 10 Gbps atau 25 Gbps untuk melayani throughput socket TCP yang lebih padat.

## 3. Kenapa Harus Dipahami ?

Meskipun sistem terdistribusi sering mengagungkan horizontal scaling, pemahaman vertical scaling sangat krusial karena:

1. **Pilihan Utama untuk Sistem Relasional (RDBMS)**: Database transaksional monolitik (PostgreSQL/MySQL) jauh lebih mudah, stabil, dan konsisten saat diperbesar secara vertikal daripada dipecah (_sharded_) ke puluhan node.
2. **Nol Kompleksitas Terdistribusi**: Tidak ada masalah _network partition_, latensi konsensus jaringan (Raft/Paxos), _two-phase commit_, atau _eventual consistency_.
3. **Efisiensi Kode Sebelum Scale-Out**: Satu node Go modern yang dioptimasi dengan baik pada server 32-core mampu melayani lebih dari 100.000 RPS. Melakukan scale-out sebelum memaksimalkan kapasitas vertikal sering kali hanya membuang anggaran cloud.

## 4. Cara Kerjanya (Diagram Alur)

```
+-------------------------------------------------------------------------+
|                  Perbandingan Kapasitas Mesin Tunggal                   |
+-------------------------------------------------------------------------+

 [Sebelum: Server Standar]                [Sesudah: Server Monster (Scale-Up)]
 +-----------------------+                +---------------------------------+
 | 4 vCPU                |                | 64 vCPU                         |
 | 16 GB RAM             |  Scale-Up      | 256 GB RAM                      |
 | Standard SSD 3k IOPS  | -------------> | NVMe Enterprise 64k IOPS        |
 | Bandwidth 1 Gbps NIC  |                | Bandwidth 25 Gbps NIC           |
 +-----------------------+                +---------------------------------+
            |                                              |
            v                                              v
 * Working Set meluap ke disk              * 98% Working set muat di RAM
 * CPU 100% saat 3.000 RPS                 * CPU stabil 30% saat 50.000 RPS
 * Latensi tinggi (Disk I/O)               * Latensi sub-milidetik (RAM I/O)
```

## 5. Cuplikan Logika Inti (Profiling & Resource Tuning di Go)

Untuk memaksimalkan performa server berspesifikasi tinggi (Scale-Up), aplikasi Go harus diatur agar tidak boros memori dan dapat memanfaatkan seluruh core CPU dengan efisien menggunakan `runtime/debug.SetMemoryLimit` dan profiling endpoint `net/http/pprof`:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	_ "net/http/pprof" // Mengaktifkan profiler runtime (/debug/pprof/)
	"runtime"
	"runtime/debug"
)

func init() {
	// 1. Maksimalkan utilisasi seluruh core prosesor mesin vertikal
	numCPU := runtime.NumCPU()
	runtime.GOMAXPROCS(numCPU)
	fmt.Printf("[Scale-Up Init] Mengalokasikan GOMAXPROCS ke %d Core CPU\n", numCPU)

	// 2. Go 1.19+ Memory Limit: Mencegah OOM Killer pada server berkapasitas besar
	//    Misal server memiliki RAM 64 GB, tetapkan batas aman GC pada 58 GB
	const memoryLimitBytes = 58 * 1024 * 1024 * 1024 // 58 GiB
	debug.SetMemoryLimit(memoryLimitBytes)
	fmt.Println("[Scale-Up Init] Soft memory limit ditetapkan untuk mencegah OOM crash")
}

func main() {
	// Menjalankan profiler internal di port terpisah untuk mendiagnosis saturasi CPU/RAM
	go func() {
		log.Println("[Profiler] pprof mendengarkan pada :6060")
		_ = http.ListenAndServe("localhost:6060", nil)
	}()

	mux := http.NewServeMux()
	mux.HandleFunc("/compute", func(w http.ResponseWriter, r *http.Request) {
		// Worker memanfaatkan thread pool dan parallelism CPU tinggi
		w.WriteHeader(http.StatusOK)
		fmt.Fprintf(w, "Proses kalkulasi selesai dieksekusi di node vertikal berkapasitas tinggi")
	})

	server := &http.Server{
		Addr:    ":8080",
		Handler: mux,
	}

	log.Println("[Server] Aplikasi Go berjalan pada :8080")
	if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		log.Fatalf("Server error: %v", err)
	}
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Kesederhanaan Arsitektur**: Tidak ada kode tambahan untuk konsistensi data terdistribusi, routing RPC, atau konsensus cluster.
-   **Kompatibilitas 100%**: Aplikasi monolitik dan database SQL standar dapat langsung berjalan tanpa perlu refactoring arsitektur.
-   **Latensi Komputasi Rendah**: Komunikasi antar proses terjadi melalui bus memori (_inter-process / memory bus_), bukan via jaringan kabel data center.

### Kekurangan

-   **Batas Keras Perangkat Keras (_Hardware Ceiling_)**: Server tidak bisa diperbesar tanpa batas; ada batas fisik socket CPU dan slot RAM motherboard.
-   **Biaya Eksponensial**: Harga server berspesifikasi ekstrem di penyedia cloud (misal instance 128 Core / 1 TB RAM) berlipat ganda jauh lebih mahal dibanding menggabungkan beberapa server ukuran sedang.
-   **Risiko Single Point of Failure (SPOF)**: Jika motherboard atau power supply server tunggal tersebut rusak, seluruh sistem mati total.
-   **Downtime Saat Resize**: Mengubah ukuran mesin virtual di cloud biasanya memerlukan reboot server yang memicu downtime singkat.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Database relasional utama (PostgreSQL / MySQL) yang memerlukan integritas transaksi ACID ketat dan relasi join yang kompleks.
-   Tahap awal produk (_early-stage startup_) di mana tim kecil harus fokus membangun fitur bisnis tanpa beban mengelola infrastruktur cluster yang rumit.
-   Sistem komputasi in-memory bervolume tinggi atau analitik data tunggal yang menuntut akses RAM super cepat.

### Kapan Jangan Pakai

-   Sistem web publik dengan trafik yang melonjak secara mendadak dan fluktuatif di mana kebutuhan autoscaling elastis sangat diperlukan.
-   Layanan yang menuntut ketersediaan tanpa henti (_zero-downtime SLA 99.999%_) di mana reboot server untuk upgrade tidak diizinkan.
-   Sistem yang beban kerjanya sudah melampaui batas spesifikasi server komersial tertinggi yang ada di pasar.

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`node_load1` vs `node_cpu_cores` (Gauge)**: Memantau apakah load average sistem sudah melampaui jumlah core fisik yang tersedia.
-   **`node_memory_MemAvailable_bytes` (Gauge)**: Sisa memori fisik yang tersedia sebelum sistem mulai menggunakan swap memory.
-   **`node_disk_io_time_seconds_total` (Counter)**: Persentase waktu disk mengalami saturasi penanganan I/O.
-   **`go_gc_duration_seconds` (Histogram)**: Durasi pause Garbage Collection pada aplikasi Go dengan heap berukuran gigabyte besar.

## 9. Studi Kasus Industri Nyata

-   **Stack Overflow**: Salah satu contoh implementasi vertical scaling paling terkenal di dunia rekayasa perangkat lunak. Stack Overflow melayani miliaran kunjungan halaman per bulan hanya menggunakan segelintir server web bare-metal dan dua server database SQL Server berukuran monster (puluhan core CPU, ratusan GB RAM, dan PCIe SSD berkecepatan tinggi) dengan efisiensi performa yang luar biasa.
-   **Shopify**: Pada layer basis data inti, Shopify mengandalkan server MySQL berspesifikasi hardware sangat tinggi untuk melayani transaksi jutaan toko online sebelum kemudian menerapkan teknik sharding pod untuk skala berikutnya.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Ilusi Skalabilitas Palsu (_False Sense of Scalability_)**: Terus menerus menaikkan spek mesin untuk menutupi query SQL yang tidak memiliki indeks atau kebocoran memori (_memory leak_). Begitu hardware mencapai batas tertinggi, sistem langsung kolaps tanpa jalan keluar instan.
2. **Kekagetan Biaya Cloud (_Cloud Bill Shock_)**: Instance cloud tier tertinggi memiliki margin harga premium yang sangat mahal per jam. Sering kali mengonfigurasi 4 mesin medium jauh lebih murah daripada 1 mesin monster berukuran 4x lipat.
3. **Waktu Pemulihan Bencana Sangat Lama (_Prolonged Disaster Recovery_)**: Merestore backup database berukuran 10 TB pada satu server tunggal saat crash darurat membutuhkan waktu berjam-jam hingga harian, memperpanjang durasi downtime operasional perusahaan.

## 11. Rangkuman Singkat

Vertical Scaling adalah strategi meningkatkan performa sistem dengan memperbesar kapasitas CPU, RAM, dan storage pada satu mesin server. Pendekatan ini mempertahankan kesederhanaan arsitektur tanpa kompleksitas jaringan terdistribusi, menjadikannya pilihan ideal untuk database relasional dan tahap awal rekayasa backend sebelum sistem benar-benar membutuhkan penskalaan horizontal.
