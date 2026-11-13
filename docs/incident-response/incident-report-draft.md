# Draf Incident Report — NIST SP 800-61r2

**Kelompok 10 | TEK1314 — Keamanan Siber | Fase 3 (Minggu 12)**

| Item | Keterangan |
|---|---|
| ID Insiden | `INC-2026-11-001` |
| Judul | Upaya akses tidak sah dan manipulasi data pada replika backend IoT |
| Tanggal insiden | 7 November 2026 |
| Tanggal deteksi | 7 November 2026, 12:29:16 UTC |
| Tanggal laporan | 13 November 2026 (draf) |
| Pelapor | Blue Team — Binggie Rashel Prasetyo |
| Klasifikasi | Simulasi terkontrol (Red Team internal), bukan insiden produksi |
| Severity | **Low–Medium** (tidak ada akses tidak sah, tidak ada perubahan data) |
| Status | **Closed (simulasi)** — penyelidikan forensik akuisisi bukti masih tertunda |
| Target | `192.168.10.5` (`srv-iot-kel10`) — Ubuntu Server replika |
| Sumber serangan | `192.168.10.100` — CyberOps Workstation (Red Team) |
| Dasar | [Purple Team Log](../purple-team-log.md), [VAR Fase 2](../reports/vulnerability-assessment-report.md) |

> **Pernyataan lingkup:** seluruh aktivitas dalam laporan ini adalah simulasi yang disengaja pada replika laboratorium dengan data dummy. Tidak ada sistem produksi, sensor fisik, pompa, atau aset mitra yang terlibat.

---

## 1. Ringkasan Eksekutif

Pada 7 November 2026, Red Team internal menjalankan rangkaian simulasi serangan tiga tahap terhadap replika backend IoT: pemindaian layanan (reconnaissance), percobaan autentikasi SSH berulang, dan percobaan manipulasi data melalui API. Blue Team mendeteksi seluruh aktivitas melalui log host dan konsol Sguil di Security Onion.

**Tidak ada** akses tidak sah yang berhasil, **tidak ada** data yang berubah, dan **tidak ada** layanan yang terganggu. Fail2ban memblokir sumber serangan selama 120 detik, dan Nginx menolak seluruh permintaan tulis ke endpoint read-only.

Insiden ditutup sebagai simulasi yang berhasil dipertahankan. Namun laporan ini mencatat **kesenjangan kesiapan forensik**: bukti masih berupa tangkapan layar, belum ada akuisisi image disk/memori, dan log belum tersentralisasi.

---

## 2. Klasifikasi dan Severity

| Dimensi | Penilaian |
|---|---|
| Confidentiality | Tidak terdampak — tidak ada data terbaca tanpa otorisasi (`401`) |
| Integrity | Tidak terdampak — seluruh PATCH ditolak (`403`) |
| Availability | Tidak terdampak — layanan tetap `200`/`303` normal |
| Severity keseluruhan | **Low–Medium** |
| Alasan | Serangan terdeteksi dan tertahan pada lapisan pertama; namun pola serangan berulang menunjukkan kebutuhan monitoring berkelanjutan |

---

## 3. Fase 1 — Preparation

### 3.1 Kesiapan yang Sudah Ada

| Komponen | Kondisi saat insiden |
|---|---|
| Hardening | 14 kontrol (A1–A4, B1–B6, C1–C3, C5) aktif dan terverifikasi |
| Baseline trafik | Terdokumentasi pada [baseline traffic](../blueteam/baseline_traffic.md) |
| Monitoring | Security Onion + Sguil aktif; ruleset default Snort/Suricata |
| Logging | `auth.log`, `fail2ban.log`, `nginx/access.log` aktif di host target |
| Kontak tim | Lead, Red Team, dan Blue Team dalam satu sesi lab terjadwal |
| Batas hukum/etika | Pengujian hanya pada replika dan data dummy |

### 3.2 Kesenjangan Kesiapan (Gap)

| Gap | Dampak pada penanganan |
|---|---|
| Belum ada prosedur akuisisi bukti (imaging, hashing, chain of custody) | Bukti utama masih berupa screenshot, bukan salinan log ter-hash |
| Log belum tersentralisasi | Jika host dikompromikan, bukti dapat hilang |
| Belum ada alert rule kustom | Deteksi bergantung pada signature default |
| Belum ada runbook IR tertulis | Eskalasi dan kontak masih informal |

---

## 4. Fase 2 — Detection and Analysis

### 4.1 Sumber Bukti

