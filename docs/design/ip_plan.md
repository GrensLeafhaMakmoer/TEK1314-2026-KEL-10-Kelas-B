# IP Plan — Kelompok 10

**Proyek IoT:** sistem autodosing nutrisi AB Mix pada tanaman melon.

Dokumen ini menjelaskan pengalamatan dan akses layanan pada replika backend setelah hardening. Seluruh pengujian menggunakan data dummy dan tidak melibatkan sistem produksi atau perangkat IoT fisik.

## 1. Segmen Jaringan

| Parameter | Nilai |
|---|---|
| Network | `192.168.10.0/24` |
| Subnet mask | `255.255.255.0` |
| Broadcast | `192.168.10.255` |
| Alokasi alamat lab | Statis |
| Jaringan pengujian | VirtualBox Host-Only |
| Gateway dan DNS lab | Tidak diperlukan untuk komunikasi antarhost pada subnet yang sama |

Adapter NAT digunakan sementara untuk instalasi dan pembaruan paket. Isolasi pengujian dilakukan dengan menonaktifkan adapter NAT setelah kebutuhan unduhan selesai; aturan UFW saja tidak menjamin isolasi internet.

## 2. Alokasi Alamat IP

| Perangkat | Peran | Alamat IP | Sistem Operasi / Keterangan |
|---|---|---|---|
| CYBEROPS-KEL10 | Attacker Node | `192.168.10.100/24` | CyberOps Workstation |
| SRV-IOT-KEL10 | Target Node | `192.168.10.5/24` | Ubuntu Server; hostname sistem `srv-iot-kel10` |
| S0-KEL10 | Monitoring Node | `192.168.10.200/24` | Security Onion; alamat antarmuka manajemen |
| Host laptop | Administrasi dan akses dashboard | `192.168.10.1/24` | Alamat Host-Only yang diizinkan dalam aturan UFW Target |
| Loopback Target | Komunikasi lokal aplikasi–database | `127.0.0.1` | Hanya dapat diakses dari Ubuntu Target |

## 3. Layanan dan Kebijakan Akses Target

| Port/Protokol | Layanan | Akses setelah Hardening | Fokus Pengujian |
|---|---|---|---|
| `22/TCP` | SSH | UFW mengizinkan sumber `192.168.10.100` dan `192.168.10.1`; login root langsung ditolak dan Fail2ban aktif | Pembatasan autentikasi berulang dan pemulihan akses |
| `80/TCP` | Nginx — dashboard dan proxy API | UFW mengizinkan sumber `192.168.10.100` dan `192.168.10.1`; dashboard pada `/dashboard.html`, API pada `/api/sensor` | Informasi versi pada respons dan penolakan metode tulis pada endpoint baca |
| `9000/TCP` | Firebase Realtime Database Emulator | Listener hanya `127.0.0.1:9000`; tidak ada izin UFW untuk akses langsung dari CyberOps | Pembatasan paparan database serta validasi data melalui pengujian lokal |
| `4400/TCP`, `4500/TCP` | Komponen pendukung Firebase Emulator | Listener loopback pada pemeriksaan sebelumnya; tidak dibuka melalui UFW | Memastikan layanan pendukung tidak terekspos ke jaringan |

UFW menerapkan **deny incoming** dan **allow outgoing**. Izin SSH/HTTP umum untuk IPv4 dan IPv6 telah dihapus dan diganti aturan sumber IPv4 yang ditentukan. Sebelum pembatasan, scan CyberOps menunjukkan port `22`, `80`, dan `9000` open; setelah pembatasan UFW, port `22` dan `80` tetap open, sedangkan `9000` menjadi filtered. Database selanjutnya dibatasi ke loopback sebagai lapisan perlindungan tambahan.

Port `4000` (Emulator UI), `5000` (Hosting Emulator), `8080` (Firestore Emulator), dan `9099` (Authentication Emulator) tidak digunakan dalam implementasi ini dan tidak perlu diizinkan pada firewall. Dashboard menggunakan Nginx, bukan Hosting Emulator.

## 4. Alur Komunikasi Aplikasi

```text
CyberOps / Host laptop yang diizinkan
                 |
                 | HTTP :80
                 v
       Nginx pada Ubuntu Target
       /dashboard.html dan /api/sensor
                 |
                 | HTTP REST lokal
                 v
       Firebase Emulator 127.0.0.1:9000
                 ^
                 | PUT data dummy setiap 5 detik
                 |
       Simulator Python pada Ubuntu Target
```

Namespace database adalah `demo-melon-lab`, dengan data sensor pada `/lab_dummy/sensor`. Browser membaca data melalui proxy Nginx, bukan melalui port database secara langsung. Endpoint `/api/sensor` mengizinkan GET/HEAD dan menolak metode tulis. Komunikasi replika masih menggunakan **HTTP REST/JSON**, belum HTTPS.

## 5. Monitoring Trafik

Trafik pengujian berasal dari CyberOps `192.168.10.100` menuju Target `192.168.10.5`. Security Onion menggunakan antarmuka sensor `eth0` tanpa alamat IPv4 untuk mengamati trafik, sedangkan `eth1` dengan IP `192.168.10.200` digunakan untuk manajemen. Adapter sensor Host-Only menggunakan promiscuous mode **Allow All**; visibilitas trafik perlu diverifikasi sebelum evaluasi deteksi.

Pemantauan tidak memerlukan izin akses ke layanan Target dari IP manajemen Security Onion. Trafik loopback simulator–database tidak melewati jaringan Host-Only, sehingga diverifikasi melalui log atau pengujian lokal di Ubuntu. Kegagalan password SSH diperiksa pada log autentikasi Target karena isi sesi SSH terenkripsi. Keberadaan trafik pada sensor tidak otomatis menunjukkan alert telah terbentuk.

## 6. Dokumen Terkait

- [Topologi jaringan](topology.png)
- [Deklarasi tech stack](../presentation-minggu9/tech_stack_declaration.md)
- [Checklist hardening dan bukti implementasi](../blueteam/hardening_checklist.md)
- [Rencana pengujian Red Team](../redteam/attack_plan.md)
