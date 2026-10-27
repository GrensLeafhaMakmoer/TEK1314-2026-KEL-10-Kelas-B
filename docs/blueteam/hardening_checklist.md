# Checklist Hardening Keamanan — Ubuntu Server

**Kelompok 10 | TEK1314 — PBL Keamanan Siber | Checkpoint Minggu 9**

## 1. Tujuan dan Ruang Lingkup

Dokumen ini menyajikan status penerapan dan cakupan verifikasi 14 kontrol hardening pada replika sistem monitoring nutrisi AB Mix tanaman melon: delapan kontrol awal, empat kontrol yang sebelumnya tercatat sebagai rencana tetapi sudah diterapkan, serta dua kontrol tambahan di luar daftar lama. Tiga kontrol yang baru dilengkapi dokumentasinya adalah A3 (LoginGraceTime), A4 (SSH public key dan password), dan B6 (rate limit API Nginx). A3 sudah termasuk dalam empat rencana lama sehingga tidak dihitung dua kali. Kontrol dikelompokkan ke dalam kategori Access & Authentication, Service & Attack Surface, serta Data & Application.

- Target: Ubuntu Server `srv-iot-kel10`, IP lab `192.168.10.5`.
- Perangkat penguji: CyberOps Workstation `192.168.10.100` dan laptop Windows `192.168.10.1` sebagai host administrasi serta penguji SSH/browser.
- Layanan: SSH, Nginx untuk dashboard HTML/JavaScript, Firebase Realtime Database Emulator, dan simulator Python.
- Project demo / namespace aplikasi: `demo-melon-lab`; data dummy: `/lab_dummy/sensor`.
- Dokumentasi bukti: tangkapan layar pada direktori `docs/blueteam/evidence/`, ditautkan pada setiap kontrol.

Seluruh implementasi dan pengujian dilakukan pada lingkungan laboratorium menggunakan data dummy. Ruang lingkup tidak mencakup Firebase cloud, sensor fisik, maupun pengendalian pompa. Komunikasi REST pada replika menggunakan HTTP dan belum dilengkapi TLS/HTTPS.

### Kriteria Status

- **Done (sudah diterapkan)**: kontrol telah diterapkan berdasarkan bukti konfigurasi, hasil pengujian, atau konfirmasi penerapan sebelumnya yang disebutkan pada uraian. Status ini tidak berarti seluruh skenario pengujian sudah dilengkapi screenshot.
- **Partial**: penerapan kontrol baru terpenuhi sebagian.
- **Not Yet**: kontrol belum diterapkan.

Kelengkapan bukti dan batasan verifikasi dicatat terpisah dari status penerapan agar kontrol yang sudah aktif tidak kembali disebut sebagai rencana hanya karena screenshot belum lengkap.

## 2. Ringkasan Implementasi

