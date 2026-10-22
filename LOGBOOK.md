# Logbook PBL — Kelompok 10 Kelas B

**Mata kuliah:** TEK1314 — Keamanan Siber  
**Proyek IoT:** Sistem autodosing nutrisi AB Mix pada tanaman melon  
**Cakupan laporan:** Progres sampai checkpoint pertemuan 9

## 1. Anggota dan Pembagian Peran

| Anggota | NIM | Peran | Tanggung Jawab |
|---|---|---|---|
| Ghazali Habibie A | J0404241145 | Lead | Koordinasi kelompok, konsistensi dokumentasi, dan persiapan presentasi. |
| Gregorius Gil Ovidio Sebayang | J0404241088 | Red Team | Identifikasi layanan, penyusunan skenario pengujian, dan pencatatan hasil pengujian. |
| Binggie Rashel Prasetyo | J0404241009 | Blue Team | Konfigurasi pertahanan, monitoring, analisis log, dan dokumentasi bukti. |

## 2. Riwayat Kegiatan Awal

| Minggu | Kegiatan yang Tercatat | Hasil / Keterangan |
|---|---|---|
| 1 | Belum tersedia catatan kegiatan terpisah. | Tidak direkonstruksi tanpa catatan pendukung. |
| 2 | Pembentukan kelompok dan pembagian peran; instalasi VirtualBox, pengunduhan ISO, serta pembuatan repository GitHub. | Lingkungan virtualisasi dan repository disiapkan untuk pekerjaan kelompok. Catatan awal mencantumkan Kali; attacker yang digunakan pada implementasi saat ini adalah CyberOps. |
| 3 | Latihan dasar Linux dan identifikasi awal kebutuhan layanan backend. | MQTT `1883`, HTTP `80`, FTP `21`, dan SSH `22` sempat menjadi kandidat layanan. Implementasi replika selanjutnya menggunakan HTTP REST/JSON, bukan MQTT atau FTP. |
| 4 | Perancangan topologi tiga node: Attacker, Target, dan Monitoring, beserta skema IP. | Dokumen desain menjadi dasar implementasi. Referensi subnet `.12`, KEL12, dan Arch Linux pada catatan lama tidak sesuai lab Kelompok 10; konfigurasi acuan terkini dicantumkan pada bagian 3. |

Kegiatan lanjutan sampai checkpoint minggu 9 dirangkum pada bagian 4. 

## 3. Lingkungan Replika yang Digunakan

Pengujian dibatasi pada jaringan VirtualBox Host-Only `192.168.10.0/24`, menggunakan data dummy dan tanpa pengendalian pompa atau pengujian terhadap sistem mitra.

| Komponen | Implementasi |
|---|---|
| Attacker | CyberOps Workstation — `192.168.10.100` |
| Target | Ubuntu Server `srv-iot-kel10` — `192.168.10.5` |
| Monitoring | Security Onion — IP manajemen `192.168.10.200`; interface sensor `eth0` |
| Host administrasi | Laptop — `192.168.10.1` |
| Database | Firebase Realtime Database Emulator, namespace `demo-melon-lab`; kebijakan listener `127.0.0.1:9000` |
| Komunikasi dan dashboard | HTTP REST/JSON; simulator Python dan dashboard HTML/JavaScript melalui Nginx port `80` |

Sistem asli menggunakan Firebase cloud, HTTPS, dan aplikasi Flutter. Replika menggunakan emulator lokal, HTTP tanpa TLS, dan dashboard web sederhana. Perbedaan tersebut dinyatakan dalam [deklarasi tech stack](docs/presentation-minggu9/tech_stack_declaration.md), bukan diklaim sebagai replika identik.

Referensi desain: [topologi jaringan](docs/design/topology.png) dan [IP plan](docs/design/ip_plan.md).

## 4. Progres sampai Checkpoint Minggu 9

### 4.1. Implementasi Backend Replika

- Menyiapkan Ubuntu Target dengan Firebase Realtime Database Emulator, simulator Python, dan Nginx untuk menyajikan dashboard serta proxy baca API.
- Menggunakan data dummy suhu, kelembapan, EC, dan waktu pembaruan pada `/lab_dummy/sensor`.
- Mengembangkan penyimpanan riwayat pada `/lab_dummy/history` dengan batas 10 entri yang diterapkan oleh simulator. Simulator telah dilaporkan kembali berjalan setelah penanganan konflik port dan penyesuaian rules.
- Bukti capture jaringan memperlihatkan pembacaan `/api/sensor` dan `/api/history` beserta respons HTTP `200 OK`. Bukti tampilan akhir dashboard belum tercantum dalam folder baseline.
- Kode tambahan PPM estimasi dengan konversi EC × 500 telah disiapkan; pemasangan dan hasil tampilan akhirnya belum diverifikasi dalam bukti repository.

