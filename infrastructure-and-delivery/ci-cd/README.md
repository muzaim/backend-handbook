# CI/CD (Continuous Integration & Delivery)

## 1. Skenario Sistem Nyata

Dalam tim rekayasa perangkat lunak modern beranggotakan 15 developer yang bekerja bersama pada repositori backend e-commerce yang melayani jutaan transaksi per hari:

Jika tim tidak memiliki pipeline otomatisasi CI/CD:
- Setiap developer bekerja di branch masing-masing selama berminggu-minggu tanpa integrasi rutin. Saat tiba jadwal rilis bulanan ("Deployment Day"), penggabungan kode (*merge*) memicu ratusan *merge conflict* yang menyakitkan.
- Pengujian dilakukan secara manual oleh tim QA. Satu baris bug regresi pada fungsi kalkulasi diskon luput dari pengujian dan lolos ke branch utama.
- Proses rilis dilakukan secara manual: seorang engineer login via SSH ke server production pada tengah malam, menjalankan perintah `git pull`, mengompilasi biner, dan merestart server. Server mengalami *downtime* 15 menit, dan saat server menyala kembali, bug kalkulasi diskon menyebabkan kerugian finansial perusahaan.

**Solusi dengan CI/CD (misal: GitHub Actions & ArgoCD):**
- **Continuous Integration (CI)**: Setiap kali developer membuka Pull Request, server CI secara otomatis mengompilasi kode, menjalankan linter kode, memindai celah keamanan, dan mengeksekusi 400 *unit test* dan *integration test* dalam waktu 3 menit. Jika ada satu saja test yang gagal, kode dilarang masuk ke branch `main`.
- **Continuous Delivery / Deployment (CD)**: Begitu kode di-merge ke branch `main`, pipeline otomatis mengemas binary ke dalam Docker image bertag spesifik, dan mendeploy-nya ke kluster production menggunakan strategi *Canary Deployment* (5% trafik pengguna nyata dialihkan ke versi baru). Jika grafik error rate tetap 0%, trafik dinaikkan bertahap hingga 100% dengan *zero downtime*. Tim dapat mendeploy pembaruan kode puluhan kali setiap hari dengan tenang.

## 2. Apa Itu dan Buat Apa?

CI/CD adalah serangkaian praktik otomatisasi yang mempercepat proses pengiriman kode software dari laptop developer hingga berjalan aman di server production.

CI/CD terbagi menjadi dua fase utama:

### 1. Continuous Integration (CI)
Praktik di mana setiap anggota tim developer secara rutin menggabungkan (*merge*) perubahan kode mereka ke repositori pusat (misal branch `main`).
Setiap kali ada *Pull Request* atau *Commit*:
- Server CI otomatis menjalankan proses build kode.
- Menjalankan **Linter & Static Code Analysis** (mengecek standar gaya kode dan potensi bug).
- Menjalankan **Automated Unit Tests & Integration Tests**.
- Jika ada satu saja test yang gagal, kode ditolak masuk (*Build Failed*). Tujuannya: mendeteksi kesalahan sedini mungkin sebelum kode bercampur dengan kode developer lain.

### 2. Continuous Delivery & Continuous Deployment (CD)
- **Continuous Delivery**: Kode yang lolos tahap CI otomatis dibungkus menjadi artefak siap rilis (misal Docker Image) dan dideploy ke lingkungan testing/staging. Proses rilis ke production tinggal menunggu persetujuan satu klik (*manual 1-click approval*).
- **Continuous Deployment**: Tidak ada tombol persetujuan manual. Setiap perubahan kode yang lolos seluruh rangkaian pengujian otomatis akan **langsung meluncur ke server production** secara otomatis dalam hitungan menit.

Contoh software populer: GitHub Actions, GitLab CI, ArgoCD, Jenkins, CircleCI.

## 3. Strategi Deployment di High-Throughput

Di sistem skala besar yang melayani jutaan pengguna, kita tidak boleh mematikan server untuk mengupdate kode (*Zero-Downtime Deployment*).

Tiga strategi deployment paling penting:

### 1. Rolling Update
Server atau pod diperbarui satu per satu secara bergantian.
- *Kelebihan*: Hemat biaya hardware, tidak butuh server cadangan ganda.
- *Tantangan*: Dalam masa transisi, versi lama (v1) dan versi baru (v2) berjalan bersamaan di jaringan. Database harus kompatibel dengan kedua versi tersebut.

### 2. Blue-Green Deployment
Menyiapkan dua lingkungan server produksi identik: **Blue (versi aktif v1)** dan **Green (versi baru v2)**.
- Versi v2 dideploy penuh dan diuji di lingkungan Green.
- Router / Load Balancer mengalihkan switch 100% trafik dari Blue ke Green dalam 1 milidetik.
- *Kelebihan*: Rollback instan jika ada masalah (cukup kembalikan switch ke Blue).

