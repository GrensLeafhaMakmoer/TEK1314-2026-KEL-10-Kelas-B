**LOGBOOK - Kelompok 12 Kelas B**  
**Anggota**  
1. Ghazali Habibie A (J0404241145) - Lead  
2. Gregorius Gil Ovidio Sebayang (J0404241088) - Red Team  
3. Binggie Rashel Prasetyo (J0404241009) - Blue Team  
**Minggu 2**  
- Bentuk kelompok dan bagi peran.  
- Install VirtualBox, download ISO Ubuntu, Kali, Security Onion.  
- Buat repo GitHub.  
**Minggu 3**  
- Latihan dasar Linux.  
- Red Team cek port yang perlu dibuka: **1883 (MQTT), 80 (HTTP), 21 (FTP), 22 (SSH)** — status masih "perlu diverifikasi", menunggu hasil scanning (nmap) di target.  
**Minggu 4**  
- Blue Team gambar topologi 3 node (attacker, target, monitoring).  
- Tentukan subnet **192.168.12.0/24**, mode Host-Only (isolated):  
  - Target: **192.168.12.5** — TARGET-KEL12 — Arch Linux Security Workstation  
  - Attacker: **192.168.12.100** — KALI-KEL12 — Kali Linux  
  - Monitoring: **192.168.12.200** — SO-KEL12 — Security Onion  
- Lead upload docs/design/topology.jpeg dan docs/design/ip_plan.md.  
- OS target: Arch Linux Security Workstation (VM yang dikasih lab).  
