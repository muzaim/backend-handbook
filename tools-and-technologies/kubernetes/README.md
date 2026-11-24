# Kubernetes (K8s)

## 1. Skenario Sistem Nyata

Pada sistem e-commerce berskala nasional yang berjalan di atas 80 container Docker backend yang tersebar di 15 server cloud:

Tepat jam 3 dini hari, Server Worker #4 di cloud data center mengalami kerusakan motherboard fisik mendadak atau *kernel panic*. Akibatnya, 12 container pod backend pembayaran yang berjalan di server tersebut mati seketika.

Jika kamu mengelola kontainer secara manual (misal hanya menggunakan `docker run` biasa di setiap mesin):
- Request checkout dari pengguna yang diarahkan ke IP Server #4 seketika mengalami error koneksi `HTTP 502 Bad Gateway` dan transaksi gagal.
- Engineer on-call harus terbangun dari tidur, membuka laptop di tengah malam, menyewa server baru, mengetik perintah SSH, dan mengkonfigurasi ulang load balancer secara manual.
- Siangnya saat ada event flash sale, trafik melonjak 8x lipat. Tanpa sistem otomatis, server kehabisan kapasitas memori dan tumbang.

**Solusi dengan Kubernetes (K8s):**
Kamu mendeklarasikan target kondisi sistem: *"Saya ingin selalu ada 20 pod service pembayaran yang aktif"*. Saat Server #4 mati, Control Plane Kubernetes mendeteksi hilangnya node dalam hitungan detik, secara otomatis menjadwalkan ulang (*rescheduling*) 12 pod yang hilang ke server lain yang sehat (**Self-Healing**), dan memperbarui tabel routing internal secara otomatis tanpa downtime. Begitu event flash sale dimulai, Horizontal Pod Autoscaler (HPA) otomatis menduplikasi pod dari 20 menjadi 80 pod sesuai beban CPU secara elastis.

## 2. Apa Itu dan Buat Apa?

Kubernetes (sering disingkat **K8s**) adalah platform orkestrasi kontainer open-source yang bertugas mengotomatiskan deployment, penskalaan (*scaling*), dan pengelolaan aplikasi yang berjalan di dalam kontainer di seluruh kluster server fisik maupun virtual.

K8s bekerja dengan filosofi **Declarative Desired State**: Kita tidak memerintahkan K8s *"nyalakan pod ini sekarang"*, melainkan kita menyatakan *"Saya ingin selalu ada 5 pod service order yang hidup setiap saat"*. Control plane K8s akan terus-menerus memantau dan mencocokkan kondisi riil di lapangan agar sesuai dengan deklarasi kita.

### Komponen Kunci di Kubernetes:
- **Pod**: Unit komputasi terkecil di K8s (biasanya membungkus satu container aplikasi beserta storage dan network IP lokalnya).
- **Deployment & ReplicaSet**: Pengendali yang menjamin jumlah replika pod selalu sesuai yang dideklarasikan dan menangani pembaruan versi aplikasi secara bertahap (*Rolling Update*).
- **Service**: Abstraksi jaringan stabil yang memberikan IP virtual permanen dan DNS internal untuk mengarahkan trafik ke kumpulan pod (yang IP fisiknya selalu berubah-ubah saat restart).
- **Ingress**: Pintu gerbang layer 7 (HTTP/HTTPS) yang merutekan trafik dari internet luar ke Service internal di dalam cluster.
- **HPA (Horizontal Pod Autoscaler)**: Komponen yang secara otomatis menambah jumlah pod dari 3 menjadi 30 saat pemakaian CPU atau antrean trafik melonjak tinggi.

## 3. Kenapa Harus Pakai Ini di High-Throughput?

Mengelola 5 container Docker di 1 server VPS itu mudah dengan `docker-compose`. Namun, bagaimana jika sistemmu membutuhkan **500 container yang tersebar di 50 server cloud** untuk melayani 500.000 pengguna bersamaan?
- Apa yang terjadi jika Server #12 terbakar atau listrik data center mati jam 3 pagi?
- Bagaimana cara memperbarui versi aplikasi tanpa memutus koneksi pengguna yang sedang aktif bertransaksi?

Tanpa Kubernetes: Engineer harus begadang semalaman memindahkan kontainer secara manual, mengupdate IP di reverse proxy satu per satu, dan merestart server.

Dengan Kubernetes: K8s mendeteksi server yang mati dalam hitungan detik, secara otomatis menjadwalkan ulang (*rescheduling*) pod-pod tersebut ke server lain yang masih sehat, dan memperbarui rute jaringan tanpa ada satu pun request pengguna yang gagal (*Self-Healing 100% Otomatis*).

## 4. Cara Kerjanya (Diagram Alur Arsitektur Cluster)

```
                            [ KUBERNETES CLUSTER ]

+-------------------------------------------------------------------------+
|                              CONTROL PLANE                              |
|                                                                         |
|  +--------------------+  +--------------------+  +-------------------+  |
|  |   API Server       |  |     Scheduler      |  | Controller Manager|  |
|  +--------------------+  +--------------------+  +-------------------+  |
|  |           etcd (Database Konsistensi State Cluster)               |  |
+-------------------------------------------------------------------------+
                                    |
          Mengarahkan dan Memantau  |  Heartbeat & Telemetri
                                    v
+------------------------------------+  +---------------------------------+
|      WORKER NODE 1 (Server A)      |  |     WORKER NODE 2 (Server B)    |
|                                    |  |                                 |
|  [ Kubelet ]       [ Kube-Proxy ]  |  |  [ Kubelet ]      [ Kube-Proxy ]|
|                                    |  |                                 |
|  +--------------+ +--------------+ |  |  +--------------+ +------------+|
|  |    Pod 1     | |    Pod 2     | |  |  |    Pod 3     | |   Pod 4    ||
|  | [Container]  | | [Container]  | |  |  | [Container]  | |[Container] ||
|  +--------------+ +--------------+ |  |  +--------------+ +------------+|
+------------------------------------+  +---------------------------------+
```

