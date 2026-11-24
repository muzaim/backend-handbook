# Exponential Backoff

## 1. Skenario Sistem Nyata

Pada arsitektur event-driven pemrosesan transaksi, armada 50 worker pod di background bertugas mengirimkan webhook notifikasi status pembayaran ke ribuan server API mitra e-commerce.

Tiba-tiba server mitra mengalami kelebihan beban sementara dan mulai mengembalikan status `HTTP 429 Too Many Requests` atau `HTTP 503 Service Unavailable`.

Jika 50 worker pod kamu melakukan retry secara agresif dengan interval jeda tetap (misal: langsung mengirim ulang request setiap 100 milidetik):

-   Ribuan request retry serentak akan menghujani server mitra yang sedang megap-megap (_Thundering Herd Effect_ atau _Retry Storm_).
-   Server mitra bukannya sembuh, malah semakin tenggelam dan mengalami crash permanen akibat serangan DoS tidak sengaja dari service kamu sendiri.
-   Tidak ada satu pun notifikasi yang berhasil terkirim.

**Solusi dengan Exponential Backoff + Jitter:**
Setiap kali request gagal, worker pod melipatgandakan waktu tunggunya secara bertahap: percobaan 1 menunggu 1 detik, percobaan 2 menunggu 2 detik, percobaan 3 menunggu 4 detik, dan seterusnya, ditambah variasi acak (_jitter_ beberapa milidetik). Strategi ini meredam kepanikan trafik secara instan dan memberikan ruang bernapas bagi server mitra untuk menuntaskan antrean internalnya dan kembali sehat.

## 2. Apa Itu dan Buat Apa?

Exponential Backoff adalah algoritma pengendalian jeda waktu (_retry interval_) di mana sistem melipatgandakan waktu tunggu secara eksponensial setiap kali sebuah percobaan operasi mengalami kegagalan.

Formula dasarnya:
$$\text{Wait Time} = \text{Base Interval} \times 2^{\text{attempt}}$$

Contoh perhitungan dengan Base Interval = 100ms:

-   Gagal Attempt 1: Tunggu $100\text{ms} \times 2^1 = 200\text{ms}$
-   Gagal Attempt 2: Tunggu $100\text{ms} \times 2^2 = 400\text{ms}$
-   Gagal Attempt 3: Tunggu $100\text{ms} \times 2^3 = 800\text{ms}$
-   Gagal Attempt 4: Tunggu $100\text{ms} \times 2^4 = 1.600\text{ms}$ (1,6 detik)
-   Ditambah batas atas maksimal (_Max Cap_, misal max 10 detik agar tidak menunggu selamanya).

Tujuan utamanya adalah **memberikan waktu pemulihan bagi server tujuan** dan meredam kepanikan trafik di jaringan.

## 3. Kenapa Harus Pakai Ini ?

Jika 1.000 worker aplikasi melakukan retry secara instan (_Immediate Retry_ tanpa jeda waktu):

-   Server backend yang tadinya sedang sibuk 100% akan langsung menerima gelombang serangan kedua secara serentak.
-   Server tidak akan pernah memiliki kesempatan untuk menyelesaikan antrean pekerjaan lamanya, membersihkan memori, atau memulihkan koneksi database.

Dengan Exponential Backoff, frekuensi serangan retry mereda secara otomatis. Semakin parah masalah yang dialami server downstream, semakin jarang client menghubunginya, memungkinkan server downstream mendinginkan CPU dan pulih secara bertahap.

## 4. Cara Kerjanya (Diagram Alur & Perbandingan)

```
[ Immediate Retry: Membunuh Server yang Sedang Sekarat ]
Waktu:     0s       0.1s     0.2s     0.3s
Client 1: [Fail] -> [Retry]->[Retry]->[Retry] (Bombardir konstan)
Client 2: [Fail] -> [Retry]->[Retry]->[Retry] (Server Overheat & Crash)


[ Exponential Backoff: Memberi Waktu Pemulihan ]
Waktu:     0s         1s                  3s                               7s
Client:   [Fail] -> (Wait 1s) -> [Fail] -> (Wait 2s) -> [Fail] -> (Wait 4s) -> [Success!]
                                            ^
                                            Server berhasil bernapas & pulih
```

## 5. Cuplikan Logika Inti (Exponential Backoff dengan Full Jitter di Go)

Berikut implementasi algoritma Exponential Backoff standar industri AWS menggunakan _Full Jitter_ di Go:

```go
package main

import (
	"context"
	"math"
	"math/rand"
	"time"
)

// SleepWithExponentialBackoff menerapkan algoritma Full Jitter AWS
func SleepWithExponentialBackoff(ctx context.Context, attempt int, baseInterval time.Duration, maxCap time.Duration) error {
	// 1. Hitung durasi eksponensial: base * 2^attempt
	multiplier := math.Pow(2, float64(attempt))
	temp := float64(baseInterval) * multiplier

	if temp > float64(maxCap) {
		temp = float64(maxCap)
	}

	// 2. Full Jitter: pilih angka acak antara 0 dan temp untuk memecah gelombang trafik
	sleepDuration := time.Duration(rand.Float64() * temp)

	select {
	case <-time.After(sleepDuration):
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Menyelamatkan Infrastruktur**: Menghindarkan downstream service dari kehancuran total akibat gelombang retry bertubi-tubi.
-   **Toleransi Rate Limiting**: Sangat efektif ketika menghadapi respon HTTP 429 Too Many Requests dari penyedia API pihak ketiga (AWS, Stripe, OpenAI).
-   **Efisiensi Jaringan**: Mengurangi congestion di switch dan router jaringan internal.

### Kekurangan

-   **Latensi Menumpuk pada Pengguna**: Jika diterapkan secara mentah di aplikasi interaktif (mobile app), pengguna bisa merasa aplikasi macet (_freezing_) karena menunggu jeda backoff yang panjang.
-   **Thread Starvation pada Caller**: Worker thread pemanggil harus tidur (_sleep_) atau memblokir eksekusi sementara menunggu timer backoff, kecuali menggunakan runtime asynchronous / non-blocking I/O.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Setiap kali mengimplementasikan mekanisme **Retries** pada panggilan jaringan eksternal.
-   Background jobs, message consumers, dan worker antrean (Celery, Sidekiq, Kafka consumer, BullMQ).
-   Polling status transaksi panjang (misal: memeriksa status pembayaran QRIS ke payment gateway setiap beberapa detik).

### Kapan Jangan Pakai

-   Transaksi sinkron web user-facing dengan SLA latensi ketat (misal autocomplete search bar; lebih baik langsung gagal dalam 300ms daripada menunggu 5 detik).
-   Error permanen seperti validasi payload salah (HTTP 400) atau unauthorized (HTTP 401).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`backoff_sleep_duration_seconds` (Histogram p50/p95/p99)**: Memantau rata-rata durasi tidur worker. Jika p99 menyentuh `max_cap`, backend tujuan sedang mengalami outage parah.
-   **`retry_attempts_histogram` (Histogram)**: Mengamati di attempt ke berapa sebagian besar transaksi berhasil sembuh (normalnya di attempt ke-1 atau ke-2).

## 9. Studi Kasus Nyata di Industri

-   **Amazon Web Services (AWS Architecture Blog)**: Tim arsitektur AWS menerbitkan paper penelitian terkenal _"Exponential Backoff And Jitter"_. Mereka membuktikan secara matematis dan simulasi bahwa Exponential Backoff murni tanpa Jitter tetap gagal meredam badai server karena request tiba dalam gelombang sinkron (_lockstep waves_). Pengenalan **Full Jitter** terbukti memangkas waktu pemulihan server hingga 70%.
-   **Google Cloud (Bigtable & Cloud Storage SDK)**: Seluruh client library Google Cloud secara default mengintegrasikan exponential backoff dengan randomized jitter saat mengakses Bigtable atau GCS untuk memastikan ribuan aplikasi konsumen tidak saling bertabrakan saat cluster Bigtable melakukan load re-balancing.

## 10. Fakta Menarik & Gotchas Production: The Power of "JITTER"

Ini adalah rahasia arsitektur paling penting yang sering dilewatkan developer junior: **Exponential Backoff tanpa Jitter adalah bom waktu di high throughput!**

Bayangkan listrik sebuah gedung padam selama 2 detik, lalu menyala kembali. Jika ada 10.000 server yang mengalami kegagalan bersamaan, dan semuanya menghitung backoff: $100\text{ms} \times 2^1 = 200\text{ms}$, maka **seluruh 10.000 server tersebut akan menembak target secara sinkron di detik ke-200ms secara bersamaan** (_Lockstep Waves_). Fenomena gelombang sinkron ini tetap akan merobohkan server tujuan!

**Solusinya adalah JITTER (Pengacakan Waktu):**
Kita menambahkan angka acak pada durasi tunggu:
$$\text{Wait Time} = \text{Random}(0, \text{Base Interval} \times 2^{\text{attempt}})$$

Dengan Full Jitter, 10.000 server tersebut tidak akan menembak secara bersamaan. Ada yang mencoba di milidetik ke-45, ada yang di ke-120, ada yang di ke-185. Trafik terdistribusi secara halus dan server downstream selamat! (Formula ini dibuktikan secara mendalam oleh tim arsitektur AWS).
