# Kong API Gateway

## 1. Analogi Sederhana & Skenario Nyata

Bayangkan sebuah bandara internasional yang sangat sibuk:

Setiap penumpang (request dari aplikasi mobile atau web) yang mendarat tidak boleh langsung berjalan bebas masuk ke kota tujuan (internal microservices seperti *Order Service*, *Payment Service*, atau *User Service*). Sebelum diizinkan masuk, semua penumpang wajib melewati satu gerbang pemeriksaan terpadu:
1. **Pemeriksaan Paspor & Visa (Autentikasi & Autorisasi)**: Petugas imigrasi memeriksa apakah penumpang memiliki identitas yang valid.
2. **Mesin X-Ray & Pemindai Bagasi (Security & WAF)**: Memastikan tidak ada barang terlarang atau serangan berbahaya seperti SQL Injection atau DDoS.
3. **Pemberian Jalur Khusus (Routing & Load Balancing)**: Mengarahkan penumpang jalur diplomatik/VIP vs ekonomi ke terminal bus yang tepat.
4. **Pencatatan Stempel Kedatangan (Logging & Metrics)**: Petugas mencatat berapa banyak wisatawan yang masuk setiap menitnya.

Jika setiap gedung kantor di kota harus membangun pos imigrasi dan mesin X-Ray sendiri, biaya keamanan akan meledak dan sistem pemeriksaan menjadi tidak konsisten. **Kong Gateway** bertindak sebagai gerbang terpadu bandara tersebut bagi seluruh ekosistem backend kalian.

---

## 2. Definisi & Konsep Inti

**Kong Gateway** adalah *cloud-native API Gateway* berkinerja tinggi yang dibangun di atas fondasi **NGINX** dan mesin **Lua (OpenResty)**. Kong bertindak sebagai perantara (*reverse proxy*) di depan sekumpulan microservice untuk menangani fungsi-fungsi lintas sektoral (*cross-cutting concerns*).

Abstraksi inti di dalam Kong terdiri dari:
1. **Service**: Representasi dari microservice backend internal kalian (misalnya `order-service` yang berjalan di `http://order-api:8080`).
2. **Route**: Aturan pencocokan request dari client publik (misalnya jika ada HTTP request dengan path prefix `/api/v1/orders` atau host `api.toko.com`). Sebuah Route dipetakan ke satu Service.
3. **Upstream & Target**: Mekanisme penyeimbang beban (*Load Balancer*) bawaan Kong untuk mendistribusikan trafik ke banyak instance backend (misal target `10.0.1.5:8080` dan `10.0.1.6:8080`) dengan *health checking* aktif/pasif.
4. **Plugin**: Modul fungsionalitas yang disematkan secara dinamis ke Route, Service, atau tingkat global (misal plugin `jwt`, `rate-limiting`, `cors`, `proxy-cache`, atau `prometheus`).
5. **Consumer**: Entitas klien terautentikasi (pengguna aplikasi mobile, partner API eksternal) yang dilekatkan kredensial khusus (API Key, OAuth2 token, JWT).

### Mode Operasional: DB-less (Declarative) vs PostgreSQL-backed
- **PostgreSQL Mode**: Kong menyimpan konfigurasi routing dan plugin di database PostgreSQL. Perubahan dilakukan secara dinamis melalui Admin API (`curl -X POST http://localhost:8001/services`).
- **DB-less Mode (Declarative YAML)**: Kong membaca seluruh konfigurasi dari satu file deklaratif (`kong.yaml`). Sangat ideal untuk prinsip **GitOps**, pipeline CI/CD, dan Kubernetes Ingress Controller karena tidak membutuhkan database eksternal dan proses boot berlangsung seketika.

---

## 3. Mengapa Sangat Penting di Skala Produksi

Pada skala ratusan microservices di perusahaan teknologi:

1. **Eliminasi Duplikasi Kode (DRY Architecture)**: Developer tim Order tidak perlu lagi memprogram verifikasi JWT, CORS headers, dan algoritma rate limiter. Semua logika tersebut dipindahkan ke Kong, sehingga kode aplikasi Go murni hanya fokus pada *business domain logic*.
2. **Keamanan Sentral & Blast-Radius Control**: Jika algoritma signing JWT diganti dari HS256 ke RS256, atau kunci publik diputar (*key rotation*), kalian cukup mengupdate konfigurasi plugin di Kong dalam 1 detik tanpa perlu me-redeploy 50 microservices di belakangnya.
3. **Performa Ekstrem Berbasis NGINX/OpenResty**: Kong mampu memproses ratusan ribu request per detik (*RPS*) dengan latensi tambahan di bawah 1 milidetik (*sub-millisecond gateway overhead*), jauh lebih efisien dibanding API Gateway berbasis JVM atau interpreted runtime.
4. **Dukungan Plugin Ekosistem Terluas**: Mendukung plugin bawaan (OAuth2, Key-Auth, Rate Limiting, Request Transformer, OpenTelemetry) serta plugin custom yang bisa ditulis dengan bahasa Lua, Go, Python, atau JavaScript.

