# MTCSWE — Implementation and Operation Guide

## Modul 1 — Introduction: Konsep Dasar Layer 2 Forwarding

**Tujuan:** Memahami cara switch meneruskan frame, konsep unicast, multicast, dan broadcast, pembelajaran MAC address, serta perbedaan antara bridge pada RouterOS, switch chip pada RouterBOARD, dan SwOS.

**Ruang lingkup sesuai outline:** Layer 2 forwarding concepts, unicast, multicast and broadcast traffic, MAC learning in bridges and switches, interface settings, RouterOS bridge overview, RouterBOARD switch chip overview, RouterBOARDs with basic switch chips, Cloud Router Switch (CRS) series devices with advanced switch chips, dan SwitchOS (SwOS) brief overview.

### Konsep Utama

- **Unicast:** Komunikasi dari satu pengirim kepada satu penerima. Switch mempelajari MAC tujuan dan meneruskan frame ke port yang sesuai.
- **Broadcast:** Komunikasi kepada seluruh port dalam broadcast domain yang sama. Contohnya adalah ARP dan DHCP Discover. Broadcast yang berlebihan menyebabkan broadcast storm.
- **Multicast:** Komunikasi kepada sekelompok penerima. Contohnya adalah streaming IPTV dan routing protocol. Tanpa pengendalian, multicast diperlakukan seperti broadcast.
- **MAC Learning:** Switch mencatat MAC sumber, port, dan VLAN pada MAC table atau bridge host table. Entri tersebut akan kedaluwarsa apabila tidak terdapat traffic.
- **Bridge dan Switch Chip:** Bridge pada CPU RouterOS bersifat fleksibel, tetapi memiliki keterbatasan throughput. Switch chip meneruskan frame pada perangkat keras sehingga memiliki throughput yang lebih tinggi serta mendukung fitur seperti VLAN pada perangkat keras, access control list, dan traffic storm control. Pada perangkat CRS, pemanfaatan switch chip sangat disarankan.

### Implementasi: Bridge Dasar dan Pemeriksaan MAC Table

```routeros
/interface bridge add name=bridge1 protocol-mode=rstp
/interface bridge port add bridge=bridge1 interface=ether1
/interface bridge port add bridge=bridge1 interface=ether2
/interface bridge port add bridge=bridge1 interface=ether3
/interface bridge port add bridge=bridge1 interface=ether4
/interface bridge port add bridge=bridge1 interface=ether5

# Pemeriksaan host / MAC table
/interface bridge host print
/interface bridge host print where bridge=bridge1

# Pemeriksaan switch chip yang tersedia
/interface ethernet switch print
/interface ethernet switch port print
```

### Verifikasi dan Operasional

```routeros
/interface bridge host print
# Flags: D=dynamic, L=local, E=external
# Apabila MAC tidak muncul, perlu diperiksa kabel, status interface, dan keanggotaan port pada bridge

/interface bridge monitor bridge1
/interface ethernet monitor ether1
```

### Perbandingan RouterOS v6 dan v7

| Aspek | RouterOS v6 | RouterOS v7 |
| --- | --- | --- |
| Bridge | Bridge berbasis CPU dengan pengaturan `use-ip-firewall` dan `fast-forward` | Bridge dengan hardware offload otomatis dan pengaturan `hw-offload` per port |
| MAC Learning | MAC table melalui `/interface bridge host`, kapasitas bergantung pada model | MAC table melalui perintah yang sama, dengan peningkatan stabilitas dan kapasitas pada CRS3xx dan CRS5xx |
| Switch Chip | Konfigurasi melalui `/interface ethernet switch` dengan fitur terbatas pada beberapa model | Konfigurasi switch chip terintegrasi dengan bridge, hardware offload untuk VLAN dan filter lebih konsisten |
| Interface Settings | Pengaturan `hw` pada bridge port untuk mengaktifkan atau menonaktifkan offload | Pengaturan `hw-offload` dan `edge` pada bridge port, serta status offload terlihat pada `/interface bridge port print` |

**Konfigurasi RouterOS v6**

```routeros
/interface bridge add name=bridge1 protocol-mode=rstp
/interface bridge port add bridge=bridge1 interface=ether1 hw=yes
/interface bridge port add bridge=bridge1 interface=ether2 hw=yes
/interface ethernet switch print
/interface bridge host print
```

**Konfigurasi RouterOS v7**

```routeros
/interface bridge add name=bridge1 protocol-mode=rstp vlan-filtering=no
/interface bridge port add bridge=bridge1 interface=ether1 hw-offload=yes edge=yes
/interface bridge port add bridge=bridge1 interface=ether2 hw-offload=yes edge=yes
/interface bridge host print
/interface bridge port print detail where hw-offload=yes
```

**Catatan Migrasi:** Pada migrasi dari v6 ke v7, perlu diperiksa status hardware offload pada setiap port. Port yang pada v6 menggunakan `hw=no` perlu disesuaikan pada v7 menjadi `hw-offload=no` apabila memang diperlukan untuk fitur bridge filter tertentu. Pencadangan dan export konfigurasi perlu dilakukan sebelum migrasi.

### Studi Kasus: Kantor dengan 20 PC, 1 Server, dan 1 Printer

Sebuah kantor memasang SW-CORE CRS326. Seluruh PC berada pada ether1 hingga ether20, server pada ether21, dan printer pada ether22. Pada tahap awal, seluruh perangkat berada dalam satu bridge tanpa VLAN. Setelah tiga PC dihubungkan, administrator memeriksa `/interface bridge host print` dan memastikan MAC PC telah dipelajari. Komunikasi ping antar PC berhasil.

Pada hari berikutnya, printer tidak dapat dijangkau melalui ping. Setelah dilakukan pemeriksaan, diketahui bahwa ether22 berada dalam kondisi disabled karena port tersebut sebelumnya digunakan untuk pengujian PoE dan belum diaktifkan kembali. Solusi yang diterapkan adalah mengaktifkan kembali interface tersebut dan memeriksa host table.

**Pembelajaran operasional:** MAC table merupakan instrumen diagnostik pertama untuk permasalahan Layer 2. Apabila MAC tidak dipelajari, permasalahan umumnya terdapat pada lapisan fisik atau port belum dimasukkan ke dalam bridge.

> **Kesalahan yang sering terjadi pada Modul 1:** Port tidak ditambahkan ke bridge sehingga komunikasi tidak dapat berlangsung. Selain itu, pembuatan dua bridge yang saling tumpang tindih dapat menyebabkan loop. Disarankan untuk menggunakan satu bridge utama pada setiap switch, kecuali terdapat alasan khusus.


## Modul 2 — MTU: Pengelolaan MTU, L2MTU, dan Jumbo Frame

**Tujuan:** Memahami MTU, L2MTU, jumbo frame, dan dampak ketidakkonsistenan MTU terhadap jaringan.

**Ruang lingkup sesuai outline:** MTU, RouterOS bridge overview, L2MTU, jumbo frames, dan potential MTU issues.

### Konsep Utama

- **MTU:** Ukuran maksimum payload Layer 3 (IP) yang dapat dikirim tanpa fragmentasi. Nilai standar Ethernet adalah 1500 byte.
- **L2MTU:** Ukuran maksimum frame Layer 2 termasuk header Ethernet dan tag VLAN. Nilai L2MTU harus lebih besar daripada MTU. Sebagai contoh, MTU 1500 memerlukan L2MTU minimal 1500 ditambah header dan tag VLAN.
- **Jumbo Frame:** Frame dengan ukuran di atas 1500 byte, umumnya 9000 byte. Jumbo frame bermanfaat untuk storage, iSCSI, pencadangan, dan transfer antar server. Jumbo frame harus didukung secara end-to-end, yaitu pada NIC, switch, dan server.
- **Dampak Ketidakkonsistenan MTU:** Apabila satu sisi menggunakan jumbo frame dan sisi lain menggunakan standar, paket berukuran besar akan dibuang atau terfragmentasi. Gejalanya antara lain ping berukuran kecil berhasil, ping berukuran besar gagal, situs web tertentu tidak dapat dibuka, dan VPN terasa lambat.

### Implementasi

```routeros
# Pemeriksaan kemampuan setiap interface
/interface print detail
/interface ethernet print detail where name=ether1
# Perlu diperhatikan kolom mtu dan l2mtu

# Jumbo frame diaktifkan hanya pada tautan antar switch dan server
# Jumbo frame tidak diaktifkan pada port pengguna atau access point yang tidak memerlukan
/interface ethernet set ether21 l2mtu=9216 mtu=9000
/interface ethernet set ether22 l2mtu=9216 mtu=9000
/interface ethernet set sfp-sfpplus1 l2mtu=9216 mtu=9000

# MTU bridge perlu disesuaikan
/interface bridge set bridge1 mtu=9000

# Port pengguna tetap menggunakan nilai standar
/interface ethernet set ether1 mtu=1500 l2mtu=1598
```

### Verifikasi

```routeros
/tool ping 192.168.10.20 size=1500 do-not-fragment=yes count=5
/tool ping 192.168.10.20 size=8972 do-not-fragment=yes count=5
# Apabila ping 8972 gagal sedangkan ping 1500 berhasil, MTU pada jalur tidak konsisten

/interface ethernet monitor ether21
# Nilai rx-too-long dan rx-fcs-error dapat mengindikasikan permasalahan MTU atau kabel
```

### Perbandingan RouterOS v6 dan v7

| Aspek | RouterOS v6 | RouterOS v7 |
| --- | --- | --- |
| MTU | MTU diatur per interface melalui `/interface ethernet set mtu`, nilai bridge mengikuti interface | MTU diatur dengan cara yang sama, tetapi bridge MTU lebih konsisten mengikuti L2MTU terendah pada port anggota |
| L2MTU | L2MTU ditampilkan pada `/interface ethernet print`, perubahan pada beberapa model memerlukan restart | L2MTU ditampilkan pada `/interface ethernet print` dan `/interface ethernet monitor`, perubahan diterapkan tanpa restart pada sebagian besar model CRS3xx |
| Jumbo Frame | Dukungan jumbo frame bergantung pada model, nilai maksimum bervariasi antara 4076 hingga 9216 | Dukungan jumbo frame lebih konsisten pada CRS3xx dan CRS5xx, nilai 9216 didukung pada sebagian besar uplink |
| Diagnostik | Pengujian dengan `/tool ping size` dan `do-not-fragment` | Pengujian yang sama, ditambah `rx-too-long` dan `rx-fcs-error` pada monitor yang lebih rinci |