| ID | Kategori | Kontrol | Status | Hasil Verifikasi |
|---|---|---|---|---|
| A1 | Access & Authentication | Menolak login SSH langsung sebagai root | Done | Konfigurasi efektif `permitrootlogin no`; percobaan root ditolak. |
| A2 | Access & Authentication | Pembatasan percobaan autentikasi SSH menggunakan Fail2ban | Done | Jail aktif, IP `192.168.10.100` masuk daftar blokir, dan koneksi SSH baru ditolak. |
| A3 | Access & Authentication | Membatasi waktu penyelesaian autentikasi SSH | Done | Konfigurasi efektif berubah dari `logingracetime 120` menjadi `30`; screenshot after menunjukkan koneksi ditutup, tetapi tidak memuat pengukuran durasi. |
| A4 | Access & Authentication | Mewajibkan SSH public key dan password | Done | Konfigurasi efektif `publickey,password`; login dengan key + password berhasil dan password saja ditolak. |
| B1 | Service & Attack Surface | Membatasi akses masuk menggunakan UFW | Done | SSH/HTTP dibatasi ke IP lab yang ditentukan; port 9000 dari CyberOps berubah menjadi filtered. |
| B2 | Service & Attack Surface | Mengikat database ke localhost | Done | Listener database berubah dari IP jaringan menjadi `127.0.0.1:9000`. |
| B3 | Service & Attack Surface | Mengurangi informasi versi Nginx pada respons | Done | Header berubah dari `nginx/1.18.0 (Ubuntu)` menjadi `nginx`; halaman dashboard merespons HTTP 200. |
| B4 | Service & Attack Surface | Menonaktifkan daftar isi direktori web | Done | Screenshot pemeriksaan ulang menunjukkan `autoindex off` pada site Nginx yang diaktifkan. |
| B5 | Service & Attack Surface | Mencegah framing melalui `X-Frame-Options: DENY` | Done | Screenshot menunjukkan konfigurasi DENY dan header pada respons dashboard HTTP 303 serta halaman login HTTP 200. |
| B6 | Service & Attack Surface | Membatasi laju request API Nginx | Done | Before: 15 respons HTTP 200; after: 5 respons HTTP 200 dan 10 respons HTTP 429 pada 15 request cepat. |
| C1 | Data & Application | Security Rules dan validasi payload sensor | Done | Data valid HTTP 200; `humidity:150` ditolak HTTP 401 setelah namespace rules diselaraskan. |
| C2 | Data & Application | Backup manual dan verifikasi pemulihan data | Done | Ekspor berhasil; setelah perubahan ke 80, nilai kelembapan kembali ke 72 pada verifikasi pemulihan. |
| C3 | Data & Application | Membatasi akses file konfigurasi dan data aplikasi | Done | Permission direktori aplikasi menjadi `700` dan tiga file menjadi `600`; akses baca pemilik diizinkan, sedangkan akun `nobody` ditolak. |
| C5 | Data & Application | Memblokir akses HTTP ke file/direktori tersembunyi | Done | Screenshot menunjukkan aturan blokir, keberadaan dummy `.env`/`.git`, dan kedua akses HTTP ditolak dengan 403. |

**Rekapitulasi: 14 Done (sudah diterapkan), 0 Partial, dan 0 Not Yet.** Implementasi terdiri atas **4 kontrol kategori A, 6 kontrol kategori B, dan 4 kontrol kategori C**, sehingga memenuhi jumlah minimal delapan kontrol dengan sedikitnya dua kontrol pada setiap kategori. Status dibatasi oleh cakupan verifikasi masing-masing kontrol dan tidak menyatakan sistem telah aman untuk penggunaan produksi.

Delapan kontrol awal tetap dipertahankan. Empat rencana lama (B4, A3, B5, dan C5) diperbarui menjadi **sudah diterapkan**, bukan diklaim sebagai empat pekerjaan baru. Dokumentasi before–konfigurasi–after ditambahkan untuk **tiga kontrol saja: A3, A4, dan B6**. Total kontrol unik adalah **8 + 4 + 2 = 14**, bukan 15. ID lama dipertahankan, termasuk C5; API rate limit memakai B6 karena B4 adalah autoindex dan B5 adalah header anti-framing. Folder bukti `B4 - api rate limit nginx` tetap memakai nama yang dibuat saat pengujian, tetapi merujuk ke kontrol B6 dalam checklist ini.

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

### A3. Membatasi waktu penyelesaian autentikasi SSH — Done (sudah diterapkan)

