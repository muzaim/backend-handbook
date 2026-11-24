# RabbitMQ (AMQP Message Broker)

## 1. Skenario Sistem Nyata

Pada sebuah platform e-commerce dan logistik terpadu, layanan pemesanan (_Order Service_) menghasilkan berbagai variasi event saat checkout terjadi. Berbagai layanan downstream membutuhkan perlakuan routing yang berbeda:

-   **Layanan Notifikasi SMS**: Hanya ingin menerima pesanan berstatus prioritas tinggi (`order.urgent.*`).
-   **Layanan Gudang & Inventaris**: Wajib menerima seluruh jenis pesanan tanpa kecuali (`order.*.*`).
-   **Layanan Investigasi Fraud**: Hanya memeriksa pesanan bernominal besar dengan metode pembayaran kartu kredit (`order.*.creditcard`).

Jika menggunakan panggilan REST API langsung:

-   Service order akan mengalami _cascading latency_ dan coupling tinggi karena harus mengenal semua endpoint downstream satu per satu.
-   Jika layanan SMS sedang down, transaksi checkout e-commerce ikut gagal.

Jika menggunakan Apache Kafka:

-   Kafka sangat hebat untuk jutaan event streaming sekuensial, namun Kafka tidak memiliki kemampuan _content/routing-key filtering_ berbasis wildcard secara native di broker; konsumen terpaksa membaca seluruh event dan memfilter sendiri di kode aplikasi.

**Solusi dengan RabbitMQ (AMQP Topic Exchange):**
Order service cukup mem-publish pesan ke sebuah **Topic Exchange** dengan _routing key_ spesifik (misal `order.urgent.creditcard`). RabbitMQ bertindak sebagai _Smart Broker_ yang secara otomatis menduplikasi dan mengarahkan pesan tersebut ke `sms_queue`, `warehouse_queue`, dan `fraud_queue` berdasarkan aturan _binding pattern_ masing-masing. Setiap worker downstream dapat memproses pesan secara independen dengan konfirmasi manual (_explicit ACK_). Jika worker gagal, pesan dapat dikembalikan ke antrean (_NACK requeue_) atau dialihkan ke _Dead Letter Exchange_ (DLX).

## 2. Apa Itu dan Buat Apa?

RabbitMQ adalah message broker open-source berbasis protokol standar industri **AMQP 0-9-1** (_Advanced Message Queuing Protocol_).

Komponen inti RabbitMQ:

1. **Producer**: Aplikasi pengirim pesan.
2. **Exchange**: Komponen penerima pesan dari producer yang bertugas menentukan ke antrean mana pesan akan diarahkan berdasarkan _routing key_.
    - **Direct Exchange**: Mencocokkan routing key secara persis (_exact match_).
    - **Fanout Exchange**: Menyebarkan pesan ke seluruh antrean terhubung tanpa peduli routing key (_broadcast_).
    - **Topic Exchange**: Mencocokkan routing key berdasarkan pola wildcard (`*` untuk satu kata, `#` untuk nol atau lebih kata).
    - **Headers Exchange**: Mengarahkan pesan berdasarkan atribut header pesan, bukan routing key.
3. **Queue**: Buffer antrean FIFO di memori dan disk tempat pesan disimpan sampai diambil oleh consumer.
4. **Binding**: Hubungan konfigurasi antara Exchange dan Queue.
5. **Consumer**: Worker aplikasi yang mengambil dan memproses pesan dari Queue.

## 3. Kenapa Harus Pakai Ini ?

RabbitMQ dan Apache Kafka sering diperbandingkan. Keduanya memiliki filosofi yang bertolak belakang:

-   **RabbitMQ ("Smart Broker, Dumb Consumer")**: Broker bertanggung jawab melacak pesan mana yang sudah dibaca, mana yang gagal (_unacknowledged_), mengatur prioritas antrean (_priority queues_), dan menghapus pesan setelah di-ACK.
-   **Latensi Sangat Rendah (< 1 ms)**: Untuk beban kerja pemrosesan tugas asinkron (_task worker queues_), RabbitMQ mengirimkan pesan langsung ke consumer yang sedang standby tanpa delay polling.
-   **Kontrol Pesan Per-Item**: Consumer dapat melakukan `ACK` (sukses), `NACK` (gagal, coba lagi), atau menolak pesan ke Dead Letter Queue (DLQ) untuk analisis bug tanpa memacetkan pesan lain di antrean.

## 4. Cara Kerjanya (Diagram Alur Topic Exchange)

