# Deklarasi Tech Stack — Kelompok 10

**Proyek IoT:** Sistem autodosing nutrisi AB Mix pada tanaman melon.

| Komponen | Teknologi pada Proyek IoT Asli | Replika pada Lab |
| --- | --- | --- |
| **Sistem operasi backend** | Infrastruktur Firebase cloud terkelola Google; OS server tidak dipilih atau dikelola kelompok | **Ubuntu Server** |
| **Database/penyimpanan data** | **Firebase**, menggunakan **Realtime Database** | **Firebase Realtime Database Emulator** dengan data sensor, setpoint, dan riwayat dosing dummy |
| **Komunikasi perangkat–backend** | **HTTPS REST API** dengan format data **JSON** | Script simulator Python mengirim data dummy melalui **HTTP REST API** ke emulator lokal |
| **Dashboard/monitoring** | **Aplikasi mobile Flutter**, terhubung ke Firebase | **Dashboard web sederhana HTML/JavaScript** yang terhubung ke emulator lokal |
