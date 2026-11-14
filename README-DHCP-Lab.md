# DHCP Server Lab --- Debian 13

> Praktik membangun DHCP Server menggunakan `isc-dhcp-server` pada
> VMware Workstation, dengan satu server dan satu client pada LAN
> Segment terisolasi.

## 1. Informasi Lingkungan

  Komponen         Detail
  ---------------- --------------------
  Virtualization   VMware Workstation
  Server OS        Debian 13
  Client OS        Debian 13
  DHCP Service     `isc-dhcp-server`
  LAN Segment      `DHCP-LAB`
  Subnet           `192.168.50.0/24`

### Topologi

  ---------------------------------------------------------------------------------
  Hostname       Interface      Network        IP Address            Peran
  -------------- -------------- -------------- --------------------- --------------
  `netlab`       `ens33`        NAT / VMnet8   `192.168.205.12/24`   Interface
                                                                     manajemen /
                                                                     jaringan NAT

  `netlab`       `ens37`        `DHCP-LAB`     `192.168.50.1/24`     DHCP Server

  `clt-02`       `ens33`        `DHCP-LAB`     `192.168.50.50/24`    DHCP Client
                                                                     dengan
                                                                     reservation
  ---------------------------------------------------------------------------------

**Catatan topologi:** `ens33` pada `netlab` tetap menggunakan NAT.
`ens37` menjadi interface khusus LAN Segment `DHCP-LAB`. VM `clt-02`
terhubung ke LAN Segment tersebut.

## 2. Persiapan

1.  Buat atau gunakan dua VM Debian 13: `netlab` dan `clt-02`.
2.  Pada VMware, hubungkan adapter kedua VM ke LAN Segment bernama
    `DHCP-LAB`.
3.  Pastikan interface server untuk LAN Segment adalah `ens37`.
4.  Pastikan interface client yang terhubung ke LAN Segment adalah
    `ens33`.
5.  Ambil salinan konfigurasi sebelum melakukan perubahan.

## 3. Konfigurasi DHCP Server (`netlab`)

### Instalasi paket

``` bash
sudo apt update
sudo apt install isc-dhcp-server
```

### Konfigurasi interface layanan

Edit `/etc/default/isc-dhcp-server`:

``` ini
INTERFACESv4="ens37"
INTERFACESv6=""
```

DHCP server dibatasi agar hanya melayani interface LAN Segment `ens37`.

### Konfigurasi alamat interface server

Tambahkan konfigurasi interface LAN pada `/etc/network/interfaces`:

``` ini
auto ens37
iface ens37 inet static
    address 192.168.50.1
    netmask 255.255.255.0
```

Pastikan interface `ens33` yang terhubung ke NAT tetap menggunakan
konfigurasi sebelumnya.

### Konfigurasi DHCP pool

Edit `/etc/dhcp/dhcpd.conf`:

``` conf
authoritative;

default-lease-time 600;
max-lease-time 7200;

subnet 192.168.50.0 netmask 255.255.255.0 {
    range 192.168.50.100 192.168.50.150;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.168.50.255;
    option routers 192.168.50.1;
    option domain-name "patir.local";
    option domain-name-servers 192.168.50.1, 1.1.1.1;
}
```

Keterangan:

-   Network: `192.168.50.0/24`
-   DHCP pool: `192.168.50.100–192.168.50.150`
-   Gateway yang diumumkan: `192.168.50.1`
-   Domain: `patir.local`
-   Default lease: 600 detik
-   Maximum lease: 7200 detik

> **Catatan:** konfigurasi DHCP mengumumkan `192.168.50.1` sebagai
> gateway dan DNS. Namun, routing/NAT untuk client serta layanan DNS
> pada alamat tersebut belum menjadi bagian dari pengujian lab ini.
> Jangan menganggap akses internet atau resolusi nama sudah teruji.

### Validasi dan menjalankan layanan

``` bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo systemctl restart isc-dhcp-server
systemctl is-active isc-dhcp-server
```

Hasil yang diharapkan dari perintah terakhir:

``` text
active
```

## 4. Konfigurasi DHCP Client (`clt-02`)

Atur interface `ens33` pada `/etc/network/interfaces` menjadi DHCP:

``` ini
auto ens33
iface ens33 inet dhcp
```

Client memperoleh alamat dari DHCP server melalui LAN Segment
`DHCP-LAB`.

Periksa alamat interface:

``` bash
ip -4 addr show ens33
```

Pada pengujian awal, client memperoleh alamat dinamis
`192.168.50.100/24`.

## 5. DHCP Reservation / MAC Binding

### Mengambil MAC Address client

