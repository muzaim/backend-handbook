# Jenkins (Automation & CI/CD Server)

## 1. Skenario Sistem Nyata

Pada sebuah perusahaan perbankan digital dengan 40 developer backend yang aktif melakukan puluhan _Pull Request_ (PR) setiap hari ke repositori monorepo microservices:

Sebelum kode baru dapat dirilis ke cluster Kubernetes produksi:

-   Kode Go harus dikompilasi, dicek standar kerapiannya (_linter_ via `golangci-lint`), dijalankan pengujian unit serta tes konkurensi (`go test -race`), dibungkus menjadi container image Docker, lalu diverifikasi celah keamanannya (_vulnerability scan_).
-   Jika proses ini dijalankan secara manual di laptop masing-masing developer: satu siklus pengujian memakan waktu 25 menit, kredensial rilis produksi harus dibagikan ke banyak orang (sangat berbahaya bagi kepatuhan audit finansial), dan sering muncul masalah klasik _"it works on my machine"_.

**Solusi dengan Jenkins CI/CD Automation:**
Perusahaan mengoperasikan server Jenkins mandiri (_self-hosted_) dengan arsitektur _Controller-Agent_. Setiap kali developer menekan tombol _Create Pull Request_ di Git, webhook secara instan memicu pipeline deklaratif di berkas `Jenkinsfile`. Jenkins Controller secara dinamis meluncurkan _ephemeral worker pod_ di Kubernetes untuk mengeksekusi kompilasi Go, menjalankan 500 unit test secara paralel dalam 90 detik, dan memblokir merge jika ada tes yang gagal. Ketika kode di-merge ke `main`, Jenkins secara otomatis mem-publish Docker image dan men-deploy-nya ke staging environment.

## 2. Apa Itu dan Buat Apa?

Jenkins adalah server otomasi sumber terbuka (_open-source automation server_) yang digunakan untuk mengotomatisasi seluruh rantai pengiriman perangkat lunak (_Continuous Integration & Continuous Delivery / CI-CD_).

Karakteristik utama:

-   **Pipeline as Code (`Jenkinsfile`)**: Seluruh alur kerja build, pengujian, dan deployment ditulis dalam bentuk kode deklaratif yang tersimpan di dalam repositori Git bersama kode aplikasi.
-   **Arsitektur Terdistribusi (Controller - Agent)**: Controller (Master) bertugas mengelola jadwal antrean dan antarmuka web, sedangkan eksekusi proses build yang berat dialihkan ke puluhan mesin _Agent_ (Worker) yang dapat bertambah secara dinamis di Kubernetes.
-   **Ekosistem Plugin Raksasa**: Memiliki ribuan plugin untuk berintegrasi dengan hampir semua teknologi (Docker, Kubernetes, Git, SonarQube, Slack, AWS).

## 3. Kenapa Harus Pakai Ini ?

Bagi tim backend berskala besar dengan ratusan microservices, Jenkins memberikan keunggulan infrastruktur yang masif:

1. **Kendali Penuh & Kepatuhan Keamanan (On-Premises / Private VPC)**: Banyak institusi finansial melarang kode sumber dan kredensial database mereka melintasi cloud publik pihak ketiga. Jenkins dapat di-hosting 100% di dalam jaringan privat perusahaan.
2. **Kapasitas Eksekusi Tanpa Batas**: Tidak ada batasan kuota menit build bulanan seperti pada layanan SaaS berbayar. Tim bebas menjalankan ratusan build paralel dengan memanfaatkan kapasitas komputasi internal.
3. **Orkestrasi Alur Kerja Kompleks**: Mampu menjalankan _matrix build_, pengujian terdistribusi, dan alur rilis bertahap (_multi-stage canary deployment_) dengan persetujuan manual (_manual approval gate_).

## 4. Cara Kerjanya (Diagram Alur Pipeline Jenkins)

```
+------------------+         +------------------+
|  Developer Git   | ------> |  Git Repository  |
|  (Push / PR)     |         | (GitHub/GitLab)  |
+------------------+         +------------------+
                                      |
                               Webhook Trigger
                                      v
                             +------------------+
                             | Jenkins Server   |
                             |  (Controller)    |
                             +------------------+
                                      |
                            Spawn Dynamic Worker
                                      v
                             +------------------+
                             | Kubernetes Pod   |
                             | (Ephemeral Agent)|
                             +------------------+
                                      |
              +-----------------------+-----------------------+
              |                       |                       |
              v                       v                       v
     [Stage 1: Checkout]     [Stage 2: Go Test]      [Stage 3: Docker Build]
     - Clone repositori      - golangci-lint         - Build image
     - Cache dependencies    - go test -race ./...   - Push ke Container
                                                       Registry
                                                              |
                                                              v
                                                     [Stage 4: Deploy K8s]
                                                     - Helm upgrade
                                                     - Kirim notif Slack
```