---

## 4. Cara Kerja & Diagram Alur (ASCII)

Siklus hidup sebuah request yang melewati Kong Gateway menuju backend microservices:

```
[ Mobile / Web Client ]
          |
          | 1. HTTP Request (GET /api/v1/orders, Header: Authorization: Bearer <JWT>)
          v
+-----------------------------------------------------------------------------------+
| KONG GATEWAY (Listening on :8000)                                                 |
|                                                                                   |
|  [ Router Engine ]                                                                |
|    |-- Cocokkan Path `/api/v1/orders` -> Route: "orders-route"                   |
|    |-- Petakan ke Service: "order-service"                                        |
|    v                                                                              |
|  [ Plugin Execution Chain (Pre-Proxy Phase) ]                                     |
|    |-- 1. Plugin CORS (Cek origin header)                                         |
|    |-- 2. Plugin Rate Limiting (Cek Redis: 100 req/min limit)                     |
|    |-- 3. Plugin JWT (Validasi signature & expiry, extract claims)                |
|    |-- 4. Plugin Request Transformer (Inject header: `X-User-ID: 12345`)          |
|    v                                                                              |
|  [ Upstream Load Balancer (Round Robin) ]                                         |
|    |-- Pilih target sehat: `http://10.0.1.15:8080`                                |
+-----------------------------------------------------------------------------------+
          |
          | 2. Forwarded Request (Header: X-User-ID: 12345)
          v
+------------------------------------+
| Internal Microservice (Go Backend) |
|   - Zero JWT verification overhead |
|   - Hanya baca `X-User-ID` header  |
|   - Eksekusi query database        |
+------------------------------------+
          |
          | 3. HTTP 200 OK (Response Body JSON)
          v
+-----------------------------------------------------------------------------------+
| KONG GATEWAY (Post-Proxy Phase)                                                   |
|    |-- Plugin Response Transformer (Hapus internal headers)                       |
|    |-- Plugin Prometheus (Catat durasi request & status code)                     |
+-----------------------------------------------------------------------------------+
          |
          | 4. HTTP 200 OK (Clean Response)
          v
[ Mobile / Web Client ]
```

---

## 5. Contoh Implementasi Konkret: DB-less Declarative Setup

Berikut adalah konfigurasi produksi deklaratif modern untuk Kong Gateway dalam mode **DB-less** menggunakan file `kong.yaml` dan `docker-compose.yml`.

### A. Berkas Konfigurasi Deklaratif (`kong.yaml`)

Konfigurasi ini mendefinisikan upstream backend, rute `/api/v1/orders`, dan memasang plugin **Rate Limiting** serta **Key Authentication**:

```yaml
_format_version: "3.0"

# 1. Definisi Upstream Load Balancer
upstreams:
  - name: order_upstream
    algorithm: round-robin
    targets:
      - target: order-backend-1:8080
        weight: 100
      - target: order-backend-2:8080
        weight: 100

# 2. Definisi Services & Routes
services:
  - name: order-service
    url: http://order_upstream
    routes:
      - name: order-route
        paths:
          - /api/v1/orders
        strip_path: false
        methods:
          - GET
          - POST
    # 3. Pasang Plugin di level Service
    plugins:
      # Rate Limiting: Maksimal 100 request per menit per Consumer/IP
      - name: rate-limiting
        config:
          minute: 100
          policy: local # atau 'redis' untuk multi-node kong cluster
          fault_tolerant: true
          hide_client_headers: false

      # Key Authentication: Wajib menyertakan header 'apikey'
      - name: key-auth
        config:
          key_names:
            - apikey
          hide_credentials: true

# 4. Definisi Consumer (Klien Terdaftar)
consumers:
  - username: mobile-app-client
    keyauth_credentials:
      - key: sec_mobile_app_prod_99812
```

### B. Menjalankan Kong via `docker-compose.yml`

```yaml
version: '3.8'