**Konfigurasi RouterOS v6**

```routeros
/interface ethernet set ether21 mtu=9000 l2mtu=9216
/interface ethernet set ether22 mtu=9000 l2mtu=9216
/interface bridge set bridge1 mtu=9000
/tool ping 192.168.10.20 size=8972 do-not-fragment=yes count=5
```

**Konfigurasi RouterOS v7**

```routeros
/interface ethernet set ether21 mtu=9000 l2mtu=9216
/interface ethernet set ether22 mtu=9000 l2mtu=9216
/interface bridge set bridge1 mtu=9000
/interface ethernet monitor ether21
/tool ping 192.168.10.20 size=8972 do-not-fragment=yes count=5
```

**Catatan Migrasi:** Setelah migrasi ke v7, perlu dilakukan pemeriksaan ulang L2MTU pada seluruh uplink dan server. Nilai yang pada v6 tidak diterapkan secara penuh dapat menjadi aktif pada v7 sehingga perlu dilakukan pengujian ping do-not-fragment pada jalur server dan antar switch.

### Studi Kasus: Pencadangan Server yang Lambat dan Sering Gagal

Server pencadangan pada ether21 dan NAS pada ether22 sering mengalami kegagalan saat mentransfer berkas berukuran di atas 100 MB. Ping standar berhasil. Setelah dilakukan pemeriksaan, diketahui bahwa server telah menggunakan MTU 9000, sedangkan switch masih menggunakan MTU 1500. Setelah L2MTU switch ditingkatkan menjadi 9216 dan MTU bridge disesuaikan, throughput pencadangan meningkat dari 80 Mbps menjadi 800 Mbps dan tidak lagi terputus.

**Pembelajaran operasional:** Jumbo frame hanya diaktifkan pada jalur yang memerlukan, yaitu antar server dan antar switch. Pengujian dilakukan dengan ping do-not-fragment dan didokumentasikan.

> **Aspek operasional MTU:** Standar yang disarankan adalah MTU 1500 untuk pengguna dan MTU 9000 untuk uplink antar switch dan server apabila didukung. Setiap penambahan perangkat baru perlu disertai pemeriksaan MTU.

---

## Modul 3 — VLAN: Implementasi VLAN 802.1Q dan 802.1ad

**Tujuan:** Menguasai VLAN 802.1Q, 802.1ad (QinQ), VLAN pada bridge, VLAN pada switch chip, dan inter-VLAN routing.

**Ruang lingkup sesuai outline:** 802.1Q and 802.1ad VLAN overview and tagging concepts, RouterOS VLAN interfaces, port based VLAN (VLAN bridging), inter-VLAN routing (router on a stick), VLANs in basic switch chips, VLANs in bridge interfaces, port based VLAN, MAC based VLAN, protocol based VLAN, QinQ (802.1ad), QinQ implementation with bridge VLAN filtering, dan QinQ implementation with VLAN interfaces.

### Konsep Utama

- **Access / Untagged:** Port untuk perangkat akhir seperti PC, printer, dan kamera. Frame yang keluar tidak memiliki tag, sedangkan frame yang masuk diberi PVID.
- **Trunk / Tagged:** Port antar switch atau menuju access point dan router. Port trunk membawa banyak VLAN dengan tag.
- **PVID:** VLAN yang diberikan kepada frame untagged yang masuk melalui port access.
- **Bridge VLAN Filtering:** Metode yang direkomendasikan pada RouterOS v7 untuk VLAN pada bridge. Satu bridge menggunakan satu tabel VLAN.
- **Port Based VLAN:** VLAN ditentukan berdasarkan port. Metode ini paling umum digunakan.
- **MAC Based VLAN:** VLAN ditentukan berdasarkan MAC address perangkat. Metode ini bermanfaat untuk perangkat yang berpindah port tetapi memerlukan VLAN yang sama.
- **Protocol Based VLAN:** VLAN ditentukan berdasarkan protokol Layer 3, misalnya IPv4 atau IPv6. Metode ini digunakan pada kebutuhan segmentasi berbasis protokol.
- **Inter-VLAN Routing:** Komunikasi antar VLAN harus melalui perangkat Layer 3. Model router on a stick menggunakan satu trunk menuju router dengan setiap VLAN sebagai sub-interface.
- **QinQ (802.1ad):** VLAN di dalam VLAN. Penyedia layanan dapat membawa banyak VLAN pelanggan dalam satu S-VLAN.

### Implementasi Lengkap: Bridge VLAN Filtering

Contoh: ether1 hingga ether4 merupakan access VLAN 10, ether5 hingga ether6 merupakan access VLAN 20, ether7 merupakan access VLAN 30, sfp-sfpplus1 merupakan trunk menuju SW-ACC1, dan sfp-sfpplus2 merupakan trunk menuju router.

```routeros
/interface bridge add name=bridge1 vlan-filtering=no protocol-mode=rstp
/interface bridge port add bridge=bridge1 interface=ether1 pvid=10
/interface bridge port add bridge=bridge1 interface=ether2 pvid=10
/interface bridge port add bridge=bridge1 interface=ether3 pvid=10
/interface bridge port add bridge=bridge1 interface=ether4 pvid=10
/interface bridge port add bridge=bridge1 interface=ether5 pvid=20
/interface bridge port add bridge=bridge1 interface=ether6 pvid=20
/interface bridge port add bridge=bridge1 interface=ether7 pvid=30
/interface bridge port add bridge=bridge1 interface=sfp-sfpplus1 pvid=1
/interface bridge port add bridge=bridge1 interface=sfp-sfpplus2 pvid=1

# Tabel VLAN
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=sfp-sfpplus1,sfp-sfpplus2 untagged=ether1,ether2,ether3,ether4
/interface bridge vlan add bridge=bridge1 vlan-ids=20 tagged=sfp-sfpplus1,sfp-sfpplus2 untagged=ether5,ether6
/interface bridge vlan add bridge=bridge1 vlan-ids=30 tagged=sfp-sfpplus1,sfp-sfpplus2 untagged=ether7
/interface bridge vlan add bridge=bridge1 vlan-ids=99 tagged=sfp-sfpplus1,sfp-sfpplus2 untagged=bridge1

# Filtering diaktifkan setelah tabel lengkap
/interface bridge set bridge1 vlan-filtering=yes

# Inter-VLAN routing pada router atau core Layer 3
/interface vlan add name=vlan10 interface=bridge1 vlan-id=10
/interface vlan add name=vlan20 interface=bridge1 vlan-id=20
/interface vlan add name=vlan99 interface=bridge1 vlan-id=99
/ip address add address=192.168.10.1/24 interface=vlan10
/ip address add address=192.168.20.1/24 interface=vlan20
/ip address add address=192.168.99.1/24 interface=vlan99
```

### VLAN pada Switch Chip

```routeros
# Contoh VLAN pada switch chip untuk CRS3xx
/interface ethernet switch vlan add ports=ether1,ether2,ether3,ether4,sfp-sfpplus1 vlan-id=10 independent-learning=yes
/interface ethernet switch vlan add ports=ether5,ether6,sfp-sfpplus1 vlan-id=20 independent-learning=yes
/interface ethernet switch port set ether1 vlan-mode=secure default-vlan-id=10
/interface ethernet switch port set ether5 vlan-mode=secure default-vlan-id=20
```

### MAC Based VLAN dan Protocol Based VLAN

```routeros
# Contoh konseptual MAC based VLAN pada bridge
# Perangkat dengan MAC tertentu ditempatkan pada VLAN 10 meskipun berpindah port
# Implementasi rinci bergantung pada model perangkat dan versi RouterOS
# Perlu diverifikasi pada /interface bridge vlan dan /interface bridge port

# Contoh konseptual protocol based VLAN
# Traffic IPv4 ditempatkan pada VLAN 10, traffic IPv6 pada VLAN 20
# Pada CRS, protocol based VLAN dikonfigurasi melalui switch rule
/interface ethernet switch rule add ports=ether1 mac-protocol=ip vlan-id=10
```

### QinQ — Implementasi dengan Bridge VLAN Filtering dan VLAN Interface

```routeros
# QinQ dengan bridge VLAN filtering
# S-VLAN 200 membawa C-VLAN 10 dan 20 dari pelanggan
/interface bridge vlan add bridge=bridge1 vlan-ids=200 tagged=sfp-sfpplus1,sfp-sfpplus2

# QinQ dengan VLAN interface
/interface vlan add name=qinq-200 interface=sfp-sfpplus1 vlan-id=200 use-service-tag=yes
```

### Verifikasi

```routeros
/interface bridge vlan print
/interface bridge port print detail
/interface bridge host print where vlan-id=10
/tool mac-scan interface=ether1
# Pengujian: PC pada VLAN 10 tidak dapat melakukan ping ke PC pada VLAN 20 tanpa routing
# Apabila komunikasi tetap berlangsung, perlu diperiksa vlan-filtering dan bridge VLAN table
```

### Perbandingan RouterOS v6 dan v7

