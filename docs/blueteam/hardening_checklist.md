# Checklist Hardening Keamanan — Ubuntu Server

**Kelompok 10 | TEK1314 — PBL Keamanan Siber | Checkpoint Minggu 9**

## 1. Tujuan dan Ruang Lingkup

Dokumen ini menyajikan penerapan dan hasil verifikasi delapan kontrol hardening pada replika sistem monitoring nutrisi AB Mix tanaman melon. Kontrol dikelompokkan ke dalam kategori Access & Authentication, Service & Attack Surface, serta Data & Application.

- Target: Ubuntu Server `srv-iot-kel10`, IP lab `192.168.10.5`.
- Perangkat penguji: CyberOps Workstation `192.168.10.100`; host administrasi pada aturan UFW: `192.168.10.1`.
- Layanan: SSH, Nginx untuk dashboard HTML/JavaScript, Firebase Realtime Database Emulator, dan simulator Python.
- Project demo / namespace aplikasi: `demo-melon-lab`; data dummy: `/lab_dummy/sensor`.
- Dokumentasi bukti: tangkapan layar pada direktori `docs/blueteam/evidence/`, ditautkan pada setiap kontrol.

Seluruh implementasi dan pengujian dilakukan pada lingkungan laboratorium menggunakan data dummy. Ruang lingkup tidak mencakup Firebase cloud, sensor fisik, maupun pengendalian pompa. Komunikasi REST pada replika menggunakan HTTP dan belum dilengkapi TLS/HTTPS.

### Kriteria Status

- **Done**: kontrol telah diterapkan dan didukung bukti konfigurasi atau hasil pengujian sesuai lingkup verifikasi yang dijelaskan.
- **Partial**: implementasi atau verifikasi kontrol baru terpenuhi sebagian.
- **Not Yet**: kontrol belum diterapkan atau belum memiliki bukti verifikasi.

## 2. Ringkasan Implementasi

| ID | Kategori | Kontrol | Status | Hasil Verifikasi |
|---|---|---|---|---|
| A1 | Access & Authentication | Menolak login SSH langsung sebagai root | Done | Konfigurasi efektif `permitrootlogin no`; percobaan root ditolak. |
| A2 | Access & Authentication | Pembatasan percobaan autentikasi SSH menggunakan Fail2ban | Done | Jail aktif, IP `192.168.10.100` masuk daftar blokir, dan koneksi SSH baru ditolak. |
| B1 | Service & Attack Surface | Membatasi akses masuk menggunakan UFW | Done | SSH/HTTP dibatasi ke IP lab yang ditentukan; port 9000 dari CyberOps berubah menjadi filtered. |
| B2 | Service & Attack Surface | Mengikat database ke localhost | Done | Listener database berubah dari IP jaringan menjadi `127.0.0.1:9000`. |
| B3 | Service & Attack Surface | Mengurangi informasi versi Nginx pada respons | Done | Header berubah dari `nginx/1.18.0 (Ubuntu)` menjadi `nginx`; halaman dashboard merespons HTTP 200. |
| C1 | Data & Application | Security Rules dan validasi payload sensor | Done | Data valid HTTP 200; `humidity:150` ditolak HTTP 401 setelah namespace rules diselaraskan. |
| C2 | Data & Application | Backup manual dan verifikasi pemulihan data | Done | Ekspor berhasil; setelah perubahan ke 80, nilai kelembapan kembali ke 72 pada verifikasi pemulihan. |
| C3 | Data & Application | Membatasi akses file konfigurasi dan data aplikasi | Done | Permission direktori aplikasi menjadi `700` dan tiga file menjadi `600`; akses baca pemilik diizinkan, sedangkan akun `nobody` ditolak. |

**Rekapitulasi: 8 Done, 0 Partial, dan 0 Not Yet.** Implementasi terdiri atas **2 kontrol kategori A, 3 kontrol kategori B, dan 3 kontrol kategori C**, sehingga memenuhi jumlah minimal delapan kontrol dengan sedikitnya dua kontrol pada setiap kategori. Status dinilai berdasarkan bukti yang tersedia dan dibatasi oleh cakupan verifikasi masing-masing kontrol; status tersebut tidak menyatakan sistem telah aman untuk penggunaan produksi.

