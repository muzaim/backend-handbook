# HAProxy (High Availability Load Balancer)

## 1. Skenario Sistem Nyata

Pada sebuah platform financial technology (fintech) dan bursa pertukaran kripto, sistem gerbang tepi (_Edge Gateway_) menerima lebih dari 250.000 request per detik (RPS) dari aplikasi mobile, bot trading frekuensi tinggi (HFT), dan mitra perbankan.

Karakteristik trafik yang masuk sangat beragam dan menuntut performa ekstrem:

-   Jutaan koneksi WebSocket persisten yang harus tetap terbuka selama berjam-jam untuk streaming harga pasar (_live order book_).
-   Panggilan API RESTful yang menuntut latensi serendah mungkin (< 2 ms) dengan algoritma pembagian beban yang adil (_Least Connections_).
-   Protokol biner TCP murni (seperti ISO-8583 untuk transfer ATM) yang tidak menggunakan HTTP sama sekali.

Jika gerbang depan ini ditangani oleh server web serbaguna yang tidak dioptimalkan murni untuk routing jaringan:

-   Setiap koneksi TCP yang menggantung memakan alokasi buffer memori yang besar, membuat server kehabisan RAM (_memory thrashing_).
-   Saat salah satu node backend Go mengalami _crash_, load balancer terlambat mendeteksi kegagalan tersebut sehingga ribuan pengguna menerima error `502 Bad Gateway`.

**Solusi dengan HAProxy:**
Infrastruktur menempatkan sepasang instance **HAProxy** di layer terdepan. HAProxy dirancang murni sebagai proxy dan load balancer Layer 4 (TCP) dan Layer 7 (HTTP) berkecepatan tinggi dengan arsitektur _event-driven single-threaded / multi-threaded_. Satu instance HAProxy sanggup menampung **ratusan ribu koneksi aktif sekaligus** dengan konsumsi RAM yang sangat kecil dan deterministik. Dengan fitur _Active Health Checks_ yang agresif, HAProxy mendeteksi node Go yang tumbang dalam hitungan milidetik dan secara otomatis mengalihkan trafik ke node yang sehat tanpa ada satu pun request pengguna yang gagal.

## 2. Apa Itu dan Buat Apa?

HAProxy (_High Availability Proxy_) adalah reverse proxy dan load balancer open-source berkinerja tinggi yang menjadi standar emas industri untuk mendistribusikan beban lalu lintas jaringan.

Karakteristik utama:

-   **Dua Mode Operasi (Layer 4 & Layer 7)**:
    -   **Mode TCP (Layer 4)**: Meneruskan paket data biner mentah secara langsung tanpa membedah payload aplikasi (sangat cepat untuk database PostgreSQL/MySQL, broker RabbitMQ, atau redis proxy).
    -   **Mode HTTP (Layer 7)**: Menganalisis header HTTP, path URL, cookie session, dan menghentikan enkripsi TLS (_TLS Termination_).
-   **Efisiensi Memori Legendaris**: Dirancang dengan model alokasi memori zero-copy dan arsitektur event non-blocking berbasis `epoll` di Linux.
-   **Runtime Socket API**: Memungkinkan tim DevOps mengubah bobot server (_weight_), mematikan node untuk maintenance (_drain mode_), atau menambahkan upstream server secara dinamis tanpa perlu me-restart proses HAProxy.

## 3. Kenapa Harus Pakai Ini ?

Di level performa throughput murni, HAProxy sering kali mengungguli web server konvensional:

1. **Performa Saturasi Jaringan Maksimal**: Mampu memaksimalkan kapasitas kartu jaringan 10 Gbps hingga 40 Gbps dengan latensi sub-milidetik.
2. **Algoritma Load Balancing Kaya**: Mendukung _Round-Robin_, _Leastconn_ (sangat ideal untuk request dengan durasi proses variatif), _Source IP Hash_, dan _Consistent Hashing_.
3. **Pendeteksi Kesehatan Node Sangat Cepat (_Fast Health Checking_)**: Mendukung health check berbasis HTTP, TCP, maupun eksternal agent check untuk memastikan request tidak pernah dikirim ke backend yang sakit.

## 4. Cara Kerjanya (Diagram Alur HAProxy)

```
                       +----------------------+
                       | Incoming Traffic     |
                       | (250,000 RPS / WSS)  |
                       +----------------------+
                                  |
                                  v
+------------------------------------------------------------------+
|                            HAProxy                               |
|                                                                  |
| [Frontend: public_http]                                          |
| * Bind :80, :443 (TLS Termination)                               |
| * ACL path_beg /api/orders  --> use_backend order_cluster        |
| * ACL path_beg /ws/market   --> use_backend websocket_cluster    |
+------------------------------------------------------------------+
              |                                    |
              | balance leastconn                  | balance roundrobin
              v                                    v
+-----------------------------+      +-----------------------------+
| [Backend: order_cluster]    |      | [Backend: websocket_cluster]|
| (Active Health Check /health)|     | (Active Health Check /health)|
+-----------------------------+      +-----------------------------+
        /             \                      /             \
       v               v                    v               v
+-------------+ +-------------+      +-------------+ +-------------+
| Go App #1   | | Go App #2   |      | WS Node #1  | | WS Node #2  |
| (:8081)     | | (:8082)     |      | (:9001)     | | (:9002)     |
+-------------+ +-------------+      +-------------+ +-------------+
```