| Aspek | RouterOS v6 | RouterOS v7 |
| --- | --- | --- |
| Bridge VLAN Filtering | Diperkenalkan pada 6.41, belum menjadi metode utama pada instalasi lama | Metode yang direkomendasikan, hardware offload untuk VLAN filtering didukung pada CRS3xx dan CRS5xx |
| VLAN pada Switch Chip | Metode utama sebelum 6.41 melalui `/interface ethernet switch vlan` | Tetap didukung, tetapi untuk konsistensi disarankan menggunakan bridge VLAN table kecuali terdapat kebutuhan khusus |
| MAC Based VLAN | Dukungan terbatas dan bergantung pada model | Dukungan lebih baik melalui bridge dan switch rule pada CRS |
| Protocol Based VLAN | Konfigurasi melalui switch rule, dokumentasi terbatas | Konfigurasi melalui switch rule dengan sintaks yang sama, tetapi lebih stabil pada v7 |
| QinQ (802.1ad) | QinQ melalui VLAN interface dengan `use-service-tag` | QinQ melalui VLAN interface dan bridge VLAN filtering, dengan penanganan S-VLAN dan C-VLAN yang lebih konsisten |

**Konfigurasi RouterOS v6**

```routeros
/interface bridge add name=bridge1 vlan-filtering=no protocol-mode=rstp
/interface bridge port add bridge=bridge1 interface=ether1 pvid=10
/interface bridge port add bridge=bridge1 interface=sfp-sfpplus1 pvid=1
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=sfp-sfpplus1 untagged=ether1
/interface bridge set bridge1 vlan-filtering=yes
/interface vlan add name=vlan10 interface=bridge1 vlan-id=10
```

**Konfigurasi RouterOS v7**

```routeros
/interface bridge add name=bridge1 vlan-filtering=no protocol-mode=rstp
/interface bridge port add bridge=bridge1 interface=ether1 pvid=10 hw-offload=yes
/interface bridge port add bridge=bridge1 interface=sfp-sfpplus1 pvid=1 hw-offload=yes
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=sfp-sfpplus1 untagged=ether1
/interface bridge set bridge1 vlan-filtering=yes
/interface vlan add name=vlan10 interface=bridge1 vlan-id=10
/interface vlan add name=qinq-200 interface=sfp-sfpplus1 vlan-id=200 use-service-tag=yes
```

**Catatan Migrasi:** Migrasi VLAN dari switch chip pada v6 ke bridge VLAN filtering pada v7 perlu dilakukan secara bertahap. Tabel VLAN perlu disalin secara lengkap, kemudian filtering diaktifkan setelah tabel lengkap. Pengujian perlu dilakukan pada satu switch terlebih dahulu sebelum diterapkan pada seluruh jaringan.

### Studi Kasus: Institusi Pendidikan dengan Tiga Jaringan Terpisah

Sebuah institusi pendidikan memiliki satu switch core dan dua switch access. Kebutuhan yang ditetapkan adalah VLAN 10 untuk guru dan tata usaha, VLAN 20 untuk siswa melalui WiFi, dan VLAN 30 untuk CCTV. Pada kondisi awal, seluruh jaringan tercampur sehingga siswa dapat mengakses server nilai. Setelah implementasi bridge VLAN filtering dan firewall antar VLAN pada router, permasalahan tersebut terselesaikan. VLAN 20 tidak diizinkan menuju VLAN 10, tetapi diizinkan menuju internet. Access point pada trunk membawa VLAN 10 dan 20 dengan SSID terpisah. CCTV pada VLAN 30 tidak diizinkan menuju internet.

**Aspek operasional:** Setiap penambahan kelas baru hanya memerlukan penambahan port access dengan PVID yang sesuai dan memastikan port trunk telah tagged untuk VLAN tersebut.

> **Kesalahan yang sering terjadi pada Modul 3:** Mengaktifkan `vlan-filtering=yes` sebelum tabel VLAN lengkap sehingga akses terputus dan diperlukan reset melalui MAC WinBox. Disarankan untuk selalu menyiapkan akses MAC atau console serta pencadangan sebelum mengaktifkan filtering.


## Modul 4 — Spanning Tree Protocol: Pencegahan Loop dengan Mempertahankan Redundansi

**Tujuan:** Memahami STP, RSTP, MSTP, BPDU, serta keamanan spanning tree.

**Ruang lingkup sesuai outline:** Spanning tree protocol (STP) concepts, STP bridge priority, STP port path cost, STP and RSTP comparison, multiple spanning tree (MSTP) concepts, MSTP definition, MSTP regions, CST/CIST, bridge protocol data unit (BPDU), dan spanning tree security.

### Konsep Utama

- **Root Bridge:** Switch dengan bridge priority terendah yang menjadi acuan topologi. Root bridge harus ditentukan secara manual dan tidak diserahkan kepada switch dengan spesifikasi terendah.
- **Peran Port:** Root Port, Designated Port, Alternate Port, dan Backup Port. Port Alternate dan Backup merupakan port yang tidak meneruskan traffic pada kondisi normal sebagai konsekuensi dari redundansi.
- **STP (802.1D):** Protokol awal dengan waktu konvergensi 30 hingga 50 detik.
- **RSTP (802.1w):** Pengembangan STP dengan waktu konvergensi 1 hingga 2 detik sehingga lebih sesuai untuk jaringan modern.
- **MSTP (802.1s):** Mengelompokkan banyak VLAN ke dalam beberapa instance sehingga beban STP lebih ringan dan load balancing antar uplink dapat dilakukan.
- **CST / CIST:** Common Spanning Tree dan Common and Internal Spanning Tree merupakan konsep pada MSTP untuk menghubungkan region MSTP dengan jaringan STP atau RSTP di luar region.
- **BPDU (Bridge Protocol Data Unit):** Pesan antar switch untuk membangun topologi. Apabila BPDU diblokir atau dimanipulasi, topologi menjadi tidak konsisten.

### Perbandingan STP dan RSTP

| Aspek | STP (802.1D) | RSTP (802.1w) |
| --- | --- | --- |
| Waktu konvergensi | 30 hingga 50 detik | 1 hingga 3 detik |
| Peran port | Root, Designated, Blocking | Root, Designated, Alternate, Backup |
| Status port | Blocking, Listening, Learning, Forwarding | Discarding, Learning, Forwarding |
| Kesesuaian | Jaringan lama | Jaringan modern, direkomendasikan |

### Implementasi: RSTP Dasar

```routeros
# SW-CORE menjadi root dengan priority terendah
/interface bridge set bridge1 protocol-mode=rstp priority=0x1000
# SW-ACC1 dan SW-ACC2 menjadi secondary
# Pada SW-ACC1:
/interface bridge set bridge1 protocol-mode=rstp priority=0x2000
# Pada SW-ACC2:
/interface bridge set bridge1 protocol-mode=rstp priority=0x3000

# Pengaturan cost pada uplink agar jalur utama lebih diprioritaskan
/interface bridge port set [find interface=sfp-sfpplus1] path-cost=10
/interface bridge port set [find interface=sfp-sfpplus2] path-cost=20

# Port menuju perangkat akhir dijadikan edge agar langsung forwarding
/interface bridge port set [find interface=ether1] edge=yes
/interface bridge port set [find interface=ether2] edge=yes
```

### Implementasi: MSTP untuk Banyak VLAN

```routeros
# Contoh 2 instance: Instance 1 untuk VLAN 10 dan 99, Instance 2 untuk VLAN 20 dan 30
/interface bridge mst add bridge=bridge1 identifier=1 vlan-mapping=10,99 priority=0x1000
/interface bridge mst add bridge=bridge1 identifier=2 vlan-mapping=20,30 priority=0x2000
/interface bridge set bridge1 protocol-mode=mstp region-name=SEKOLAH-REGION revision=1

# Verifikasi MSTP
/interface bridge mst print
/interface bridge mst port print
```

### Verifikasi dan Operasional

```routeros
/interface bridge monitor bridge1
# Perlu diperhatikan: root-bridge, root-path-cost, dan port yang discarding

/interface bridge port print
# Status yang perlu diperiksa: forwarding, discarding, learning

# Simulasi: kabel uplink utama dilepas, kemudian diukur durasi failover
# Failover RSTP seharusnya terjadi dalam 1 hingga 3 detik, bukan 30 detik
```

### Perbandingan RouterOS v6 dan v7

| Aspek | RouterOS v6 | RouterOS v7 |
| --- | --- | --- |
| STP dan RSTP | STP dan RSTP tersedia, RSTP menjadi pilihan utama | STP dan RSTP tersedia dengan konvergensi yang sama, ditambah perbaikan stabilitas BPDU |
| MSTP | MSTP tersedia mulai 6.43 dengan fitur terbatas | MSTP stabil, konfigurasi region, revision, dan instance melalui `/interface bridge mst` lebih lengkap |
| CST dan CIST | Konsep CST dan CIST berlaku, tetapi konfigurasi MSTP masih terbatas | CST dan CIST diterapkan secara konsisten untuk interoperabilitas antar region MSTP dan jaringan RSTP |
| BPDU Guard | BPDU guard tersedia pada versi 6.45 ke atas dengan perilaku terbatas pada beberapa model | BPDU guard stabil dan direkomendasikan pada seluruh port access |
| Edge Port | Pengaturan edge melalui bridge port, tetapi tidak selalu konsisten pada beberapa model | Pengaturan edge melalui `edge=yes` pada bridge port, dengan deteksi edge yang lebih andal |

**Konfigurasi RouterOS v6**

```routeros
/interface bridge set bridge1 protocol-mode=rstp priority=0x1000
/interface bridge port set [find interface=sfp-sfpplus1] path-cost=10
/interface bridge port set [find interface=ether1] edge=yes
/interface bridge monitor bridge1
```

**Konfigurasi RouterOS v7**

```routeros
/interface bridge set bridge1 protocol-mode=mstp priority=0x1000 region-name=SEKOLAH-REGION revision=1
/interface bridge mst add bridge=bridge1 identifier=1 vlan-mapping=10,99 priority=0x1000
/interface bridge mst add bridge=bridge1 identifier=2 vlan-mapping=20,30 priority=0x2000
/interface bridge port set [find interface=ether1] edge=yes bpdu-guard=yes
/interface bridge monitor bridge1
/interface bridge mst print
```

