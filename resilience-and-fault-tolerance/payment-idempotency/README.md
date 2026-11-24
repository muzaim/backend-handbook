# Idempotency Key & Payment Deduplication (Transactional Boundary)

## 1. Analogi Sederhana & Skenario Nyata

Kalian sedang berbelanja di minimarket dan menggesek kartu debit di mesin EDC kasir. Tepat ketika kalian menekan tombol **Enter PIN**, layar mesin EDC tiba-tiba macet (blank/loading lama) karena sinyal seluler kasir mendadak putus. Kasir merasa transaksi belum masuk dan meminta kalian menggesek kartu kembali untuk kedua kalinya. 

Jika bank penerbit kartu tidak memiliki sistem **Idempotency Key & Deduplication**, rekening kalian akan langsung terpotong dua kali untuk satu kantong belanjaan yang sama. Namun, pada sistem pembayaran modern, transaksi pertama telah dilekatkan nomor unik transaksi (*RRN - Retrieval Reference Number* atau *Idempotency Key*). Ketika kasir mencoba menggesek kartu dengan nomor referensi yang sama, sistem perbankan mengenali transaksi tersebut, menolak pemotongan ganda, dan langsung mengembalikan struk bukti transaksi pertama yang sebelumnya sukses.

Di backend engineering, fenomena ini terjadi setiap detik: koneksi internet pengguna mobile mendadak beralih dari Wi-Fi ke 4G saat checkout, library HTTP client melakukan *automatic retry* saat mengalami timeout, atau pengguna menekan tombol "Bayar Sekarang" berulang kali secara agresif.

---

## 2. Definisi & Konsep Inti

**Idempotency** adalah sifat dari suatu operasi API di mana pemanggilan operasi tersebut satu kali atau berkali-kali dengan parameter identik akan menghasilkan efek samping (*side-effect*) status sistem yang sama persis pada server.

Pada protokol HTTP:
- `GET`, `PUT`, dan `DELETE` secara spesifikasi bersifat idempoten.
- `POST` dan `PATCH` secara default **tidak idempoten** (membuat resource baru setiap kali dipanggil).

Untuk membuat endpoint mutasi pembayaran (`POST /v1/payments/charges`) aman dari pemotongan ganda, industri menggunakan mekanisme **Idempotency Key**:
1. **Idempotency-Key Header**: Client meng-generate UUIDv4 unik di sisi frontend/mobile app sebelum mengirim request dan menyematkannya pada header `Idempotency-Key: 9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d`.
2. **Payload Fingerprinting (SHA-256)**: Backend menghitung hash dari body request bersama URL path dan User ID untuk memastikan client tidak menggunakan kunci yang sama untuk transaksi yang berbeda (*payload tampering*).
3. **Distributed State Machine**: Kunci dikelola menggunakan status siklus hidup transaksi:
   - `STARTED` / `IN_PROGRESS`: Kunci sedang dikunci secara atomik (misal via Redis `SETNX`) dan transaksi database sedang berjalan.
   - `SUCCESS`: Transaksi database commit, pembayaran eksternal sukses, dan response snapshot (status code + response body) disimpan di cache/DB.
   - `FAILED`: Transaksi gagal secara permanen atau dibatalkan, kunci dibuka atau ditandai gagal agar client dapat mencoba kembali dengan payload yang benar.
4. **Cached Response Replay**: Jika request dengan kunci yang sama masuk saat status sudah `SUCCESS`, backend **tidak mengeksekusi ulang transaksi**, melainkan langsung mengembalikan salinan respons sebelumnya (*cached response replay*).

---

## 3. Mengapa Sangat Penting di Skala Produksi

Pada skala puluhan ribu transaksi per detik (*TPS*) di sistem fintech dan e-commerce:

1. **Mencegah Double-Charging & Kerugian Finansial**: Tanpa deduplikasi, retry otomatis akibat network spike atau connection drop akan menagih kartu kredit atau memotong saldo dompet digital customer berulang kali, memicu sengketa chargeback dan denda regulasi perbankan.
2. **Perlindungan Terhadap Double-Disbursement**: Pada sistem penggajian atau penarikan dana mitra driver, kegagalan retry bisa mentransfer uang riil keluar dari rekening perusahaan dua kali tanpa bisa ditarik kembali secara otomatis.
3. **Mencegah Double-Booking Inventory**: Menghindari alokasi stok tiket pesawat, kamar hotel, atau barang flash sale berkurang dua kali untuk satu pengguna yang sama.
4. **Menangani Skenario Jaringan Tidak Pasti (Two Generals' Problem)**: Pengirim tidak pernah tahu apakah timeout terjadi saat request sedang menuju server, saat server sedang memproses database, atau saat respons balasan sedang dikirim kembali ke client. Idempotency key menyelesaikan ketidakpastian ini di sisi server.

---

## 4. Cara Kerja & Diagram Alur (ASCII)

Berikut adalah siklus hidup penanganan Idempotency Key pada transaksi pembayaran:

```
Client (App/Web)                API Gateway / Backend Service              Redis Lock / PostgreSQL
      |                                       |                                       |
      |-- 1. POST /charges ------------------>|                                       |
      |   (Header: Idempotency-Key: <UUID>)   |                                       |
      |   (Body: {"amount": 50000})           |-- 2. Compute SHA-256(Body) ---------->|
      |                                       |                                       |
      |                                       |-- 3. Atomic SETNX key:lock (TTL 60s)->|
      |                                       |<-- Lock Acquired (Success) -----------|
      |                                       |                                       |
      |                                       |-- 4. Check DB for existing key ------>|
      |                                       |<-- Not found (Brand New Request) -----|
      |                                       |                                       |
      |                                       |-- 5. INSERT (key, hash, 'STARTED')--->|
      |                                       |                                       |
      |                                       |-- 6. Execute Payment & DB Debit ----->|
      |                                       |-- 7. UPDATE (key, 'SUCCESS', resp)--->|
      |                                       |-- 8. Release Redis Lock ------------->|
      |                                       |                                       |
      |<-- 9. HTTP 201 Created ---------------|                                       |
      |    (Body: {"charge_id": "ch_123"})    |                                       |
      |                                       |                                       |
   ======================= SKENARIO DUPLIKAT (NETWORK RETRY) ==========================
      |                                       |                                       |
      |-- 10. POST /charges (RETRY) --------->|                                       |
      |    (Same Idempotency-Key & Payload)   |-- 11. Compute SHA-256(Body) --------->|
      |                                       |                                       |
      |                                       |-- 12. Check DB / Cache for key ------>|
      |                                       |<-- Found: Status='SUCCESS', resp_json-|
      |                                       |                                       |
      |<-- 13. HTTP 201 Created (REPLAY) -----| (NO PAYMENT PROCESSED, ZERO DOUBLE CHARGE)
      |    (Cached Body: {"charge_id":...})   |                                       |
      |                                       |                                       |
   ======================= SKENARIO KONKURENSI TINGGI (RACE CONDITION) ================
      |                                       |                                       |
      |-- 14. POST /charges (Rapid Click) --->|                                       |
      |    (Concurrent duplicate request)     |-- 15. Atomic SETNX key:lock --------->|
      |                                       |<-- Lock Failed: Already Locked -------|
      |                                       |                                       |
      |<-- 16. HTTP 409 Conflict -------------| (Request in-flight, please wait)      |
```

---

## 5. Contoh Implementasi Nyata di Kode (Go)

Contoh berikut menunjukkan implementasi production-ready untuk middleware dan handler transaksi pembayaran di Go dengan:
- Ekstraksi dan validasi header `Idempotency-Key`.
- Fingerprinting payload request dengan `SHA-256`.
- Distributed lock non-blocking menggunakan Redis (`SET NX EX`).
- Boundary penyimpanan state dan cached response replay.

```go
package main

import (
	"bytes"
	"context"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log"
	"net/http"
	"sync"
	"time"
)

// IdempotencyRecord merepresentasikan snapshot transaksi di database.
type IdempotencyRecord struct {
	Key          string    `json:"key"`
	RequestHash  string    `json:"request_hash"`
	Status       string    `json:"status"` // "STARTED", "SUCCESS", "FAILED"
	ResponseCode int       `json:"response_code"`
	ResponseBody []byte    `json:"response_body"`
	CreatedAt    time.Time `json:"created_at"`
}

// MockStorage mensimulasikan database PostgreSQL dan Redis distributed lock.
type MockStorage struct {
	mu      sync.Mutex
	locks   map[string]time.Time
	records map[string]*IdempotencyRecord
}

var store = &MockStorage{
	locks:   make(map[string]time.Time),
	records: make(map[string]*IdempotencyRecord),
}

// AcquireLock mensimulasikan redis.SetNX(ctx, "lock:"+key, "1", 60*time.Second)
func (s *MockStorage) AcquireLock(key string, ttl time.Duration) bool {
	s.mu.Lock()
	defer s.mu.Unlock()

	exp, exists := s.locks[key]
	if exists && time.Now().Before(exp) {
		return false // Kunci sedang dipegang oleh request lain (in-flight)
	}
	s.locks[key] = time.Now().Add(ttl)
	return true
}

func (s *MockStorage) ReleaseLock(key string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	delete(s.locks, key)
}

func (s *MockStorage) GetRecord(key string) (*IdempotencyRecord, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()
	rec, exists := store.records[key]
	return rec, exists
}

func (s *MockStorage) SaveRecord(rec *IdempotencyRecord) {
	s.mu.Lock()
	defer s.mu.Unlock()
	store.records[rec.Key] = rec
}

type PaymentRequest struct {
	AccountID string  `json:"account_id"`
	Amount    float64 `json:"amount"`
	Currency  string  `json:"currency"`
}

type PaymentResponse struct {
	TransactionID string `json:"transaction_id"`
	Status        string `json:"status"`
	Message       string `json:"message"`
}

// PaymentChargeHandler menangani mutasi pembayaran dengan boundary idempotency.
func PaymentChargeHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Method Not Allowed", http.StatusMethodNotAllowed)
		return
	}

	idempotencyKey := r.Header.Get("Idempotency-Key")
	if idempotencyKey == "" {
		http.Error(w, `{"error":"Missing required header: Idempotency-Key"}`, http.StatusBadRequest)
		return
	}

	// 1. Baca dan hitung hash SHA-256 dari request body untuk mendeteksi payload tampering
	bodyBytes, err := io.ReadAll(r.Body)
	if err != nil {
		http.Error(w, `{"error":"Unable to read request body"}`, http.StatusBadRequest)
		return
	}
	r.Body = io.NopCloser(bytes.NewBuffer(bodyBytes)) // Reset body stream

	hasher := sha256.New()
	hasher.Write(bodyBytes)
	currentHash := hex.EncodeToString(hasher.Sum(nil))

	// 2. Cek apakah record transaksi sebelumnya sudah pernah tersimpan di database
	if existing, found := store.GetRecord(idempotencyKey); found {
		// Validasi apakah payload identik dengan request pertama
		if existing.RequestHash != currentHash {
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusUnprocessableEntity)
			w.Write([]byte(`{"error":"Idempotency-Key reused with different request payload"}`))
			return
		}

		// Jika transaksi sebelumnya sedang berjalan di goroutine/worker lain
		if existing.Status == "STARTED" {
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusConflict)
			w.Write([]byte(`{"error":"A transaction with this Idempotency-Key is currently in progress"}`))
			return
		}

		// REPLAY RESPONSE: Jika sudah selesai dan sukses, kembalikan cached snapshot
		w.Header().Set("Content-Type", "application/json")
		w.Header().Set("X-Cache-Lookup", "HIT - Idempotent Replay")
		w.WriteHeader(existing.ResponseCode)
		w.Write(existing.ResponseBody)
		return
	}

	// 3. Akuisisi distributed lock untuk mencegah race condition (misal pengguna klik 2x cepat)
	lockKey := "idemp_lock:" + idempotencyKey
	if !store.AcquireLock(lockKey, 30*time.Second) {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusConflict)
		w.Write([]byte(`{"error":"Concurrent request locked, please retry shortly"}`))
		return
	}
	defer store.ReleaseLock(lockKey)

	// Tandai state awal transaksi sebagai STARTED
	initRecord := &IdempotencyRecord{
		Key:         idempotencyKey,
		RequestHash: currentHash,
		Status:      "STARTED",
		CreatedAt:   time.Now(),
	}
	store.SaveRecord(initRecord)

	// 4. Eksekusi domain logic pembayaran inti
	var payReq PaymentRequest
	if err := json.Unmarshal(bodyBytes, &payReq); err != nil {
		http.Error(w, `{"error":"Invalid JSON payload"}`, http.StatusBadRequest)
		return
	}

	// Simulasi pemrosesan gateway / core banking
	txID := fmt.Sprintf("tx_charge_%d", time.Now().UnixNano())
	payResp := PaymentResponse{
		TransactionID: txID,
		Status:        "SETTLED",
		Message:       fmt.Sprintf("Successfully charged %.2f %s", payReq.Amount, payReq.Currency),
	}

	respBytes, _ := json.Marshal(payResp)

	// 5. Simpan respons final ke record idempotency di DB secara transaksional
	initRecord.Status = "SUCCESS"
	initRecord.ResponseCode = http.StatusCreated
	initRecord.ResponseBody = respBytes
	store.SaveRecord(initRecord)

	// 6. Kembalikan respons ke client
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	w.Write(respBytes)
}

func main() {
	http.HandleFunc("/v1/payments/charges", PaymentChargeHandler)
	log.Println("Payment Gateway running on :8080...")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Garansi Konsistensi Finansial (Zero Double-Deduction)**: Menyelesaikan masalah double-spending dan duplicate billing secara deterministik pada arsitektur terdistribusi.
- **Client Auto-Retry yang Aman**: Client mobile atau microservice pemanggil dapat menerapkan strategi retry agresif tanpa khawatir merusak konsistensi data saat terjadi timeout sesaat.
- **Transparansi Jaringan**: Menutupi celah kegagalan transmisi jaringan antara client dan server (*network partition resilience*).

### Kekurangan
- **Overhead Latensi Tambahan**: Setiap request pembayaran wajib melakukan round-trip ke distributed lock (Redis) dan query lookup ke database sebelum domain logic dijalankan.
- **Manajemen Storage & Housekeeping**: Tabel/cache `idempotency_keys` tumbuh sangat cepat jika traffic tinggi. Memerlukan partisi tabel harian dan background cleaner (TTL expiration 24–48 jam).
- **Kompleksitas Penanganan Error**: Developer harus berhati-hati menentukan apakah suatu kegagalan (misalnya saldo tidak cukup vs database disk full) harus disimpan sebagai status `FAILED` permanen atau dibuka kuncinya agar bisa di-retry.

---

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- **Financial & Money Transfers**: Endpoint pengisian saldo (top-up), potong saldo (*charge*), penarikan dana (*disbursement*), dan refund.
- **Order Placement & Checkout**: Sistem pemesanan barang e-commerce atau booking tiket di mana stok dipotong dan invoice di-generate.
- **External Webhook Dispatching**: Saat mengirimkan webhook ke sistem pihak ketiga untuk memastikan partner tidak memproses payload event yang sama berkali-kali.

### Kapan Jangan Pakai
- **Operasi Baca Murni (Read-Only)**: Endpoint `GET`, `HEAD`, atau `OPTIONS` yang sudah idempoten secara natural. Menaruh lookup idempotency di sini hanya membuang resource komputasi.
- **High-Velocity Telemetry/IoT Ingestion**: Ingestion log metrik atau koordinat GPS kendaraan setiap 500ms di mana jika ada frame yang hilang, frame terbaru berikutnya lebih berharga daripada memblokir operasi demi idempotency key.
- **Operasi Update Bersifat Penggantian Mutlak**: Endpoint `PUT /users/profile` yang mengganti nama pengguna secara absolut (kecuali jika membutuhkan audit trail strictly-ordered).

---

## 8. Metrik Kunci & Observabilitas

Untuk memastikan sistem deduplikasi berjalan optimal di produksi, pantau metrik-metrik berikut:

1. **Idempotency Hit Rate (`idempotency_cache_hits_total`)**: Rasio request yang dilayani langsung dari cache replay dibanding total request. Lonjakan mendadak menandakan client sedang mengalami bad network loop atau bug auto-retry berlebihan.
2. **Payload Hash Mismatch Alerts (`idempotency_hash_mismatch_total`)**: Jumlah request yang menggunakan `Idempotency-Key` sama tetapi payload-nya berbeda. Metrik ini krusial untuk mendeteksi bug frontend (misal UUID di-cache dan tidak di-generate ulang) atau percobaan penipuan (*tampering attack*).
3. **In-Flight Lock Contention (`idempotency_lock_conflicts_total`)**: Frekuensi respons `409 Conflict` akibat concurrent duplicates. Jika tinggi, periksa UI client apakah tombol bayar tidak di-disable saat status loading.
4. **Lookup Latency Overhead (p99)**: Waktu yang dihabiskan untuk memeriksa state idempotency di Redis dan database (targetkan < 5ms).

---

## 9. Studi Kasus Nyata di Industri

### Skenario: Stripe Payment API & Mobile Network Flakiness

#### Problem
Stripe melayani jutaan merchant e-commerce di seluruh dunia. Pada jaringan seluler 3G/4G di negara berkembang, tingkat *packet loss* sangat tinggi. Pembeli sering mengalami timeout saat melakukan pembayaran. Jika mobile browser melakukan refresh atau SDK merchant melakukan retry otomatis saat request pertama sebenarnya sudah berhasil diproses di Visa/Mastercard, kartu pembeli akan terdebit ganda. Hal ini memicu sengketa *chargeback*, kepanikan pengguna, dan biaya penalti kartu kredit bagi merchant.

#### Solusi Arsitektur
Stripe merancang arsitektur **Idempotency Layer** yang terstandarisasi secara global:
1. Setiap SDK (iOS, Android, React Native, Node.js) mewajibkan atau meng-generate header `Idempotency-Key` berbentuk UUIDv4 sebelum koneksi TCP dibuka.
2. Lapisan proxy gateway (Envoy/internal proxy) melakukan hashing pada URL endpoint, HTTP method, User Account, dan body request (menggunakan SHA-256).
3. Record idempotency disimpan dalam cluster penyimpanan terdistribusi dengan state locking atomik. Jika ada request kedua dengan kunci yang sama dan status pembayaran telah selesai, respons JSON dan HTTP header asli langsung di-replay secara identik dari storage snapshot tanpa menyentuh core payment processor lagi.
4. Kunci kadaluwarsa secara otomatis setelah rentang waktu 24 jam.

#### Hasil / Dampak
- **Zero Double Charges**: Mengeliminasi 100% insiden pemotongan ganda akibat network retry di seluruh dunia.
- **Developer Confidence**: Merchant dapat dengan percaya diri mengaktifkan retry otomatis hingga 5 kali dengan *exponential backoff* pada SDK mereka tanpa takut merusak ledger keuangan.
- **Menjadi Standar De Facto**: Pola header `Idempotency-Key` dari Stripe kini diadaptasi oleh IETF (Internet Engineering Task Force) sebagai draft standar HTTP RFC untuk transaksi finansial.

---

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan Hash Mismatch pada JSON Serialization**: Urutan key pada JSON body tidak dijamin (`{"amount":100, "user":"A"}` vs `{"user":"A", "amount":100}`). Jika client atau middleware melakukan serialize ulang dan urutan key berubah, hash SHA-256 akan berbeda drastis! Solusi: Lakukan hashing pada raw bytes tepat saat request diterima, atau lakukan *canonical JSON sorting* sebelum di-hash.
2. **The "Forever In-Progress" Zombie Lock**: Jika service backend mengalami panic (OOM crash atau pod di-kill oleh Kubernetes) di tengah-tengah pemrosesan pembayaran sebelum sempat mengupdate state menjadi `SUCCESS` atau `FAILED`, kunci di database bisa terkunci selamanya dengan status `STARTED`. Solusi: Selalu berikan lease expiration time (TTL) pada distributed lock di Redis, dan miliki cron sweeper yang memverifikasi transaksi gantung ke payment gateway.
3. **Jangan Menyimpan Respons Error Sementara Sebagai Sukses**: Jika database replika mengalami timeout sesaat (transient error 500/503), **JANGAN** simpan status tersebut sebagai cached response replay permanen! Kunci harus dibuka kembali agar retry berikutnya dari client mendapatkan kesempatan untuk mencoba ulang.
4. **Scope Idempotency Key Berdasarkan Tenant/User**: Selalu gabungkan `Idempotency-Key` dengan `user_id` atau `merchant_id` di database (`UNIQUE(merchant_id, idempotency_key)`). Jika hanya mengandalkan string UUID mentah dari client tanpa scoping, satu pengguna jahat dapat menebak atau mendahului kunci pengguna lain (*key squatting*).

---

## 11. Rangkuman Singkat

1. **Tujuan Utama**: Mencegah pemotongan ganda (*double-charging*), pemesanan ganda, dan inkonsistensi data saat client melakukan auto-retry akibat koneksi jaringan yang putus atau timeout.
2. **Komponen Inti**: Header `Idempotency-Key` (UUIDv4), fingerprinting payload dengan hash `SHA-256`, dan state machine transaksi (`STARTED`, `SUCCESS`, `FAILED`).
3. **Mekanisme Replay**: Request kedua dengan key dan payload yang sama tidak mengeksekusi ulang kode bisnis, melainkan langsung mengembalikan salinan respons sebelumnya (*cached response replay*).
4. **Perlindungan Konkurensi**: Gunakan distributed lock non-blocking (Redis `SET NX EX`) agar dua request identik yang datang bersamaan dalam hitungan milidetik langsung ditolak dengan `409 Conflict`.
