# CDN & Edge Caching (Content Delivery Network)

## 1. Skenario Sistem Nyata

Pada platform e-commerce global atau aplikasi media sosial yang server utamanya (_origin server_) berlokasi di Jakarta (`ap-southeast-3`), pengguna dari berbagai belahan dunia mengakses aset yang sama secara serentak. Sekarang, seorang pengguna dari Frankfurt, Jerman, membuka halaman produk utama yang memuat file JavaScript bundle (500 KB), stylesheet CSS (100 KB), puluhan gambar produk berkualitas tinggi (3 MB total), serta API katalog produk populer.

Tanpa Content Delivery Network (CDN):

-   Paket data harus menempuh kabel fiber optik bawah laut antar-benua sejauh belasan ribu kilometer.
-   Setiap _round-trip time_ (RTT) antara Frankfurt dan Jakarta memakan waktu **180 ms - 250 ms**.
-   Karena handshake TLS 1.3 dan TCP membutuhkan beberapa RTT, waktu tunggu sebelum byte pertama diterima (_Time to First Byte_ / TTFB) melonjak hingga **1.2 detik**, dan total pemuatan halaman memakan waktu **3 hingga 5 detik**!
-   Ketika jutaan pengguna di seluruh dunia mengakses aplikasi secara serentak, origin server di Jakarta mengalami kelebihan beban bandwidth (_bandwidth saturation_) dan kehabisan memori hingga down.

**Solusi dengan CDN & Edge Caching:**
Konten statis (gambar, JS, CSS) dan respons API yang dapat di-cache disalin dan disimpan di ratusan server tepi (_Edge Points of Presence_ / PoP) yang tersebar di seluruh dunia (seperti Cloudflare, AWS CloudFront, atau Fastly). Ketika pengguna di Frankfurt membuka aplikasi, request mereka dicegat oleh server Edge CDN yang berjarak hanya beberapa kilometer di Frankfurt. RTT anjlok dari 200 ms menjadi **5 ms**, TTFB turun drastis ke **15 ms**, dan **95% beban bandwidth** sama sekali tidak menyentuh origin server di Jakarta!

## 2. Apa Itu dan Buat Apa?

**CDN (Content Delivery Network)** adalah jaringan server terdistribusi secara geografis yang ditempatkan sedekat mungkin dengan pengguna akhir (_edge of the network_). Tujuannya adalah mempercepat pengiriman konten web berukuran besar maupun dinamis serta melindungi server asal (_origin server_) dari lonjakan lalu lintas global.

### Komponen Utama CDN:

1. **Origin Server**: Server pusat tempat aplikasi dan database utama berada (misal Kubernetes cluster di AWS atau GCP).
2. **Edge PoP (Point of Presence)**: Data center lokal di berbagai kota besar di dunia yang berisi ribuan server proxy cache.
3. **Anycast DNS / BGP Routing**: Teknologi perutean jaringan yang mengarahkan IP pengguna secara otomatis ke PoP terdekat secara geografis dan latensi jaringan terpendek.
4. **Edge Compute / Edge Workers**: Kemampuan menjalankan kode logika ringan (V8 isolates / WebAssembly) langsung di server tepi sebelum request diteruskan ke origin (misal verifikasi JWT, A/B testing header injection, image resizing on-the-fly).

## 3. Kenapa Harus Pakai Ini ?

### 1. Offloading Beban Jaringan dan CPU Origin Server

Pada sistem bervolume tinggi, melayani file statis dan query katalog publik melalui origin server adalah pemborosan besar kapasitas komputasi. CDN bertindak sebagai perisai (_origin shield_): jika _Cache Hit Ratio_ mencapai 95%, artinya sebuah origin server yang hanya mampu menangani 5.000 RPS kini sanggup melayani **100.000 RPS ekuivalen**, karena 95.000 request dipenuhi langsung oleh Edge server.