Pada `clt-02`:

``` bash
ip link show ens33
```

MAC Address yang digunakan:

``` text
00:0c:29:62:a8:d4
```

### Menambahkan reservation pada server

Pada `netlab`, tambahkan blok berikut di bagian bawah
`/etc/dhcp/dhcpd.conf`:

``` conf
host clt-02 {
    hardware ethernet 00:0c:29:62:a8:d4;
    fixed-address 192.168.50.50;
}
```

Alamat `192.168.50.50` berada di luar dynamic pool `.100–.150`, sehingga
tidak bertabrakan dengan rentang alamat dinamis yang dibagikan server.

Validasi dan restart DHCP:

``` bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo systemctl restart isc-dhcp-server
```

### Memperbarui alamat client

Pada console VMware `clt-02`, perbarui koneksi jaringan menggunakan:

``` bash
sudo systemctl restart networking
```

Kemudian periksa alamat:

``` bash
ip -4 addr show ens33
```

Hasil yang diharapkan:

``` text
192.168.50.50/24
```

## 6. Pengujian Konektivitas dan SSH

### Pengujian ping

Dari `clt-02`, uji koneksi ke interface LAN server:

``` bash
ping -c 4 192.168.50.1
```

Hasil pengujian: client berhasil menerima balasan dari `192.168.50.1`.

### Pengujian SSH

Dari `netlab`, hubungkan ke alamat reservation milik client:

``` bash
ssh patir@192.168.50.50
```

Verifikasi sesi SSH pada client:

``` bash
hostname
whoami
pwd
```

Hasil yang diharapkan:

-   Hostname: `clt-02`
-   User: `patir`
-   Home directory: `/home/patir`

**Catatan:** gunakan IP reservation terbaru (`192.168.50.50`), bukan
alamat lease dinamis lama (`192.168.50.100`).

## 7. Troubleshooting

  -----------------------------------------------------------------------
  Kendala                             Penanganan
  ----------------------------------- -----------------------------------
  DHCP service gagal aktif            Periksa interface pada
                                      `/etc/default/isc-dhcp-server` dan
                                      validasi `dhcpd.conf` dengan
                                      `dhcpd -t`.

  Client belum mendapat IP            Periksa adapter VMware, LAN
                                      Segment, status interface, dan
                                      konfigurasi DHCP client.

  `dhclient: command not found`       Paket `dhclient` tidak tersedia
                                      pada client; pembaruan koneksi
                                      dilakukan melalui service
                                      networking.

  SSH menolak dengan `publickey`      Periksa konfigurasi autentikasi SSH
                                      efektif, password akun, dan restart
                                      service SSH setelah perubahan.

  SSH ke IP lama gagal                Gunakan alamat reservation terbaru,
                                      yaitu `192.168.50.50`.

  `unable to resolve host clt-02`     Periksa pemetaan hostname lokal
                                      pada `/etc/hosts`.
  -----------------------------------------------------------------------

## 8. Dokumentasi Pengujian

Tambahkan screenshot hasil praktik pada bagian berikut.

### A. Konfigurasi jaringan VMware

`<!-- Sisipkan screenshot adapter netlab dan clt-02 -->`

### B. Konfigurasi DHCP Server

`<!-- Sisipkan screenshot /etc/default/isc-dhcp-server dan dhcpd.conf -->`

### C. Status layanan DHCP

`<!-- Sisipkan screenshot status isc-dhcp-server active -->`

### D. Client memperoleh IP dinamis

`<!-- Sisipkan screenshot IP awal 192.168.50.100 -->`

### E. DHCP Reservation

`<!-- Sisipkan screenshot konfigurasi host reservation dan IP 192.168.50.50 -->`

### F. Pengujian ping dan SSH

`<!-- Sisipkan screenshot ping dan sesi SSH berhasil -->`

## 9. Hasil Akhir

  Pengujian                                    Hasil
  -------------------------------------------- -------
  DHCP Server aktif pada `netlab`              PASS
  Client memperoleh IP dinamis                 PASS
  DHCP Reservation berdasarkan MAC Address     PASS
  Client memperoleh IP tetap `192.168.50.50`   PASS
  Ping client ke server                        PASS
  SSH dari server ke client                    PASS

## 10. Kesimpulan

Praktik ini berhasil membangun layanan DHCP menggunakan
`isc-dhcp-server` pada Debian 13. Server `netlab` membagikan alamat IPv4
kepada client `clt-02` melalui LAN Segment `DHCP-LAB`. DHCP Reservation
berdasarkan MAC Address juga berhasil diterapkan sehingga client
memperoleh alamat tetap `192.168.50.50`