## 5. Cuplikan Logika Inti

Manifest deklaratif `Deployment` standar production dengan strategi rolling update, resource constraints, dan health probes yang aman:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Pod baru dibuat maksimal 25% di atas kuota
      maxUnavailable: 0     # Nol pod lama dimatikan sebelum pod baru 100% sehat
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: api
        image: registry.internal/order-service:v2.4.1
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "250m"       # 0.25 core CPU dijamin tersedia
            memory: "512Mi"   # 512MB RAM dijamin tersedia
          limits:
            cpu: "1000m"      # Plafon maksimal 1 core CPU
            memory: "1Gi"     # Plafon 1GB RAM (lewat dari ini kena OOMKilled)
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Self-Healing & High Availability**: Pod yang crash otomatis di-restart; node yang mati otomatis ditinggalkan.
- **Zero-Downtime Rolling Update & Rollback**: Update aplikasi dilakukan pod-demi-pod; jika versi baru error, rollback instan dalam 1 perintah.
- **Abstraksi Cloud-Agnostic**: Konfigurasi YAML K8s berjalan identik di AWS (EKS), Google Cloud (GKE), Azure (AKS), maupun On-Premise bare-metal.

### Kekurangan
- **Kurva Belajar yang Sangat Curam**: Salah satu teknologi dengan tingkat kerumitan arsitektur dan debugging tertinggi di dunia software engineering.
- **Overhead Resource Control Plane**: Kluster K8s membutuhkan server khusus hanya untuk menjalankan etcd dan komponen control plane.
- **Kutukan Konfigurasi YAML**: Puluhan file manifest YAML yang panjang dan rawan salah ketik tanpa tooling seperti Helm atau Kustomize.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Sistem backend berskala menengah hingga besar dengan arsitektur microservices terdistribusi.
- Kebutuhan autoscaling dinamis berbasis lonjakan trafik real-time.
- Tim engineering yang memiliki dedicated DevOps / Platform Engineer untuk mengelola cluster.

### Kapan Jangan Pakai (Over-Engineering!)
- Aplikasi monolith sederhana atau situs portofolio dengan trafik stabil (cukup gunakan VPS tunggal, Docker Compose, atau PaaS seperti Render / Fly.io / Heroku).
- Tim startup tahap awal beranggotakan 2-3 developer; mengurus K8s akan menghabiskan 80% waktumu daripada membangun fitur produk bisnis.

## 8. Metrik Kunci & Observabilitas

Untuk memantau stabilitas cluster Kubernetes melalui Prometheus dan Grafana:

- **`kube_pod_container_status_restarts_total`**: Lonjakan jumlah restart pod mengindikasikan adanya uncaught panic atau fatal exception di kode aplikasi.
- **`kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}`**: Pod yang gagal startup berulang kali.
- **`kube_node_status_condition{condition="MemoryPressure|DiskPressure"}`**: Peringatan bahwa server worker node kehabisan memori atau ruang disk.
- **`kube_pod_status_phase{phase="Pending"}`**: Jumlah pod yang antre menunggu dijadwalkan; lonjakan ini berarti kapasitas CPU/RAM cluster habis dan butuh node baru.
- **`hpa_current_cpu_utilization_percentage`**: Utilisasi CPU pod relatif terhadap target scaling HPA.

## 9. Studi Kasus Nyata di Industri

- **Migrasi Airbnb 100% ke Kubernetes**: Airbnb memigrasikan ratusan service monolitik mereka ke lebih dari 250 cluster Kubernetes yang mengelola puluhan ribu kontainer. Standarisasi K8s memungkinkan tim developer Airbnb mendeploy kode ke production lebih dari 500 kali per hari secara independen.
- **Infrastruktur Model AI OpenAI**: OpenAI menjalankan kluster Kubernetes berskala raksasa (ribuan GPU node) untuk melatih dan melayani model bahasa besar (seperti GPT-4). Kubernetes mengorkestrasi penjadwalan komputasi parallel berskala tinggi secara elastis.

## 10. Fakta Menarik & Gotchas Production

- **The OOMKilled Disaster (Requests vs Limits)**: Setiap pod di K8s harus didefinisikan `resources.requests` (garansi minimum) dan `resources.limits` (batas plafon maksimal). Jika aplikasimu mengalami lonjakan memori sedikit saja melebihi batas `limits`, Linux kernel K8s akan langsung membunuh prosesmu seketika dengan status **`OOMKilled (Exit Code 137)`**. Selalu pantau grafik memory consumption sebelum menentukan limit!
- **Liveness vs Readiness Probe**: Ini adalah kesalahan paling sering di production:
  - **Readiness Probe**: Memeriksa apakah pod sudah siap menerima trafik (misal: koneksi DB sudah siap). Jika gagal, K8s hanya mencabut pod dari Load Balancer tanpa membunuhnya.
  - **Liveness Probe**: Memeriksa apakah aplikasi macet/deadlock. Jika gagal, K8s akan **MERESTART PAKSA** pod tersebut.  
  *Gotchas:* Jika liveness probe kamu arahkan ke dependency eksternal (misal database), saat database overload dan timeout, K8s akan membunuh dan merestart seluruh 50 pod aplikasi secara bersamaan (*CrashLoopBackOff massal*)! Liveness probe harus murni mengecek kesehatan internal container itu sendiri.