**Catatan Migrasi:** Apabila jaringan v6 hanya menggunakan RSTP dan akan berpindah ke MSTP pada v7, perlu dipastikan seluruh switch menggunakan region-name dan revision yang sama. Perbedaan region menyebabkan MSTP membentuk CIST yang terpisah sehingga load balancing tidak berjalan sesuai rencana.

### Studi Kasus: Gedung Dua Lantai dengan Loop Akibat Kesalahan Pengkabelan

Sebuah gedung pendidikan memiliki dua uplink antar lantai untuk redundansi. Pada suatu waktu, teknisi secara tidak sengaja menghubungkan kabel patch dari switch yang sama ke switch yang sama sehingga terbentuk loop. Tanpa STP, broadcast storm menyebabkan CPU seluruh switch mencapai 100 persen dan jaringan tidak dapat digunakan. Setelah RSTP diaktifkan dan SW-CORE ditetapkan sebagai root, loop terdeteksi dan satu port secara otomatis berubah menjadi discarding. Jaringan kembali normal. Pada log tercatat perubahan port menjadi discarding.

**Pembelajaran operasional:** RSTP atau MSTP harus selalu diaktifkan pada seluruh switch, root harus ditentukan secara eksplisit, dan STP tidak boleh dinonaktifkan untuk memperoleh kecepatan sesaat.

> **Aspek operasional STP:** Nilai priority default 0x8000 tidak boleh dibiarkan sama pada seluruh switch. Root dan secondary harus ditentukan. Dokumentasi mengenai switch yang berperan sebagai root pada setiap gedung perlu dipelihara. Topologi STP perlu diperiksa setiap bulan atau setelah terjadi perubahan kabel.


## Modul 5 — Link Aggregation: Penggabungan Tautan untuk Kapasitas dan Redundansi

**Tujuan:** Memahami bonding pada RouterOS, mode bonding, dan kompatibilitas dengan agregasi statis dari vendor lain.

**Ruang lingkup sesuai outline:** RouterOS bonding, bonding modes, dan compatibility with other static link aggregation.

### Konsep Utama dan Pilihan Mode

| Mode | Cara Kerja | Penggunaan yang Disarankan |
| --- | --- | --- |
| 802.3ad / LACP | Negosiasi dinamis, load balancing per flow | Standar, direkomendasikan untuk switch ke switch |
| balance-rr | Round robin per paket | Pengujian throughput, berpotensi menyebabkan out-of-order |
| active-backup | Satu aktif, satu standby | Redundansi sederhana, server ke switch |
| balance-xor | Hash berdasarkan MAC atau IP | Apabila switch lawan tidak mendukung LACP |
| broadcast | Pengiriman ke seluruh slave | Kasus khusus, jarang digunakan |

### Implementasi: LACP Antar Switch

```routeros
# Pada SW-CORE, gabungkan sfp-sfpplus1 dan sfp-sfpplus2 menuju SW-ACC1
/interface bonding add name=bond-to-acc1 mode=802.3ad slaves=sfp-sfpplus1,sfp-sfpplus2 transmit-hash-policy=layer-3-and-4 lacp-rate=fast

# Yang dimasukkan ke bridge adalah interface bonding, bukan port fisik
/interface bridge port add bridge=bridge1 interface=bond-to-acc1 pvid=1
/interface bridge vlan add bridge=bridge1 vlan-ids=10,20,30,99 tagged=bond-to-acc1

# Pada SW-ACC1, dilakukan konfigurasi yang sama dengan nama bond-to-core
/interface bonding add name=bond-to-core mode=802.3ad slaves=sfp-sfpplus1,sfp-sfpplus2 transmit-hash-policy=layer-3-and-4

# Pemeriksaan status LACP
/interface bonding monitor bond-to-acc1
```

### Implementasi: Bonding Server

```routeros
# Server dengan 2 NIC menuju switch
/interface bonding add name=bond-server mode=active-backup slaves=ether21,ether22 primary=ether21
/interface bridge port add bridge=bridge1 interface=bond-server pvid=10
# Apabila satu kabel terputus, server tetap beroperasi melalui NIC kedua
```

### Verifikasi

```routeros
/interface bonding print
/interface bonding monitor bond-to-acc1
# Kondisi yang diharapkan: status active, kedua slave aktif, dan traffic terbagi

/tool bandwidth-test 192.168.99.2 direction=both duration=10s
# Pengujian: satu kabel dilepas, ping tidak boleh terputus lebih dari 1 hingga 2 detik
```

### Perbandingan RouterOS v6 dan v7

| Aspek | RouterOS v6 | RouterOS v7 |
| --- | --- | --- |
| Mode Bonding | Mode 802.3ad, balance-rr, active-backup, balance-xor, broadcast tersedia | Mode yang sama tersedia, dengan negosiasi LACP yang lebih stabil |
| LACP | LACP tersedia dengan `lacp-rate` dan `transmit-hash-policy` | LACP tersedia dengan parameter yang sama, ditambah status LACP yang lebih rinci pada monitor |
| Transmit Hash Policy | Kebijakan `layer-2`, `layer-3-and-4` tersedia | Kebijakan yang sama tersedia, dengan distribusi per flow yang lebih konsisten pada CRS |
| Monitoring | Monitoring melalui `/interface bonding monitor` dengan informasi terbatas | Monitoring melalui perintah yang sama, dengan informasi `lacp-system-id` dan status slave yang lebih lengkap |

**Konfigurasi RouterOS v6**

```routeros
/interface bonding add name=bond-to-acc1 mode=802.3ad slaves=sfp-sfpplus1,sfp-sfpplus2 transmit-hash-policy=layer-3-and-4 lacp-rate=fast
/interface bridge port add bridge=bridge1 interface=bond-to-acc1 pvid=1
/interface bonding monitor bond-to-acc1
```

**Konfigurasi RouterOS v7**

```routeros
/interface bonding add name=bond-to-acc1 mode=802.3ad slaves=sfp-sfpplus1,sfp-sfpplus2 transmit-hash-policy=layer-3-and-4 lacp-rate=fast
/interface bridge port add bridge=bridge1 interface=bond-to-acc1 pvid=1 hw-offload=yes
/interface bonding monitor bond-to-acc1
/interface bridge vlan add bridge=bridge1 vlan-ids=10,20,30,99 tagged=bond-to-acc1
```

**Catatan Migrasi:** Konfigurasi bonding pada v6 dapat dipindahkan ke v7 tanpa perubahan sintaks yang signifikan. Yang perlu diperhatikan adalah port fisik tidak dimasukkan ke bridge, yang dimasukkan adalah interface bonding. Kedua sisi harus menggunakan mode yang sama, yaitu LACP dengan LACP atau static dengan static.

### Studi Kasus: Uplink 1 Gbps Tidak Mencukupi untuk 60 PC

Laboratorium komputer dengan 60 PC sering mengalami kelambatan saat seluruh PC mengunduh materi secara bersamaan. Uplink 1 Gbps dari access menuju core menjadi bottleneck. Administrator menggabungkan 2x 1 Gbps menjadi bond 2 Gbps dengan LACP. Hasilnya, throughput meningkat dan saat satu kabel dilepas untuk pemeliharaan, jaringan tetap beroperasi.

**Pembelajaran operasional:** Bonding tidak hanya menambah bandwidth, tetapi juga menyediakan redundansi. Pengujian failover perlu dilakukan dengan melepas satu kabel secara sengaja pada jam pemeliharaan.

> **Catatan bonding:** Port fisik yang telah menjadi slave tidak dimasukkan ke bridge. Yang dimasukkan ke bridge adalah interface bonding. Kedua sisi harus menggunakan mode yang sesuai, yaitu LACP dengan LACP atau static dengan static.


## Modul 6 — Port Isolation: Isolasi Port Tanpa Penambahan VLAN Berlebihan

**Tujuan:** Memahami bridge horizon pada RouterOS dan port isolation pada switch chip.

**Ruang lingkup sesuai outline:** RouterOS bridge horizon dan switch port isolation.

### Konsep Utama

- **Bridge Horizon:** Port dengan nilai horizon yang sama tidak dapat saling berkomunikasi pada Layer 2. Port dengan nilai horizon yang berbeda dapat saling berkomunikasi. Metode ini sederhana dan bekerja pada bridge.
- **Switch Port Isolation:** Fitur perangkat keras pada switch chip untuk isolasi yang lebih efisien, termasuk private VLAN sederhana.

### Implementasi: Bridge Horizon untuk WiFi Tamu dan Fasilitas Penginapan

```routeros
# Contoh: ether5 dan ether6 merupakan kamar atau klien tamu yang harus terisolasi
# ether7 merupakan gateway atau uplink yang harus dapat diakses seluruh klien
/interface bridge port set [find interface=ether5] horizon=10
/interface bridge port set [find interface=ether6] horizon=10
/interface bridge port set [find interface=ether7] horizon=1
# ether5 tidak dapat melakukan ping ke ether6, tetapi keduanya dapat melakukan ping ke ether7 (gateway)

# Contoh hotel: seluruh kamar terisolasi, tetapi seluruh kamar dapat menuju server
/interface bridge port set [find interface=ether10] horizon=20
/interface bridge port set [find interface=ether11] horizon=20
/interface bridge port set [find interface=ether12] horizon=20
/interface bridge port set [find interface=ether13] horizon=1
```

### Implementasi: Port Isolation pada Switch Chip CRS

```routeros
/interface ethernet switch port set ether5 isolation-profile=10
/interface ethernet switch port set ether6 isolation-profile=10
# Pemeriksaan lanjutan:
/interface ethernet switch port print detail
```

### Verifikasi

```routeros
/interface bridge port print
# Nilai horizon pada setiap port perlu diperiksa

# Pengujian dari PC kamar 1
/ping 192.168.20.11
# Hasil yang diharapkan adalah timeout apabila terisolasi, tetapi ping ke gateway harus berhasil
/ping 192.168.20.1
```

### Perbandingan RouterOS v6 dan v7