Rekapitulasi tersebut berlaku untuk **delapan kontrol awal A1–C3**. Empat kontrol lanjutan pada bagian 6 masih berstatus **Not Yet (rencana; belum terverifikasi)** dan tidak dihitung sebagai implementasi selesai. Keseluruhan daftar memuat **12 kontrol: 8 Done, 0 Partial, dan 4 Not Yet**. Kekurangan bukti pada kontrol awal tetap dicatat pada uraian masing-masing dan tidak dihitung ulang sebagai kontrol baru.

## 3. Kategori A — Access & Authentication

### A1. Menonaktifkan login root langsung melalui SSH — Done

- **Tujuan:** mencegah autentikasi SSH langsung menggunakan akun root dan mempertahankan administrasi melalui akun pengguna yang berwenang.
- **Kondisi awal (Before):** pemeriksaan awal menunjukkan `without-password`, yang melarang autentikasi password root tetapi masih memungkinkan metode non-password. Untuk demonstrasi terkontrol, login root sempat diaktifkan dengan `PermitRootLogin yes`. Bukti Before yang dilampirkan merekam kondisi demonstrasi tersebut, **bukan baseline awal atau konfigurasi bawaan Ubuntu**.
- **Implementasi:** menetapkan `PermitRootLogin no` pada `/etc/ssh/sshd_config.d/00-pbl-hardening.conf`, memvalidasi sintaks, dan memuat ulang layanan SSH.
- **Hasil verifikasi (After):** konfigurasi efektif menunjukkan `permitrootlogin no` dan percobaan login root baru dari CyberOps ditolak. Pemeriksaan dilakukan menggunakan `sudo /usr/sbin/sshd -t`, `sudo /usr/sbin/sshd -T`, serta `ssh root@192.168.10.5`.
- **Cakupan dan batasan:** kontrol membatasi login SSH langsung, bukan peningkatan hak akses melalui `sudo` oleh pengguna yang berwenang. Bukti akses SSH akun biasa tercantum pada A2.

Bukti:

- [Konfigurasi demonstrasi BEFORE](<evidence/kontrol A1 — SSH permitlogin/ssh-before-hardening.png>)
- [Login root pada demonstrasi terkontrol](<evidence/kontrol A1 — SSH permitlogin/hasil-before-hardening.png>)
- [File konfigurasi hardening](<evidence/kontrol A1 — SSH permitlogin/afterhardening-ssh-1.png>)
- [Konfigurasi efektif AFTER](<evidence/kontrol A1 — SSH permitlogin/after-hardening-ssh.png>)
- [Percobaan login root ditolak](<evidence/kontrol A1 — SSH permitlogin/hasil-after-hardening.png>)

### A2. Pembatasan percobaan login SSH dengan Fail2ban — Done

- **Tujuan:** membatasi percobaan autentikasi SSH berulang dengan memblokir sementara alamat IP sumber.
- **Kondisi awal (Before):** paket dan unit layanan Fail2ban belum tersedia; akun biasa dapat mengakses SSH.
- **Implementasi:** memasang Fail2ban dan mengaktifkan jail `sshd` pada `/etc/fail2ban/jail.d/pbl-sshd.local` dengan parameter `backend=systemd`, `port=22`, `maxretry=5`, `findtime=600`, dan `bantime=120`.
- **Hasil verifikasi (After):** konfigurasi dinyatakan valid, layanan aktif, dan IP CyberOps `192.168.10.100` tercantum pada daftar blokir setelah percobaan autentikasi gagal. Koneksi SSH baru dari sumber tersebut kemudian ditolak. Verifikasi menggunakan `sudo fail2ban-client -t`, `sudo fail2ban-client status sshd`, dan pengujian SSH terkontrol.
- **Cakupan dan batasan:** durasi blokir 120 detik digunakan untuk demonstrasi laboratorium. Pemulihan akses setelah blokir belum didokumentasikan secara terpisah. Autentikasi password akun biasa tetap diaktifkan.
- **Keterangan bukti:** direktori menggunakan nama lama “SSH key-based authentication”; seluruh bukti pada bagian ini mendokumentasikan Fail2ban, bukan penerapan autentikasi key-only.

