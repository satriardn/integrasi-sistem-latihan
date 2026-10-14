# Laporan Praktikum Pertemuan 4
Integrasi Sistem Informasi - Integrasi Modern Web Services & API Gateway (RESTful HTTP/JSON)

## Bagian A: Tabel Hasil Pengujian Benchmark

Hasil dari menjalankan benchmark_serialization.py:

| Jumlah Rekaman | Ukuran XML (Bytes) | Ukuran JSON (Bytes) | Persentase Efisiensi Ukuran (%) |
|---|---|---|---|
| 10 Data | 1438 | 1166 | 18.92% |
| 50 Data | 7058 | 5826 | 17.46% |
| 100 Data | 14084 | 11652 | 17.27% |
| 500 Data | 70684 | 58652 | 17.02% |


## Bagian B: Analisis Pertanyaan Kritis

### 1. Analisis Struktur Payload

Faktor sintaksis yang bikin XML lebih besar dari JSON untuk data yang sama:

- Setiap nilai ditulis dengan tag buka dan tag tutup, misalnya `<nama>Budi</nama>`, jadi nama field muncul dua kali. Di JSON cukup `"nama":"Budi"`.
- Setiap rekaman dibungkus tag `<student>...</student>`, sedangkan JSON cukup pakai kurung kurawal dan koma di dalam array.
- Dalam satu rekaman ada 4 field, jadi di XML ada 8 kali penulisan nama field (buka dan tutup), di JSON hanya 4 kali.
- XML punya banyak karakter markup (`<`, `>`, `/`) dan bisa ditambah deklarasi, namespace, atau atribut, sedangkan JSON hanya pakai simbol sederhana seperti `{ } [ ] : , "`.

Karena itu semakin banyak rekaman, selisih byte antara XML dan JSON juga makin besar.

### 2. Analisis Deserialisasi / Parsing di Sisi Klien

Kalau responsnya JSON, front-end tinggal memanggil `response.json()` atau `JSON.parse()`. Hasilnya langsung jadi objek dan array JavaScript, jadi data bisa diakses seperti `data.data[0].nama`. Hanya satu langkah dan tidak butuh library tambahan.

Kalau responsnya XML, langkahnya lebih panjang:

1. Teks XML diubah jadi pohon DOM dengan `DOMParser`.
2. Menelusuri node yang dibutuhkan (misalnya pakai `getElementsByTagName` atau XPath).
3. Mengambil `textContent` dari tiap elemen.
4. Memetakan hasilnya secara manual ke objek JavaScript. Semua nilai bertipe string, jadi kalau perlu angka atau boolean harus dikonversi sendiri.

Alasan web modern beralih ke JSON:

- JSON native di JavaScript sehingga parsing lebih sederhana dan cepat.
- Ukuran payload lebih kecil (terbukti dari benchmark).
- Mudah dibaca dan langsung cocok dengan struktur data di banyak bahasa (dict/object dan list).
- Didukung hampir semua framework dan tools API seperti Flask, Postman, dan curl.

### 3. Bottleneck Arsitektur RESTful Synchronous

Kalau sistem legacy butuh 1,5 detik untuk file locking saat menulis ke data.txt, penulisan hanya bisa dilakukan satu per satu. Dengan 100 klien yang POST bersamaan, klien pertama selesai sekitar 1,5 detik, klien kedua sekitar 3 detik, dan klien ke-100 baru selesai sekitar 150 detik (100 x 1,5 detik). Permintaan lainnya mengantre menunggu giliran.

Karena komunikasinya sinkron, tiap klien menahan koneksi dan thread di server selama menunggu. Dampaknya:

- Thread dan koneksi menumpuk sampai sumber daya habis (resource exhaustion).
- Banyak klien bisa kena timeout atau error.
- Latensi naik, dan endpoint lain seperti GET ikut lambat kalau berbagi worker yang sama.

Kenapa synchronous blocking di HTTP/1.1 jadi bottleneck:

- Synchronous coupling: klien harus menunggu sampai sistem legacy selesai menulis, jadi kecepatan seluruh alur ditentukan oleh bagian yang paling lambat.
- Head-of-Line blocking: HTTP/1.1 hanya melayani satu permintaan aktif per koneksi TCP, jadi permintaan berikutnya di koneksi yang sama harus menunggu respons sebelumnya selesai.

Cara gRPC dan Message Broker mengatasinya:

- gRPC (HTTP/2) memakai multiplexing, jadi banyak request bisa jalan bersamaan dalam satu koneksi TCP dan HoL blocking di level HTTP berkurang. Payload-nya biner (Protobuf) sehingga lebih kecil dan cepat diproses, dan ada dukungan streaming serta deadline. Tapi untuk panggilan unary biasa klien tetap menunggu respons, jadi kalau penulisan ke legacy lambat, klien tetap ikut menunggu. gRPC memperbaiki masalah transport, belum menghilangkan coupling sinkron.
- Asynchronous Message Broker (misalnya RabbitMQ atau Kafka) menyelesaikan masalah utamanya. Gateway menerima POST, memasukkan pesan ke antrean, lalu langsung membalas `202 Accepted` tanpa menunggu penulisan selesai. Worker/consumer kemudian memproses pesan satu per satu ke sistem legacy sesuai kecepatannya. Hasilnya klien tidak terblokir, lonjakan 100 permintaan ditampung antrean, pesan tidak hilang kalau legacy lambat atau gagal (bisa di-retry), dan gateway dengan sistem legacy tidak lagi saling bergantung langsung.