services:
  kong:
    image: kong:3.5-alpine
    container_name: kong-gateway
    environment:
      KONG_DATABASE: "off"                      # Mengaktifkan mode DB-less
      KONG_DECLARATIVE_CONFIG: /etc/kong/kong.yaml
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001           # Admin API (Read-Only di mode DB-less)
      KONG_PROXY_LISTEN: 0.0.0.0:8000           # Gateway Proxy Publik
    volumes:
      - ./kong.yaml:/etc/kong/kong.yaml:ro
    ports:
      - "8000:8000" # Traffic Ingress publik
      - "8001:8001" # Admin Inspection lokal
    restart: unless-stopped
```

### C. Verifikasi dari Sisi Client / Backend Go

Ketika client melakukan request tanpa `apikey`:
```bash
curl -i http://localhost:8000/api/v1/orders
# Output: HTTP/1.1 401 Unauthorized
# {"message":"No API key found in request"}
```

Ketika client menyertakan `apikey` valid:
```bash
curl -i http://localhost:8000/api/v1/orders -H "apikey: sec_mobile_app_prod_99812"
# Output: HTTP/1.1 200 OK
# X-RateLimit-Limit-Minute: 100
# X-RateLimit-Remaining-Minute: 99
```

---

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Throughput Sangat Tinggi & Latensi Rendah**: Fondasi event-loop NGINX dan LuaJIT memberikan performa kelas dunia dengan penggunaan resource memori yang jauh lebih irit dibanding gateway berbasis Java/JVM.
- **Ekosistem Plugin yang Luas & Dinamis**: Memiliki puluhan plugin resmi siap pakai tanpa perlu restart gateway saat plugin diaktifkan.
- **Fleksibilitas Arsitektur (DB-less vs Hybrid vs DB)**: Mendukung mode *DB-less* untuk Kubernetes Ingress (KIC) dan mode *Hybrid* (Control Plane terpisah dari Data Plane tepi).

### Kekurangan
- **Debugging Lua yang Sulit**: Jika kalian menulis *custom plugin* yang kompleks dalam bahasa Lua, penanganan error dan debugging memory leak membutuhkan keahlian OpenResty khusus.
- **Keterbatasan Mode DB-less**: Pada mode DB-less, beberapa plugin yang membutuhkan mutasi state dinamis runtime (seperti pendaftaran OAuth2 credential melalui Admin API) tidak bisa digunakan tanpa men-deploy ulang file YAML konfigurasi.
- **Kompleksitas Scaling Multi-Node Rate Limiter**: Jika menggunakan rate limiter lokal pada 5 instance Kong, kuota request pengguna akan berlipat ganda 5x lipat kecuali jika dihubungkan ke Redis terpusat.

---

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- **Arsitektur Microservices Multi-Bahasa (Polyglot)**: Tim kalian menggunakan Go, Python, dan Node.js. Menyeragamkan standar autentikasi JWT, SSL/TLS termination, dan rate limiting di layer Kong jauh lebih efisien dibanding membuat library middleware di 3 bahasa berbeda.
- **Eksposur API Publik / Developer Portal B2B**: Membutuhkan sistem API Key management, kuota request berbayar (monetisasi API), dan throttling ketat untuk ribuan partner luar.
- **Kubernetes Ingress Controller**: Menggunakan Kong Ingress Controller (KIC) untuk mengarahkan trafik eksternal langsung ke Kubernetes Pods secara deklaratif via Kubernetes CRDs.

### Kapan Jangan Pakai
- **Aplikasi Monolitik Sederhana**: Jika sistem kalian hanya satu binary Go monolit dengan 1 instance database, menempatkan Kong di depannya hanya menambah *hop* jaringan dan kerumitan deployment yang tidak perlu (cukup gunakan Reverse Proxy NGINX biasa).
- **Kebutuhan Service Mesh Internal yang Kompleks**: Jika kebutuhan kalian adalah enkripsi mutual TLS (mTLS) *east-west* antar ratusan microservice internal, solusi Service Mesh seperti Istio atau Linkerd lebih tepat dibanding memasang API Gateway di setiap panggilan service-ke-service.

---

## 8. Metrik Kunci & Observabilitas

Kong menyediakan plugin bawaan `prometheus` yang mengekspos metrik standar untuk di-scrape di port `:8001/metrics`:

1. **Gateway Latency Overhead (`kong_latency_ms`)**: Waktu internal yang dihabiskan oleh Kong sendiri untuk menjalankan chain plugin dan routing (target produksi: p99 < 2ms).
2. **Upstream Latency (`kong_upstream_latency_ms`)**: Waktu yang dihabiskan oleh backend microservice internal kalian untuk merespons request.
3. **HTTP Status Code Distribution (`kong_http_requests_total`)**: Pantau rasio error `4xx` (misal `401 Unauthorized`, `429 Too Many Requests`) dan `5xx` (misal `502 Bad Gateway` saat upstream service down).
4. **Nginx Connection States (`kong_nginx_connections_active`)**: Jumlah koneksi TCP aktif yang sedang ditangani oleh worker process Kong.

---

## 9. Studi Kasus Nyata di Industri

### Skenario: Migrasi Fintech Gateway & Konsolidasi Keamanan

#### Problem
Sebuah startup fintech berkembang dari 3 monolit menjadi 45 microservices dalam kurun waktu 2 tahun. Setiap service memverifikasi JWT dan mengecek rate limiter dengan caranya sendiri. Akibatnya:
- Jika ada celah keamanan di library JWT Node.js, 15 service berbasis Node.js rentan diretas, sementara service Go aman.
- Partner perbankan eksternal sering membanjiri request ke endpoint transfer dana sehingga mematikan database core banking karena ketiadaan rate limiter global terpusat.

#### Solusi Arsitektur
Engineering team mengadopsi **Kong Gateway** dalam mode **Kubernetes Ingress**:
1. Menghapus seluruh middleware verifikasi JWT di 45 microservices.
2. Memasang plugin `jwt` dan `cors` di level ingress Kong. Kong memvalidasi signature kriptografis token di tepi (*edge*) dan menginjeksi header internal tepercaya `X-Consumer-Custom-ID: <user_id>` ke request yang diteruskan.
3. Memasang plugin `rate-limiting` berbasis cluster Redis terpusat untuk membatasi setiap partner maksimal 50 RPS.
4. Menerapkan mode deklaratif GitOps (`decK`) di mana setiap perubahan rute wajib melalui Pull Request di repository GitHub.

#### Hasil / Dampak
- **Zero Insecure Endpoints**: 100% trafik eksternal melewati verifikasi token terpadu; jika token kedaluwarsa atau cacat, request langsung dipotong di Kong tanpa pernah menyentuh CPU microservices internal.
- **Penurunan Latensi Rata-rata**: Menghilangkan overhead decoding JWT berulang di tiap microservice; latensi jaringan internal menjadi lebih terprediksi.
- **Pencegahan Outage Finansial**: Lonjakan trafik agresif dari partner luar otomatis tertahan dengan status `429 Too Many Requests` di layer gateway, menjaga database backend tetap stabil di bawah batas aman kapasitasnya.

---

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan DNS Caching & Upstream IP Staleness**: Kong (NGINX) secara default meng-cache DNS record upstream sangat lama. Di lingkungan dinamis seperti Kubernetes atau AWS ECS di mana IP Pod sering berganti saat auto-scaling, Kong bisa terus mengirim trafik ke IP pod lama yang sudah mati (*502 Bad Gateway*). Solusi: Set konfigurasi `dns_stale_ttl` dan pasang parameter resolver dengan TTL rendah (misal `dns_resolver = 10.96.0.10:53`, validitas TTL 5s).
2. **DB-less Mode Sangat Direkomendasikan untuk Kubernetes**: Menggunakan PostgreSQL sebagai database kong di Kubernetes sering menjadi *single point of failure*. Jika database PostgreSQL Kong mengalami restart, seluruh Admin API macet. Gunakan mode DB-less dengan *decK* atau CRD Ingress Controller agar Kong bekerja murni secara in-memory dan stateless.
3. **Bahaya Menggunakan Policy 'local' pada Rate Limiting Multi-Instance**: Jika kalian memiliki 4 replika container Kong Gateway dan memasang limit 100 req/menit dengan policy `local`, seorang penyerang sebenarnya bisa menembus hingga 400 req/menit (100 req per pod). Selalu gunakan policy `redis` jika kalian membutuhkan penegakan kuota yang presisi antar-replika.

---

## 11. Rangkuman Singkat

1. **Fungsi Utama**: Bertindak sebagai *single entry point* dan *reverse proxy* cerdas untuk memusatkan autentikasi, routing, rate limiting, dan observabilitas microservices.
2. **Arsitektur Berbasis NGINX & Lua**: Menawarkan throughput luar biasa tinggi dengan latensi gateway sub-milidetik, didukung ekosistem plugin dinamis.
3. **Abstraksi Kunci**: Terdiri dari **Service** (tujuan backend), **Route** (aturan pencocokan URL/Host), **Upstream** (load balancer), dan **Plugin** (fitur proteksi & transformasi).
4. **Pola Modern (DB-less & GitOps)**: Di lingkungan cloud-native, gunakan mode deklaratif tanpa database (`kong.yaml`) untuk menjamin deployment yang stateless, instan, dan mudah diotomatisasi.
