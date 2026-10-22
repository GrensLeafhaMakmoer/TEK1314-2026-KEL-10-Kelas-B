# Baseline Trafik Normal — Backend IoT

**Kelompok 10 | TEK1314 — PBL Keamanan Siber | Checkpoint Minggu 9**

## 1. Tujuan dan Ruang Lingkup

Dokumen ini merangkum bukti trafik normal pada replika backend sistem monitoring nutrisi AB Mix tanaman melon. Aktivitas yang diamati meliputi ping, pembacaan API, dan permintaan HTTP dashboard. Pengamatan menggunakan Sguil dan tcpdump pada Security Onion, dengan data aplikasi dummy tanpa melibatkan sistem produksi atau perangkat IoT fisik.

Dokumentasi disusun berdasarkan tangkapan layar yang tersedia, bukan pengukuran kontinu terhadap seluruh trafik lab. Status hardening dibahas terpisah dalam [checklist hardening](hardening_checklist.md).

## 2. Lingkungan Pengamatan

| Komponen | Alamat / Fungsi |
|---|---|
| CyberOps Workstation | `192.168.10.100`, client pembuat trafik |
| Ubuntu Target | `192.168.10.5`, Nginx dan aplikasi replika |
| Security Onion | IP manajemen sesuai rancangan: `192.168.10.200`; sensor `seconion-eth0` |
| Sguil | Versi `0.9.0`, pengamatan event IDS |
| tcpdump | Pengamatan paket jaringan pada Security Onion |
| Endpoint aplikasi | HTTP port `80`: `/api/sensor` dan `/api/history` |


## 3. Hasil Pengamatan

| Aktivitas / Pemeriksaan | Hasil yang Terlihat | Bukti |
|---|---|---|
| Status layanan monitoring | `so-status` menunjukkan `[OK]` pada Sguil server, komponen sensor `seconion-eth0`, dan layanan lain yang ditampilkan. | B01 |
| Ping dari CyberOps ke Ubuntu Target | Balasan dari `192.168.10.5` terlihat pada terminal CyberOps hingga `icmp_seq=84`.  | B05 |
| Event ICMP di Sguil | Terlihat event `GPL ICMP_INFO PING *NIX` dengan sumber `192.168.10.100` dan tujuan `192.168.10.5`; daftar rincian juga memperlihatkan beberapa event ping. | B03, B04 |
| Pembacaan API sensor dengan curl | Permintaan `GET /api/sensor` memperoleh HTTP `200 OK` dan payload JSON sensor. Header server menampilkan `nginx` tanpa nomor versi. | B02 |
| Event curl di Sguil | Terlihat event berawalan `ET POLICY curl User-Agent ...` dari `192.168.10.100` ke `192.168.10.5`, port tujuan `80`. Korelasi langsung dengan screenshot curl B02. | B03 |
| Pembacaan sensor dan riwayat melalui HTTP | tcpdump memperlihatkan `GET /api/sensor`, `GET /api/history`, dan respons `HTTP/1.1 200 OK` antara CyberOps dan Ubuntu Target. | B06 |
| Ping Ubuntu ke IP sendiri | Terminal `iotuser@srv-iot-kel10` menunjukkan ping ke `192.168.10.5`. | B07 |

## 4. Interpretasi Baseline

- Trafik normal yang didukung bukti mencakup **ICMP** untuk pemeriksaan konektivitas dan **HTTP/TCP port 80** untuk pembacaan data aplikasi.
- Sumber client yang terlihat adalah `192.168.10.100`, sedangkan server tujuan adalah `192.168.10.5`. Pada respons HTTP, arah komunikasi berbalik dari server ke client.
- Bukti tcpdump menunjukkan trafik HTTP aplikasi diterima pada titik capture Security Onion. Event ping dan curl di Sguil menunjukkan adanya pemrosesan event IDS untuk aktivitas tersebut; hal ini tidak menyatakan seluruh trafik telah terdeteksi atau dianalisis.
- Event ping dan kebijakan curl **tidak otomatis berarti serangan**. Pada konteks baseline, keduanya berkaitan dengan aktivitas pemeriksaan normal dan harus dinilai bersama waktu, sumber, tujuan, serta aktivitas pengguna.
- Dashboard dirancang melakukan pembaruan sekitar setiap 5 detik. Screenshot capture memperlihatkan pembacaan kedua endpoint, tetapi belum cukup untuk menetapkan statistik interval, volume trafik, atau durasi pengamatan secara menyeluruh.
- Komunikasi simulator dengan Firebase Emulator melalui `127.0.0.1:9000` berlangsung di dalam Ubuntu Target dan tidak tercakup oleh capture jaringan antar-VM ini.


## 5. Daftar Bukti

Seluruh bukti berupa screenshot yang disimpan di repository; tidak diperlukan file capture atau direktori output di dalam VM.

| ID | Tangkapan Layar |
|---|---|
| B01 | [Status layanan Security Onion](evidence/baseline/baseline-01-security-onion-service-status.png) |
| B02 | [Respons API sensor HTTP 200 dan output Nmap](evidence/baseline/baseline-02-cyberops-api-sensor-http-200-and-nmap-output.png) |
| B03 | [RealTime Events Sguil: ping dan curl](evidence/baseline/baseline-03-sguil-realtime-ping-and-curl-events.png) |
| B04 | [Rincian event ICMP ping di Sguil](evidence/baseline/baseline-04-sguil-icmp-ping-event-details.png) |
| B05 | [Balasan ping Target pada CyberOps](evidence/baseline/baseline-05-cyberops-ping-target-replies.png) |
| B06 | [tcpdump: pembacaan sensor/riwayat dan HTTP 200](evidence/baseline/baseline-06-tcpdump-http-sensor-history-200.png) |
| B07 | [Self-ping pada Ubuntu Target](evidence/baseline/baseline-07-ping-from-iotuser.png) |
