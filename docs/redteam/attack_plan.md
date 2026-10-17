# Attack Plan — Pengujian Keamanan Backend IoT

**Kelompok 10 | TEK1314 — Keamanan Siber | Fase 2**

## 1. Tujuan

Pengujian bertujuan mengevaluasi keamanan replika backend sistem autodosing nutrisi AB Mix tanaman melon, khususnya paparan layanan, pembatasan autentikasi SSH, dan perlindungan data melalui API. Dokumen ini memuat **rencana tiga skenario dari tiga kategori berbeda**; hasil pelaksanaan akan dicatat pada log Purple Team.

## 2. Lingkungan Pengujian

| Komponen | Keterangan |
|---|---|
| Attacker | CyberOps Workstation — `192.168.10.100` |
| Target | Ubuntu Server — `192.168.10.5` |
| Monitoring | Security Onion — IP manajemen `192.168.10.200` |
| Layanan yang diuji | SSH port `22`, Nginx port `80`, dan pembatasan akses database port `9000` |
| Aplikasi | Dashboard HTML/JavaScript dengan endpoint baca `/api/sensor` |
| Database | Firebase Realtime Database Emulator — `127.0.0.1:9000` |
| Batas lingkungan | Jaringan Host-Only dan data dummy; tanpa koneksi ke perangkat IoT atau sistem produksi |

## 3. Rencana Skenario

| ID | Kategori | Skenario dan Tools | Target | Hasil yang Diharapkan |
|---|---|---|---|---|
| RT-01 | Reconnaissance | Pemindaian port dan identifikasi layanan menggunakan **Nmap**, disertai pemeriksaan header HTTP menggunakan **curl** | TCP `22`, `80`, dan `9000` pada `192.168.10.5` | Port SSH dan HTTP tetap tersedia bagi CyberOps sesuai izin; database tidak dapat diakses langsung dari jaringan; header Nginx tidak menampilkan nomor versi. |
| RT-02 | Exploitation / Akses | Simulasi percobaan autentikasi SSH gagal secara terbatas menggunakan **OpenSSH client** | Akun uji `iotuser` pada SSH port `22` | Fail2ban memblokir IP penguji setelah ambang kegagalan tercapai; koneksi SSH baru ditolak selama masa blokir. |
| RT-03 | Web/API/Protokol IoT | Percobaan perubahan data sensor melalui metode PATCH menggunakan **curl** | Endpoint `http://192.168.10.5/api/sensor` | GET menghasilkan HTTP `200`, PATCH ditolak Nginx dengan HTTP `403`, dan data sensor tidak berubah. |

### RT-01 — Pemetaan Layanan

Skenario ini didasarkan pada temuan awal bahwa port database dapat dijangkau dari CyberOps. Red Team akan memeriksa kembali port yang tersedia setelah penerapan UFW dan pembatasan database ke localhost. Blue Team membandingkan hasil pemindaian dengan konfigurasi firewall dan listener, sedangkan Security Onion digunakan untuk mengamati trafik serta alert yang tersedia.

### RT-02 — Percobaan Autentikasi Berulang

Skenario ini mengevaluasi potensi tebakan kredensial SSH. Pengujian dilakukan secara manual dengan satu akun uji dan password salah, maksimal lima percobaan dalam sepuluh menit, serta dihentikan apabila pemblokiran telah terjadi. Blue Team memverifikasi daftar blokir Fail2ban dan log autentikasi Ubuntu. Setelah bukti diambil, akses dipulihkan melalui konsol dan login menggunakan kredensial benar diuji kembali.

### RT-03 — Percobaan Manipulasi Data API

Skenario ini mengevaluasi potensi perubahan data melalui endpoint dashboard yang seharusnya hanya menyediakan akses baca. Simulator dihentikan sementara agar data awal dan akhir dapat dibandingkan. Red Team membaca data, mengirim PATCH dengan nilai kelembapan dummy yang valid, lalu membaca ulang data. Blue Team memeriksa respons serta log Nginx. Apabila perubahan diterima, data dikembalikan ke nilai awal dan pembatasan metode diperbaiki sebelum pengujian ulang.

Dasar pemilihan skenario merujuk pada layanan dan fokus pengujian dalam [IP plan](../design/ip_plan.md) serta temuan implementasi pada [checklist hardening](../blueteam/hardening_checklist.md). Target pengujian disesuaikan dengan konfigurasi terkini: SSH port `22`, Nginx port `80`, dan pembatasan akses langsung ke database port `9000`.

## 4. Pengamatan dan Evaluasi

Setiap skenario mengikuti lima tahap Purple Team: **pengujian pada kondisi saat ini → deteksi → analisis → respons/mitigasi → pengujian ulang**. Mitigasi ditentukan berdasarkan temuan, kemudian skenario yang sama diulang untuk menilai efektivitas perbaikan.

- **Red Team:** menjalankan pengujian dan mencatat hasil aktual.
- **Blue Team:** menganalisis log, memverifikasi kontrol, menerapkan perbaikan, dan memulihkan layanan.
- **Monitoring:** mengamati trafik dan alert melalui Security Onion; kegagalan autentikasi SSH diverifikasi melalui log Ubuntu karena isi koneksi SSH terenkripsi.
- **Lead:** mengoordinasikan pelaksanaan dan kelengkapan dokumentasi.

Bukti berupa tangkapan layar disimpan di `docs/redteam/evidence/`. Analisis dan hasil lima tahap dicatat pada [docs/purple-team-log.md](../purple-team-log.md). Apabila alert tidak muncul, kondisi tersebut dicatat sebagai temuan, bukan dianggap telah terdeteksi.

## 5. Batasan dan Status

Pengujian hanya dilakukan pada Target replika menggunakan data dummy, tanpa DoS, malware, atau tindakan terhadap sistem mitra. Konfigurasi hardening yang telah aktif tidak dinonaktifkan untuk membuat serangan berhasil. Pengujian dihentikan jika layanan tidak stabil atau akses pemulihan tidak tersedia.

**Status seluruh skenario: Direncanakan.** Keberhasilan pemblokiran, deteksi, dan pemulihan akan dinilai berdasarkan hasil eksekusi, bukan hanya hasil yang diharapkan dalam dokumen ini.
