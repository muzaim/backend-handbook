# Horizontal Scaling (Scale-Out)

## 1. Skenario Sistem Nyata

Pada kampanye belanja nasional tanggal kembar (11.11), backend layanan pemrosesan pesanan (*Order Service*) mengalami lonjakan lalu lintas dari 1.500 request per detik (RPS) pada hari biasa menjadi 75.000 RPS dalam 30 detik pertama sejak promo dibuka.

Jika backend dirancang hanya mengandalkan satu mesin server tunggal berukuran besar:
- Meskipun mesin tersebut memiliki 64 Core CPU dan 256 GB RAM, batas kapasitas komputasi, bandwidth network card (NIC), dan antrean socket TCP pada satu sistem operasi akan tetap mencapai saturasi maksimum (*thread starvation*).
- Jika server tunggal tersebut mengalami crash akibat kegagalan kernel atau pemadaman listrik di data center, seluruh sistem pembayaran e-commerce mati total (*Single Point of Failure*).

**Solusi dengan Horizontal Scaling (Scale-Out):**
Sistem dipecah menjadi layanan tanpa status (*stateless services*). Alih-alih memperbesar satu server, sistem menduplikasi aplikasi menjadi 40 instance (pod/container) berukuran sedang (misal 2 Core, 4 GB RAM) di balik sebuah Load Balancer. Semua session dan status pesanan disimpan di layer eksternal (Redis cluster dan PostgreSQL). Ketika trafik melonjak, Horizontal Pod Autoscaler (HPA) secara otomatis menambah jumlah instance dari 10 menjadi 40 dalam 90 detik. Begitu promo berakhir, sistem secara bertahap memangkas jumlah instance kembali ke 10 untuk menghemat biaya cloud.

## 2. Apa Itu dan Buat Apa?

Horizontal Scaling (*Scale-Out*) adalah strategi meningkatkan kapasitas sistem dengan cara menambah jumlah unit komputasi (server, virtual machine, atau container) yang berjalan secara paralel, bukan dengan memperbesar spesifikasi mesin yang sudah ada.

Karakteristik utama:
- **Stateless Architecture**: Aplikasi tidak boleh menyimpan data session atau status pengguna di memori lokal server (`local memory state`). State wajib dipindahkan ke *shared storage* (seperti Redis, PostgreSQL, atau Object Storage).
- **Load Distribution**: Seluruh instance berada di balik Reverse Proxy atau Load Balancer (misal NGINX, AWS ALB, atau Kubernetes Ingress) yang mendistribusikan beban secara merata (Round-Robin atau Least Connections).
- **Dynamic Autoscaling**: Kapasitas dapat bertambah atau berkurang secara elastis mengikuti metrik utilisasi CPU, memori, atau antrean pesan.

## 3. Kenapa Harus Pakai Ini di High-Throughput?

Dalam arsitektur modern berskala besar, penskalaan horizontal adalah satu-satunya cara untuk mencapai throughput hampir tak terbatas (*near-infinite scaling*):
1. **Tidak Ada Batas Langit Perangkat Keras**: Server termahal di dunia pun memiliki batas fisik CPU dan RAM. Dengan scale-out, sistem dapat bertumbuh dari 2 server menjadi 2.000 server di beberapa zona ketersediaan (*multi-availability zone*).
2. **Ketersediaan Tinggi (High Availability)**: Jika 3 dari 20 server mati mendadak karena hardware failure, sistem tetap berjalan normal melayani trafik melalui 17 server yang tersisa.
3. **Efisiensi Biaya (Elasticity)**: Biaya sewa server hanya dibayarkan saat dibutuhkan melalui mekanisme autoscaling.

## 4. Cara Kerjanya (Diagram Alur)

```
                            +-------------------+
                            |  Incoming Traffic |
                            |   (75,000 RPS)    |
                            +-------------------+
                                      |
                                      v
                            +-------------------+
                            |   Load Balancer   |
                            | (AWS ALB / NGINX) |
                            +-------------------+
                                      |
           +--------------------------+--------------------------+
           |                          |                          |
           v                          v                          v
+---------------------+    +---------------------+    +---------------------+
| Order Instance #1   |    | Order Instance #2   |    | Order Instance #N   |
| (Stateless Go App)  |    | (Stateless Go App)  |    | (Stateless Go App)  |
+---------------------+    +---------------------+    +---------------------+
           |                          |                          |
           +--------------------------+--------------------------+
                                      |
                     +----------------+----------------+
                     |                                 |
                     v                                 v
          +---------------------+           +---------------------+
          |    Shared Cache     |           |   Shared Database   |
          |   (Redis Cluster)   |           |  (PostgreSQL Read/  |
          |   [Session/Tokens]  |           |   Write Replicas)   |
          +---------------------+           +---------------------+
```

## 5. Cuplikan Logika Inti (Stateless Session Handler di Go)

Kunci sukses horizontal scaling adalah memastikan instance aplikasi tidak menyimpan status login di memori lokal. Berikut implementasi autentikasi berbasis Redis terpusat di Go:

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"time"

	"github.com/redis/go-redis/v9"
)

var (
	ctx = context.Background()
	rdb *redis.Client
)

type UserSession struct {
	UserID    string    `json:"user_id"`
	Role      string    `json:"role"`
	CreatedAt time.Time `json:"created_at"`
}

