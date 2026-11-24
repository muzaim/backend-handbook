# WebSockets & Server-Sent Events (SSE)

## 1. Skenario Sistem Nyata

Pada sistem pelacakan armada kurir makanan (*live driver tracking*) berskala 100.000 driver aktif konkuren pada aplikasi seluler seperti Grab atau Gojek, Ada 100.000 pengguna aktif yang sedang memantau posisi motor kurir di peta ponsel mereka secara bersamaan.

Jika sistem ini dibangun menggunakan teknik tradisional **HTTP Short-Polling** (aplikasi ponsel menjalankan `setInterval` untuk memanggil `GET /api/v1/driver/location` setiap 2 detik):
- Server backend akan dibombardir oleh **50.000 request per detik** secara terus-menerus!
- 90% dari panggilan tersebut menghasilkan data yang belum berubah (*redundant empty reads*), tetapi server tetap harus membayar mahal: alokasi socket, negosiasi TLS, header HTTP 800-byte, dan query database.
- Baterai ponsel pengguna cepat panas dan kuota internet cepat terkuras.
- Jika kurir tiba-tiba berbelok di detik ke-0.5, pengguna baru mengetahui lokasinya 1.5 detik kemudian (*polling delay*).

**Solusi dengan WebSockets atau SSE (Server-Sent Events):**
Koneksi TCP persisten dibuka satu kali di awal. Saat posisi GPS kurir berubah di backend, server secara proaktif **mendorong (*push*) data koordinat baru** ke ponsel pengguna dalam hitungan milidetik melalui koneksi yang sudah terbuka. Tidak ada lagi puluhan ribu request polling sia-sia, overhead header terpangkas dari 800 bytes menjadi hanya 2-6 bytes per frame, dan pergerakan kurir di layar ponsel pengguna terlihat mulus secara *real-time*.

## 2. Apa Itu dan Buat Apa?

WebSockets dan Server-Sent Events (SSE) adalah dua teknologi protokol web yang dirancang untuk komunikasi data real-time berlatensi rendah antara client (browser / aplikasi mobile) dan backend server.

### 1. WebSockets (`ws://` atau `wss://`)
- **Dua Arah (*Full-Duplex / Bidirectional*)**: Client dan server dapat saling mengirim data secara bersamaan kapan saja secara independen.
- **Cara Kerja**: Dimulai dengan jabat tangan HTTP standar (*HTTP Upgrade Request*), lalu server membalas dengan status **`101 Switching Protocols`**. Setelah itu, koneksi beralih dari protokol HTTP menjadi protokol TCP murni berbasis framing biner ringan.
- **Karakteristik**: Sangat efisien untuk pertukaran data dua arah yang intensif (chat multi-user, game online multiplayer, collaborative document editing).

### 2. Server-Sent Events (SSE)
- **Satu Arah (*Unidirectional: Server-to-Client*)**: Hanya server yang mendorong data ke client secara terus-menerus. Client tidak bisa mengirim data balik melalui saluran ini (client menggunakan POST HTTP biasa jika ingin mengirim data).
- **Cara Kerja**: Berjalan di atas protokol HTTP standar biasa dengan header khusus: **`Content-Type: text/event-stream`** dan koneksi dibiarkan tetap terbuka (*keep-alive*).
- **Karakteristik**: Didukung secara native oleh browser via API `EventSource`, memiliki fitur *auto-reconnect* otomatis jika koneksi putus, dan sangat ramah firewall/proxy HTTP. Cocok untuk live ticker saham, notifikasi feed, dan streaming token teks AI (seperti respon ChatGPT).

## 3. Kenapa Harus Pakai Ini di High-Throughput?

### 1. Eliminasi Jutaan Request Polling Sia-Sia
Short polling adalah musuh utama sistem high-throughput karena membuang-buang bandwidth jaringan dan CPU thread server. Dengan koneksi persisten (WebSocket/SSE), beban server berubah dari model *pull* (menjawab 50.000 request polling per detik) menjadi model *push* (hanya mengirim data saat ada perubahan state riil).

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
- **Latensi Sub-Milidetik**: Data sampai ke pengguna detik itu juga begitu event terjadi di backend (*zero polling delay*).
- **Penghematan Bandwidth Masif**: Mengeliminasi ribuan request handshake HTTP berulang dan overhead header.
- **Koneksi Efisien**: Satu koneksi TCP persisten dapat menyalurkan ribuan pesan tanpa membuka socket baru.

