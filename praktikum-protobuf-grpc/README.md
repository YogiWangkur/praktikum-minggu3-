Praktikum: Pertukaran Data Protobuf Real-Time via Docker dan WSL2

Nama: Eugenius Arlanda Wangkur NIM: 2415354072 Mata Kuliah: Integrasi Sistem Informasi

1. Tujuan

Mempelajari cara membuat, mengompilasi, melakukan serialisasi, dan mengirimkan data biner terstruktur antar lingkungan server yang terisolasi menggunakan Protocol Buffers (Protobuf) dan gRPC. Praktikum ini juga mengukur ukuran payload dan waktu transmisi data antar dua kontainer Docker.

2. Dasar Teori Singkat
Protocol Buffers (Protobuf) adalah mekanisme serialisasi data terstruktur. Skema data ditulis pada file .proto, lalu dikompilasi menjadi kode. Hasil serialisasinya berupa data biner yang ringkas karena tidak menyertakan nama field, hanya nomor field dan nilainya.
gRPC adalah kerangka kerja Remote Procedure Call yang berjalan di atas HTTP/2 dan memakai Protobuf sebagai format pesan.
Docker menjalankan aplikasi dalam kontainer terisolasi. Kontainer pada network bridge yang sama dapat saling memanggil menggunakan nama kontainer.
3. Alat dan Lingkungan
Komponen	Keterangan
Sistem operasi	Windows dengan WSL2 (Ubuntu)
Container engine	Docker Desktop (WSL 2 based engine)
Bahasa pemrograman	Python 3.11 (di dalam kontainer)
Pustaka	grpcio, grpcio-tools, protobuf
Editor	Visual Studio Code
4. Struktur Folder
praktikum-protobuf-grpc/
├── protos/
│   └── exchange.proto
├── server-a-sender/
│   ├── Dockerfile
│   ├── main.py
│   ├── exchange_pb2.py
│   └── exchange_pb2_grpc.py
├── server-b-receiver/
│   ├── Dockerfile
│   ├── main.py
│   ├── exchange_pb2.py
│   └── exchange_pb2_grpc.py
└── screenshots/
5. Langkah Kerja
5.1 Persiapan Lingkungan

WSL2 dengan distro Ubuntu dipasang dengan perintah wsl --install -d Ubuntu. Docker Desktop dipasang dan fitur WSL Integration diaktifkan untuk distro Ubuntu agar perintah docker dapat dipakai dari terminal Ubuntu.

Show Image

5.2 Pembuatan Struktur Direktori
bash
mkdir -p ~/protobuf-lab/protos ~/protobuf-lab/server-a-sender ~/protobuf-lab/server-b-receiver

Show Image

5.3 Skema Protobuf

File protos/exchange.proto berisi kontrak data. Pesan StudentRequest memiliki tiga field (NIM, nama, dan timestamp pengiriman dalam nanodetik), pesan StudentResponse berisi status konfirmasi, dan layanan ExchangeService mendefinisikan satu fungsi RPC bernama SendData.

protobuf
syntax = "proto3";

package exchange;

message StudentRequest {
    string student_id = 1;
    string student_name = 2;
    int64 sent_timestamp_ns = 3;
}

message StudentResponse {
    bool status = 1;
}

service ExchangeService {
    rpc SendData (StudentRequest) returns (StudentResponse);
}

Show Image

5.4 Server B (Penerima)

Server B adalah server gRPC yang mendengarkan pada port 50051. Setiap kali menerima permintaan SendData, server mencatat waktu kedatangan, menghitung waktu transmisi (selisih waktu kedatangan dan sent_timestamp_ns), membaca ukuran payload dengan ByteSize(), menampilkannya di layar, lalu membalas dengan status=True.

5.5 Server A (Pengirim)

Server A adalah klien gRPC interaktif. Program meminta NIM dan nama dari terminal, membuat objek StudentRequest beserta timestamp pengiriman, menampilkan ukuran hasil serialisasi, lalu memanggil SendData ke alamat receiver-container:50051. Mengetik exit pada bagian NIM mengakhiri program.

5.6 Kompilasi Protobuf dan Pembuatan Image

Kompilasi dilakukan dengan grpcio-tools di dalam virtual environment Python, karena Ubuntu versi baru menolak pip install langsung ke sistem.

bash
python3 -m venv .venv
source .venv/bin/activate
pip install grpcio-tools