// StatelessAuthMiddleware memeriksa token session ke shared Redis cluster,
// sehingga request pengguna dapat diarahkan ke instance Go mana pun tanpa terputus.
func StatelessAuthMiddleware(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		sessionToken := r.Header.Get("X-Session-Token")
		if sessionToken == "" {
			http.Error(w, "Unauthorized: missing session token", http.StatusUnauthorized)
			return
		}

		sessionKey := fmt.Sprintf("session:%s", sessionToken)

		// Ambil data session dari Redis terpusat (bukan local memory)
		val, err := rdb.Get(ctx, sessionKey).Result()
		if err == redis.Nil {
			http.Error(w, "Unauthorized: invalid or expired session", http.StatusUnauthorized)
			return
		} else if err != nil {
			http.Error(w, "Internal Server Error", http.StatusInternalServerError)
			return
		}

		var session UserSession
		if err := json.Unmarshal([]byte(val), &session); err != nil {
			http.Error(w, "Internal Server Error", http.StatusInternalServerError)
			return
		}

		// Suntikkan UserID ke context request untuk diproses handler berikutnya
		ctxWithUser := context.WithValue(r.Context(), "user_id", session.UserID)
		next(w, r.WithContext(ctxWithUser))
	}
}

func OrderHandler(w http.ResponseWriter, r *http.Request) {
	userID := r.Context().Value("user_id").(string)
	w.WriteHeader(http.StatusOK)
	fmt.Fprintf(w, "Order successfully processed for user %s on stateless node", userID)
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Skalabilitas Nyaris Tanpa Batas**: Kapasitas sistem bertambah linier seiring penambahan node.
- **Tahan Banting (*Fault Tolerance*)**: Tidak ada ketergantungan fatal pada satu mesin server fisik.
- **Zero-Downtime Deployment**: Update kode dapat dilakukan bergantian (*Rolling Update / Canary*) tanpa menghentikan layanan.

### Kekurangan
- **Kompleksitas Jaringan & Orkestrasi**: Membutuhkan alat orkestrasi seperti Kubernetes, service mesh, dan service discovery.
- **Biaya Koneksi Database Meledak**: Jika 50 pod baru menyala dan masing-masing membuka 20 koneksi DB, database langsung kehabisan connection pool (1.000 koneksi serentak).
- **Latensi Jaringan Tambahan**: Komunikasi antar node dan ke database/cache terpusat melewati jaringan fisik data center.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Sistem melayani beban kerja bervolume tinggi dengan fluktuasi trafik dinamis (e-commerce, ride-hailing, streaming video).
- Kebutuhan availability tinggi (SLA 99.99%) yang tidak mentolerir downtime saat server fisik rusak.
- Aplikasi sudah dirancang dengan pola microservices atau containerized backend.

### Kapan Jangan Pakai
- Sistem monolit internal kecil dengan pengguna terbatas (misal: sistem administrasi kantor dengan 50 karyawan).
- Aplikasi *stateful* yang sangat bergantung pada file lokal di disk hard drive atau memori RAM lokal yang sulit disinkronkan.
- Anggaran operasional awal sangat terbatas dan tim belum memiliki keahlian mengelola cluster container/Kubernetes.

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

- **`pod_replicas_count` (Gauge)**: Jumlah instance/container aktif yang sedang melayani request.
- **`request_distribution_skew` (Calculation)**: Deviasi standar request antar instance. Nilai yang jomplang menandakan algoritma load balancer kurang optimal.
- **`hpa_cpu_utilization_percent` (Gauge)**: Rata-rata pemakaian CPU di seluruh replica untuk menentukan pemicu autoscaling.
- **`db_connections_total` (Gauge)**: Total koneksi aktif ke database dari seluruh instance horizontal.

## 9. Studi Kasus Industri Nyata

- **Netflix**: Mengoperasikan puluhan ribu instance microservices di AWS EC2. Seluruh sistem backend didesain *stateless* sehingga ribuan instance dapat dimatikan, dipindahkan antar-region, atau dibuat ulang dalam hitungan menit tanpa mengganggu jutaan penonton yang sedang streaming.
- **Uber**: Selama perayaan malam Tahun Baru, jumlah pesanan perjalanan melonjak puluhan kali lipat. Infrastruktur orkestrasi internal Uber secara otomatis melakukan scale-out ribuan container komputasi untuk menangani geolokasi dan kalkulasi rute secara real-time.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Jebakan In-Memory State**: Developer pemula sering menyimpan data autentikasi atau cache di variabel global memory Go. Saat instance di-scale menjadi 5 node, pengguna mendadak logout saat request kedua diarahkan load balancer ke instance lain yang tidak memiliki data session tersebut.
2. **Kelelahan Database Connection (*Connection Exhaustion*)**: Saat HPA menaikkan pod dari 10 menjadi 100 instance, database PostgreSQL langsung kolaps karena menerima lonjakan koneksi baru secara masif. *Solusi*: Pasang dedicated database connection pooler seperti **PgBouncer** di depan database.
3. **Autoscaling Thrashing / Flapping**: HPA terlalu agresif menaikkan dan menurunkan pod dalam rentang waktu singkat (scale up lalu 1 menit kemudian scale down). *Solusi*: Atur *stabilization window* (misal 5 menit *cooldown*) pada konfigurasi HPA Kubernetes.

## 11. Rangkuman Singkat

Horizontal Scaling adalah strategi menambah jumlah node server stateless secara elastis untuk mendistribusikan beban kerja secara paralel. Pola ini adalah pilar utama backend modern berkemampuan tinggi yang menjamin ketersediaan tinggi, elastisitas biaya, dan ketahanan sistem tanpa terbatasi batas fisik perangkat keras tunggal.
