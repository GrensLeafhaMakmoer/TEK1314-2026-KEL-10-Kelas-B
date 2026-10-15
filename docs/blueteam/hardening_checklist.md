# Hardening Checklist — Ubuntu Target Kelompok 10

## Lingkup dan status

- Target: Ubuntu Server `srv-iot-kel10`, IP lab `192.168.10.5`.
- Penguji: CyberOps Workstation `192.168.10.100`; host administrasi pada aturan UFW: `192.168.10.1`.
- Layanan: SSH, Nginx untuk dashboard HTML/JavaScript, Firebase Realtime Database Emulator, dan simulator Python.
- Project demo / namespace aplikasi: `demo-melon-lab`; data dummy: `/lab_dummy/sensor`.
- Bukti berupa screenshot di `docs/blueteam/evidence/`. Checklist ini tidak berarti konfigurasi diterapkan ke Firebase cloud atau perangkat fisik.

Arti status:

- **Done**: tujuan kontrol yang disebutkan sudah didukung bukti konfigurasi atau pengujian yang relevan. Tidak berarti seluruh aspek keamanan aplikasi selesai.
- **Partial**: baru sebagian diterapkan atau verifikasi utama belum tersedia.
- **Not Yet**: rencana, belum ada bukti penerapan dan verifikasi.

## Ringkasan

| ID | Kategori | Kontrol | Status | Hasil utama / tindak lanjut |
|---|---|---|---|---|
| A1 | Access & Authentication | Menolak login SSH langsung sebagai root | Done | Konfigurasi efektif `permitrootlogin no`; percobaan root ditolak. |
| A2 | Access & Authentication | Fail2ban untuk kegagalan autentikasi SSH | Done | Jail aktif, IP `.100` masuk daftar ban, koneksi SSH baru ditolak. |
| B1 | Service & Attack Surface | Membatasi akses masuk menggunakan UFW | Done | SSH/HTTP dibatasi ke IP lab yang ditentukan; port 9000 dari CyberOps berubah menjadi filtered. |
| B2 | Service & Attack Surface | Mengikat database ke localhost | Done | Listener database berubah dari IP jaringan menjadi `127.0.0.1:9000`. |
| B3 | Service & Attack Surface | Mengurangi informasi versi Nginx pada respons | Not Yet | Rencana `server_tokens off`; perlu bukti sebelum/sesudah dan uji akses dashboard. |
| C1 | Data & Application | Security Rules dan validasi payload sensor | Done | Data valid HTTP 200; `humidity:150` ditolak HTTP 401 setelah namespace rules diselaraskan. |
| C2 | Data & Application | Backup manual dan verifikasi pemulihan data | Done | Ekspor berhasil; setelah perubahan ke 80, nilai kelembapan kembali ke 72 pada verifikasi pemulihan. |
| C3 | Data & Application | Membatasi endpoint proxy sensor menjadi baca saja | Not Yet | Blok GET-only pernah disarankan, tetapi konfigurasi aktif dan tes penolakan metode tulis belum terdokumentasi. |

**Progres: 6 Done, 0 Partial, 2 Not Yet.** Terdapat 8 entri rencana dengan minimal 2 per kategori, tetapi **belum 8 kontrol selesai**. B3 dan C3 tidak dihitung sebagai implementasi yang sudah dilakukan.

## A — Access & Authentication

### A1. Menonaktifkan login root langsung melalui SSH — Done

- **Before:** baseline awal yang pernah diperiksa adalah `without-password` (password root dilarang, metode non-password masih mungkin). Screenshot BEFORE yang disimpan menunjukkan `PermitRootLogin yes` dan login root berhasil pada **percobaan sementara yang sengaja diaktifkan**. Bukti tersebut bukan baseline asli dan tidak boleh dilaporkan sebagai kondisi bawaan Ubuntu.
- **Perubahan:** menetapkan `PermitRootLogin no` pada `/etc/ssh/sshd_config.d/00-pbl-hardening.conf`, memvalidasi sintaks, lalu reload SSH.
- **After:** output konfigurasi efektif menunjukkan `permitrootlogin no`; login root baru dari CyberOps ditolak.
- **Verifikasi:** `sudo /usr/sbin/sshd -t`, `sudo /usr/sbin/sshd -T`, dan `ssh root@192.168.10.5`.
- **Batasan:** `sudo -i` oleh akun berhak tetap diperbolehkan. Penolakan login saja tidak membuktikan kebijakan root tanpa output konfigurasi efektif. Login akun biasa tersedia pada bukti A2.

