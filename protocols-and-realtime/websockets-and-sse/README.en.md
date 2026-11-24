# WebSockets & Server-Sent Events (SSE)

## 1. Real-World System Scenario

In live courier fleet tracking features for ride-hailing and on-demand delivery apps (like Uber or DoorDash) supporting 100,000 active concurrent driver updates, There are 100,000 active users simultaneously watching courier motorcycle positions move on their phone screens.

If this system were built using traditional **HTTP Short-Polling** (where mobile apps run `setInterval` calling `GET /api/v1/driver/location` every 2 seconds):
- The backend server would be bombarded with **50,000 requests per second** continuously!
- 90% of those requests return unchanged data (*redundant empty reads*), yet the server still pays a heavy price: socket allocations, TLS handshakes, 800-byte HTTP header parsing, and database read queries.
- User device batteries run hot and mobile data plans are rapidly drained.
- If a driver turns at second 0.5, the user only observes the new coordinate 1.5 seconds later (*polling latency*).

**Solution with WebSockets or SSE (Server-Sent Events):**
A persistent TCP connection is established once upfront. Whenever courier GPS coordinates update on the backend, the server proactively **pushes fresh coordinates** to the user's mobile device within single-digit milliseconds through the established pipe. Tens of thousands of wasteful polling requests vanish, network header overhead drops from 800 bytes to 2-6 bytes per frame, and courier movements appear silky smooth on screen in real time.

## 2. What Is It and What Is It For?

WebSockets and Server-Sent Events (SSE) are two web protocols designed for low-latency, real-time data delivery between clients (browsers / mobile apps) and backend servers.

### 1. WebSockets (`ws://` or `wss://`)
- **Bidirectional / Full-Duplex**: Both client and server can independently send messages to each other at any moment over a single connection.
- **How It Works**: Initiated via a standard HTTP handshake (*HTTP Upgrade Request*), to which the server responds with **`101 Switching Protocols`**. The underlying TCP socket is then retained and transitions to a lightweight binary framing protocol.
- **Characteristics**: Ideal for high-frequency two-way interactions (multiplayer gaming, instant messaging rooms, collaborative canvas editing).

### 2. Server-Sent Events (SSE)
- **Unidirectional (Server-to-Client)**: Only the server pushes data streams to the client continuously. The client cannot send data back through this pipe (the client issues standard HTTP POST/PUT requests if it needs to submit data).
- **How It Works**: Operates over standard HTTP/1.1 or HTTP/2 connections with a dedicated header: **`Content-Type: text/event-stream`**, keeping the connection open indefinitely (*keep-alive*).
- **Characteristics**: Natively supported by web browsers via the `EventSource` API, includes built-in automatic reconnect capabilities, and seamlessly navigates enterprise firewalls and corporate HTTP proxies. Ideal for live stock tickers, activity feeds, and token streaming for generative AI (such as ChatGPT responses).

## 3. Why Use It in High-Throughput Systems?

### 1. Elimination of Wasteful Polling Traffic
Short polling is among the biggest throughput killers in web architectures because it burns network bandwidth and backend thread capacity on no-op round-trips. With persistent connections (WebSocket/SSE), system workload shifts from a pull model (processing 50,000 queries per second) to an event-driven push model (transmitting data strictly when state transitions occur).

### 2. Radical Reduction of Framing Overhead
Every standard HTTP/1.1 request transmits 500 to 1,500 bytes of repetitive plaintext headers (`Cookie`, `User-Agent`, `Authorization`, etc.). In WebSockets, once the initial upgrade handshake completes, subsequent messages are encapsulated in binary frames as compact as **2 to 6 bytes**, slashing egress bandwidth overhead by over 90%.

## 4. How It Works (Polling vs SSE vs WebSockets)

```
[ 1. HTTP Polling ]                [ 2. Server-Sent Events (SSE) ]        [ 3. WebSockets ]

Client            Server           Client               Server            Client            Server
  |                 |                |                    |                 |                 |
  |-- GET /loc ---->|                |-- GET /events ---->|                 |-- HTTP Upgrade->|
  |<- Data (200) ---|                |   (text/event-stream)                |<- 101 Switch ---|
  |   (Wait 2s)     |                |<- Data Stream 1 ---|                 |                 |
  |-- GET /loc ---->|                |<- Data Stream 2 ---|                 |<-- Push Data ---|
  |<- "No Change" --|                |<- Data Stream 3 ---|                 |--- Send Data -->|
  v                 v                v                    v                 v                 v
(Heavy Redundant Traffic)           (Unidirectional Server Push)           (Full-Duplex Bidirectional)
```

