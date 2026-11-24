# OpenTelemetry (OTel) & Distributed Tracing

## 1. Skenario Sistem Nyata

Pada platform e-commerce travel dan tiket terdistribusi, seorang pengguna menekan tombol "Bayar Pesanan" untuk memesan tiket penerbangan dan hotel. Request HTTP ini masuk ke `API Gateway`, lalu bercabang memanggil 8 microservices secara beruntun dan paralel: `Order Service` -> `User Service` -> `Flight Booking Service` -> `Hotel Service` -> `Payment Gateway` -> `Fraud Check` -> `Notification Service` -> `Ledger Service`.

Secara keseluruhan, request memakan waktu 4.5 detik (melebihi batas toleransi pengguna yang mengharapkan response < 1 detik).
Jika sistem hanya memiliki metrik agregat dan log teks terisolasi:

-   Metrik Prometheus hanya memberitahu bahwa `API Gateway` lambat, tetapi tidak bisa memberitahu microservice internal mana yang menjadi biang keladi kelambatan.
-   Membaca log di 8 service berbeda terasa seperti mencari jarum di tumpukan jerami karena tidak ada identitas unik pengikat (_Correlation ID_) yang menghubungkan seluruh request tersebut dari hulu ke hilir.
-   Engineer menghabiskan waktu berjam-jam saling lempar tanggung jawab antar-tim service (_blamestorming_).

**Solusi dengan OpenTelemetry (OTel):**
OpenTelemetry menginjeksi sebuah **`Trace ID`** unik berstandar W3C (`traceparent` header) pada request pertama di API Gateway. Saat request melintasi service lain, header ini diteruskan (_Context Propagation_). Setiap service membuat segmen waktu pemrosesan (_Span_), mencatat durasi query database, panggilan HTTP, dan rincian error. Dalam visualisasi trace (seperti Jaeger atau Grafana Tempo), engineer langsung melihat grafik alur berbentuk _Waterfall_: dari total 4.5 detik, ternyata 3.8 detik dihabiskan oleh satu query lambat (_slow query_) tanpa index di `Flight Booking Service`!

## 2. Apa Itu dan Buat Apa?

**OpenTelemetry (sering disingkat OTel)** adalah standar terbuka, API, pustaka SDK, dan agen kolektor (_OTel Collector_) yang dirancang untuk mengumpulkan data observabilitas terpadu—terdiri dari **Traces, Metrics, dan Logs (telemetri 3-in-1)**—dari aplikasi backend secara seragam tanpa terikat pada satu vendor tertentu (_vendor-agnostic_).

### Konsep Kunci Distributed Tracing di OTel:

1. **Trace**: Representasi perjalanan end-to-end lengkap dari sebuah request saat melintasi sistem terdistribusi. Sebuah Trace diidentifikasi oleh satu `Trace ID` 16-byte unik.
2. **Span**: Unit kerja tunggal di dalam sebuah Trace (misal pemanggilan fungsi HTTP, eksekusi query SQL, atau enkripsi password). Span memiliki nama, waktu mulai, waktu selesai, status, dan metadata (_Span Attributes_).
3. **Trace Context Propagation**: Mekanisme menyuntikkan (_inject_) dan mengekstrak (_extract_) metadata identitas trace melintasi batas jaringan via header HTTP atau gRPC metadata (menggunakan standar W3C TraceContext).
4. **OTel Collector**: Proxy perantara berkinerja tinggi yang menerima data telemetri dari aplikasi, memproses/memfilter data, lalu mengirimkannya (_export_) ke backend penyimpanan (Jaeger, Zipkin, Datadog, Prometheus, Tempo).

## 3. Kenapa Harus Pakai Ini ?

### 1. Eliminasi "Blamestorming" di Arsitektur Microservices

Di sistem monolitik, pelacakan error cukup melihat satu file stack trace. Di sistem microservices dengan 50 service terpisah, distributed tracing adalah satu-satunya cara ilmiah untuk membuktikan service mana yang menyebabkan latensi tinggi, timeout, atau kegagalan transaksi (_root cause analysis_).

### 2. Trace Sampling Rate untuk Mengendalikan Biaya Jaringan & Disk

Pada sistem high-throughput yang memproses 100.000 RPS, menyimpan 100% data trace akan memakan bandwidth jaringan dan disk Terabyte per hari! OTel menyediakan mekanisme **Sampling**:

-   **Head-Based Sampling**: Menentukan pengambilan sampel di awal request (misal merekam hanya 1% dari total request sukses).
-   **Tail-Based Sampling**: Menganalisis trace di OTel Collector setelah request selesai; merekam **100% request yang error (5xx) atau yang lambat (> 2 detik)**, dan membuang request sukses yang cepat!

### 3. Satu Standar Instrumentasi untuk Semua Vendor