- **Tujuan:** membatasi waktu koneksi SSH baru yang belum menyelesaikan autentikasi, bukan memutus sesi pengguna yang sudah berhasil login.
- **Kondisi awal (Before):** hasil `sudo /usr/sbin/sshd -T | grep '^logingracetime '` menunjukkan `logingracetime 120`.
- **Implementasi:** menetapkan `LoginGraceTime 30` pada `/etc/ssh/sshd_config.d/00-pbl-login-grace.conf`, memvalidasi sintaks dengan `sudo /usr/sbin/sshd -t`, lalu menjalankan `sudo systemctl reload ssh`. Pemeriksaan konfigurasi efektif menunjukkan `logingracetime 30`.
- **Hasil verifikasi (After):** screenshot koneksi SSH dari Windows memperlihatkan prompt password kemudian pesan `Connection closed by 192.168.10.5 port 22` setelah autentikasi tidak diselesaikan. Nilai efektif 30 detik didukung screenshot konfigurasi.
- **Cakupan dan batasan:** screenshot after tidak memuat timestamp awal/akhir sehingga belum membuktikan sendiri durasi tepat 30 detik. Prompt password yang masih terlihat di klien tidak membuktikan koneksi masih hidup. Batas waktu berlaku untuk keseluruhan autentikasi, termasuk waktu memasukkan passphrase key dan password pada A4.

Perintah pemeriksaan langsung di Ubuntu (tanpa mengubah ulang kontrol):

```bash
sudo cat /etc/ssh/sshd_config.d/00-pbl-login-grace.conf
sudo /usr/sbin/sshd -t
sudo /usr/sbin/sshd -T | grep '^logingracetime '
```

Bukti:

- [BEFORE: konfigurasi efektif 120 detik](<evidence/kontrol A3 — SSH login grace/before.png>)
- [Konfigurasi dan nilai efektif 30 detik](<evidence/kontrol A3 — SSH login grace/konfig.png>)
- [AFTER: koneksi tanpa penyelesaian autentikasi ditutup](<evidence/kontrol A3 — SSH login grace/hasil-setelah-30detik.png>)

### A4. Autentikasi SSH menggunakan public key dan password — Done (sudah diterapkan)

- **Tujuan:** mewajibkan kepemilikan private key yang sesuai dan password akun Ubuntu pada satu koneksi SSH.
- **Kondisi awal (Before):** login password saja dari laptop Windows masih berhasil, sebagaimana screenshot before.
- **Implementasi:** membuat pasangan key di laptop Windows pada folder PBL; private key `pbl_iot_key` tetap di Windows, sedangkan public key pasangannya ditambahkan ke `/home/iotuser/.ssh/authorized_keys`. Drop-in `/etc/ssh/sshd_config.d/00-pbl-key-password.conf` berisi pengaturan berikut:

```text
PubkeyAuthentication yes
PasswordAuthentication yes
KbdInteractiveAuthentication no
AuthenticationMethods publickey,password
```

- **Validasi konfigurasi:** `sudo /usr/sbin/sshd -t` tidak menampilkan error; `sudo /usr/sbin/sshd -T` menunjukkan `pubkeyauthentication yes`, `passwordauthentication yes`, dan `authenticationmethods publickey,password`. Konfigurasi dimuat ulang menggunakan `sudo systemctl reload ssh`.
- **Hasil verifikasi (After):** login memakai key dan password berhasil sampai shell `iotuser@srv-iot-kel10`. Log pengujian yang disampaikan saat pelaksanaan menunjukkan `Authenticated using "publickey" with partial success.`, kemudian prompt password Ubuntu. Percobaan password saja ditolak dengan `Permission denied (publickey).`
- **Cakupan dan batasan:** koma pada `publickey,password` berarti kedua metode wajib berurutan, bukan pilihan salah satu. Password tetap diaktifkan; passphrase private key berbeda dari password Ubuntu. Screenshot uji key saja belum tersedia, sehingga penolakannya tidak diklaim sebagai hasil uji yang sudah terdokumentasi. Private key tidak ditampilkan atau dijadikan bukti laporan.

Perintah uji dari PowerShell Windows, menggunakan key yang sudah ada:

```powershell
cd 'D:\KULIAH\sem-5\SIBER\PBL\PBL 2026'
ssh -v -i .\pbl_iot_key -o IdentitiesOnly=yes -o ControlPath=none -o PreferredAuthentications=publickey,password -o KbdInteractiveAuthentication=no iotuser@192.168.10.5
```