Bukti:

- [Konfigurasi percobaan sementara BEFORE](<evidence/kontrol A1 -SSH permitlogin/ssh-before-hardening.png>)
- [Login root pada percobaan sementara](<evidence/kontrol A1 -SSH permitlogin/hasil-before-hardening.png>)
- [File hardening](<evidence/kontrol A1 -SSH permitlogin/afterhardening-ssh-1.png>)
- [Konfigurasi efektif AFTER](<evidence/kontrol A1 -SSH permitlogin/after-hardening-ssh.png>)
- [Percobaan login root ditolak](<evidence/kontrol A1 -SSH permitlogin/hasil-after-hardening.png>)

### A2. Pembatasan percobaan login SSH dengan Fail2ban — Done

- **Before:** paket/unit Fail2ban belum ditemukan; akun biasa dapat login melalui SSH.
- **Perubahan:** memasang Fail2ban dan mengaktifkan jail `sshd` pada `/etc/fail2ban/jail.d/pbl-sshd.local` dengan `backend=systemd`, `port=22`, `maxretry=5`, `findtime=600`, dan `bantime=120`.
- **After:** konfigurasi lolos tes, layanan aktif, IP CyberOps `192.168.10.100` terdaftar sebagai banned.
- **Verifikasi:** `sudo fail2ban-client -t`, `sudo fail2ban-client status sshd`, dan percobaan login gagal terkontrol dari CyberOps; koneksi SSH baru kemudian ditolak.
- **Batasan:** ban 120 detik adalah parameter demonstrasi lab. Pemulihan/unban dan login ulang setelah ban belum memiliki screenshot tersendiri. Autentikasi password akun biasa tetap dipertahankan.
- **Catatan folder:** nama folder bukti masih menyebut SSH key-based authentication, tetapi isinya adalah pengujian **Fail2ban**, bukan bukti penerapan autentikasi key-only.

Bukti:

- [Status BEFORE](<evidence/kontrol A2 — SSH key-based authentication/before-fail2ban-status.png>)
- [Akses SSH akun biasa](<evidence/kontrol A2 — SSH key-based authentication/fail2ban-02-before-ssh-access.png>)
- [Konfigurasi jail](<evidence/kontrol A2 — SSH key-based authentication/fail2ban-03-sshd-config.png>)
- [Layanan aktif](<evidence/kontrol A2 — SSH key-based authentication/after-fail2ban-status.png>)
- [Daftar IP yang diblokir](<evidence/kontrol A2 — SSH key-based authentication/fail2ban-after-ip-banned.png>)
- [Percobaan gagal dan koneksi ditolak](<evidence/kontrol A2 — SSH key-based authentication/fail2ban-failed-login-attempts.png>)

## B — Service & Attack Surface

### B1. Firewall hanya mengizinkan akses yang diperlukan — Done

- **Before:** UFW sudah aktif dengan default deny incoming dan allow outgoing, tetapi SSH dan profil Nginx HTTP masih diizinkan dari Anywhere (IPv4/IPv6). Port 9000 diizinkan dari CyberOps. Scan menunjukkan 22, 80, dan 9000 open.
- **Perubahan:** memberi izin TCP 22 dan 80 dari `.100` dan `.1`, menghapus izin luas SSH/Nginx termasuk pasangan IPv6, dan menghapus izin langsung port 9000 dari CyberOps.
- **After:** aturan masuk hanya menampilkan izin sumber lab tersebut untuk SSH/HTTP. Scan CyberOps menunjukkan 22 dan 80 open, 9000 filtered.
- **Verifikasi:** `sudo ufw status verbose`, `sudo ss -lntup`, dan `nmap -sT -p 22,80,9000 192.168.10.5`.
- **Batasan:** belum ada bukti tes dari sumber yang tidak diizinkan untuk port 22/80, atau uji IPv6 terpisah. Default allow outgoing bukan isolasi internet; adapter NAT perlu dikelola sesuai skenario lab. Screenshot dashboard AFTER belum tersedia.

Bukti:

- [Port dan aturan BEFORE](<evidence/Kontrol B1 — Firewall hanya mengizinkan layanan yang diperlukan/before-ports-and-ufw.png>)
- [Scan BEFORE](<evidence/Kontrol B1 — Firewall hanya mengizinkan layanan yang diperlukan/before-hasil-nmap.png>)
- [Perubahan dan aturan AFTER](<evidence/Kontrol B1 — Firewall hanya mengizinkan layanan yang diperlukan/after-ufw.png>)
- [Scan AFTER](<evidence/Kontrol B1 — Firewall hanya mengizinkan layanan yang diperlukan/after-hasil-nmap.png>)
- [Pemeriksaan ulang aturan yang sudah diterapkan](<evidence/Kontrol B1 — Firewall hanya mengizinkan layanan yang diperlukan/konfig-ufw.png>) — pesan aturan sudah ada/tidak ditemukan berasal dari pengulangan command, bukan penerapan pertama.

### B2. Database hanya mendengarkan pada localhost — Done

- **Before:** proses Java database mendengarkan pada `[::ffff:192.168.10.5]:9000`.
- **Perubahan:** mengganti host emulator menjadi `127.0.0.1` di `firebase.json`. URL simulator dan upstream proxy Nginx harus memakai localhost agar alur aplikasi tetap berfungsi.
- **After:** hasil `ss` menunjukkan listener `[::ffff:127.0.0.1]:9000`, bukan alamat jaringan atau wildcard.
- **Verifikasi:** `sudo ss -lntp 'sport = :9000'`; akses REST lokal juga terlihat pada pengujian C1/C2.
- **Batasan:** bukti listener sudah tersedia, tetapi screenshot simulator dan dashboard bekerja setelah perubahan belum dilampirkan. Binding loopback tidak menggantikan Security Rules dan tidak melindungi dari proses lokal yang memiliki akses.

Bukti:

- [Listener BEFORE](<evidence/kontrol B2 — Bind ke localhost/before-db-listener.png>)
- [Konfigurasi localhost](<evidence/kontrol B2 — Bind ke localhost/localhost-config.png>)
- [Listener AFTER](<evidence/kontrol B2 — Bind ke localhost/after-db-listener.png>)

### B3. Mengurangi informasi versi Nginx — Not Yet

- **Rencana:** periksa header HTTP awal, tetapkan `server_tokens off` pada konfigurasi Nginx yang aktif, validasi `nginx -t`, lalu reload.
- **Kriteria selesai:** header/halaman error tidak mengungkap nomor versi Nginx dan dashboard tetap dapat diakses. Nama produk Nginx masih bisa terlihat; kontrol ini hanya mengurangi pengungkapan informasi, bukan menggantikan pembaruan keamanan.
- **Bukti:** belum tersedia. Jangan mengubah kembali pengaturan aman jika ternyata sudah diterapkan; dokumentasikan kondisi yang ditemukan.

## C — Data & Application

### C1. Security Rules dan validasi payload — Done

- **Before:** file rules sudah berisi validasi, tetapi PATCH `humidity:150` masih mendapat HTTP 200. Artinya validasi belum berlaku efektif pada namespace yang diuji.
- **Perubahan:** menyelaraskan namespace aplikasi `demo-melon-lab` dengan instance rules melalui konfigurasi `database` berbentuk array berisi `instance` dan `rules`, kemudian restart emulator. Penulisan cabang sensor menggunakan syarat `newData.exists()`.
- **Kebijakan:** akses default ditolak; cabang `/lab_dummy/sensor` mengizinkan baca/tulis dummy dengan field wajib `temperature`, `humidity`, `ec`, `updated_at`. Tipe harus angka, suhu 0–60, kelembapan 0–100, EC 0–10, timestamp positif, dan field tambahan ditolak oleh rules.
- **After / verifikasi:** PUT data lengkap valid mendapat HTTP 200; PATCH `humidity:150` ditolak HTTP 401 dengan `Permission denied` pada namespace yang sama. Baca lokal dengan kelembapan 72 juga terlihat pada bukti C2.
- **Batasan:** baru penerimaan data valid dan penolakan kelembapan di luar rentang yang dibuktikan. Tes tipe salah, field tambahan, akses root, serta penghapusan belum dilampirkan. Screenshot editor rules AFTER masih menunjukkan buffer berubah; bukti perilaku API adalah verifikasi efektif untuk tes yang dilakukan. Tambahkan screenshot konfigurasi tersimpan dan array instance final.
- **Keamanan lab:** akses cabang dummy tetap tanpa autentikasi, dibatasi localhost/UFW dan proxy. Emulator bukan server produksi. REST masih **HTTP**, belum HTTPS. Rentang validasi ini bukan rekomendasi agronomi.

