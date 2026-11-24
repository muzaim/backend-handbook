# Webhooks (HTTP Callbacks)

## 1. Skenario Sistem Nyata

Pada sebuah platform SaaS dengan 100.000 pelanggan aktif, sistem mengintegrasikan gerbang pembayaran (_Payment Gateway_, misal Stripe atau Xendit) untuk memproses pembayaran invoice bulanan.

Jika backend aplikasi memeriksa status pembayaran menggunakan metode tradisional **Polling**:

-   Backend harus menjalankan proses background yang secara berulang memanggil API Payment Gateway setiap 10 detik:  
    `GET https://api.gateway.com/v1/invoices/inv_99812/status` untuk puluhan ribu invoice yang belum lunas.
-   Ini menghasilkan jutaan panggilan API yang sia-sia setiap hari, memboroskan kuota _rate limit_, dan membebani server dengan 99% respon yang masih berstatus `"PENDING"`.

**Solusi dengan Webhooks (Reverse API / Event Push):**
Backend aplikasi mendaftarkan endpoint publik: `POST https://api.saas.com/webhooks/payments`.
Alih-alih backend kita yang terus-menerus bertanya, Payment Gateway secara proaktif "menelepon" endpoint backend kita dengan mengirimkan payload HTTP POST berisi detail event sesaat setelah pelanggan berhasil menyelesaikan pembayaran di bank:

```json
{
	"event": "invoice.payment_succeeded",
	"data": {
		"invoice_id": "inv_99812",
		"amount": 250000,
		"paid_at": 1732440000
	}
}
```

Status akun pelanggan langsung aktif seketika dalam hitungan milidetik tanpa membuang sumber daya komputasi dan bandwidth jaringan.

## 2. Apa Itu dan Buat Apa?

Webhook adalah pola komunikasi antar-sistem berbasis arsitektur _event-driven_ di mana sistem pengirim (producer) mengirimkan data ke sistem penerima (consumer) melalui panggilan HTTP POST segera setelah suatu kejadian bisnis (_event_) terjadi. Sering disebut juga sebagai **Reverse API** atau **HTTP Callback**.

Karakteristik utama:

-   **Push vs Pull**: Menggantikan mekanisme polling yang lambat dan boros dengan notifikasi dorong (_push_) seketika.
-   **Payload Berformat Standar**: Biasanya dikirim dalam format JSON bersama header metadata (seperti ID event, timestamp, dan signature keamanan).
-   **Asinkron & Terdistribusi**: Menjadi standar industri untuk komunikasi antar-organisasi yang berbeda (misal GitHub, Stripe, Shopify, Slack, Twilio).

## 3. Kenapa Harus Pakai Ini ?

Dalam skala sistem tinggi, webhook memberikan efisiensi yang dramatis:

1. **Mengeliminasi Beban Polling yang Sia-Sia**: Menghemat hingga 95% panggilan jaringan tak berguna, membebaskan thread server dan connection pool.
2. **Latensi Notifikasi Near Zero**: Perubahan status transaksi diterima dalam rentang milidetik setelah kejadian selesai di sistem pihak ketiga.
3. **Decoupled Architecture**: Sistem pihak ketiga tidak perlu mengetahui struktur basis data internal kita; mereka cukup mengirimkan kontrak payload JSON ke URL yang disepakati.

## 4. Cara Kerjanya (Diagram Alur Webhook)

```
+-------------------+                          +-------------------+
|  Payment Gateway  |                          |  Backend Aplikasi |
|  (Event Producer) |                          | (Webhook Consumer)|
+-------------------+                          +-------------------+
          |                                              |
    [Event Muncul]                                       |
  (Pembayaran Sukses)                                    |
          |                                              |
 1. Generate Signature                                   |
    (HMAC-SHA256 Secret)                                 |
          |                                              |
 2. HTTP POST /webhooks/payments ----------------------> |
    Header: X-Signature, X-Timestamp                     |
    Payload: {"event": "payment.success"}                |
          |                                       3. Verifikasi Signature
          |                                          & Timestamp
          |                                              |
          |                                       4. Balas 200 OK Cepat
          | <------------------------------------    (< 200 ms)
          |                                              |
          |                                       5. Masukkan Payload ke
          |                                          Internal Queue
          |                                          (Redis / RabbitMQ)
          |                                              |
          |                                              v
          |                                   [Background Worker Go]
          |                                   (Update Status Akun di DB)
```