Setelah login, jalankan `whoami` dan `hostname`, lalu `exit`. Uji password saja pada koneksi baru:

```powershell
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password iotuser@192.168.10.5
```

Bukti:

- [BEFORE: login password saja](<evidence/A4 - SSH private key/before-password-only.png>)
- [Konfigurasi tahap pertama](<evidence/A4 - SSH private key/konfig.png>)
- [Konfigurasi lanjutan](<evidence/A4 - SSH private key/config-2.png>)
- [Validasi sintaks dan konfigurasi efektif](<evidence/A4 - SSH private key/verif-konfig.png>)
- [AFTER: password saja ditolak](<evidence/A4 - SSH private key/after-masukkan-password-only.png>)
- [AFTER: login key dan password berhasil](<evidence/A4 - SSH private key/after-masukkab-privatekey-dan-password.png>)

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

### B6. Pembatasan laju request API menggunakan Nginx — Done (sudah diterapkan)

- **Tujuan:** membatasi request HTTP berlebihan ke API sensor dan history per alamat IP sumber untuk mengurangi beban pada aplikasi/upstream.
- **Kondisi awal (Before):** pengujian 15 request cepat dari console browser Windows yang sudah login menghasilkan 15 respons HTTP 200, tanpa respons 429 pada pengujian tersebut.
- **Implementasi:** mendefinisikan zona pada `/etc/nginx/conf.d/pbl-api-rate-limit.conf` dalam konteks `http`:

```nginx
limit_req_zone $binary_remote_addr zone=pbl_api:10m rate=2r/s;
```

Pada kedua lokasi `/api/sensor` dan `/api/history` di `/etc/nginx/sites-available/default`, ditambahkan:

```nginx
limit_req zone=pbl_api burst=5 nodelay;
limit_req_status 429;
```

- **Hasil verifikasi (After):** pengujian 15 request cepat ke `/api/sensor` menghasilkan 5 respons HTTP 200 dan 10 respons HTTP 429 (`Too Many Requests`). Respons 429 membuktikan pembatasan berjalan pada endpoint yang diuji, bukan sekadar adanya teks konfigurasi pada editor.
- **Cakupan dan batasan:** kedua endpoint menggunakan zona yang sama sehingga berbagi perhitungan laju per IP. `burst=5 nodelay` mengizinkan burst terbatas tanpa penundaan; jumlah respons 200 tidak harus selalu lima pada setiap pengulangan karena dipengaruhi waktu dan request lain. Screenshot pengujian terpisah untuk `/api/history`, request normal berjarak, dan log limiter belum tersedia. Tampilan dashboard memuat peringatan data lama, sehingga bukti ini tidak menyatakan simulator sedang memperbarui data secara normal. Kontrol ini adalah mitigasi request berlebih pada lapisan HTTP, **bukan perlindungan SYN flood**. Layanan tetap menggunakan HTTP.
- **Keterangan bukti:** nama folder lama `B4 - api rate limit nginx` dipertahankan; ID kontrol pada checklist adalah B6 agar tidak bertabrakan dengan B4 autoindex.

Perintah pemeriksaan langsung di Ubuntu:

```bash
sudo nginx -t
sudo nginx -T 2>&1 | grep -nE 'limit_req|location = /api/(sensor|history)'
```

Untuk mengulang uji terbatas di console browser setelah login melalui `http://192.168.10.5/auth/login`, gunakan JavaScript berikut. Tidak perlu mengubah konfigurasi atau melemahkan kontrol untuk mengulang before.

```javascript
const hasil = await Promise.all(Array.from({ length: 15 }, () =>
  fetch('/api/sensor', { credentials: 'same-origin', cache: 'no-store' })
    .then(respons => respons.status)
));
console.table(hasil);
```