| Aspek | RouterOS v6 | RouterOS v7 |
| --- | --- | --- |
| Bridge Horizon | Horizon tersedia melalui `/interface bridge port set horizon`, isolasi pada CPU | Horizon tersedia dengan cara yang sama, dengan kinerja yang lebih baik pada bridge yang menggunakan hardware offload |
| Switch Port Isolation | Isolation melalui `/interface ethernet switch port` dengan `isolation-profile` pada CRS | Isolation melalui perintah yang sama, dengan dukungan private VLAN yang lebih konsisten pada CRS3xx |
| Konfigurasi | Isolasi dan horizon dikonfigurasi secara terpisah, perlu dokumentasi manual | Isolasi dan horizon dikonfigurasi secara terpisah, tetapi verifikasi melalui `/interface bridge port print` lebih rinci |
| Penggunaan | Cocok untuk isolasi antar klien dalam satu VLAN | Cocok untuk isolasi antar klien dalam satu VLAN, dengan rekomendasi yang sama |

**Konfigurasi RouterOS v6**

```routeros
/interface bridge port set [find interface=ether5] horizon=10
/interface bridge port set [find interface=ether6] horizon=10
/interface bridge port set [find interface=ether7] horizon=1
/interface bridge port print where horizon=10
```

**Konfigurasi RouterOS v7**

```routeros
/interface bridge port set [find interface=ether5] horizon=10 hw-offload=yes
/interface bridge port set [find interface=ether6] horizon=10 hw-offload=yes
/interface bridge port set [find interface=ether7] horizon=1 hw-offload=yes
/interface ethernet switch port set ether5 isolation-profile=10
/interface ethernet switch port set ether6 isolation-profile=10
```

**Catatan Migrasi:** Nilai horizon pada v6 dan v7 memiliki makna yang sama. Migrasi tidak memerlukan perubahan nilai, tetapi perlu dipastikan port uplink memiliki horizon yang berbeda dari port klien. Apabila terdapat printer bersama, printer tersebut harus ditempatkan pada horizon uplink.

### Studi Kasus: Apartemen dan Hotspot

Pengelola apartemen memasang satu switch untuk 12 kamar. Pada kondisi awal, antar kamar dapat saling melakukan ping dan mengakses file sharing tetangga. Keluhan mengenai privasi muncul. Administrator menerapkan horizon yang sama untuk seluruh port kamar dan horizon yang berbeda untuk port uplink menuju router. Setelah penerapan tersebut, antar kamar tidak dapat saling melakukan ping, tetapi seluruh kamar dapat mengakses internet. Solusi tersebut tidak memerlukan pembuatan 12 VLAN.

**Pembelajaran operasional:** Port isolation merupakan solusi yang cepat dan ringan untuk isolasi antar klien dalam satu VLAN yang sama.

> **Aspek operasional port isolation:** Nilai horizon untuk klien dan uplink perlu didokumentasikan. Apabila terdapat printer bersama, printer tersebut harus ditempatkan pada horizon uplink agar dapat diakses oleh seluruh kamar.


## Modul 7 — QoS: Penerapan Quality of Service pada Layer 2

**Tujuan:** Memahami QoS Layer 2 802.1p, prioritas pada bridge dan CRS, serta traffic shaping dengan queue dan pembatasan bandwidth pada switch chip.

**Ruang lingkup sesuai outline:** Layer 2 QoS (802.1p), RouterOS bridge filter priority, CRS priority configuration, traffic shaping, bandwidth limiting in bridge with queues, dan bandwidth limiting in switch chip.

### Konsep Utama

- **802.1p / PCP (Priority Code Point):** Tiga bit prioritas pada tag VLAN dengan nilai 0 hingga 7. Nilai 0 merupakan best effort, nilai 5 merupakan voice, dan nilai 6 hingga 7 merupakan network control.
- **Prioritas pada Bridge:** Prioritas dapat ditetapkan melalui bridge filter berdasarkan MAC, VLAN, protokol, atau port.
- **Traffic Shaping:** Pengaturan bandwidth per port atau per pengguna agar satu pengguna tidak menghabiskan seluruh bandwidth.
- **Bandwidth Limiting:** Pembatasan bandwidth pada bridge dengan queue atau pada switch chip.

### Implementasi: Prioritas Berdasarkan Jenis Traffic

```routeros
# Prioritas tinggi untuk VoIP dan video conference
/interface bridge filter add chain=forward action=set-priority new-priority=5 passthrough=yes comment="VoIP prioritas tinggi" mac-protocol=ip ip-protocol=udp dst-port=5060,10000-20000
/interface bridge filter add chain=forward action=set-priority new-priority=4 passthrough=yes comment="Video conference" dst-port=443,8801-8810

# Urutan filter perlu diperiksa, filter yang paling spesifik ditempatkan di atas
/interface bridge filter print

# Pembatasan bandwidth per port pada bridge
/interface bridge port set [find interface=ether5] ingress-rate=20M egress-rate=20M
/interface bridge port set [find interface=ether6] ingress-rate=10M egress-rate=10M
```

### Implementasi: Queue untuk Pengguna

```routeros
/queue simple add name=guest-limit target=192.168.20.0/24 max-limit=50M/50M queue=default/default
/queue simple add name=staff-prioritas target=192.168.10.0/24 max-limit=100M/100M priority=2/2

# Pada CRS, pembatasan pada switch chip lebih efisien
/interface ethernet switch port set ether5 ingress-rate=20M egress-rate=20M
```

### Verifikasi

```routeros
/interface bridge filter print stats
/queue simple print stats
/interface ethernet switch port print stats
# Pada saat video call berlangsung, perlu dipastikan packet drop pada queue kecil dan latensi stabil
```

### Perbandingan RouterOS v6 dan v7

| Aspek | RouterOS v6 | RouterOS v7 |
| --- | --- | --- |
| 802.1p dan PCP | Prioritas 802.1p melalui bridge filter `set-priority`, nilai 0 hingga 7 | Prioritas 802.1p melalui bridge filter dengan sintaks yang sama, dengan penerapan pada hardware offload yang lebih konsisten |
| Bridge Filter Priority | Bridge filter mendukung `set-priority` dan `passthrough` | Bridge filter mendukung sintaks yang sama, dengan urutan filter yang perlu diperhatikan |
| Traffic Shaping | Queue simple dan queue tree sebagai metode utama | Queue simple dan queue tree tetap didukung, ditambah queue type `cake` dan `fq-codel` untuk latensi yang lebih rendah |
| Bandwidth Limiting pada Bridge | Pembatasan melalui `ingress-rate` dan `egress-rate` pada bridge port | Pembatasan melalui parameter yang sama, dengan penerapan yang lebih akurat pada CRS |
| CRS Priority Configuration | Prioritas pada CRS melalui switch port dan switch rule | Prioritas pada CRS melalui switch port dan switch rule dengan sintaks yang sama, tetapi lebih stabil pada v7 |

**Konfigurasi RouterOS v6**

```routeros
/interface bridge filter add chain=forward action=set-priority new-priority=5 mac-protocol=ip ip-protocol=udp dst-port=5060 comment="VoIP"
/queue simple add name=guest-limit target=192.168.20.0/24 max-limit=50M/50M
/interface bridge port set [find interface=ether5] ingress-rate=20M egress-rate=20M
```

**Konfigurasi RouterOS v7**

```routeros
/interface bridge filter add chain=forward action=set-priority new-priority=5 mac-protocol=ip ip-protocol=udp dst-port=5060 comment="VoIP" passthrough=yes
/queue simple add name=guest-limit target=192.168.20.0/24 max-limit=50M/50M queue=cake/cake
/interface bridge port set [find interface=ether5] ingress-rate=20M egress-rate=20M
/interface ethernet switch port set ether5 ingress-rate=20M egress-rate=20M
```

**Catatan Migrasi:** Pada migrasi ke v7, queue type perlu ditinjau kembali. Queue type `default` pada v6 dapat diganti dengan `cake` pada v7 untuk traffic yang sensitif terhadap latensi seperti VoIP dan video conference. Pengujian QoS perlu dilakukan pada saat jaringan sibuk.

### Studi Kasus: Rapat Daring yang Selalu Terputus pada Jam Pengunduhan

Sebuah kantor dengan 30 staf mengalami gangguan pada rapat daring setiap pukul 10.00 karena pada jam tersebut terjadi pencadangan dan pengunduhan besar. Rapat daring selalu terputus. Administrator menerapkan prioritas 802.1p untuk traffic Zoom dan VoIP serta pembatasan 20 Mbps per port untuk pengguna yang melakukan pengunduhan. Setelah penerapan tersebut, rapat menjadi stabil meskipun pengunduhan tetap berlangsung.

**Pembelajaran operasional:** QoS tidak menambah bandwidth, tetapi mengatur traffic yang didahulukan saat terjadi perebutan bandwidth. Pengujian QoS perlu dilakukan pada saat jaringan sibuk, bukan pada saat jaringan sepi.

> **Aspek operasional QoS:** Disarankan untuk menggunakan tiga kelas, yaitu tinggi untuk voice dan video, normal untuk bisnis, dan rendah untuk pengunduhan dan tamu. Jumlah kelas yang terlalu banyak menyebabkan konfigurasi sulit dipelihara.


## Modul 8 — Layer 2 Security: Pengamanan Jaringan Layer 2

**Tujuan:** Menguasai IGMP snooping, DHCP snooping, loop protect, traffic storm control, Layer 2 firewall, BPDU guard, ARP modes, port security, 802.1X, dan switch security.

**Ruang lingkup sesuai outline:** IGMP snooping, DHCP snooping, loop protect, traffic storm control, Layer 2 firewall, RouterOS bridge filter features, switch access control list, BPDU guard, ARP modes, port security, 802.1X, dan switch security.

### 1. IGMP Snooping — Pengendalian Multicast

```routeros
/interface bridge set bridge1 igmp-snooping=yes igmp-version=3
# Multicast hanya diteruskan ke port yang meminta, bukan ke seluruh port
/interface bridge mdb print
```

