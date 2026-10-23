<div align="center">

# MelonLab — Backend IoT Security

**PBL Keamanan Siber · TEK1314 · Kelompok 10 · Kelas B · 2026**

Pengamanan replika backend sistem autodosing nutrisi AB Mix pada tanaman melon.

[Logbook](LOGBOOK.md) · [Arsitektur](docs/design/ip_plan.md) · [Hardening](docs/blueteam/hardening_checklist.md) · [Attack Plan](docs/redteam/attack_plan.md)

</div>

---

## Tentang Proyek

Repository ini memuat dokumentasi proyek **Simulated SOC Engagement** untuk mempelajari pengamanan backend IoT melalui hardening, monitoring trafik, dan perencanaan pengujian keamanan.

Target pengujian merupakan **replika laboratorium**, bukan sistem IoT yang digunakan mitra. Backend dibangun menggunakan Ubuntu Server, Firebase Realtime Database Emulator, simulator Python, serta dashboard HTML/JavaScript yang dilayani Nginx.

> **Batas pengujian:** seluruh aktivitas hanya dilakukan pada lingkungan lab yang diizinkan dengan data dummy. Tidak ada pengujian terhadap sistem produksi, sensor fisik, atau pengendalian pompa.

## Tim dan Pembagian Peran

| Anggota | NIM | Peran | Fokus Tanggung Jawab |
|---|---|---|---|
| **Ghazali Habibie A** | J0404241145 | **Leader** | Koordinasi tim, pengelolaan dokumentasi, dan persiapan presentasi. |
| **Binggie Rashel Prasetyo** | J0404241009 | **Blue Team** | Hardening server, monitoring keamanan, analisis log, dan mitigasi. |
| **Gregorius Gil Ovidio Sebayang (Greg)** | J0404241088 | **Red Team** | Identifikasi layanan, perencanaan skenario, dan pengujian keamanan terkontrol. |

## Lingkungan Laboratorium

| Node | Platform | Alamat IP | Fungsi |
|---|---|---|---|
| Attacker | CyberOps Workstation | `192.168.10.100` | Client pengujian keamanan dan pembuat trafik lab. |
| Target | Ubuntu Server | `192.168.10.5` | Backend replika, dashboard, dan API. |
| Monitoring | Security Onion | `192.168.10.200` | Alamat manajemen; trafik diamati melalui interface sensor. |

Jaringan pengujian menggunakan **VirtualBox Host-Only `192.168.10.0/24`**. Kebijakan akses serta detail interface dijelaskan dalam [IP plan](docs/design/ip_plan.md).

### Teknologi Replika

| Komponen | Teknologi |
|---|---|
| Sistem operasi | Ubuntu Server |
| Database | Firebase Realtime Database Emulator |
| Komunikasi | HTTP REST API dengan format JSON |
| Dashboard | HTML/JavaScript melalui Nginx |

Simulator Python menyediakan data dummy. Security Onion, Sguil, dan tcpdump digunakan untuk pengamatan keamanan dan trafik.

**Perbedaan dari sistem asli:** Firebase cloud dan aplikasi Flutter disederhanakan menjadi emulator lokal dan dashboard web. Komunikasi replika masih menggunakan **HTTP, bukan HTTPS**. Pemetaan lengkap tersedia pada [deklarasi tech stack](docs/presentation-minggu9/tech_stack_declaration.md).

## Dokumentasi Utama

| Dokumen | Isi |
|---|---|
| [Logbook](LOGBOOK.md) | Progres kelompok, pembagian tugas, kendala, dan tindak lanjut. |
| [Topologi jaringan](docs/design/topology.png) | Diagram arsitektur lingkungan simulasi. |
| [IP plan](docs/design/ip_plan.md) | Pengalamatan, layanan, dan kebijakan akses. |
| [Deklarasi tech stack](docs/presentation-minggu9/tech_stack_declaration.md) | Pemetaan teknologi proyek asli ke replika. |
| [Checklist hardening](docs/blueteam/hardening_checklist.md) | Kontrol keamanan, status penerapan, bukti, dan rencana lanjutan. |
| [Baseline trafik](docs/blueteam/baseline_traffic.md) | Hasil pengamatan aktivitas normal pada lab. |
| [Attack plan](docs/redteam/attack_plan.md) | Rencana skenario pengujian, tools, target, dan batas pelaksanaan. |
| [Log Purple Team](docs/purple-team-log.md) | Tempat pencatatan pengujian, deteksi, analisis, respons, dan verifikasi ulang. |
| [Slide checkpoint minggu 9](docs/presentation-minggu9/slides.pdf) | Berkas PDF presentasi checkpoint. |

