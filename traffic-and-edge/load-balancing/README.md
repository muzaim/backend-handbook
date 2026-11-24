# Load Balancing

## 1. Skenario Sistem Nyata

Pada sistem backend pemesanan tiket konser musik atau flash sale e-commerce dengan lonjakan trafik mendadak hingga 40.000 HTTP request per detik saat penjualan dibuka:

Jika seluruh trafik ini diarahkan ke satu server backend tunggal (misal instance 8-Core CPU, 16GB RAM):
- Server tersebut hanya sanggup memproses sekitar 2.000 request per detik sebelum socket connection pool habis, CPU menyentuh 100%, dan memori meledak (*OOM crash*).
- 38.000 pengguna lainnya mengalami timeout, error `HTTP 502 Bad Gateway`, dan transaksi gagal massal.
- Server tunggal tersebut menjadi titik kegagalan fatal (*Single Point of Failure* / SPOF). Jika mesin tersebut mati mendadak, seluruh bisnis lumpuh total.

**Solusi dengan Load Balancer (misal: HAProxy / AWS ALB / NGINX):**
Kamu menduplikasi aplikasi menjadi 20 instance pod backend identik (`Backend 1` hingga `Backend 20`) di jaringan privat. Load Balancer ditempatkan di depan dengan satu alamat IP publik. Saat 40.000 request per detik masuk, Load Balancer membagi beban secara merata (misal masing-masing 2.000 req/s) menggunakan algoritma seperti Round-Robin atau Least Connections. Jika `Backend 7` tiba-tiba mengalami crash, Load Balancer mendeteksi lewat *health check* dalam hitungan detik dan langsung mengalihkan request ke 19 server lain yang sehat dengan *zero downtime*.

## 2. Apa Itu dan Buat Apa?

Load Balancing adalah teknik mendistribusikan beban lalu lintas jaringan (*network traffic*) atau beban komputasi secara merata ke beberapa server aplikasi backend (disebut *upstream pool* atau *server farm*).

Tujuan utamanya adalah:
- **Mencegah overload**: Memastikan tidak ada satu server pun yang kewalahan sementara server lain menganggur.
- **Toleransi kegagalan (*High Availability*)**: Melakukan *Health Check* secara berkala. Jika ada satu server backend yang mati atau crash, load balancer otomatis menghentikan pengiriman trafik ke server tersebut.
- **Skalabilitas horizontal (*Scale-Out*)**: Memungkinkan kita menambah mesin server baru kapan saja saat trafik naik tanpa memutus layanan.

### Algoritma Load Balancing Populer:
- **Round Robin**: Membagi request secara bergiliran (1, 2, 3, 1, 2, 3...).
- **Least Connections**: Mengirimkan request ke server yang saat itu memiliki jumlah koneksi aktif paling sedikit (cocok untuk request yang durasi prosesnya bervariasi).
- **IP Hash / Consistent Hashing**: Menentukan server tujuan berdasarkan hash dari alamat IP client (memastikan client yang sama selalu mendarat di server yang sama).
- **Weighted Round Robin**: Membagi trafik berdasarkan kapasitas hardware mesin (mesin 16-core dapat porsi lebih banyak dari mesin 4-core).

Contoh software & hardware: NGINX, HAProxy, AWS ALB/NLB, Cloudflare, F5 BIG-IP.

## 3. Kenapa Harus Pakai Ini di High-Throughput?

Tidak peduli seberapa mahal dan canggih spesifikasi sebuah server (misal server 128 Core RAM 512GB), ada batas fisik maksimal (*vertical scaling limit*). Ketika sistem harus melayani 100.000 request per detik, satu server tunggal pasti akan mengalami kegagalan memori, thermal throttling, atau bottleneck socket TCP.

Selain itu, jika hanya ada satu server, pembaruan aplikasi (*deployment*) atau kerusakan hardware akan langsung menyebabkan pemadaman total (*total outage*). 

Dengan load balancing, sistem bisa memanfaatkan 20 server kecil yang murah (*horizontal scaling*). Jika 2 server mati mendadak di tengah malam, sistem tetap berjalan normal 100% bagi pengguna.

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

