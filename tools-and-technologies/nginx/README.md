# NGINX

## 1. Skenario Sistem Nyata

Pada gateway terdepan portal berita nasional atau platform video streaming saat terjadi lonjakan trafik masif (_breaking news_), ratusan ribu pengunjung mengakses situs secara bersamaan dalam jendela waktu sempit. Dalam hitungan 5 menit, trafik melonjak dari 10.000 menjadi **200.000 pengguna yang membuka situs secara bersamaan** melalui browser dan aplikasi seluler.

Jika kamu menghadapkan server aplikasi backend-mu secara langsung ke internet publik (misal server Python Django, Node.js Express, atau Ruby on Rails):

-   Banyak pengguna menggunakan koneksi internet seluler 3G/4G yang lambat (_slow clients_). Mengirim gambar sebesar 2 MB ke client lambat membutuhkan waktu 10 detik. Server aplikasi backend-mu yang bertipe _thread-per-connection_ atau _worker-based_ akan terikat (_blocked_) selama 10 detik penuh hanya untuk menunggu client selesai mengunduh paket data!
-   Setelah 500 thread backend terisi penuh oleh _slow clients_, server kehabisan kapasitas dan mulai menolak ribuan pengunjung berikutnya dengan error `502 Bad Gateway` atau `504 Gateway Timeout`.
-   Server aplikasi kewalahan karena harus melakukan komputasi berat negosiasi TLS/SSL handshake, dekompresi Gzip/Brotli, dan melayani file gambar statis.

**Solusi dengan NGINX:**
NGINX ditempatkan sebagai Reverse Proxy & Web Server terdepan. Menggunakan arsitektur _asynchronous event-driven (epoll)_, satu proses worker NGINX sanggup menahan **puluhan ribu koneksi TCP secara bersamaan** hanya dengan beberapa megabyte RAM! NGINX menangani SSL Termination, menampung respons dari backend secepat kilat (_response buffering_), lalu menyalurkannya perlahan ke client yang lambat, membebaskan server aplikasi backend untuk fokus memproses logika bisnis!

## 2. Apa Itu dan Buat Apa?

**NGINX (dibaca "engine-x")** adalah web server sumber terbuka berkinerja tinggi yang juga berfungsi sebagai reverse proxy, HTTP cache, load balancer, dan mail proxy.

### Peran Utama NGINX dalam Arsitektur Backend:

1. **High-Performance Web Server**: Melayani aset statis (HTML, CSS, JS, gambar, video streaming) langsung dari disk OS dengan kecepatan ekstrem tanpa menyentuh server aplikasi backend.
2. **Reverse Proxy & SSL Termination**: Menjadi pintu depan tunggal yang menerima request HTTPS publik, mendekripsi sertifikat SSL/TLS, lalu meneruskan request HTTP/2 atau HTTP/1.1 murni ke backend internal.
3. **Load Balancing**: Mendistribusikan trafik masuk ke kumpulan backend server menggunakan berbagai algoritma (Round Robin, Least Connections, IP Hash).
4. **Edge Rate Limiting & Protection**: Membatasi laju request per IP client menggunakan algoritma _Leaky Bucket_ untuk meredam serangan brute-force dan DDoS.

## 3. Kenapa Harus Pakai Ini ?

### 1. Arsitektur Asynchronous Event-Driven (Non-Blocking I/O)

Berbeda dengan web server tradisional seperti Apache HTTPD (era lama) yang membuat satu thread atau proses OS untuk setiap koneksi client (membuat RAM cepat habis saat ada ribuan koneksi), NGINX menggunakan **arsitektur Master-Worker berbasis Event Loop**. Satu proses worker NGINX ditenagai oleh kernel Linux _epoll_ multiplexing, memungkinkan satu proses menangani lebih dari 10.000 koneksi konkuren secara non-blocking dengan konsumsi RAM yang sangat stabil (hanya ~2-3 MB per worker).

### 2. Response Buffering (Penyelamat Backend dari Slow Clients)

Ketika backend Go atau Java menghasilkan respons dalam waktu 5 ms, NGINX membaca seluruh isi respons tersebut ke dalam buffer lokalnya seketika. Koneksi backend langsung ditutup (_freed_), dan NGINX yang akan menghabiskan waktu beberapa detik mengirimkan data tersebut ke browser client yang lambat. Backend server-mu tetap bebas melayani request berikutnya tanpa tercekik.

### 3. Static File Caching & Zero-Copy (`sendfile`)

NGINX memanfaatkan kernel system call Linux **`sendfile()`** untuk mentransfer file statis langsung dari disk ke network interface card (NIC) tanpa menyalin data ke memori NGINX terlebih dahulu (_zero memory copies_), menghasilkan kecepatan penyajian file statis mendekati limit kapasitas hardware.

## 4. Cara Kerjanya (Arsitektur Master-Worker & Event Loop NGINX)

