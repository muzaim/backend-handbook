# gRPC & Protocol Buffers (Protobuf)

## 1. Skenario Sistem Nyata

Pada arsitektur backend e-commerce skala besar, satu request checkout dari pengguna memicu belasan panggilan internal antar-microservice (_east-west traffic_: Order Service -> User Service, Payment Service, Inventory Service, Promotion Service, Fraud Service, dsb.).

Jika seluruh komunikasi internal antar-microservice menggunakan REST API standar (JSON over HTTP/1.1):

-   Setiap kali data dikirim, server pengirim harus melakukan _JSON serialization_ (mengubah objek memori menjadi string teks), dan server penerima harus melakukan _JSON parsing_ (membaca string teks kembali menjadi objek). Proses konversi teks ini sangat rakus CPU saat trafik mencapai 50.000 req/s.
-   Ukuran payload teks JSON membengkak karena nama-nama field (`"transaction_id"`, `"customer_email"`, `"created_at"`) diulang-ulang di setiap baris data.
-   Protokol HTTP/1.1 membutuhkan negosiasi handshake TCP baru untuk setiap request paralel atau mengalami masalah _Head-of-Line (HoL) Blocking_.

**Solusi dengan gRPC & Protocol Buffers:**
Microservices berkomunikasi menggunakan gRPC di atas protokol HTTP/2 dengan format serialisasi biner Protocol Buffers (`.proto`). Alih-alih mengirim teks panjang, Protobuf mengompresi data menjadi deretan biner ringkas menggunakan angka penanda (_field tags_). Satu koneksi TCP HTTP/2 dapat digunakan bersama untuk ratusan request asinkron sekaligus (_multiplexing_), memangkas penggunaan bandwidth hingga 60% dan menurunkan latensi latensi inter-service hingga 5x lipat.

## 2. Apa Itu dan Buat Apa?

gRPC (Google Remote Procedure Call) adalah framework RPC open-source berkinerja tinggi yang awalnya dikembangkan oleh Google. gRPC memungkinkan aplikasi client memanggil method atau fungsi pada server di mesin berbeda seolah-olah memanggil fungsi lokal di memori sendiri.

gRPC didukung oleh dua teknologi inti:

1. **Protocol Buffers (Protobuf)**: Mekanisme serialisasi data terstruktur biner yang independen dari bahasa pemrograman (_language-neutral_) dan platform.
2. **HTTP/2 Transport**: Protokol jaringan modern yang mendukung _bidirectional streaming_, kompresi header (HPACK), dan _multiplexing_ banyak request dalam satu koneksi TCP fisik tunggal.

Empat jenis metode komunikasi di gRPC:

-   **Unary RPC**: Client kirim 1 request, Server balas 1 response (seperti REST biasa).
-   **Server Streaming RPC**: Client kirim 1 request, Server membalas dengan aliran (_stream_) data berkelanjutan.
-   **Client Streaming RPC**: Client mengirim aliran stream data, Server membalas dengan 1 response rangkuman.
-   **Bidirectional Streaming RPC**: Client dan Server saling bertukar aliran data stream secara independen bersamaan.

## 3. Kenapa Harus Pakai Ini ?

### 1. Efisiensi Serialisasi Biner vs Teks

JSON dan XML adalah format teks berbasis karakter string yang mudah dibaca manusia (_human-readable_), tetapi sangat tidak efisien untuk mesin. Protobuf mengodekan data ke dalam format biner yang sangat padat (_type-safe_ dan terkompresi). Proses encode/decode Protobuf di CPU jauh lebih cepat daripada parser JSON tercepat sekalipun.

### 2. HTTP/2 Connection Multiplexing

Pada HTTP/1.1, client harus membuka banyak koneksi TCP terpisah jika ingin mengirim beberapa request paralel. Di gRPC (HTTP/2), ribuan request dan response dapat berjalan bolak-balik bersamaan di dalam **satu koneksi TCP tunggal** melalui sistem framing biner. Ini menghemat alokasi port soket dan jabat tangan TLS yang mahal.

## 4. Cara Kerjanya (Diagram Alur)

