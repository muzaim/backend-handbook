# Centralized Logging & Log Aggregation

## 1. Skenario Sistem Nyata

Pada arsitektur perbankan digital dan e-commerce berskala 200 pod Kubernetes, seorang nasabah melaporkan bahwa saldo dompet digitalnya terpotong Rp 250.000, tetapi transaksi pembayaran merchant dinyatakan gagal dan tidak ada bukti struk yang terbit.

Jika sistem masih mengandalkan pencatatan log tradisional yang tersimpan lokal di dalam masing-masing container:

-   Customer Support dan Engineer tidak tahu di pod mana dari 200 pod yang memproses request transaksi nasabah tersebut.
-   Kubernetes sering melakukan _auto-scaling down_ atau me-restart pod yang bermasalah. Ketika pod mati (_evicted/terminated_), seluruh file log lokal di dalam disk container terhapus permanen detik itu juga (_ephemeral container storage_), menghilangkan bukti audit forensik!
-   Melakukan SSH manual ke server produksi untuk menjalankan `grep` dilarang keras oleh standar kepatuhan keamanan industri perbankan (PCI-DSS / ISO 27001).

**Solusi dengan Centralized Logging:**
Seluruh pod aplikasi backend menulis log dalam format **Structured JSON** ke `stdout/stderr` dengan menyertakan `trace_id` dan `user_id`. Log Shipper ringan (seperti Promtail, Fluent Bit, atau Vector) yang berjalan sebagai DaemonSet di setiap node server langsung menyedot aliran log tersebut, mengompresinya, dan mengirimkannya ke kluster log terpusat (**Grafana Loki** atau **Elasticsearch**). Hanya dalam hitungan 2 detik, tim support cukup mengetik `trace_id` transaksi di Grafana/Kibana untuk melihat rekaman lengkap perjalanan error finansial tersebut secara kronologis, meskipun pod yang memprosesnya sudah mati 3 jam yang lalu!

## 2. Apa Itu dan Buat Apa?

**Centralized Logging (Pencatatan Log Terpusat)** adalah arsitektur pengumpulan, pengindeksan, pengayaan, dan penyimpanan seluruh aliran data log aplikasi dan infrastruktur dari ratusan server/kontainer ke satu penyimpanan terpusat yang dapat dicari secara cepat (_searchable storage_).

### Komponen Utama Pipeline Centralized Logging:

1. **Application Structured Logger**: Pustaka logging di aplikasi backend (seperti `log/slog` atau `uber-go/zap` di Go) yang mencetak log dalam format key-value terstruktur (JSON).
2. **Log Collector / Shipper Agent**: Agen latar belakang berkinerja tinggi (Promtail, Fluent Bit, Vector, Logstash) yang membaca file log kontainer, menambahkan metadata label (pod name, namespace), lalu mengirimkannya ke server log.
3. **Log Storage & Indexing Engine**: Mesin penyimpanan log skala besar:
    - **Grafana Loki**: _"Like Prometheus, but for logs"_; hanya mengindeks metadata label dan menyimpan payload log mentah terkompresi di object storage (S3/GCS), menghasilkan penghematan biaya storage hingga 80%.
    - **Elasticsearch / OpenSearch**: Mengindeks seluruh kata (_Full-Text Inverted Index_) untuk pencarian teks super cepat dengan kebutuhan RAM lebih tinggi.
4. **Visualization & Query Interface**: Antarmuka web (Grafana Explore / Kibana) untuk menjalankan query log secara interaktif menggunakan LogQL atau KQL.

## 3. Kenapa Harus Pakai Ini ?

### 1. Data Permanen Mengatasi Karakter Ephemeral Kontainer

Di era Kubernetes, container pod bersifat sementara (_ephemeral_). Pod bisa dibuat, di-reschedule, atau dimusnahkan kapan saja oleh scheduler. Centralized logging memastikan bahwa hilangnya pod fisik tidak menyebabkan hilangnya rekaman audit kepatuhan (_audit trail_) dan rekam jejak error transaksi.

### 2. Structured JSON vs Text String Biasa

Di sistem high-throughput, log berbasis teks bebas (`fmt.Println("User login gagal untuk user: 123")`) sangat sulit dianalisis mesin secara otomatis. **Structured Logging** (`{"level":"error","msg":"login_failed","user_id":"123","latency_ms":42}`) memungkinkan mesin log mengurai, memfilter, dan mengagregasi ribuan error per detik berdasarkan parameter tertentu secara instan.

### 3. Korelasi Log dengan Distributed Tracing (Trace ID Injection)

Dengan menyuntikkan `trace_id` yang sama ke dalam setiap baris log di seluruh microservices, engineer dapat berpindah secara mulus dari metrik spike di Grafana, ke waterfall trace di Tempo, langsung ke **baris log persis yang mencetak pesan error exception** hanya dengan satu klik!

## 4. Cara Kerjanya (Arsitektur Pipeline Log Shipper ke Loki / Elasticsearch)