Sebelum ada OTel, developer harus memasang SDK Jaeger untuk trace, SDK Prometheus untuk metrik, dan SDK vendor berbayar lainnya. Dengan OTel, kode Go-mu diinstrumentasi satu kali menggunakan OTel API standar. Kamu bebas mengganti backend penyimpanan telemetri kapan saja tanpa mengubah satu baris pun kode aplikasi.

## 4. Cara Kerjanya (Trace Waterfall & Context Propagation)

```
[ Client Request: POST /checkout ]
(Diberi Header: traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01)
               |
               v
[ 1. API Gateway ] ========================================== (Total: 4.5s)
         |
         +---> [ 2. User Service: GetProfile ] === (0.1s)
         |
         +---> [ 3. Flight Booking Service ] ====================== (3.8s SLOW!)
         |                 |
         |                 v
         |          [ Query SQL: SELECT * FROM flights ] ======== (3.6s BOTTLENECK!)
         |
         +---> [ 4. Payment Service: Charge ] ===== (0.4s)
```

## 5. Cuplikan Logika Inti

### Instrumentasi Distributed Tracing di Go dengan OpenTelemetry SDK

```go
package main

import (
	"context"
	"net/http"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("order-service")

// CallDownstreamWithTrace meneruskan Context W3C ke HTTP Request berikutnya
func CallDownstreamWithTrace(ctx context.Context, targetURL string) (*http.Response, error) {
	// 1. Buat Child Span baru di dalam context trace yang sedang berjalan
	ctx, span := tracer.Start(ctx, "HTTP Outbound: FlightBooking",
		trace.WithSpanKind(trace.SpanKindClient),
	)
	defer span.End()

	req, err := http.NewRequestWithContext(ctx, http.MethodPost, targetURL, nil)
	if err != nil {
		span.RecordError(err)
		return nil, err
	}

	// 2. Suntikkan (Inject) Traceparent Header ke HTTP Request Header (W3C standard)
	otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))

	// 3. Tambahkan attribute observabilitas kustom pada span
	span.SetAttributes(
		attribute.String("http.url", targetURL),
		attribute.String("peer.service", "flight-booking-service"),
	)

	client := &http.Client{Timeout: 5 * time.Second}
	resp, err := client.Do(req)
	if err != nil {
		span.RecordError(err)
		return nil, err
	}

	span.SetAttributes(attribute.Int("http.status_code", resp.StatusCode))
	return resp, nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Visibilitas Latensi End-to-End**: Menghilangkan tebak-tebakan; menyajikan grafik waterfall waktu eksekusi setiap baris microservice dan query database secara visual.
-   **Standarisasi Global CNCF**: Didukung penuh oleh Google Cloud, AWS, Microsoft, Datadog, Grafana Labs, dan seluruh raksasa teknologi.
-   **Tail-Based Sampling Cerdas**: Memungkinkan penyimpanan 100% data transaksi error atau berlatensi tinggi tanpa membuang anggaran penyimpanan untuk request sukses biasa.

### Kekurangan

-   **Overhead CPU dan Network Kecil**: Membuat dan mengekspor span untuk setiap request menambah sedikit latency (~0.1 ms - 0.5 ms) dan konsumsi memori alokasi objek Go context.
-   **Disiplin Context Propagation**: Jika ada developer yang lupa meneruskan `context.Context` Go ke pemanggilan database atau goroutine baru, rantai trace akan putus (_broken trace graph_).
-   **Volume Data Sangat Besar**: Tanpa konfigurasi sampling yang tepat di high-throughput, data trace akan membanjiri storage cluster hingga kehabisan disk.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Pakai OpenTelemetry

-   Arsitektur microservices terdistribusi dengan alur pemanggilan request bercabang dan melibatkan banyak tim independen.
-   Menemukan _bottleneck_ latensi P99 pada rantai panggilan synchronous (HTTP REST, gRPC) dan asynchronous (Kafka, RabbitMQ).
-   Menghubungkan antara metrik Prometheus, log error aplikasi, dan jejak trace dalam satu dashboard investigasi.

### Kapan Jangan Pakai OTel / Tracing Rumit

-   Aplikasi monolitik sederhana dengan satu database di mana profiling lokal (`pprof`) sudah cukup untuk menemukan fungsi yang lambat.
-   Tim kecil tanpa infrastruktur penyimpanan trace terdistribusi (seperti Jaeger atau Tempo).

## 8. Metrik Kunci & Observabilitas

Metrik untuk memantau performa agent OTel Collector:

-   **`otelcol_receiver_accepted_spans`**: Jumlah span telemetri yang berhasil diterima dari aplikasi backend.
-   **`otelcol_receiver_refused_spans`**: Jumlah span yang ditolak karena buffer penuh atau network failure.
-   **`otelcol_exporter_enqueue_failed_spans`**: Jumlah span yang gagal dikirimkan ke backend penyimpanan (Jaeger/Tempo).
-   **`otelcol_processor_dropped_spans`**: Jumlah span yang dibuang secara sengaja oleh aturan sampling filter.

## 9. Studi Kasus Nyata di Industri

### Pelacakan Latensi Terdistribusi Lintas 40 Microservices dan Diagnosa Bottleneck Database

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah platform super-app ride-hailing dan delivery memproses alur checkout pesanan yang melintasi 40 microservices (API Gateway -> Order -> Promo -> Driver Matching -> Fraud -> Payment -> Partner Switch). Rata-rata latensi p99 pesanan mendadak melonjak dari 400 ms menjadi 9.500 ms secara acak. Log server di masing-masing service hanya mencatat pesan generik `request completed`, tanpa ada petunjuk service mana yang menyebabkan kelambatan. Tim engineering dari 6 divisi berbeda saling menyalahkan (_finger-pointing_) selama 5 hari tanpa menemukan akar masalah.
- **Implementasi Solusi Industri (The Architectural Solution)**: Seluruh arsitektur microservices distandarisasi menggunakan *OpenTelemetry (OTel)* SDK dan W3C Trace Context (`traceparent` header). Setiap panggilan HTTP dan gRPC secara otomatis menginjeksikan dan mengekstrak konteks trace terdistribusi. Dijalankan armada *OTel Collector* yang mengumpulkan span telemetri dan mengekspornya ke Grafana Tempo/Jaeger. Diterapkan kebijakan *Tail-Based Sampling*: 100% trace yang menghasilkan error 5xx atau latensi di atas 2 detik disimpan secara permanen, sementara trace normal disampling hanya 2%.
- **Dampak & Hasil Sistem (The Outcome)**: Rantai grafik waterfall visualisasi trace mengungkap bottleneck dalam hitungan menit: sebuah query `SELECT ... FOR UPDATE` di Fraud Service mengalami antrean row-lock selama 8,2 detik akibat index miss. Waktu diagnosa akar masalah (*Root Cause Analysis / RCA*) terpangkas dari 5 hari menjadi di bawah 10 menit. Latensi checkout p99 berhasil dipulihkan kembali ke 250 ms, menghemat ratusan jam kerja engineer dan menekan biaya storage trace hingga 85% berkat tail-based sampling.

## 10. Fakta Menarik & Gotchas Production

-   **The Broken Context Leak (Penyebab Utama Rantai Trace Putus)**:  
    Di Go, developer pemula sering membuat goroutine baru dengan `context.Background()` alih-alih meneruskan `r.Context()` dari request HTTP awal, misalnya: `go processOrder(context.Background())`. Akibatnya, `Trace ID` hilang, dan span yang berjalan di dalam goroutine tersebut terputus dari rantai trace induknya (_orphaned span_)!  
    _Aturan Wajib:_ Selalu teruskan `ctx context.Context` aktif ke seluruh pemanggilan fungsi, goroutine, dan query database.
-   **100% Trace Sampling Bankruptcy**:  
    Merekam 100% trace pada sistem bervolume 100.000 RPS akan menghasilkan **8,6 miliar span per hari**, yang dapat menghabiskan puluhan Terabyte storage SSD dan menelan biaya cloud jutaan dolar per bulan!  
    _Solusi Wajib:_ Gunakan **Tail-Based Sampling** di OTel Collector: setel sampling rate 1% untuk request sukses berlatensi normal (< 200 ms), tetapi setel **100% sampling** untuk semua request berstatus HTTP 5xx atau latensi > 1 detik.

## 11. Rangkuman Singkat

- **Standar Terbuka Observabilitas Global**: OpenTelemetry (OTel) menyediakan satu set API, SDK, dan tooling standar vendor-agnostik untuk mengumpulkan tiga pilar observabilitas (Traces, Metrics, Logs).
- **Propagasi W3C Trace Context**: Kunci sukses distributed tracing adalah meneruskan header `traceparent` di setiap panggilan HTTP/gRPC dan membawa objek `context.Context` aktif ke seluruh goroutine dan query database.
- **Hemat Biaya dengan Tail-Based Sampling**: Merekam 100% trace pada sistem trafik tinggi akan menguras kapasitas penyimpanan; gunakan OTel Collector untuk menyaring trace sukses (1-2%) namun menyimpan 100% trace error atau latensi tinggi.
- **Isolasi Bottleneck dalam Hitungan Detik**: Distributed tracing mengubah dugaan abstrak menjadi grafik waterfall yang jelas, menunjukkan durasi presisi eksekusi setiap service, antrean pesan, dan query database.
