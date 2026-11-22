# Catatan Remediasi Minggu 13 — Replika Sistem IoT

- **Proyek / Kelompok:** TEK1314-2026 Kelompok 10 — Kelas B (`demo-melon-lab`, lingkungan lab `192.168.10.0/24`)
- **Tanggal pelaksanaan:** 22 November 2026 (WIB `UTC+7` dan UTC)
- **Host terdampak / diverifikasi:**
  - `SRV-IOT-KEL10` (`192.168.10.5`, domain lab `melonlab.local`) — Web/IoT Target (Ubuntu)
  - `SO-KEL10` (`192.168.10.200`) — Security Onion (Snort NIDS & Syslog Collector)
  - `CYBEROPS-KEL10` (`192.168.10.100`) — Klien pengujian / CyberOps
- **Rujukan temuan:** [Vulnerability Assessment Report](../reports/vulnerability-assessment-report.md), [Draf Incident Report](../incident-response/incident-report-draft.md), [Manifest Bukti Forensik](../incident-response/evidence-manifest.md), dan [Hardening Checklist](hardening_checklist.md).

---

## 1. Ringkasan Eksekutif & Status Remediasi

| Prioritas / ID | Item Remediasi | Kondisi Sebelum (Before) | Kondisi Sesudah (After) | Status |
|---|---|---|---|---|
| P1 (`F-06`) | Rule kustom NIDS & uji ulang RT-02 / RT-03 | Mengandalkan rule default; rekomendasi VAR menyebut contoh Suricata | 2 rule kustom (`sid:1000010`, `sid:1000011`) aktif di Snort `2.9.15.1` (`SO-KEL10`), memicu alert pada `securityonion_db` saat uji ulang RT-02/RT-03 (`09:40:22 UTC` / `16:40:22 WIB`) | **Selesai** |
| P1 (`F-05`) | Sentralisasi log target ke Security Onion | Log hanya tersimpan lokal di `SRV-IOT-KEL10` (`/var/log/`) | `rsyslogd` (`imfile` + `omfwd` TCP `:514`, antrean disk `so_fwd`) aktif ke `192.168.10.200`; `syslog-ng` menyimpan log ke `/var/log/remote/srv-iot-kel10.log` dengan rotasi harian 30 hari | **Selesai** |
| P2 (`F-03`) | Hardening Fail2ban (`bantime`, increment, `recidive`) | `bantime` dasar 120 detik | `sshd` `bantime = 3600`, `bantime.increment = true` (`factor = 2`, `maxtime = 86400`), dan jail `recidive` aktif (`maxretry = 5`, `findtime = 86400`, `bantime = 604800`, ban allports) | **Selesai** |
| P2 (`F-02` & B2) | Verifikasi binding Firebase Emulator ke loopback | Triage 13 Nov (`evidence-manifest.md` §4) menemukan `[::]:9000` (`pid=1787`) | `firebase.json` (`127.0.0.1:9000`) terverifikasi aktif di `[::ffff:127.0.0.1]:9000` (`java`, `pid=5816`); Nmap dari `CYBEROPS-KEL10` menunjukkan `9000/tcp filtered` | **Selesai** |
| P2 (`F-07`) & Domain | Aktifkan TLS, HSTS, cookie `Secure`, dan domain `melonlab.local` | HTTP port 80 tanpa TLS dan akses menggunakan IP `192.168.10.5` | Sertifikat CA lokal (`CN=melonlab.local`, SAN `melonlab.local`, `srv-iot-kel10`, `192.168.10.5`), redirect `308` ke `https://melonlab.local`, HSTS `max-age=31536000`, `X-Frame-Options: DENY`, dan `AUTH_ORIGIN=https://melonlab.local` (`Set-Cookie: ...; HttpOnly; SameSite=Lax; Secure`) | **Selesai** |

> [!NOTE]
> Seluruh konfigurasi yang diubah dicadangkan terlebih dahulu ke `/var/backups/pbl-minggu13/` pada masing-masing VM. Aturan pengujian internal (`grep -c "Failed password"` = 4 baris / 5 percobaan) dan file yang dilindungi tidak diubah.

---

## 2. Detail Implementasi & Verifikasi

### 2.1. P1 — Rule Kustom NIDS (`F-06`) dan Uji Ulang RT-02 / RT-03