Jika Server B mendadak gagal merespons *Health Check*:

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

Berikut implementasi algoritma Round Robin dengan seleksi node sehat (*health-aware*) di Go:

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
- **Zero Downtime Deployment**: Kita bisa mematikan server satu per satu untuk update kode (*rolling update*).
- **Ketahanan tinggi (*Fault Tolerance*)**: Otomatis mendeteksi dan mengisolasi server yang bermasalah.
- **Efisiensi biaya**: Lebih fleksibel dan murah menambah server skala menengah daripada membeli satu super-komputer.

### Kekurangan
- **Manajemen Session State (*Sticky Session problem*)**: Server backend harus stateless. Jika login session disimpan di memori server lokal, request pengguna berikutnya yang mendarat di server lain akan menyebabkan pengguna logout mendadak. Solusinya: simpan session di Redis terpusat atau gunakan stateless JWT.
- **Kompleksitas Layer 4 vs Layer 7**: Memilih antara L4 balancer (berbasis TCP packet, sangat kencang tapi buta isi request) atau L7 balancer (berbasis HTTP header/path, pintar tapi butuh CPU lebih besar).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Setiap sistem yang membutuhkan ketersediaan tinggi (*High Availability 99.9% SLA*).
- Ketika trafik aplikasi sudah melampaui kemampuan satu mesin server.
- Untuk mengimplementasikan strategi rilis bertahap (*Canary deployment* atau *Blue-Green deployment*).

### Kapan Jangan Pakai
- Server internal utilitas kecil atau prototipe MVP tahap awal di mana downtime beberapa menit tidak berdampak bisnis.
- Aplikasi stateful kuno yang belum mendukung arsitektur multi-instance (harus di-refactor terlebih dahulu).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

- **`healthy_host_count` vs `unhealthy_host_count` (Gauge)**: Rasio jumlah server aktif vs server yang dicap mati oleh health check.
- **`backend_connection_errors_total` (Counter)**: Jumlah koneksi yang gagal tersambung ke backend (indikator port socket overload).
- **`request_distribution_per_target` (Counter)**: Memastikan grafik trafik terbagi rata; jika satu server menerima 80% trafik, ada bug pada algoritma load balancing atau sticky session.
- **`surge_queue_length` (Gauge)**: Jumlah request yang antre di buffer load balancer saat seluruh server backend sedang 100% sibuk.

## 9. Studi Kasus Nyata di Industri

- **Google (Maglev)**: Google mendesain software load balancer terdistribusi bernama **Maglev** yang berjalan di atas server komoditas Linux biasa. Maglev menangani seluruh lalu lintas Google Search dan YouTube menggunakan teknik *Consistent Hashing* khusus yang sanggup menyerap jutaan paket data per detik tanpa merusak koneksi TCP pengguna meski ada node Maglev yang di-restart.
- **GitHub**: Berhasil bermigrasi ke load balancing berbasis **GLB (GitHub Load Balancer)** yang beroperasi di Layer 4 menggunakan teknologi kernel Linux eBPF/XDP. GLB memungkinkan scaling horizontal tanpa terputusnya koneksi TCP yang sedang aktif (*resilient connection draining*).

## 10. Fakta Menarik & Gotchas Production

- **Health Check Flapping**: Jika konfigurasi health check terlalu agresif (misal: timeout 100ms dan gagal 2 kali langsung dicap mati), saat server sedang sibuk mengerjakan GC (Garbage Collection), load balancer akan menganggap server mati. Beban akan dialihkan ke server yang tersisa, membuat server lainnya ikut kolaps secara berantai (*cascading failure*).
- **DNS Round Robin Bukan Load Balancer Sejati**: Banyak yang mengira memasukkan beberapa IP address di record DNS (A Record) sudah cukup sebagai load balancing. Padahal, DNS tidak tahu apakah server target sedang hidup atau mati, dan client sering meng-cache IP lama selama berjam-jam.