### 3. Canary Deployment (Paling Populer di High-Throughput)
Merilis versi baru hanya ke **sebagian kecil pengguna nyata (misal 1% atau 5% trafik)** terlebih dahulu.
- Sistem monitoring mengawasi metrik secara real-time: Apakah tingkat error naik? Apakah latensi CPU memburuk?
- Jika metrik stabil selama 15 menit, trafik dinaikkan bertahap: 5% -> 25% -> 50% -> 100%.
- Jika metrik menunjukkan anomali, sistem otomatis melakukan rollback seketika. Hanya 1% pengguna yang sempat merasakan dampak bug, sementara 99% pengguna lainnya aman.

## 4. Cara Kerjanya (Diagram Alur Pipeline Otomatis)

```
[ Developer ]
      |
      | 1. git push origin feature-payment
      v
+-------------------------------------------------------+
|                 CI PIPELINE (GitHub Actions)          |
|                                                       |
|  [ Lint & Format ] -> [ Unit Tests ] -> [ Sec Scan ]  |
+-------------------------------------------------------+
      |
      | 2. Lolos Semua Uji Coba!
      v
+-------------------------------------------------------+
|             CONTAINER BUILD & PACKAGING               |
|                                                       |
|  Build Docker Image -> Push ke Container Registry     |
+-------------------------------------------------------+
      |
      | 3. Trigger CD (ArgoCD / GitOps)
      v
+-------------------------------------------------------+
|             CD PIPELINE: CANARY DEPLOYMENT            |
|                                                       |
|  Trafik ke Versi Baru: 5% -----(Pantau Error Rate)----+
|                                |                      |
|                       [ Metrik Sehat? ]               |
|                                |                      |
|                  +-------------+-------------+        |
|                  |                           |        |
|               [ YA ]                      [ TIDAK ]   |
|                  |                           |        |
|                  v                           v        |
|         Naikkan Trafik 100%           Rollback Instan |
+-------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

Contoh pipeline otomatis GitHub Actions untuk CI dan manifest Argo Rollouts untuk Canary Deployment:

```yaml
# .github/workflows/ci.yml
name: Backend CI Pipeline

on:
  pull_request:
    branches: [ main ]

jobs:
  test-and-verify:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Setup Go Environment
      uses: actions/setup-go@v5
      with:
        go-version: '1.22'

    - name: Run Linters & Security Scan
      run: |
        go vet ./...
        go run honnef.co/go/tools/cmd/staticcheck@latest ./...

    - name: Run Unit & Race Condition Tests
      run: go test -v -race -coverprofile=coverage.out ./...

    - name: Build & Push Docker Image
      if: github.ref == 'refs/heads/main'
      run: |
        docker build -t registry.internal/api:${{ github.sha }} .
        docker push registry.internal/api:${{ github.sha }}
