# Reverse Proxy

## 1. Skenario Sistem Nyata

Pada arsitektur aplikasi web modern berskala besar, runtime backend Go atau Node.js biasanya berjalan di port internal (seperti `3000`) pada private network yang melayani ribuan request konkuren sekaligus.

Jika server aplikasi langsung diekspos ke internet publik:
- Setiap pengguna melakukan negosiasi enkripsi dan dekripsi HTTPS (*TLS handshake*) langsung ke runtime aplikasi, menghabiskan 30-40% siklus CPU hanya untuk operasi kriptografi alih-alih mengeksekusi logika bisnis database.
- File aset statis (gambar produk, CSS, JavaScript bundle) harus dibaca dari disk oleh thread aplikasi backend, menyita connection pool dan alokasi memori.
- Alamat IP asli server database dan backend terekspos ke publik, rentan terhadap port scanning dan serangan DDoS langsung.

**Solusi dengan Reverse Proxy (misal: NGINX / Envoy):**
Kamu menempatkan NGINX di port publik `80/443`. NGINX menangani terminasi SSL/TLS di edge jaringan, menyajikan aset statis langsung dari disk cache dengan kecepatan tinggi, mengompresi response dengan Gzip/Brotli, lalu meneruskan request HTTP murni yang bersih ke backend di `http://127.0.0.1:3000`. Alamat IP dan topologi internal backend tetap tersembunyi aman di jaringan privat.

## 2. Apa Itu dan Buat Apa?

Reverse proxy adalah server perantara yang berada di depan satu atau lebih web server backend. Tugas utamanya adalah menerima request dari client (internet publik) dan meneruskannya (*forwarding*) ke server internal yang sesuai di jaringan privat.

Fungsi utama reverse proxy:
- **Menyembunyikan identitas backend**: Alamat IP asli server aplikasi internal tidak terekspos ke internet.
- **SSL/TLS Termination**: Menangani enkripsi/dekripsi HTTPS sehingga server aplikasi di belakangnya cukup melayani HTTP biasa yang ringan.
- **Kompresi data**: Mengompresi response (Gzip/Brotli) sebelum dikirim kembali ke client untuk menghemat bandwidth.
- **Proteksi awal**: Menyaring trafik kotor atau serangan dasar sebelum menyentuh aplikasi backend.

Contoh software populer: NGINX, HAProxy, Envoy, Traefik, Caddy.

## 3. Kenapa Harus Pakai Ini di High-Throughput?

Ketika server melayani puluhan ribu request per detik, CPU server aplikasi (Node.js, Go, Python, Java) sangat mahal jika harus terbebani oleh *TLS Handshake* (proses enkripsi kriptografi) dan kompresi file statis. 

Jika server aplikasi langsung diekspos ke publik:
- Setiap koneksi TCP yang lambat (*slowloris attack*) akan mengunci worker thread aplikasi backend hingga kehabisan memory.
- Menambahkan atau mematikan server backend (*rolling update*) akan menyebabkan *downtime* karena client menyimpan IP server langsung.

Dengan reverse proxy, koneksi luar ditangani oleh server C/C++ yang sangat efisien dalam menangani *event-driven network I/O*, membebaskan server aplikasi untuk murni mengeksekusi logika bisnis.

## 4. Cara Kerjanya (Diagram Alur)

```
+-------------+
|   Client    | (Internet Publik)
+-------------+
       |
       | 1. HTTPS Request (Port 443)
       v
+-------------------------------+
|         Reverse Proxy         |
|  (NGINX / Envoy / HAProxy)    |
|                               |
| - SSL/TLS Decryption          |
| - Header Injection (X-Forward)|
| - Connection Pooling          |
+-------------------------------+
       |
       | 2. HTTP Internal (Port 8080/8081)
       +-----------------------+
       |                       |
       v                       v
+---------------+       +---------------+
| Backend App 1 |       | Backend App 2 |
| (Private IP)  |       | (Private IP)  |
+---------------+       +---------------+
```

## 5. Cuplikan Logika Inti (Konfigurasi & Forwarding)

Berikut representasi logika forwarding inti pada reverse proxy (dalam format deklaratif NGINX & pseudocode):