python -m grpc_tools.protoc -I./protos --python_out=./server-a-sender --grpc_python_out=./server-a-sender ./protos/exchange.proto
python -m grpc_tools.protoc -I./protos --python_out=./server-b-receiver --grpc_python_out=./server-b-receiver ./protos/exchange.proto

Hasilnya adalah file exchange_pb2.py dan exchange_pb2_grpc.py pada kedua folder aplikasi.

Show Image

Selanjutnya dibuat network Docker dan kedua image:

bash
docker network create integration-net
docker build -t image-server-b ./server-b-receiver
docker build -t image-server-a ./server-a-sender

Show Image

5.7 Eksekusi dan Verifikasi

Terminal 1, menjalankan Server B (penerima):

bash
docker run --rm --name receiver-container --network integration-net -p 50051:50051 image-server-b

Terminal 2, menjalankan Server A (pengirim):

bash
docker run --rm -it --name sender-container --network integration-net image-server-a

Tiga data dikirim dari Server A. Dua data memakai nama Eugenius Arlanda Wangkur, dan satu data memakai penulisan Eugenius Arlanda wangkur.

Server A (pengirim):

Show Image

Server B (penerima):

Show Image

6. Hasil Pengamatan
Transaksi	NIM	Nama	Ukuran Payload	Waktu Transmisi
1	2415354072	Eugenius Arlanda Wangkur	48 bytes	17,7704 ms
2	2415354072	Eugenius Arlanda Wangkur	48 bytes	1,4839 ms
3	2415354072	Eugenius Arlanda wangkur	48 bytes	1,1338 ms

Seluruh transaksi menerima konfirmasi (Ack) dari Server B, yang menunjukkan komunikasi gRPC antar kontainer berhasil.

7. Analisis
7.1 Ukuran Payload

Setiap pesan berukuran 48 bytes. Rinciannya mengikuti aturan pengodean Protobuf, di mana setiap field terdiri dari tag, panjang (untuk string), dan isi.

Field	Rincian	Ukuran
student_id	tag 1 byte + panjang 1 byte + 10 karakter	12 bytes
student_name	tag 1 byte + panjang 1 byte + 24 karakter	26 bytes
sent_timestamp_ns	tag 1 byte + varint nanodetik (sekitar 9 byte)	10 bytes
Total		48 bytes

Ketiga transaksi berukuran sama karena panjang NIM dan nama sama. Perbedaan huruf besar dan kecil pada transaksi ketiga tidak mengubah ukuran, sebab setiap karakter ASCII berukuran 1 byte. Protobuf tidak mengirim nama field (seperti student_id), hanya nomor field, sehingga hasilnya lebih ringkas dibandingkan format teks seperti JSON. Ukuran pesan akan berubah mengikuti panjang NIM dan nama yang dimasukkan.

7.2 Waktu Transmisi

Waktu transmisi pertama (17,7704 ms) jauh lebih tinggi dibanding transaksi kedua (1,4839 ms) dan ketiga (1,1338 ms), sekitar 12 sampai 15 kali lebih lambat. Penyebabnya adalah koneksi gRPC pada panggilan pertama baru dibuat, mencakup resolusi nama kontainer dan pembentukan koneksi HTTP/2. Karena start_time diambil sebelum pemanggilan SendData, waktu pembentukan koneksi ikut terhitung pada transaksi pertama. Pada transaksi berikutnya koneksi yang sudah terbuka dipakai kembali, sehingga waktu transmisi turun dan stabil di kisaran 1 ms.

7.3 Validitas Pengukuran

Waktu transmisi dihitung dari selisih jam pengirim dan penerima. Pengukuran ini valid karena kedua kontainer berjalan di satu host fisik dan memakai sumber waktu yang sama. Pada dua mesin berbeda, selisih jam (clock skew) membuat pengukuran satu arah seperti ini tidak akurat. Nilai di atas juga hasil satu kali percobaan sehingga dapat bervariasi pada percobaan lain.

8. Kesimpulan
Pertukaran data biner terstruktur antar kontainer Docker berhasil dilakukan menggunakan Protobuf dan gRPC. Seluruh transaksi diterima dan dikonfirmasi oleh Server B.
Skema .proto berperan sebagai kontrak data yang dikompilasi menjadi kode Python, sehingga pengirim dan penerima memakai struktur data yang sama.
Ukuran payload 48 bytes menunjukkan serialisasi Protobuf yang ringkas, karena hanya nomor field dan nilainya yang dikirim.
Waktu transmisi pertama (17,7704 ms) lebih tinggi karena adanya pembentukan koneksi, sedangkan transaksi berikutnya stabil di kisaran 1 ms.