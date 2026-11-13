# Manifest Bukti — Insiden `INC-2026-11-001`

**Kelompok 10 | TEK1314 — Keamanan Siber | Fase 3**

Dokumen ini mencatat bukti yang digunakan pada [Draf Incident Report](incident-report-draft.md) beserta nilai hash SHA-256 untuk integritas. Hash dihitung pada 13 November 2026 dari salinan berkas di repository ini.

## 1. Prosedur Verifikasi Integritas

```powershell
Get-FileHash -Algorithm SHA256 -LiteralPath "<berkas>"
```

Perintah pembanding pada Linux:

```bash
sha256sum -c evidence.sha256
```

Berkas `evidence.sha256` (format `sha256sum`) tersedia di folder ini dan dapat diverifikasi ulang kapan saja.

## 2. Tabel Bukti

| ID | Berkas | Ukuran (byte) | SHA-256 | Sumber |
|---|---|---|---|---|
| E-01 | `docs/redteam/evidence/RT-02/rt02-log-auth.png` | 14.587 | `8F350F987D2800A10045F592BF774390028D99C0830DA8E928876000DF798AF2` | Konsol `SRV-IOT-KEL10`, `/var/log/auth.log` |
| E-02 | `docs/redteam/evidence/RT-02/rt02-log-fail2ban.png` | 11.919 | `72B0D57475D564957131DD00B99719B58A10BB15F1AD3AF06798A287D542E620` | Konsol `SRV-IOT-KEL10`, `/var/log/fail2ban.log` |
| E-03 | `docs/redteam/evidence/RT-03/rt03-log-nginx-403.png` | 10.152 | `31DBDD60C0D70B36A9ACB723851418C45414E131E1D3D96DD556656310DF14BD` | Konsol `SRV-IOT-KEL10`, `/var/log/nginx/access.log` |
| E-04 | `docs/redteam/evidence/so-sguil-alerts.png` | 90.228 | `54E9CE12C0642F3C4B6A2E03B70AE8723C082F11F838A0440C4F5F334724A9CB` | Konsol `SO-KEL10`, Sguil |
| E-05 | `docs/redteam/evidence/RT-01/rt01-nmap.png` | 635.233 | `A0B42D94F15647E8EA6A7EFC1B11D814BD5175D3523D0D940865615D221D8546` | Konsol `CYBEROPS-KEL10` |
| E-06 | `docs/redteam/evidence/RT-03/rt03-curl-401-403.png` | 651.118 | `A1860DB0855F74191C5D088604A95800A1359E31933D7776D936D3FB2B99270E` | Konsol `CYBEROPS-KEL10` |
| E-07 | `docs/redteam/evidence/RT-02/rt02-ssh-tanpa-key.png` | 665.291 | `A30D043A89C5426E8B80CC9054F62CCE75EA56CD3ABFDADA4B1B2361372DC868` | Konsol `CYBEROPS-KEL10` |
| E-08 | `docs/redteam/evidence/RT-01/rt01-header.png` | 676.320 | `F66DB821D96E669CA1BA16688376CAC625332017EF92B99491F5DFDFEDC55BCC` | Konsol `CYBEROPS-KEL10` |

## 3. Bukti Akuisisi Live (13 November 2026)

Akuisisi dilakukan melalui konsol `SRV-IOT-KEL10` pada 13 November 2026, 15:54–15:57 UTC (22:54–22:57 WIB).

| ID | Berkas | Ukuran (byte) | SHA-256 |
|---|---|---|---|
| E-09 | `docs/incident-response/evidence/triage-01-system-info.png` | 11.268 | `064D746B3480646721AA647FE7D13C1717862747C5A37BB08A013FC715AECB9F` |
| E-10 | `docs/incident-response/evidence/triage-02-persistence.png` | 14.180 | `C2017D3D17ECFDC1AAF940A609D5C781B13FCEA94FFEFA6D10BDC42F0D4BD9F3` |
| E-11 | `docs/incident-response/evidence/triage-03-listeners.png` | 12.276 | `B65B55783F625944DDEBC219A02EA249E0A3994BE5481BB21B10F73D63794F64` |
| E-12 | `docs/incident-response/evidence/triage-04-log-summary.png` | 7.833 | `E1AD0FA3D910F0273BD789CC2A1DB1C8E0E9810B00C27CD1E7A6837129A82088` |
| E-13 | `docs/incident-response/evidence/triage-05-hash-verify.png` | 11.035 | `45B66DF9B417FDFE0E2E440AAAAE595B8EC3C761D0A3D625C4A311E8A64B9D07` |

### 3.1 Salinan Log di Host Target

