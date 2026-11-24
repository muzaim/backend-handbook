# Load Balancing

## 1. Skenario Sistem Nyata

Pada sistem backend pemesanan tiket konser musik atau flash sale e-commerce dengan lonjakan trafik mendadak hingga 40.000 HTTP request per detik saat penjualan dibuka:

Jika seluruh trafik ini diarahkan ke satu server backend tunggal (misal instance 8-Core CPU, 16GB RAM):

-   Server tersebut hanya sanggup memproses sekitar 2.000 request per detik sebelum socket connection pool habis, CPU menyentuh 100%, dan memori meledak (_OOM crash_).
-   38.000 pengguna lainnya mengalami timeout, error `HTTP 502 Bad Gateway`, dan transaksi gagal massal.
-   Server tunggal tersebut menjadi titik kegagalan fatal (_Single Point of Failure_ / SPOF). Jika mesin tersebut mati mendadak, seluruh bisnis lumpuh total.

**Solusi dengan Load Balancer (misal: HAProxy / AWS ALB / NGINX):**
Kamu menduplikasi aplikasi menjadi 20 instance pod backend identik (`Backend 1` hingga `Backend 20`) di jaringan privat. Load Balancer ditempatkan di depan dengan satu alamat IP publik. Saat 40.000 request per detik masuk, Load Balancer membagi beban secara merata (misal masing-masing 2.000 req/s) menggunakan algoritma seperti Round-Robin atau Least Connections. Jika `Backend 7` tiba-tiba mengalami crash, Load Balancer mendeteksi lewat _health check_ dalam hitungan detik dan langsung mengalihkan request ke 19 server lain yang sehat dengan _zero downtime_.

## 2. Apa Itu dan Buat Apa?

Load Balancing adalah teknik mendistribusikan beban lalu lintas jaringan (_network traffic_) atau beban komputasi secara merata ke beberapa server aplikasi backend (disebut _upstream pool_ atau _server farm_).

Tujuan utamanya adalah:

-   **Mencegah overload**: Memastikan tidak ada satu server pun yang kewalahan sementara server lain menganggur.
-   **Toleransi kegagalan (_High Availability_)**: Melakukan _Health Check_ secara berkala. Jika ada satu server backend yang mati atau crash, load balancer otomatis menghentikan pengiriman trafik ke server tersebut.
-   **Skalabilitas horizontal (_Scale-Out_)**: Memungkinkan kita menambah mesin server baru kapan saja saat trafik naik tanpa memutus layanan.

### Algoritma Load Balancing Populer:

-   **Round Robin**: Membagi request secara bergiliran (1, 2, 3, 1, 2, 3...).
-   **Least Connections**: Mengirimkan request ke server yang saat itu memiliki jumlah koneksi aktif paling sedikit (cocok untuk request yang durasi prosesnya bervariasi).
-   **IP Hash / Consistent Hashing**: Menentukan server tujuan berdasarkan hash dari alamat IP client (memastikan client yang sama selalu mendarat di server yang sama).
-   **Weighted Round Robin**: Membagi trafik berdasarkan kapasitas hardware mesin (mesin 16-core dapat porsi lebih banyak dari mesin 4-core).

Contoh software & hardware: NGINX, HAProxy, AWS ALB/NLB, Cloudflare, F5 BIG-IP.

## 3. Kenapa Harus Pakai Ini ?

Tidak peduli seberapa mahal dan canggih spesifikasi sebuah server (misal server 128 Core RAM 512GB), ada batas fisik maksimal (_vertical scaling limit_). Ketika sistem harus melayani 100.000 request per detik, satu server tunggal pasti akan mengalami kegagalan memori, thermal throttling, atau bottleneck socket TCP.

Selain itu, jika hanya ada satu server, pembaruan aplikasi (_deployment_) atau kerusakan hardware akan langsung menyebabkan pemadaman total (_total outage_).

Dengan load balancing, sistem bisa memanfaatkan 20 server kecil yang murah (_horizontal scaling_). Jika 2 server mati mendadak di tengah malam, sistem tetap berjalan normal 100% bagi pengguna.

## 4. Cara Kerjanya (Diagram Alur)

