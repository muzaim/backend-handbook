# Service Discovery

## 1. Skenario Sistem Nyata

Pada arsitektur cloud microservices dinamis, `Order Service` bertugas memanggil `Payment Service` melalui panggilan HTTP internal.

Di lingkungan cloud modern dan kluster kontainer (seperti Kubernetes atau AWS ECS), instance pod bersifat **Ephemeral (dinamis dan mudah berganti)**:
- Setiap kali tim developer melakukan *rolling deployment* versi baru, atau saat sistem melakukan *autoscaling*, pod-pod lama dimatikan dan digantikan oleh pod baru dengan alamat IP privat internal yang acak (misal berpindah dari `10.244.1.24` ke `10.244.4.91`).
- Jika kamu menuliskan (*hardcode*) IP address statis di file konfigurasi: `PAYMENT_API_URL=http://10.244.1.24:8080`, maka begitu pod berganti IP, `Order Service` langsung gagal terhubung dengan jutaan error `Connection Refused`.
- Jika kamu menaikkan jumlah pod payment dari 3 menjadi 30 instance saat lonjakan transaksi, usaha tersebut sia-sia karena `Order Service` tidak mengetahui alamat IP dari 27 pod baru tersebut!

**Solusi dengan Service Discovery (misal: Kubernetes CoreDNS / HashiCorp Consul):**
Setiap kali pod payment baru booting, pod secara otomatis mendaftarkan alamat IP dan status kesehatannya ke **Service Registry**. `Order Service` tidak perlu menghafal IP dinamis tersebut, melainkan cukup memanggil nama DNS logis yang stabil: `http://payment-service:8080`. Sistem Service Discovery secara otomatis dan transparan meresolusi nama tersebut ke alamat IP pod payment yang sedang aktif dan sehat saat itu juga.

## 2. Apa Itu dan Buat Apa?

Service Discovery adalah mekanisme otomatis dalam arsitektur sistem terdistribusi yang memungkinkan aplikasi saling menemukan lokasi jaringan (**Alamat IP dan Port**) dari service lain secara dinamis tanpa perlu menuliskan (*hardcode*) IP address statis di file konfigurasi.

Service Discovery terdiri dari 3 pilar utama:
1. **Service Registry**: Database pusat yang mencatat daftar seluruh service yang aktif beserta alamat IP, port, dan status kesehatannya (misal: Consul, Eureka, etcd, ZooKeeper, K8s CoreDNS).
2. **Registration (Pendaftaran)**: Proses saat sebuah instance service baru menyala (*startup*), ia mendaftarkan IP dan port-nya ke Registry, dan mengirim sinyal *Heartbeat* berkala. Saat mati, ia dicabut dari daftar.
3. **Lookup (Pencarian)**: Proses saat Service A ingin berbicara ke Service B, ia mencari alamat IP Service B yang sedang sehat melalui Registry.

## 3. Dua Pola Implementasi: Client-Side vs Server-Side

```
[ 1. Client-Side Discovery (Contoh: Netflix Eureka / Consul SDK) ]

+-----------------+                 +-----------------------+
|   Order Svc     | --1. Tanya IP-> |   Service Registry    |
| (Client-Side)   | <-2. Daftar IP- | (Consul / Eureka)     |
+-----------------+                 +-----------------------+
        |
        | 3. Client memilih sendiri salah satu IP (Client Load Balancing)
        v
+-----------------+
| Payment Svc #2  |
| (10.0.1.45:8080)|
+-----------------+
```

```
[ 2. Server-Side Discovery (Contoh: Kubernetes Service / AWS ALB) ]

+-----------------+
|   Order Svc     |
+-----------------+
        |
        | 1. Panggil nama stabil: "http://payment-service"
        v
+---------------------------------------+
|          Load Balancer / Proxy        |
|  (K8s Service / Kube-Proxy / Ingress) | <--- Selalu terhubung ke Registry
+---------------------------------------+
        |
        | 2. Proxy meneruskan ke Pod yang sehat
        v
+-----------------+
| Payment Pod #2  |
+-----------------+
```

## 4. Kenapa Harus Pakai Ini di High-Throughput?

Di era cloud computing dan Kubernetes, server bersifat **Ephemeral (sementara dan mudah berganti)**:
- Pod otomatis dimatikan dan dibuat baru saat autoscaling, rolling update, atau node failover.
- Sebuah pod mungkin hanya hidup selama 3 jam, dan saat pod baru menyala, ia mendapatkan IP address internal yang sama sekali baru (misal dari `10.244.1.15` berganti menjadi `10.244.2.89`).

Jika kamu menuliskan IP address statis di file konfigurasi:
- Setiap kali pod berganti IP, service lain akan gagal terhubung (*Connection Refused*).
- Menambah server baru dari 5 menjadi 50 mesin saat lonjakan trafik tidak akan ada gunanya karena tidak ada client yang tahu alamat IP 45 mesin baru tersebut!

Dengan Service Discovery, perubahan topologi jaringan (penambahan, pengurangan, dan perpindahan server) terdeteksi secara otomatis dalam hitungan milidetik.

## 5. Cuplikan Logika Inti (Client-Side Service Discovery di Go)

Implementasi pola client-side discovery dengan in-memory caching dan health checking di Go:

```go
package main

import (
	"fmt"
	"net/http"
	"sync"
	"time"
)

type ServiceDiscoveryClient struct {
	mu          sync.RWMutex
	serviceName string
	instances   []string
	lastUpdated time.Time
}

// GetInstance mengembalikan endpoint sehat dari cache in-memory lokal
func (sd *ServiceDiscoveryClient) GetInstance() (string, error) {
	sd.mu.RLock()
	if time.Since(sd.lastUpdated) < 10*time.Second && len(sd.instances) > 0 {
		inst := sd.instances[0]
		sd.mu.RUnlock()
		return inst, nil
	}
	sd.mu.RUnlock()

	// Refresh daftar endpoint dari Service Registry (misal Consul / etcd)
	sd.mu.Lock()
	defer sd.mu.Unlock()
	instances, err := fetchFromRegistry(sd.serviceName)
	if err != nil {
		if len(sd.instances) > 0 {
			return sd.instances[0], nil // Fallback ke cache lama
		}
		return "", err
	}
	sd.instances = instances
	sd.lastUpdated = time.Now()
	return sd.instances[0], nil
}

func fetchFromRegistry(name string) ([]string, error) {
	// Melakukan query HTTP ke Consul agent: GET /v1/health/service/:name
	_ = http.Get
	return []string{"10.0.1.20:8080", "10.0.1.21:8080"}, nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Otomatisasi Skalabilitas Penuh**: Instance baru langsung menerima trafik begitu dinyatakan sehat tanpa intervensi konfigurasi manual.
- **Toleransi Kegagalan Jaringan**: Instance yang mengalami crash otomatis diisolasi dari daftar lookup dalam beberapa detik.
- **Abstraksi Bersih**: Developer cukup mengingat nama logis service (`payment-service`) daripada menghafal IP server.

### Kekurangan
- **Ketergantungan Kritis (*Centralized Dependency*)**: Jika Service Registry down dan client tidak memiliki cache lokal, seluruh komunikasi antar-service di dalam sistem terputus total.
- **Risiko Heartbeat Storm**: Ribuan container yang secara agresif mengirimkan ping heartbeat setiap 1 detik bisa membebani CPU service registry.
- **Latensi DNS / Registry Lookup**: Menambahkan satu langkah pencarian sebelum request pertama dikirimkan.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Seluruh arsitektur microservices terdistribusi yang berjalan di atas cloud container (Kubernetes, Docker Swarm, Nomad, AWS ECS).
- Sistem dengan jumlah instance yang dinamis berubah (*autoscaling*).

### Kapan Jangan Pakai
- Arsitektur monolith sederhana dengan satu server database tetap yang IP-nya statis selama bertahun-tahun.
- Lingkungan on-premise kecil dengan arsitektur tetap di mana nama domain DNS internal statis sudah mencukupi.

## 8. Metrik Kunci & Observabilitas

Untuk memonitor performa dan resolusi service discovery pada Grafana / Datadog:

- **`service_registry_healthy_instances_count`**: Jumlah instance sehat per service name; jika angka ini turun mendadak, terjadi kegagalan deployment atau node crash.
- **`service_discovery_lookup_latency_ms`**: Latensi membaca daftar IP dari cache atau registry server.
- **`service_registry_heartbeat_failure_rate`**: Persentase kegagalan heartbeat yang memicu deregistrasi otomatis.
- **`dns_lookup_duration_seconds` (CoreDNS)**: Waktu resolusi DNS internal di kluster Kubernetes.
- **`service_registry_churn_rate`**: Frekuensi penambahan dan pencabutan instance per menit; churn yang sangat tinggi menandakan pod flapping.

## 9. Studi Kasus Nyata di Industri

- **Netflix Eureka Service Discovery**: Di puncak migrasi arsitektur microservices mereka di AWS, Netflix merancang Eureka untuk menangani puluhan ribu instance server yang terus menyala dan mati di berbagai availability zones. Client Netflix (Ribbon) menyimpan cache lokal daftar server sehingga jika server Eureka mati, sistem tetap dapat beroperasi normal.
- **HashiCorp Consul di Skala Enterprise**: Consul menyediakan service mesh dan discovery terdistribusi multi-datacenter berbasis protokol Gossip (Serf). Perusahaan fintech global menggunakan Consul untuk merutekan ribuan transaksi lintas region tanpa ketergantungan pada DNS manual.

## 10. Fakta Menarik & Gotchas Production

- **The Infamous Java DNS Caching Gotcha**: Di JVM (Java Virtual Machine), settingan default resolusi nama DNS untuk keamanan jaringan sering kali bernilai **`networkaddress.cache.ttl = -1` (artinya: CACHE FOREVER / Simpan Selamanya)**! Akibatnya, saat pod K8s berganti IP, aplikasi Java-mu akan terus-menerus menembak alamat IP lama yang sudah mati dan melemparkan jutaan error `ConnectException`. Selalu setel konfigurasi JVM `networkaddress.cache.ttl` ke angka kecil (misal 5 hingga 10 detik)!
- **Deregistration Delay**: Saat pod dimatikan secara mendadak, ada jeda waktu beberapa detik antara proses mati dengan terhapusnya IP dari Service Registry. Selama jeda waktu itu, request yang dikirimkan client masih bisa tersasar ke pod yang sudah mati. Solusinya: implementasikan **Graceful Shutdown** dan mekanisme **Retries otomatis dengan timeout pendek** di sisi client.
