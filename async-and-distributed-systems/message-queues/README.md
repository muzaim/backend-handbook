# Message Queues

## 1. Skenario Sistem Nyata

Pada platform streaming media bervolume tinggi, pengguna mengunggah file rekaman video berukuran 1.5 GB melalui API backend (`POST /api/v1/videos/upload`).

Setelah file tersimpan, sistem harus memproses video tersebut: melakukan kompresi dan *transcoding* ke 4 resolusi berbeda (1080p, 720p, 480p, 360p) serta mengekstrak 10 thumbnail gambar. Proses komputasi CPU dan GPU yang intensif ini memakan waktu sekitar 3 hingga 5 menit.

Jika proses komputasi berat ini dijalankan secara sinkron langsung di dalam alur HTTP request:
- Web server terpaksa menahan koneksi soket HTTP pengguna selama 5 menit penuh.
- Browser atau aplikasi mobile pengguna akan mengalami *Gateway Timeout*, dan worker thread web server habis terkunci menunggu proses encoding selesai.
- Server web tidak lagi sanggup menerima request dari pengguna lain (*thread starvation*).

**Solusi dengan Message Queue (misal: RabbitMQ / AWS SQS / Celery):**
Saat file video selesai diupload ke object storage (S3), API server hanya memasukkan sebuah pesan pekerjaan ringan (*Job Payload*: `{"job_id": 901, "video_id": "vid-882", "s3_url": "..."}`) ke dalam Message Queue, lalu langsung mengembalikan respon `HTTP 202 Accepted: Video sedang diproses` kepada pengguna dalam waktu 150 milidetik. Thread web server langsung bebas melayani pengguna lain. Di belakang layar, armada *Worker Pool* khusus mengambil job dari antrean satu per satu secara asinkron sesuai kapasitas CPU yang tersedia.

## 2. Apa Itu dan Buat Apa?

Message Queue (Antrean Pesan) adalah komponen arsitektur yang menyediakan media penyimpanan sementara berbasis antrean (biasanya *FIFO - First-In, First-Out*) untuk memfasilitasi komunikasi **asynchronous (tidak langsung)** antar aplikasi atau service.

Pola intinya adalah **Point-to-Point (Competing Consumers)**:
- Satu atau lebih Producer memasukkan pesan (*message/job*) ke dalam antrean.
- Satu pesan yang masuk ke antrean **hanya akan diambil dan diproses oleh TEPAT SATU Consumer**.
- Setelah Consumer selesai memproses pesan, Consumer mengirimkan sinyal konfirmasi (**ACK - Acknowledgment**), dan pesan tersebut dihapus dari antrean. Jika Consumer mati saat memproses, pesan dikembalikan ke antrean agar diambil worker lain.

Contoh software populer: RabbitMQ, AWS SQS, Apache ActiveMQ, Redis Streams/BullMQ, Celery.

## 3. Kenapa Harus Pakai Ini di High-Throughput?

### 1. Pemuatan Puncak (*Peak Shaving / Load Leveling*)
Bayangkan toko online diserbu 50.000 pesanan per detik saat promo tengah malam. Database inventory dan payment partner hanya sanggup memproses 2.000 transaksi per detik.
- **Tanpa Message Queue**: 48.000 request sisanya mengalami timeout, database crash, dan transaksi gagal massal.
- **Dengan Message Queue**: 50.000 order langsung ditampung dengan aman di dalam queue dalam waktu 5 milidetik. Worker di belakang memprosesnya secara konstan di kecepatan stabil 2.000 per detik hingga antrean habis. Sistem tidak pernah tumbang.

### 2. Memisahkan Proses Cepat vs Proses Berat (*Decoupling*)
Saat pengguna mengupload video baru, API tidak boleh menahan koneksi HTTP selama 5 menit menunggu video selesai dikompresi ke resolusi 1080p, 720p, dan 480p. API cukup melempar pesan job ke queue, merespons `HTTP 202 Accepted: Video Sedang Diproses`, dan membebaskan thread web server.

## 4. Cara Kerjanya (Diagram Alur)

```
+---------------+
|  Web API Svc  | (Producer)
+---------------+
        |
        | 1. Publish Job: { order_id: 101, user: "andi" }
        v
+-------------------------------------------------------+
|                     Message Queue                     |
|                                                       |
|  [ Job 103 ] -> [ Job 102 ] -> [ Job 101 ]            |
+-------------------------------------------------------+
       |                           |
       | 2a. Ambil Job 101         | 2b. Ambil Job 102
       v                           v
+---------------+           +---------------+
|   Worker 1    |           |   Worker 2    | (Consumers Pool)
| (Memproses)   |           | (Memproses)   |
+---------------+           +---------------+
       |                           |
       | 3a. ACK (Selesai!)        | 3b. ACK (Selesai!)
       v                           v
    (Hapus Job 101)             (Hapus Job 102)
```

## 5. Cuplikan Logika Inti (Consumer Loop & DLQ di Go)

Implementasi pola Consumer Loop dengan penanganan ACK, retry bertahap, dan pemindahan ke Dead Letter Queue (DLQ) di Go:

```go
package main

import (
	"context"
	"log"
	"time"
)

type Message struct {
	ID        string
	Payload   []byte
	Attempt   int
}

// StartQueueWorker menjalankan loop konsumsi pesan dengan retry dan DLQ
func StartQueueWorker(ctx context.Context, queue <-chan Message, dlq chan<- Message) {
	for {
		select {
		case msg := <-queue:
			err := processVideoEncoding(msg.Payload)
			if err != nil {
				msg.Attempt++
				log.Printf("Gagal memproses pesan %s (percobaan ke-%d): %v", msg.ID, msg.Attempt, err)

				if msg.Attempt >= 3 {
					log.Printf("Pesan %s dialihkan ke Dead Letter Queue (DLQ)", msg.ID)
					dlq <- msg
				} else {
					// Masukkan kembali ke antrean utama untuk dicoba lagi
					go func(m Message) {
						time.Sleep(2 * time.Second)
						queue <- m
					}(msg)
				}
				continue
			}

			// Sukses diproses (ACK)
			log.Printf("Pesan %s berhasil diproses", msg.ID)

		case <-ctx.Done():
			return
		}
	}
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Ketahanan Luar Biasa (*Fault Isolation*)**: Jika seluruh worker service mati, pesan tetap tersimpan aman di antrean disk (*persisted*) dan tidak akan hilang. Ketika worker dinyalakan kembali, pemrosesan lanjut secara otomatis.
- **Elastisitas Scaling**: Kita bisa menambah jumlah worker dari 2 menjadi 50 mesin kapan saja saat antrean menumpuk tanpa mengubah kode API.
- **Sistem Menjadi Decoupled**: Tim yang mengelola Web API tidak perlu tahu bahasa pemrograman atau arsitektur worker pengolah data di belakangnya.

### Kekurangan
- **Eventual Consistency**: Data tidak terupdate secara instan. Pengguna mungkin harus menunggu beberapa detik sebelum melihat status pesanannya berubah menjadi "Selesai".
- **Overhead Operasional**: Menambah infrastruktur baru yang harus di-maintain, di-cluster, dan dipantau ketersediaan disk-nya.
- **Kompleksitas Debugging**: Melacak error (*distributed tracing*) antar proses asynchronous jauh lebih menantang daripada membaca log HTTP sinkron biasa.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Tugas komputasi berat: Pengiriman email massal, notifikasi push, generate laporan PDF/Excel, encoding video, pemrosesan AI/ML.
- Redam lonjakan trafik transaksi checkout di jam-jam sibuk (*buffer queue*).
- Komunikasi antar microservice yang tidak membutuhkan respon saat itu juga.

### Kapan Jangan Pakai
- Operasi sinkron murni yang membutuhkan respon detik itu juga (misal: verifikasi password saat login pengguna; pengguna tidak bisa disuruh menunggu 1 menit hanya untuk tahu password-nya salah).
- Volume trafik sangat kecil di mana database tabel biasa dengan status `status: PENDING` sudah lebih dari cukup.

## 8. Metrik Kunci & Observabilitas

Untuk memantau performa message queue pada dashboard monitoring (Grafana / Datadog):

- **`queue_messages_visible` / `queue_depth`**: Jumlah total pesan yang sedang menunggu untuk diambil worker di antrean.
- **`queue_consumer_lag_seconds`**: Selisih waktu antara kapan pesan dipublish oleh producer hingga pesan tersebut mulai diproses oleh consumer.
- **`queue_messages_unacknowledged`**: Jumlah pesan yang sedang aktif dikerjakan oleh worker dan belum menerima ACK/NACK.
- **`dlq_messages_total`**: Lonjakan jumlah pesan yang masuk ke Dead Letter Queue, menandakan adanya bad payload atau bug di kode consumer.
- **`consumer_throughput_messages_per_sec`**: Kecepatan pemrosesan worker fleet dibandingkan dengan tingkat kedatangan pesan producer.

## 9. Studi Kasus Nyata di Industri

- **DoorDash Delivery Dispatching**: DoorDash menggunakan RabbitMQ dan Celery untuk menangani ribuan pesanan makanan per detik. Saat pengguna memesan, order dimasukkan ke antrean dispatch. Kumpulan worker khusus menghitung rute kurir dan estimasi pengiriman secara paralel di belakang layar tanpa memperlambat interaksi pengguna di aplikasi seluler.
- **Uber Driver Webhook Ingestion**: Uber memanfaatkan AWS SQS untuk menampung jutaan pembaruan lokasi dan webhook status perjalanan dari aplikasi pengemudi. Antrean ini meratakan lonjakan beban trafik liar (*burst traffic*) pada jam sibuk pulang kantor, sehingga backend core Uber tetap stabil.

## 10. Fakta Menarik & Gotchas Production

- **The Poison Pill Message**: Apa yang terjadi jika ada satu pesan yang rusak (*corrupt data*) masuk ke antrean? Worker 1 mengambil pesan itu, mengalami *panic/crash*, dan mati. Queue mendeteksi worker mati, lalu mengembalikan pesan ke antrean. Worker 2 mengambil pesan yang sama, dan ikut crash! Ini terus berulang hingga seluruh worker mati (*poison pill cascade*).  
  *Solusi Wajib:* Terapkan **DLQ (Dead Letter Queue)**. Jika sebuah pesan gagal diproses sebanyak 3 kali berturut-turut, pindahkan pesan tersebut ke antrean khusus DLQ untuk diinvestigasi oleh engineer tanpa menyumbat antrean utama.
- **Consumer Lag Alert**: Metrik terpenting yang wajib dipantau di dashboard bukan CPU atau memori queue, melainkan **Consumer Lag** (selisih antara jumlah pesan yang masuk vs jumlah pesan yang selesai diproses). Jika consumer lag terus menanjak tajam, itu sinyal merah bahwa sistemmu butuh autoscaling worker!
