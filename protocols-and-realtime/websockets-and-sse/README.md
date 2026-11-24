# WebSockets & Server-Sent Events (SSE)

## 1. Skenario Sistem Nyata

Pada sistem pelacakan armada kurir makanan (_live driver tracking_) berskala 100.000 driver aktif konkuren pada aplikasi seluler seperti Grab atau Gojek, Ada 100.000 pengguna aktif yang sedang memantau posisi motor kurir di peta ponsel mereka secara bersamaan.

Jika sistem ini dibangun menggunakan teknik tradisional **HTTP Short-Polling** (aplikasi ponsel menjalankan `setInterval` untuk memanggil `GET /api/v1/driver/location` setiap 2 detik):

-   Server backend akan dibombardir oleh **50.000 request per detik** secara terus-menerus!
-   90% dari panggilan tersebut menghasilkan data yang belum berubah (_redundant empty reads_), tetapi server tetap harus membayar mahal: alokasi socket, negosiasi TLS, header HTTP 800-byte, dan query database.
-   Baterai ponsel pengguna cepat panas dan kuota internet cepat terkuras.
-   Jika kurir tiba-tiba berbelok di detik ke-0.5, pengguna baru mengetahui lokasinya 1.5 detik kemudian (_polling delay_).

**Solusi dengan WebSockets atau SSE (Server-Sent Events):**
Koneksi TCP persisten dibuka satu kali di awal. Saat posisi GPS kurir berubah di backend, server secara proaktif **mendorong (_push_) data koordinat baru** ke ponsel pengguna dalam hitungan milidetik melalui koneksi yang sudah terbuka. Tidak ada lagi puluhan ribu request polling sia-sia, overhead header terpangkas dari 800 bytes menjadi hanya 2-6 bytes per frame, dan pergerakan kurir di layar ponsel pengguna terlihat mulus secara _real-time_.

## 2. Apa Itu dan Buat Apa?

WebSockets dan Server-Sent Events (SSE) adalah dua teknologi protokol web yang dirancang untuk komunikasi data real-time berlatensi rendah antara client (browser / aplikasi mobile) dan backend server.

### 1. WebSockets (`ws://` atau `wss://`)

-   **Dua Arah (_Full-Duplex / Bidirectional_)**: Client dan server dapat saling mengirim data secara bersamaan kapan saja secara independen.
-   **Cara Kerja**: Dimulai dengan jabat tangan HTTP standar (_HTTP Upgrade Request_), lalu server membalas dengan status **`101 Switching Protocols`**. Setelah itu, koneksi beralih dari protokol HTTP menjadi protokol TCP murni berbasis framing biner ringan.
-   **Karakteristik**: Sangat efisien untuk pertukaran data dua arah yang intensif (chat multi-user, game online multiplayer, collaborative document editing).

### 2. Server-Sent Events (SSE)

-   **Satu Arah (_Unidirectional: Server-to-Client_)**: Hanya server yang mendorong data ke client secara terus-menerus. Client tidak bisa mengirim data balik melalui saluran ini (client menggunakan POST HTTP biasa jika ingin mengirim data).
-   **Cara Kerja**: Berjalan di atas protokol HTTP standar biasa dengan header khusus: **`Content-Type: text/event-stream`** dan koneksi dibiarkan tetap terbuka (_keep-alive_).
-   **Karakteristik**: Didukung secara native oleh browser via API `EventSource`, memiliki fitur _auto-reconnect_ otomatis jika koneksi putus, dan sangat ramah firewall/proxy HTTP. Cocok untuk live ticker saham, notifikasi feed, dan streaming token teks AI (seperti respon ChatGPT).

## 3. Kenapa Harus Pakai Ini ?

### 1. Eliminasi Jutaan Request Polling Sia-Sia

Short polling adalah musuh utama sistem high-throughput karena membuang-buang bandwidth jaringan dan CPU thread server. Dengan koneksi persisten (WebSocket/SSE), beban server berubah dari model _pull_ (menjawab 50.000 request polling per detik) menjadi model _push_ (hanya mengirim data saat ada perubahan state riil).

### 2. Efisiensi Ukuran Frame Jaringan

Setiap request HTTP/1.1 membawa header teks (Cookie, User-Agent, Authorization, dsb.) berukuran 500 hingga 1.500 bytes. Pada WebSocket, setelah handshake pertama selesai, setiap pesan hanya dibungkus oleh frame header biner seringkas **2 hingga 6 bytes**. Ini memangkas konsumsi bandwidth egress cloud hingga lebih dari 90%.

## 4. Cara Kerjanya (Perbandingan Polling vs SSE vs WebSockets)