```
                       +-------------------+
                       |  Incoming Traffic |
                       | (10,000 req/sec)  |
                       +-------------------+
                                 |
                                 v
                     +-----------------------+
                     |     Load Balancer     |
                     |                       |
                     | Periodic Health Check |
                     | Algoritma: Round Robin|
                     +-----------------------+
                       /         |         \
         3,333 req/s  /          |          \  3,333 req/s
                     /  3,333    |           \
                    v   req/s    v            v
             +----------+  +----------+  +----------+
             | Server A |  | Server B |  | Server C |
             | (Healthy)|  | (Healthy)|  | (Healthy)|
             +----------+  +----------+  +----------+
```

Jika Server B mendadak gagal merespons _Health Check_:

```
                     +-----------------------+
                     |     Load Balancer     |
                     |  [Server B UNHEALTHY] |
                     +-----------------------+
                       /                   \
         5,000 req/s  /                     \  5,000 req/s
                     v                       v
             +----------+                 +----------+
             | Server A |  [X] Server B   | Server C |
             | (Healthy)|   (Isolated)    | (Healthy)|
             +----------+                 +----------+
```

## 5. Cuplikan Logika Inti (Algoritma Round Robin di Go)

Berikut implementasi algoritma Round Robin dengan seleksi node sehat (_health-aware_) di Go:

```go
package main

import (
	"errors"
	"sync"
)

type ServerNode struct {
	Host    string
	Healthy bool
}

type RoundRobinLoadBalancer struct {
	mu      sync.Mutex
	servers []*ServerNode
	index   int
}

func NewRoundRobinLoadBalancer(servers []*ServerNode) *RoundRobinLoadBalancer {
	return &RoundRobinLoadBalancer{
		servers: servers,
	}
}

// GetNextServer memilih server berikutnya yang berada dalam kondisi sehat (Healthy)
func (lb *RoundRobinLoadBalancer) GetNextServer() (*ServerNode, error) {
	lb.mu.Lock()
	defer lb.mu.Unlock()

	totalServers := len(lb.servers)
	for i := 0; i < totalServers; i++ {
		selected := lb.servers[lb.index]
		lb.index = (lb.index + 1) % totalServers

		if selected.Healthy {
			return selected, nil
		}
	}

	return nil, errors.New("seluruh upstream server sedang down / unhealthy")
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Zero Downtime Deployment**: Kita bisa mematikan server satu per satu untuk update kode (_rolling update_).
-   **Ketahanan tinggi (_Fault Tolerance_)**: Otomatis mendeteksi dan mengisolasi server yang bermasalah.
-   **Efisiensi biaya**: Lebih fleksibel dan murah menambah server skala menengah daripada membeli satu super-komputer.

### Kekurangan

-   **Manajemen Session State (_Sticky Session problem_)**: Server backend harus stateless. Jika login session disimpan di memori server lokal, request pengguna berikutnya yang mendarat di server lain akan menyebabkan pengguna logout mendadak. Solusinya: simpan session di Redis terpusat atau gunakan stateless JWT.
-   **Kompleksitas Layer 4 vs Layer 7**: Memilih antara L4 balancer (berbasis TCP packet, sangat kencang tapi buta isi request) atau L7 balancer (berbasis HTTP header/path, pintar tapi butuh CPU lebih besar).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Setiap sistem yang membutuhkan ketersediaan tinggi (_High Availability 99.9% SLA_).
-   Ketika trafik aplikasi sudah melampaui kemampuan satu mesin server.
-   Untuk mengimplementasikan strategi rilis bertahap (_Canary deployment_ atau _Blue-Green deployment_).

### Kapan Jangan Pakai

-   Server internal utilitas kecil atau prototipe MVP tahap awal di mana downtime beberapa menit tidak berdampak bisnis.
-   Aplikasi stateful kuno yang belum mendukung arsitektur multi-instance (harus di-refactor terlebih dahulu).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`healthy_host_count` vs `unhealthy_host_count` (Gauge)**: Rasio jumlah server aktif vs server yang dicap mati oleh health check.
-   **`backend_connection_errors_total` (Counter)**: Jumlah koneksi yang gagal tersambung ke backend (indikator port socket overload).
-   **`request_distribution_per_target` (Counter)**: Memastikan grafik trafik terbagi rata; jika satu server menerima 80% trafik, ada bug pada algoritma load balancing atau sticky session.
-   **`surge_queue_length` (Gauge)**: Jumlah request yang antre di buffer load balancer saat seluruh server backend sedang 100% sibuk.

## 9. Studi Kasus Industri Nyata

### Penanganan Hotspot Node & Zero-Downtime Deployment di Sistem Pembayaran

- **Tantangan Rekayasa Riil (The Problem)**:
  Sebuah platform payment gateway menggunakan algoritma standar Round-Robin pada layer proxy untuk mendistribusikan trafik ke 20 instance worker. Masalah muncul karena variasi durasi transaksi sangat heterogen: request cek saldo selesai dalam 5ms, sedangkan request rekonsiliasi pembayaran kartu memakan waktu 1.500ms. Algoritma Round-Robin terus menembakkan request baru secara buta ke server yang kebetulan sedang memproses banyak transaksi berat. Tiga node server mengalami *thread pool starvation*, CPU mencapai 100%, dan request pengguna lain yang mendarat di node tersebut mendadak terkena timeout (HTTP 504).

- **Implementasi Solusi Industri (The Architectural Solution)**:
  1. **Transisi ke Dynamic Least Connections (Peak-EWMA)**: Load balancer dikonfigurasi ulang menggunakan algoritma *Peak Exponentially Weighted Moving Average* (Peak-EWMA), yang memprioritaskan server dengan jumlah koneksi aktif paling sedikit dan latensi historis paling rendah.
  2. **Intelligent Deep Health Checking**: Endpoint `/healthz` di setiap node Go dimodifikasi agar tidak hanya mengembalikan status `200 OK`, melainkan memeriksa utilisasi goroutine aktif dan koneksi database pool. Jika goroutine aktif melampaui batas aman (misal > 5.000), node secara proaktif melaporkan status *degraded* sehingga load balancer berhenti mengirimkan beban baru untuk sementara.
  3. **Graceful Connection Draining**: Saat proses rolling update deployment di Kubernetes, ingress load balancer memberikan toleransi `deregistration_delay` selama 30 detik agar koneksi transaksi yang sedang berlangsung diselesaikan secara tuntas sebelum pod lama dimatikan.

- **Dampak & Hasil Sistem (The Engineering Outcome)**:
  - Lonjakan HTTP 504 Gateway Timeout tereliminasi hingga 0%.
  - Distribusi latensi p99 seluruh sistem menurun sebesar 62% karena tidak ada lagi server yang mengalami penumpukan antrean (*hotspot nodes*).
  - Tim engineering dapat melakukan deployment kode baru ke produksi 20 kali sehari tanpa pernah memutus transaksi pembayaran pengguna.

## 10. Fakta Menarik & Gotchas Production

-   **Health Check Flapping**: Jika konfigurasi health check terlalu agresif (misal: timeout 100ms dan gagal 2 kali langsung dicap mati), saat server sedang sibuk mengerjakan GC (Garbage Collection), load balancer akan menganggap server mati. Beban akan dialihkan ke server yang tersisa, membuat server lainnya ikut kolaps secara berantai (_cascading failure_).
-   **DNS Round Robin Bukan Load Balancer Sejati**: Banyak yang mengira memasukkan beberapa IP address di record DNS (A Record) sudah cukup sebagai load balancing. Padahal, DNS tidak tahu apakah server target sedang hidup atau mati, dan client sering meng-cache IP lama selama berjam-jam.

## 11. Rangkuman Singkat

1. **Pemerata Beban & Pencegah Hotspot**: Mendistribusikan request secara adil agar kapasitas seluruh armada server termanfaatkan maksimal tanpa ada satu server yang tumbang sendirian.
2. **Pilih Algoritma Sesuai Karakteristik**: Gunakan *Least Connections* atau *Peak-EWMA* untuk beban kerja heterogen; hindari Round-Robin statis jika ada transaksi yang memakan waktu lama.
3. **Health Check Adaptif**: Pastikan health check mencerminkan kesehatan riil dependensi sistem tanpa terlalu sensitif hingga memicu *flapping domino failure*.
4. **Connection Draining Adalah Kunci**: Berikan jendela waktu pengeringan koneksi saat deployment agar transaksi pengguna yang sedang berjalan tidak terputus di tengah jalan.