| No | Sumber | Lokasi | Jenis | Status |
|---|---|---|---|---|
| E-01 | `auth.log` | `/var/log/auth.log` (host target) | Log autentikasi | Tersedia (screenshot) |
| E-02 | `fail2ban.log` | `/var/log/fail2ban.log` | Log aksi pemblokiran | Tersedia (screenshot) |
| E-03 | `access.log` | `/var/log/nginx/access.log` | Log akses HTTP | Tersedia (screenshot) |
| E-04 | Sguil realtime events | Security Onion `192.168.10.200` | Alert NIDS | Tersedia (screenshot) |
| E-05 | Output Nmap | CyberOps `192.168.10.100` | Hasil pemindaian | Tersedia (screenshot) |
| E-06 | Output curl | CyberOps `192.168.10.100` | Hasil request HTTP | Tersedia (screenshot) |
| E-07 | Salinan log ter-hash | Belum diakuisisi | Bukti forensik | **Tertunda** |
| E-08 | Image disk / memori | Belum diakuisisi | Bukti forensik | **Tertunda** |

Hash SHA-256 seluruh bukti screenshot tercatat pada [manifest bukti](evidence-manifest.md).

### 4.2 Timeline Insiden

Seluruh waktu host target menggunakan **UTC**; CyberOps dan Security Onion menggunakan **WIB (UTC+7)**. Konversi dilakukan eksplisit untuk menghindari salah korelasi.

| Waktu UTC | Waktu WIB | Aktor | Peristiwa | Bukti |
|---|---|---|---|---|
| 12:29:16 | 19:29:16 | Red | Percobaan password SSH gagal ke-1 (`iotuser`, port 43130) | E-01 |
| 12:29:17 | 19:29:17 | Blue | Fail2ban mencatat `Found 192.168.10.100` | E-02 |
| 12:29:21 | 19:29:21 | Red | Percobaan password gagal ke-2 | E-02 |
| 12:29:26 | 19:29:26 | Red | Percobaan gagal ke-3 (log: `message repeated 2 times`) | E-01 |
| 12:29:27 | 19:29:27 | Blue | Fail2ban `Found` ke-3 | E-02 |
| 12:29:28 | 19:29:28 | Target | Koneksi ditutup `[preauth]` | E-01 |
| 12:29:56 | 19:29:56 | Red | Percobaan gagal ke-4 (port 37804) | E-01 |
| 12:30:01 | 19:30:01 | Red | Percobaan gagal ke-5 | E-01 |
| **12:30:02** | **19:30:02** | Blue | **`Ban 192.168.10.100`** (maxretry=5 tercapai) | E-02 |
| 12:32:02 | 19:32:02 | Blue | `Unban 192.168.10.100` (bantime 120 detik berakhir) | E-02 |
| 12:39:08 | 19:39:08 | Red | `GET /api/sensor` → `401`; `PATCH /api/sensor` → `403` | E-03 |
| 12:39:10 | 19:39:10 | Red | Pengulangan GET `401` dan PATCH `403` | E-03 |
| 12:39:23 | 19:39:23 | Red | Pengulangan GET `401` dan PATCH `403` | E-03 |
| ±13:10 | ±20:10 | Red | Pemindaian Nmap `22,80,9000` → `22` open, `80` open, `9000` filtered | E-05 |
| 13:20:48 | 20:20:48 | Blue | Sguil mengonfirmasi event NIDS terekam | E-04 |

**Catatan korelasi:** `Ban` terjadi pada detik yang sama dengan percobaan gagal ke-5 (12:30:02), konsisten dengan `maxretry=5`. Tidak ada aktivitas SSH dari IP tersebut setelah `Unban` pada 12:32:02.

### 4.3 Indikator Kompromi (IOC)

| Tipe | Nilai | Konteks |
|---|---|---|
| IPv4 | `192.168.10.100` | Sumber seluruh aktivitas simulasi |
| Akun | `iotuser` | Akun yang ditargetkan pada SSH |
| User-Agent | `curl/7.81.0` | Klien pada serangan API |
| Pola HTTP | `PATCH /api/sensor` berulang | Upaya manipulasi data |
| Pola auth | ≥5 `Failed password` dalam <60 detik | Pola brute force |
| Port sumber acak | `43130`, `37804` | Port ephemeral klien SSH |
| Alert NIDS | `ET POLICY curl User-Agent`, `GPL WEB_SERVER 403 Forbidden`, `GPL ICMP_INFO PING *NIX` | Terlihat di Sguil |