Salinan log disimpan **di dalam VM**, bukan di repository, untuk menghindari pemalsuan bukti di luar host:

| Berkas | Ukuran | mtime (host, UTC) |
|---|---|---|
| `/home/iotuser/forensics/INC-2026-11-001/auth.log` | 5.202 B | 2026-11-07 13:26 |
| `/home/iotuser/forensics/INC-2026-11-001/fail2ban.log` | 2.160 B | 2026-11-07 12:32 |
| `/home/iotuser/forensics/INC-2026-11-001/access.log` | 1.428 B | 2026-11-07 13:11 |
| `/home/iotuser/forensics/INC-2026-11-001/SHA256SUMS.txt` | 231 B | 2026-11-13 15:55 |

**Verifikasi:** `sha256sum -c SHA256SUMS.txt` → `access.log: OK`, `auth.log: OK`, `fail2ban.log: OK` (lihat E-13).

> Nilai hash lengkap salinan log **tidak ditulis ulang secara manual** di dokumen ini. Sumber otoritatifnya adalah `SHA256SUMS.txt` di dalam VM dan tangkapan layar E-13. Transkripsi manual dari konsol berisiko menimbulkan galat; percobaan transkripsi memang pernah gagal verifikasi dan karena itu tidak dipakai.

## 4. Temuan Triage

| Temuan | Detail | Implikasi |
|---|---|---|
| Tidak ada jadwal cron pengguna | `crontab -l` → `no crontab for iotuser` | Tidak ada persistensi berbasis cron pengguna |
| Unit enabled normal | Daftar unit didominasi layanan standar Ubuntu (snap, apparmor, cron, fail2ban, getty) | Tidak ada unit mencurigakan pada 25 entri pertama |
| `fail2ban` dan `nginx` aktif | `systemctl is-active` → `active` | Kontrol deteksi dan layanan web berjalan |
| **Listener database tidak hanya loopback** | `ss -tulpn` → `[::]:9000` (java, pid 1787) | Emulator Firebase mendengarkan seluruh interface IPv6; isolasi sepenuhnya bergantung pada UFW, bukan pada binding loopback. Klaim B2 perlu dikoreksi. |
| Port web dan SSH terbuka | `0.0.0.0:80` (node), `0.0.0.0:22` (sshd) | Sesuai desain; dibatasi UFW per sumber |
| Jumlah kegagalan SSH | `grep -c "Failed password"` → **4 baris**, tetapi satu baris menyatakan `message repeated 2 times` | Total percobaan = **5**, sesuai ambang `maxretry=5` Fail2ban |
| Log tidak berubah sejak insiden | Baris terakhir `fail2ban.log`: `Unban` 2026-11-07 12:32:02 | Tidak ada aktivitas autentikasi baru setelah insiden |

## 5. Chain of Custody

| Tanggal/Waktu | Aktivitas | Pelaksana | Catatan |
|---|---|---|---|
| 2026-11-07 12:29–13:10 UTC | Peristiwa berlangsung (simulasi) | Red Team | Sumber `192.168.10.100` |
| 2026-11-07 19:29–20:20 WIB | Pengambilan tangkapan layar | Blue Team | Via konsol VirtualBox |
| 2026-11-07 20:22 WIB | Commit bukti ke repository | Blue Team | Commit `fa4520d` |
| 2026-11-13 15:54 UTC | Triage sistem dan akuisisi log | Blue Team | Konsol `SRV-IOT-KEL10` |
| 2026-11-13 15:55 UTC | Pembuatan `SHA256SUMS.txt` dan verifikasi | Blue Team | Ketiga berkas `OK` |
| 2026-11-13 | Penghitungan hash screenshot triage | Blue Team | Dokumen ini |

## 6. Keterbatasan Bukti

| Keterbatasan | Dampak | Rencana |
|---|---|---|
| Salinan log hanya berada di dalam VM | Belum dapat di-query dari repository | Ekspor terkontrol pada sesi berikutnya atau kirim ke Security Onion |
| Tidak ada image disk/memori | Analisis artefak mendalam terbatas | Opsional; hanya jika diwajibkan |
| Log tidak tersentralisasi | Risiko kehilangan bukti bila host dikompromikan | Kirim ke Security Onion |
| Jam host UTC vs penguji WIB | Risiko salah korelasi | Konversi eksplisit pada timeline |
| Transkripsi hash manual tidak dipakai | Nilai hash hanya terbaca dari screenshot | Verifikasi langsung di VM dengan `sha256sum -c` |

> Hash pada tabel E-09…E-13 membuktikan **integritas berkas bukti sejak di-commit**. Integritas salinan log dibuktikan oleh `SHA256SUMS.txt` **di dalam VM**, bukan oleh repository ini.