Bukti:

- [Isi rules BEFORE](<evidence/kontrol C1 — Security Rules dan validasi data/before-security-rules.png>)
- [Nilai tidak valid masih diterima BEFORE](<evidence/kontrol C1 — Security Rules dan validasi data/before-hasil-security-rule.png>)
- [Pengeditan rules AFTER](<evidence/kontrol C1 — Security Rules dan validasi data/after-security-rules.png>)
- [Data valid diterima dan humidity 150 ditolak AFTER](<evidence/kontrol C1 — Security Rules dan validasi data/after-hasil-security-rule.png>)

### C2. Backup manual dan uji pemulihan data — Done

- **Before:** data sensor dummy menunjukkan `humidity:72` sebelum ekspor.
- **Perubahan:** mengekspor data dengan `firebase emulators:export` ke folder bertimestamp, kemudian mengubah kelembapan menjadi 80 sebagai pembeda kondisi aktif dan backup.
- **After / verifikasi:** ekspor selesai dan metadata tersedia; screenshot verifikasi pemulihan menunjukkan GET HTTP 200 dengan kelembapan kembali 72 dan data sensor sesuai kondisi backup, bukan 80.
- **Prosedur:** hentikan simulator sebelum uji; hentikan emulator lama secara bersih sebelum menjalankan instance dengan `--import`. Port taken pada percobaan awal merupakan bentrok proses, bukan bukti backup rusak. Simpan backup terpisah dari direktori ekspor operasional `./data`.
- **Batasan:** screenshot command/log impor belum tersedia, sehingga tambahkan pada uji berikutnya untuk melengkapi jejak pemulihan. Backup manual masih berada di VM yang sama, belum terjadwal/off-host; tidak melindungi kehilangan seluruh VM. Ekspor data tidak otomatis mencadangkan konfigurasi, rules, simulator, atau dashboard. Kontrol ini tidak mengklaim backup produksi lengkap.

Bukti:

- [Data sebelum backup: 72](<evidence/kontrol C2 — backup dan uji pemulihan data Firebase Emulator/before-backup.png>)
- [Ekspor selesai dan metadata backup](<evidence/kontrol C2 — backup dan uji pemulihan data Firebase Emulator/backup-export-success.png>)
- [Perubahan setelah backup: 80](<evidence/kontrol C2 — backup dan uji pemulihan data Firebase Emulator/data-berubah-after-backup.png>)
- [Verifikasi data kembali 72](<evidence/kontrol C2 — backup dan uji pemulihan data Firebase Emulator/restored-data.png>)

### C3. Endpoint proxy sensor hanya untuk membaca — Not Yet

- **Rencana:** periksa blok aktif `location = /api/sensor` pada Nginx. Pastikan metode selain GET/HEAD ditolak dengan `limit_except GET { deny all; }` dan proxy hanya menunjuk path sensor yang diperlukan.
- **Kriteria selesai:** GET dari client yang diizinkan menghasilkan HTTP 200; PATCH/PUT/DELETE melalui `/api/sensor` ditolak (misalnya HTTP 403), data tidak berubah, dan dashboard tetap berfungsi.
- **Bukti:** belum tersedia. Konfigurasi ini pernah disarankan saat membuat dashboard, tetapi belum diperiksa pada VM; jangan diklaim sebagai perubahan baru jika ternyata sudah aktif.

## Prioritas berikutnya

1. Lengkapi screenshot konfigurasi instance/rules tersimpan, dashboard dan simulator setelah B1/B2, pemulihan akses Fail2ban, serta command/log impor backup.
2. Periksa dan kerjakan B3 serta C3 dengan bukti kondisi awal dan verifikasi AFTER; tidak perlu melemahkan konfigurasi yang sudah aman demi screenshot.
3. Tinjau kebutuhan HTTPS, autentikasi aplikasi, pembaruan paket, dan backup off-host sebelum penggunaan di luar lab. Hal-hal tersebut belum dinyatakan selesai dalam checklist ini.