Bukti:

- [Status BEFORE](<evidence/kontrol A2 — SSH key-based authentication/before-fail2ban-status.png>)
- [Akses SSH akun biasa](<evidence/kontrol A2 — SSH key-based authentication/fail2ban-02-before-ssh-access.png>)
- [Konfigurasi jail](<evidence/kontrol A2 — SSH key-based authentication/fail2ban-03-sshd-config.png>)
- [Layanan aktif](<evidence/kontrol A2 — SSH key-based authentication/after-fail2ban-status.png>)
- [Daftar IP yang diblokir](<evidence/kontrol A2 — SSH key-based authentication/fail2ban-after-ip-banned.png>)
- [Percobaan gagal dan koneksi ditolak](<evidence/kontrol A2 — SSH key-based authentication/fail2ban-failed-login-attempts.png>)

## 4. Kategori B — Service & Attack Surface

### B1. Firewall hanya mengizinkan akses yang diperlukan — Done

- **Tujuan:** membatasi koneksi masuk pada layanan dan alamat IP sumber yang diperlukan untuk operasional laboratorium.
- **Kondisi awal (Before):** UFW telah aktif dengan kebijakan default deny incoming dan allow outgoing. SSH dan Nginx HTTP masih diizinkan dari seluruh sumber IPv4/IPv6, sedangkan port 9000 diizinkan dari CyberOps. Pemindaian menunjukkan port 22, 80, dan 9000 berstatus open.
- **Implementasi:** membatasi izin TCP 22 dan 80 ke `192.168.10.100` dan `192.168.10.1`, menghapus aturan akses umum SSH/Nginx beserta pasangan IPv6, serta menghapus izin akses langsung CyberOps ke port 9000.
- **Hasil verifikasi (After):** aturan UFW menampilkan sumber yang ditentukan untuk SSH/HTTP. Pemindaian dari CyberOps menunjukkan port 22 dan 80 tetap open, sedangkan port 9000 menjadi filtered. Pemeriksaan menggunakan `sudo ufw status verbose`, `sudo ss -lntup`, dan `nmap -sT -p 22,80,9000 192.168.10.5`.
- **Cakupan dan batasan:** pengujian dilakukan dari CyberOps; pengujian sumber lain dan IPv6 secara terpisah belum didokumentasikan. Kebijakan allow outgoing tidak memberikan isolasi internet. Verifikasi pada bagian ini mencakup aturan firewall dan keterjangkauan port, bukan tampilan dashboard.

Bukti:

- [Port dan aturan BEFORE](<evidence/Kontrol B1 — Firewall hanya mengizinkan layanan yang diperlukan/before-ports-and-ufw.png>)
- [Scan BEFORE](<evidence/Kontrol B1 — Firewall hanya mengizinkan layanan yang diperlukan/before-hasil-nmap.png>)
- [Perubahan dan aturan AFTER](<evidence/Kontrol B1 — Firewall hanya mengizinkan layanan yang diperlukan/after-ufw.png>)
- [Scan AFTER](<evidence/Kontrol B1 — Firewall hanya mengizinkan layanan yang diperlukan/after-hasil-nmap.png>)
- [Pemeriksaan ulang aturan](<evidence/Kontrol B1 — Firewall hanya mengizinkan layanan yang diperlukan/konfig-ufw.png>) — pesan aturan sudah ada/tidak ditemukan merupakan hasil pengulangan perintah setelah konfigurasi diterapkan.

### B2. Database hanya mendengarkan pada localhost — Done

