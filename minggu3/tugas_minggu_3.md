# Tugas Analisis Minggu 3

## 1. Analisis Pola Arsitektur

Rancangan ini termasuk Adapter Pattern karena program C hanya bisa membaca dan menulis file teks lokal dan tidak punya akses jaringan. Server `adapter_service.py` menjadi perantara yang mengubah permintaan HTTP/XML dari klien menjadi format CSV di file teks, dan sebaliknya. Kode program C tidak diubah.

Keuntungan untuk klien web:
- Klien cukup tahu endpoint dan format XML, tidak perlu tahu format file legacy.
- Sistem legacy bisa diganti nanti tanpa mengubah klien selama format XML tetap sama.
- Sistem lama tidak perlu dimodifikasi sehingga risikonya lebih kecil.
- HTTP dan XML bisa digunakan dari berbagai platform.

## 2. Analisis Overhead Serialisasi Data

Respons dari `GET /students/2415354001`:

```xml
<?xml version='1.0' encoding='utf-8'?>
<StudentResponse><NIM>2415354001</NIM><Nama>I Made Sujana</Nama><Jurusan>Teknologi Informasi</Jurusan><Status>ACTIVE</Status></StudentResponse>
```

Data sebenarnya:
- 2415354001 = 10 byte
- I Made Sujana = 13 byte
- Teknologi Informasi = 19 byte
- ACTIVE = 6 byte

Total data = 48 byte.

Ukuran respons XML:
- Tag pembungkus = 95 byte
- Isi dokumen = 95 + 48 = 143 byte
- Deklarasi XML + newline = 39 byte
- Total = 182 byte

Overhead:
- 182 - 48 = 134 byte
- 134 / 182 x 100% = 73,6%
- Jika dibandingkan dengan data aslinya: 134 / 48 x 100% = 279,2%

Jadi sekitar tiga perempat isi respons hanya berupa tag, bukan data. Sebagai perbandingan, satu baris CSV aslinya hanya 51 byte. Header HTTP tidak dihitung.

## 3. Prediksi Keterbatasan Integrasi

Server bawaan Python (`HTTPServer`) memproses satu permintaan per satu. Jadi kalau ada 500 POST, permintaan akan mengantre dan sebagian bisa timeout.

Kalau server dibuat multi-thread atau ada program lain yang menulis ke file yang sama, `students_db.txt` juga bisa bermasalah, misalnya:
- data tercampur atau terpotong karena penulisan bersamaan
- data duplikat karena tidak ada pengecekan NIM unik
- data hilang jika proses berhenti saat menulis
- tidak ada mekanisme locking pada file teks

Message Broker seperti RabbitMQ atau Kafka bisa digunakan karena:
- permintaan bisa ditampung dalam antrean lalu diproses satu per satu
- adapter bisa langsung memberi balasan "diterima" tanpa menunggu penulisan selesai
- pesan bisa dikirim ulang jika penulisan gagal
- klien dan sistem legacy menjadi lebih terpisah dan mudah dikembangkan