# IP Plan - Kelompok 10 

Proyek IoT: **sistem autodosing nutrisi AB Mix pada tanaman melon**.


## Topologi Jaringan

## 1. Segmen Jaringan

| Parameter | Nilai |
|---|---|
| Network | `192.168.10.0/24` |
| Subnet mask | `255.255.255.0` |
| Broadcast | `192.168.10.255` |
| Alokasi alamat | IP statis |
| Rencana jaringan VM | Host-Only/isolated|

Subnet `/24` dan mode Host-Only merupakan rencana implementasi VM.

## 2. Alokasi IP dan Hostname

| Hostname | Peran | IP Address | Sistem Operasi | 
|---|---|---|---|---|
| CYBEROPS-KEL10 | Attacker Node | `192.168.10.100/24` | CyberOps Workstation | 
| SRV-IOT-KEL10 | Target Node | `192.168.10.5/24` | Ubuntu Server | 
| S0-KEL10 | Monitoring Node | `192.168.10.200/24` | Security Onion | 

## 3. Monitoring Traffic

Traffic pengujian berasal dari `192.168.10.100` menuju Target `192.168.10.5`. Monitoring menggunakan IP `192.168.10.200` sesuai diagram.

## 4. Daftar Port, Layanan, dan Potensi Celah

| Port/Protokol | Layanan | Status pada Target | Potensi Celah yang Perlu Diuji |
|---|---|---|---|
| `22/TCP` | SSH | Belum diverifikasi | Password lemah/default, login root langsung, dan percobaan login berulang jika pembatasan akses tidak diterapkan. |
| `9000/TCP` | Realtime Database Emulator | Belum diverifikasi; jika memilih Realtime Database | Security Rules terlalu permisif memungkinkan pembacaan data sensor atau perubahan setpoint/perintah dosing tanpa izin; validasi yang kurang memungkinkan data tidak sesuai. |
| `8080/TCP` | Firestore Emulator | Belum diverifikasi; alternatif jika memilih Firestore | Kesalahan Security Rules atau pembatasan peran memungkinkan akses dokumen milik pengguna/perangkat lain dan perubahan konfigurasi dosing tanpa izin. |
| `9099/TCP` | Authentication Emulator | Belum diverifikasi; jika digunakan | Endpoint administrasi emulator yang terekspos dapat memungkinkan manipulasi akun uji. Token emulator tidak boleh dipercaya oleh sistem produksi. |
| `5000/TCP` | Dashboard melalui Hosting Emulator | Belum diverifikasi; jika digunakan | XSS jika input ditampilkan tanpa penanganan yang aman; perubahan setpoint/perintah dosing tanpa izin jika otorisasi hanya diterapkan pada UI. |
| `4000/TCP` | Emulator UI | Belum diverifikasi; direncanakan hanya pada loopback | Paparan data dan fungsi administrasi emulator jika UI dapat diakses pihak tidak berwenang. |