1. **Penyesuaian Engine (`Snort 2` vs draf Suricata di VAR §6):**
   - Pemeriksaan pada `SO-KEL10` menunjukkan sensor menjalankan **`ENGINE=snort`** (`Snort 2.9.15.1 GRE Build 15125`) pada konfigurasi `/etc/nsm/seconion-eth0/snort.conf`, dengan `HOME_NET [192.168.0.0/16,10.0.0.0/8,172.16.0.0/12]` dan `EXTERNAL_NET any`.
   - Skrip `nsm_sensor_ps-restart` dan `nsm_sensor_ps-start` di Security Onion secara eksplisit mengomentari baris `#include $RULE_PATH/local.rules` dan menggabungkan rule aktif melalui `/etc/nsm/rules/downloaded.rules`. Selain itu, daftar `http_methods` di `/etc/nsm/seconion-eth0/snort.conf` tidak menyertakan metode `PATCH`, sehingga keyword `http_method` tidak mencocokkan `PATCH`.
2. **Implementasi pada `SO-KEL10`:**
   - Backup `/etc/nsm/rules/local.rules`, `/etc/nsm/rules/sid-msg.map`, `/etc/nsm/rules/downloaded.rules`, dan `/etc/nsm/seconion-eth0/snort.conf` ke `/var/backups/pbl-minggu13/`.
   - Menulis dua rule Snort 2 di `/etc/nsm/rules/local.rules`, menggabungkannya ke `/etc/nsm/rules/downloaded.rules`, dan mendaftarkan deskripsi SID di `/etc/nsm/rules/sid-msg.map`:
     - `alert tcp $EXTERNAL_NET any -> $HOME_NET 80 (msg:"LOCAL IoT API PATCH attempt on read-only endpoint"; flow:to_server,established; content:"PATCH "; depth:6; content:"/api/"; distance:0; within:32; classtype:web-application-attack; sid:1000010; rev:1;)`
     - `alert tcp $EXTERNAL_NET any -> $HOME_NET 22 (msg:"LOCAL SSH repeated authentication attempt"; flow:to_server,established; content:"SSH-"; depth:4; threshold:type both, track by_src, count 5, seconds 60; classtype:attempted-admin; sid:1000011; rev:1;)`
   - Restart sensor dengan `sudo nsm_sensor_ps-restart --only-snort-alert && sudo nsm_sensor_ps-restart --only-barnyard2` (`[ OK ]`).
3. **Verifikasi Ulang RT-02 & RT-03 (`2026-11-22T09:40:22Z` / `2026-11-22T16:40:22+07:00`):**
   - Dari `CYBEROPS-KEL10` (`192.168.10.100`): eksekusi `PATCH http://192.168.10.5/api/sensor` mengembalikan **`PATCH:403`** ([01-rt02-rt03-retest-cyberops.png](evidence/remediation-minggu13/01-rt02-rt03-retest-cyberops.png)), dan 5 percobaan SSH (`PubkeyAuthentication=no`) ditolak (`Permission denied (publickey).`).
   - Query ke `securityonion_db` di `SO-KEL10` ([02-snort-custom-rules-securityonion-db.png](evidence/remediation-minggu13/02-snort-custom-rules-securityonion-db.png)) serta antarmuka **SGUIL-0.9.0** ([08-so-sguil-gui-custom-alerts.png](evidence/remediation-minggu13/08-so-sguil-gui-custom-alerts.png), `2026-11-22 10:34:48 GMT` / `17:34 WIB`) merekam kedua alert kustom pada sensor `seconion-eth0`:
     - `2026-11-22 09:40:23 UTC` (`16:40:23 WIB`) — Alert ID `3.4342`, `signature_id 1000010` (`LOCAL IoT API PATCH attempt on read-only endpoint`), `192.168.10.100:48998 -> 192.168.10.5:80`
     - `2026-11-22 09:40:24 UTC` (`16:40:24 WIB`) — Alert ID `3.4347`, `signature_id 1000011` (`LOCAL SSH repeated authentication attempt`), `192.168.10.100:46292 -> 192.168.10.5:22`
4. **Catatan Keterbatasan Pasca-TLS (Port 443):**
   - Setelah trafik HTTP (`80`) dialihkan (`308`) ke HTTPS (`443`), payload HTTP terenkripsi sehingga NIDS pasif (`sid:1000010`) tetap bertugas menangkap percobaan `PATCH /api/` plaintext pada port `80`, sementara `sid:1000011` memantau SSH port `22`. Visibilitas metode & status HTTP/HTTPS Layer-7 (`PATCH /api/sensor 403`) dipastikan tetap terekam melalui sentralisasi `nginx-access` dan `nginx-error` ke `SO-KEL10`.

### 2.2. P1 — Sentralisasi Log Target ke Security Onion (`F-05`)