### Kekurangan
- **Stateful Server Footprint**: Berbeda dengan REST yang bersifat stateless, setiap koneksi WebSocket/SSE menahan memori RAM (sekitar 10KB - 50KB per koneksi) dan satu *file descriptor* OS di server selama koneksi terbuka.
- **Kompleksitas Scaling Horizontal**: Saat server di-scale menjadi 10 pod, Pod A tidak tahu koneksi WebSocket pengguna yang terhubung di Pod B. Membutuhkan *Message Backplane* (seperti Redis Pub/Sub, NATS, atau Kafka) untuk menjembatani pesan antar-pod.
- **Masalah Load Balancer Timeout**: Load balancer cloud (seperti AWS ALB) otomatis memutus koneksi TCP jika tidak ada data yang lewat selama 60 detik. Membutuhkan mekanisme *Heartbeat (Ping/Pong)* berkala.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Pakai WebSockets
- Aplikasi interaktif dua arah yang sangat intensif: Game multiplayer real-time, obrolan chat grup (*instant messaging*), aplikasi papan tulis kolaboratif (*Figma/Miro-like canvas*), platform trading saham dengan order book interaktif.

### Kapan Pakai Server-Sent Events (SSE)
- Kebutuhan data satu arah dari server ke client: Live sports score ticker, feed notifikasi linimasa, dashboard monitoring metrik real-time, dan streaming token bahasa AI (LLM generation).

### Kapan Cukup Pakai HTTP REST / Polling Biasa
- Data yang sangat jarang berubah (misal memeriksa status pembayaran setiap 30 detik pada halaman checkout statis).
- Sistem dengan kebutuhan stateless murni di mana kompleksitas memelihara koneksi socket tidak sebanding dengan manfaatnya.

## 8. Metrik Kunci & Observabilitas

Untuk memantau stabilitas server WebSocket & SSE pada Grafana / Datadog:

- **`active_websocket_connections_count`**: Jumlah koneksi soket aktif yang sedang terhubung ke server pod.
- **`websocket_connection_churn_rate`**: Jumlah koneksi baru (*connect*) vs koneksi putus (*disconnect*) per menit; churn yang tinggi menandakan sinyal jaringan klien tidak stabil atau gateway timeout.
- **`websocket_messages_in_per_sec` & `out_per_sec`**: Throughput frame pesan yang masuk dan keluar dari cluster socket.
- **`process_open_file_descriptors`**: Jumlah file descriptor yang sedang digunakan oleh proses server (waspadai batas `ulimit`).
- **`memory_usage_per_active_connection_bytes`**: Rata-rata konsumsi RAM heap per koneksi terbuka.

## 9. Studi Kasus Nyata di Industri

- **Slack & Discord Real-Time Messaging**: Slack dan Discord mengelola jutaan koneksi WebSocket persisten secara bersamaan untuk menyalurkan pesan obrolan, status kehadiran online (*presence*), dan indikator mengetik (*typing indicators*) ke jutaan pengguna secara instan dengan Redis/Elixir backplane.
- **OpenAI ChatGPT Streaming Response**: OpenAI menggunakan Server-Sent Events (SSE) untuk mengalirkan kata demi kata hasil generate model GPT secara langsung ke browser pengguna, memberikan respons instan tanpa harus menunggu model menyelesaikan seluruh paragraf teks.

## 10. Fakta Menarik & Gotchas Production

- **The Linux File Descriptor (uLimit) Wall**: Di sistem operasi Linux, *everything is a file*—termasuk sebuah koneksi TCP socket! Secara default, konfigurasi Linux membatasi satu proses hanya boleh membuka **1.024 file descriptor (`ulimit -n 1024`)**. Saat server WebSocket-mu mencapai koneksi ke-1.025, aplikasi akan crash seketika dengan error fatal **`socket: too many open files`**!  
  *Solusi Wajib:* Naikkan batas `ulimit -n` di server Linux atau container manifest menjadi minimal `65535` atau `1000000`.
- **The AWS ALB 60-Second Idle Timeout Trap**: AWS Application Load Balancer (ALB) memiliki konfigurasi *Idle Timeout* default sebesar 60 detik. Jika room chat sedang sepi dan tidak ada pengguna yang mengirim pesan selama 60 detik, ALB akan memutus koneksi TCP secara sepihak!  
  *Solusi Wajib:* Terapkan mekanisme **Heartbeat / Ping-Pong**: client atau server wajib mengirimkan frame ping ringan setiap 20-30 detik untuk menjaga koneksi tetap dianggap aktif oleh load balancer (*keep-alive*).