```

Dan konfigurasi Canary Rollout otomatis dengan Argo Rollouts:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-api
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 5         # 1. Alihkan 5% trafik ke versi baru
      - pause: {duration: 10m} # 2. Pantau metrik selama 10 menit
      - setWeight: 20        # 3. Jika sehat, naikkan ke 20%
      - pause: {duration: 15m}
      - setWeight: 100       # 4. Alihkan penuh 100%
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Kecepatan Rilis Tinggi**: Tim dapat merilis fitur atau perbaikan bug puluhan kali dalam sehari (*High Deployment Velocity*).
- **Stabilitas Terukur**: Bug tertangkap di tahap otomatisasi sebelum sempat dilihat oleh pengguna akhir.
- **Bebas Stres Saat Rilis**: Tidak ada lagi ritual begadang di akhir pekan untuk deployment massal yang menegangkan.

### Kekurangan
- **Biaya Pemeliharaan Pipeline**: Menulis dan menjaga skrip pipeline CI/CD yang kompleks membutuhkan waktu dan keahlian platform engineering tersendiri.
- **Biaya Infrastruktur Tambahan**: Menjalankan runner testing dan lingkungan ganda (Blue/Green) membutuhkan biaya server cloud ekstra.
- **Bahaya Uji Coba Palsu (*Flaky Tests*)**: Tes yang kadang sukses kadang gagal tanpa sebab pasti akan membuat developer kehilangan kepercayaan pada pipeline.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Seluruh proyek perangkat lunak modern yang melibatkan lebih dari satu developer.
- Setiap sistem produksi yang tidak mengizinkan adanya downtime saat pembaruan kode.
- Tim yang ingin menerapkan filosofi *Agile* dan *Continuous Improvement*.

### Kapan Jangan Pakai
- Skrip throwaway satu kali pakai yang tidak akan pernah di-deploy ke server mana pun.

## 8. Metrik Kunci & Observabilitas (DORA Metrics)

Empat metrik industri standar DevOps Research and Assessment (DORA) untuk mengukur performa CI/CD:

- **Deployment Frequency**: Seberapa sering kode sukses dideploy ke production (tim elit: beberapa kali sehari).
- **Lead Time for Changes**: Waktu yang dibutuhkan sejak commit pertama dibuat hingga kode tersebut aktif berjalan di production (tim elit: kurang dari 1 jam).
- **Change Failure Rate**: Persentase deployment yang menyebabkan insiden, degradasi service, atau membutuhkan rollback darurat (target: < 5%).
- **Time to Restore Service (MTTR)**: Waktu yang dibutuhkan untuk memulihkan service saat terjadi kegagalan di production (target: < 15 menit melalui rollback otomatis).
- **CI Build Duration**: Waktu yang dihabiskan developer menunggu feedback pengujian otomatis per commit (target: < 10 menit).

## 9. Studi Kasus Nyata di Industri

### Otomatisasi Progressive Canary Rollout dan Deteksi Anomali Metrik Real-Time

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah portal fintech dengan 50 juta pengguna aktif awalnya melakukan deployment mingguan secara manual di tengah malam dengan metode "all-at-once". Suatu ketika, rilis baru membawa kebocoran memori (memory leak) tersembunyi yang baru terpicu setelah 10 menit melayani trafik produksi. Seluruh 200 pod crash bersamaan karena OOM (Out Of Memory), mengakibatkan downtime total selama 45 menit dan kerugian finansial reputasional yang masif.
- **Implementasi Solusi Industri (The Architectural Solution)**: Tim merancang pipeline CI/CD modern berbasis *GitOps* (ArgoCD + Argo Rollouts). Deployment baru diuji melalui pipeline CI (linting, test unit/integrasi, dan image vulnerability scanning). Di level delivery produksi, diterapkan strategi *Canary Release*: versi baru awalnya hanya menerima 5% trafik publik selama 15 menit. Argo Rollouts menganalisis metrik Prometheus (HTTP error rate 5xx, latency p99, dan error log spikes). Jika error rate melonjak > 0,5%, pipeline secara otonom membatalkan rollout (*automated instant rollback*) dalam 3 detik tanpa intervensi manusia.
- **Dampak & Hasil Sistem (The Outcome)**: Change Failure Rate anjlok dari 14% menjadi kurang dari 0,8%. MTTR (Mean Time to Recover) terpangkas dari 45 menit menjadi < 15 detik berkat rollback otomatis. Tim bertransformasi dari deployment mingguan yang menegangkan menjadi 15-20 deployment mandiri per hari kerja dengan *zero user-facing downtime*.

## 10. Fakta Menarik & Gotchas Production

- **The Database Migration Trap in Deployments**: Kesalahan paling fatal developer saat CI/CD adalah mengubah struktur kolom database secara *breaking change* (misal: langsung me-rename kolom `phone_number` menjadi `mobile_phone`). Saat rolling update atau canary sedang berjalan, pod versi lama (v1) masih aktif dan akan langsung crash karena kolom `phone_number` mendadak hilang dari database!  
  *Solusi Wajib: Expand and Contract Pattern*:
  1. Tahap 1: Tambah kolom baru `mobile_phone` tanpa menghapus kolom lama.
  2. Tahap 2: Deploy kode aplikasi baru yang menulis ke kedua kolom sekaligus.
  3. Tahap 3: Salin data historis dari kolom lama ke kolom baru.
  4. Tahap 4: Hapus kolom lama setelah seluruh pod 100% menggunakan kode baru.
- **Flaky Tests Are Tech Debt Cancer**: Jika ada satu unit test yang gagal 1 dari 10 kali karena timing race condition yang tidak rapi, developer akan mulai membiasakan diri mengklik tombol *"Re-run failed jobs"* tanpa memeriksa kode. Begitu kebiasaan ini terbentuk, satu bug berbahaya yang nyata akan lolos ke production karena dianggap hanya "flaky test biasa".

## 11. Rangkuman Singkat

- **Otomatisasi Penuh dari Commit ke Deploy**: CI memvalidasi kebenaran kode (build, lint, test) secara otomatis di setiap PR; CD mengantarkan kode yang terverifikasi ke production secara aman dan dapat diulang.
- **Gunakan Strategi Rilis Progresif**: Hindari rilis serentak (*all-at-once*); terapkan *Canary* atau *Blue/Green Deployment* untuk membatasi *blast radius* jika versi baru memiliki bug tersembunyi.
- **Automated Rollback Berbasis Metrik**: Hubungkan pipeline CD dengan metrik observabilitas (SLO/error rate); batalkan deployment secara otomatis jika ambang batas error terlampaui.
- **Pola Expand and Contract untuk Database**: Jangan pernah merilis migrasi skema database yang memutus kompatibilitas mundur (*breaking changes*); pisahkan penambahan kolom dan penghapusan kolom dalam siklus rilis bertahap.