### 2. DHCP Snooping — Pencegahan DHCP Palsu

```routeros
/interface bridge set bridge1 dhcp-snooping=yes add-dhcp-option82=yes
# Port uplink menuju DHCP server merupakan trusted
/interface bridge port set [find interface=sfp-sfpplus2] trusted=yes
# Port pengguna merupakan untrusted (default)
/interface bridge port set [find interface=ether1] trusted=no
```

### 3. Loop Protect dan Traffic Storm Control

```routeros
/interface ethernet switch set 0 loop-protect=yes loop-protect-send-interval=5s loop-protect-disable-time=5m
# Apabila loop terdeteksi, port dinonaktifkan selama 5 menit kemudian aktif kembali secara otomatis

# Traffic storm control pada CRS
/interface ethernet switch port set ether1 storm-rate=5%
# Alternatif pada bridge dengan filter untuk broadcast storm
```

### 4. Layer 2 Firewall dengan Bridge Filter

```routeros
# Pemblokiran DHCP dari port pengguna yang tidak seharusnya menjadi server
/interface bridge filter add chain=forward action=drop mac-protocol=ip ip-protocol=udp dst-port=67-68 in-interface=ether1 comment="Blokir DHCP server liar dari pengguna"

# Pemblokiran traffic antar VLAN yang tidak diperlukan pada Layer 2
/interface bridge filter add chain=forward action=drop src-mac-address=AA:BB:CC:00:00:00/FF:FF:FF:00:00:00 comment="Blokir MAC spoofing contoh"

# Pemeriksaan ACL pada switch chip
/interface ethernet switch rule print
```

### 5. BPDU Guard, Port Security, dan ARP Modes

```routeros
# BPDU guard: apabila terdapat switch yang tidak sah terhubung ke port access, port langsung dinonaktifkan
/interface bridge port set [find interface=ether1] bpdu-guard=yes edge=yes

# Port security: pembatasan jumlah MAC yang dipelajari per port
/interface ethernet switch port set ether1 learn-limit=2 learn-override=yes

# ARP mode pada bridge: reply-only untuk mencegah ARP spoofing
/interface bridge set bridge1 arp=reply-only
```

### 6. 802.1X — Autentikasi Port

```routeros
# Contoh 802.1X dengan RADIUS
/radius add service=dot1x address=192.168.99.10 secret=RadiusSecret123
/interface dot1x server add interface=ether1 auth-types=dot1x
# PC harus melakukan autentikasi terlebih dahulu sebelum memperoleh akses jaringan
# Untuk perangkat yang tidak mendukung 802.1X, digunakan MAC auth bypass (MAB)
```

### 7. Switch Security

Switch security mencakup penggantian password default, penonaktifan service yang tidak digunakan, pembatasan akses manajemen hanya melalui VLAN manajemen, dan pencatatan log secara berkala. Akses manajemen tidak boleh dibuka melalui VLAN pengguna.

```routeros
# Contoh penonaktifan service yang tidak digunakan
/ip service disable telnet,ftp,www,api
/ip service set ssh port=2200
# Pembatasan akses WinBox hanya dari VLAN manajemen dilakukan melalui firewall
```

### Verifikasi

```routeros
/interface bridge print detail
/interface bridge port print detail
/interface ethernet switch port print
/log print where topics~"dhcp" or topics~"loop" or topics~"dot1x"
# Log perlu diperiksa secara berkala untuk serangan DHCP dan loop
```

### Perbandingan RouterOS v6 dan v7

| Aspek | RouterOS v6 | RouterOS v7 |
| --- | --- | --- |
| IGMP Snooping | IGMP snooping tersedia dengan versi 2, dukungan v3 terbatas pada beberapa model | IGMP snooping versi 3 tersedia secara stabil, dengan MDB yang lebih akurat |
| DHCP Snooping | DHCP snooping tersedia mulai 6.41 dengan `add-dhcp-option82` | DHCP snooping tersedia dengan sintaks yang sama, dengan penanganan trusted dan untrusted port yang lebih konsisten |
| Loop Protect | Loop protect melalui `/interface ethernet switch set loop-protect` | Loop protect melalui perintah yang sama, dengan interval dan disable-time yang lebih fleksibel |
| Traffic Storm Control | Traffic storm control melalui switch port `storm-rate` | Traffic storm control melalui perintah yang sama, dengan penerapan pada hardware yang lebih konsisten |
| BPDU Guard | BPDU guard tersedia mulai 6.45 ke atas | BPDU guard tersedia secara stabil pada seluruh port access |
| 802.1X | 802.1X melalui `/interface dot1x` dengan integrasi RADIUS terbatas | 802.1X melalui `/interface dot1x server` dengan integrasi RADIUS dan MAC auth bypass yang lebih lengkap |
| ARP Modes | ARP mode `reply-only` pada bridge | ARP mode `reply-only` pada bridge dengan sintaks yang sama |

**Konfigurasi RouterOS v6**

```routeros
/interface bridge set bridge1 igmp-snooping=yes dhcp-snooping=yes
/interface bridge port set [find interface=sfp-sfpplus2] trusted=yes
/interface ethernet switch set 0 loop-protect=yes
/interface bridge port set [find interface=ether1] bpdu-guard=yes edge=yes
```

**Konfigurasi RouterOS v7**

```routeros
/interface bridge set bridge1 igmp-snooping=yes igmp-version=3 dhcp-snooping=yes add-dhcp-option82=yes arp=reply-only
/interface bridge port set [find interface=sfp-sfpplus2] trusted=yes
/interface bridge port set [find interface=ether1] trusted=no bpdu-guard=yes edge=yes
/interface ethernet switch set 0 loop-protect=yes loop-protect-send-interval=5s loop-protect-disable-time=5m
/radius add service=dot1x address=192.168.99.10 secret=RadiusSecret123
/interface dot1x server add interface=ether1 auth-types=dot1x
```

**Catatan Migrasi:** Fitur keamanan Layer 2 yang pada v6 belum diaktifkan karena keterbatasan model perlu diaktifkan pada v7 setelah memastikan model perangkat mendukung. DHCP snooping, BPDU guard, dan loop protect perlu diaktifkan pada seluruh port access sejak awal setelah migrasi.

### Studi Kasus: Jaringan Institusi Pendidikan yang Tidak Dapat Digunakan Akibat DHCP Palsu dan Loop

Dua insiden terjadi dalam satu minggu. Pertama, seorang siswa membawa router WiFi rumahan dan menghubungkannya ke jaringan sekolah. Router tersebut menjadi DHCP server liar sehingga seluruh PC memperoleh alamat IP yang salah dan tidak dapat mengakses internet. Kedua, kabel patch yang salah terhubung menyebabkan loop. Administrator mengaktifkan DHCP snooping dengan hanya port uplink yang bersifat trusted, BPDU guard pada seluruh port access, loop protect, dan traffic storm control. Setelah penerapan tersebut, saat terdapat router liar, port secara otomatis diblokir dan log mencatat kejadian tersebut. Saat terjadi loop, port dinonaktifkan selama 5 menit dan administrator memperoleh notifikasi.

**Pembelajaran operasional:** Keamanan Layer 2 harus diaktifkan sebelum insiden terjadi, bukan setelah insiden. DHCP snooping, BPDU guard, dan loop protect perlu diaktifkan pada seluruh port access sejak awal.


## Modul 9 — PoE: Penyaluran Daya Melalui Kabel Jaringan

**Tujuan:** Memahami mode PoE pada RouterOS, prioritas PoE, kompatibilitas PoE, dan monitoring PoE.

**Ruang lingkup sesuai outline:** RouterOS PoE modes and compatibility, RouterOS PoE priority settings, dan RouterOS PoE monitoring.

### Konsep Utama

- **PoE-out:** Switch memberikan daya kepada perangkat. Mode yang tersedia adalah auto, forced-on, dan off.
- **PoE-in:** Switch menerima daya dari PoE injector atau switch lain.
- **Prioritas PoE:** Apabila total daya tidak mencukupi, perangkat dengan prioritas rendah akan dinonaktifkan terlebih dahulu.
- **Kompatibilitas:** Terdapat PoE pasif 24V dan PoE aktif 802.3af, 802.3at, dan 802.3bt 48V. Ketidaksesuaian voltase dapat merusak perangkat.

### Implementasi

```routeros
# Access point pada ether5, kamera pada ether6 dan ether7
/interface ethernet poe set ether5 poe-out=auto-on poe-priority=10 poe-lldp-enabled=yes
/interface ethernet poe set ether6 poe-out=auto-on poe-priority=8
/interface ethernet poe set ether7 poe-out=auto-on poe-priority=8
/interface ethernet poe set ether8 poe-out=off

# Pengaturan forced-on perlu dilakukan dengan hati-hati
/interface ethernet poe set ether5 poe-out=forced-on
# Perhatian: forced-on akan memberikan daya meskipun perangkat tidak terdeteksi

# Pemeriksaan status PoE
/interface ethernet poe monitor ether5
/interface ethernet poe print
```

### Verifikasi dan Operasional

```routeros
/interface ethernet poe monitor 0
# Perlu diperhatikan: poe-out-voltage, poe-out-current, dan poe-out-power

# Pemeriksaan total budget
/system health print
# Apabila total daya mendekati batas PSU, prioritas perangkat yang tidak penting perlu diturunkan

# Contoh penjadwalan: access point tamu dinonaktifkan pada malam hari untuk efisiensi daya
/system scheduler add name=poe-malam-off on-event="/interface ethernet poe set ether5 poe-out=off" start-time=23:00:00 interval=1d
```

### Perbandingan RouterOS v6 dan v7

