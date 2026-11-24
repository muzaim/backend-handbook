# Timeouts

## 1. Skenario Sistem Nyata

Pada arsitektur microservices pembayaran e-commerce, `Order Service` bertugas memanggil API eksternal `Bank Payment Gateway` melalui jaringan publik HTTP untuk memvalidasi kartu kredit nasabah.

Jika client HTTP backend tidak memasang batas timeout eksplisit (menggunakan default sistem operasi yang bisa mencapai berjam-jam):

-   Ketika jaringan partner bank mengalami gangguan atau silent packet drop, koneksi TCP menggantung tanpa kejelasan.
-   Setiap request pembayaran yang masuk akan mengunci satu worker thread / goroutine di `Order Service`.
-   Dalam waktu 30 detik pada trafik tinggi, seluruh 200 connection worker threads habis terkunci menunggu respon bank (_thread pool exhaustion_).
-   Dampak fatalnya: `Order Service` lumpuh total dan tidak bisa lagi melayani endpoint lain (bahkan untuk melihat riwayat pesanan biasa), menyebabkan kegagalan berantai (_cascading failure_) ke seluruh platform.

**Solusi dengan Timeout (misal: Connect Timeout 500ms, Read Timeout 2.5s):**
Kamu menetapkan batas waktu maksimal 2.5 detik menggunakan `context.WithTimeout` di kode HTTP client backend. Jika partner bank tidak membalas dalam 2.5 detik, koneksi langsung diputus secara paksa, worker thread segera dibebaskan kembali ke pool, dan sistem dengan cepat mengembalikan respons error terkontrol atau menyarankan pengguna memilih metode pembayaran lain.

## 2. Apa Itu dan Buat Apa?

Timeout adalah aturan batas waktu maksimal yang ditentukan oleh pemanggil (_caller_) untuk menunggu respon dari operasi jaringan atau I/O (seperti HTTP call ke downstream microservice, query database, atau pembacaan file).

Jika operasi tidak selesai dalam batas waktu yang ditentukan, proses akan **dihentikan secara paksa**, error dilempar (misal `Context Deadline Exceeded` atau `SocketTimeoutException`), dan alokasi resource segera dibebaskan.

Jenis-jenis timeout penting dalam backend:

-   **Connection Timeout**: Batas waktu untuk menyelesaikan _TCP Handshake_ saat pertama kali membuka koneksi ke server tujuan (biasanya singkat: 500ms - 2 detik).
-   **Read/Socket Timeout**: Batas waktu menunggu data masuk setelah koneksi terbentuk (misal: 3 - 5 detik).
-   **Write Timeout**: Batas waktu untuk mengirim seluruh payload data keluar melalui socket.
-   **Request/Global Timeout**: Batas waktu menyeluruh dari saat request pertama kali masuk hingga response akhir dikirimkan ke client.

## 3. Kenapa Harus Pakai Ini ?

Di sistem skala besar, musuh paling mematikan bukanlah server downstream yang mati total (_hard crash_), melainkan server downstream yang **merespons sangat lambat (_slow degradation_)**.

Ketika downstream service membutuhkan waktu 60 detik untuk membalas sebuah request:

-   Setiap request yang masuk ke API-mu akan mengunci 1 thread atau goroutine, 1 file descriptor socket, dan sejumlah memori selama 60 detik penuh.
-   Jika trafikmu 500 request per detik, dalam waktu 10 detik sistemmu akan menumpuk 5,000 thread yang macet (_hung threads_).
-   Server kehabisan memory, connection pool kehabisan slot, dan akhirnya seluruh aplikasi ikut tumbang secara berantai (_cascading failure_).

Dengan menyetel timeout yang ketat (misal 2 detik), resource akan segera dilepas kembali ke pool, dan server tetap sehat melayani request lainnya.

## 4. Cara Kerjanya (Diagram Alur)

```
[ Tanpa Timeout: Thread Macet Selamanya ]
Client -----> [ Service A (Thread Terkunci) ] ===== Tunggu 60 Detik =====> [ Service B (Lag) ]
Client -----> [ Service A (Thread Terkunci) ] ===== Tunggu 60 Detik =====> [ Service B (Lag) ]
(Akumulasi ribuan thread hingga Service A Crash kehabisan Memori)


[ Dengan Timeout: Fail-Fast & Resource Dilepas ]
Client -----> [ Service A ]
                    |
                    | 1. Panggil Service B (Batas Timeout: 2 Detik)
                    v
              [ Timer Berjalan: 0s ... 1s ... 2s ]
                    |
                    +-- Service B belum membalas setelah 2 Detik!
                    |
                    v
              [ TIMEOUT TRIGGERED ]
                    |
                    | 2. Putus Socket, Lepas Thread
                    | 3. Kembalikan Fallback / Error 504 Gateway Timeout
                    v
Client <----- [ Response Cepat: "Layanan Sedang Sibuk" ]
```

## 5. Cuplikan Logika Inti (Konfigurasi Timeout HTTP Client di Go)

