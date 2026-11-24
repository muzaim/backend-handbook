# Elasticsearch (Distributed Search & Analytics Engine)

## 1. Skenario Sistem Nyata

Pada sebuah platform marketplace e-commerce dengan katalog 15.000.000 (15 juta) produk, jutaan pengguna aktif mencari barang belanjaan melalui search bar aplikasi dengan kata kunci dinamis:  
`"sepatu lari nike pria merah ukuran 42"`

Jika fitur pencarian ini dipaksakan menggunakan basis data relasional (PostgreSQL atau MySQL):

-   Query SQL terpaksa menggunakan operator LIKE dengan wildcard awalan:  
    `SELECT * FROM products WHERE description ILIKE '%sepatu%' AND description ILIKE '%nike%';`
-   Indeks standar B-Tree **tidak bisa digunakan** untuk pola wildcard awalan (`%kata%`). Database terpaksa melakukan pembacaan seluruh 15 juta baris tabel dari disk (_Full Table Scan_).
-   Latensi query melonjak hingga 7,8 detik per pencarian. Saat 1.500 pengguna mencari barang secara bersamaan, database utama langsung tumbang karena CPU 100%.
-   Selain lambat, database SQL tidak mampu menangani kesalahan ketik (_typo tolerance_, misal user mengetik "sepattu"), tidak paham sinonim kata, dan tidak bisa mengurutkan hasil berdasarkan relevansi (_relevance scoring_).

**Solusi dengan Elasticsearch:**
Katalog produk disinkronkan ke cluster terdistribusi Elasticsearch. Alih-alih menyimpan teks mentah per baris, Elasticsearch menggunakan struktur data **Inverted Index**. Dokumen teks dipecah menjadi token kata, diubah ke bentuk dasar (_stemming_), dan dipetakan ke ID produk. Query pencarian di 15 juta produk selesai dalam waktu **15 milidetik (< 0.02 detik)** dengan hasil yang diurutkan secara cerdas berdasarkan skor algoritma relevansi **BM25**, lengkap dengan fitur koreksi typo otomatis.

## 2. Apa Itu dan Buat Apa?

Elasticsearch adalah mesin pencari teks penuh (_full-text search engine_) dan analitik terdistribusi berbasis pustaka **Apache Lucene** yang diakses melalui antarmuka RESTful API berbasis JSON.

Karakteristik utama:

-   **Inverted Index**: Struktur data seperti indeks di halaman belakang buku. Mengubah pencarian dari "mencari kata di dalam dokumen" menjadi "mencari dokumen mana saja yang memuat kata tersebut".
-   **Analisis Teks Canggih (Text Analyzers)**: Mengubah teks bebas menjadi token terstandarisasi melalui proses _Character Filtering_, _Tokenization_, _Lowercasing_, dan _Stemming_ (misal: "berlari", "pelari", "berlarian" dinormalisasi ke kata dasar "lari").
-   **Arsitektur Terdistribusi (Sharding & Replication)**: Indeks data dibagi menjadi beberapa pecahan (_shards_) yang disebar ke puluhan node server untuk eksekusi query paralel dan ketahanan terhadap kegagalan hardware (_high availability_).

## 3. Kenapa Harus Pakai Ini ?

Dalam skala arsitektur modern, memisahkan beban kerja pencarian (_search workload_) dari basis data transaksional utama adalah hukum wajib:

1. **Pencarian Skala Besar Super Cepat**: Melayani puluhan ribu query pencarian kompleks per detik dengan latensi sub-50ms tanpa membebani database PostgreSQL/MySQL.
2. **Skor Relevansi Dinamis (BM25 Algorithm)**: Menilai dokumen mana yang paling cocok berdasarkan frekuensi kemunculan kata kunci (_TF-IDF / Okapi BM25_).
3. **Agregasi Multi-Dimensi (_Faceted Search_)**: Menghitung filter produk secara instan (misal: jumlah produk per kategori merek, rentang harga, dan rating bintang dalam satu request).

## 4. Cara Kerjanya (Diagram Alur Inverted Index)