```
[ 1. HTTP Polling ]                [ 2. Server-Sent Events (SSE) ]        [ 3. WebSockets ]

Client            Server           Client               Server            Client            Server
  |                 |                |                    |                 |                 |
  |-- GET /loc ---->|                |-- GET /events ---->|                 |-- HTTP Upgrade->|
  |<- Data (200) ---|                |   (text/event-stream)                |<- 101 Switch ---|
  |   (Jeda 2 dtk)  |                |<- Data Stream 1 ---|                 |                 |
  |-- GET /loc ---->|                |<- Data Stream 2 ---|                 |<-- Push Data ---|
  |<- "No Change" --|                |<- Data Stream 3 ---|                 |--- Kirim Data ->|
  v                 v                v                    v                 v                 v
 (Banyak Request Kosong)           (Server Push Satu Arah)                (Full-Duplex Dua Arah)
```

## 5. Cuplikan Logika Inti

### 1. Implementasi Server-Sent Events (SSE) di Go

```go
func handleSSE(w http.ResponseWriter, r *http.Request) {
    // 1. Set header khusus SSE agar koneksi tidak ditutup
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("Access-Control-Allow-Origin", "*")

    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "Streaming unsupported", http.StatusInternalServerError)
        return
    }

    notifyChannel := make(chan string)
    // Daftarkan channel client ke subscriber manager...

    for {
        select {
        case msg := <-notifyChannel:
            // 2. Format standar SSE: "data: <payload>

"
            fmt.Fprintf(w, "data: %s

", msg)
            flusher.Flush() // Dorong data seketika tanpa buffering
        case <-r.Context().Done():
            // Client menutup koneksi (tutup tab / sinyal hilang)
            return
        }
    }
}
```

### 2. WebSocket Cluster dengan Redis Pub/Sub Backplane di Go