## 5. Core Implementation Logic

### 1. Server-Sent Events (SSE) Implementation in Go

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

### 2. WebSocket Cluster Scalability via Redis Pub/Sub in Go

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

## 6. Trade-offs (Pros and Cons)

### Pros
- **Sub-Millisecond Latency**: Messages reach destination devices as soon as events occur in the backend (*zero polling lag*).
- **Substantial Bandwidth Conservation**: Eliminates repetitive HTTP connection setups, TLS handshakes, and header overhead.
- **Connection Efficiency**: A single long-lived TCP connection transports thousands of updates without socket churn.

### Cons
- **Stateful Server Footprint**: Unlike stateless REST, each open WebSocket or SSE connection consumes operating system memory (roughly 10KB - 50KB RAM) and an open OS file descriptor for the entire session duration.
- **Horizontal Scaling Complexity**: When scaling out to multiple pods, Pod A has no direct knowledge of client sockets anchored on Pod B. Requires a centralized *Message Backplane* (such as Redis Pub/Sub, NATS, or Kafka) to route inter-pod messages.
- **Load Balancer Idle Timeouts**: Cloud infrastructure (like AWS ALB) automatically drops idle TCP sockets after 60 seconds of inactivity without traffic. Requires periodic *Heartbeat (Ping/Pong)* pinging.

## 7. When to Use vs When Not to Use

### When to Use WebSockets
- Highly interactive bidirectional applications: Real-time multiplayer gaming, group chat systems, collaborative whiteboard canvases (e.g., Figma or Miro), and high-frequency trading platforms with live order books.

### When to Use Server-Sent Events (SSE)
- Unidirectional server-to-client updates: Sports score tickers, notification center streams, real-time monitoring metric dashboards, and generative AI text token streaming (LLM completion feeds).

### When Plain REST / Polling Suffices
- Infrequently modified data (e.g., checking order payment status every 30 seconds on a static checkout confirmation page).
- Simple stateless architectures where the infrastructure complexity of socket state management outweighs the latency gains.

## 8. Key Metrics & Observability

Essential telemetry for Grafana / Datadog dashboards monitoring WebSocket and SSE fleets:

- **`active_websocket_connections_count`**: Current tally of live socket sessions anchored to each pod instance.
- **`websocket_connection_churn_rate`**: Rate of connection connects vs disconnects per minute; high churn flags client network instability or edge gateway timeouts.
- **`websocket_messages_in_per_sec` & `out_per_sec`**: Ingress and egress message frame throughput across the cluster.
- **`process_open_file_descriptors`**: Count of operating system file handles held by the runtime process (vital for catching `ulimit` exhaustion).
- **`memory_usage_per_active_connection_bytes`**: Heap allocation footprint per open socket.

## 9. Real-World Case Studies

- **Slack & Discord Real-Time Messaging**: Slack and Discord maintain millions of concurrent persistent WebSocket connections to deliver instant channel messages, typing indicators, and user presence state across distributed worker fleets backed by Redis and Elixir clusters.
- **OpenAI ChatGPT Streaming Response**: OpenAI leverages Server-Sent Events (SSE) to push generated completion tokens incrementally to browser clients, delivering instant perception of responsiveness without waiting for full response paragraph completion.

## 10. Interesting Facts & Production Gotchas

- **The Linux File Descriptor (`ulimit`) Wall**: In Unix and Linux systems, *everything is a file*—including open TCP sockets! By default, standard Linux distributions often cap per-process file descriptor limits to **1,024 (`ulimit -n 1024`)**. When your WebSocket server attempts to accept connection 1,025, the process abruptly crashes or rejects connections with the fatal error **`socket: too many open files`**!  
  *Production Fix:* Explicitly configure `ulimit -n` in systemd units or container definitions to at least `65535` or `1000000`.
- **The AWS ALB 60-Second Idle Timeout Trap**: The AWS Application Load Balancer (ALB) enforces a default *Idle Timeout* of 60 seconds. If a chat channel is quiet and neither party sends packets for 60 seconds, the ALB unceremoniously severs the TCP connection!  
  *Production Fix:* Implement an active **Heartbeat / Ping-Pong** protocol: the client or server must transmit lightweight ping frames every 20-30 seconds to keep the intermediary load balancer connection state alive.