```
 [Dokumen Mentah di Database]
 Doc 1: "Sepatu lari Nike pria"
 Doc 2: "Sepatu basket Adidas pria"
 Doc 3: "Baju lari Nike wanita"
                     |
                     v (Text Analysis & Tokenization)
+-------------------------------------------------------------+
|                      INVERTED INDEX                         |
+-------------------+-----------------------------------------+
| Term (Kata Kunci) | Document IDs (Posting List)             |
+-------------------+-----------------------------------------+
| adidas            | [Doc 2]                                 |
| baju              | [Doc 3]                                 |
| basket            | [Doc 2]                                 |
| lari              | [Doc 1, Doc 3]                          |
| nike              | [Doc 1, Doc 3]                          |
| pria              | [Doc 1, Doc 2]                          |
| sepatu            | [Doc 1, Doc 2]                          |
| wanita            | [Doc 3]                                 |
+-------------------+-----------------------------------------+
                     |
                     v
 [Pencarian User]: "sepatu nike"
 1. Lookup "sepatu" -> [Doc 1, Doc 2]
 2. Lookup "nike"   -> [Doc 1, Doc 3]
 3. Irisan (Intersection) -> [Doc 1] (Skor Relevansi Tertinggi!)
 [Waktu Eksekusi: < 5 ms]
```

## 5. Cuplikan Logika Inti (Search Query Multi-Match di Go)

Berikut implementasi pencarian produk berbasis Elasticsearch di Go menggunakan client resmi `github.com/elastic/go-elasticsearch/v8`, lengkap dengan fitur multi-match, fuzzy search (toleransi typo), dan filter kategori:

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"log"

	"github.com/elastic/go-elasticsearch/v8"
)

type SearchResult struct {
	Hits struct {
		Total struct {
			Value int `json:"value"`
		} `json:"total"`
		Hits []struct {
			ID     string          `json:"_id"`
			Score  float64         `json:"_score"`
			Source json.RawMessage `json:"_source"`
		} `json:"hits"`
	} `json:"hits"`
}