1. **Pengirim Log (`SRV-IOT-KEL10`):**
   - Membuat `/etc/rsyslog.d/60-security-onion.conf` menggunakan modul `imfile` (`PollingInterval="2"`) untuk membaca:
     - `/var/log/fail2ban.log` (`Tag="fail2ban-log:"`)
     - `/var/log/nginx/access.log` (`Tag="nginx-access:"`)
     - `/var/log/nginx/error.log` (`Tag="nginx-error:"`)
   - Meneruskan seluruh log (`*.*`) via TCP `omfwd` ke `192.168.10.200:514` dengan antrean disk-assisted `so_fwd`.
   - Verifikasi: `sudo rsyslogd -N1` lolos; `sudo ss -tnp | grep :514` menampilkan koneksi aktif `ESTAB 192.168.10.5:57208 192.168.10.200:514` (`users:(("rsyslogd",pid=8098,fd=19))`).
2. **Penerima & Retensi (`SO-KEL10`):**
   - Membuka UFW `514/tcp` khusus dari `192.168.10.5`.
   - Backup `/etc/syslog-ng/syslog-ng.conf` ke `/var/backups/pbl-minggu13/`, lalu menambahkan `filter f_srv_iot_kel10` (`netmask("192.168.10.5/32")`) dan `destination d_srv_iot_kel10` ke `/var/log/remote/srv-iot-kel10.log` (`create-dirs(yes)`, `perm(0640)`).
   - Menambahkan kebijakan rotasi `/etc/logrotate.d/srv-iot-kel10` (`daily`, `rotate 30`, `compress`, `delaycompress`, `missingok`, `notifempty`, `postrotate` reload `syslog-ng`).
   - Verifikasi ([03-central-syslog-so-initial.png](evidence/remediation-minggu13/03-central-syslog-so-initial.png) dan [04-snort-rules-logrotate-and-https-syslog-so.png](evidence/remediation-minggu13/04-snort-rules-logrotate-and-https-syslog-so.png)): `syslog-ng -s` lolos dan file `/var/log/remote/srv-iot-kel10.log` (`234K`) aktif menerima entri real-time `sshd`, `sudo`, `systemd`, `nginx-access`, dan `nginx-error` termasuk akses HTTPS ke domain `melonlab.local` pada `2026-11-22T10:24:45–46Z` (`17:24:45–46 WIB`).

### 2.3. P2 — Hardening Fail2ban (`F-03`)

1. Backup `/etc/fail2ban/jail.d/pbl-sshd.local` ke `/var/backups/pbl-minggu13/` pada `SRV-IOT-KEL10`.
2. Konfigurasi baru pada `/etc/fail2ban/jail.d/pbl-sshd.local`:
   - `[sshd]`: `enabled = true`, `port = 22`, `backend = systemd`, `maxretry = 5`, `findtime = 600`, **`bantime = 3600`**, **`bantime.increment = true`**, `bantime.factor = 2`, `bantime.maxtime = 86400`.
   - **`[recidive]`**: `enabled = true`, `logpath = /var/log/fail2ban.log`, `banaction = %(banaction_allports)s`, `bantime = 604800`, `findtime = 86400`, `maxretry = 5`.
3. Verifikasi (`2026-11-22T10:26:08Z` / `2026-11-22T17:26:08+07:00`, [05-fail2ban-and-rsyslog-srv.png](evidence/remediation-minggu13/05-fail2ban-and-rsyslog-srv.png)):
   - `sudo fail2ban-client status` -> `Number of jail: 2` (`recidive, sshd`)
   - `sudo fail2ban-client get sshd bantime` -> `3600`
   - `sudo fail2ban-client get sshd bantime.increment` -> `True`
   - `sudo fail2ban-client status recidive` memantau `/var/log/fail2ban.log`.

### 2.4. P2 — Verifikasi Binding Loopback Firebase Database (`F-02` & Penyelarasan B2)