```
[ Ribuan Client Ingress: Web & Mobile ]
Client 1     Client 2     Client 3 ... Client 50.000
    |            |            |               |
    +------------+------------+---------------+
                         |
                         v
            +-------------------------+
            |   NGINX Master Process  | (Membaca konfigurasi & memantau worker)
            +-------------------------+
                         |
           +-------------+-------------+
           |                           |
           v                           v
+-----------------------+   +-----------------------+
|  Worker Process 1     |   |  Worker Process 2     |  (Pin ke CPU Core)
|  [ Non-blocking Event |   |  [ Non-blocking Event |
|    Loop via epoll ]   |   |    Loop via epoll ]   |
+-----------------------+   +-----------------------+
           |                           |
           +-------------+-------------+
                         |
       (Reverse Proxy & Load Balancing)
                         |
    +--------------------+--------------------+
    |                    |                    |
    v                    v                    v
[ Backend Pod A ]   [ Backend Pod B ]   [ Backend Pod C ]
```

## 5. Cuplikan Konfigurasi Inti

### Konfigurasi Reverse Proxy, Load Balancer, dan SSL Termination (`nginx.conf`)

```nginx
events {
    # Jumlah koneksi simultan yang dapat ditangani oleh SATU worker process
    worker_connections 10240;
    use epoll;
    multi_accept on;
}

http {
    # Optimasi transfer data kernel
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    # Rate limiting: maksimal 10 request/detik per IP address
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    upstream backend_cluster {
        # Load balancing berbasis koneksi paling sedikit
        least_conn;
        server 10.0.1.10:8080 max_fails=3 fail_timeout=10s;
        server 10.0.1.11:8080 max_fails=3 fail_timeout=10s;
        server 10.0.1.12:8080 max_fails=3 fail_timeout=10s;
        keepalive 64; # Mempertahankan koneksi keepalive ke backend
    }

    server {
        listen 443 ssl http2;
        server_name api.kamusbackend.com;

        ssl_certificate /etc/ssl/certs/bundle.crt;
        ssl_certificate_key /etc/ssl/private/api.key;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 1d;

        location /static/ {
            root /var/www/assets;
            expires 30d;
            add_header Cache-Control "public, no-transform";
        }

        location /api/ {
            # Terapkan rate limiter dengan burst buffer 20 request
            limit_req zone=api_limit burst=20 nodelay;

            proxy_pass http://backend_cluster;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Buffering respons backend agar backend cepat bebas
            proxy_buffering on;
            proxy_buffers 16 16k;
            proxy_busy_buffers_size 32k;
        }
    }
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Efisiensi Memori Luar Biasa**: Menangani puluhan ribu koneksi TCP konkuren hanya dengan puluhan megabyte RAM.
-   **Performa Raw Caching & Proxying**: Latensi internal proxy sangat minim (sub-milidetik, ~0.1 ms - 0.5 ms).
-   **Stabilitas Sangat Tinggi**: Sangat jarang crash; mendukung _hot reload_ konfigurasi (`nginx -s reload`) tanpa memutus koneksi pengguna aktif (_zero-downtime config reload_).

### Kekurangan

-   **Bahasa Konfigurasi Deklaratif yang Statis**: Tidak sefleksibel proxy modern yang mendukung konfigurasi dinamis via API (seperti Envoy atau Traefik). Setiap perubahan upstream server mengharuskan reload config.
-   **Ekstensi C-Module Rumit**: Menulis modul kustom di NGINX memerlukan kompilasi ulang kode sumber C (kecuali menggunakan OpenResty / Lua module).
-   **Fitur Enterprise Berbayar di NGINX Plus**: Beberapa fitur canggih seperti metrik monitoring dashboard visual dan dynamic DNS resolving di upstream hanya tersedia di versi berbayar NGINX Plus.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Pakai NGINX

-   Pintu gerbang utama (_Edge Reverse Proxy_) di depan aplikasi web dan API backend.
-   Melayani ribuan request file statis (gambar, JS, CSS) berkecepatan tinggi.
-   SSL/TLS Termination terpusat agar server aplikasi tidak terbebani enkripsi.
-   Load balancer Layer 7 (HTTP) dan Layer 4 (TCP/UDP stream proxy).

### Kapan Mempertimbangkan Alternatif (Envoy / Traefik / HAProxy)

-   Di lingkungan Kubernetes Service Mesh yang sangat dinamis dengan pergantian ratusan pod per menit (Envoy lebih unggul karena mendukung kontrol _xDS dynamic API_).
-   Jika membutuhkan load balancing Layer 4 murni dengan algoritma health check yang sangat kaya (HAProxy sering menjadi pilihan spesifik).
-   Jika membutuhkan gateway terintegrasi dengan otomatisasi sertifikat Let's Encrypt native tanpa konfigurasi manual (Traefik / Caddy).

## 8. Metrik Kunci & Observabilitas

Metrik wajib di Grafana melalui `nginx-prometheus-exporter`:

-   **`nginx_connections_active`**: Jumlah total koneksi soket client yang sedang terbuka.
-   **`nginx_connections_reading`, `writing`, `waiting`**: Membedakan koneksi yang sedang membaca header, mengirim respons (_writing_), atau sedang idle (_waiting_ / keep-alive).
-   **`nginx_http_requests_total` (per Status Code)**: Tingkat request per detik (RPS) dikelompokkan berdasarkan kode status (2xx, 4xx, 5xx); lonjakan 502/504 menandakan backend cluster tumbang.
-   **`nginx_upstream_response_time_seconds`**: Waktu yang dibutuhkan upstream backend server untuk membalas request ke NGINX.

## 9. Studi Kasus Nyata di Industri

### Penanganan 250.000 RPS Ingress Traffic dan Terminasi SSL Terpusat pada E-Commerce Skala Besar

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah portal e-commerce nasional menghadapi lonjakan trafik hingga 250.000 request per detik saat festival belanja tanggal kembar. Awalnya, enkripsi SSL/TLS dan kompresi gzip dieksekusi langsung di dalam container aplikasi Go dan Node.js. Beban berat kriptografi handshake TLS dan kompresi payload menghabiskan 60% siklus CPU pod aplikasi, memicu *garbage collection pauses*, menurunkan throughput API hingga 50%, dan menaikkan latensi p99 checkout dari 120 ms menjadi 4.500 ms.
- **Implementasi Solusi Industri (The Architectural Solution)**: Tim menempatkan cluster *NGINX Reverse Proxy* sebagai lapisan edge ingress terdepan. NGINX dikonfigurasi menggunakan arsitektur *event-driven non-blocking* (epoll), worker processes disesuaikan dengan jumlah CPU core fisik, dan `worker_rlimit_nofile` dinaikkan ke 65535. Seluruh kalkulasi enkripsi HTTPS di-offload ke NGINX dengan session caching SSL (`ssl_session_cache shared:SSL:50m`) dan dukungan HTTP/2. Aset statis (gambar produk dan bundel JS) disajikan langsung dari memory disk cache NGINX dengan header `sendfile on;` dan `tcp_nopush on;`.
- **Dampak & Hasil Sistem (The Outcome)**: Konsumsi CPU pada server aplikasi backend anjlok dari 85% menjadi 22%, membebaskan seluruh daya komputasi container untuk logika bisnis. Waktu handshake TLS berkurang 70% berkat session resumption. Latensi p99 seluruh gateway API turun ke angka stabil 45 ms, dan NGINX melayani 250.000 RPS secara mulus hanya dengan 4 instans compute c5.2xlarge.

## 10. Fakta Menarik & Gotchas Production

-   **The Worker Connections & Ulimit Mismatch**:  
    Banyak engineer menyetel `worker_connections 20000;` di `nginx.conf`, namun saat lonjakan trafik tiba, NGINX langsung melempar error fatal **`24: Too many open files`**. Ini terjadi karena batas file descriptor sistem operasi Linux (`ulimit -n`) masih berada di default `1024`!  
    _Solusi Wajib:_ Tambahkan direktif **`worker_rlimit_nofile 65535;`** di blok utama `nginx.conf` agar NGINX diizinkan membuka ribuan socket file descriptor di level kernel.
-   **The DNS Resolution Caching Trap di Upstream**:  
    Secara default, NGINX hanya me-resolve nama domain upstream (misal `server backend.internal.com;`) **SATU KALI saat NGINX pertama kali startup**! Jika backend-mu berada di AWS ALB atau Kubernetes Service yang IP-nya berganti secara berkala, NGINX akan terus mengirim request ke alamat IP lama yang sudah mati, menghasilkan error `502 Bad Gateway`!  
    _Solusi Wajib:_ Gunakan variabel proxy pass bersama direktif resolver:  
    `resolver 10.0.0.2 valid=10s; set $upstream_endpoint "http://backend.internal.com"; proxy_pass $upstream_endpoint;`

## 11. Rangkuman Singkat

- **Arsitektur Event-Driven Berkinerja Ekstrem**: NGINX menggunakan event loop asynchronous (epoll/kqueue) yang mampu melayani puluhan ribu koneksi konkuren per worker process dengan konsumsi RAM yang sangat minim.
- **Offload SSL/TLS dan Kompresi di Edge**: Bebaskan server aplikasi backend dari beban kriptografi; pusatkan terminasi sertifikat HTTPS, HTTP/2 termination, dan kompresi gzip/brotli di layer reverse proxy NGINX.
- **Penyajian File Statis Berkecepatan Tinggi**: Manfaatkan direktif `sendfile on;` dan `tcp_nopush on;` agar Linux kernel mentransfer byte data dari disk langsung ke socket jaringan tanpa menyalinnya ke user-space buffer.
- **Sinkronkan Limit File Descriptor OS**: Pastikan direktif `worker_rlimit_nofile` diatur sejajar dengan kapasitas `worker_connections` agar server tidak kehabisan soket dan crash dengan error `24: Too many open files`.