| Aspek | RouterOS v6 | RouterOS v7 |
| --- | --- | --- |
| PoE Modes | Mode `auto-on`, `forced-on`, `off` melalui `/interface ethernet poe` | Mode yang sama tersedia, dengan penanganan deteksi perangkat yang lebih andal |
| PoE Priority | Prioritas melalui `poe-priority`, nilai 0 hingga 10 | Prioritas melalui parameter yang sama, dengan perilaku prioritas yang lebih konsisten saat total daya tidak mencukupi |
| PoE Monitoring | Monitoring melalui `/interface ethernet poe monitor` dengan voltase dan arus | Monitoring melalui perintah yang sama, ditambah informasi `poe-out-power` dan status LLDP yang lebih rinci |
| PoE LLDP | LLDP untuk PoE tersedia secara terbatas | LLDP untuk PoE tersedia melalui `poe-lldp-enabled=yes` dan lebih stabil |
| PoE Firmware | Upgrade firmware PoE melalui paket RouterOS | Upgrade firmware PoE melalui paket RouterOS, dengan verifikasi versi firmware pada monitor |

**Konfigurasi RouterOS v6**

```routeros
/interface ethernet poe set ether5 poe-out=auto-on poe-priority=10
/interface ethernet poe set ether6 poe-out=auto-on poe-priority=8
/interface ethernet poe monitor ether5
```

**Konfigurasi RouterOS v7**

```routeros
/interface ethernet poe set ether5 poe-out=auto-on poe-priority=10 poe-lldp-enabled=yes
/interface ethernet poe set ether6 poe-out=auto-on poe-priority=8 poe-lldp-enabled=yes
/interface ethernet poe monitor ether5
/interface ethernet poe print detail
```

**Catatan Migrasi:** Mode `forced-on` pada v6 dan v7 memiliki risiko yang sama, yaitu memberikan daya meskipun perangkat tidak terdeteksi. Mode tersebut tidak boleh digunakan kecuali perangkat telah dipastikan kompatibel. Setelah migrasi, perlu dilakukan pemeriksaan ulang total budget PoE dan prioritas.

### Studi Kasus: Hotel dengan 24 Access Point dan 16 Kamera

Sebuah hotel memasang dua switch PoE untuk 24 access point dan 16 kamera. Total kebutuhan daya mendekati batas PSU. Administrator mengatur prioritas sebagai berikut: kamera dan access point lobi memiliki prioritas 10, access point koridor memiliki prioritas 5, dan access point gudang memiliki prioritas 1. Saat terjadi lonjakan daya, access point gudang yang dinonaktifkan terlebih dahulu, bukan kamera. Monitoring harian menunjukkan satu kamera mengalami peningkatan konsumsi daya yang tidak wajar. Setelah dilakukan pemeriksaan, diketahui kabel kamera tersebut lembap dan hampir mengalami short. Setelah kabel diganti, konsumsi daya kembali normal.

**Pembelajaran operasional:** Monitoring PoE perlu dilakukan setiap minggu dengan memeriksa daya per port. Mode forced-on tidak boleh digunakan kecuali perangkat telah dipastikan kompatibel.

> **Aspek operasional PoE:** Kabel PoE perlu diberi label dengan jelas. Perangkat non-PoE tidak boleh dihubungkan ke port forced-on. Cadangan PSU perlu disiapkan apabila switch PoE merupakan perangkat kritis.

## Modul 10 — Tools: Diagnostik dan Pemantauan Layer 2

**Tujuan:** Menguasai Layer 2 diagnostic tools dan port mirroring.

**Ruang lingkup sesuai outline:** Layer 2 diagnostic tools dan port mirroring.

### Tools yang Perlu Dikuasai

| Tool | Fungsi | Perintah Contoh |
| --- | --- | --- |
| Torch | Menampilkan traffic real-time per interface | `/tool torch ether1` |
| Packet Sniffer | Menangkap paket untuk analisis | `/tool sniffer start interface=ether1` |
| MAC Scan / Neighbor | Mencari perangkat MikroTik pada Layer 2 | `/tool mac-scan, /ip neighbor print` |
| Ping / Bandwidth Test | Menguji konektivitas dan throughput | `/tool ping, /tool bandwidth-test` |
| Port Mirroring | Menduplikasi traffic ke port monitoring | `/interface ethernet switch set mirror-source, mirror-target` |
| Cable Test | Menguji kabel dan panjang kabel | `/interface ethernet cable-test ether1` |

### Implementasi: Port Mirroring untuk Troubleshooting

```routeros
# Mirror traffic dari ether1 (PC bermasalah) ke ether24 (laptop Wireshark)
/interface ethernet switch set 0 mirror-source=ether1 mirror-target=ether24 mirror-egress=yes mirror-ingress=yes

# Pada laptop, Wireshark digunakan untuk capture pada interface yang terhubung ke ether24
# Setelah selesai, mirroring dinonaktifkan agar tidak membebani switch
/interface ethernet switch set 0 mirror-source=none mirror-target=none

# Alternatif: Sniffer pada RouterOS kemudian file pcap diunduh
/tool sniffer set filter-interface=ether1 filter-stream=yes
/tool sniffer start
/tool sniffer stop
/tool sniffer packet print
# Berkas dapat diunduh melalui menu Files untuk dibuka pada Wireshark
```

### Implementasi: Diagnostik Kabel dan Interface

```routeros
/interface ethernet cable-test ether1
/interface ethernet monitor ether1
/interface bridge host print
/tool mac-scan interface=bridge1
/ip neighbor print detail
```

### Perbandingan RouterOS v6 dan v7

| Aspek | RouterOS v6 | RouterOS v7 |
| --- | --- | --- |
| Torch | Torch tersedia dengan filter interface, MAC, dan protokol | Torch tersedia dengan filter yang sama, ditambah tampilan yang lebih rinci dan stabil |
| Packet Sniffer | Sniffer tersedia dengan filter interface dan streaming ke Wireshark | Sniffer tersedia dengan sintaks yang sama, ditambah `filter-stream` dan penyimpanan berkas pcap yang lebih andal |
| MAC Scan dan Neighbor | MAC scan melalui `/tool mac-scan` dan neighbor melalui `/ip neighbor print` | Perintah yang sama tersedia, dengan deteksi perangkat MikroTik yang lebih cepat |
| Cable Test | Cable test tersedia pada sebagian model dengan informasi pair | Cable test tersedia dengan sintaks yang sama, dengan informasi panjang kabel dan status pair yang lebih lengkap pada CRS3xx |
| Port Mirroring | Port mirroring melalui `/interface ethernet switch set mirror-source` dan `mirror-target` | Port mirroring melalui perintah yang sama, dengan dukungan egress dan ingress yang lebih konsisten |

**Konfigurasi RouterOS v6**

```routeros
/tool torch ether1
/tool sniffer set filter-interface=ether1
/tool sniffer start
/interface ethernet switch set 0 mirror-source=ether1 mirror-target=ether24
/interface ethernet cable-test ether1
```

**Konfigurasi RouterOS v7**

```routeros
/tool torch ether1 src-address=192.168.10.0/24
/tool sniffer set filter-interface=ether1 filter-stream=yes
/tool sniffer start
/interface ethernet switch set 0 mirror-source=ether1 mirror-target=ether24 mirror-egress=yes mirror-ingress=yes
/interface ethernet cable-test ether1
/interface ethernet monitor ether1
```

**Catatan Migrasi:** Sniffer dan mirroring pada v6 dan v7 memiliki sintaks yang hampir sama. Setelah migrasi, perlu dipastikan mirroring dinonaktifkan setelah troubleshooting selesai agar tidak membebani switch. Berkas pcap pada v7 dapat diunduh melalui menu Files untuk dibuka pada Wireshark.

### Studi Kasus: Satu PC Tidak Dapat Mengakses Server, Sedangkan PC Lain Dapat Mengakses

Pada sebuah laboratorium, satu PC tidak dapat melakukan ping ke server, sedangkan PC lain pada VLAN yang sama dapat melakukan ping. Administrator tidak langsung mengganti switch. Langkah yang dilakukan adalah sebagai berikut:
1. Pemeriksaan kabel dengan cable-test menunjukkan pair 3 open.
2. Kabel diganti, tetapi permasalahan tetap terjadi.
3. Pemeriksaan bridge host menunjukkan MAC PC tidak muncul.
4. Pemeriksaan port menunjukkan port dalam kondisi disabled akibat BPDU guard setelah teknisi secara tidak sengaja menghubungkan switch kecil.
5. Port diaktifkan kembali setelah kabel switch kecil dilepas, dan PC kembali normal.

**Pembelajaran operasional:** Urutan diagnostik Layer 2 adalah lapisan fisik, port, MAC table, VLAN, kemudian Layer 3. Diagnostik tidak boleh langsung menuju Layer 3 sebelum Layer 2 dipastikan berfungsi.

> **Aspek operasional Tools:** Pemeriksaan neighbor, host table, log, dan cable-test untuk port yang sering bermasalah perlu dilakukan secara berkala. Hasil sniffer hanya disimpan pada saat troubleshooting dan tidak dibiarkan berjalan secara terus-menerus.


## Modul 11 — SwOS: Switching dengan Sistem Operasi Ringan

**Tujuan:** Memahami SwOS, kompatibilitas dual-boot pada RouterBOARD, instalasi SwOS, manajemen SwOS, dan konfigurasi fitur Layer 2 pada SwOS.

**Ruang lingkup sesuai outline:** Introduction to SwOS, RouterBOARD dual-boot compatibility, installing SwOS, managing SwOS, dan configuration of Layer 2 features with SwOS, yaitu VLANs, (R)STP, port trunking, QoS, dan Layer 2 security.

### Kapan Menggunakan RouterOS dan SwOS

| Aspek | RouterOS | SwOS |
| --- | --- | --- |
| Fungsi utama | Router, switch, dan firewall lengkap | Switch murni Layer 2 |
| Manajemen | WinBox, CLI, API, WebFig | Web browser |
| Fitur Layer 2 | Sangat lengkap dan fleksibel | Lengkap untuk switching dasar hingga menengah |
| Throughput | Bergantung pada CPU untuk fitur Layer 3 | Hardware switching penuh |
| Kesesuaian | Core, gateway, dan switch yang memerlukan firewall | Access switch dan switch distribusi sederhana |