Bukti:

- [BEFORE: 15 request mendapat HTTP 200](<evidence/B4 - api rate limit nginx/before-api-rate-limit.png>)
- [Konfigurasi zona 2 request/detik](<evidence/B4 - api rate limit nginx/config.png>)
- [Pengeditan limiter pada API sensor dan history](<evidence/B4 - api rate limit nginx/config-2.png>)
- [AFTER: 5 respons HTTP 200 dan 10 respons HTTP 429](<evidence/B4 - api rate limit nginx/after-api-rate-limit.png>)

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

## 6. Pembaruan Status Empat Rencana Hardening Sebelumnya

Setelah pemeriksaan ulang dan konfirmasi penerapan, empat kontrol pada daftar rencana lama dinyatakan **sudah diterapkan**. Tabel ini memperbarui status lama, bukan menambah empat kontrol baru. A3 diuraikan bersama dua kontrol tambahan A4 dan B6 pada kategori masing-masing. Acuan **NIST SP 800-53 Rev. 5** digunakan sebagai panduan tujuan kontrol, bukan klaim sertifikasi kepatuhan.

| ID | Kontrol yang Sudah Diterapkan | Dasar Status dan Kelengkapan Bukti | Acuan NIST | Status |
|---|---|---|---|---|
| B4 | Menonaktifkan daftar isi direktori web melalui `autoindex off` pada Nginx. | Pemeriksaan ulang pada site yang diaktifkan menunjukkan `autoindex off;`, root web, dan daftar file index. [Screenshot konfigurasi](<evidence/kontrol B4 - Nginx autoindex/verifikasi-autoindex.png>). | CM-7 | Done — sudah diterapkan |
| A3 | Membatasi waktu penyelesaian autentikasi SSH melalui `LoginGraceTime 30`. | Bukti before, konfigurasi efektif 30 detik, dan koneksi ditutup tersedia pada A3. Pengukuran durasi bertimestamp belum tercantum pada screenshot after. | CM-6 | Done — sudah diterapkan |
| B5 | Menambahkan header Nginx `X-Frame-Options: DENY` untuk mengurangi risiko clickjacking. | Konfigurasi memuat `add_header X-Frame-Options DENY always;`. Respons dashboard tanpa login adalah HTTP 303 menuju login, sedangkan halaman login HTTP 200; keduanya memuat `X-Frame-Options: DENY`. [Screenshot konfigurasi dan header](<evidence/kontrol B5 - X-Frame-Options/verifikasi-header-deny.png>). | CM-6 | Done — sudah diterapkan |
| C5 | Memblokir akses HTTP ke file/direktori tersembunyi melalui Nginx, seperti `.env` dan `.git`. | Aturan `location ~ /\.` memuat `deny all`. Dummy `.env` dan `.git` pada direktori `pbl-controls-20261026` benar-benar ada; kedua akses HTTP ditolak dengan 403 tanpa menampilkan isinya. [Screenshot konfigurasi dan hasil HTTP](<evidence/kontrol C5 - Blokir file tersembunyi/verifikasi-dotfiles.png>). | AC-3 | Done — sudah diterapkan |

Tiga screenshot pemeriksaan ulang diambil melalui MCP console VirtualBox pada Ubuntu Server tanpa mengubah konfigurasi hardening. Screenshot memperlihatkan kondisi yang sudah terpasang saat diperiksa, bukan before penerapan. Verifikasi B4 terbatas pada konfigurasi site yang diaktifkan; pengujian ulang direktori tanpa index dengan sesi terautentikasi tidak termasuk screenshot ini. Verifikasi B5 mencakup header HTTP, bukan uji iframe di browser. Tidak ada kontrol keempat yang ditambahkan.

Referensi: [NIST SP 800-53 Rev. 5 — Security and Privacy Controls for Information Systems and Organizations](https://doi.org/10.6028/NIST.SP.800-53r5).