Berikut implementasi penyetelan timeout bertingkat yang benar pada HTTP Client backend menggunakan Go Context:

```go
package main

import (
	"bytes"
	"context"
	"net/http"
	"time"
)

// CallDownstreamWithTimeout menerapkan timeout ketat pada level HTTP Client dan Context
func CallDownstreamWithTimeout(ctx context.Context, payload []byte) (*http.Response, error) {
	// 1. Batas timeout operasi keseluruhan via Context
	reqCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
	defer cancel()

	req, err := http.NewRequestWithContext(reqCtx, http.MethodPost, "http://payment-service:8080/charge", bytes.NewBuffer(payload))
	if err != nil {
		return nil, err
	}
	req.Header.Set("Content-Type", "application/json")

	// 2. Client HTTP dengan timeout koneksi bertingkat
	client := &http.Client{
		Timeout: 2500 * time.Millisecond,
		Transport: &http.Transport{
			ResponseHeaderTimeout: 1500 * time.Millisecond,
			TLSHandshakeTimeout:   1000 * time.Millisecond,
		},
	}

	return client.Do(req)
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Fail-Fast**: Menghindari pemborosan waktu pengguna dan thread server.
-   **Mencegah Efek Domino**: Mengisolasi kelambatan di satu komponen agar tidak merembet ke seluruh ekosistem backend.
-   **Prediktabilitas SLA**: Menjamin respon API selalu kembali dalam batas waktu yang telah dijanjikan ke pengguna.

### Kekurangan

-   **False Negative**: Operasi yang sebenarnya sedang berhasil diselesaikan di backend downstream bisa terlanjur dibatalkan oleh pemanggil.
-   **Inkonsistensi Data (Ghost Writes)**: Client menganggap request gagal karena timeout, padahal database di server downstream berhasil menyimpan transaksi setelah detik ke-3.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   **Setiap panggilan jaringan eksternal tanpa terkecuali**: Panggilan HTTP ke 3rd party API, query Database, Redis call, dan RPC internal.
-   Di level Reverse Proxy dan API Gateway (`proxy_read_timeout` di NGINX).

### Kapan Jangan Pakai (atau butuh durasi sangat longgar)

-   Proses komputasi asynchronous / background batch job (misal: export laporan keuangan bulanan berukuran 10GB yang memang butuh waktu 30 menit). Gunakan Message Queue untuk kasus ini, bukan synchronous HTTP!

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`http_client_timeout_errors_total` (Counter)**: Frekuensi kejadian timeout ke dependency downstream tertentu.
-   **`request_duration_seconds` (Histogram p95/p99)**: Jika grafik p99 mulai mendekati garis batas timeout (misal menyentuh 2.3 detik pada timeout 2.5 detik), downstream sedang mengalami degradasi serius.
-   **`thread_pool_active_workers` (Gauge)**: Memantau lonjakan worker yang tertahan saat timeout terjadi.

## 9. Studi Kasus Nyata di Industri

-   **Amazon Web Services (AWS SDK)**: AWS mendesain seluruh client SDK resminya dengan default socket read timeout yang sangat agresif (sering kali di bawah 1 detik untuk internal call DynamoDB). Amazon menemukan bahwa membatalkan request yang lambat dan mengirimkan request baru ke node replika lain jauh lebih cepat daripada menunggu node yang lambat pulih.
-   **Google (gRPC Context Deadlines)**: Google mewajibkan penggunaan _Deadlines_ di protokol gRPC mereka. Nilai deadline waktu ini diteruskan secara otomatis (_propagated_) melintasi puluhan service microservices. Jika waktu sisa tinggal 100ms saat request tiba di Service ke-5, Service ke-5 tidak akan membuang CPU mengeksekusi operasi yang dipastikan akan timeout di sisi user.

## 10. Fakta Menarik & Gotchas Production

-   **Default Timeout Is Dangerously Infinite**: Di banyak bahasa pemrograman populer (seperti `http.Client` bawaan Go atau `requests` bawaan Python), default timeout koneksi HTTP adalah **NOL (artinya UNLIMITED/tidak ada batas waktu)**! Jika downstream tidak pernah membalas, aplikasi akan menunggu selamanya sampai proses di-kill oleh OS. Selalu set timeout secara eksplisit!
-   **Timeout Inversion (Bencana Timeout Terbalik)**: Ini adalah bug arsitektur paling sering terjadi di microservices.  
    Misal: Client Frontend timeout = 3 detik. API Gateway timeout = 5 detik. Downstream Service timeout = 10 detik.  
    _Akibatnya:_ Pada detik ke-3, browser pengguna sudah menutup koneksi dan menyerah. Namun API Gateway dan Downstream Service tetap membakar CPU dan I/O sampai detik ke-10 mengerjakan pekerjaan yang hasilnya sudah tidak dipedulikan oleh siapa pun (_wasted compute_). Aturan emasnya: **Timeout di layer terdalam harus selalu lebih pendek daripada timeout di layer terluar!**