### 2. Eliminasi Latensi Jarak Fisik (_Speed of Light Constraint_)

Cepatnya sinyal listrik atau cahaya di kabel fiber optik dibatasi oleh hukum fisika (~200.000 km/s di kaca fiber). Menghubungkan client di belahan bumi lain ke satu data center terpusat pasti menimbulkan latensi fisik. Memindahkan cache data ke kota tempat client berada adalah satu-satunya cara memangkas latensi ke level satu digit milidetik.

### 3. Perlindungan Terhadap Serangan DDoS

Edge server CDN memiliki kapasitas bandwidth agregat puluhan hingga ratusan Terabit per detik (Tbps). Lonjakan serangan lalu lintas berbahaya (Volumetric DDoS Attack) diserap dan difilter langsung di lapisan terluar tepi jaringan sebelum sempat mencapai origin network.

## 4. Cara Kerjanya (Diagram Alur CDN & Edge Caching)

```
[ Pengguna Global ]                      [ Edge Server (PoP) ]                   [ Origin Server ]
(Misal di Frankfurt)                    (CDN Frankfurt Edge)                   (Data Center Jakarta)
       |                                          |                                      |
       |----- 1. GET /product/banner.webp ------->|                                      |
       |                                          |-- 2. Cek Local Cache?                |
       |                                          |   [ CACHE MISS ]                     |
       |                                          |                                      |
       |                                          |----- 3. Fetch ke Origin ------------>|
       |                                          |<---- 4. 200 OK + Cache-Control ------|
       |                                          |                                      |
       |                                          |-- 5. Simpan di Edge Disk/RAM         |
       |<---- 6. Return Data (TTFB: 180ms) -------|                                      |
       |                                          |                                      |
       | === PENGGUNA BERIKUTNYA DARI KOTA SAMA ======================================   |
       |                                          |                                      |
       |----- 7. GET /product/banner.webp ------->|                                      |
       |                                          |-- 8. Cek Local Cache?                |
       |                                          |   [ CACHE HIT! ]                     |
       |<---- 9. Return Langsung (TTFB: 12ms!) ---| (Tidak perlu menghubungi Origin!)   |
```

## 5. Cuplikan Logika Inti

### Konfigurasi HTTP Header `Cache-Control` di Backend (Go / Node.js)

Origin server memberitahu CDN Edge berapa lama konten boleh disimpan dan bagaimana strategi revalidasinya menggunakan header standar HTTP:

```go
func handleCatalogAPI(w http.ResponseWriter, r *http.Request) {
    data := getPopularCatalog()

    // 1. max-age=60: Browser client menyimpan cache selama 60 detik
    // 2. s-maxage=3600: Server CDN Edge menyimpan cache selama 1 jam (3600 detik)
    // 3. stale-while-revalidate=300: Jika cache expired, CDN boleh kirim cache lama
    //    ke user secara instan sambil diam-diam refresh cache ke origin di background
    w.Header().Set("Cache-Control", "public, max-age=60, s-maxage=3600, stale-while-revalidate=300")
    w.Header().Set("Content-Type", "application/json")
    w.Write(data)
}
```

### Edge Worker Script (Cloudflare Worker / V8 Edge)

Contoh manipulasi request di edge sebelum menyentuh origin:

```javascript
export default {
	async fetch(request, env, ctx) {
		const url = new URL(request.url);
		const cache = caches.default;

		// Cek cache edge terlebih dahulu
		let response = await cache.match(request);
		if (response) {
			// Tambahkan header observabilitas kustom
			let newHeaders = new Headers(response.headers);
			newHeaders.set("X-Edge-Status", "HIT-LOCAL-EDGE");
			return new Response(response.body, { headers: newHeaders });
		}

		// Jika Miss, teruskan ke origin server
		response = await fetch(request);

		// Salin respons dan simpan ke Edge Cache secara asinkron
		if (response.status === 200) {
			ctx.waitUntil(cache.put(request, response.clone()));
		}

		return response;
	},
};
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Penurunan Latensi Drastis**: TTFB turun dari ratusan milidetik ke hitungan belasan milidetik di seluruh dunia.
-   **Penghematan Biaya Egress Origin**: Trafik bandwidth keluar dari origin cloud server turun drastis (menghemat tagihan cloud AWS Data Transfer Out / GCP Egress).
-   **Ketersediaan Tinggi (_High Availability_)**: Jika origin server mengalami restart sesaat atau down parsial, CDN masih dapat menyajikan konten lama kepada pengguna menggunakan fitur `stale-if-error`.

### Kekurangan

-   **Kompleksitas _Cache Invalidation_**: _"There are only two hard things in Computer Science: cache invalidation and naming things."_ Ketika ada harga produk diskon yang diubah mendadak, menghapus (_purge_) cache di 300+ PoP di seluruh dunia memerlukan waktu propagasi (beberapa detik hingga hitungan menit).
-   **Risiko _Cache Stampede / Thundering Herd_ di Edge**: Ketika sebuah file video atau rilis versi baru viral pertama kali diminta bersamaan oleh 50.000 user dari 50 kota, semua Edge PoP serentak melakukan _fetch origin_ secara bersamaan jika fitur _Origin Shielding_ belum diaktifkan.
-   **Biaya Tambahan CDN**: Meskipun menghemat origin egress, penggunaan fitur canggih seperti dynamic routing, edge workers compute time, dan log streaming real-time menambah pos biaya infrastruktur tersendiri.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Pakai CDN

-   Semua aset web statis: Gambar (JPEG/WebP/AVIF), font, video streaming chunk (HLS/DASH), file JavaScript, CSS.
-   API Read-Heavy dengan data semi-statis: Halaman beranda (_landing page_), katalog produk publik, daftar artikel berita, ranking/leaderboard yang diperbarui setiap beberapa menit.
-   Aplikasi dengan basis pengguna yang tersebar di banyak pulau atau negara.

### Kapan Jangan Meng-cache di Edge CDN

-   Respons API yang memuat data pribadi sensitif pengguna (misal `GET /api/v1/user/profile`, data saldo rekening, kartu kredit)—jika ter-cache di CDN publik secara keliru, data satu user bisa bocor ke user lain!
-   Request mutasi data (POST, PUT, DELETE, PATCH).
-   Data real-time berlatensi ultra-rendah yang berubah setiap sub-detik (misal order book bursa saham real-time).

## 8. Metrik Kunci & Observabilitas

Metrik wajib di dashboard monitoring CDN (Cloudflare Analytics, Fastly Insights, Datadog CDN integration):

-   **`edge_cache_hit_ratio_percent`**: Persentase request yang berhasil diselesaikan langsung oleh edge CDN tanpa menyentuh origin (target ideal: > 90% untuk statis, > 60% untuk API publik).
-   **`origin_bandwidth_offloaded_bytes`**: Total volume bandwidth jaringan yang berhasil dihemat oleh CDN.
-   **`edge_time_to_first_byte_p95_ms` (TTFB)**: Waktu respons antara pengguna mengirim request sampai menerima byte pertama di edge.
-   **`origin_error_rate_5xx`**: Persentase error 502/503/504 dari origin yang dilaporkan oleh Edge PoP.
-   **`cache_purge_propagation_latency_ms`**: Waktu yang dibutuhkan instruksi invalidasi purge API untuk membersihkan seluruh PoP global.

## 9. Studi Kasus Nyata di Industri

### Penyerapan 95% Trafik Flash Sale Global dan Perlindungan Server Origin via Edge Caching

- **Tantangan Rekayasa Riil (The Problem)**: Sebuah marketplace global menggelar kampanye belanja akhir tahun dengan lonjakan trafik hingga 120.000 HTTP request per detik pada katalog produk dan landing page promo. Server origin (database dan cluster API backend) kewalahan menangani pembacaan data statis dan semi-dinamis yang berulang, menyebabkan utilisasi database CPU menyentuh 100%, p99 latency melonjak dari 80 ms menjadi 12.000 ms, dan ribuan user mengalami kegagalan akses saat membuka katalog.
- **Implementasi Solusi Industri (The Architectural Solution)**: Tim merekayasa strategi edge caching terdistribusi menggunakan CDN tiering (Cloudflare / CloudFront) dengan mengonfigurasi header `Cache-Control: public, max-age=60, s-maxage=600, stale-while-revalidate=30`. Aset media statis dikunci dengan immutable hash tags. Untuk data katalog semi-dinamis, diaktifkan *Tiered Caching / Origin Shield* guna mencegah *cache stampede* dari ratusan Edge PoP ke origin server, dikombinasikan dengan webhooks otomatis untuk memicu API cache purge instan saat harga produk diperbarui admin.
- **Dampak & Hasil Sistem (The Outcome)**: CDN Edge Hit Ratio melonjak hingga 96,4%, memotong 96% beban request ke origin server. Waktu muat halaman (Time to First Byte / TTFB) bagi pengunjung global terpangkas dari 1.200 ms menjadi rata-rata 32 ms. Server origin beroperasi stabil di bawah 25% CPU selama puncak flash sale tanpa penambahan instans backend.

## 10. Fakta Menarik & Gotchas Production

-   **The Private Cache Header Disaster (Bocornya Akun Orang Lain)**:  
    Salah satu insiden paling mematikan dalam sejarah web architecture adalah salah konfigurasi header `Cache-Control`. Jika seorang developer lupa menyetel header `private` pada endpoint profil pengguna yang terautentikasi (seperti `GET /api/v1/me`) dan secara tidak sengaja mengembalikan `Cache-Control: public, s-maxage=3600`, maka server CDN Edge akan menyimpan respons profil User A! Akibatnya, ratusan pengguna lain yang membuka halaman tersebut berikutnya akan melihat data akun, nama, dan email milik User A!  
    _Aturan Wajib:_ Semua endpoint terautentikasi (`Authorization` / session cookie) wajib membawa header **`Cache-Control: private, no-store, no-cache`**.
-   **Origin Shielding (Menyelamatkan Origin dari Serangan Cache Miss Global)**:  
    Jika CDN-mu memiliki 350 PoP di seluruh dunia, dan satu cache key kedaluwarsa pada aset populer, ke-350 PoP tersebut bisa secara bersamaan mengirim 350 request paralel ke origin server-mu (_multi-pop stampede_).  
    _Solusi Wajib:_ Aktifkan fitur **Origin Shield** (Cloudflare Tiered Cache / AWS CloudFront Origin Shield). Fitur ini menempatkan satu lapisan super-PoP perantara antara edge dan origin, sehingga hanya 1 request perantara yang diteruskan ke origin server.

## 11. Rangkuman Singkat

- **Dekatkan Konten ke Pengguna**: CDN mendistribusikan salinan konten statis dan semi-dinamis ke ratusan PoP (Point of Presence) edge global untuk memangkas latensi jaringan secara dramatis.
- **Kuasai Header Cache-Control**: Manfaatkan `s-maxage` untuk cache CDN dan `stale-while-revalidate` untuk menyajikan konten stale secara instan sambil memperbarui cache di latar belakang secara asinkron.
- **Bahaya Kebocoran Data Pribadi**: Pastikan endpoint terautentikasi dan data profil user selalu menyertakan `Cache-Control: private, no-store` agar respons privat tidak tersimpan di cache publik edge.
- **Gunakan Origin Shielding**: Hindari kejatuhan server origin akibat *cache stampede* serentak dari ratusan PoP edge dengan menempatkan layer perantara *Tiered Cache / Origin Shield*.