## 5. Cuplikan Logika Inti (Webhook Receiver dengan Verifikasi HMAC di Go)

Endpoint penerima webhook di production wajib memverifikasi tanda tangan kriptografis (_HMAC SHA-256_) untuk memastikan request berasal dari pihak resmi, serta langsung mengembalikan HTTP 200 OK sebelum memproses data di background:

```go
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"io"
	"log"
	"net/http"
	"time"
)

const webhookSecret = "whsec_super_secret_signing_key_production"

type WebhookPayload struct {
	EventID   string `json:"event_id"`
	EventType string `json:"event_type"`
	Data      any    `json:"data"`
}

// verifySignature memvalidasi bahwa payload tidak dimanipulasi oleh pihak ketiga (MitM)
func verifySignature(payload []byte, signatureHeader string, secret string) bool {
	mac := hmac.New(sha256.New, []byte(secret))
	mac.Write(payload)
	expectedSignature := hex.EncodeToString(mac.Sum(nil))

	// Menggunakan hmac.Equal untuk mencegah timing attack
	return hmac.Equal([]byte(signatureHeader), []byte(expectedSignature))
}

func WebhookHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Metode tidak diizinkan", http.StatusMethodNotAllowed)
		return
	}

	// 1. Baca seluruh payload body
	body, err := io.ReadAll(r.Body)
	if err != nil {
		http.Error(w, "Gagal membaca body", http.StatusBadRequest)
		return
	}
	defer r.Body.Close()

	// 2. Verifikasi tanda tangan HMAC dari header
	signature := r.Header.Get("X-Hub-Signature-256")
	if !verifySignature(body, signature, webhookSecret) {
		log.Println("[SECURITY WARNING] Tanda tangan webhook tidak valid!")
		http.Error(w, "Unauthorized: invalid signature", http.StatusUnauthorized)
		return
	}

	// 3. Parse JSON payload
	var event WebhookPayload
	if err := json.Unmarshal(body, &event); err != nil {
		http.Error(w, "Payload JSON tidak valid", http.StatusBadRequest)
		return
	}

	// 4. POLA KRUSIAL: Langsung balas HTTP 200 OK secepat mungkin (< 200 ms)
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`{"status": "accepted"}`))

	// 5. Lempar pemrosesan ke background goroutine / task queue terpisah
	go func(evt WebhookPayload) {
		processWebhookAsync(evt)
	}(event)
}

func processWebhookAsync(evt WebhookPayload) {
	log.Printf("[Worker] Memproses event %s (ID: %s) secara asynchronous", evt.EventType, evt.EventID)
	// Lakukan operasi database, update saldo, atau kirim email di sini
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Efisiensi Jaringan Maksimal**: Meniadakan ribuan request polling yang tidak menghasilkan data baru.
-   **Near-Instant Synchronization**: Notifikasi data tiba tepat saat kejadian selesai dieksekusi.
-   **Standar Ekosistem Global**: Didukung secara universal oleh ribuan platform pihak ketiga di seluruh dunia.

### Kekurangan

-   **Risiko Kegagalan Pengiriman (_Delivery Failure_)**: Jika server penerima sedang down saat webhook dikirim, event bisa hilang jika pengirim tidak memiliki sistem retry yang baik.
-   **Tuntutan Idempotensi Mutlak**: Pengirim webhook biasanya menerapkan aturan _At-Least-Once Delivery_, sehingga satu event bisa terkirim 2 atau 3 kali saat terjadi timeout jaringan semu.
-   **Paparan Keamanan Publik**: Endpoint webhook harus terbuka ke internet publik, sehingga rentan terhadap serangan DDoS dan serangan pemalsuan payload (_replay attack_).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Integrasi antar-organisasi atau antar-perusahaan yang berbeda domain jaringan (Payment Gateway, kurir logistik, notifikasi SMS/WhatsApp, build CI/CD GitHub).
-   Notifikasi pembaruan status yang kejadiannya tidak dapat diprediksi waktunya (misal: kapan customer mentransfer dana ke Virtual Account).
-   Komunikasi asynchronous antar-layanan publik yang tidak berada di dalam satu Virtual Private Cloud (VPC).

### Kapan Jangan Pakai

-   Komunikasi internal antar microservices di dalam satu cluster privat Kubernetes yang sama (lebih cepat dan aman menggunakan gRPC atau Message Broker internal seperti Kafka/RabbitMQ).
-   Komunikasi synchronous di mana user di browser menunggu respon langsung seketika (_blocking request-response_).
-   Aliran data bervolume ekstrem kontinu (misal 50.000 log per detik); koneksi HTTP webhook individual akan membebani overhead TCP handshake.

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`webhook_delivery_success_rate` (Gauge / Persentase)**: Persentase webhook yang menghasilkan status 2xx. Target SLA produksi adalah **> 99.9%**.
-   **`webhook_response_latency_ms` (Histogram)**: Waktu yang dibutuhkan endpoint consumer untuk membalas respon HTTP. Wajib di bawah 500 ms agar pengirim tidak menganggapnya _timeout_.
-   **`webhook_retries_total` (Counter)**: Jumlah percobaan ulang pengiriman webhook yang dipicu akibat kegagalan koneksi atau error 5xx.
-   **`webhook_signature_failures_total` (Counter)**: Lonjakan kegagalan verifikasi tanda tangan dapat mengindikasikan serangan brute force atau konfigurasi secret yang kedaluwarsa.

## 9. Studi Kasus Industri Nyata

-   **Stripe**: Mengirimkan miliaran notifikasi webhook pembayaran setiap hari ke jutaan developer di seluruh dunia. Stripe merancang infrastruktur webhook tangguh yang otomatis mencoba ulang (_exponential backoff retry_) hingga 72 jam jika server pelanggan mengalami kegagalan, dilengkapi dashboard monitoring interaktif di mana developer dapat memeriksa log payload dan melakukan _replay event_ manual.
-   **GitHub**: Menjadi tulang punggung integrasi DevOps global. Setiap kali ada commit atau Pull Request yang di-push ke repository, GitHub Webhooks menembakkan event ke sistem CI/CD (seperti Jenkins, CircleCI, atau GitHub Actions runner) untuk memicu proses build dan testing otomatis.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Memproses Pekerjaan Berat Secara Sinkron Sebelum Balas 200 OK**: Melakukan kalkulasi database yang berat atau pemanggilan API pihak ketiga di dalam handler webhook utama. Gateway pengirim biasanya memiliki timeout 5 detik; jika melebihi itu, mereka menganggap request gagal dan akan mengirim ulang event yang sama berkali-kali!  
   _Solusi_: Validasi signature, simpan payload ke antrean (Queue/Redis), langsung balas 200 OK, lalu proses data secara asynchronous.
2. **Mengabaikan Idempotensi**: Menerima webhook `payment.succeeded` yang sama dua kali karena kendala koneksi sementara pada pengirim. Tanpa pengecekan idempotensi (misal tabel `processed_events`), saldo pengguna akan terisi dua kali lipat!
3. **Menerima Webhook Tanpa Validasi Signature**: Menerima request POST begitu saja tanpa memverifikasi HMAC secret atau timestamp. Penyerang dapat menembak endpoint `/webhooks/payments` secara manual dan memalsukan event bahwa akun mereka sudah membayar langganan.

## 11. Rangkuman Singkat

Webhook adalah mekanisme notifikasi asinkron berbasis HTTP POST proaktif (_Reverse API_) yang menggantikan polling boros. Untuk skala produksi, webhook menuntut verifikasi tanda tangan kriptografis (HMAC), respon HTTP 200 OK secepat kilat melalui pemrosesan background, serta penanganan idempotensi mutlak terhadap duplikasi event.
