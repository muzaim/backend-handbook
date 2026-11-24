# Publish / Subscribe (Pub/Sub)

## 1. Skenario Sistem Nyata

Pada arsitektur microservices e-commerce skala besar, saat seorang pengguna berhasil menyelesaikan pembayaran transaksi belanjaan (`OrderPaid`), serangkaian sistem downstream memerlukan informasi tersebut secara serentak.

Ada 4 service independen di backend yang berkepentingan terhadap fakta ini:

1. `Inventory Service` harus memotong stok barang di database gudang.
2. `Notification Service` harus mengirimkan email kwitansi dan notifikasi push WhatsApp ke pelanggan.
3. `Logistics Service` harus menerbitkan manifest order ke sistem kurir pengiriman.
4. `Analytics / BI Service` harus mencatat atribusi konversi kampanye promosi ke data lake.

Jika sistem dirancang secara sinkron (panggilan HTTP langsung antar-service):

-   `Order Service` terpaksa melakukan 4 panggilan jaringan beruntun ke 4 service tersebut (_tight temporal coupling_).
-   Jika `Notification Service` mengalami latensi 4 detik, alur checkout pengguna tertahan 4 detik.
-   Jika tim data engineer ingin menambahkan service ke-5 (misal: _Fraud Auditing Service_), tim `Order Service` harus memodifikasi source code mereka dan mendeploy ulang aplikasinya.

**Solusi dengan Pub/Sub (misal: Apache Kafka / Google Cloud Pub/Sub):**
`Order Service` hanya berperan sebagai _Publisher_. Saat pembayaran lunas, ia cukup menerbitkan satu event ke broker: `Topic: order-paid` dalam waktu < 2 milidetik. Sistem broker secara otomatis menggandakan dan mendistribusikan (_fan-out_) payload event tersebut ke seluruh service yang berlangganan (_Subscribers_) secara paralel, asinkron, dan terisolasi penuh. Service baru dapat ditambahkan kapan saja cukup dengan mendaftar sebagai subscriber baru tanpa menyentuh satu baris pun kode di `Order Service`.

## 2. Apa Itu dan Buat Apa?

Pub/Sub (Publish/Subscribe) adalah pola arsitektur pertukaran pesan asinkron di mana pengirim pesan (**Publisher**) tidak mengirimkan pesan langsung ke penerima tertentu, melainkan mengelompokkan pesan ke dalam saluran topik (**Topic/Channel**).

Penerima (**Subscriber**) mendaftarkan ketertarikannya pada satu atau lebih topik. Setiap kali ada pesan baru di topik tersebut, sistem broker akan **menggandakan dan mendistribusikan pesan tersebut ke seluruh subscriber yang terdaftar (_Fan-Out Pattern_)**.

### Perbedaan Kunci: Message Queue vs Pub/Sub

-   **Message Queue (1-to-1)**: Satu pesan dimasukkan ke antrean dan hanya diambil oleh **SATU** worker yang menang (_competing consumers_). Pesan langsung hilang setelah di-ACK.
-   **Pub/Sub (1-to-Many)**: Satu pesan dikirim ke sebuah topik dan disalin ke **SEMUA** subscriber yang berlangganan. Jika ada 5 subscriber berbeda, kelimanya akan mendapatkan salinan pesan yang sama persis.

Contoh software populer: Apache Kafka, Google Cloud Pub/Sub, AWS SNS, NATS, Redis Pub/Sub, RabbitMQ (menggunakan Fanout Exchange).

## 3. Kenapa Harus Pakai Ini ?

Dalam aplikasi belanja online saat seorang pengguna berhasil melakukan pembayaran order (`OrderPaid`):

1. Service Inventory harus memotong stok barang di gudang.
2. Service Notifikasi harus mengirimkan email receipt dan push notification ke HP pengguna.
3. Service Logistik harus mencetak label pengiriman kurir.
4. Service Analisis Data/Marketing harus mencatat metrik konversi iklan.

**Tanpa Pub/Sub (Panggilan HTTP Sinkron / Monolith Coupling):**
Service Order harus memanggil 4 API service tersebut satu per satu via HTTP. Jika Service Notifikasi sedang lemot 4 detik, checkout pengguna tertahan 4 detik. Jika tim data engineer ingin menambahkan service ke-5 (misal Fraud Detection), tim Order Service harus mengubah kode mereka dan mendeploy ulang aplikasinya.

**Dengan Pub/Sub:**
Service Order cukup mempublikasikan 1 event: `Topic: order-paid` ke broker (memakan waktu 1 milidetik). Keempat service lainnya mendengarkan topik tersebut dan memproses tugasnya masing-masing secara independen dan paralel!

## 4. Cara Kerjanya (Diagram Alur - Fan-Out)