```go
func handleWebSocketCluster(w http.ResponseWriter, r *http.Request, rdb *redis.Client, userID string) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		return
	}
	defer conn.Close()

	pubsub := rdb.Subscribe(r.Context(), "user:"+userID)
	defer pubsub.Close()

	// Goroutine 1: Terima pesan dari Redis Pub/Sub dan kirim ke client
	go func() {
		for msg := range pubsub.Channel() {
			_ = conn.WriteMessage(websocket.TextMessage, []byte(msg.Payload))
		}
	}()

	// Goroutine 2: Baca pesan dari client dan publikasikan ke broker Redis
	for {
		_, message, err := conn.ReadMessage()
		if err != nil {
			break
		}
		targetUser := parseTargetUser(message)
		rdb.Publish(r.Context(), "user:"+targetUser, message)
	}
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Latensi Sub-Milidetik**: Data sampai ke pengguna detik itu juga begitu event terjadi di backend (_zero polling delay_).
-   **Penghematan Bandwidth Masif**: Mengeliminasi ribuan request handshake HTTP berulang dan overhead header.
-   **Koneksi Efisien**: Satu koneksi TCP persisten dapat menyalurkan ribuan pesan tanpa membuka socket baru.

### Kekurangan

-   **Stateful Server Footprint**: Berbeda dengan REST yang bersifat stateless, setiap koneksi WebSocket/SSE menahan memori RAM (sekitar 10KB - 50KB per koneksi) dan satu _file descriptor_ OS di server selama koneksi terbuka.
-   **Kompleksitas Scaling Horizontal**: Saat server di-scale menjadi 10 pod, Pod A tidak tahu koneksi WebSocket pengguna yang terhubung di Pod B. Membutuhkan _Message Backplane_ (seperti Redis Pub/Sub, NATS, atau Kafka) untuk menjembatani pesan antar-pod.
-   **Masalah Load Balancer Timeout**: Load balancer cloud (seperti AWS ALB) otomatis memutus koneksi TCP jika tidak ada data yang lewat selama 60 detik. Membutuhkan mekanisme _Heartbeat (Ping/Pong)_ berkala.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Pakai WebSockets

-   Aplikasi interaktif dua arah yang sangat intensif: Game multiplayer real-time, obrolan chat grup (_instant messaging_), aplikasi papan tulis kolaboratif (_Figma/Miro-like canvas_), platform trading saham dengan order book interaktif.

### Kapan Pakai Server-Sent Events (SSE)

-   Kebutuhan data satu arah dari server ke client: Live sports score ticker, feed notifikasi linimasa, dashboard monitoring metrik real-time, dan streaming token bahasa AI (LLM generation).

### Kapan Cukup Pakai HTTP REST / Polling Biasa

-   Data yang sangat jarang berubah (misal memeriksa status pembayaran setiap 30 detik pada halaman checkout statis).
-   Sistem dengan kebutuhan stateless murni di mana kompleksitas memelihara koneksi socket tidak sebanding dengan manfaatnya.

## 8. Metrik Kunci & Observabilitas

Untuk memantau stabilitas server WebSocket & SSE pada Grafana / Datadog:

-   **`active_websocket_connections_count`**: Jumlah koneksi soket aktif yang sedang terhubung ke server pod.
-   **`websocket_connection_churn_rate`**: Jumlah koneksi baru (_connect_) vs koneksi putus (_disconnect_) per menit; churn yang tinggi menandakan sinyal jaringan klien tidak stabil atau gateway timeout.
-   **`websocket_messages_in_per_sec` & `out_per_sec`**: Throughput frame pesan yang masuk dan keluar dari cluster socket.
-   **`process_open_file_descriptors`**: Jumlah file descriptor yang sedang digunakan oleh proses server (waspadai batas `ulimit`).
-   **`memory_usage_per_active_connection_bytes`**: Rata-rata konsumsi RAM heap per koneksi terbuka.

## 9. Studi Kasus Nyata di Industri

### Penanganan 2 Juta Koneksi Persisten Bersamaan pada Platform Live Chat & Ticker Finansial

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah platform trading kripto melayani 2 juta pengguna aktif yang membutuhkan pembaruan order book secara instan. Awalnya sistem menggunakan polling HTTP setiap 1 detik. Trafik polling menghasilkan 2 juta request/detik ke backend, membanjiri CPU reverse proxy dengan jutaan TLS handshake berulang dan menyebabkan latensi lonjakan harga tertunda hingga 3-5 detik, memicu kerugian finansial akibat slippage harga bagi trader.
- **Implementasi Solusi Industri (The Architectural Solution)**: Tim memisahkan arsitektur koneksi: untuk pembaruan order book dan harga publik satu arah, diterapkan *Server-Sent Events (SSE)* melalui CDN edge yang mendukung HTTP/2 streaming; untuk transaksi order dan chat interaktif dua arah, digunakan gateway *WebSocket cluster* berbasis Go. Diterapkan buffer pool per koneksi untuk menekan overhead RAM, mekanisme heartbeat ping-pong setiap 25 detik agar koneksi tidak diputus load balancer, dan Redis Pub/Sub cluster sebagai backplane distribusi pesan lintas node server WebSocket.
- **Dampak & Hasil Sistem (The Outcome)**: Latensi pengiriman pembaruan harga terpangkas dari 3.000 ms menjadi di bawah 15 ms secara real-time. Beban CPU di API Gateway anjlok sebesar 80% karena hilangnya jutaan request polling per detik. Utilisasi memori per koneksi ditekan hingga hanya ~8 KB per soket, memungkinkan kluster melayani 2 juta koneksi persisten hanya dengan 20 node instance.

## 10. Fakta Menarik & Gotchas Production

-   **The Linux File Descriptor (uLimit) Wall**: Di sistem operasi Linux, _everything is a file_—termasuk sebuah koneksi TCP socket! Secara default, konfigurasi Linux membatasi satu proses hanya boleh membuka **1.024 file descriptor (`ulimit -n 1024`)**. Saat server WebSocket-mu mencapai koneksi ke-1.025, aplikasi akan crash seketika dengan error fatal **`socket: too many open files`**!  
    _Solusi Wajib:_ Naikkan batas `ulimit -n` di server Linux atau container manifest menjadi minimal `65535` atau `1000000`.
-   **The AWS ALB 60-Second Idle Timeout Trap**: AWS Application Load Balancer (ALB) memiliki konfigurasi _Idle Timeout_ default sebesar 60 detik. Jika room chat sedang sepi dan tidak ada pengguna yang mengirim pesan selama 60 detik, ALB akan memutus koneksi TCP secara sepihak!  
    _Solusi Wajib:_ Terapkan mekanisme **Heartbeat / Ping-Pong**: client atau server wajib mengirimkan frame ping ringan setiap 20-30 detik untuk menjaga koneksi tetap dianggap aktif oleh load balancer (_keep-alive_).

## 11. Rangkuman Singkat

- **Pilih Sesuai Kebutuhan Arah Data**: Gunakan WebSocket untuk komunikasi dua arah dua arah (*bidirectional*) latensi rendah (chat, game); gunakan Server-Sent Events (SSE) untuk streaming satu arah dari server ke klien (notifikasi, AI token streaming, price feeds).
- **Perhatikan Limit File Descriptor (`ulimit`)**: Setiap koneksi socket adalah file descriptor di OS Linux; selalu naikkan limit sistem (`ulimit -n 65535+`) agar server tidak kehabisan soket saat menerima ribuan koneksi.
- **Wajib Heartbeat Ping-Pong**: Reverse proxy dan cloud load balancer memutus soket idle setelah 60 detik; implementasikan frame ping-pong berkala untuk menjaga koneksi persisten tetap hidup.
- **Gunakan Pub/Sub Backplane**: Server socket bersifat stateful; gunakan broker pesan (seperti Redis Pub/Sub) untuk menyebarkan pesan ke pengguna yang terhubung di instans server yang berbeda.