## 5. Cuplikan Konfigurasi Inti (Declarative `Jenkinsfile` untuk Backend Go)

Berikut contoh `Jenkinsfile` deklaratif standar industri untuk aplikasi backend Go yang berjalan di dalam agent kontainer Docker terisolasi:

```groovy
pipeline {
    agent {
        docker {
            // Gunakan base image resmi Go agar lingkungan build selalu konsisten
            image 'golang:1.22-alpine'
            args '-v /tmp/go-cache:/go/pkg/mod' // Mount cache dependensi lokal
        }
    }

    environment {
        APP_NAME    = 'order-service'
        REGISTRY    = 'registry.company.internal/backend'
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
        CGO_ENABLED = '0' // Static binary compilation
    }

    stages {
        stage('Lint & Static Analysis') {
            steps {
                echo "--> Menjalankan linter kode Go..."
                sh 'go vet ./...'
            }
        }

        stage('Run Unit & Race Tests') {
            steps {
                echo "--> Menjalankan unit test dengan deteksi data race..."
                sh 'go test -v -race -coverprofile=coverage.out ./...'
            }
        }

        stage('Compile Binary') {
            steps {
                echo "--> Mengompilasi executable binary Go..."
                sh 'go build -ldflags="-s -w" -o bin/app cmd/server/main.go'
            }
        }

        stage('Build & Push Container') {
            steps {
                script {
                    echo "--> Membangun Docker Image: ${REGISTRY}/${APP_NAME}:${IMAGE_TAG}"
                    // Menggunakan kredensial aman yang tersimpan di Jenkins Credentials Store
                    withCredentials([usernamePassword(credentialsId: 'docker-registry-cred', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                        sh """
                            docker login -u ${REG_USER} -p ${REG_PASS} ${REGISTRY}
                            docker build -t ${REGISTRY}/${APP_NAME}:${IMAGE_TAG} .
                            docker push ${REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs() // Membersihkan workspace agar disk server tidak penuh
        }
        failure {
            echo "Pipeline GAGAL! Mengirimkan alert darurat ke kanal engineering..."
        }
        success {
            echo "Pipeline BERHASIL! Artefak siap dideploy ke cluster produksi."
        }
    }
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **100% Open-Source & Bebas Biaya Lisensi**: Bebas dijalankan di server bare-metal atau cloud mandiri tanpa biaya langganan bulanan.
-   **Kustomisasi Ekstrem**: Fleksibilitas Groovy script memungkinkan orkestrasi skenario deployment yang sangat rumit.
-   **Kemandirian Data**: Seluruh riwayat build, log pengujian, dan artefak tersimpan di infrastruktur privat perusahaan.

### Kekurangan

-   **Beban Perawatan Infrastruktur (_Maintenance Overhead_)**: Tim DevOps harus memelihara server OS, runtime Java JVM, backup database Jenkins, dan update plugin secara berkala.
-   **Kerentanan Keamanan Plugin (_Plugin Sprawl_)**: Terlalu banyak memasang plugin pihak ketiga yang tidak terawat dapat membuka celah keamanan serius (_CVE vulnerabilities_).
-   **Antarmuka Klasik**: UI web bawaan terasa lebih lawas dibandingkan solusi modern seperti GitLab CI atau GitHub Actions.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Perusahaan enterprise atau institusi finansial yang memiliki regulasi ketat mengenai kepemilikan data dan hosting server privat (_on-premises_).
-   Proyek dengan frekuensi build sangat masif (ribuan build per hari) di mana biaya layanan SaaS berbasis menit akan membengkak drastis.
-   Sistem yang membutuhkan integrasi dengan sistem legacy atau hardware khusus di data center lokal.

### Kapan Jangan Pakai

-   Tim startup kecil yang belum memiliki dedicated DevOps engineer (lebih bijak menggunakan GitHub Actions atau GitLab CI).
-   Proyek open-source publik sederhana yang tidak membutuhkan infrastruktur build privat.
-   Arsitektur yang murni mengadopsi GitOps cloud-native (misal kombinasi GitHub Actions + ArgoCD).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`jenkins_executor_busy_count` vs `total` (Gauge)**: Rasio pemakaian slot worker agent. Jika menyentuh 100%, developer akan mulai mengantre lama (_queue latency_).
-   **`jenkins_job_duration_seconds` (Histogram)**: Durasi total eksekusi pipeline per service. Pipeline yang mendadak lambat mengindikasikan cache dependensi rusak.
-   **`jenkins_failed_jobs_total` (Counter)**: Jumlah kegagalan build harian untuk mengukur stabilitas cabang `main`.
-   **`jenkins_disk_space_free_bytes` (Gauge)**: Sisa kapasitas hard disk pada direktori `$JENKINS_HOME`.

## 9. Studi Kasus Industri Nyata

### Orkestrasi Pipeline CI/CD Ephemeral Kubernetes Agent pada Institusi Finansial Air-Gapped

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah institusi perbankan dengan regulasi kedaulatan data ketat dilarang menggunakan platform CI/CD SaaS publik. Tim awalnya mengoperasikan server Jenkins mandiri berbasis VM statis dengan 10 worker agent tetap. Ketika 200 software engineer melakukan merge PR serentak di sore hari, antrean pipeline build melonjak hingga menunggu 45 menit (*queue backlog*). Selain itu, artefak build yang tersisa di VM worker saling menimpa (*workspace pollution*), menyebabkan tes regresi gagal palsu (*flaky false alarms*), dan server Jenkins Controller sering crash kehabisan memori (OOM) karena beberapa developer menjalankan build langsung di master node.
- **Implementasi Solusi Industri (The Architectural Solution)**: Tim merestrukturisasi infrastruktur Jenkins ke arsitektur *Dynamic Kubernetes Agents*. Jumlah executor di Jenkins Controller disetel ke 0 untuk mencegah eksekusi di master node. Setiap pemicuan pipeline (`Jenkinsfile`) secara dinamis mem-provision pod agent baru di cluster Kubernetes privat menggunakan Kubernetes Plugin. Pod agent dilengkapi kontainer terisolasi (Kaniko untuk Docker build tanpa daemon root, linter, dan Go runner) yang otomatis musnah (*self-destruct*) begitu tahap build selesai. Diterapkan cache volume terdistribusi (NFS/MinIO) untuk mempercepat caching dependensi tanpa meninggalkan jejak workspace lokal.
- **Dampak & Hasil Sistem (The Outcome)**: Waktu tunggu antrean build developer terpangkas dari 45 menit menjadi 0 detik karena kapasitas agen berskala elastis mengikuti jumlah job. Kasus kegagalan tes akibat kontaminasi workspace turun menjadi 0%. Stabilitas Jenkins Controller mencapai 99,99% uptime tanpa insiden OOM crash, sementara biaya komputasi server worker berkurang 40% karena pod agent hanya menyala saat ada kompilasi aktif.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Menjalankan Build Langsung di Controller (Master Node)**: Menjalankan eksekusi kompilasi Go atau Docker build berat langsung di node Controller. Jika build kehabisan memori (_OOM_), Controller Jenkins akan crash dan seluruh antrean pipeline perusahaan mati serentak!  
   _Solusi_: Nonaktifkan slot executor pada Controller (set ke 0), wajibkan semua build berjalan di ephemeral Agent/pod.
2. **Kepanikan Hard Disk Penuh (_Disk Space Exhaustion_)**: Jenkins secara default menyimpan riwayat artefak binary dan log setiap build. Dalam beberapa bulan, hard disk server bisa 100% penuh.  
   _Solusi_: Selalu aktifkan opsi `Discard old builds` (simpan maksimal 15 build terakhir) dan panggil fungsi `cleanWs()` di blok post pipeline.
3. **Terjebak Versi Plugin yang Rusak (_Plugin Incompatibility Hell_)**: Mengupdate plugin secara sembarangan di production yang menyebabkan pipeline gagal jalan karena dependensi Java yang tidak kompatibel.

## 11. Rangkuman Singkat

- **Kedaulatan Penuh atas Pipeline**: Jenkins menyediakan platform CI/CD self-hosted tanpa biaya lisensi per-menit, ideal untuk enterprise dengan kepatuhan regulasi data on-premise yang ketat.
- **Wajib Pisahkan Controller dan Agent**: Jangan pernah menjalankan eksekusi build di node master/controller; setel executor master ke 0 dan delegasikan seluruh build ke worker agents.
- **Manfaatkan Dynamic Ephemeral Pods**: Gunakan Kubernetes Plugin agar setiap build berjalan di dalam pod terisolasi yang otomatis dibuat dan dihancurkan, mengeliminasi risiko kontaminasi workspace.
- **Kendalikan Retensi Disk dan Plugin**: Terapkan rotasi pembersihan build lawas (`Discard old builds`) dan batasi pemasangan plugin pihak ketiga secara selektif untuk menjaga stabilitas JVM.
