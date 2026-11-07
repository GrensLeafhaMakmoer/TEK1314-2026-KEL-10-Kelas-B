# Log Purple Team — Simulated SOC Engagement

**Kelompok 10 | TEK1314 — Keamanan Siber | Fase 2**

## 1. Tujuan dan Ruang Lingkup

Dokumen ini mencatat pelaksanaan **lima tahap Purple Team Loop** untuk tiga skenario pada [attack plan](redteam/attack_plan.md). Setiap skenario diuji pada replika Target Node dengan data dummy di jaringan Host-Only laboratorium.

Ruang lingkup terbatas pada:

- Target `192.168.10.5` (Ubuntu Server `srv-iot-kel10`).
- Penguji `192.168.10.100` (CyberOps Workstation).
- Pengamatan trafik melalui Security Onion `192.168.10.200`.
- Tanpa pengujian terhadap sistem produksi, perangkat sensor fisik, atau sistem mitra.

Status pada dokumen ini membedakan **hasil yang sudah memiliki bukti** dari **bagian yang belum diverifikasi**. Bagian yang belum diverifikasi tidak diisi dengan asumsi.

## 2. Lingkungan dan Waktu Pengujian

| Komponen | Keterangan |
|---|---|
| Attacker | CyberOps Workstation — `192.168.10.100` |
| Target | Ubuntu Server `srv-iot-kel10` — `192.168.10.5` |
| Monitoring | Security Onion — `192.168.10.200` |
| Layanan yang diuji | SSH `22`, Nginx `80`, database `9000` |
| Data | Dummy, namespace `demo-melon-lab` |
| Tanggal pengujian | 7 November 2026 (WIB) |
| Kondisi hardening | Checklist A1–A4, B1–B6, C1–C3, C5 aktif |

Waktu pengujian yang tercantum pada screenshot/log:

| Skenario | Waktu tercatat |
|---|---|
| RT-01 | 7 November 2026, 20:10 WIB (13:10 UTC) |
| RT-02 | 7 November 2026, 12:29:16 – 12:32:02 UTC (19:29 – 19:32 WIB) |
| RT-03 | 7 November 2026, 19:39:10 dan 19:39:23 WIB (12:39 UTC) |

## 3. Ringkasan Status

| ID | Kategori | Hasil Serangan | Deteksi | Status Loop |
|---|---|---|---|---|
| RT-01 | Reconnaissance | Port `22` dan `80` terbuka; `9000` filtered; tidak ada versi Nginx yang bocor | Belum diverifikasi di Security Onion | Lengkap di host target; deteksi NIDS belum diverifikasi |
| RT-02 | Exploitation / Akses | Password saja ditolak; key + password salah ditolak; IP penguji diblokir Fail2ban | Terdeteksi — `auth.log` mencatat 5 kegagalan; Fail2ban memblokir `192.168.10.100` | Lengkap (5 tahap terverifikasi dengan bukti log) |
| RT-03 | Web/API | GET tanpa login `401`; PATCH `403`; data tidak berubah | Terdeteksi — `access.log` Nginx mencatat GET 401 dan PATCH 403 | Lengkap (5 tahap terverifikasi dengan bukti log) |

---

## 4. RT-01 — Reconnaissance

### 4.1 Tahap 1 — Serang (Baseline)

Pemindaian port dan pemeriksaan header HTTP dilakukan dari CyberOps (`192.168.10.100`):

```bash
date -Is
nmap -sT -p 22,80,9000 192.168.10.5
curl -sSI http://192.168.10.5/dashboard.html
curl -sSI http://192.168.10.5/auth/login
```

Hasil pemindaian:

| Port | Status | Layanan |
|---|---|---|
| `22/tcp` | open | ssh |
| `80/tcp` | open | http |
| `9000/tcp` | filtered | cslistener |

Respons HTTP:

| URL | Status | Header penting |
|---|---|---|
| `/dashboard.html` | `303 See Other` | `Location: http://192.168.10.5/auth/login`, `Server: nginx`, `X-Frame-Options: DENY` |
| `/auth/login` | `200 OK` | `Server: nginx` tanpa nomor versi, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff` |

### 4.2 Tahap 2 — Deteksi

- **Security Onion:** belum diverifikasi (layar Sguil terkunci saat pengujian). Pemindaian TCP SYN/Connect berpotensi memicu alert Snort/Suricata, namun bukti tangkapan layar belum diambil.
- **Ubuntu:** port `9000` berstatus filtered karena aturan firewall menolak paket dari luar.

### 4.3 Tahap 3 — Analisis

- Port `9000` berstatus **filtered**, konsisten dengan kontrol B1 (UFW) dan B2 (database hanya mendengarkan loopback `127.0.0.1`). Database tidak dapat dijangkau langsung oleh penyerang di jaringan.
- Header `Server: nginx` tidak memuat nomor versi, membuktikan efektivitas kontrol B3 (`server_tokens off`). Penyerang tidak memperoleh informasi versi web server secara pasif.
- Permintaan dashboard tanpa sesi diarahkan ke halaman login (`303`), menunjukkan proteksi autentikasi aktif.
- Header `X-Frame-Options: DENY` aktif pada respons, sesuai kontrol B5 (anti clickjacking).
- **Permukaan serangan yang tersisa:** port `22` dan `80` tetap terbuka karena dibutuhkan untuk administrasi dan akses web pengguna.

### 4.4 Tahap 4 — Respons (Mitigasi)

Tidak diperlukan perubahan konfigurasi. Serangan reconnaissance tidak berhasil memperoleh akses, dan informasi yang terekspos sudah diminimalkan sesuai checklist hardening.

### 4.5 Tahap 5 — Verifikasi Ulang

Pemindaian diulang pada kondisi yang sama dan memberikan hasil konsisten: `22` dan `80` tetap tersedia, `9000` tetap filtered, tidak ada kebocoran versi server.

### Bukti RT-01

- [Hasil pemindaian Nmap dari CyberOps](<redteam/evidence/RT-01/rt01-nmap.png>)
- [Header dashboard dan halaman login Nginx](<redteam/evidence/RT-01/rt01-header.png>)

---

## 5. RT-02 — Percobaan Autentikasi SSH Berulang

### 5.1 Tahap 1 — Serang (Baseline)

Pengujian dilakukan dari CyberOps (`192.168.10.100`) dalam dua kondisi:

**a. Tanpa public key (hanya password):**

```bash
ssh -o PubkeyAuthentication=no iotuser@192.168.10.5
```

Hasil:

```text
iotuser@192.168.10.5: Permission denied (publickey).
```

Server menolak koneksi langsung tanpa menanyakan password karena tahap publickey belum dipenuhi.

**b. Mensimulasikan attacker memiliki public key, lalu mencoba brute force password:**

Key Red Team dipasang sementara, kemudian dilakukan percobaan password salah berulang:

```bash
ssh -i ~/.ssh/pbl_redteam iotuser@192.168.10.5
```

Hasil:
- Setiap percobaan password salah ditolak (`Permission denied, please try again`).
- Setelah akumulasi 5 kegagalan, koneksi ditutup oleh server (`Connection closed by 192.168.10.5 port 22`).

### 5.2 Tahap 2 — Deteksi

Sistem pertahanan Ubuntu mencatat dan merespons serangan:

**a. Log Autentikasi SSH (`/var/log/auth.log`):**

```text
Nov  7 12:29:16 srv-iot-kel10 sshd[3085]: Failed password for iotuser from 192.168.10.100 port 43130 ssh2
Nov  7 12:29:26 srv-iot-kel10 sshd[3085]: message repeated 2 times: [ Failed password for iotuser from 192.168.10.100 port 43130 ssh2]
Nov  7 12:29:28 srv-iot-kel10 sshd[3085]: Connection closed by authenticating user iotuser 192.168.10.100 port 43130 [preauth]
Nov  7 12:29:56 srv-iot-kel10 sshd[3104]: Failed password for iotuser from 192.168.10.100 port 37804 ssh2
Nov  7 12:30:01 srv-iot-kel10 sshd[3104]: Failed password for iotuser from 192.168.10.100 port 37804 ssh2
```

**b. Log Fail2ban (`/var/log/fail2ban.log`):**

```text
2026-11-07 12:29:17,533 fail2ban.filter [724]: INFO [sshd] Found 192.168.10.100 - 2026-11-07 12:29:16
2026-11-07 12:29:22,123 fail2ban.filter [724]: INFO [sshd] Found 192.168.10.100 - 2026-11-07 12:29:21
2026-11-07 12:29:27,412 fail2ban.filter [724]: INFO [sshd] Found 192.168.10.100 - 2026-11-07 12:29:26
2026-11-07 12:29:56,920 fail2ban.filter [724]: INFO [sshd] Found 192.168.10.100 - 2026-11-07 12:29:56
2026-11-07 12:30:02,413 fail2ban.filter [724]: INFO [sshd] Found 192.168.10.100 - 2026-11-07 12:30:01
2026-11-07 12:30:02,501 fail2ban.actions [724]: NOTICE [sshd] Ban 192.168.10.100
2026-11-07 12:32:02,109 fail2ban.actions [724]: NOTICE [sshd] Unban 192.168.10.100
```

Tepat pada percobaan ke-5 (12:30:02 UTC), Fail2ban menjatuhkan sanksi `Ban 192.168.10.100` selama `bantime=120` detik hingga `Unban` pada 12:32:02 UTC.

### 5.3 Tahap 3 — Analisis

- Kontrol A4 (`AuthenticationMethods publickey,password`) berhasil mencegah serangan brute-force berbasis password murni (seperti tools Hydra standar). Tanpa memiliki private key yang terdaftar, percobaan password tidak pernah dievaluasi.
- Saat penyerang diasumsikan memiliki key, lapisan kedua (password) menahan akses tak berwenang.
- Filter Fail2ban `sshd` berhasil mencocokkan kegagalan password pada `systemd journal` (`_SYSTEMD_UNIT=sshd.service`), menghitung `maxretry=5`, dan mengeksekusi aksi pemblokiran iptables.
- Koneksi ditutup (`Connection closed ... [preauth]`) terjadi saat `LoginGraceTime 30` detik habis atau saat aturan blokir mulai aktif.

### 5.4 Tahap 4 — Respons (Mitigasi)

- Konfigurasi Fail2ban terbukti efektif menahan serangan tebak kredensial.
- Setelah pengujian selesai, key penguji (`cyberops-redteam`) segera dihapus dari `/home/iotuser/.ssh/authorized_keys` menggunakan perintah `sed -i '/cyberops-redteam/d' ~/.ssh/authorized_keys`.
- Verifikasi pasca-pengujian memastikan hanya 2 key sah yang tersisa (`thinkpad@GIEE` dan `windows-pbl-key`).

### 5.5 Tahap 5 — Verifikasi Ulang

Setelah key CyberOps dicabut, CyberOps kembali diuji mencoba login SSH: koneksi langsung ditolak dengan `Permission denied (publickey)` tanpa pernah meminta password.

### Bukti RT-02

- [Akses SSH tanpa public key ditolak (Permission denied)](<redteam/evidence/RT-02/rt02-ssh-tanpa-key.png>)
- [Log autentikasi auth.log mencatat 5 kegagalan password](<redteam/evidence/RT-02/rt02-log-auth.png>)
- [Log Fail2ban mencatat Found, Ban 192.168.10.100, dan Unban](<redteam/evidence/RT-02/rt02-log-fail2ban.png>)

---

## 6. RT-03 — Percobaan Manipulasi Data API

### 6.1 Tahap 1 — Serang (Baseline)

Percobaan manipulasi data sensor dijalankan dari CyberOps (`192.168.10.100`) menggunakan HTTP GET dan PATCH tanpa kredensial sesi:

```bash
date -Is
curl -sS -o /dev/null -w 'GET  /api/sensor -> HTTP %{http_code}\n' http://192.168.10.5/api/sensor
curl -sS -o /dev/null -w 'PATCH /api/sensor -> HTTP %{http_code}\n' \
  -X PATCH -H 'Content-Type: application/json' \
  -d '{"humidity":55}' http://192.168.10.5/api/sensor
