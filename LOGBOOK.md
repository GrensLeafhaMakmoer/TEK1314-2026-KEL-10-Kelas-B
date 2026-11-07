# Logbook PBL — Kelompok 10 Kelas B

**Mata kuliah:** TEK1314 — Keamanan Siber  
**Proyek IoT:** Sistem autodosing nutrisi AB Mix pada tanaman melon  
**Cakupan laporan:** Progres sampai checkpoint pertemuan 11 (Fase 2)

## 1. Anggota dan Pembagian Peran

| Anggota | NIM | Peran | Tanggung Jawab |
|---|---|---|---|
| Ghazali Habibie A | J0404241145 | Lead | Koordinasi kelompok, konsistensi dokumentasi, dan persiapan presentasi. |
| Gregorius Gil Ovidio Sebayang | J0404241088 | Red Team | Identifikasi layanan, penyusunan skenario pengujian, eksekusi serangan, dan pencatatan hasil. |
| Binggie Rashel Prasetyo | J0404241009 | Blue Team | Konfigurasi pertahanan, monitoring, analisis log, dokumentasi bukti, dan pemulihan baseline. |

## 2. Riwayat Kegiatan

| Minggu | Kegiatan yang Tercatat | Hasil / Keterangan |
|---|---|---|
| 1 | Belum tersedia catatan kegiatan terpisah. | Tidak direkonstruksi tanpa catatan pendukung. |
| 2 | Pembentukan kelompok dan pembagian peran; instalasi VirtualBox, pengunduhan ISO, serta pembuatan repository GitHub. | Lingkungan virtualisasi dan repository disiapkan untuk pekerjaan kelompok. Attacker yang digunakan pada implementasi adalah CyberOps Workstation. |
| 3 | Latihan dasar Linux dan identifikasi awal kebutuhan layanan backend. | Evaluasi kandidat layanan; implementasi replika difokuskan pada HTTP REST/JSON untuk API dan dashboard. |
| 4 | Perancangan topologi tiga node: Attacker, Target, dan Monitoring, beserta skema IP. | Topologi host-only `192.168.10.0/24` ditetapkan: Target `.5`, Attacker `.100`, Security Onion `.200`, Host `.1`. |
| 5–9 | Pembangunan backend replika (Firebase emulator, simulator Python, Nginx) dan penerapan hardening dasar (8 kontrol). | Layanan dummy berjalan normal; dokumentasi checklist hardening awal dan baseline trafik Minggu 9 selesai. |
| 10 | Pengembangan checklist hardening lanjutan menjadi 14 kontrol dan penyusunan detail attack plan Fase 2. | Hardening diperluas: autentikasi ganda SSH (key + password), rate limiting Nginx, security headers, dan proteksi dotfiles. |
| 11 | Eksekusi penuh skenario Purple Team (RT-01, RT-02, RT-03), pengumpulan bukti log autentik, dan penyusunan log Purple Team. | Siklus 5 tahap Purple Team berhasil dijalankan dan diverifikasi; key sementara dibersihkan; dokumen `docs/purple-team-log.md` diselesaikan. |

Kegiatan detail sampai checkpoint minggu 9 dirangkum pada bagian 4, dan eksekusi Fase 2 pada bagian 5. 

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

### 4.4. Rencana Pengujian Red Team Fase 2 (Arsip Rencana)

Pada checkpoint minggu 9, [attack plan](docs/redteam/attack_plan.md) telah dirancang untuk mencakup tiga skenario dari kategori berbeda:

| ID | Kategori | Rencana dan Tools Awal |
|---|---|---|
| RT-01 | Reconnaissance | Pemeriksaan port/layanan dan header HTTP menggunakan Nmap serta curl. |
| RT-02 | Exploitation / Akses | Percobaan autentikasi SSH gagal secara manual menggunakan OpenSSH client untuk mengevaluasi Fail2ban. |
| RT-03 | Web/API | Percobaan PATCH melalui curl pada endpoint `/api/sensor` untuk mengevaluasi penolakan modifikasi data. |

Rencana pengujian ini telah sepenuhnya direalisasikan dan dieksekusi pada Fase 2 (pertemuan 11). Detail pelaksanaan dan temuan dicatat pada bagian 5 dan didokumentasikan lengkap pada [Log Purple Team](docs/purple-team-log.md).

## 5. Progres Fase 2 (Minggu 10–11) — Eksekusi Purple Team

### 5.1. Peningkatan Hardening Target (14 Kontrol)

Pada minggu 10, tim Blue Team memperluas kontrol keamanan dari 8 kontrol awal menjadi 14 kontrol terverifikasi yang didokumentasikan dalam [hardening checklist](docs/blueteam/hardening_checklist.md):
- **A1–A4 (Access & Authentication):** Penonaktifan login root SSH, integrasi Fail2ban, pembatasan durasi login (`LoginGraceTime 30`), dan penerapan autentikasi ganda wajib public key + password (`AuthenticationMethods publickey,password`).
- **B1–B6 (Service & Attack Surface):** Firewall UFW aktif, binding database internal ke `127.0.0.1:9000`, penyembunyian banner versi Nginx (`server_tokens off`), rate limiting API Nginx (`limit_req_zone`), penambahan security headers (`X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`), serta pencegahan directory listing (autoindex off) dan pemblokiran akses file tersembunyi (`.*`).
- **C1–C5 (Data & Application):** Aturan validasi schema pada backend, permission file aplikasi yang ketat (`chmod/chown`), dan prosedur backup-restore database dummy.

