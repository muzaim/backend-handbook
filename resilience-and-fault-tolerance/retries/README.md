# Retries

## 1. Skenario Sistem Nyata

Pada ekosistem microservices logistik, `Order Fulfillment Service` secara berkala memanggil API eksternal mitra pengiriman untuk menerbitkan nomor resi paket belanjaan.

Dalam komunikasi jaringan cloud terdistribusi, gangguan jaringan singkat (_transient network glitches_) adalah hal yang lumrah:

-   Server partner logistik mungkin sedang melakukan restart pod (berdurasi 2 detik), switch router cloud mengalami micro-loss, atau koneksi TCP terputus sesaat (_Connection Reset by Peer_) sehingga request pertamamu merespons `HTTP 503 Service Unavailable`.
-   Tanpa Retries: Sistem langsung menganggap pesanan gagal permanen. Pengiriman barang terhenti, status pesanan macet, dan tim operasional harus turun tangan membuat resi secara manual.
-   Namun sebaliknya, jika server partner merespons `HTTP 400 Bad Request` (format alamat salah) atau `HTTP 401 Unauthorized` (API key kedaluwarsa), mencoba ulang request berkali-kali tidak akan pernah berhasil dan hanya membuang bandwidth.

**Solusi dengan Retries Terukur:**
Kamu menerapkan kebijakan retry otomatis: jika request gagal akibat transient error (`502`, `503`, `504`, atau TCP drop), client backend otomatis melakukan percobaan ulang hingga 3 kali dengan jeda singkat. Percobaan kedua di detik berikutnya langsung berhasil, nomor resi terbit, dan pengguna mendapatkan status pesanan mereka secara mulus (_self-healing API interaction_).

## 2. Apa Itu dan Buat Apa?

Retries adalah mekanisme di mana sebuah client atau service secara otomatis mengulang kembali eksekusi request yang gagal, dengan harapan kegagalan tersebut disebabkan oleh masalah jaringan atau server yang bersifat sementara (_transient faults_).

Jenis-jenis error yang **LAYAK** di-retry (_Transient Errors_):

-   `503 Service Unavailable` (server sedang sibuk sesaat).
-   `504 Gateway Timeout` atau Socket Read Timeout.
-   Network connection reset (`ECONNRESET` / TCP dropped packet).
-   Deadlock database sesaat (_concurrency serialization conflict_).

Jenis-jenis error yang **TIDAK BOLEH** di-retry (_Permanent/Non-Transient Errors_):

-   `400 Bad Request` (payload JSON salah format, diulang 1.000 kali pun akan tetap salah).
-   `401 Unauthorized` atau `403 Forbidden` (kredensial login salah atau tidak punya hak akses).
-   `404 Not Found` (endpoint atau data tidak ada).
-   `422 Unprocessable Entity` (validasi bisnis gagal, misal saldo tidak mencukupi).

## 3. Kenapa Harus Pakai Ini ?

Dalam sistem terdistribusi skala besar (cloud computing), jaringan bukanlah kabel tembaga yang sempurna. Ada prinsip terkenal: _"The Network is NOT Reliable"_.

Di balik layar arsitektur high-throughput:

-   Container Kubernetes sering berpindah node (_pod rescheduling_).
-   Router cloud melakukan failover rute jaringan.
-   Load balancer sesekali membuang koneksi TCP lama (_idle connection purge_).

Jika aplikasi langsung menyerah dan menampilkan error ke layar pengguna pada kegagalan pertama yang berdurasi 10 milidetik, angka _availability_ sistemmu akan buruk. Dengan retry pintar sebanyak 1-2 kali, 90% gangguan mikro di jaringan dapat diatasi secara transparan tanpa disadari oleh pengguna.

## 4. Cara Kerjanya (Diagram Alur)

```
Client / Service A                     Service B (Target)
      |                                        |
      | 1. HTTP Request (Attempt 1)            |
      |--------------------------------------->| (Sedang restart pod)
      | <--------------------------------------|
      | 2. Error: 503 / Connection Refused     |
      |                                        |
 [ Cek Error Type ]                            |
   Apakah Transient? -> YA                     |
   Apakah Kuota Retry Masih Ada? -> YA (1/3)   |
      |                                        |
  [ Jeda Waktu / Backoff ]                     |
      |                                        |
      | 3. HTTP Request (Attempt 2)            |
      |--------------------------------------->| (Pod sudah aktif kembali!)
      | <--------------------------------------|
      | 4. HTTP 200 OK + Data                  |
      v                                        v
[ Transaksi Sukses ]
```

## 5. Cuplikan Logika Inti (Smart Retry Loop di Go)

Berikut implementasi logika retry cerdas dengan penyaringan status code yang tepat di Go:

```go
package main

import (
	"context"
	"errors"
	"net/http"
	"time"
)

// ExecuteWithRetry menjalankan request HTTP dengan filter status code yang tepat
func ExecuteWithRetry(ctx context.Context, reqFn func() (*http.Response, error), maxRetries int) (*http.Response, error) {
	var resp *http.Response
	var err error

	for attempt := 1; attempt <= maxRetries; attempt++ {
		resp, err = reqFn()

		// Jika sukses dan status code 2xx atau 4xx (client error), jangan di-retry
		if err == nil && resp.StatusCode < 500 {
			return resp, nil
		}

		// Hanya retry jika error jaringan (I/O timeout) atau server error 502/503/504
		if resp != nil && (resp.StatusCode == http.StatusBadGateway ||
			resp.StatusCode == http.StatusServiceUnavailable ||
			resp.StatusCode == http.StatusGatewayTimeout) {
			// Status sementara, boleh retry
		} else if err == nil {
			return resp, nil // Error 500 biasa atau status lainnya, hentikan retry
		}

		if attempt < maxRetries {
			select {
			case <-time.After(time.Duration(attempt*100) * time.Millisecond):
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		}
	}

	if err != nil {
		return nil, err
	}
	return nil, errors.New("request gagal setelah batas maksimal retry")
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Self-Healing Otomatis**: Menyerap fluktuasi jaringan mikro tanpa intervensi manusia.
-   **Peningkatan SLA Sukses**: Mengubah request yang awalnya gagal menjadi sukses di balik layar.

### Kekurangan

-   **Bahaya "Retry Storm" (Badai Retry)**: Jika service tujuan sedang tumbang karena kelebihan beban, dan 10.000 client serempak melakukan 3x retry, trafik yang menghantam server akan melonjak 300% (dari 10k menjadi 40k req/sec), memastikan server tujuan mati total dan tidak bisa bangkit kembali.
-   **Latensi Menumpuk**: Melakukan retry berkali-kali memperpanjang waktu tunggu pengguna akhir.
-   **Bahaya Eksekusi Ganda**: Jika request penarikan uang yang timeout di-retry tanpa pengaman, uang pengguna bisa terdebit dua kali.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Hanya pada operasi yang bersifat **Idempotent** (operasi yang jika dijalankan berkali-kali menghasilkan hasil yang sama persis, misal: `GET`, `PUT`, `DELETE`, atau `POST` yang dilengkapi _Idempotency-Key_).
-   Pada komunikasi asynchronous antar background worker atau message queue consumers.

### Kapan Jangan Pakai

-   Pada request **Non-Idempotent** yang belum memiliki idempotency key (misal: endpoint `/api/v1/charge-credit-card` atau `/api/v1/orders/checkout`).
-   Saat downstream service mengembalikan error validasi bisnis atau client error (`4xx`).
-   Ketika sisa waktu request timeout global pemanggil sudah hampir habis.

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`http_client_retries_total` (Counter)**: Total operasi retry yang dipicu di dalam sistem.
-   **`retry_success_ratio` (Gauge)**: Persentase retry yang akhirnya berhasil (`Successful Retries / Total Retries`). Jika angka ini < 10%, berarti downstream service sedang benar-benar mati, bukan mengalami gangguan transient.
-   **`retry_budget_exhausted_total` (Counter)**: Jumlah transaksi yang dibatalkan karena kuota retry habis.

## 9. Studi Kasus Nyata di Industri

-   **Envoy Proxy (Lyft & Istio)**: Envoy menyematkan fitur canggih bernama **Retry Budgets**. Aturan default-nya mencegah client mengirim retry jika rasio retry melebihi 20% dari total trafik normal. Jika downstream sedang limbung, Envoy otomatis menahan retry untuk melindungi service hilir.
-   **Stripe**: SDK resmi Stripe secara otomatis melakukan retry hingga 2 kali saat mendapati gangguan jaringan atau status HTTP 500/503. Stripe mewajibkan SDK menyertakan header `Idempotency-Key` pada setiap request awal sehingga retry otomatis tersebut 100% aman dan tidak pernah mendebit saldo pelanggan dua kali.

## 10. Fakta Menarik & Gotchas Production

-   **The Legendary Retry Storm Disaster**: Di production, penyebab utama server downstream gagal pulih (_fail to recover_) setelah restart adalah _Retry Storm_. Ketika database sempat mati 5 detik, semua aplikasi serempak mengirimkan jutaan retry tanpa jeda. Solusi wajib: **Batasi jumlah maksimal retry (maksimal 2 atau 3 kali)** dan selalu padukan dengan **Exponential Backoff dengan Jitter** serta **Circuit Breaker**.
-   **Retry Budgeting**: Arsitektur modern skala raksasa (seperti yang digunakan Google dan Envoy Proxy) menggunakan konsep _Retry Budget_. Aturannya: client hanya boleh mengalokasikan maksimal 10% dari total trafiknya untuk operasi retry. Jika dalam 1 menit persentase retry sudah melebihi 10%, hentikan seluruh retry dan langsung gagalkan request agar sistem downstream tidak tenggelam.
