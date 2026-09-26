# IP Plan - Kelompok 10 (IoT Protocol Guardian)

**Network:** 192.168.10.0/24 | Netmask: 255.255.255.0 | Mode: Host-Only (isolated)

| Hostname | Peran | IP Address | OS |
|---|---|---|---|
| SRV-IOT-KEL10 | Target Server (Korban) | 192.168.10.5 | Ubuntu Server 22.04 LTS CLI + Mosquitto MQTT |
| KALI-KEL10 | Attacker Node | 192.168.10.100 | Kali Linux |
| SO-KEL10 | Monitoring Node | 192.168.10.200 | Security Onion |

Keterangan:
- Segmen 192.168.10.0/24 unik milik Kelompok 10.
- Semua VM memakai IP statis dan jaringan Host-Only agar isolated sesuai RoE.
- Serangan hanya ditujukan ke 192.168.10.5.
- Security Onion memakai NIC tambahan mode promiscuous untuk memantau trafik MQTT.

## Daftar Port & Potensi Celah (Red Team)

| Port | Service | Status di Target | Potensi Eksploitasi |
|---|---|---|---|
| 1883/TCP | MQTT (Mosquitto) | OPEN | Anonymous publish/subscribe, replay attack perintah ON/OFF tanpa izin, eavesdropping karena plain-text |
| 8883/TCP | MQTT over TLS | CLOSED (rencana hardening) | Dibuka setelah hardening dengan username/password + TLS untuk menutup celah 1883 |
| 22/TCP | SSH | OPEN (filtered) | Brute-force password, hanya diizinkan dari 192.168.10.100, dimonitor Security Onion |
| 80/TCP | HTTP | CLOSED | Tidak dipakai di skenario IoT, wajib tetap tertutup. Jika terbuka (misal terinstall web dashboard) berpotensi directory traversal & info disclosure, harus difirewall |