### 5.2. Eksekusi Penuh Siklus 5 Tahap Purple Team Loop

Pada tanggal 7 November 2026, tim melaksanakan pengujian terpadu Red Team dan Blue Team secara berurutan:

| Skenario | Hasil Serangan (Red Team) | Deteksi & Pertahanan (Blue Team) | Evaluasi & Status Loop |
|---|---|---|---|
| **RT-01 (Reconnaissance)** | Nmap mendeteksi port `22` dan `80` terbuka; port `9000` **filtered**. Curl header mengonfirmasi server tanpa nomor versi, `X-Frame-Options: DENY`, dan akses `/dashboard.html` dialihkan (`303`) ke login. | UFW berhasil memfilter port internal database. Direktif `server_tokens off` mencegah footprinting versi web server secara pasif. | **Selesai (Lengkap).** Kontrol B1, B2, B3, dan B5 terbukti efektif. Permukaan serang terminimalisasi. |
| **RT-02 (Akses SSH / Brute Force)** | Percobaan login tanpa key langsung ditolak (`Permission denied (publickey)`). Pada pengujian key sementara, 5 tebakan password salah ditolak berturut-turut, lalu koneksi diputus oleh target. | `/var/log/auth.log` mencatat 5 baris `Failed password for iotuser`. Fail2ban mendeteksi insiden, lalu mencatat `Ban 192.168.10.100` pada `/var/log/fail2ban.log` selama 120 detik hingga `Unban`. | **Selesai (Lengkap).** Kontrol A2 dan A4 terbukti efektif. Key Red Team segera dihapus pasca uji; baseline 2 key sah dipulihkan. |
| **RT-03 (Manipulasi Data API)** | Permintaan data tanpa sesi menghasilkan HTTP `401 Unauthorized`. Percobaan modifikasi payload sensor (`PATCH /api/sensor` dengan `{"humidity":55}`) langsung ditolak dengan HTTP `403 Forbidden`. | `/var/log/nginx/access.log` mencatat request penyerang lengkap dengan timestamp, metode, kode status `401` dan `403`, serta User-Agent curl. | **Selesai (Lengkap).** Kontrol B6 (`limit_except GET`) mencegah manipulasi data sewenang-wenang. Integritas database dummy tetap aman. |

### 5.3. Pengumpulan Bukti Log dan Screenshot Autentik

Seluruh hasil pengujian dan log sistem diambil langsung dari console VirtualBox (`SRV-IOT-KEL10` dan `CYBEROPS-KEL10`) tanpa rekayasa data:
- `docs/redteam/evidence/RT-01/rt01-nmap.png` & `rt01-header.png`: Bukti output Nmap dan header HTTP respons.
- `docs/redteam/evidence/RT-02/rt02-ssh-tanpa-key.png`: Bukti penolakan SSH tanpa public key.
- `docs/redteam/evidence/RT-02/rt02-log-auth.png` & `rt02-log-fail2ban.png`: Bukti pencatatan insiden pada `auth.log` dan aksi penindakan Fail2ban.
- `docs/redteam/evidence/RT-03/rt03-curl-401-403.png`: Bukti respons penolakan HTTP 401 dan 403 dari terminal CyberOps.
- `docs/redteam/evidence/RT-03/rt03-log-nginx-403.png`: Bukti catatan akses berbahaya pada log web server Nginx.
- `docs/redteam/evidence/so-sguil-alerts.png`: Bukti status aktif konsol monitoring Sguil pada Security Onion dengan timestamp tersinkronisasi.

Seluruh detail pengamatan, korelasi waktu (WIB vs UTC), analisis arsitektur, dan rekomendasi dirangkum dalam dokumen utama **[Log Purple Team](docs/purple-team-log.md)**.

## 6. Kendala dan Tindak Lanjut

| Kendala / Temuan | Penanganan / Tindak Lanjut | Status |
|---|---|---|
| Simulator menerima HTTP `401 Permission denied` (Fase 1). | Penyesuaian rules namespace Firebase emulator dan restart service simulator. | Tuntas |
| Port `9000` dipakai proses Java lama (Fase 1). | Identifikasi PID proses Java yang tertinggal dan restart proses secara bersih. | Tuntas |
| Kontrol SSH A4 (`publickey,password`) menolak brute-force password murni secara langsung. | Pengujian RT-02 dibagi menjadi dua tahap: (1) bukti penolakan tanpa key, dan (2) pemasangan key sementara CyberOps untuk menguji batas gagal password Fail2ban. Key sementara segera dihapus setelah pengujian. | Tuntas (Baseline kembali ke 2 key sah) |
| Simulator memperbarui data secara periodik sehingga nilai numerik sensor terus bergerak. | Pembuktian integritas data pada RT-03 difokuskan pada penolakan HTTP `403 Forbidden` di Nginx dan tidak adanya modifikasi data dari penyerang luar. | Tuntas |
| Konsol Security Onion (`SO-KEL10`) sempat terkunci screensaver. | Pengguna membuka sesi konsol; tangkapan layar antarmuka Sguil (monitoring NIDS) berhasil diambil dan didokumentasikan ke repositori. | Tuntas |

## 7. Rencana Berikutnya

1. Menyusun Vulnerability Assessment Report (VAR) dan materi presentasi evaluasi Fase 2 berbasis data logbook dan purple team log.
2. Melakukan sinkronisasi repositori git kelompok ke remote GitHub (`origin/main`).
3. Menjaga kestabilan VM target lab dan mempertahankan konfigurasi keamanan yang sudah teruji.