> IOC di atas **spesifik untuk lingkungan lab**. Pada produksi, `192.168.10.100` adalah workstation penguji resmi, bukan penyerang eksternal.

### 4.4 Analisis Teknik (Pemetaan MITRE ATT&CK)

| Tahap | Teknik | ID ATT&CK | Hasil |
|---|---|---|---|
| Reconnaissance | Active Scanning: Vulnerability Scanning | T1595.002 | Port `22`/`80` teridentifikasi, `9000` filtered |
| Discovery | Network Service Discovery | T1046 | Versi Nginx tidak terungkap |
| Credential Access | Brute Force: Password Guessing | T1110.001 | Gagal; diblokir Fail2ban |
| Initial Access | Exploit Public-Facing Application | T1190 | Gagal; `PATCH` ditolak `403` |
| Impact (percobaan) | Stored Data Manipulation | T1565.001 | Tidak terjadi |

### 4.5 Dampak

| Aset | Dampak aktual | Dampak potensial bila kontrol gagal |
|---|---|---|
| Kredensial SSH | Tidak ada | Penguasaan akun `iotuser` |
| Data sensor | Tidak ada perubahan | Manipulasi nilai kelembapan/nutrisi pada dashboard |
| Ketersediaan layanan | Tidak ada gangguan | Potensi DoS bila brute force tidak dibatasi |
| Reputasi/data mitra | Tidak ada (data dummy) | Kebocoran data operasional pada sistem nyata |

### 4.6 Root Cause

1. **Permukaan serangan yang memang terbuka:** port `22` dan `80` diperlukan, sehingga wajar menjadi target reconnaissance.
2. **Ketiadaan rate limit pada lapisan aplikasi sebelum hardening:** sebelum B6 diterapkan, `PATCH` dapat mencapai backend. Saat insiden, kontrol sudah aktif sehingga serangan gagal.
3. **Visibilitas terbatas:** log hanya lokal, sehingga analisis bergantung pada sesi konsol dan screenshot, bukan kueri terpusat.

**Kesimpulan root cause:** bukan kegagalan kontrol, melainkan **keterbatasan visibilitas dan kesiapan forensik** yang membuat penanganan insiden bergantung pada bukti manual.

---

## 5. Fase 3 — Containment, Eradication, and Recovery

### 5.1 Containment

| Tindakan | Waktu | Bukti |
|---|---|---|
| Fail2ban otomatis memblokir `192.168.10.100` via iptables | 12:30:02 UTC | E-02 |
| Nginx menolak seluruh metode non-GET pada `/api/` | 12:39:08 UTC | E-03 |
| Tidak ada perubahan konfigurasi darurat yang diperlukan | — | — |

### 5.2 Eradication

| Tindakan | Keterangan |
|---|---|
| Pencabutan key sementara Red Team | Baris `cyberops-redteam` dihapus dari `/home/iotuser/.ssh/authorized_keys` |
| Verifikasi baseline kunci | `authorized_keys` kembali berisi 2 key sah (`thinkpad@GIEE`, `windows-pbl-key`) |
| Pemeriksaan persistensi | Tidak ditemukan indikator persistensi pada simulasi ini (belum diverifikasi dengan tooling forensik) |

### 5.3 Recovery

| Tindakan | Hasil |
|---|---|
| Verifikasi ulang SSH tanpa key | `Permission denied (publickey)` — baseline pulih |
| Verifikasi ulang API | `GET 401`, `PATCH 403` konsisten |
| Verifikasi layanan | Dashboard dan API tetap melayani permintaan sah |
| Unban otomatis | 12:32:02 UTC — IP penguji dapat kembali diuji sesuai prosedur |

### 5.4 Status Akhir

| Kriteria | Status |
|---|---|
| Akses tidak sah | Tidak terjadi |
| Perubahan data | Tidak terjadi |
| Layanan pulih | Ya (tidak pernah terganggu) |
| Bukti terkumpul | Sebagian (screenshot + hash) |
| Investigasi forensik lengkap | **Belum** — menunggu sesi akuisisi bukti |

---

## 6. Fase 4 — Post-Incident Activity

### 6.1 Lessons Learned

