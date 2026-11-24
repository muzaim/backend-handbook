# Circuit Breakers

## 1. Skenario Sistem Nyata

Pada platform e-commerce berskala 10.000 transaksi per menit, `Checkout Service` bergantung pada panggilan HTTP synchronous ke `Fraud Detection Service` untuk memvalidasi skor risiko akun pengguna.

Tiba-tiba `Fraud Detection Service` mengalami kebocoran memori parah dan mati suri: setiap request yang masuk menggantung hingga batas timeout 5 detik.

Jika sistem tidak memiliki mekanisme perlindungan sirkuit:

-   Ribuan pengguna yang checkout secara bersamaan tertahan 5 detik per transaksi.
-   Thread pool dan koneksi soket di `Checkout Service` habis terkuras menunggu respon service fraud.
-   Akibatnya: `Checkout Service` ikut tumbang total. Seluruh operasional e-commerce lumpuh hanya karena satu service pendukung minor mengalami gangguan (_cascading failure_).

**Solusi dengan Circuit Breaker (misal: Netflix Hystrix / Resilience4j):**
Kamu memasang Circuit Breaker pada pemanggilan ke Fraud Service. Ketika terdeteksi tingkat kegagalan (_error rate_) melampaui batas ambang (misal 50% request gagal dalam 10 detik terakhir), Circuit Breaker langsung trip ke status **OPEN (memutus sirkuit)**. Semua request checkout berikutnya seketika melewati Fraud Service tanpa menunggu jaringan (fail-fast < 1 ms) dan langsung dialihkan ke strategi _Graceful Fallback_ (misal: izinkan transaksi nominal kecil dan jadwalkan audit fraud di background). Sistem checkout utama tetap 100% responsif bagi pembeli.

## 2. Apa Itu dan Buat Apa?

Circuit Breaker adalah pola desain stabilitas (_stability pattern_) dalam arsitektur microservices yang bertugas mendeteksi kegagalan pada service downstream dan **seketika memutus aliran request** ke service tersebut jika tingkat kegagalan (_error rate_) melampaui batas toleransi.

Alih-alih membiarkan request terus menunggu sampai timeout, Circuit Breaker akan langsung mengembalikan respon _Fail-Fast_ atau respon cadangan (_Graceful Fallback_) dalam hitungan mikrodetik.

### Tiga Status (_State Machine_) Circuit Breaker:

1. **CLOSED (Sirkuit Tertutup - Normal)**:

    - Aliran request berjalan lancar seperti biasa.
    - Circuit breaker menghitung statistik: berapa request sukses dan berapa yang gagal/timeout.
    - Jika rasio kegagalan (misal > 50% dalam 10 detik terakhir) terlampaui, sirkuit otomatis berpindah ke **OPEN**.

2. **OPEN (Sirkuit Terbuka - Trip/Putus)**:

    - Sirkuit terputus total. Request dari client **TIDAK AKAN dikirimkan ke downstream service**.
    - Pemanggil langsung mengembalikan error atau data fallback seketika (_instant fail-fast_, latensi 0ms).
    - Memberikan waktu istirahat penuh bagi downstream service untuk pulih.
    - Timer pendinginan (_Sleep Window / Cooldown_, misal 30 detik) mulai berjalan.

3. **HALF-OPEN (Sirkuit Setengah Terbuka - Uji Coba)**:
    - Setelah waktu pendinginan habis, circuit breaker mengizinkan sejumlah kecil request percobaan (misal 5-10 request) untuk lewat ke downstream.
    - Jika request percobaan tersebut sukses, circuit breaker menganggap downstream sudah sembuh dan kembali ke **CLOSED**.
    - Jika ada request percobaan yang gagal, sirkuit langsung kembali ke **OPEN** dan timer pendinginan di-reset.

Contoh library populer: Resilience4j (Java), Sony/gobreaker (Go), Polly (.NET), Opossum (Node.js).

## 3. Kenapa Harus Pakai Ini ?

Bayangkan sistem E-Commerce saat Flash Sale:
Service Order memanggil Service Rekomendasi untuk menampilkan "Produk Serupa". Tiba-tiba Service Rekomendasi mengalami out-of-memory dan hang.

Tanpa Circuit Breaker:

-   Setiap pengguna yang ingin checkout harus menunggu timeout 3 detik di Service Rekomendasi.
-   Ribuan thread di Service Order terkunci menunggu rekomendasi yang tidak pernah datang.
-   Akhirnya, Service Order ikut tumbang! Fitur vital (pembayaran) mati hanya gara-gara fitur pelengkap (rekomendasi produk).

Dengan Circuit Breaker:

-   Setelah 20 request gagal berturut-turut, Circuit Breaker langsung beralih ke **OPEN**.
-   10.000 request berikutnya tidak akan menyentuh Service Rekomendasi sama sekali.
-   Service Order langsung mengembalikan array kosong `[]` untuk rekomendasi dan transaksi checkout utama pengguna berjalan mulus dalam 50 milidetik!

## 4. Cara Kerjanya (Diagram State Machine)

```
       +-------------------------------------------------------+
       |                                                       |
       |  Request Berhasil (Success Rate Normal)               |
       v                                                       |
+--------------+        Tingkat Kegagalan Melebihi Batas       |
|    CLOSED    | -------------------------------------\        |
| (Normal Ops) |                                      |        |
+--------------+                                      v        |
       ^                                       +--------------+
       |                                       |     OPEN     |
       |                                       | (Fail-Fast / |
       |                                       |   Fallback)  |
       |                                       +--------------+
       | Uji Coba Sukses                              |
       |                                              | Cooldown Selesai
       |                                              | (e.g. 30 detik)
       |               +---------------+              |
       \-------------- |   HALF-OPEN   | <------------/
                       | (Uji Sebagian)|
                       +---------------+
                               |
                               | Uji Coba Gagal
                               v
                         (Kembali OPEN)
```