- **Tujuan:** menghilangkan paparan langsung layanan database pada antarmuka jaringan Target.
- **Kondisi awal (Before):** layanan database mendengarkan pada `[::ffff:192.168.10.5]:9000`.
- **Implementasi:** mengubah alamat host emulator pada `firebase.json` menjadi `127.0.0.1`. Alamat localhost digunakan sebagai tujuan akses lokal simulator dan upstream proxy Nginx.
- **Hasil verifikasi (After):** `sudo ss -lntp 'sport = :9000'` menunjukkan listener `[::ffff:127.0.0.1]:9000`, bukan alamat jaringan atau wildcard. Akses REST lokal turut terdokumentasi pada pengujian C1 dan C2.
- **Cakupan dan batasan:** verifikasi berfokus pada alamat listener dan akses REST lokal. Operasional simulator serta tampilan dashboard setelah perubahan tidak didokumentasikan pada bagian ini. Pembatasan loopback tidak menggantikan Security Rules dan tidak membatasi proses lokal yang telah memiliki akses.

Bukti:

- [Listener BEFORE](<evidence/kontrol B2 — Bind ke localhost/before-db-listener.png>)
- [Konfigurasi localhost](<evidence/kontrol B2 — Bind ke localhost/localhost-config.png>)
- [Listener AFTER](<evidence/kontrol B2 — Bind ke localhost/after-db-listener.png>)

### B3. Mengurangi informasi versi Nginx — Done

- **Tujuan:** mengurangi pengungkapan informasi versi perangkat lunak melalui respons HTTP.
- **Kondisi awal (Before):** pemeriksaan `curl -I http://127.0.0.1/dashboard.html` menghasilkan HTTP 200 dengan header `Server: nginx/1.18.0 (Ubuntu)`. Directive `server_tokens off` pada konfigurasi masih dikomentari.
- **Implementasi:** mengaktifkan `server_tokens off;` di dalam blok `http` pada `/etc/nginx/nginx.conf`.
- **Hasil verifikasi (After):** pemeriksaan dari Ubuntu Target menggunakan `curl -I http://192.168.10.5/dashboard.html` menghasilkan HTTP 200 dengan header `Server: nginx`, tanpa nomor versi maupun label Ubuntu.
- **Cakupan dan batasan:** hasil verifikasi terbatas pada header respons halaman dashboard. Halaman error dan tampilan browser tidak termasuk pengujian ini. Nama produk Nginx tetap terlihat; kontrol tidak menggantikan pembaruan keamanan maupun mencegah seluruh metode identifikasi versi.

Bukti:

- [Header dan konfigurasi BEFORE](<evidence/kontrol B3 — menyembunyikan versi Nginx/before-nginx-version.png>)
- [Konfigurasi server_tokens off](<evidence/kontrol B3 — menyembunyikan versi Nginx/server-token-off.png>)
- [Header AFTER tanpa nomor versi, HTTP 200](<evidence/kontrol B3 — menyembunyikan versi Nginx/after-nginx-version.png>)

## 5. Kategori C — Data & Application

### C1. Security Rules dan validasi payload — Done

- **Tujuan:** membatasi struktur serta rentang data sensor yang dapat disimpan pada cabang database aplikasi.
- **Kondisi awal (Before):** file rules telah memuat validasi, tetapi PATCH dengan `humidity:150` masih memperoleh HTTP 200. Validasi belum diterapkan secara efektif pada namespace yang digunakan aplikasi.
- **Implementasi:** menyelaraskan namespace `demo-melon-lab` dengan instance rules melalui konfigurasi `database` berbentuk array yang memuat `instance` dan `rules`, kemudian menjalankan ulang emulator. Penulisan pada cabang sensor menggunakan syarat `newData.exists()`.
- **Kebijakan validasi:** akses ditolak secara default. Cabang `/lab_dummy/sensor` mengizinkan baca/tulis data dummy dengan field wajib `temperature`, `humidity`, `ec`, dan `updated_at`. Rules mensyaratkan tipe numerik, suhu 0–60, kelembapan 0–100, EC 0–10, timestamp positif, serta penolakan field tambahan.
- **Hasil verifikasi (After):** PUT dengan data lengkap yang valid memperoleh HTTP 200, sedangkan PATCH dengan `humidity:150` ditolak dengan HTTP 401 dan pesan `Permission denied`. Pembacaan data valid dengan kelembapan 72 juga terdokumentasi pada C2.
- **Cakupan dan batasan:** pengujian membuktikan penerimaan payload valid dan penolakan kelembapan di luar rentang. Penolakan tipe salah, field tambahan, akses root database, dan penghapusan belum diverifikasi dalam bukti yang dilampirkan. Bukti konfigurasi memperlihatkan proses pengeditan rules; hasil respons API menjadi dasar verifikasi perilaku yang diuji.
- **Ketentuan laboratorium:** cabang dummy masih menggunakan akses tanpa autentikasi, dengan paparan jaringan dibatasi melalui loopback, UFW, dan proxy. Rentang validasi digunakan untuk demonstrasi teknis, bukan sebagai rekomendasi budidaya.