## 5. Cuplikan Konfigurasi & Integrasi Go

### Konfigurasi Produksi `haproxy.cfg` (Layer 7 dengan Active Health Check)

```haproxy
global
    log /dev/log local0 info
    maxconn 100000          ; Menampung hingga 100.000 koneksi konkuren
    user haproxy
    group haproxy
    daemon
    stats socket /var/run/haproxy.sock mode 660 level admin  ; Runtime Socket API

defaults
    log     global
    mode    http            ; Mode Layer 7 HTTP
    option  httplog
    option  dontlognull
    timeout connect 5000ms  ; Batas waktu koneksi ke upstream server
    timeout client  50000ms ; Batas waktu idle dari client
    timeout server  50000ms ; Batas waktu respons dari upstream backend
    retries 3               ; Coba ulang otomatis jika koneksi gagal

frontend public_ingress
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/company.pem alpn h2,http/1.1

    # Aturan ACL Routing berdasarkan Path URL
    acl is_order_api path_beg /api/v1/orders

    use_backend order_service_backend if is_order_api
    default_backend default_fallback_backend

backend order_service_backend
    balance leastconn       ; Algoritma: kirim ke server dengan koneksi paling sedikit
    option httpchk GET /healthz  ; Active Health Check ke endpoint aplikasi Go
    http-check expect status 200

    # Server pool upstream dengan interval check setiap 2 detik (inter 2000)
    server order_app_1 10.0.1.101:8080 check inter 2000 fall 3 rise 2 weight 100
    server order_app_2 10.0.1.102:8080 check inter 2000 fall 3 rise 2 weight 100
    server order_app_backup 10.0.1.103:8080 check backup  ; Server cadangan
```

### Handler Health Check Standar di Aplikasi Go

Aplikasi Go harus menyediakan endpoint `/healthz` yang ringan dan cepat untuk merespon probe HAProxy:

```go
package main

import (
	"database/sql"
	"net/http"
)

func HealthCheckHandler(db *sql.DB) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// Periksa kesehatan internal (misal konektivitas database)
		if err := db.PingContext(r.Context()); err != nil {
			// Balas non-200 agar HAProxy segera mencabut node ini dari pool aktif
			http.Error(w, "Database down", http.StatusServiceUnavailable)
			return
		}

		w.WriteHeader(http.StatusOK)
		w.Write([]byte("OK"))
	}
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Performa Throughput & Konkurensi Ekstrem**: Sangat stabil menangani ratusan ribu koneksi konkuren tanpa degradasi memori.
-   **Dukungan Murni Layer 4 & Layer 7**: Sangat fleksibel mem-proxy protokol HTTP/gRPC maupun koneksi TCP murni database/messaging.
-   **Observabilitas Bawaan Luar Biasa**: Dilengkapi dashboard metrik web bawaan (_HAProxy Stats Page_) dan socket runtime untuk kontrol dinamis.

### Kekurangan

-   **Bukan Web Server Statis**: HAProxy tidak dirancang untuk membaca file HTML, CSS, atau gambar langsung dari hard disk (berbeda dengan NGINX atau Apache).
-   **Tidak Memiliki Modul Scripting Fleksibel**: Tidak memiliki integrasi scripting sepopuler modul Lua di OpenResty/NGINX.
-   **Sintaks Konfigurasi Khusus**: Konfigurasi berbasis ACL memiliki format tersendiri yang membutuhkan kurva pembelajaran bagi pemula.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Layer load balancer terdepan di depan ratusan node aplikasi backend Go, gRPC service, atau WebSocket cluster.
-   Load balancing layer 4 untuk cluster database (PostgreSQL Read Replicas, MySQL, Galera Cluster) atau RabbitMQ.
-   Sistem dengan volume koneksi TCP persisten jangka panjang (_Long-lived connections_).

### Kapan Jangan Pakai

-   Kebutuhan sederhana melayani file website statis (HTML/JS/gambar) di mana NGINX atau CDN jauh lebih cocok.
-   Kebutuhan API Gateway modern dengan manipulasi request dinamis berbasis JavaScript/Lua (misal Kong atau Apache APISIX).
-   Aplikasi kecil monolitik tunggal yang tidak memiliki upstream server pool.

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`haproxy_frontend_current_sessions` (Gauge)**: Jumlah koneksi aktif yang sedang ditangani frontend proxy.
-   **`haproxy_backend_response_time_average_seconds` (Gauge)**: Rata-rata waktu respon upstream server backend.
-   **`haproxy_server_status` (Gauge: 1=UP, 0=DOWN)**: Status kesehatan masing-masing node server. Pasang alert darurat jika ada node yang DOWN.
-   **`haproxy_backend_current_queue` (Gauge)**: Jumlah request yang sedang mengantre menunggu server backend siap. Jika > 0, server backend kewalahan.

## 9. Studi Kasus Industri Nyata

### Penyeimbangan Beban 300.000 Koneksi TCP Persisten dan Failover PostgreSQL Sub-Detik

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah platform transaksi keuangan dan gaming real-time mengelola armada ribuan koneksi WebSocket persisten dan cluster database PostgreSQL berukuran besar (1 Primary dan 4 Read Replicas). Awalnya, load balancer cloud berbasis DNS digunakan untuk mengarahkan trafik database dan websocket. Ketika node primary database mengalami hardware failure, propagasi DNS membutuhkan waktu hingga 60 detik, menyebabkan ribuan aplikasi terus mengirim query transaksi write ke node yang mati (menghasilkan ribuan pesan error `500 Server Error`). Selain itu, load balancer cloud sering memutus koneksi WebSocket klien secara acak tanpa peringatan karena keterbatasan penanganan konkurensi koneksi persisten.
- **Implementasi Solusi Industri (The Architectural Solution)**: Tim menempatkan sepasang server *HAProxy* aktif-pasif (dikelola via Keepalived VRRP) di lapisan load balancing Layer 4 dan Layer 7. Untuk cluster database, HAProxy dikonfigurasi dalam mode TCP stream dengan algoritma `roundrobin` dan pemeriksaan kesehatan berkala via *HTTP health check probe* (`http-check expect status 200` ke agent Patroni). Begitu Patroni mempromosikan replica baru saat primary mati, HAProxy mendeteksi perubahan status dalam waktu 500 ms dan secara otonom membelokkan trafik penulisan ke primary baru tanpa campur tangan DNS. Untuk WebSocket, HAProxy menyetel `timeout tunnel 3600s` dan mengalokasikan `maxconn 150000` dengan kernel socket tuning.
- **Dampak & Hasil Sistem (The Outcome)**: Waktu pemulihan failover database otomatis (RTO) terpangkas dari 60 detik menjadi < 1 detik dengan *zero data write loss*. Armada HAProxy menangani 300.000 koneksi persisten bersamaan secara stabil dengan utilisasi CPU di bawah 12% dan konsumsi RAM yang stabil. Insiden putusnya koneksi WebSocket turun hingga 99,9%, memberikan pengalaman streaming dan transaksi tanpa interupsi bagi pengguna.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Batas Limit File Descriptors OS (`ulimit -n`) Terlalu Rendah**: Menyetel `maxconn 100000` di konfigurasi HAProxy, tetapi lupa menaikkan limit `nofile` di sistem operasi Linux (default biasanya hanya 1024). HAProxy akan crash saat koneksi menyentuh angka 1024.  
   _Solusi_: Atur `fs.file-max` dan `/etc/security/limits.conf` ke nilai minimal 250.000.
2. **Reload Konfigurasi Memutus Koneksi Lama**: Melakukan reload biasa (`systemctl reload haproxy`) pada server dengan ribuan koneksi WebSocket. Koneksi lama akan terputus seketika.  
   _Solusi_: Gunakan fitur _Seamless Reload_ (flag `-x` untuk transfer listen socket) yang menjaga koneksi lama tetap hidup hingga selesai alami.
3. **DNS Caching Statis**: HAProxy secara default menyelesaikan IP hostname upstream saat startup. Jika upstream berada di AWS ELB/Kubernetes yang IP-nya dinamis berubah, HAProxy akan terus menembak IP lama yang sudah mati.  
   _Solusi_: Konfigurasikan blok `resolvers` bawaan di `haproxy.cfg`.

## 11. Rangkuman Singkat

- **Spesialis Load Balancing L4 & L7 Berperforma Tinggi**: HAProxy dirancang khusus untuk mem-proxy dan menyeimbangkan beban trafik jaringan (TCP, HTTP, gRPC, WebSocket) dengan latensi sub-milidetik dan throughput luar biasa.
- **Pemeriksaan Kesehatan Tingkat Lanjut**: HAProxy menyediakan health check aktif berlapis (TCP handshake, HTTP response status, hingga regex response payload) untuk mendeteksi node rusak secara instan.
- **Dukungan Seamless Reload**: Gunakan flag `-x` saat reload konfigurasi produksi agar proses baru mewarisi listen socket proses lama tanpa memutus ribuan koneksi persisten yang sedang aktif.
- **Bukan Web Server Statis**: HAProxy tidak melayani file disk statis (HTML/gambar); kombinasikan HAProxy dengan NGINX atau CDN untuk kebutuhan penyajian aset statis.