## 5. Cuplikan Logika Inti (State Machine Circuit Breaker di Go)

Berikut logika state machine Circuit Breaker di Go:

```go
package main

import (
	"errors"
	"sync"
	"time"
)

type State int

const (
	StateClosed State = iota
	StateOpen
	StateHalfOpen
)

type CircuitBreaker struct {
	mu               sync.Mutex
	state            State
	failureCount     int
	failureThreshold int
	recoveryTimeout  time.Duration
	lastFailureTime  time.Time
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
	cb.mu.Lock()
	now := time.Now()

	// Jika OPEN dan waktu recovery sudah lewat, beralih ke HALF-OPEN untuk uji coba
	if cb.state == StateOpen && now.Sub(cb.lastFailureTime) > cb.recoveryTimeout {
		cb.state = StateHalfOpen
	}

	if cb.state == StateOpen {
		cb.mu.Unlock()
		return errors.New("circuit breaker is OPEN (permintaan ditolak otomatis)")
	}
	cb.mu.Unlock()

	err := fn()

	cb.mu.Lock()
	defer cb.mu.Unlock()

	if err != nil {
		cb.failureCount++
		cb.lastFailureTime = time.Now()
		if cb.failureCount >= cb.failureThreshold || cb.state == StateHalfOpen {
			cb.state = StateOpen
		}
		return err
	}

	// Jika sukses di status HALF-OPEN, pulihkan sirkuit ke CLOSED
	if cb.state == StateHalfOpen {
		cb.state = StateClosed
		cb.failureCount = 0
	}
	return nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Mencegah Keruntuhan Berantai (_Cascading Failure_)**: Mengisolasi kegagalan satu komponen agar tidak merobohkan seluruh sistem.
-   **Fail-Fast Superior**: Menghemat CPU, memori, dan socket thread secara drastis saat downstream service sedang bermasalah.
-   **Graceful Degradation**: Pengguna tetap bisa menggunakan fungsi inti aplikasi meski beberapa fitur sekunder dimatikan sementara.

### Kekurangan

-   **Kompleksitas Parameter Tuning**: Sulit menentukan angka threshold yang ideal (berapa persentase error yang memicu OPEN? berapa detik cooldown?).
-   **Kebutuhan Logika Fallback**: Setiap pemanggilan service harus disiapkan alternatifnya (misal: mengambil data dari cache lama, atau mengembalikan nilai default).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Seluruh panggilan sinkron antar microservice yang saling bergantung.
-   Panggilan ke API pihak ketiga (Payment Gateway, Kurir Logistik, SMS OTP).
-   Integrasi ke database eksternal atau distributed cache.

### Kapan Jangan Pakai

-   Panggilan fungsi internal dalam memori lokal (_in-process memory function call_).
-   Komunikasi asynchronous berbasis message queue (karena queue sudah memiliki mekanisme backpressure dan dead-letter queue bawaan).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`circuit_breaker_state` (Gauge)**: Nilai diskrit 0=Closed, 1=Half-Open, 2=Open. Buat alert PagerDuty prioritas tinggi jika bernilai 2 (Open) selama > 1 menit.
-   **`circuit_breaker_trip_total` (Counter)**: Frekuensi seberapa sering sirkuit jebol.
-   **`fallback_invocations_total` (Counter)**: Jumlah panggilan yang dialihkan ke fungsi fallback darurat.

## 9. Studi Kasus Nyata di Industri

-   **Netflix (Hystrix)**: Netflix menciptakan library legendaris **Hystrix** pada tahun 2011 setelah insiden outage besar di mana satu bug kecil di service bookmark metadata merembet mengunci seluruh thread pool tomcat, melumpuhkan pemutaran video jutaan pengguna di seluruh dunia. Hystrix mengisolasi setiap dependency call dalam thread pool atau semaphore circuit breaker terpisah.
-   **FinTech Payment Gateways**: Gerbang pembayaran modern membungkus integrasi perbankan dengan circuit breaker. Jika koneksi API salah satu bank BUMN mengalami gangguan jaringan, circuit breaker secara otomatis terbuka dan sistem langsung mengalihkan rute transaksi pengguna ke bank alternatif (_smart payment rerouting_).

## 10. Fakta Menarik & Gotchas Production

-   **Absolute Count vs Error Percentage Trap**: Jangan pernah mengonfigurasi circuit breaker hanya berdasarkan _jumlah absolut error_ (misal: "buka sirkuit jika 10 error terjadi"). Jika trafik sedang tinggi (10.000 req/sec), 10 error itu hanya 0.1% (anomali wajar), tetapi sirkuit malah terbuka dan memutus layanan! Selalu gunakan kombinasi: **Minimal Request Volume** (misal: minimal 50 request dalam 10 detik) DAN **Error Rate Percentage** (misal: > 50% error).
-   **Graceful Fallback Bukan Berarti Silent Bug**: Seringkali tim membuat fallback yang terlalu "ramah" (misal: return status kosong) tanpa mengirim alert ke Prometheus/Datadog. Akibatnya, circuit breaker dalam status OPEN selama 3 hari tanpa ada engineer yang sadar bahwa downstream service-nya mati! Selalu pantau metrik status circuit breaker di dashboard observability.
