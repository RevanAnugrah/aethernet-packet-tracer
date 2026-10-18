# AetherNet — Simulasi Jaringan Kampus Multi-Protokol (Cisco Packet Tracer)

Proyek lab jaringan skala kampus yang dibangun di Cisco Packet Tracer. Topologinya menggabungkan **OSPF, eBGP, EIGRP, dan RIPv2** dalam satu jaringan, lengkap dengan VLAN, DHCP, telepon IP (Call Manager Express), layanan server internal, dan manajemen perangkat lewat SSH.

Tujuan proyek ini adalah latihan merancang jaringan end-to-end: dari routing antar-area dan redistribusi rute, sampai layanan pendukung seperti DNS, syslog, dan NTP.

---

## Ringkasan Topologi

| Area | Protokol | Perangkat utama |
|---|---|---|
| **Pusat (HQ)** | OSPF area 0 | Core `C1`, `C2`, switch gedung `D1`–`D4` (Catalyst 3650) |
| **Cabang** | eBGP | `WAN1`, `WAN2` (AS 65000), `BR1` (AS 65001), `BR2` (AS 65002) |
| **Server dan distribusi** | EIGRP 100 | `EDGE1`, `EDGE2`, `SW28`, `R3`, `R4`, `R5` |
| **Jaringan R6** | RIPv2 | `R6`, `SW13` |

### Titik perbatasan antar-protokol

Rute antar-area hanya dipertukarkan di titik tertentu supaya tidak terjadi routing loop:

| Perangkat | Menghubungkan |
|---|---|
| `WAN1`, `WAN2` | BGP cabang ↔ OSPF pusat |
| `EDGE1` | OSPF pusat ↔ EIGRP (satu-satunya titik redistribusi) |
| `R5` | EIGRP ↔ RIPv2 (`R6`) |

---

## Arsitektur

```
                         ┌────────────── HQ (OSPF area 0) ──────────────┐
  Gedung 1..4 (D1-D4) ───┤  C1  ◄────────►  C2   (Server0, VOIP,        │
  masing-masing 2 core   │                        AdminCenter, PC/Phone)│
                         └───┬──────────────┬───────────────┬──────────┘
                             │              │               │
                        WAN1/WAN2        EDGE1/EDGE2     (VOIP, Server0)
                        (BGP AS 65000)   (OSPF ↔ EIGRP)
                         │      │             │
                      BR2       BR1          SW28 ── Server1
                  (AS 65002) (AS 65001)       │
                   LAN cabang LAN cabang      R3 ── R4 ── R5 ── R6 (RIPv2)
                                                        │      │
                                                     SW8      SW13
                                                  Server2      Server3
```

---

## Skema Pengalamatan

### VLAN dan subnet LAN

| Lokasi | VLAN data | VLAN telepon |
|---|---|---|
| Gedung 1 (`D1`) | 11 — `10.1.11.0/24` | 12 — `10.1.12.0/24` |
| Gedung 2 (`D2`) | 21 — `10.1.21.0/24` | 22 — `10.1.22.0/24` |
| Gedung 3 (`D3`) | 31 — `10.1.31.0/24` | 32 — `10.1.32.0/24` |
| Gedung 4 (`D4`) | 41 — `10.1.41.0/24` | 42 — `10.1.42.0/24` |
| Ruang Core (`C2`) | 51 — `10.1.51.0/24` | 52 — `10.1.52.0/24` |
| Cabang 1 (`BR1`) | 71 — `10.1.71.0/24` | 72 — `10.1.72.0/24` |
| Cabang 2 (`BR2`) | 81 — `10.1.81.0/24` | 82 — `10.1.82.0/24` |

| VLAN | Fungsi | Subnet |
|---|---|---|
| 60 | Server layanan (`Server0`) | `10.1.60.0/24` |
| 99 | Manajemen (`AdminCenter`) | `10.1.99.0/24` |
| 10 | `Server1` (di `SW28`) | `10.2.10.0/24` |
| — | LAN `R5` (`Server2`, `PC20`) | `10.2.20.0/24` |
| — | LAN `R6` (`Server3`, `PC25`) | `10.2.30.0/24` |

### Link point-to-point (/30)

| Subnet | Link |
|---|---|
| `10.0.0.4`, `.12`, `.20`, `.28` | `C1` ↔ `D1`–`D4` |
| `10.0.0.8`, `.16`, `.24`, `.32` | `C2` ↔ `D1`–`D4` |
| `10.0.0.36` | `C2` ↔ `VOIP` |
| `10.0.0.40` | `WAN1` ↔ `BR2` (serial) |
| `10.0.0.44` | `WAN2` ↔ `BR1` (serial) |
| `10.0.0.48`, `.52` | `BR1`, `BR2` ↔ switch multilayer cabang |
| `10.0.0.56`, `.60`, `.64`, `.68` | `WAN1`, `WAN2` ↔ `C1`, `C2` |
| `10.0.0.72`, `.76`, `.80`, `.84` | `EDGE1`, `EDGE2`, `SW28`, `R3` (EIGRP) |
| `10.0.0.88`, `.92`, `.96` | `R3` ↔ `R4` ↔ `R5` ↔ `R6` |
| `10.0.0.100`, `.104`, `.108`, `.112` | `EDGE1`, `EDGE2` ↔ `C1`, `C2` |

---

## Layanan dan Fitur

### Routing
- **OSPF area 0** di pusat, dengan routed port `/30` antara core dan tiap gedung.
- **eBGP** antara `WAN1`/`WAN2` dan router cabang lewat link serial.
- **EIGRP 100** untuk area server dan distribusi.
- **RIPv2** untuk jaringan di belakang `R6`.
- Redistribusi dua arah di `EDGE1` (OSPF ↔ EIGRP) dan `R5` (EIGRP ↔ RIPv2).