1. Konfigurasi `/home/iotuser/melon-lab/firebase.json` terkonfirmasi memiliki `"host": "127.0.0.1"` dan `"port": 9000`.
2. Verifikasi pada `SRV-IOT-KEL10` (`2026-11-22T10:24:42Z` / `17:24:42 WIB`, [06-srv-tls-cert-listeners-loopback.png](evidence/remediation-minggu13/06-srv-tls-cert-listeners-loopback.png)): `sudo ss -tulpn` menunjukkan listener **`[::ffff:127.0.0.1]:9000`** (`users:(("java",pid=5816,fd=18))`), tidak lagi wildcard `[::]:9000` seperti saat triage 13 November 2026 (`pid=1787`).
3. Verifikasi eksternal dari `CYBEROPS-KEL10` (`17:24:42 WIB`, [07-cyberops-nmap-https-hsts-secure-cookie.png](evidence/remediation-minggu13/07-cyberops-nmap-https-hsts-secure-cookie.png)): `nmap -sT -p 22,80,443,9000 melonlab.local` menampilkan **`9000/tcp filtered cslistener`**. Kontrol B2 pada [Hardening Checklist](hardening_checklist.md#b2-database-hanya-mendengarkan-pada-localhost--done) telah diselaraskan.

### 2.5. P2 — TLS + HSTS + Cookie `Secure` + Domain `melonlab.local` (`F-07`)

1. **Resolusi Nama Domain Lab:** Menambahkan pemetaan `192.168.10.5 melonlab.local` pada `/etc/hosts` ketiga VM (`SRV-IOT-KEL10`, `SO-KEL10`, `CYBEROPS-KEL10`). Untuk membuka web dari peramban host Windows di luar VM, tambahkan entri `192.168.10.5 melonlab.local` pada berkas `hosts` Windows (dan impor `/etc/nginx/pbl-tls/ca.crt` jika ingin meniadakan peringatan CA lokal).
2. **Sertifikat & Konfigurasi TLS Nginx (`SRV-IOT-KEL10`):**
   - Menghasilkan sertifikat `/etc/nginx/pbl-tls/server.crt` yang ditandatangani `/etc/nginx/pbl-tls/ca.crt` dengan `subject=CN = melonlab.local, O = PBL Kel10`, masa berlaku `Nov 22 10:04:55 2026 GMT` hingga `Feb 20 10:04:55 2027 GMT`, dan `Subject Alternative Name`: `DNS:melonlab.local, DNS:srv-iot-kel10, IP Address:192.168.10.5`.
   - Membuat `/etc/nginx/snippets/pbl-tls.conf` (`ssl_protocols TLSv1.2 TLSv1.3;`, `ssl_session_tickets off;`, `add_header Strict-Transport-Security "max-age=31536000" always;`).
   - Membackup `/etc/nginx/sites-available/default` ke `/var/backups/pbl-minggu13/nginx-default.before-tls`, lalu mengonfigurasi server block `443 ssl` untuk `server_name melonlab.local;` serta block `default_server` (`80` dan `443 ssl`, `server_name _;`, `server_tokens off;`) yang mengalihkan akses HTTP maupun akses IP langsung dengan `return 308 https://melonlab.local$request_uri;`.
   - Membuka UFW `443/tcp` dari `192.168.10.100` dan `192.168.10.1`.
3. **Penyelarasan Layanan Autentikasi (`pbl-web-auth`):**
   - Menambahkan override systemd `/etc/systemd/system/pbl-web-auth.service.d/https.conf` berisi `Environment=AUTH_ORIGIN=https://melonlab.local` agar validasi `Origin` menerima `https://melonlab.local` dan flag `Secure` aktif otomatis pada cookie sesi serta `lab_csrf`.
4. **Verifikasi Eksternal dari `CYBEROPS-KEL10` (`2026-11-22T10:24:42Z` / `17:24:42 WIB`):**
   - `nmap -sT -p 22,80,443,9000 melonlab.local`: `22/tcp open ssh`, `80/tcp open http`, `443/tcp open https`, `9000/tcp filtered cslistener`.
   - `curl -sI http://192.168.10.5/auth/login`: `HTTP/1.1 308 Permanent Redirect` dan `Location: https://melonlab.local/auth/login`.
   - `curl -skI https://melonlab.local/auth/login`: `HTTP/1.1 200 OK`, `Strict-Transport-Security: max-age=31536000`, `X-Frame-Options: DENY`, dan `Set-Cookie: lab_csrf=...; Path=/auth; Max-Age=900; HttpOnly; SameSite=Lax; Secure`.
   - Uji API tanpa sesi pada `https://melonlab.local/api/sensor`: `GET:401` dan `PATCH:403`.
   - Verifikasi browser Firefox di `CYBEROPS-KEL10` (`17:34:48 WIB` / `10:34:48 UTC`, [09-firefox-https-melonlab-local.png](evidence/remediation-minggu13/09-firefox-https-melonlab-local.png)): dashboard `https://melonlab.local` berhasil login, memuat riwayat `10 /10 DATA` dari Firebase Realtime Database Emulator melalui HTTPS.

---

## 3. Daftar Bukti & Hash SHA-256

Seluruh tangkapan layar verifikasi disimpan di `docs/blueteam/evidence/remediation-minggu13/`:

| Berkas | Deskripsi Verifikasi | Waktu Acuan (UTC / WIB) | SHA-256 |
|---|---|---|---|
| [01-rt02-rt03-retest-cyberops.png](evidence/remediation-minggu13/01-rt02-rt03-retest-cyberops.png) | Uji ulang RT-02/RT-03 dari `CYBEROPS-KEL10` (`PATCH:403` dan 5× SSH) | `2026-11-22 09:40:22 UTC` / `16:40:22 WIB` | `0ba262d1f9600fef7099269c74fc6e550f75e2f59e9a547f19ae8678453af4ec` |
| [02-snort-custom-rules-securityonion-db.png](evidence/remediation-minggu13/02-snort-custom-rules-securityonion-db.png) | Query `securityonion_db.event` menampilkan alert `1000010` & `1000011` | `2026-11-22 09:40:23–24 UTC` / `16:40:23–24 WIB` | `371f76e7a6086f891e4e0fd002f2b309294b9833f84dc45b3997a72e35690186` |
| [03-central-syslog-so-initial.png](evidence/remediation-minggu13/03-central-syslog-so-initial.png) | Log `sshd` dan `nginx-access` di `/var/log/remote/srv-iot-kel10.log` | `2026-11-22 09:18 UTC` / `16:18 WIB` | `03e9e2be302acba0f5ba98bd9ac307596dfec0da5e96d0cb4e7e954d3b5d42eb` |
| [04-snort-rules-logrotate-and-https-syslog-so.png](evidence/remediation-minggu13/04-snort-rules-logrotate-and-https-syslog-so.png) | Isi `local.rules`, `logrotate.d/srv-iot-kel10`, & log HTTPS `melonlab.local` di `SO-KEL10` | `2026-11-22 10:24:45–46 UTC` / `17:24:45–46 WIB` | `70db162a4d0961825fb7878eab287c8b7d57b690aff331bb4757dd53fb8a6af2` |
| [05-fail2ban-and-rsyslog-srv.png](evidence/remediation-minggu13/05-fail2ban-and-rsyslog-srv.png) | Status Fail2ban (`recidive, sshd`, `3600`, `True`) & `rsyslogd` `ESTAB :514` | `2026-11-22 10:26:08 UTC` / `17:26:08 WIB` | `c273effcaa6a6aec88b93c463f50f8a5620819ffde86d319c2b8e68dbfed96c1` |
| [06-srv-tls-cert-listeners-loopback.png](evidence/remediation-minggu13/06-srv-tls-cert-listeners-loopback.png) | Listener `[::ffff:127.0.0.1]:9000`, `127.0.0.1:9081`, `443`, cert SAN `melonlab.local`, `AUTH_ORIGIN` | `2026-11-22 10:24:42 UTC` / `17:24:42 WIB` | `236324d943b0cbbcca2ceaec29a4bdec50972d78ac8986e3954690fc94a38ab6` |
| [07-cyberops-nmap-https-hsts-secure-cookie.png](evidence/remediation-minggu13/07-cyberops-nmap-https-hsts-secure-cookie.png) | Nmap (`9000 filtered`), redirect `308`, HTTPS `200`, HSTS, cookie `Secure`, `GET:401`/`PATCH:403` | `2026-11-22 10:24:42 UTC` / `17:24:42 WIB` | `7389f95f4afad68c13f52d631147799ce9d9e08ce69b1e0a050bb9ca856baeb5` |
| [08-so-sguil-gui-custom-alerts.png](evidence/remediation-minggu13/08-so-sguil-gui-custom-alerts.png) | Konsol GUI `SGUIL-0.9.0` menampilkan event real-time `3.4342` (`sid:1000010`) & `3.4347` (`sid:1000011`) | `2026-11-22 10:34:48 UTC` / `17:34 WIB` | `8d6f33e3cd1ed4a9e602bba9b307199091a1a587981b2bf23fa88aa9f7a8161a` |
| [09-firefox-https-melonlab-local.png](evidence/remediation-minggu13/09-firefox-https-melonlab-local.png) | Akses browser HTTPS ke `https://melonlab.local` (`MelonLab — Monitoring Nutrisi`, terhubung & login) | `2026-11-22 10:34:48 UTC` / `17:34:48 WIB` | `fe641a146849fc27aa61bc4dc0e952f40ce8720bfaabc637a6dfe2e02950269c` |