### 4.2. Hardening Target

Delapan kontrol telah didokumentasikan dalam [checklist hardening](docs/blueteam/hardening_checklist.md), dengan status dan batasan verifikasi pada masing-masing kontrol.

| Kategori | Kontrol yang Didokumentasikan | Hasil Utama |
|---|---|---|
| A — Access & Authentication | Larangan login root SSH dan Fail2ban. | Login root ditolak; IP penguji masuk daftar blokir Fail2ban pada pengujian terbatas. |
| B — Service & Attack Surface | UFW, binding database ke localhost, dan `server_tokens off` pada Nginx. | Bukti hardening menunjukkan pembatasan port database, listener loopback, dan penghilangan nomor versi Nginx. |
| C — Data & Application | Security Rules/validasi data, backup–restore, dan permission file aplikasi. | Payload kelembapan di luar rentang ditolak; ekspor backup berhasil dan nilai uji kembali dari 80 ke 72; akses baca file oleh akun `nobody` ditolak. |

### 4.3. Baseline Trafik dan Monitoring

- Mendokumentasikan status layanan Security Onion yang menunjukkan `[OK]` pada komponen yang ditampilkan.
- Mengamati ping CyberOps ke Ubuntu Target dan event `GPL ICMP_INFO PING *NIX` pada Sguil.
- Mengamati event kebijakan berawalan `ET POLICY curl User-Agent ...` pada Sguil serta trafik HTTP kedua endpoint melalui tcpdump.

Ringkasan dan tautan screenshot tersedia pada [baseline trafik](docs/blueteam/baseline_traffic.md). Korelasi waktu antar-bukti perlu diverifikasi sebelum dianggap berasal dari satu sesi pengamatan yang sama.

### 4.4. Rencana Pengujian Red Team Fase 2

[Attack plan](docs/redteam/attack_plan.md) telah disusun dengan tiga skenario dari tiga kategori berbeda:

| ID | Kategori | Rencana dan Tools |
|---|---|---|
| RT-01 | Reconnaissance | Pemeriksaan port/layanan dan header HTTP menggunakan Nmap serta curl. |
| RT-02 | Exploitation / Akses | Percobaan autentikasi SSH gagal secara manual dan terbatas menggunakan OpenSSH client untuk mengevaluasi Fail2ban. |
| RT-03 | Web/API | Percobaan PATCH melalui curl pada endpoint baca `/api/sensor` untuk mengevaluasi penolakan perubahan data. |

Ketiga skenario masih berstatus **direncanakan** dalam attack plan. Pengujian kontrol yang sudah terdokumentasi tidak disamakan dengan penyelesaian lima tahap Purple Team. [Log Purple Team](docs/purple-team-log.md) belum berisi hasil pelaksanaan lengkap.

## 5. Kendala dan Tindak Lanjut

| Kendala / Temuan | Penanganan / Tindak Lanjut |
|---|---|
| Simulator menerima HTTP `401 Permission denied`. | Memeriksa namespace dan rules sensor/riwayat. Setelah penyesuaian dan restart emulator, simulator dilaporkan kembali berjalan. |
| Port `9000` dipakai proses Java emulator lama. | Proses pemakai port diidentifikasi sebelum prosedur penghentian dan menjalankan ulang emulator; risiko kehilangan data tanpa ekspor diperhatikan. |
| Blok `location /api/history` Nginx sempat bersarang. | Struktur diarahkan menjadi blok sejajar dan diuji dengan `nginx -t`; bukti berikutnya memperlihatkan respons HTTP `200` untuk kedua endpoint. |
| Jam CyberOps dan Security Onion tidak selaras. | Sinkronkan jam dengan acuan yang sama, lalu ulangi baseline dan dokumentasikan waktu pengamatan. |
| Output Nmap dalam bukti baseline masih menunjukkan port `9000` open, berbeda dari bukti hardening. | Verifikasi ulang listener, UFW, dan keterjangkauan database dari CyberOps sebelum menyimpulkan kondisi akhir. |


## 6. Rencana Berikutnya

1. Menyelesaikan slide PDF checkpoint, latihan pembagian presentasi, serta demo monitoring trafik normal.
2. Menjalankan tiga skenario attack plan hanya pada replika, lalu mencatat lima tahap: pengujian awal, deteksi, analisis, respons, dan verifikasi ulang.
3. Menyusun Vulnerability Assessment Report untuk minggu 11 berdasarkan hasil aktual