### Switching dan akses
- VLAN data dan telepon per gedung, trunk antar-switch akses.
- DHCP di switch multilayer dan router, dengan `option 150` untuk telepon IP.

### Telepon IP
- **Call Manager Express** di router `VOIP` (2811), nomor `1001`–`1013`.
- Telepon di tiap gedung terdaftar otomatis lewat DHCP option 150.

### Server internal

| Server | Alamat | Peran |
|---|---|---|
| `Server0` | `10.1.60.10` | DNS utama, HTTP, TFTP, syslog |
| `Server1` | `10.2.10.10` | Email, FTP, NTP |
| `Server2` | `10.2.20.10` | Autentikasi login perangkat (RADIUS) |
| `Server3` | `10.2.30.10` | DNS cadangan dan halaman dokumentasi |
| `AdminCenter` | `10.1.99.10` | PC manajemen (SSH ke perangkat) |

### Manajemen dan keamanan
- SSH versi 2 dengan user lokal di perangkat, akses dari `AdminCenter` di VLAN manajemen.
- Syslog terpusat ke `Server0`, timestamp log dengan `service timestamps`.
- Portal web internal di `Server0` dan dokumentasi di `Server3` (lihat folder `web/`).

---

## Status Pengujian

| Bagian | Status |
|---|---|
| OSPF pusat, adjacency core ↔ gedung | Terverifikasi |
| eBGP `WAN1`/`WAN2` ↔ `BR1`/`BR2` | Sesi *Established* terverifikasi |
| Jalur PC cabang → router cabang → `WAN` | Terverifikasi |
| DHCP, DNS, HTTP, syslog | Terverifikasi |
| Telepon IP gedung 1–4 dan `C2` terdaftar di CME | Terverifikasi |
| SSH dari `AdminCenter` | Terverifikasi di `C2`, `WAN1` |
| EDGE ↔ core, redistribusi OSPF ↔ EIGRP | Dikonfigurasi, verifikasi ujung ke ujung perlu dicatat |
| RIPv2 `R6`, redistribusi di `R5` | Dikonfigurasi, verifikasi ujung ke ujung perlu dicatat |
| Telepon cabang → CME | Menunggu jalur routing HQ ↔ cabang |
| RADIUS di router | Dikonfigurasi di `WAN1`, belum diuji penuh |

> Perbarui tabel ini setelah menjalankan tes ujung ke ujung (`ping`/`tracert` dari PC cabang ke server HQ dan sebaliknya).

---

## Verifikasi Cepat

| Tujuan | Perintah |
|---|---|
| Tetangga OSPF | `show ip ospf neighbor` |
| Sesi BGP | `show ip bgp summary` |
| Rute BGP dan routing | `show ip bgp`, `show ip route` |
| Tetangga EIGRP | `show ip eigrp neighbors` |
| Telepon terdaftar | `show ephone` (di `VOIP`) |
| Tetangga fisik | `show cdp neighbors` |
| Lease DHCP | `show ip dhcp binding` |

---

## Keterbatasan Packet Tracer yang Ditemui

Beberapa perintah IOS asli tidak didukung simulator, sehingga desainnya disesuaikan:

| Keterbatasan | Penyesuaian |
|---|---|
| **iBGP tidak didukung** | `WAN1` dan `WAN2` bertukar rute lewat OSPF, bukan iBGP |
| `neighbor ... default-originate` tidak ada | Rute default statis di router cabang |
| `ip radius source-interface` tidak ada | Client IP dicatat per interface di server RADIUS |
| RADIUS client tidak ada di Catalyst 3650 | RADIUS hanya di router, switch memakai user lokal |
| `dns-server` di DHCP hanya satu alamat | DNS dibagi per lokasi, tidak otomatis failover |
| Voice VLAN di port 3650 (`C2`) tidak stabil | Port telepon dijadikan access VLAN telepon |
| `show ephone-dn` tidak tersedia | Cek lewat `show ephone` |

---

## Struktur Repositori yang Disarankan

```
.
├── README.md
├── topology/
│   ├── aethernet.pkt          # file Packet Tracer
│   └── screenshots/           # tangkapan layar topologi dan hasil tes
├── configs/                   # hasil "show running-config" per perangkat
│   ├── C1.txt
│   ├── C2.txt
│   └── ...
└── web/
    ├── index.html             # halaman portal Server0
    └── index_server3.html     # halaman dokumentasi Server3
```

---

## Cara Menjalankan

1. Buka `topology/aethernet.pkt` di Cisco Packet Tracer.
2. Tunggu semua link menjadi hijau dan OSPF/BGP konvergen (sekitar 1–2 menit).
3. Dari `AdminCenter`, uji SSH ke perangkat, atau buka `http://10.1.60.10` untuk portal internal.

> Kredensial lab (user SSH, secret RADIUS) **tidak** disertakan di repositori. Ganti dengan nilai milik sendiri saat menyalin konfigurasi.

---

## Pengembangan Lanjutan

- ACL pembatas SSH hanya dari VLAN manajemen (`SSH-ADMIN`).
- Backup konfigurasi ke TFTP di `Server0` untuk seluruh perangkat.
- Filter rute BGP (prefix-list) di router cabang.
- Failover DNS otomatis di DHCP (di Packet Tracer perlu cara lain).

---

*Dibuat sebagai proyek belajar jaringan. Semua alamat dan konfigurasi hanya untuk simulasi.*
