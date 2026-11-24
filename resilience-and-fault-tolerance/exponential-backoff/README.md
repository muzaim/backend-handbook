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

## 9. Studi Kasus Industri Nyata

### Eliminasi Gelombang Lockstep Thundering Herd pada Dispatcher Webhook

- **Tantangan Rekayasa Riil (The Problem)**:
  Sebuah payment gateway memproses pengiriman notifikasi webhook transaksi ke ribuan server merchant melalui armada worker Go. Ketika terjadi gangguan jaringan selama 4 detik pada salah satu penyedia cloud merchant, 12.000 worker webhook gagal secara serentak. Sistem awalnya mengimplementasikan *Deterministic Exponential Backoff* tanpa randomisasi:
  $$\text{Attempt 1} = 1.000\text{ms}, \quad \text{Attempt 2} = 2.000\text{ms}, \quad \text{Attempt 3} = 4.000\text{ms}$$
  Hasilnya menjadi malapetaka: ketika server merchant mulai pulih, seluruh 12.000 worker menembakkan paket HTTP POST secara persis bersamaan di detik ke-1, lalu serentak lagi di detik ke-2, dan serentak lagi di detik ke-4 (*Lockstep Waves*). Beban impuls mendadak sebesar 12.000 RPS ini membanjiri buffer socket server merchant, memicu *connection refused*, dan memaksa server merchant tumbang kembali berulang kali (*self-inflicted denial of service*).

- **Implementasi Solusi Industri (The Architectural Solution)**:
  1. **Transisi ke Algoritma Full Jitter**: Formula penundaan waktu di kode Go diubah menjadi Full Jitter:
     ```go
     maxWait := baseInterval * time.Duration(1<<attempt)
     if maxWait > maxCap {
         maxWait = maxCap
     }
     sleep := time.Duration(rand.Int63n(int64(maxWait)))
     ```
  2. **Decorrelated Noise Injection**: Alih-alih menembak serentak di milidetik yang identik, waktu tunggu 12.000 worker tersebar merata di sepanjang rentang waktu acak kontinu antara 0 hingga batas maksimum backoff.
  3. **Plafon Atas (Max Backoff Cap)**: Ditetapkan batas atas penundaan maksimal sebesar 30 detik agar antrean webhook tidak tertunda berjam-jam saat server merchant membutuhkan waktu pemulihan lebih lama.

- **Dampak & Hasil Sistem (The Engineering Outcome)**:
  - Puncak lonjakan beban trafik (*peak traffic burst*) terpangkas sebesar 91%, merata dari gelombang impuls 12.000 RPS menjadi aliran stabil ~1.100 RPS.
  - Server merchant berhasil pulih sepenuhnya dalam 14 detik tanpa mengalami insiden crash sekunder.
  - Seluruh notifikasi webhook berhasil terkirim tuntas dengan tingkat keberhasilan 100% tanpa ada data transaksi yang hilang.

## 10. Fakta Menarik & Gotchas Production: The Power of "JITTER"

Ini adalah rahasia arsitektur paling penting yang sering dilewatkan developer junior: **Exponential Backoff tanpa Jitter adalah bom waktu di high throughput!**

Ketika terjadi gangguan jaringan yang menyebabkan 10.000 worker mengalami kegagalan bersamaan, jika semuanya menghitung backoff deterministik: $100\text{ms} \times 2^1 = 200\text{ms}$, maka **seluruh 10.000 instance tersebut akan menembak target secara sinkron di detik ke-200ms secara bersamaan** (_Lockstep Waves_). Fenomena gelombang sinkron ini tetap akan merobohkan server tujuan!

**Solusinya adalah JITTER (Pengacakan Waktu):**
Suntikkan angka acak pada durasi tunggu:
$$\text{Wait Time} = \text{Random}(0, \text{Base Interval} \times 2^{\text{attempt}})$$

Dengan Full Jitter, 10.000 server tersebut tidak akan menembak secara bersamaan. Ada yang mencoba di milidetik ke-45, ada yang di ke-120, ada yang di ke-185. Trafik terdistribusi secara halus dan server downstream selamat.

## 11. Rangkuman Singkat

1. **Eksponensial Meredakan Beban**: Memperpanjang jeda waktu secara eksponensial memberi ruang napas bagi service downstream yang sedang mengalami pemulihan sistemik.
2. **Jitter adalah Komponen Mutlak**: Exponential backoff tanpa jitter menciptakan gelombang serangan serentak (*lockstep thundering herd*); selalu suntikkan randomisasi *Full Jitter*.
3. **Plafon Maksimum (Max Cap)**: Selalu tentukan batas atas durasi tunggu (misal 30 detik) agar kalkulasi penundaan tidak membengkak tanpa batas hingga berjam-jam.
4. **Patuhi Batas Timeout Context**: Pastikan total waktu penundaan backoff tidak melampaui sisa waktu deadline context yang dialokasikan oleh request pemanggil.