### Implementasi: Dual-Boot dan Instalasi SwOS

```routeros
# Pemeriksaan mode boot saat ini pada RouterOS
/system routerboard print
# Boot ke SwOS:
/system routerboard settings set boot-device=swos
/system reboot

# Alternatif: tahan tombol reset saat boot untuk memilih OS
# Setelah boot ke SwOS, akses melalui browser:
# Default IP SwOS: 192.168.88.1, user admin, tanpa password
# Password dan IP manajemen perlu segera diubah
```

### Konfigurasi Layer 2 pada SwOS melalui Web

- **1. System:** Mengubah identity, IP manajemen pada VLAN 99, password, dan upgrade firmware SwOS.
- **2. Link:** Mengatur speed, duplex, PoE, dan enable atau disable port. Port yang tidak digunakan perlu dinonaktifkan.
- **3. VLAN:** Membuat VLAN 10, 20, 30, dan 99. Port access diatur dengan PVID dan port trunk diatur dengan tagged. Konsepnya sama dengan bridge VLAN filtering, tetapi melalui GUI.
- **4. (R)STP:** Mengaktifkan RSTP, mengatur priority agar core tetap menjadi root, dan memeriksa port yang blocking.
- **5. Port Trunking:** Menggabungkan dua port uplink dengan LACP pada menu Link Aggregation.
- **6. QoS:** Mengatur prioritas 802.1p dan rate limit per port pada menu QoS.
- **7. Layer 2 Security:** Mengaktifkan port isolation, traffic storm control, IGMP snooping, dan ACL sederhana pada menu Forwarding dan Security.

### Perbandingan RouterOS v6 dan v7 serta SwOS 1.x dan SwOS 2.x

| Aspek | RouterOS v6 dan SwOS 1.x | RouterOS v7 dan SwOS 2.x |
| --- | --- | --- |
| Sistem Operasi | RouterOS v6 untuk routing dan switching, SwOS 1.x untuk switching pada CRS1xx dan CRS2xx | RouterOS v7 untuk routing dan switching, SwOS 2.x untuk switching pada CRS3xx dan CRS5xx |
| Dual-Boot | Dual-boot melalui `/system routerboard settings set boot-device`, dengan kompatibilitas terbatas pada model lama | Dual-boot melalui perintah yang sama, dengan kompatibilitas yang lebih baik pada CRS3xx dan CRS5xx |
| VLAN | VLAN pada SwOS 1.x melalui menu VLAN dengan fitur dasar | VLAN pada SwOS 2.x melalui menu VLAN dengan fitur yang sama, ditambah penanganan trunk dan PVID yang lebih konsisten |
| (R)STP | RSTP pada SwOS 1.x dengan prioritas dasar | RSTP pada SwOS 2.x dengan prioritas dan status port yang lebih rinci |
| Port Trunking | Port trunking melalui LACP dengan fitur dasar | Port trunking melalui LACP dengan negosiasi yang lebih stabil |
| QoS dan Layer 2 Security | QoS dan security dasar melalui menu QoS dan Forwarding | QoS dan security pada SwOS 2.x dengan traffic storm control, IGMP snooping, dan port isolation yang lebih lengkap |
| Manajemen | Manajemen melalui web browser pada 192.168.88.1 | Manajemen melalui web browser pada alamat yang sama, dengan tampilan dan export konfigurasi yang lebih baik |

**Konfigurasi RouterOS v6 — Dual-Boot ke SwOS**

```routeros
/system routerboard print
/system routerboard settings set boot-device=swos
/system reboot
# Setelah boot ke SwOS, akses melalui browser 192.168.88.1
# Konfigurasi VLAN, RSTP, dan trunking dilakukan melalui menu web SwOS 1.x
```

**Konfigurasi RouterOS v7 — Dual-Boot ke SwOS**

```routeros
/system routerboard print
/system routerboard settings set boot-device=swos
/system reboot
# Setelah boot ke SwOS 2.x, akses melalui browser 192.168.88.1
# Ubah password dan IP manajemen pada VLAN 99
# Konfigurasi VLAN, (R)STP, port trunking, QoS, dan Layer 2 security melalui menu web SwOS 2.x
# Export konfigurasi secara manual setelah setiap perubahan
```

**Catatan Migrasi:** Migrasi dari RouterOS v6 ke v7 tidak mengubah cara dual-boot, tetapi SwOS yang digunakan perlu disesuaikan dengan model perangkat. CRS1xx dan CRS2xx menggunakan SwOS 1.x, sedangkan CRS3xx dan CRS5xx menggunakan SwOS 2.x. Pencadangan konfigurasi RouterOS perlu dilakukan sebelum berpindah ke SwOS, karena SwOS tidak memiliki pencadangan otomatis.

### Studi Kasus: Migrasi Dua Switch Access dari RouterOS ke SwOS

Sebuah institusi pendidikan memiliki dua switch access yang hanya digunakan untuk switching dan tidak memerlukan routing atau firewall. Administrator memutuskan untuk melakukan migrasi ke SwOS agar manajemen lebih ringan dan throughput lebih konsisten. Langkah migrasi yang dilakukan adalah sebagai berikut:
1. Pencadangan konfigurasi RouterOS.
2. Pencatatan VLAN, RSTP priority, dan port mapping.
3. Boot ke SwOS dan konfigurasi VLAN dan RSTP yang sama melalui web.
4. Pengujian ping antar VLAN dan failover RSTP.
5. Setelah satu minggu stabil, SwOS dijadikan boot default.

Hasilnya, CPU switch menurun dari 40 persen menjadi 5 persen, dan staf tata usaha dapat memeriksa status port melalui web tanpa memerlukan WinBox.

**Pembelajaran operasional:** Pemilihan sistem operasi perlu disesuaikan dengan kebutuhan. Apabila hanya diperlukan switching, SwOS sering kali lebih sederhana dan stabil.

> **Aspek operasional SwOS:** SwOS tidak memiliki pencadangan otomatis seperti RouterOS. Konfigurasi perlu diexport secara manual setelah setiap perubahan yang signifikan. Berkas pencadangan disimpan pada folder dokumentasi.


## Lampiran A — Template Konfigurasi Core Lengkap

Template ini menggabungkan Modul 1 hingga Modul 8 dalam satu konfigurasi core. Template ini digunakan sebagai titik awal dan perlu disesuaikan dengan kebutuhan.

```routeros
# === SW-CORE - TEMPLATE LENGKAP ===
/system identity set name=SW-CORE
/interface bridge add name=bridge1 protocol-mode=rstp priority=0x1000 vlan-filtering=no igmp-snooping=yes dhcp-snooping=yes arp=reply-only

# Port access
/interface bridge port add bridge=bridge1 interface=ether1 pvid=10 horizon=1 edge=yes bpdu-guard=yes trusted=no
/interface bridge port add bridge=bridge1 interface=ether2 pvid=10 horizon=1 edge=yes bpdu-guard=yes trusted=no
/interface bridge port add bridge=bridge1 interface=ether3 pvid=20 horizon=10 edge=yes bpdu-guard=yes trusted=no
/interface bridge port add bridge=bridge1 interface=ether4 pvid=20 horizon=10 edge=yes bpdu-guard=yes trusted=no

# Bonding uplink menuju ACC1
/interface bonding add name=bond-acc1 mode=802.3ad slaves=sfp-sfpplus1,sfp-sfpplus2 transmit-hash-policy=layer-3-and-4
/interface bridge port add bridge=bridge1 interface=bond-acc1 pvid=1

# VLAN table
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=bond-acc1 untagged=ether1,ether2
/interface bridge vlan add bridge=bridge1 vlan-ids=20 tagged=bond-acc1 untagged=ether3,ether4
/interface bridge vlan add bridge=bridge1 vlan-ids=99 tagged=bond-acc1 untagged=bridge1
/interface bridge set bridge1 vlan-filtering=yes

# Inter-VLAN
/interface vlan add name=vlan10 interface=bridge1 vlan-id=10
/interface vlan add name=vlan20 interface=bridge1 vlan-id=20
/interface vlan add name=vlan99 interface=bridge1 vlan-id=99
/ip address add address=192.168.10.1/24 interface=vlan10
/ip address add address=192.168.20.1/24 interface=vlan20
/ip address add address=192.168.99.1/24 interface=vlan99

# QoS dan Security
/interface bridge filter add chain=forward action=set-priority new-priority=5 mac-protocol=ip ip-protocol=udp dst-port=5060 comment="VoIP"
/interface ethernet switch set 0 loop-protect=yes
/system backup save name=core-final
/export file=core-final
```

## Lampiran B — Troubleshooting Cepat per Gejala

| Gejala | Kemungkinan Penyebab | Langkah Pertama |
| --- | --- | --- |
| Tidak dapat melakukan ping sama sekali | Kabel, port disabled, belum masuk bridge | Pemeriksaan cable-test, /interface print, /interface bridge host print |
| Ping kecil berhasil, ping besar gagal | MTU tidak konsisten | /tool ping size=8972 do-not-fragment, pemeriksaan L2MTU |
| Dapat melakukan ping dalam satu VLAN, tidak dapat antar VLAN | VLAN filtering, routing, firewall | Pemeriksaan /interface bridge vlan print dan /ip route print |
| Jaringan tiba-tiba lambat secara menyeluruh | Loop, broadcast storm | Pemeriksaan /interface bridge port print, log loop, pelepasan kabel yang baru dipasang |
| Memperoleh alamat IP yang salah | DHCP server liar | Pemeriksaan DHCP snooping, /ip dhcp-server lease print |
| PC baru tidak memperoleh alamat IP | Port security, BPDU guard, DHCP snooping | Pemeriksaan /log print dan /interface bridge port print |
| Video call terputus-putus | QoS belum diatur, bandwidth habis | Pemeriksaan /queue simple print stats dan /interface monitor |
| PoE tidak menyala | Mode PoE, daya tidak mencukupi, kabel | /interface ethernet poe monitor, pemeriksaan prioritas |
