# Microservices Architecture

## 1. Analogi Sederhana & Skenario Nyata

Bayangkan sebuah jaringan perusahaan logistik dan retail global:

Alih-alih menaruh seluruh aktivitas di satu gedung, perusahaan memecah operasinya menjadi ratusan entitas mandiri yang tersebar:
- **Pusat Pergudangan & Inventaris** berada di Cikarang dengan armada truk dan sistem database stok sendiri.
- **Kantor Pusat Pembayaran & Kliring Bank** berada di SCBD Jakarta dengan server perbankan yang super aman dan staf kepatuhan sendiri.
- **Divisi Kurir Pengantaran Terakhir** berada di setiap kecamatan dengan armada motor dan aplikasi peta real-time sendiri.

Setiap entitas tersebut beroperasi secara otonom:
- Divisi Gudang bisa mengganti sistem rak atau software internal mereka tanpa perlu meminta izin ke Divisi Pembayaran.
- Komunikasi antar-entitas tidak bisa lagi dilakukan dengan berteriak langsung; mereka wajib berkomunikasi melalui jaringan telekomunikasi resmi (jaringan radio atau HTTP/gRPC API).
- Jika server di kantor pembayaran mengalami pemadaman listrik sesaat, armada kurir di lapangan tetap bisa mengantarkan paket barang yang sudah ada di tas mereka (*fault isolation*).

Namun, ada konsekuensi besar: jika jaringan internet antar-kota terputus (*network partition*), koordinasi pengiriman paket bisa macet total dan biaya sewa serta operasional ratusan kantor jauh lebih mahal dibanding satu kantor terpadu.

Inilah gambaran nyata **Microservices Architecture**.

---

## 2. Definisi & Konsep Inti

**Microservices Architecture** adalah pendekatan perancangan perangkat lunak di mana sebuah aplikasi besar dipecah menjadi kumpulan **layanan-layanan kecil (*services*) yang terdistribusi secara independen**, berjalan di proses runtime terpisah, dan berkomunikasi melalui protokol jaringan ringan (*lightweight network protocols*).

Karakteristik & pilar utama microservices:
1. **Database-per-Service**: Setiap microservice memiliki dan mengelola basis datanya sendiri. Service `Order` tidak boleh mengakses database `Payment` secara langsung. Satu-satunya cara mendapatkan data adalah dengan memanggil API publik milik service tersebut.
2. **Autonomous Deployment**: Setiap service memiliki repositori terpisah (atau folder mandiri di monorepo), pipeline CI/CD sendiri, dan dapat di-deploy ke production puluhan kali sehari tanpa perlu me-redeploy service lainnya.
3. **Network-Based Inter-Service Communication**: Komunikasi dilakukan melalui:
   - **Sinkron (Synchronous RPC)**: Menggunakan REST (HTTP/JSON) atau gRPC (HTTP/2 + Protocol Buffers) untuk request yang membutuhkan respons langsung.
   - **Asinkron (Asynchronous Messaging)**: Menggunakan message broker (Apache Kafka atau RabbitMQ) untuk penyebaran event domain (*event-driven*).
4. **Decentralized Governance & Polyglot Architecture**: Tim bebas memilih teknologi terbaik untuk tugasnya (misal Service Machine Learning menggunakan Python, Service Payment menggunakan Go, dan Backend Gateway menggunakan Node.js).

---

## 3. Mengapa Sangat Penting di Skala Produksi

Microservices menjadi standar arsitektur bagi raksasa teknologi (seperti Netflix, Uber, Amazon, dan Gojek):