```nginx
# Contoh Logika Forwarding NGINX
upstream backend_cluster {
    server 10.0.1.10:8080 max_fails=3 fail_timeout=10s;
    server 10.0.1.11:8080 max_fails=3 fail_timeout=10s;
    keepalive 64; # Connection pooling ke upstream
}

server {
    listen 443 ssl http2;
    server_name api.kamusbackend.com;

    ssl_certificate     /etc/ssl/certs/bundle.crt;
    ssl_certificate_key /etc/ssl/private/api.key;

    location / {
        proxy_pass http://backend_cluster;
        proxy_http_version 1.1;
        proxy_set_header Connection ""; # Wajib untuk HTTP Keep-Alive
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 2s;
        proxy_read_timeout 5s;
    }
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Keamanan tinggi**: Backend tersembunyi di dalam private subnet / VPC.
- **Efisiensi komputasi**: Offloading TLS dan static assets dari server aplikasi inti.
- **Fleksibilitas deployment**: Server backend bisa di-restart atau diganti tanpa memutus endpoint publik.

### Kekurangan
- **Single point of failure (SPOF)**: Jika reverse proxy mati dan tidak memiliki redundansi, seluruh sistem mati.
- **Sedikit overhead latensi**: Menambahkan satu hop jaringan tambahan (biasanya sub-milidetik, tetapi tetap ada).
- **Kompleksitas konfigurasi**: Perlu pemeliharaan sertifikat SSL, buffer size, timeout, dan header forwarding yang teliti.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Setiap aplikasi production yang melayani trafik publik melalui internet.
- Sistem yang memerlukan terminasi HTTPS terpusat.
- Ketika memiliki beberapa service internal di domain yang sama (misal `/api` ke Go dan `/static` ke CDN/storage).

### Kapan Jangan Pakai
- Lingkungan development lokal kecil di mana menjalankan proxy tambahan hanya membuang waktu setup.
- Komunikasi antar microservice internal dalam satu service mesh murni (yang sudah diatur oleh sidecar proxy seperti Linkerd atau Envoy sidecar).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

Saat mengawasi reverse proxy di Grafana atau Datadog, berikut metrik utama yang wajib dipantau:
- **`active_connections` (Gauge)**: Jumlah koneksi TCP client yang sedang terbuka secara bersamaan.
- **`upstream_response_time_seconds` (Histogram/p95/p99)**: Waktu yang dihabiskan server backend untuk merespons request. Jika grafik ini melonjak tajam, masalah ada di aplikasi backend, bukan di proxy.
- **`upstream_connect_time_seconds` (Histogram)**: Waktu yang dibutuhkan proxy untuk membuka koneksi TCP ke backend. Jika > 1 detik, connection pool upstream penuh atau network jenuh.
- **`http_status_codes_total{status=~"5xx"}` (Counter)**: Lonjakan HTTP 502 (Bad Gateway) atau 504 (Gateway Timeout) adalah indikator instan bahwa backend sedang tumbang.

## 9. Studi Kasus Nyata di Industri

- **Cloudflare**: Menjalankan jaringan reverse proxy global berbasis NGINX (dan kini bermigrasi ke engine Rust buatan sendiri bernama *Pingora*) di lebih dari 300 kota di dunia. Reverse proxy ini memproses puluhan triliun request per hari, menangani terminasi TLS, mitigasi DDoS, dan caching di tepi jaringan (*edge*).
- **GitHub**: Menggunakan **HAProxy** dan **Envoy** sebagai reverse proxy terdepan untuk menerima seluruh trafik Git SSH dan HTTPS. GitHub memanfaatkan kemampuan Envoy dalam melakukan dynamic hot-reloading konfigurasi tanpa memutus koneksi streaming git push/pull yang sedang berjalan.

## 10. Fakta Menarik & Gotchas Production

- **Masalah X-Forwarded-For**: Backend sering salah membaca IP pengunjung menjadi IP si Reverse Proxy. Kamu wajib mengonfigurasi header `X-Forwarded-For` dan `X-Real-IP` agar fitur logging, geolokasi, dan anti-fraud tidak keliru.
- **Connection Keep-Alive ke Upstream**: Banyak developer lupa menyalakan HTTP keep-alive antara reverse proxy dan backend server. Akibatnya, reverse proxy membuka-tutup socket TCP baru untuk setiap request, yang memicu masalah *port exhaustion* (*TIME_WAIT storm*) saat trafik melonjak tajam.
