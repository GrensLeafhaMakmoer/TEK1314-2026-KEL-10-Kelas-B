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

## 3. Chain of Custody

| Tanggal/Waktu | Aktivitas | Pelaksana | Catatan |
|---|---|---|---|
| 2026-11-07 12:29–13:10 UTC | Peristiwa berlangsung (simulasi) | Red Team | Sumber `192.168.10.100` |
| 2026-11-07 19:29–20:20 WIB | Pengambilan tangkapan layar | Blue Team | Via konsol VirtualBox |
| 2026-11-07 20:22 WIB | Commit bukti ke repository | Blue Team | Commit `fa4520d` |
| 2026-11-13 | Penghitungan hash dan manifest | Blue Team | Dokumen ini |

## 4. Keterbatasan Bukti

| Keterbatasan | Dampak | Rencana |
|---|---|---|
| Bukti berupa screenshot, bukan salinan log mentah | Tidak dapat di-query ulang secara forensik | Akuisisi salinan log + `sha256sum` pada sesi Minggu 12 |
| Tidak ada image disk/memori | Analisis artefak dan persistensi terbatas | Opsional; hanya jika diwajibkan |
| Log tidak tersentralisasi | Risiko kehilangan bukti bila host dikompromikan | Kirim ke Security Onion |
| Jam host UTC vs penguji WIB | Risiko salah korelasi | Konversi eksplisit pada timeline |

> Hash di atas membuktikan **integritas berkas bukti sejak di-commit**, bukan membuktikan integritas log asli pada host. Klaim tersebut hanya dapat dibuat setelah salinan log di-hash pada sumbernya.