func SearchProducts(ctx context.Context, es *elasticsearch.Client, keyword string, category string) (*SearchResult, error) {
	// Membangun Query DSL JSON dengan fitur Multi-Match dan Fuzziness (Anti-Typo)
	queryMap := map[string]interface{}{
		"query": map[string]interface{}{
			"bool": map[string]interface{}{
				"must": map[string]interface{}{
					"multi_match": map[string]interface{}{
						"query":     keyword,
						"fields":    []string{"title^3", "description^1", "brand^2"}, // Boosting prioritas judul
						"fuzziness": "AUTO",                                         // Toleransi salah ketik
					},
				},
				"filter": []map[string]interface{}{
					{
						"term": map[string]interface{}{
							"category.keyword": category,
						},
					},
				},
			},
		},
		"size": 10,
	}

	var buf bytes.Buffer
	if err := json.NewEncoder(&buf).Encode(queryMap); err != nil {
		return nil, fmt.Errorf("gagal encode query: %w", err)
	}

	// Eksekusi pencarian ke index "products"
	res, err := es.Search(
		es.Search.WithContext(ctx),
		es.Search.WithIndex("products"),
		es.Search.WithBody(&buf),
		es.Search.WithTrackTotalHits(true),
	)
	if err != nil {
		return nil, fmt.Errorf("gagal eksekusi search: %w", err)
	}
	defer res.Body.Close()

	if res.IsError() {
		return nil, fmt.Errorf("elasticsearch error response: %s", res.Status())
	}

	var result SearchResult
	if err := json.NewDecoder(res.Body).Decode(&result); err != nil {
		return nil, fmt.Errorf("gagal decode response: %w", err)
	}

	return &result, nil
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan

-   **Performa Pencarian Teks Tiada Tanding**: Menemukan jutaan kata dan dokumen dalam hitungan milidetik melalui Inverted Index.
-   **Relevance Scoring & Fuzzy Matching**: Hasil pencarian diurutkan berdasarkan tingkat kecocokan kontekstual, bukan sekadar cocok biner.
-   **Skalabilitas Horizontal Bawaan**: Kapasitas cluster dapat diperbesar dengan mudah melalui penambahan node data dan re-sharding otomatis.

### Kekurangan

-   **Bukan Pengganti Database Transaksional (Non-ACID)**: Elasticsearch tidak mendukung transaksi multi-dokumen ACID atau referensial foreign keys; rentan kehilangan data jika dijadikan sumber kebenaran tunggal (_Source of Truth_).
-   **Konsumsi Memori RAM & JVM Heap Sangat Tinggi**: Membutuhkan alokasi memori besar (50% RAM dialokasikan untuk JVM Heap, 50% untuk OS Page Cache).
-   **Kompleksitas Operasional Cluster**: Menjaga cluster tetap sehat (_Green status_) membutuhkan pemahaman mendalam tentang alokasi shard, JVM GC tuning, dan master node quorum.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai

-   Fitur pencarian katalog e-commerce, portal berita, atau media sosial yang membutuhkan pencarian teks bebas, autocomplete, dan filter atribut (_faceted filters_).
-   Agregasi dan analisis log terpusat berskala raksasa (ELK Stack: Elasticsearch, Logstash, Kibana).
-   Pencarian dokumen geospasial berdasarkan radius koordinat GPS.

### Kapan Jangan Pakai

-   Sistem pembukuan akuntansi, ledger perbankan, atau data transaksional yang menuntut konsistensi ACID mutlak.
-   Kebutuhan penyimpanan data sederhana yang cukup dilayani oleh PostgreSQL `LIKE` atau indeks `GIN` untuk tabel berskala kecil (< 100.000 baris).
-   Tim dengan anggaran infrastruktur terbatas yang belum siap mengelola cluster server dengan kebutuhan RAM minimal puluhan gigabyte.

## 8. Metrik Kunci & Observabilitas (Monitoring Dashboard)

-   **Cluster Health Status (`green` / `yellow` / `red`)**: Status hijau berarti semua shard primer dan replika aktif. Status merah berarti ada data shard primer yang hilang dari cluster.
-   **`jvm_gc_collection_time_ms` (Counter)**: Waktu henti pembersihan memori (_Garbage Collection pause_). Lonjakan GC pause akan menyebabkan node dianggap mati oleh cluster coordinator (_node disconnect_).
-   **`elasticsearch_indices_search_query_latency_ms` (Histogram)**: Latensi eksekusi query pencarian p95 dan p99.
-   **`unassigned_shards_count` (Gauge)**: Jumlah pecahan data yang belum berhasil dialokasikan ke node aktif.

## 9. Studi Kasus Industri Nyata

-   **Wikipedia**: Menggunakan Elasticsearch untuk mengindeks dan melayani miliaran pencarian artikel ensiklopedia dalam ratusan bahasa secara real-time di seluruh dunia.
-   **GitHub**: Menggunakan kluster Elasticsearch berukuran masif untuk menjalankan fitur _Code Search_ yang mengindeks miliaran baris kode sumber dari jutaan repositori publik dan privat.

## 10. Jebakan Umum di Lingkungan Produksi (Gotchas)

1. **Jebakan Heap Melebihi 32 GB (_Compressed OOPs Threshold_)**: Mengalokasikan RAM JVM Heap sebesar 64 GB dengan harapan performa naik 2x lipat. Faktanya, JVM akan menonaktifkan fitur _Compressed Object Pointers (Compressed OOPs)_ saat heap menyentuh 32 GB, membuat pointer objek membengkak 2x lipat dan performa justru melambat drastis!  
   _Solusi_: Jangan pernah menyetel JVM heap lebih dari 31 GB (`-Xmx31g`).
2. **Kelebihan Shard (_Over-sharding Trap_)**: Membuat 50 shard untuk indeks kecil berukuran ratusan megabyte. Setiap shard adalah instans Lucene tersendiri yang memakan file descriptor dan memori heap.  
   _Solusi_: Targetkan ukuran satu shard ideal berada di kisaran 20 GB hingga 50 GB.
3. **Pemberitahuan Status Red Akibat Kehabisan Disk**: Elasticsearch memiliki fitur _Disk Watermark_. Jika kapasitas hard disk menyentuh 95%, Elasticsearch otomatis mengunci seluruh indeks menjadi _Read-Only_ dan menolak semua penulisan baru (_Flood Stage Block_).

## 11. Rangkuman Singkat

Elasticsearch adalah mesin pencari dan analitik terdistribusi berkecepatan tinggi yang mengandalkan struktur data Inverted Index. Dengan memindahkan beban pencarian teks bebas dan analitik dari basis data relasional ke Elasticsearch, sistem backend berskala tinggi dapat memberikan pengalaman pencarian sub-milidetik yang cerdas dan tahan banting.
