# Laporan Akhir Praktikum — Firewall & NAT (Tugas Modul)

> **Platform:** PNETLab  
> **Topik:** Firewall NAT, Firewall Filter, Port Forwarding, DMZ  
> **Tanggal:** 7 Juni 2026

---

## Daftar Isi

1. [Topologi Jaringan](#1-topologi-jaringan)
2. [Tabel IP Address](#2-tabel-ip-address)
3. [Konfigurasi Tiap Perangkat](#3-konfigurasi-tiap-perangkat)
   - [3.1 MikroTik ISP](#31-mikrotik-isp)
   - [3.2 FortiGate Firewall](#32-fortigate-firewall)
   - [3.3 Cisco Router (vIOS)](#33-cisco-router-vios)
   - [3.4 Ubuntu Server DMZ](#34-ubuntu-server-dmz)
4. [Hasil Pengujian](#4-hasil-pengujian)
   - [4.1 Pengujian dari Client LAN](#41-pengujian-dari-client-lan)
   - [4.2 Pengujian dari Client WAN](#42-pengujian-dari-client-wan)
5. [Analisis](#5-analisis)
6. [Kesimpulan](#6-kesimpulan)

---

## 1. Topologi Jaringan

Simulasi dibangun di atas platform **PNETLab** dengan perangkat-perangkat berikut yang saling terhubung membentuk tiga zona jaringan: **WAN**, **LAN**, dan **DMZ**.

```
          [ Internet / Net ]
                 |
              eth1 (DHCP)
           [ MikroTik ISP ]
         eth2          eth3
    (10.10.10.1/30)  (172.16.100.1/24)
          |                  |
        port1           [ Client WAN ]
     [ FortiGate ]     172.16.100.10
    port2     port3
(10.20.20.1)  (192.168.20.1/24)
      |               |
    Gi0/0         [ Ubuntu DMZ ]
  [ vIOS / Cisco ]   192.168.20.10
    Gi0/1
  (192.168.10.1/24)
      |
  [ Client LAN ]
  192.168.10.10
```

![Topologi Jaringan PNETLab](img/1__gambar_topologi.png)

> *Gambar: Topologi jaringan lengkap pada PNETLab — tampak MikroTik ISP (atas), FortiGate sebagai firewall utama (tengah), Cisco vIOS sebagai router internal, Ubuntu Linux sebagai web server DMZ, dan dua TinyCore Linux sebagai client LAN dan WAN.*

---

## 2. Tabel IP Address

| Perangkat        | Interface         | IP Address          | Keterangan                        |
|------------------|-------------------|---------------------|-----------------------------------|
| MikroTik ISP     | ether1            | DHCP (dinamis)      | Koneksi ke jaringan lab/internet  |
| MikroTik ISP     | ether2            | 10.10.10.1/30       | Koneksi ke FortiGate (WAN)        |
| MikroTik ISP     | ether3            | 172.16.100.1/24     | Koneksi ke Client WAN             |
| FortiGate        | port1             | 10.10.10.2/30       | Sisi WAN (ke MikroTik)            |
| FortiGate        | port2             | 10.20.20.1/30       | Sisi internal (ke Cisco Router)   |
| FortiGate        | port3             | 192.168.20.1/24     | Zona DMZ                          |
| Cisco vIOS       | GigabitEthernet0/0| 10.20.20.2/30       | Koneksi ke FortiGate              |
| Cisco vIOS       | GigabitEthernet0/1| 192.168.10.1/24     | Koneksi ke jaringan LAN           |
| Ubuntu DMZ       | eth0              | 192.168.20.10/24    | Web server di zona DMZ            |
| Client LAN       | eth0              | 192.168.10.10/24    | TinyCore Linux sisi LAN           |
| Client WAN       | eth0              | 172.16.100.10/24    | TinyCore Linux sisi WAN           |

---

## 3. Konfigurasi Tiap Perangkat

### 3.1 MikroTik ISP

MikroTik dikonfigurasi sebagai ISP yang menghubungkan jaringan simulasi ke internet. Fungsinya mencakup: DHCP client ke jaringan lab, distribusi IP ke FortiGate dan Client WAN, NAT masquerade untuk akses internet, serta static route ke jaringan internal.

**IP Address yang dikonfigurasi:**
```
/ip address
0   10.10.10.1/30    → ether2  (ke FortiGate)
1   172.16.100.1/24  → ether3  (ke Client WAN)
2D  10.4.89.197/24   → ether1  (DHCP dari lab)
```

**Routing table:**
```
/ip route
0 ADS  0.0.0.0/0        gateway: 10.4.89.1      (default route DHCP)
1 ADC  10.4.89.0/24     via ether1
2 ADC  10.10.10.0/30    via ether2
3 ADC  172.16.100.0/24  via ether3
4 A S  192.168.10.0/24  gateway: 10.10.10.2     (static ke LAN via FortiGate)
5 A S  192.168.20.0/24  gateway: 10.10.10.2     (static ke DMZ via FortiGate)
```

**NAT Masquerade:**
```
/ip firewall nat
chain=srcnat  action=masquerade  out-interface=ether1
```

![Konfigurasi MikroTik ISP](img/3_4_cek_konfigurasi_mikrotik.png)

> *Screenshot: Output perintah `/ip address print`, `/ip route print`, dan `/ip firewall nat print` pada terminal MikroTik ISP. Terlihat seluruh IP address, routing table statis dan dinamis, serta aturan NAT masquerade telah terkonfigurasi dengan benar.*

---

### 3.2 FortiGate Firewall

FortiGate berperan sebagai firewall utama yang memisahkan dan mengontrol akses antar zona WAN, LAN, dan DMZ.

#### Interface & Static Route

```
config system interface
  edit "port1"
    set ip 10.10.10.2 255.255.255.252   ← WAN (ke MikroTik)
    set allowaccess ping
  edit "port2"
    set ip 10.20.20.1 255.255.255.252   ← ke Cisco Router (LAN side)
    set allowaccess ping
  edit "port3"
    set ip 192.168.20.1 255.255.255.0   ← DMZ

config router static
  edit 1
    set gateway 10.10.10.1              ← default route ke MikroTik
    set device "port1"
  edit 2
    set dst 192.168.10.0 255.255.255.0  ← static route ke LAN
    set gateway 10.20.20.2
    set device "port2"
```

![Konfigurasi Interface dan Static Route FortiGate](img/3__fortinet__show_system_interface___show_router_static.png)

> *Screenshot: Output `show system interface` dan `show router static` pada FortiGate. Terlihat konfigurasi IP pada port1 (WAN), port2 (internal), port3 (DMZ), serta static route default dan ke jaringan LAN.*

#### Firewall Address Object

```
edit "LAN_NETWORK"
  set subnet 192.168.10.0 255.255.255.0

edit "DMZ_Server"
  set subnet 192.168.20.10 255.255.255.255

edit "WAN_Client_Network"
  (jaringan sisi WAN)
```

![Firewall Address Object FortiGate](img/3_1__fortinet__show_firewall_address.png)

> *Screenshot: Output `show firewall address` pada FortiGate. Terlihat address object yang telah didefinisikan untuk LAN_NETWORK, DMZ_Server, dan berbagai object bawaan sistem.*

#### Firewall Policy & VIP

Empat policy dikonfigurasi untuk mengatur aliran trafik antar zona:

| No | Nama Policy       | Src Interface | Dst Interface | Aksi   | NAT    | Keterangan                              |
|----|-------------------|---------------|---------------|--------|--------|-----------------------------------------|
| 1  | LAN_to_WAN        | port2         | port1         | accept | enable | LAN → Internet, dengan NAT              |
| 2  | LAN_to_DMZ        | port2         | port3         | accept | off    | LAN → DMZ Server, tanpa NAT             |
| 3  | WAN_to_DMZ_HTTP   | port1         | port3         | accept | off    | WAN → DMZ via VIP (port forwarding)     |
| 4  | DMZ_to_Internet   | port3         | port1         | accept | enable | DMZ Server → Internet, dengan NAT       |

**VIP Port Forwarding:**
```
config firewall vip
  edit "VIP_DMZ_HTTP"
    set extip 10.10.10.2        ← IP publik FortiGate
    set mappedip "192.168.20.10" ← IP asli DMZ Server
    set extintf "port1"
    set portforward enable
    set extport 80
    set mappedport 80
```

![Firewall Policy dan VIP FortiGate](img/3_2__fortinet__show_firewall_policy_dan_vip.png)

> *Screenshot: Output `show firewall policy` dan `show firewall vip` pada FortiGate. Terlihat empat policy aktif beserta konfigurasi VIP yang memetakan port 80 IP publik ke IP DMZ Server.*

---

### 3.3 Cisco Router (vIOS)

Cisco vIOS bertugas sebagai router internal yang menghubungkan FortiGate ke jaringan LAN tempat Client LAN berada.

```
interface GigabitEthernet0/0
 ip address 10.20.20.2 255.255.255.252   ← ke FortiGate port2
 duplex auto
 speed auto
 media-type rj45

interface GigabitEthernet0/1
 ip address 192.168.10.1 255.255.255.0   ← ke jaringan LAN
 duplex auto
 speed auto
 media-type rj45

ip route 0.0.0.0 0.0.0.0 10.20.20.1     ← default route ke FortiGate
```

![Running-config Cisco vIOS](img/3_3_show_running_config_di_vios.png)

> *Screenshot: Output `show running-config` pada Cisco vIOS. Terlihat konfigurasi interface GigabitEthernet0/0 (10.20.20.2/30 ke FortiGate) dan GigabitEthernet0/1 (192.168.10.1/24 ke LAN), serta default route menuju FortiGate.*

---

### 3.4 Ubuntu Server DMZ

Ubuntu Server berperan sebagai web server yang ditempatkan di zona DMZ. Nginx digunakan sebagai web server dengan halaman yang dimodifikasi sesuai identitas kelompok.

**Konfigurasi IP Statis (`/etc/netplan/00-installer-config.yaml`):**
```yaml
network:
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.20.10/24
      gateway4: 192.168.20.1
      nameservers:
        addresses: [8.8.8.8]
  version: 2
```

**Status Nginx:**
```bash
$ systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled)
   Active: active (running) since Sun 2026-06-07 12:28:19 CST
   Main PID: 1981 (nginx)

$ curl http://192.168.20.10
Tumod_4_DMZ_Firewall_27-Peer to Peer
```

![Konfigurasi Ubuntu Server DMZ](img/3_5_Linux_cek_konfig.png)

> *Screenshot: Terminal Ubuntu Server DMZ menampilkan isi file netplan (IP statis 192.168.20.10/24), output `ip addr show eth0`, status nginx yang aktif berjalan, serta hasil `curl http://192.168.20.10` yang mengembalikan halaman identitas kelompok.*

---

## 4. Hasil Pengujian

### 4.1 Pengujian dari Client LAN

Client LAN dikonfigurasi dengan IP `192.168.10.10/24`, gateway `192.168.10.1` (Cisco Router).

| Target                       | IP Tujuan      | Hasil     | Keterangan                                  |
|------------------------------|----------------|-----------|---------------------------------------------|
| Cisco Router (gateway)       | 192.168.10.1   | ✅ Reply  | Koneksi ke gateway LAN berhasil             |
| FortiGate port2 (internal)   | 10.20.20.1     | ✅ Reply  | Koneksi ke firewall sisi internal berhasil  |
| Ubuntu DMZ Server            | 192.168.20.10  | ✅ Reply  | Policy LAN_to_DMZ pada FortiGate berjalan   |

```
VPCS> ip 192.168.10.10 255.255.255.0 192.168.10.1
PC1 : 192.168.10.10 255.255.255.0 gateway 192.168.10.1

VPCS> ping 192.168.10.1
84 bytes from 192.168.10.1  icmp_seq=1 ttl=255  time=1.610 ms
84 bytes from 192.168.10.1  icmp_seq=2 ttl=255  time=1.767 ms
...

VPCS> ping 10.20.20.1
84 bytes from 10.20.20.1  icmp_seq=1 ttl=254  time=5.883 ms
84 bytes from 10.20.20.1  icmp_seq=2 ttl=254  time=2.460 ms
...

VPCS> ping 192.168.20.10
84 bytes from 192.168.20.10  icmp_seq=1 ttl=62  time=3.423 ms
84 bytes from 192.168.20.10  icmp_seq=2 ttl=62  time=1.343 ms
...
```

![Pengujian dari Client LAN](img/4__pengujian_PC_LAN.png)

> *Screenshot: Terminal TinyCore Linux (Client LAN) menampilkan konfigurasi IP dan hasil ping ke gateway Cisco Router (192.168.10.1 — reply), ke FortiGate internal (10.20.20.1 — reply), dan ke DMZ Server (192.168.20.10 — reply). Semua pengujian berhasil sesuai policy yang dikonfigurasi.*

---

### 4.2 Pengujian dari Client WAN

Client WAN dikonfigurasi dengan IP `172.16.100.10/24`, gateway `172.16.100.1` (MikroTik ISP).

| Target                        | IP Tujuan      | Hasil       | Keterangan                                            |
|-------------------------------|----------------|-------------|-------------------------------------------------------|
| MikroTik ISP (gateway)        | 172.16.100.1   | ✅ Reply    | Koneksi ke gateway WAN berhasil                       |
| FortiGate WAN (port1)         | 10.10.10.2     | ✅ Reply    | FortiGate dapat di-ping dari WAN                      |
| Client LAN                    | 192.168.10.10  | ❌ Timeout  | Firewall memblokir akses langsung WAN → LAN ✓         |
| DMZ Server (IP asli)          | 192.168.20.10  | ❌ Timeout  | Firewall memblokir akses langsung WAN → DMZ IP ✓      |
| DMZ Server (via VIP/HTTP)     | 10.10.10.2:80  | ✅ Reply    | Port forwarding VIP berhasil meneruskan ke DMZ        |

```
VPCS> ip 172.16.100.10 255.255.255.0 172.16.100.1
PC1 : 172.16.100.10 255.255.255.0 gateway 172.16.100.1

VPCS> ping 172.16.100.1
84 bytes from 172.16.100.1  icmp_seq=1 ttl=64  time=1.596 ms
...

VPCS> ping 10.10.10.2
84 bytes from 10.10.10.2  icmp_seq=1 ttl=254  time=0.745 ms
...

VPCS> ping 192.168.10.10
192.168.10.10  icmp_seq=1  timeout
192.168.10.10  icmp_seq=2  timeout
...

VPCS> ping 192.168.20.10
192.168.20.10  icmp_seq=1  timeout
192.168.20.10  icmp_seq=2  timeout
...
```

Akses HTTP ke web server DMZ melalui VIP berhasil (diverifikasi dari Ubuntu Server DMZ via `curl`):
```bash
root@kvm:~# curl http://192.168.20.10
Tumod_4_DMZ_Firewall_27-Peer to Peer
```

![Pengujian dari Client WAN](img/4_2_PING_WAN_.png)

> *Screenshot: Terminal TinyCore Linux (Client WAN) menampilkan ping ke MikroTik (172.16.100.1 — reply), ke FortiGate WAN (10.10.10.2 — reply), ke Client LAN (192.168.10.10 — timeout), dan ke DMZ Server langsung (192.168.20.10 — timeout). Hasil timeout sesuai ekspektasi karena firewall memblokir akses langsung dari WAN ke jaringan internal.*

---

## 5. Analisis

### Efektivitas Segmentasi Jaringan

Topologi tiga zona (WAN–LAN–DMZ) terbukti berhasil diterapkan. Setiap zona hanya dapat berkomunikasi dengan zona lain sesuai dengan policy yang telah didefinisikan di FortiGate, tidak ada akses yang melanggar batas zona yang diizinkan.

### Peran Masing-Masing Perangkat

- **MikroTik ISP**: Bertindak sebagai gerbang internet menggunakan NAT masquerade pada `ether1`. Static route ke LAN dan DMZ memastikan paket balasan dari internet dapat sampai ke jaringan internal dengan benar.

- **FortiGate**: Berhasil menjalankan fungsi firewall berlapis. Policy `LAN_to_WAN` dengan NAT memungkinkan client LAN mengakses internet tanpa IP publik. Policy `LAN_to_DMZ` tanpa NAT memungkinkan komunikasi internal langsung. Policy `WAN_to_DMZ_HTTP` yang dikombinasikan dengan VIP memungkinkan akses terkontrol dari luar ke web server DMZ tanpa mengekspos IP aslinya.

- **VIP Port Forwarding**: Mekanisme ini membuktikan bahwa Destination NAT pada FortiGate berfungsi identik dengan DstNAT pada MikroTik — trafik yang masuk ke IP publik FortiGate (10.10.10.2:80) secara transparan diteruskan ke IP privat DMZ Server (192.168.20.10:80).

- **Cisco vIOS**: Berperan sebagai router murni antara FortiGate dan jaringan LAN. Default route ke FortiGate (10.20.20.1) memastikan seluruh trafik keluar dari LAN diarahkan melalui firewall.

### Verifikasi Hasil Pengujian

Seluruh hasil pengujian konsisten dengan konfigurasi policy yang dibuat:

| Skenario                         | Ekspektasi | Hasil Aktual | Status |
|----------------------------------|------------|--------------|--------|
| LAN ping ke gateway Cisco        | Reply      | Reply        | ✅ Sesuai |
| LAN ping ke FortiGate internal   | Reply      | Reply        | ✅ Sesuai |
| LAN ping ke DMZ Server           | Reply      | Reply        | ✅ Sesuai |
| WAN ping ke MikroTik             | Reply      | Reply        | ✅ Sesuai |
| WAN ping ke FortiGate WAN        | Reply      | Reply        | ✅ Sesuai |
| WAN ping ke Client LAN           | Timeout    | Timeout      | ✅ Sesuai |
| WAN ping ke DMZ Server (direct)  | Timeout    | Timeout      | ✅ Sesuai |
| WAN akses HTTP via VIP           | Reply      | Reply        | ✅ Sesuai |

---

## 6. Kesimpulan

Seluruh konfigurasi pada tugas modul berhasil diselesaikan dan diverifikasi melalui pengujian end-to-end. Beberapa poin penting yang dapat disimpulkan:

1. **Segmentasi zona** WAN–LAN–DMZ berhasil diterapkan menggunakan FortiGate sebagai firewall utama, membuktikan pentingnya pemisahan zona dalam arsitektur keamanan jaringan.

2. **NAT masquerade** pada MikroTik memungkinkan seluruh perangkat dalam simulasi mengakses internet hanya dengan satu IP publik, sesuai prinsip Source NAT.

3. **VIP port forwarding** pada FortiGate membuktikan bahwa layanan internal (web server DMZ) dapat diekspos ke publik secara aman melalui Destination NAT tanpa mengungkap IP asli server.

4. **Firewall policy** berbasis zona terbukti efektif: akses yang tidak diizinkan (WAN langsung ke LAN atau DMZ) berhasil diblokir, sementara akses yang diizinkan (LAN ke DMZ, WAN via VIP) berjalan lancar.

5. Penggunaan platform **PNETLab** memungkinkan simulasi topologi kompleks multi-vendor (MikroTik, FortiGate, Cisco, Linux) dalam satu lingkungan terpadu secara efisien.