```
[ Developer Mendefinisikan Kontrak .proto ]
                     |
                     v
             [ protoc Compiler ]
            /                   \
           v                     v
   [ Go Client Stub ]   [ Java Server Skeleton ]
           |                     ^
           | 1. Panggil method   | 4. Eksekusi fungsi
           |    GetUser(id: 42)  |    lokal di Java
           v                     |
+-------------------------------------------------------+
|                    HTTP/2 Connection                  |
|                                                       |
|  Frame: [HEADERS] -> [DATA (Biner Protobuf Ringkas)]  |
+-------------------------------------------------------+
           |                     |
           v                     v
   [ Client Machine ]    [ Server Machine ]
```

## 5. Cuplikan Logika Inti

Definisi skema kontrak antarmuka di file `order_service.proto`:

```protobuf
syntax = "proto3";

package order.v1;

option go_package = "github.com/company/order/v1;orderv1";

service OrderService {
  // Unary RPC untuk membuat order baru
  rpc CreateOrder (CreateOrderRequest) returns (CreateOrderResponse);

  // Server Streaming RPC untuk streaming log status order
  rpc TrackOrderStatus (TrackOrderRequest) returns (stream OrderStatusUpdate);
}

message CreateOrderRequest {
  string customer_id = 1;
  double amount = 2;
  repeated string item_ids = 3;
}

message CreateOrderResponse {
  string order_id = 1;
  string status = 2;
  int64 created_at = 3;
}

message TrackOrderRequest {
  string order_id = 1;
}

message OrderStatusUpdate {
  string step = 1;       // Misal: "PACKING", "COURIER_ASSIGNED"
  string description = 2;
  int64 timestamp = 3;
}
```

Implementasi server handler di Go:

```go
type server struct {
    orderv1.UnimplementedOrderServiceServer
}

func (s *server) CreateOrder(ctx context.Context, req *orderv1.CreateOrderRequest) (*orderv1.CreateOrderResponse, error) {
    orderID := uuid.New().String()

    // Logika simpan transaksi ke database...

    return &orderv1.CreateOrderResponse{
        OrderId:   orderID,
        Status:    "PENDING",
        CreatedAt: time.Now().Unix(),
    }, nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Kecepatan & Throughput Ekstrem**: Payload biner 3-10x lebih kecil dari JSON dan proses de/serialisasi 5-8x lebih cepat di CPU.
-   **Kontrak Tipe Data Ketat (_Strict Type-Safety_)**: File `.proto` menjadi satu-satunya sumber kebenaran (_Single Source of Truth_) yang mencegah mismatch tipe data antar tim.
-   **Dukungan Multi-Bahasa Otomatis**: Kode client/server SDK untuk Go, Java, Python, Node.js, C#, dan Rust digenerate secara otomatis oleh `protoc`.

### Kekurangan

-   **Tidak Mudah Dibaca Manusia**: Payload biner tidak bisa diinspeksi langsung dengan mata telanjang di `curl` atau terminal tanpa tooling khusus seperti `grpcurl` atau Postman gRPC.
-   **Dukungan Browser Terbatas**: Browser web tidak dapat membuka koneksi HTTP/2 gRPC murni secara native; membutuhkan proxy perantara seperti _gRPC-Web_ atau Envoy.
-   **Masalah Load Balancing Default**: Karena HTTP/2 menggunakan koneksi TCP tunggal yang persisten, load balancer Layer 4 biasa akan mengarahkan seluruh trafik ke 1 server saja.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Komunikasi internal antar-microservices di dalam jaringan data center / VPC privat (_East-West traffic_).
-   Sistem pemrosesan streaming real-time yang membutuhkan throughput jutaan pesan per detik.
-   Komunikasi antar service dengan bahasa pemrograman yang berbeda-beda (_polyglot microservices_).

### Kapan Jangan Pakai

-   Public API yang langsung diakses oleh web frontend / mobile pihak ketiga (REST JSON jauh lebih universal dan mudah diintegrasikan).
-   Aplikasi CRUD sederhana dengan trafik rendah di mana fleksibilitas payload JSON lebih diutamakan daripada efisiensi biner.

## 8. Metrik Kunci & Observabilitas

Untuk memonitor kesehatan pemanggilan gRPC pada Prometheus / Datadog:

-   **`grpc_server_handling_seconds`**: Latensi eksekusi RPC di tingkat server (pantau distribusi persentil P95 dan P99).
-   **`grpc_server_handled_total{grpc_code="OK|DeadlineExceeded|Unavailable"}`**: Jumlah panggilan RPC yang sukses vs yang gagal berdasarkan status code gRPC.
-   **`grpc_server_msg_received_total` & `msg_sent_total`**: Volume frame pesan yang dikonsumsi dan diterbitkan, terutama pada alur streaming.
-   **`grpc_client_open_connections`**: Jumlah koneksi TCP fisik HTTP/2 yang aktif antar-pod.
-   **`grpc_stream_duration_seconds`**: Waktu aktif sebuah koneksi streaming gRPC sebelum terputus.

## 9. Studi Kasus Nyata di Industri

### Optimasi Throughput dan Serialisasi Biner pada Komunikasi Inter-Microservices Skala Raksasa

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah platform ride-hailing dengan ribuan microservices internal memproses lebih dari 400.000 panggilan RPC per detik antar-service menggunakan REST over HTTP/1.1 dengan payload JSON teks. Tingginya beban parsing/serializing string JSON dan *head-of-line blocking* pada ratusan koneksi TCP baru per request menyebabkan konsumsi CPU kluster membengkak hingga 65% hanya untuk memproses serialisasi data, disertai p99 inter-service latency di atas 85 ms.
- **Implementasi Solusi Industri (The Architectural Solution)**: Tim merancang ulang seluruh komunikasi internal microservice beralih ke gRPC di atas HTTP/2 dengan kontrak Protocol Buffers (Protobuf). Skema API dikunci secara terpusat dalam repositori `.proto` dengan code generator otomatis untuk Go dan Java. Koneksi HTTP/2 tunggal dimultiplexing untuk ribuan concurrent streams per host, dengan L7 load balancing menggunakan Envoy Proxy sidecars untuk mendistribusikan frame RPC secara merata ke seluruh pod backend.
- **Dampak & Hasil Sistem (The Outcome)**: Ukuran payload data di jaringan terpangkas rata-rata 68% (karena representasi biner Protobuf yang sangat ringkas tanpa field name strings). Penggunaan CPU server untuk serialisasi data anjlok sebesar 45%. Latensi pemanggilan inter-service p99 turun drastis dari 85 ms menjadi hanya 12 ms, sementara insiden *type mismatch* runtime antar-tim berkurang menjadi 0 berkat validasi tipe statis Protobuf.

## 10. Fakta Menarik & Gotchas Production

-   **The L4 Load Balancing Trap**: Ini adalah jebakan maut paling sering saat tim pertama kali migrasi ke gRPC: Load Balancer tradisional Layer 4 (seperti AWS NLB / round-robin TCP balancer) membagi beban berdasarkan _koneksi TCP_. Karena gRPC HTTP/2 hanya membuka **SATU koneksi TCP persisten**, seluruh 100.000 RPC call berikutnya akan terus ditembakkan ke Pod Server yang sama persis, sementara 19 pod lainnya menganggur!  
    _Solusi Wajib:_ Gunakan **L7 Load Balancer** yang mengerti frame HTTP/2 (seperti Envoy Proxy, Traefik, atau Kubernetes gRPC Service) atau terapkan _Client-Side Load Balancing_.
-   **Pantangan Mengubah Tag Nomor di .proto**: Di Protobuf, nama variabel (misal `customer_id`) tidak pernah dikirim lewat jaringan. Yang dikirim adalah nomor tag-nya (`1`). Jika kamu mengubah `string customer_id = 1;` menjadi `string customer_id = 2;`, data lama akan corrupt dan sistem penerima akan membaca data ke field yang salah! Tag nomor bersifat permanen selamanya.

## 11. Rangkuman Singkat

- **Serialisasi Biner Super Efisien**: Protobuf mengkodekan data menjadi biner murni berbasis integer field tags, menghasilkan payload yang jauh lebih kecil dan cepat di-parse dibanding teks JSON.
- **Multiplexing HTTP/2**: gRPC memanfaatkan koneksi HTTP/2 tunggal yang persisten untuk mengalirkan ribuan concurrent RPC streams tanpa overhead handshake TCP berulang.
- **Wajib L7 Load Balancing**: Hindari L4 TCP load balancer yang akan memusatkan seluruh trafik ke satu instance; gunakan reverse proxy L7 (Envoy/Traefik) atau client-side load balancing.
- **Stabilitas Tag Nomor Protobuf**: Jangan pernah mengubah nomor field tag dalam file `.proto` setelah dirilis, karena integritas backward compatibility data biner bergantung sepenuhnya pada nomor tag tersebut.