```
                            +--------------------+
                            |   Order Service    |
                            |     (Producer)     |
                            +--------------------+
                                      |
                      Publish: "order.urgent.creditcard"
                                      v
                            +--------------------+
                            |   Topic Exchange   |
                            |   (amq.topic)      |
                            +--------------------+
                                 /    |     \
               Binding:         /     |      \      Binding:
           "order.urgent.*"    /      |       \   "order.*.creditcard"
                              /       |        \
                             v        |         v
                      +------------+  |  +------------+
                      | SMS Queue  |  |  |Fraud Queue |
                      +------------+  |  +------------+
                            |         |         |
                            v         |         v
                       [SMS Worker]   |   [Fraud Worker]
                                      |
                           Binding: "order.#"
                                      |
                                      v
                               +--------------+
                               |Warehouse Que |
                               +--------------+
                                      |
                                      v
                              [Warehouse Worker]
```

## 5. Cuplikan Logika Inti (AMQP Consumer dengan Explicit ACK di Go)

Berikut implementasi consumer RabbitMQ di Go yang mendengarkan antrean, memproses event, dan mengirimkan konfirmasi `ACK` atau `NACK` secara manual:

```go
package main

import (
	"context"
	"log"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
	// 1. Hubungkan koneksi TCP ke server RabbitMQ
	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
	if err != nil {
		log.Fatalf("Gagal terhubung ke RabbitMQ: %v", err)
	}
	defer conn.Close()

	ch, err := conn.Channel()
	if err != nil {
		log.Fatalf("Gagal membuka channel AMQP: %v", err)
	}
	defer ch.Close()

	// 2. Terapkan QoS (Prefetch Count): Batasi maksimal 10 pesan yang belum di-ACK per worker
	//    PENTING agar satu worker tidak rakus mengambil semua pesan lalu kehabisan memori
	err = ch.Qos(
		10,    // prefetch count
		0,     // prefetch size
		false, // global
	)
	if err != nil {
		log.Fatalf("Gagal mengatur QoS: %v", err)
	}

	// 3. Deklarasikan Queue durable (pesan aman tersimpan di disk jika broker restart)
	q, err := ch.QueueDeclare(
		"order_processing_queue", // nama queue
		true,                     // durable
		false,                    // auto-delete
		false,                    // exclusive
		false,                    // no-wait
		nil,                      // arguments (e.g. DLX)
	)
	if err != nil {
		log.Fatalf("Gagal mendeklarasikan queue: %v", err)
	}

	// 4. Daftarkan consumer dengan autoAck = false (wajib manual ACK)
	msgs, err := ch.Consume(
		q.Name, // queue
		"",     // consumer tag
		false,  // auto-ack (FALSE = konfirmasi manual)
		false,  // exclusive
		false,  // no-local
		false,  // no-wait
		nil,    // args
	)
	if err != nil {
		log.Fatalf("Gagal mendaftarkan consumer: %v", err)
	}

	log.Println("[*] Menunggu event pesanan di RabbitMQ. Tekan CTRL+C untuk keluar.")

	for msg := range msgs {
		log.Printf("[Worker] Menerima pesan: %s (RoutingKey: %s)", string(msg.Body), msg.RoutingKey)

		// Simulasi pemrosesan bisnis
		success := processOrder(msg.Body)

		if success {
			// Berhasil: Kirim ACK agar pesan dihapus dari queue RabbitMQ
			_ = msg.Ack(false)
			log.Println("[Worker] Pesan berhasil diproses dan di-ACK")
		} else {
			// Gagal: Kirim NACK dengan requeue=false (arahkan ke Dead Letter Queue)
			_ = msg.Nack(false, false)
			log.Println("[Worker] Pemrosesan gagal! Pesan di-NACK dan dialihkan ke DLQ")
		}
	}
}

func processOrder(payload []byte) bool {
	// Logika database atau integrasi eksternal
	time.Sleep(50 * time.Millisecond)
	return true
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Fleksibilitas Routing Luar Biasa**: Mendukung skenario pertukaran Direct, Fanout, Topic, dan Header tanpa kode kustom di consumer.
-   **Granular Message Control**: Setiap pesan memiliki lifecycle sendiri; dapat di-ACK, di-NACK, atau di-requeue satu per satu.
-   **Fitur Queue Bawaan**: Memiliki dukungan native untuk _Message TTL_, _Priority Queues_, _Dead Letter Exchanges_, dan _Delayed Messages_.

### Kekurangan

-   **Bukan Penyimpan Log Permanen**: Setelah pesan di-ACK, pesan langsung dihapus dari disk/RAM; tidak bisa diputar ulang (_replay_) seperti di Apache Kafka.
-   **Throughput Lebih Rendah dibanding Kafka**: Karena broker harus mengelola status tracking ACK per pesan, throughput mentah biasanya berada di kisaran puluhan ribu msg/detik (Kafka bisa jutaan msg/detik).
-   **Sensitif Terhadap RAM Penuh**: Jika antrean menumpuk jutaan pesan yang lambat dikonsumsi, RabbitMQ akan membunyikan _Memory Alarm_ dan memblokir seluruh koneksi publisher.

## 7. Kapan Harus Pakai RabbitMQ vs Kafka

### Pilih RabbitMQ Jika:

-   Kebutuhan utama adalah **Task Queue / Background Job Processing** (misal: antrean render video, pengiriman email/SMS, verifikasi transaksi pembayaran).
-   Membutuhkan routing pesan yang rumit dan dinamis antar banyak consumer.
-   Membutuhkan mekanisme _acknowledgment_ per pesan dan fitur prioritas antrean.

### Pilih Apache Kafka Jika:

-   Kebutuhan utama adalah **Event Streaming Logs / Big Data Ingestion** (misal: clickstream analytics, IoT telemetry, audit log yang harus disimpan berbulan-bulan).
-   Membutuhkan kemampuan _event replay_ (memutar ulang histori pesan dari awal).
-   Throughput penulisan sangat ekstrem (ratusan ribu hingga jutaan event per detik).

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **`rabbitmq_queue_messages_ready` (Gauge)**: Jumlah pesan di antrean yang siap diambil consumer. Jika grafik ini terus menanjak, jumlah worker consumer harus segera di-scale out.
-   **`rabbitmq_queue_messages_unacknowledged` (Gauge)**: Jumlah pesan yang sedang dipegang oleh worker tapi belum di-ACK. Jika angka ini tinggi, ada worker yang hang/macet.
-   **`rabbitmq_node_mem_used` vs `mem_limit` (Gauge)**: Utilisasi RAM server RabbitMQ. Wajib diwaspadai jika mendekati threshold _memory alarm_.
-   **`rabbitmq_channel_messages_ack_rate` (Counter/Rate)**: Laju keberhasilan konsumsi pesan per detik.

## 9. Studi Kasus Industri Nyata

-   **Reddit**: Menggunakan RabbitMQ secara masif sebagai antrean pemrosesan asynchronous untuk mendistribusikan vote, kalkulasi karma pengguna, serta moderasi komentar spam sebelum disimpan ke basis data permanen.
-   **Zomato**: Layanan pesan antar makanan terkemuka yang menggunakan RabbitMQ untuk mengorkestrasi alur pesanan dari pelanggan, restoran, hingga alokasi kurir terdekat dengan routing key geografis.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Lupa Mengatur `Prefetch Count` (QoS)**: Secara default di banyak driver AMQP, RabbitMQ akan menyuntikkan seluruh pesan di antrean ke satu worker pertama yang terhubung. Worker tersebut langsung kehabisan memori (_OOM_), sementara worker lain yang menganggur tidak mendapatkan bagian kerja.  
   _Solusi_: Selalu panggil `ch.Qos(prefetchCount, 0, false)` (misal nilai 10 hingga 50).
2. **Koneksi AMQP dibuat di setiap HTTP Request**: Membuat koneksi TCP baru (`amqp.Dial`) setiap kali ada request REST API masuk. Handshake AMQP sangat mahal bagi CPU RabbitMQ.  
   _Solusi_: Buat satu koneksi singleton jangka panjang (_long-lived connection_), dan buka/tutup `Channel` ringan di dalamnya.
3. **Queue Menumpuk Menjadi Monster Disk**: Producer memompa pesan terus menerus sementara consumer mati selama berhari-hari. Jutaan pesan di antrean akan memaksa RabbitMQ melakukan _paging to disk_, menurunkan throughput broker hingga 90%.

## 11. Rangkuman Singkat

RabbitMQ adalah message broker AMQP tangguh yang unggul dalam routing fleksibel, latensi rendah, dan kontrol pesan per-item melalui konfirmasi eksplisit (ACK/NACK). RabbitMQ adalah pilihan utama untuk sistem antrean tugas asinkron dan komunikasi decoupled antar microservices yang menuntut jaminan pengiriman presisi.