```
                            +-----------------+
                            |  Order Service  | (Publisher)
                            +-----------------+
                                     |
                1. Publish Event:    |
                   Topic: "order-paid"
                   Payload: { order_id: 88, amount: 250000 }
                                     v
                 +---------------------------------------+
                 |            Pub/Sub Broker             |
                 |         [ Topic: order-paid ]         |
                 +---------------------------------------+
                     /               |               \
       Salin ke Sub 1 /    Salin ke Sub 2 |     \ Salin ke Sub 3
                     /               |               \
                    v                v                v
            +---------------+ +---------------+ +---------------+
            | Inventory Svc | | Notif Svc     | | Analytics Svc |
            | (Potong Stok) | | (Kirim Email) | | (Catat Metrik)|
            +---------------+ +---------------+ +---------------+
```

## 5. Cuplikan Logika Inti (Pub/Sub di Go)

Implementasi Publisher event dan Subscriber loop berbasis topik di Go:

```go
package main

import (
	"context"
	"fmt"

	"github.com/redis/go-redis/v9"
)

var ctx = context.Background()

// PublishOrderEvent mempublikasikan event satu-ke-banyak (fan-out)
func PublishOrderEvent(rdb *redis.Client, topic string, orderJSON string) error {
	return rdb.Publish(ctx, topic, orderJSON).Err()
}

// SubscribeOrderEvents mengonsumsi aliran event secara independen
func SubscribeOrderEvents(rdb *redis.Client, topic string, consumerName string) {
	pubsub := rdb.Subscribe(ctx, topic)
	ch := pubsub.Channel()

	for msg := range ch {
		fmt.Printf("[%s] Menerima event baru: %s
", consumerName, msg.Payload)
		// Proses event sesuai kebutuhan masing-masing subscriber (Notification / Analytics)
	}
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Ekstrem Decoupled**: Publisher tidak perlu tahu siapa saja subscriber-nya, di mana IP address mereka, atau berapa jumlah subscriber yang ada.
-   **Kemudahan Menambah Fitur Baru**: Tim lain bisa menambahkan service baru (misal audit log atau accounting) cukup dengan berlangganan ke topik yang sudah ada tanpa perlu mengubah satu baris pun kode di Publisher.
-   **Throughput Skala Masif**: Sistem seperti Apache Kafka mampu mendistribusikan jutaan event per detik berkat partisi topik paralel.

### Kekurangan

-   **Kehilangan Visibilitas Alur Pesan**: Sulit memahami rantai sebab-akibat bisnis secara visual karena tidak ada satu orchestrator pusat yang memegang peta alur (_harder to trace end-to-end flow_).
-   **Duplikasi Beban Jaringan**: Menggandakan 1 pesan ke 10 subscriber berarti meningkatkan volume I/O jaringan 10 kali lipat di cluster broker.
-   **Pesan Hilang pada Ephemeral Broker**: Beberapa implementasi Pub/Sub ringan (seperti Redis Pub/Sub biasa) bersifat _fire-and-forget_; jika subscriber sedang offline saat pesan dipublish, pesan hilang selamanya (berbeda dengan Kafka yang menyimpan log event di disk).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Penyiaran perubahan data bisnis (_Business Events Broadcasting_) ke banyak domain microservices independen.
-   Sistem metrik, audit log, dan distributed tracing yang dikonsumsi oleh data warehouse dan security tools.
-   Fitur real-time kolaborasi (chat multi-user, live sports score ticker, order status updates via WebSockets).

### Kapan Jangan Pakai

-   Alur kerja sekuensial yang membutuhkan jaminan balasan (_Request-Reply pattern_, misal login user butuh JWT token kembali).
-   Jika penerima pesan hanya ada satu dan tugasnya murni antrean kerja komputasi (gunakan Message Queue point-to-point biasa).

## 8. Metrik Kunci & Observabilitas

Untuk memonitor kesehatan cluster Pub/Sub dan consumer group pada Grafana atau Datadog:

-   **`kafka_consumergroup_lag` / `pubsub_subscription_backlog`**: Jumlah event yang belum dibaca per partisi oleh consumer group tertentu.
-   **`topic_publish_rate_messages_per_sec`**: Volume pesan masuk per detik di tingkat topik.
-   **`consumergroup_rebalance_rate`**: Frekuensi terjadinya rebalance antar pod/instance worker consumer; jika terlalu sering, ada worker yang mengalami crash loop.
-   **`bytes_in_per_sec` & `bytes_out_per_sec`**: Bandwidth I/O broker untuk mengantisipasi saturasi kartu jaringan (_NIC saturation_).
-   **`undelivered_messages_dropped_total`**: Jumlah pesan yang gagal dikirimkan ke subscriber karena timeout atau overflow antrean.

## 9. Studi Kasus Industri Nyata

### Decoupling Fan-Out Event Penyelesaian Perjalanan pada Ride-Hailing

- **Tantangan Rekayasa Riil (The Problem)**:
  Pada sebuah platform transportasi online berskala jutaan perjalanan harian, layanan pemesanan (`trip-service`) awalnya memanggil 6 downstream microservices secara sinkron melalui HTTP setiap kali perjalanan selesai (`TripCompleted`): layanan penagihan dompet (`billing-service`), bonus insentif pengemudi (`driver-incentive`), deteksi kecurangan (`fraud-service`), poin loyalitas pengguna (`loyalty-service`), notifikasi rating (`push-notification`), dan analitik data warehouse. Bencana terjadi ketika database pada `loyalty-service` mengalami degradasi latensi hingga 5 detik per request. Seluruh proses penyelesaian perjalanan di smartphone pengemudi menggantung selama 5 detik, pengemudi tidak bisa menerima pesanan berikutnya, dan antrean goroutine di `trip-service` kolaps. Selain itu, setiap kali tim bisnis ingin menambah fitur baru (seperti kalkulasi emisi karbon), tim inti `trip-service` terpaksa mengubah kode dan melakukan deployment berisiko tinggi (*tight coupling*).

- **Implementasi Solusi Industri (The Architectural Solution)**:
  1. **Event Fan-Out via Pub/Sub Cluster**: Ketergantungan sinkron HTTP dihapus total. Begitu perjalanan selesai, `trip-service` hanya menerbitkan satu event tunggal `TripCompletedEvent` ke topik `trips.lifecycle` di Apache Kafka dalam waktu 1.5ms, lalu langsung membebaskan pengemudi untuk mengambil order baru.
  2. **Isolated Independent Consumer Groups**: Keenam downstream service didaftarkan sebagai *Consumer Group* terpisah yang membaca partisi topik Kafka secara mandiri. Kecepatan baca masing-masing consumer terisolasi penuh: jika `loyalty-service` lambat, consumer lag hanya menumpuk pada group loyalitas tanpa memperlambat proses penagihan dompet (`billing-group`) yang membutuhkan latensi kilat.
  3. **Zero-Code Extensibility**: Ketika tim ingin meluncurkan fitur audit emisi karbon, mereka cukup mendaftarkan consumer group baru (`carbon-footprint-service`) yang membaca topik yang sama, tanpa pernah menyentuh satu baris kode pun di `trip-service`.

- **Dampak & Hasil Sistem (The Engineering Outcome)**:
  - Durasi penyelesaian perjalanan di smartphone pengemudi terpangkas 96%, turun dari 5.200ms menjadi hanya 180ms.
  - Kerusakan atau pemeliharaan server loyalitas dan analitik menghasilkan dampak 0% terhadap kelancaran penagihan finansial dan ketersediaan pengemudi.
  - Kecepatan inovasi fitur tim produk meningkat pesat karena service baru dapat langsung mengonsumsi data historis dan real-time secara mandiri.

## 10. Fakta Menarik & Gotchas Production

-   **The Slow Subscriber Problem**: Apa yang terjadi jika salah satu dari lima subscriber berjalan sangat lambat sementara Publisher terus memompa 50.000 event per detik? Di broker berbasis memory, antrean buffer untuk subscriber lambat tersebut akan membengkak hingga broker kehabisan RAM (_OOM Crash_), melumpuhkan subscriber lain yang sehat!  
    _Pencegahan:_ Gunakan broker berbasis persistent log seperti **Apache Kafka** di mana setiap subscriber membaca offset secara mandiri dari disk tanpa membebani buffer memory subscriber lain.
-   **Toko Sebelah Menghitung Metrik 2 Kali**: Jika subscriber di-restart dan membaca ulang pesan dari offset lama tanpa proteksi, sistem analytics bisa mencatat omset harian dua kali lipat. Pastikan seluruh subscriber Pub/Sub menerapkan prinsip **Idempotency**!

## 11. Rangkuman Singkat

1. **Pola Satu-ke-Banyak (*Fan-Out Decoupling*)**: Publisher menerbitkan satu event tanpa perlu mengetahui siapa atau berapa banyak consumer yang mendengarkan, memutus keterikatan ketat antar microservice.
2. **Isolasi Kecepatan Konsumsi**: Setiap consumer group memelihara cursor offset pembacaan secara mandiri; kelambatan satu consumer tidak akan mengganggu pemrosesan consumer lainnya.
3. **Ekstensibilitas Tanpa Downtime**: Menambahkan fitur, analitik, atau downstream listener baru dapat dilakukan kapan saja tanpa perlu mengubah atau me-reboot service penerbit event.
4. **Wajib Idempotent di Sisi Konsumen**: Sistem Pub/Sub umumnya menjamin *at-least-once delivery*; pastikan setiap subscriber mengecek duplikasi event agar metrik dan data tidak terhitung ganda.