```

Hasil pada `2026-11-07T19:39:10+07:00` dan diulang `19:39:23+07:00`:

```text
GET   /api/sensor -> HTTP 401
PATCH /api/sensor -> HTTP 403
```

Payload `{"humidity":55}` ditolak langsung pada gerbang web server.

### 6.2 Tahap 2 — Deteksi

Bukti deteksi tercatat langsung pada `/var/log/nginx/access.log` di server Ubuntu:

```text
192.168.10.100 - - [07/Nov/2026:12:39:08 +0000] "GET /api/sensor HTTP/1.1" 401 172 "-" "curl/7.81.0"
192.168.10.100 - - [07/Nov/2026:12:39:08 +0000] "PATCH /api/sensor HTTP/1.1" 403 146 "-" "curl/7.81.0"
192.168.10.100 - - [07/Nov/2026:12:39:10 +0000] "GET /api/sensor HTTP/1.1" 401 172 "-" "curl/7.81.0"
192.168.10.100 - - [07/Nov/2026:12:39:10 +0000] "PATCH /api/sensor HTTP/1.1" 403 146 "-" "curl/7.81.0"
192.168.10.100 - - [07/Nov/2026:12:39:23 +0000] "GET /api/sensor HTTP/1.1" 401 172 "-" "curl/7.81.0"
192.168.10.100 - - [07/Nov/2026:12:39:23 +0000] "PATCH /api/sensor HTTP/1.1" 403 146 "-" "curl/7.81.0"
```

Setiap request dari `192.168.10.100` tercatat lengkap dengan timestamp, metode HTTP, URL endpoint, kode status HTTP (`401` dan `403`), serta User-Agent `curl/7.81.0`.

### 6.3 Tahap 3 — Analisis

- `GET` tanpa sesi menghasilkan `401 Unauthorized` karena direktif `auth_request` di Nginx mewajibkan cookie autentikasi yang valid.
- `PATCH` menghasilkan `403 Forbidden` karena blok `limit_except GET { deny all; }` menolak secara eksplisit semua metode selain `GET` pada lokasi `/api/`.
- Penolakan metode HTTP non-GET dievaluasi sebelum diteruskan ke backend database. Dengan demikian, muatan berbahaya tidak pernah menyentuh Firebase Emulator.
- **Catatan pembuktian integritas:** Karena simulator secara aktif memperbarui parameter nutrisi/kelembapan secara periodik, integritas data tidak dibuktikan dari angka absolut yang tetap, melainkan dari fakta bahwa request penulisan penyerang mendapatkan penolakan `403` dan tidak ada modifikasi sewenang-wenang (`55`).

### 6.4 Tahap 4 — Respons (Mitigasi)

Arsitektur reverse proxy dengan pembatasan metode baca-saja (`limit_except GET`) dan verifikasi autentikasi (`auth_request`) terbukti solid melindungi antarmuka IoT. Tidak diperlukan perubahan rule tambahan.

### 6.5 Tahap 5 — Verifikasi Ulang

Pengujian kedua pada `19:39:23` WIB menghasilkan kode status identik (`401` dan `403`), dan tercatat berurutan pada log akses Nginx.

### Bukti RT-03

- [Percobaan GET dan PATCH dari CyberOps (HTTP 401 dan 403)](<redteam/evidence/RT-03/rt03-curl-401-403.png>)
- [Log Nginx access.log mencatat percobaan serangan dan status 403](<redteam/evidence/RT-03/rt03-log-nginx-403.png>)

---

## 7. Temuan dan Keterbatasan

| Temuan | Dampak | Tindak Lanjut |
|---|---|---|
| Kontrol A4 (`publickey,password`) meniadakan serangan brute force password murni | Penyerang diwajibkan memiliki private key terlebih dahulu; alat brute-force standar tidak relevan | Dokumentasikan arsitektur two-factor key+password sebagai pertahanan utama SSH |
| Fail2ban aktif memblokir IP setelah 5 kegagalan autentikasi | Upaya eksploitasi dihentikan di tingkat kernel/iptables selama 120 detik | Monitor `/var/log/fail2ban.log` secara periodik |
| Nginx `limit_except GET` menolak modifikasi API dengan `403 Forbidden` | Penyerang tidak dapat mengubah status atau data sensor tanpa akses fisik/lokal | Pertahankan konfigurasi read-only pada reverse proxy |
| Log Nginx dan log auth berhasil mencatat jejak serangan secara detail | Forensik dan deteksi insiden dapat dilakukan secara akurat dari file log host | Integrasikan pengiriman log ke sentralisasi log jika diperlukan |
| Layar Security Onion (Sguil) terkunci saat demonstrasi | Alert jaringan Snort/Suricata belum diverifikasi pada sesi ini | Verifikasi status alert NIDS pada sesi pengujian terjadwal berikutnya |

## 8. Rencana Tindak Lanjut

1. Menyelesaikan dokumentasi laporan kemajuan Fase 2 pada `LOGBOOK.md`.
2. Melakukan commit dan push berkas dokumentasi serta bukti screenshot log ke repository GitHub tim.
3. Memastikan VM lab tetap dalam kondisi hardened (key sementara Red Team sudah dibersihkan).