```
[ KUBERNETES NODE ]
Pod 1 (Go App) ---> Write JSON ke stdout/stderr
Pod 2 (Go App) ---> Write JSON ke stdout/stderr
        |
        v
/var/log/pods/ (Docker/Containerd Node Disk)
        |
        v (Diserap secara real-time)
[ Log Shipper: Fluent Bit / Promtail (DaemonSet) ]
  - Parse JSON & Inject K8s Metadata (Pod, Namespace)
  - Batching & Kompresi Gzip/Snappy
        |
        +-----> (Kirim Batch melalui Jaringan)
        |
        v
+-------------------------------------------------+
| CENTRAL LOG STORAGE (Grafana Loki / Elastic)    |
| - Simpan Chunk Log Terkompresi ke S3 / MinIO    |
| - Indeks Metadata Label                         |
+-------------------------------------------------+
        |
        v (Query LogQL / KQL)
+-------------------------------------------------+
| GRAFANA EXPLORE / KIBANA DASHBOARD              |
| Query: {app="order"} |= "status=500"            |
+-------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### 1. Structured Logging di Go dengan `log/slog` & Otomatisasi Trace ID

Menggunakan pustaka logging standar modern Go `log/slog` dengan korelasi `Trace ID` dari Context:

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"

	"go.opentelemetry.io/otel/trace"
)

var logger *slog.Logger

func init() {
	// Inisialisasi Structured JSON Handler ke stdout
	handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	})
	logger = slog.New(handler)
}

// LogWithContext mengekstrak TraceID dari Context dan menyuntikkannya ke baris log JSON
func LogWithContext(ctx context.Context) *slog.Logger {
	span := trace.SpanFromContext(ctx)
	if span.SpanContext().IsValid() {
		return logger.With(
			slog.String("trace_id", span.SpanContext().TraceID().String()),
			slog.String("span_id", span.SpanContext().SpanID().String()),
		)
	}
	return logger
}

func handlePayment(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	log := LogWithContext(ctx)

	// Baris log ini otomatis membawa trace_id yang sama dengan distributed tracing
	log.Info("memulai pemrosesan pembayaran",
		slog.String("order_id", "ORD-12345"),
		slog.Float64("amount", 250000.0),
	)

	// Jika terjadi error:
	log.Error("gagal mendebit saldo nasabah",
		slog.String("order_id", "ORD-12345"),
		slog.String("error_code", "INSUFFICIENT_FUNDS"),
	)
}
```

### 2. Konfigurasi Log Shipper DaemonSet (Fluent Bit YAML)

