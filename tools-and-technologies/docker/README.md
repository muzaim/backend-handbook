# Docker & Kontainerisasi

## 1. Skenario Sistem Nyata

Dalam tim engineering backend lintas platform (developer menggunakan macOS ARM64, Linux Ubuntu, dan Windows 11), aplikasi backend membutuhkan lingkungan eksekusi yang identik hingga ke level dependensi sistem C. Aplikasi backend dibangun menggunakan Go/Node.js yang membutuhkan dependensi pustaka native C tingkat sistem (seperti `libvips` untuk pemrosesan gambar atau driver spesifik `libpq-dev`).

Jika tim mendistribusikan dan menjalankan aplikasi secara langsung di sistem operasi host masing-masing:
- Di laptop developer A aplikasi berjalan lancar (*"Di laptop saya normal kok!"*), tetapi saat dideploy ke server staging Ubuntu Linux, aplikasi seketika crash dengan error `libvips.so.42: cannot open shared object file`.
- Server production yang memiliki versi glibc atau runtime Python yang sedikit berbeda mengalami perilaku tak terduga (*runtime environment drift*).
- Setiap kali ada developer baru yang bergabung ke tim, dibutuhkan waktu 2 hari hanya untuk menginstall puluhan tools, variabel sistem, dan dependensi lokal yang sering kali berbenturan satu sama lain.

**Solusi dengan Docker & Kontainerisasi:**
Developer mendefinisikan satu berkas deklaratif `Dockerfile`. Kode aplikasi, runtime, dependensi sistem operasi tingkat rendah, dan konfigurasi dikemas menjadi satu paket terisolasi (*immutable*) yang disebut **Docker Image**. Image ini dijamin berjalan dengan perilaku 100% identik di laptop macOS developer, di server otomatisasi CI/CD, maupun di server kluster cloud production tanpa konflik dependensi.

## 2. Apa Itu dan Buat Apa?

Docker adalah platform kontainerisasi open-source yang memungkinkan developer membungkus aplikasi beserta seluruh dependensinya (kode, runtime, library, file konfigurasi, dan environment variables) ke dalam satu paket terisolasi yang disebut **Docker Image**.

Image tersebut kemudian dapat dijalankan di mana saja (laptop Mac developer, server Linux testing, atau cluster cloud) sebagai **Container** dengan jaminan perilaku 100% identik tanpa masalah *"It works on my machine"*.

### Perbedaan Utama: Kontainer vs Virtual Machine (VM)
- **Virtual Machine (VM)**: Menjalankan seluruh sistem operasi tamu (*Guest OS*) lengkap di atas perangkat keras virtual (*Hypervisor*). Berat, butuh gigabytes memori, dan membutuhkan waktu beberapa menit untuk booting.
- **Docker Container**: Mengisolasi proses menggunakan fitur kernel Linux (**Namespaces** untuk isolasi proses/jaringan dan **cgroups** untuk pembatasan CPU/RAM). Kontainer berbagi kernel yang sama dengan Host OS. Sangat ringan, berukuran megabytes, dan dapat menyala dalam hitungan **milidetik**.

## 3. Kenapa Harus Pakai Ini di High-Throughput?

### 1. Kecepatan Autoscaling Kilat
Saat trafik melonjak 10x lipat dalam waktu 1 menit (misal detik-detik peluncuran tiket konser):
- Menyalakan 50 mesin Virtual Machine baru butuh waktu 5 hingga 10 menit. Trafik sudah keburu tumbang sebelum server siap.
- Menyalakan 50 Docker Container baru hanya butuh waktu **2 hingga 5 detik**. Sistem mampu merespons lonjakan trafik secara elastis seketika.

### 2. Kerapatan Resource (*Resource Density*)
Karena kontainer tidak memboroskan CPU dan RAM untuk menjalankan puluhan kernel OS duplikat, satu server fisik yang tadinya hanya mampu menampung 4 Virtual Machine kini mampu menjalankan 40 Docker Container sekaligus. Ini memangkas biaya infrastruktur cloud hingga 60-70%.

## 4. Cara Kerjanya (Diagram Alur: VM vs Kontainer)

```
[ Virtual Machine Architecture ]          [ Docker Container Architecture ]

+-------------------------------+         +-------------------------------+
| App A | App B | App C         |         | App A | App B | App C         |
| (Deps)| (Deps)| (Deps)        |         | (Deps)| (Deps)| (Deps)        |
+-------+-------+-------+-------+         +-------+-------+-------+-------+
| Guest | Guest | Guest |       |         |         Docker Engine         |
| OS 1  | OS 2  | OS 3  |       |         |    (Namespaces & cgroups)     |
+-------+-------+-------+-------+         +-------------------------------+
|          Hypervisor           |         |       Host Operating System   |
+-------------------------------+         |         (Shared Linux Kernel) |
|         Host Hardware         |         +-------------------------------+
|      (Bare-Metal Server)      |         |         Host Hardware         |
+-------------------------------+         +-------------------------------+
```

## 5. Cuplikan Logika Inti

Pola penulisan `Dockerfile` standar produksi menggunakan **Multi-Stage Build** dan user non-root untuk memangkas ukuran image dan mengamankan runtime:

```dockerfile
# Stage 1: Build stage (menggunakan toolchain kompilasi lengkap)
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
# Kompilasi binary statis tanpa ketergantungan CGO
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o server ./cmd/api

# Stage 2: Production runtime stage (hanya image minimalis tanpa build tools)
FROM alpine:3.19

# Tambahkan sertifikat SSL dan user non-root demi keamanan
RUN apk --no-cache add ca-certificates \
    && addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /home/appuser
COPY --from=builder /app/server .

# Pindah eksekusi dari root ke user biasa
USER appuser

EXPOSE 8080

# Gunakan format exec (array JSON) agar sinyal SIGTERM diteruskan langsung ke proses
ENTRYPOINT ["./server"]
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Konsistensi Lingkungan Mutlak**: Mengeliminasi perbedaan versi Node.js/Go/Python antara komputer lokal developer dan server production.
- **Kecepatan Deployment & Rollback**: Proses build, push, dan pull image terstandardisasi via Container Registry (Docker Hub, AWS ECR, GitHub Packages).
- **Isolasi Kegagalan**: Jika Container A mengalami memory leak atau crash, Container B di server yang sama tidak akan terpengaruh (dibatasi oleh cgroups).

### Kekurangan
- **Keamanan Kernel Bersama (*Shared Kernel Risk*)**: Jika ada celah keamanan (*zero-day exploit*) pada kernel Linux host, penyerang yang membobol satu kontainer berpotensi melarikan diri (*container escape*) ke host utama.
- **Overhead Disk Storage Layer**: Pengelolaan image yang buruk dapat menimbun puluhan gigabyte layer Docker usang di disk server.
- **Bukan Pengganti VM Murni**: Kontainer Linux tidak bisa menjalankan aplikasi Windows secara native tanpa lapisan emulasi.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Standar utama seluruh aplikasi microservices dan web backend modern.
- Pipeline CI/CD untuk pengujian otomatis yang bersih dan reproducible.
- Infrastruktur cloud yang menggunakan Kubernetes, Nomad, atau AWS ECS.

### Kapan Jangan Pakai
- Beban kerja komputasi super-ekstrem (*High-Performance Computing - HPC*) yang membutuhkan akses langsung ke hardware tanpa overhead abstraksi kernel.
- Aplikasi desktop GUI lama yang dirancang khusus terikat erat dengan display driver OS tertentu.

## 8. Metrik Kunci & Observabilitas

Untuk memonitor kesehatan dan utilisasi container pada cAdvisor, Prometheus, atau Datadog:

- **`container_cpu_cfs_throttled_periods_total`**: Persentase periode CPU di mana container dipaksa throttling oleh cgroups karena melewati kuota batas (*CPU limits*).
- **`container_memory_working_set_bytes`**: Penggunaan RAM riil yang dihitung oleh kernel Linux untuk evaluasi OOM (Out-Of-Memory).
- **`container_oom_events_total`**: Jumlah kejadian di mana kernel membunuh container karena melebihi batas memori (`OOMKilled`).
- **`container_network_receive_bytes_total` & `transmit_bytes_total`**: Throughput bandwidth jaringan yang melewati virtual ethernet (veth) container.
- **`image_pull_duration_seconds`**: Waktu yang dibutuhkan worker node untuk mendownload image baru saat autoscaling.

## 9. Studi Kasus Nyata di Industri

- **Migrasi Infrastruktur Spotify ke Kontainer**: Spotify memindahkan ribuan microservices backend mereka dari pengelolaan server virtual VM statis ke kontainer Docker yang diorkestrasi di cloud. Langkah ini memangkas waktu booting service dari 15 menit menjadi kurang dari 10 detik dan menghemat jutaan dolar biaya sewa VM.
- **Uber Global Microservice Fleet**: Uber mengelola lebih dari 4.000 microservices terpisah yang seluruhnya dikemas dalam Docker containers. Standardisasi ini memungkinkan ribuan insinyur Uber mendeploy kode ke ribuan host bare-metal secara konsisten setiap hari.

## 10. Fakta Menarik & Gotchas Production

- **The Root User Security Hazard**: Secara default, jika kamu tidak menulis instruksi `USER` di Dockerfile, aplikasi di dalam kontainer akan berjalan sebagai **`root`**. Jika aplikasi webmu memiliki celah *Remote Code Execution*, penyerang mendapatkan akses root di server host! Selalu buat user non-root khusus (misal: `USER appuser`).
- **Bloated Image Size vs Multi-Stage Build**: Developer junior sering membuat Docker image berukuran 2 GB karena memasukkan compiler Go/Rust, source code, dan git history ke image final production. Terapkan **Multi-Stage Build**: gunakan container pertama untuk kompilasi kode, lalu salin hanya file biner murni hasil kompilasi ke base image minimal seperti `alpine` atau `distroless` (ukuran image menyusut drastis dari 2.000 MB menjadi 25 MB!).
- **Handling Sinyal SIGTERM**: Saat Docker menghentikan kontainer (`docker stop`), ia mengirim sinyal `SIGTERM` dan memberi waktu toleransi 10 detik sebelum mengirim `SIGKILL`. Jika aplikasimu tidak mendengarkan sinyal `SIGTERM`, servermu akan memutus koneksi transaksi database pengguna secara kasar (*abrupt drop*). Selalu tangani *Graceful Shutdown* di kodemu!
