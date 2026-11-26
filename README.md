<div align="center">

# MelonLab — Backend IoT Security

**PBL Keamanan Siber · TEK1314 · Kelompok 10 · Kelas B · 2026**

Pengamanan replika backend sistem autodosing nutrisi AB Mix pada tanaman melon melalui hardening, Purple Team Loop, forensik digital, dan remediasi berbasis NIST SP 800-61r2.

[Ringkasan Akhir](#ringkasan-akhir-proyek-fase-13) · [Artefak Fase 3](#fase-3--incident-response-remediasi--final-minggu-1215) · [Logbook](LOGBOOK.md) · [Hardening](docs/blueteam/hardening_checklist.md) · [Purple Team Log](docs/purple-team-log.md)

</div>

---

## Tentang Proyek

Repository ini memuat dokumentasi lengkap proyek **Simulated SOC Engagement** untuk mempelajari pengamanan backend IoT sepanjang satu siklus penuh (Fase 1–3): perancangan arsitektur & hardening, simulasi serangan & deteksi (*Purple Team Loop*), investigasi forensik & *incident response* mengacu **NIST SP 800-61r2**, serta remediasi celah keamanan.

Target pengujian merupakan **replika laboratorium**, bukan sistem IoT nyata milik mitra. Backend dibangun menggunakan Ubuntu Server, Firebase Realtime Database Emulator, simulator Python, layanan autentikasi `pbl-web-auth`, serta dashboard HTML/JavaScript yang dilayani Nginx.

> **Batas pengujian:** seluruh aktivitas hanya dilakukan pada lingkungan lab yang diizinkan dengan data dummy (`demo-melon-lab`). Tidak ada pengujian terhadap sistem produksi, sensor fisik, atau pengendalian pompa.

## Tim dan Pembagian Peran

| Anggota | NIM | Peran | Fokus Tanggung Jawab |
|---|---|---|---|
| **Ghazali Habibie A** | J0404241145 | **Leader** | Koordinasi tim, konsistensi dokumentasi & Incident Report final, persiapan presentasi, dan kelengkapan repository. |
| **Binggie Rashel Prasetyo** | J0404241009 | **Blue Team** | Hardening server, monitoring Security Onion, analisis log & forensik, remediasi, serta penyusunan bagian pertahanan pada Incident Report. |
| **Gregorius Gil Ovidio Sebayang (Greg)** | J0404241088 | **Red Team** | Identifikasi layanan, perencanaan & eksekusi skenario serangan terkontrol, uji verifikasi ulang, serta ringkasan skenario serangan. |

## Ringkasan Akhir Proyek (Fase 1–3)

| Fase | Fokus Pekerjaan | Hasil Utama |
|---|---|---|
| **Fase 1** *(Minggu 1–9)* | Desain topologi, replika backend IoT, baseline trafik, dan hardening | **14 kontrol hardening** selesai (`4` Kategori A, `6` Kategori B, `4` Kategori C — `0 Partial`, `0 Not Yet`). |
| **Fase 2** *(Minggu 9–11)* | Simulasi serangan 3 skenario (*Purple Team Loop*) & penilaian risiko | Seluruh 5 tahap loop terverifikasi untuk `RT-01` (Reconnaissance), `RT-02` (SSH Auth), dan `RT-03` (Web/API). **Tidak ada** akses tidak sah maupun perubahan data. |
| **Fase 3** *(Minggu 12–15)* | Forensik digital (`INC-2026-11-001`), remediasi IoT, dan penutupan siklus | Seluruh **5 item remediasi P1/P2** Minggu 13 selesai: 2 rule kustom Snort (`sid:1000010`, `sid:1000011`), sentralisasi log (`rsyslog` → `syslog-ng`), hardening Fail2ban (`bantime=3600`, increment, `recidive`), verifikasi loopback DB (`127.0.0.1:9000`), serta aktifasi **TLS/HTTPS + HSTS + cookie `Secure`** pada domain lab `melonlab.local`. |

## Lingkungan Laboratorium

| Node | Hostname | Platform | Alamat IP | Fungsi |
|---|---|---|---|---|
| Attacker | `CYBEROPS-KEL10` | CyberOps Workstation | `192.168.10.100` | Client pengujian keamanan dan pembuat trafik lab. |
| Target | `SRV-IOT-KEL10` | Ubuntu Server (`melonlab.local`) | `192.168.10.5` | Backend replika, dashboard HTTPS, layanan autentikasi, dan API. |
| Monitoring | `SO-KEL10` | Security Onion | `192.168.10.200` | NIDS (Snort 2 + Sguil) dan penerima syslog terpusat (`514/tcp`). |

Jaringan pengujian menggunakan **VirtualBox Host-Only `192.168.10.0/24`**. Kebijakan akses serta detail interface dijelaskan dalam [IP plan](docs/design/ip_plan.md).

### Teknologi Replika

| Komponen | Teknologi |
|---|---|
| Sistem operasi | Ubuntu Server |
| Database | Firebase Realtime Database Emulator (terikat ke loopback `127.0.0.1:9000`) |
| Komunikasi | HTTPS REST API (JSON) via Nginx (`TLSv1.2`/`TLSv1.3`, HSTS, redirect `308` dari HTTP `80` ke `https://melonlab.local`) |
| Dashboard & Auth | HTML/JavaScript melalui Nginx + `pbl-web-auth` (cookie `HttpOnly`, `SameSite=Lax`, `Secure`) |
| Monitoring & Logging | Security Onion (`Snort 2` + Sguil), `rsyslog` → `syslog-ng` terpusat (`/var/log/remote/srv-iot-kel10.log`), dan Fail2ban (`sshd`, `recidive`) |

**Perbedaan dari sistem asli:** Firebase cloud dan aplikasi Flutter disederhanakan menjadi emulator lokal dan dashboard web. Enkripsi komunikasi telah ditingkatkan dari HTTP (Fase 1–2) menjadi **HTTPS dengan CA lokal (`melonlab.local`)** pada remediasi Minggu 13. Pemetaan awal tersedia pada [deklarasi tech stack](docs/presentation-minggu9/tech_stack_declaration.md).

---

## Dokumentasi & Artefak (Fase 1–3)

### Fase 1 — Arsitektur, Baseline, & Hardening (Minggu 1–9)

| Dokumen | Isi |
|---|---|
| [Logbook](LOGBOOK.md) | Progres kelompok, pembagian tugas, kendala, dan tindak lanjut sepanjang semester. |
| [Topologi jaringan](docs/design/topology.png) | Diagram arsitektur lingkungan simulasi. |
| [IP plan](docs/design/ip_plan.md) | Pengalamatan, layanan, dan kebijakan akses. |
| [Deklarasi tech stack](docs/presentation-minggu9/tech_stack_declaration.md) | Pemetaan teknologi proyek asli ke replika. |
| [Checklist hardening](docs/blueteam/hardening_checklist.md) | 14 kontrol keamanan (Kategori A/B/C), status penerapan, dan bukti verifikasi. |
| [Baseline trafik](docs/blueteam/baseline_traffic.md) | Hasil pengamatan aktivitas normal pada lab. |
| [Slide checkpoint minggu 9](docs/presentation-minggu9/slides.pdf) | Berkas PDF presentasi checkpoint Fase 1. |

### Fase 2 — Simulasi Serangan & Penilaian Risiko (Minggu 9–11)

| Dokumen | Isi |
|---|---|
| [Attack plan](docs/redteam/attack_plan.md) | Rencana 3 skenario pengujian (`RT-01`, `RT-02`, `RT-03`), tools, target, dan batas pelaksanaan. |
| [Log Purple Team](docs/purple-team-log.md) | Pencatatan 5 tahap Purple Team Loop (serang, deteksi, analisis, mitigasi, verifikasi ulang). |
| [Vulnerability Assessment Report](docs/reports/vulnerability-assessment-report.md) | Penilaian risiko Fase 2, temuan, matriks risiko, dan rekomendasi. |

### Fase 3 — Incident Response, Remediasi, & Final (Minggu 12–15)

| Dokumen / Artefak | Isi | Status |
|---|---|---|
| [Draf Incident Report](docs/incident-response/incident-report-draft.md) | Draf penanganan insiden `INC-2026-11-001` sesuai NIST SP 800-61r2 (Minggu 12). | Selesai |
| [Manifest Bukti Forensik](docs/incident-response/evidence-manifest.md) | Daftar bukti, hash SHA-256 (`evidence.sha256`), dan *chain of custody*. | Selesai |
| [Catatan Remediasi Minggu 13](docs/blueteam/remediation-minggu13.md) | Implementasi & verifikasi 5 item remediasi P1/P2 + bukti SHA-256. | Selesai |
| **Incident Report Final** — `docs/final/incident_report.pdf` | Laporan akhir lengkap 8 bagian sesuai NIST SP 800-61r2 (format PDF). | *Menunggu unggah sebelum Minggu 14* |
| **Video Demo Purple Team Loop** — `docs/final/video_demo.mp4` *(atau tautan YouTube/Drive unlisted)* | Rekaman 5–8 menit menampilkan 1 siklus penuh Purple Team Loop. | *Menunggu unggah / tautan sebelum Minggu 14* |
| **Slide Presentasi Final** — `docs/presentation-final/slides.pdf` | Slide presentasi akhir (maks. 10–12 slide, format PDF). | *Menunggu unggah sebelum Minggu 14* |

> **Tautan Video Demo (Unlisted):** *(tambahkan tautan YouTube/Google Drive di sini jika berkas `.mp4` tidak diunggah langsung ke `docs/final/video_demo.mp4`)*