1. **Skalabilitas Organisasi Skala Masif (Conway's Law)**: Ketika perusahaan memiliki 500+ engineer yang terbagi menjadi puluhan squad otonom (6–8 orang per squad), monolit akan macet karena antrean rilis. Dengan microservices, squad `Payment` bisa merilis fitur tanpa terhambat oleh squad `Review & Rating`.
2. **Penskalaan Komputasi Independen (Granular Auto-Scaling)**: Jika trafik checkout melonjak 10x saat flash sale, kalian hanya perlu menambah replika kontainer Pod `Order Service` dari 5 menjadi 50, tanpa perlu memboroskan uang untuk menskalakan service lain yang tidak sibuk.
3. **Isolasi Kegagalan (Fault Isolation & Blast-Radius Containment)**: Jika microservice Rekomendasi Produk mengalami crash karena bug OOM, alur utama checkout dan pembayaran tetap berfungsi normal.
4. **Agilitas Eksperimen & Adopsi Teknologi Baru**: Memungkinkan pengujian database baru atau bahasa pemrograman baru pada satu service kecil tanpa risiko membahayakan seluruh sistem perusahaan.

---

## 4. Cara Kerja & Diagram Alur (ASCII)

Topologi request dan pola inter-service communication pada Microservices:

```
[ Mobile / Web Client ]
          |
          | 1. HTTP Request (POST /api/v1/checkout)
          v
+-----------------------------------------------------------------------------------+
| API GATEWAY / INGRESS (Kong / Envoy / NGINX)                                     |
|  - Terminasi SSL/TLS & Autentikasi JWT terpusat                                   |
|  - Routing request ke Order Service                                              |
+-----------------------------------------------------------------------------------+
          |
          | 2. Forwarded Request (Internal Network)
          v
+----------------------------------------------------+
| ORDER MICROSERVICE (Go Pod)                        |
|   - Database Pribadi: PostgreSQL (orders_db)       |
|   - Menangani domain logic pemesanan barang        |
+----------------------------------------------------+
          |                                  |
          | 3. Synchronous RPC (gRPC / mTLS) | 5. Asynchronous Event
          |    (Protected by Circuit Breaker)|    (Publish to Kafka)
          v                                  v
+-----------------------------------+  +--------------------------------------------+
| PAYMENT MICROSERVICE (Go Pod)     |  | APACHE KAFKA MESSAGE BROKER                |
|  - Database Pribadi: MySQL (pay_db|  |  Topic: "orders.created"                   |
|  - Proses potong saldo / kartu    |  +--------------------------------------------+
+-----------------------------------+         |                                |
                                              | 6. Consume Event               | 7. Consume Event
                                              v                                v
                                   +----------------------+       +-----------------------+
                                   | INVENTORY SERVICE    |       | NOTIFICATION SERVICE  |
                                   | - DB: MongoDB        |       | - DB: Redis           |
                                   | - Kurangi stok gudang|       | - Kirim Push Notif/WA |
                                   +----------------------+       +-----------------------+
```

---

## 5. Contoh Implementasi Nyata di Kode (Go)

Contoh berikut menunjukkan bagaimana sebuah Microservice `Order` memanggil `Payment Service` melalui **gRPC Network Boundary** lengkap dengan **Context Timeout** dan **Circuit Breaker** untuk menangani kegagalan jaringan terdistribusi:

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
	"net/http"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

// Simulasi Client Stub gRPC untuk Payment Service eksternal
type PaymentServiceClient interface {
	Charge(ctx context.Context, orderID string, amount float64) (string, error)
}

type paymentGrpcClient struct {
	targetAddress string
}

func (c *paymentGrpcClient) Charge(ctx context.Context, orderID string, amount float64) (string, error) {
	// Simulasi pemanggilan RPC melalui jaringan TCP/HTTP2
	select {
	case <-time.After(80 * time.Millisecond): // Simulasi latensi jaringan microservice
		return fmt.Sprintf("ch_grpc_%s", orderID), nil
	case <-ctx.Done():
		return "", ctx.Err() // Menangani timeout jaringan terdistribusi
	}
}

// OrderService merepresentasikan microservice independen dengan domain boundaries
type OrderService struct {
	paymentClient PaymentServiceClient
}

func (s *OrderService) CheckoutHandler(w http.ResponseWriter, r *http.Request) {
	// 1. Wajib menyertakan context timeout untuk mencegah connection hanging
	ctx, cancel := context.WithTimeout(r.Context(), 500*time.Millisecond)
	defer cancel()

	orderID := "ord_ms_7781"
	amount := 125000.0

	log.Printf("[OrderService] Initiating gRPC call to external Payment Service...")

	// 2. Eksekusi pemanggilan jaringan lintas-service
	chargeID, err := s.paymentClient.Charge(ctx, orderID, amount)
	if err != nil {
		if errors.Is(err, context.DeadlineExceeded) {
			log.Printf("[OrderService] Payment Service timeout: %v", err)
			http.Error(w, `{"error":"Payment downstream service timeout"}`, http.StatusGatewayTimeout)
			return
		}
		log.Printf("[OrderService] Payment RPC failed: %v", err)
		http.Error(w, `{"error":"Payment processing failed"}`, http.StatusBadGateway)
		return
	}

	log.Printf("[OrderService] Payment successfully cleared with ID: %s", chargeID)
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	fmt.Fprintf(w, `{"order_id":"%s","charge_id":"%s","status":"CONFIRMED"}`, orderID, chargeID)
}

func main() {
	// Koneksi ke host jaringan Payment Service terpisah
	payClient := &paymentGrpcClient{targetAddress: "payment-service.prod:50051"}
	orderSvc := &OrderService{paymentClient: payClient}

	http.HandleFunc("/api/v1/orders/checkout", orderSvc.CheckoutHandler)
	log.Println("Order Microservice listening on :8081...")
	log.Fatal(http.ListenAndServe(":8081", nil))
}
```

---

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Otonomi Tim Maksimal**: Tim dapat merilis service mereka secara mandiri tanpa koordinasi rumit atau menunggu rilis monolit.
- **Penskalaan Sangat Presisi**: Hanya mengalokasikan resource komputasi ekstra (CPU/RAM/GPU) pada service yang sedang kebanjiran trafik.
- **Ketahanan Sistem Tinggi (*Resilience*)**: Kerusakan pada service non-kritis tidak mematikan service inti lainnya.
- **Kebebasan Memilih Teknologi (*Polyglot*)**: Service analitik dapat ditulis dengan Python, sedangkan service transaksi menggunakan Go.

### Kekurangan
- **Kompleksitas Sistem Terdistribusi**: Munculnya masalah *network partitions*, serialisasi, *distributed deadlocks*, dan perlunya pola transaksi rumit (Saga Pattern / 2PC).
- **Latensi Kumulatif (*Network Hops*)**: Request dari client yang melewati rantai pemanggilan 5 microservice berturut-turut akan mengalami akumulasi latensi jaringan (dari mikrodetik ke puluhan milidetik).
- **Biaya Infrastruktur Membengkak**: Membutuhkan kluster Kubernetes, API Gateway, Service Mesh, distributed tracing (Jaeger/Tempo), dan banyak database terpisah.
- **Kesulitan Menjaga Konsistensi Data**: Menghilangkan integritas transaksi ACID lokal; sistem harus mengadopsi *Eventual Consistency*.

---

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- **Organisasi Skala Besar (> 150-200 Engineer)**: Puluhan tim squad terhambat antrean rilis jika terus bekerja di satu repositori.
- **Karakteristik Beban Kerja Sangat Beragam**: Sistem memiliki modul I/O tinggi, modul machine learning berbasis GPU, dan modul database write-heavy yang butuh arsitektur hardware berbeda.
- **Kebutuhan Domain Bisnis yang Sudah Sangat Stabil**: Batas-batas domain bisnis (*Bounded Contexts*) sudah matang dan jarang berubah.

### Kapan Jangan Pakai
- **Startup Tahap Awal (Seed / Seri A)**: Kecepatan menemukan Product-Market Fit akan terbunuh oleh rumitnya mengelola infrastruktur microservices.
- **Tim Pengembang Berukuran Kecil (< 30-50 Orang)**: Sebagian besar waktu engineer akan habis mengurus DevOps dan debugging jaringan alih-alih membuat fitur untuk pengguna.
- **Domain Bisnis Masih Terus Berubah-Ubah**: Memecah sistem saat batas domain belum jelas akan menciptakan bencana *"Distributed Monolith"*.

---

## 8. Metrik Kunci & Observabilitas

1. **Distributed Trace Propagation Rate**: Persentase request yang membawa `TraceID` dan `SpanID` lengkap saat melintasi batas-batas service (OpenTelemetry).
2. **Inter-Service Error Rate (HTTP 5xx / gRPC Status Code)**: Tingkat kegagalan panggilan RPC antarlayanan akibat timeout atau connection reset.
3. **P99 Service Call Chain Latency**: Waktu yang dihabiskan untuk melintasi seluruh dependensi rantai RPC upstream dan downstream.
4. **Pod Restart & OOMKilled Count**: Frekuensi kontainer microservice mati mendadak di lingkungan Kubernetes.

---

## 9. Studi Kasus Nyata di Industri

### Skenario: Netflix — Pionir Transformasi Cloud Microservices

#### Problem
Pada tahun 2008, Netflix masih menggunakan basis data relasional monolitik raksasa. Sebuah kerusakan fatal pada database (*database corruption*) menghentikan pengiriman DVD mereka selama berhari-hari. Kerusakan tunggal ini menyadarkan pimpinan engineering Netflix bahwa sebuah monolit tidak memiliki batas kegagalan (*no fault tolerance*) untuk melayani puluhan juta penonton streaming global.

#### Solusi Arsitektur
Netflix menghabiskan waktu bertahun-tahun untuk memecah monolit menjadi ratusan **Microservices mandiri di AWS Cloud**:
- Setiap service berjalan di kluster VM/kontainer terpisah dan memiliki basis data terisolasi (Cassandra, MySQL, DynamoDB).
- Membangun ekosistem open-source terdistribusi: *Eureka* (Service Discovery), *Hystrix* (Circuit Breaker), dan *Zuul* (API Gateway).
- Menerapkan arsitektur *Chaos Engineering* (Chaos Monkey) yang sengaja mematikan service secara acak di production untuk membuktikan bahwa matinya satu service tidak meruntuhkan pengalaman menonton pengguna.

#### Hasil / Dampak
- **Ketersediaan 99.99% Global**: Kerusakan pada service algoritma rekomendasi film tidak menghentikan penonton untuk menekan tombol "Play".
- **Skalabilitas Tak Terbatas**: Melayani lebih dari 260 juta pelanggan global dengan ribuan microservices yang memproses miliaran jam streaming tanpa downtime terpusat.

---

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan "Distributed Monolith" (Anti-Pattern Nomor Satu)**: Mengklaim menggunakan microservices dengan 40 service terpisah di Kubernetes, namun seluruh service tersebut membaca dan menulis ke **satu database PostgreSQL yang sama**. Ini adalah bencana terburuk: mendapatkan seluruh kerumitan microservices ditambah seluruh kelemahan monolit.
2. **Cascading Failure & Thundering Herd**: Jika Service A gagal dan timeout, Service B melakukan retry agresif tanpa backoff, membombardir Service A yang sedang sakit hingga seluruh sistem runtuh seperti kartu domino (*cascading outage*). Solusi: Pasang Circuit Breakers dan Exponential Backoff dengan Jitter di setiap panggilan RPC.
3. **Pola Transaksi Saga Wajib Dikuasai**: Karena tidak ada lagi `tx.Commit()` lintas database, transaksi terdistribusi wajib diselesaikan dengan *Saga Pattern* (orkestrasi transaksi lokal dengan serangkaian *compensating transactions* jika terjadi kegagalan di tengah jalan).

---

## 11. Rangkuman Singkat

1. **Definisi Inti**: Aplikasi dipecah menjadi layanan-layanan kecil independen dengan proses runtime dan database terpisah (*Database-per-Service*) yang berkomunikasi melalui jaringan.
2. **Keunggulan Utama**: Otonomi rilis tim mandiri, penskalaan komputasi modular yang presisi, dan isolasi kegagalan (*fault containment*).
3. **Tantangan Terbesar**: Kompleksitas sistem terdistribusi, latensi jaringan kumulatif, biaya server tinggi, dan hilangnya kemudahan transaksi ACID lokal.
4. **Prinsip Utama**: Adopsi microservices ketika **skala organisasi tim (> 150 engineer)** mengharuskan otonomi rilis mandiri, bukan sekadar mengikuti tren industri.