Bukti:

- [Isi rules BEFORE](<evidence/kontrol C1 — Security Rules dan validasi data/before-security-rules.png>)
- [Nilai tidak valid masih diterima BEFORE](<evidence/kontrol C1 — Security Rules dan validasi data/before-hasil-security-rule.png>)
- [Pengeditan rules AFTER](<evidence/kontrol C1 — Security Rules dan validasi data/after-security-rules.png>)
- [Data valid diterima dan humidity 150 ditolak AFTER](<evidence/kontrol C1 — Security Rules dan validasi data/after-hasil-security-rule.png>)

### C2. Backup manual dan uji pemulihan data — Done

- **Tujuan:** menyediakan salinan data emulator dan memverifikasi pengembalian data ke kondisi saat pencadangan.
- **Kondisi awal (Before):** data sensor memiliki nilai `humidity:72` sebelum ekspor.
- **Implementasi:** melakukan ekspor menggunakan `firebase emulators:export` ke direktori bertimestamp. Setelah ekspor, kelembapan diubah menjadi 80 untuk membedakan kondisi data aktif dari salinan cadangan. Pemulihan menggunakan mekanisme impor emulator.
- **Hasil verifikasi (After):** ekspor dinyatakan selesai dan metadata tersedia. Pembacaan setelah pemulihan memperoleh HTTP 200 dengan kelembapan kembali menjadi 72, sesuai kondisi pencadangan dan berbeda dari nilai 80 setelah perubahan.
- **Cakupan dan batasan:** bukti mencakup hasil ekspor, perubahan data, serta pembacaan data hasil pemulihan; perintah dan log impor tidak tercantum pada bukti. Pencadangan dilakukan secara manual pada VM yang sama, belum terjadwal maupun disimpan di luar VM. Ekspor ini mencakup data emulator, bukan keseluruhan konfigurasi, rules, simulator, dan dashboard, sehingga belum memberikan perlindungan terhadap kehilangan seluruh VM.

Bukti:

- [Data sebelum backup: 72](<evidence/kontrol C2 — backup dan uji pemulihan data Firebase Emulator/before-backup.png>)
- [Ekspor selesai dan metadata backup](<evidence/kontrol C2 — backup dan uji pemulihan data Firebase Emulator/backup-export-success.png>)
- [Perubahan setelah backup: 80](<evidence/kontrol C2 — backup dan uji pemulihan data Firebase Emulator/data-berubah-after-backup.png>)
- [Verifikasi data kembali 72](<evidence/kontrol C2 — backup dan uji pemulihan data Firebase Emulator/restored-data.png>)

### C3. Membatasi akses file konfigurasi dan data aplikasi — Done