```yaml
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            docker
    Tag               kube.*
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Merge_Log           On
    Merge_Log_Key       log_processed
    Keep_Log            Off

[OUTPUT]
    Name   loki
    Match  kube.*
    Host   loki.monitoring.svc.cluster.local
    Port   3100
    Labels job=fluentbit, namespace=$kubernetes['namespace_name']
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Audit Permanen & Keamanan**: Log transaksi tersimpan aman di luar node server aplikasi dan mematuhi regulasi perbankan/fintech.
-   **Pencarian Cepat Antar-Service**: Kemampuan memfilter jutaan baris log dalam hitungan detik menggunakan query terstruktur.
-   **Korelasi Metrik-Trace-Log**: Mengklik titik error di grafik Grafana langsung membuka log error yang relevan di detik yang sama.

### Kekurangan

-   **Biaya Penyimpanan & Egress Sangat Besar**: Menulis jutaan baris log per detik ke disk dapat memakan puluhan Terabyte data per bulan jika tidak diatur level log-nya (_DEBUG log trap_).
-   **Overhead CPU I/O Node**: Agen log shipper (Fluent Bit / Promtail) yang membaca ribuan file disk kontainer dapat memakan 10-20% CPU node jika parsing regex terlalu berat.
-   **Log Storm saat Downtime**: Saat database tumbang, 200 pod secara serentak mencetak error stack trace ribuan kali per detik, membanjiri kluster logging (_thundering log storm_).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Pakai Centralized Logging

-   Seluruh arsitektur microservices berbasis Kubernetes dan kontainer cloud.
-   Sistem keuangan, transaksi pembayaran, dan otentikasi yang memerlukan jejak audit forensik.
-   Menghubungkan log error dengan tracing OpenTelemetry melalui `trace_id`.

### Kapan Jangan Mencatat Log Berlebihan (Over-Logging)

-   Jangan mencetak data payload besar di loop komputasi berkecepatan tinggi (misal mencetak paket biner GPS kurir setiap 100 ms).
-   Jangan mencetak data sensitif pengguna (PII) ke log: nomor kartu kredit, CVV, password plaintext, atau PIN bank (risiko pelanggaran hukum privasi GDPR / PDP).

## 8. Metrik Kunci & Observabilitas

Metrik untuk memantau stabilitas pipeline logging:

-   **`fluentbit_output_proc_records_total`**: Jumlah record log yang berhasil dikirim ke server penyimpanan log.
-   **`fluentbit_output_retries_failed_total`**: Jumlah pengiriman log yang gagal karena server penyimpanan down.
-   **`loki_ingester_bytes_per_second`**: Volume bandwidth log yang masuk ke kluster Loki.
-   **`loki_discarded_samples_total`**: Jumlah baris log yang terpaksa dibuang karena rate limit ingestion terlampaui.

## 9. Studi Kasus Nyata di Industri

### Pipa Penyerapan 15 Terabyte Log Harian dengan Metadata Labeling dan Eliminasi Biaya CloudWatch

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah institusi fintech dan perbankan digital mengoperasikan 250 microservices di kluster Kubernetes dengan output 15 Terabyte log mentah per hari. Awalnya, log dialirkan langsung ke cloud SaaS proprietary (AWS CloudWatch / Datadog) menggunakan format teks bebas tanpa parsing. Ketika terjadi insiden kegagalan payment switch, lonjakan error log menghasilkan *log storm* yang mendongkrak tagihan monitoring bulanan hingga ratusan ribu dolar. Selain itu, investigasi audit kepatuhan membutuhkan waktu 3 jam per query karena sistem harus melakukan full-text regex scanning pada data log tak terstruktur yang terfragmentasi di ratusan stream.
- **Implementasi Solusi Industri (The Architectural Solution)**: Tim merancang ulang pipa pengumpulan log terpusat menggunakan *Fluent Bit* sebagai DaemonSet di setiap worker node K8s dan *Grafana Loki* yang didukung object storage terdistribusi (MinIO/S3). Aplikasi diwajibkan menggunakan library *Structured JSON Logging* (`slog` di Go) dengan atribut wajib: `trace_id`, `service`, `environment`, dan `error_code`. Fluent Bit membaca log container dari node disk, memfilter dan mem-parsing JSON, mengekstrak label metadata terbatas (namespace, pod, service), dan mengirimkan chunks log terkompresi ke Loki. Query audit dan investigasi insiden dijalankan via Grafana LogQL dengan filter berbasis label dan trace ID.
- **Dampak & Hasil Sistem (The Outcome)**: Biaya infrastruktur penyimpanan log bulanan anjlok hingga 78% (penghematan ratusan ribu dolar per tahun) karena Loki hanya mengindeks metadata label alih-alih mengindeks setiap kata dalam pesan log. Waktu eksekusi query audit forensik terpangkas dari 3 jam menjadi < 5 detik. Rata-rata CPU overhead agent logging di setiap node turun menjadi < 1,5% CPU berkat efisiensi engine C Fluent Bit.

## 10. Fakta Menarik & Gotchas Production

-   **The Production `DEBUG` Log Bill Shock**:  
    Developer sering lupa mematikan log level `DEBUG` saat mendeploy kode ke production. Di sistem yang melayani 50.000 RPS, satu pod yang mencetak log debug untuk setiap request akan memproduksi **4,3 miliar baris log per hari**! Tagihan cloud logging (seperti AWS CloudWatch Logs atau Datadog) bisa melonjak ribuan dolar dalam semalam hanya untuk menyimpan teks sampah!  
    _Aturan Wajib:_ Kunci log level di production pada **`INFO`** atau **`WARN`**. Hanya aktifkan `DEBUG` secara selektif via konfigurasi dinamis.
-   **Log Injection Vulnerability (Log Forging / Log4Shell)**:  
    Jika aplikasimu mencatat input teks mentah dari pengguna ke log (misal `log.Info("Pencarian: " + userInput)`), penyerang dapat menyuntikkan karakter newline (`\n`) dan membuat baris log palsu bertuliskan `[INFO] Admin transfer berhasil Rp 1.000.000.000` untuk mengelabui auditor!  
    _Solusi Wajib:_ Selalu gunakan **Structured JSON Logging** (`log.Info("pencarian", slog.String("query", userInput))`), di mana karakter berbahaya otomatis di-escape dalam format string JSON yang aman.

## 11. Rangkuman Singkat

- **Wajib Structured JSON Logging**: Selalu catat log aplikasi dalam format JSON terstruktur (menggunakan `slog` di Go) agar mudah di-parse dan difilter oleh mesin pengumpul log otomatis.
- **Sertakan Korelasi Trace ID**: Tambahkan `trace_id` di setiap baris log transaksi; ini adalah jembatan emas yang menghubungkan log individual dengan distributed tracing OpenTelemetry.
- **Pilih Pendekatan Indexing yang Tepat**: Gunakan Elasticsearch jika butuh pencarian teks bebas instan di isi pesan, atau gunakan Grafana Loki (hanya mengindeks label metadata) untuk menghemat biaya storage hingga 75%.
- **Disiplin Log Level di Production**: Kunci log level produksi pada `INFO` atau `WARN`; jangan biarkan level `DEBUG` aktif permanen karena akan memicu lonjakan tagihan storage yang sangat masif (*bill shock*).
