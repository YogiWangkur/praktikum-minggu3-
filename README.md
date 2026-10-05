# Praktikum Minggu 3 - Implementasi Sistem Legacy & EAI (HTTP + XML)

**Nama:** [Eugenius Arlanda Wangkur]
**NIM:** [2415354072]
**Kelas:** 5D TRPL
**Mata Kuliah:** Integrasi Sistem Informasi

## Tujuan
Membangun layanan Adapter berbasis HTTP dan XML (Python) yang menjembatani klien modern dengan sistem legacy berbasis C dan file teks, tanpa mengubah kode sumber legacy.

## Struktur Berkas
- `students_db.txt` : basis data teks (legacy data store)
- `legacy_core.c` : mesin legacy (C)
- `adapter_service.py` : adapter EAI (Python HTTP/XML)

## Cara Menjalankan
```powershell
gcc legacy_core.c -o legacy_core
.\legacy_core.exe
python adapter_service.py
curl.exe -i http://localhost:8000/students
```

## Hasil Pengujian

### Legacy Engine (awal)
![Legacy awal](screenshots/01_legacy_core_awal.png)

### Skenario A: GET
![GET all](screenshots/02_get_all.png)
![GET by NIM](screenshots/03_get_by_nim.png)

### Skenario B: POST XML
![POST](screenshots/04_post.png)

### Skenario C: Verifikasi di Legacy
![Legacy setelah POST](screenshots/05_legacy_core_setelah_post.png)

## Analisis

### 1. Analisis Pola Arsitektur
Rancangan ini adalah Adapter Pattern karena program C hanya dapat membaca dan menulis file CSV lokal tanpa HTTP dan XML. `adapter_service.py` menerjemahkan antarmuka modern (HTTP + XML) ke format internal legacy (baris CSV) dan sebaliknya, tanpa mengubah kode sumber legacy.

Keuntungan bagi klien web:
- Loose coupling: klien hanya mengetahui kontrak `GET/POST /students` dan skema XML, bukan lokasi file atau format CSV.
- Isolasi perubahan: jika penyimpanan diganti database SQL, hanya adapter yang berubah.
- Interoperabilitas: klien apa pun dapat mengakses data lewat protokol standar.
- Risiko rendah: sistem legacy yang vital tidak perlu dimodifikasi.

### 2. Analisis Overhead Serialisasi XML
Respons `GET /students/2415354001`:
```xml
<?xml version='1.0' encoding='utf-8'?>
<StudentResponse><NIM>2415354001</NIM><Nama>I Made Sujana</Nama><Jurusan>Teknologi Informasi</Jurusan><Status>ACTIVE</Status></StudentResponse>
```

| Field | Nilai | Byte |
|---|---|---|
| NIM | 2415354001 | 10 |
| Nama | I Made Sujana | 13 |
| Jurusan | Teknologi Informasi | 19 |
| Status | ACTIVE | 6 |
| **Total data aktual** | | **48** |

- Total respons XML (hasil pengukuran): **182 byte**
- Byte tag dan deklarasi: 182 - 48 = **134 byte**
- Overhead terhadap total respons: 134 / 182 = **73,6%**
- Overhead terhadap data murni: 134 / 48 = **279%**

Kesimpulan: XML bersifat verbose karena setiap field memiliki tag pembuka dan penutup, sehingga pada data kecil ukuran tag melebihi ukuran isinya.

### 3. Prediksi Keterbatasan Integrasi
Jika 500 permintaan POST datang bersamaan:
- `HTTPServer` bawaan bersifat single-threaded, sehingga permintaan diproses satu per satu. Antrean koneksi terbatas, sehingga sebagian permintaan dapat timeout atau ditolak.
- Jika dibuat multi-thread, banyak thread menulis ke file yang sama tanpa file locking, sehingga baris data dapat tercampur atau rusak (race condition).
- Program C dan adapter menulis ke file yang sama tanpa koordinasi.
- Tidak ada validasi duplikat NIM, dan nama yang mengandung koma merusak format CSV.
- Akar masalahnya adalah file teks datar tidak memiliki transaksi maupun kontrol konkurensi.

Message Broker (misalnya RabbitMQ atau Kafka) dibutuhkan karena:
- Permintaan tulis masuk antrean dan diproses secara berurutan oleh satu consumer, sehingga tidak ada tabrakan penulisan.
- Lonjakan beban diserap antrean, sehingga klien tidak ditolak.
- Pesan bersifat persisten dan dapat di-retry, sehingga tidak ada data hilang.
- Producer (adapter) dan consumer (penulis legacy) terpisah secara asinkron, sejalan dengan prinsip loose coupling.

## Kesimpulan
Adapter berbasis HTTP + XML berhasil mengintegrasikan sistem legacy C dengan klien modern tanpa mengubah kode legacy. Data yang ditambahkan lewat HTTP POST langsung terbaca oleh program C. Namun XML memiliki overhead ukuran yang besar, dan file teks tidak aman untuk akses konkuren.