- **Tujuan:** membatasi akses lokal terhadap direktori aplikasi dan file konfigurasi kepada akun pemilik aplikasi.
- **Kondisi awal (Before):** direktori `/home/iotuser/melon-lab` milik `iotuser:iotuser` memiliki permission `775` (`drwxrwxr-x`). File `firebase.json`, `database.rules.json`, dan `simulator.py` memiliki permission `664` (`-rw-rw-r--`). Direktori `data` dan `backup-20261015-143946` telah memiliki permission `700` sejak pemeriksaan awal.
- **Implementasi:** menetapkan permission direktori aplikasi menjadi `700` dan ketiga file menjadi `600` menggunakan `chmod`. Permission direktori data dan backup dipertahankan; direktori dashboard `/var/www/html` tidak termasuk perubahan.
- **Hasil verifikasi (After):** pemeriksaan `ls` menunjukkan permission direktori `drwx------` dan file `-rw-------`, dengan kepemilikan tetap `iotuser:iotuser`. Uji `test -r` pada `firebase.json` menghasilkan kode keluar `0` untuk pemilik dan `1` untuk akun `nobody`, yang menunjukkan akses baca pemilik diizinkan dan akses akun pembanding ditolak.
- **Cakupan dan batasan:** pengujian baca langsung dilakukan pada `firebase.json`; hasil akses dipengaruhi permission file dan direktori induknya. Operasional simulator serta dashboard setelah perubahan permission tidak termasuk bukti pengujian ini. Kontrol tidak mengenkripsi data serta tidak mencegah akses root atau penyalahgunaan akun pemilik yang telah dikompromikan.

Bukti:

- [Permission file, direktori aplikasi, data dan backup BEFORE](<evidence/kontrol C3 — Membatasi akses file konfigurasi dan data aplikasi/before-file-permissions.png>)
- [Penerapan chmod dan permission AFTER](<evidence/kontrol C3 — Membatasi akses file konfigurasi dan data aplikasi/after-file-permissions.png>)
- [Pemilik dapat membaca, akun nobody ditolak](<evidence/kontrol C3 — Membatasi akses file konfigurasi dan data aplikasi/hasil-hak-akses.png>)

## 6. Rencana Hardening Lanjutan

Empat rencana sederhana berikut mengacu pada tujuan kontrol **NIST SP 800-53 Rev. 5** sebagai panduan

| ID | Rencana Kontrol | Bukti yang Perlu Disiapkan | Acuan NIST | Status |
|---|---|---|---|---|
| B4 | Memastikan daftar isi direktori web tidak ditampilkan melalui pengaturan `autoindex off` pada Nginx. | Konfigurasi dan uji direktori tanpa file index: daftar file tidak muncul; dashboard tetap dapat diakses. | CM-7 | Not Yet |
| A3 | Membatasi akun yang boleh login SSH menggunakan `AllowUsers`, misalnya hanya `iotuser` sesuai kebutuhan administrasi. | Konfigurasi efektif, login akun yang diizinkan berhasil, dan akun uji lain dengan kredensial valid ditolak karena aturan akses. | AC-3 | Not Yet |
| B5 | Menonaktifkan X11 forwarding SSH yang tidak digunakan melalui `X11Forwarding no`. | Konfigurasi efektif menunjukkan `x11forwarding no`; sesi SSH biasa tetap berfungsi. | CM-7 | Not Yet |
| C5 | Memblokir akses HTTP ke file/direktori tersembunyi melalui Nginx, seperti `.env` dan `.git`. | Konfigurasi dan uji file tersembunyi dummy yang benar-benar ada: akses ditolak tanpa menampilkan isinya; dashboard/API tetap berfungsi. | AC-3 | Not Yet |

Jika daftar direktori sudah nonaktif, catat sebagai verifikasi pengaturan yang ada, bukan perubahan before/after yang dibuat-buat. Pengujian file tersembunyi menggunakan file dummy tanpa rahasia; HTTP 404 untuk file yang tidak ada saja tidak membuktikan pemblokiran. Atur pengecualian `.well-known` bila diperlukan untuk layanan yang sah. Status diubah menjadi **Done** setelah bukti dan fungsi normal diperiksa.

Sebelum mengubah SSH, pastikan semua akun administrasi yang diperlukan masuk daftar izin, siapkan akses konsol VM, dan validasi konfigurasi. Uji penolakan akun dilakukan terbatas agar tidak memicu Fail2ban; jangan memakai password salah sebagai bukti penolakan `AllowUsers`. Jika X11 forwarding sudah nonaktif, dokumentasikan sebagai verifikasi kondisi yang ada. Seluruh tambahan tetap berupa rencana dan belum diterapkan melalui perubahan dokumen ini.

Referensi: [NIST SP 800-53 Rev. 5 — Security and Privacy Controls for Information Systems and Organizations](https://doi.org/10.6028/NIST.SP.800-53r5).