1. **Autentikasi berlapis bekerja sesuai desain.** `AuthenticationMethods publickey,password` membuat brute force password murni tidak relevan; ini juga menjelaskan mengapa pengujian harus dibagi dua tahap agar Fail2ban dapat dievaluasi.
2. **Fail2ban efektif, tetapi durasi blokir lab terlalu singkat** untuk skenario nyata.
3. **`limit_except GET` adalah kontrol yang sangat murah dan efektif** untuk API read-only.
4. **Deteksi default NIDS tidak cukup spesifik.** Alert `curl User-Agent` muncul, tetapi tidak secara eksplisit menandai brute force atau PATCH.
5. **Bukti screenshot tidak memadai untuk forensik formal.** Diperlukan salinan log ter-hash dan image bila insiden nyata.

### 6.2 Rekomendasi

| Prioritas | Rekomendasi | Target |
|---|---|---|
| P1 | Akuisisi salinan `auth.log`, `fail2ban.log`, `nginx/access.log` + `sha256sum` ke folder bukti | Minggu 12 |
| P1 | Kirim log ke Security Onion dan aktifkan retensi terpusat | Minggu 12–13 |
| P1 | Terapkan dan uji rule kustom Suricata untuk brute force dan PATCH | Minggu 12 |
| P2 | Naikkan `bantime`, aktifkan `bantime.increment` dan jail `recidive` | Minggu 13 |
| P2 | Susun runbook IR (kontak, eskalasi, kriteria severity, template laporan) | Minggu 13 |
| P3 | Tambahkan TLS pada replika dan uji HSTS/cookie `Secure` | Minggu 13 |

### 6.3 Rencana Tindak Lanjut Forensik (Minggu 12)

Pekerjaan berikut **belum dapat diselesaikan** karena VM dalam keadaan *saved/poweroff* saat laporan ini disusun:

1. Hidupkan `SRV-IOT-KEL10`, lakukan *live triage*: `date -Is`, `uptime`, `who`, `last -n 20`.
2. Hitung hash log: `sha256sum /var/log/auth.log /var/log/fail2ban.log /var/log/nginx/access.log`.
3. Salin log ke direktori bukti dengan nama bertimestamp dan catat hash-nya.
4. Periksa persistensi: `crontab -l`, `systemctl list-unit-files --state=enabled`, `ss -tulpn`.
5. Ambil screenshot tiap langkah dan simpan ke `docs/incident-response/evidence/`.
6. Perbarui [manifest bukti](evidence-manifest.md) dengan hash salinan log.

---

## 7. Lampiran

### 7.1 Bukti

| ID | Deskripsi | Berkas | SHA-256 (ringkas) |
|---|---|---|---|
| E-01 | `auth.log` — 5 kegagalan password | [rt02-log-auth.png](../redteam/evidence/RT-02/rt02-log-auth.png) | `8F350F98…DF798AF2` |
| E-02 | `fail2ban.log` — Found/Ban/Unban | [rt02-log-fail2ban.png](../redteam/evidence/RT-02/rt02-log-fail2ban.png) | `72B0D574…D542E620` |
| E-03 | Nginx `access.log` — 401/403 | [rt03-log-nginx-403.png](../redteam/evidence/RT-03/rt03-log-nginx-403.png) | `31DBDD60…10DF14BD` |
| E-04 | Sguil — event NIDS | [so-sguil-alerts.png](../redteam/evidence/so-sguil-alerts.png) | `54E9CE12…4724A9CB` |
| E-05 | Nmap — port scan | [rt01-nmap.png](../redteam/evidence/RT-01/rt01-nmap.png) | `A0B42D94…221D8546` |
| E-06 | curl — 401/403 | [rt03-curl-401-403.png](../redteam/evidence/RT-03/rt03-curl-401-403.png) | `A1860DB0…2B99270E` |
| E-07 | SSH tanpa key ditolak | [rt02-ssh-tanpa-key.png](../redteam/evidence/RT-02/rt02-ssh-tanpa-key.png) | `A30D043A…372DC868` |
| E-08 | Header HTTP | [rt01-header.png](../redteam/evidence/RT-01/rt01-header.png) | `F66DB821…EDC55BCC` |

Hash lengkap: [evidence-manifest.md](evidence-manifest.md).

### 7.2 Referensi

- NIST SP 800-61 Rev. 2 — *Computer Security Incident Handling Guide*.
- NIST SP 800-86 — *Guide to Integrating Forensic Techniques into Incident Response*.
- NIST SP 800-115 — *Technical Guide to Information Security Testing and Assessment*.
- MITRE ATT&CK — Enterprise Matrix.
- [Purple Team Log](../purple-team-log.md) dan [VAR Fase 2](../reports/vulnerability-assessment-report.md).
