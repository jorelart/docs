---
title: Konfigurasi Juniper
parent: Juniper
---




## Konfigurasi Tunnel Mitra

Penggunaan tunnel untuk distribusi ke mitra ini masih ada karena mengingat belum adanya jalur Metro yang sampai ke lokasi mitra. 
Sebelum melakukan konfigurasi lebih lanjut, pastikan bahwa NOC sudah bisa mengakses kedua router, yaitu :

##### 1. Router Mitra Baru
Karena mitra dengan jalur tunnel ini tidak punya koneksi ke JSN sama sekali, maka cara satu-satunya untuk bisa mengakses router adalah dengan remote (Anydesk atau Teamviewer, dll). Untuk cara remote dengan Anydesk kita anggap sudah paham semua.

##### 2. Router Daerah (Cyber)
Buka Router Daerah (Cyber) via remote The Dude.
Jika kedua Router sudah terbuka, saatnya melakukan konfigurasi.



## Step by step Konfigurasi Tunnel

### 1. Membuat Profile PPP
Buka Menu `PPP` `Profile` kemudian buat profile baru dengan nama `"INET1"`, pada tab `Script` isi dengan script dibawah ini :


`On Up :`
```yaml
/interface l2tp-client disable JSN-1
/ip rou rule set [find comment="INET1"] src-address=$"local-address"
/ip route set [find comment="INET1"] gateway=$"remote-address"
/int ipip set JSN-IPIP-1 local-address=$"local-address"
/int l2tp-client en JSN-1
```
`On Down:`
```yaml
/interface l2tp-client disable JSN-1
```


### 2. Ganti nama interface
Buka menu `Interface` dan temukan `PPoE Client` yang terhubung ke `Indihome` ganti nama jadi `"INET1"`, pada tab `Dial Out` ganti profile dengan `"INET1"` yang berhasil dibuat tadi.

### 3. Setting DNS
Buka menu `ip` kemudian `DNS` dan isikan 
- Server1 : `103.80.80.231`
- Server2 : `103.80.80.232`
- Centang `Allow Remote Request : [x]`
- Kemudian `OK`
```sh
### ...setting DNS

/ip dns
set allow-remote-requests=yes cache-max-ttl=30m cache-size=20480KiB \
    max-concurrent-queries=1000 max-concurrent-tcp-sessions=200 servers=\
    103.80.80.231,103.80.80.232
```

### 4. Buat Scheduler baru
- Scheduler `"INET1"`
yang berfungsi memantau interface jalur tunel INET1 supaya selalu mendapatkan Ip Publik.

```yaml
# ...atau paste script dibawah ini :

/system scheduler
add interval=5s name=INET1 on-event=":if ( [/ip address get [find interface=\"\
    INET1\"] address] in 10.0.0.0/8) do={ \r\
    \n/int pppoe-client disable \"INET1\";\r\
    \ndelay 10\r\
    \n/int pppoe-client en \"INET1\";\r\
    \n}\r\
    \n" policy=reboot,read,write,test,password,sniff,sensitive,romon \
    start-time=startup
```
- Scheduler `"DNS-Health"`
Berfungsi mengembalikan settingan DNS sesuai SOP secara otomatis jika kemungkinan Mitra merubahnya.


```yaml
#... atau paste script dibawah ini :

/system scheduler
add interval=1m name=DNS-Health on-event="{\r\
    \n:local dns1 0; \r\
    \n:local dns2 0; \r\
    \n:set \$dns1 [:resolve dnstest.jsn.net.id server=103.80.80.231];\r\
    \n:set \$dns2 [:resolve dnstest.jsn.net.id server=103.80.80.232];\r\
    \n:if ( \$dns1 = 0 && \$dns2 = 0 && [/ip dns get server] = \"103.80.80.231\
    ;103.80.80.232\" ) do={ \r\
    \n/ip dns set servers=1.1.1.1,8.8.8.8,1.0.0.1\r\
    \n} \r\
    \n:if ( \$dns1 != 0 && \$dns2 != 0 && [/ip dns get server] != \"103.80.80.\
    231;103.80.80.232\") do={ \r\
    \n/ip dns set servers=103.80.80.231,103.80.80.232\r\
    \n} \r\
    \n}\r\
    \n" policy=\
    ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon \
    start-time=startup
```

### 5. Membuat Secret User l2tp
Pada router Daerah (Cyber). Buat secret user L2TP baru, Copy-Paste urut dari user terakhir kemudian ganti user sesuai nama mitra dengan penambahan `("-1")` `contoh : "BUDI-1"`, kemudian isikan Local Address & Remote Address berurutan dan ditambah 1. 

### 6. Dial l2tp
Kembali ke router mitra. Buka Interface, tambah `[+]` L2TP Client, isikan Nama `"JSN-1"` kemudian connet-to isikan dengan ip loopback tujuan tunnel (pilih salah satu), kemudian User & Password sesuai secrete yang kita buat pada Router Cyber. Pastikan koneksi l2tp sudah terhubung/muncul tanda `R`.


> Jika koneksi l2tp putus/nyambung. kembali ke router cyber dan buka secret yang sebelumnya dan ganti profile menjadi `"Profile1"` (sudah ada). Coba sambungkan lagi sampai terhubung dan stabil.

Kemudian kembali ke Router Cyber dan temukan koneksi l2tp dinamic dengan secret sesuai diatas. Copy dan ubah nama menjadi `"NAMAMITRA-1"` `contoh: "BUDI-1"` atau jika nama lebih dari satu kata, pisahkan dengan tanda strip `(-)` `contoh: "BUDI-DOREMI-1"`.

### 7. Membuat Koneksi IPIP

>Langkah ini harus dilakukan pada kedua router yang akan dihubungkan.

Pertama lakukan di router mitra dengan membuat IPIP baru, buka Interface tab Ip Tunnel, add `[+]` interface baru. Nama isikan "JSN-IPIP-1" kemudian Local Address isikan Ip Publik Ind*** dan Remote Address isikan ip tujuan tunnel yang kita pilih diawal tadi.

Kemudian langkah yang sama kita lakukan di router cyber. Untuk nama diisikan nama mitra sesuai format `"NAMAMITRA-IPIP-1"` `( ex: BUDI-IPIP-1")` jika nama mitra lebih dari satu kata,pisahkan dengan tanda strip `(-)` `contoh: "BUDI-DOREMI-IPIP-1"`
> Pastikan kedua Koneksi IPIP sudah terhubung!

### 8. Membuat Bridge Loopback

Tambahkan interface `Bridge` baru. Buka tab `Bridge` add `[+]` new interface bridge, isikan dengan nama`"lo"`.

### 9. Membuat IP Address

Saatnya memberikan Ip Address pada koneksi `IPIP` yang berhasil kita buat sebelumnya pada kedua router terhubung. Dan tambahkan ip publik yang dialokasikan untuk mitra. Dibuat di [PhpIpham](http://103.80.80.102:9191/index.php?page=subnets&section=1&subnetId=202)

- [x]  Pada router cyber.

Buka jendela `Address List` atau klik `IP > Addresses` kemudian temukan Ip untuk user l2tp urutan terakhir dan buat alokasi ip dengan urutan setelahnya `(bisa copas)` dan pada interface pilih koneksi `IPIP` yang kita maksudkan.

- [x] Pada router mitra.

Buka jendela `Address List` atau klik `IP > Addresses` tambahkan `[+]` Ip sesuai yang sudah kita buat dicyber dengan membalik value `Address dan Network` kemudian pada Interface pilih `"JSN-IPIP-1'`

Tambahkan juga `ip address` pada interface `Bridge lo` isikan `Network` dan `Address` sesuai dengan alokasi yang sudah kita buat sebelumnya.

> Coba tes `ping` dari kedua router. Jika tidak ada kesalahan pada konfigurasi diatas seharusnya ping akan reply.

### 10. Tambahkan Interface List

Buat Interface list baru yang berisi koneksi `l2tp` dan `ipip` dengan cara masuk ke `Interface` pada tab `Interface List` Pilih tombol ``List`` add `[+]` new list dengan nama `"JSN"`. Kembali ke tab `Interface List` tambahkan `[+]` interface `"JSN-1"` dan `"JSN-IPIP-1"`.


```yaml
# ...atau paste script dibawah ini :

/interface list
add name=JSN

/interface list member
add interface=JSN-1 list=JSN
add interface=JSN-IPIP-1 list=JSN
```

### 11. Konfigurasi Firewall

##### 1. Tambahkan Address List
Masuk ke menu `IP > Firewall` pada tab `Address List` add `[+]` new list dengan nama `"RFC-1918"` Address isikan `10.0.0.0/8` kemudian ulangi hal yang sama, berikutnya isikan Address `192.168.0.0/16` dan terakhir `172.16.0.0/12`.

```yaml
# ...atau paste script ke terminal :

/ip firewall address-list
add address=10.0.0.0/8 list=RFC-1918
add address=192.168.0.0/16 list=RFC-1918
add address=172.16.0.0/12 list=RFC-1918
```

##### 2. Konfigurasi NAT
Ada tiga konfigurasi yang akan kita lakukan pada `firewall nat` ini.

* Pertama kita buat `firewall nat` dengan `action srcnat`.

Masuk ke menu `IP` kemudian `Firewall` pada tab `NAT` add `[+]` NAT baru. Pada tab `General` pilih chain `srcnat` kemudian `Out. Interface List:` pilih list `JSN`. Pindah ke tab `Advannce` pada kolom `Src. Address List:` pilih list `RFC-1918`. Pada `Dst. Address Lst` pilih list `RFC-1918` dan aktifkan `!` yang artinya kecuali list yang kita pilih. 
Pindah ke tab `Action` pilih `srcnat` kemudian `To Addresses` isikan `Ip Publik` yang sudah ditentukan diawal.

* Sekarang kita buat `nat redirect` untuk `tcp`.

Masih di tab `NAT` add `[+]` `New NAT Rule` pada kolom chain isikan `dstnat` Protocol pilih `udp` dan `Dst. Port` isikan `53,5353` kemudian pindah ke tab `Action` pilih `redirect` dan `To Ports` isikan `53` kemudian `Apply`.
 
* Kemudian kita buat untuk protocol `tcp`.

Caranya tinggal `Copy` `NAT` yang sebelumnya kita buat, kemudian ganti protocolnya menjadi `tcp` klik `OK` dan selesai.

Atau jika tidak ingin cara manual bisa pastekan `script` dibawah ini (**Ganti ip publik**):

```sh
# ...atau paste script dibawah ini :

/ip firewall nat
add action=src-nat chain=srcnat dst-address-list=!RFC-1918 \
    out-interface-list=JSN protocol=!ospf src-address-list=RFC-1918 \
    to-addresses=103.153.148.46
add action=redirect chain=dstnat dst-port=53,5353 protocol=udp to-ports=53
add action=redirect chain=dstnat dst-port=53,5353 protocol=tcp to-ports=53
```

##### 3. Tambahkan Filter Rule
Masih pada tab `Firewall` kita akan tambahkan 4 buah `filter rule` untuk drop ip lokal yang mengarah ke ip publik.
* Pertama filtering port `udp` input

Pada menu `Firewall` klik add `[+]` new rule `Chain: input` kemudian `Protocol: udp` `Dst.Port: 53` dan `In.Interface: INET1` pindah ke tab `Action` pilih `drop` `Apply`.

* Kedua Filtering port `tcp` input

Silahkan `Copy` rule `udp` yang sebelumnya kemudian ganti `Protocol;` menjadi `tcp` kemudian `OK`.

* Ketiga Filtering port `udp` forward

Masih di tab menu `Firewall` add `[+]` new rule, pada tab `General` isikan `Chain: forward` lalu `Protocol: udp` kemudian `Dst. Port: 445,137-139` dan `Out. Interface List: JSN`. Pada tab `Advance` isikan `Src. Address list: RFC-1918`. Pindah ke tab `Action` isikan `Action: drop` kemudian beri komen `"BLOCK ARAH JSN"` dan `Apply`.

* Keempat Filtering port `tcp` forward

Silahkan `Copy` filtering `udp` terakhir. Kemudian pada tab `General` ubah `Protocol:` menjadi `tcp` dan `Dst. Port:` isikan `25,1443,445,137,139,8291,3389` kemudian `Apply`

```yaml
# ...atau paste script dibawah ini :

/ip firewall filter
add action=drop chain=input dst-port=53 in-interface=INET1 protocol=tcp
add action=drop chain=input dst-port=53 in-interface=INET1 protocol=udp
add action=drop chain=forward comment="BLOCK ARAH JSN" dst-port=\
    25,1443,445,137,139,8291,3389 out-interface-list=JSN protocol=tcp \
    src-address-list=RFC-1918
add action=drop chain=forward comment="BLOCK ARAH JSN" dst-port=445,137-139 \
    out-interface-list=JSN protocol=udp src-address-list=RFC-1918
```

### 12. Konfigurasi Routing

Untuk routing harus kita lakukan pada router Mitra dan juga Cyber. Supaya kedua router dapat berkomunikasi melewati gateway yang sudah kita tentukan.

##### Yang pertama kita konfigurasi routing pada kedua router

- [x] Pada router Mitra

Buka menu `IP` `Routes` pada tab `Routes`
* add `[+]` new route `Gateway: "INET1"` `Distance: 3` dan `Apply`. Kemudian Copy ubah `Distance: 1` dan `Routing Mark: INET1` kemudian `OK`.
* add `[+]` new route `Gateway: ip-ipip` `Distance: 1` dan `Apply`. Kemudian Copy isikan `Routing Mark: "JSN"` dan `OK`.
* add `[+]` new route `Gateway: ip-l2tp` `Distance: 2` dan `Apply`. Kemudian Copy isikan `Routing Mark: "JSN"` tambahkan `Comment` `"IP TUJUAN TUNNEL"` dan `OK`.
* add `[+]` new route `Dst. Address : IP-PUBLIK` `Gateway: "INET1" kemudian tambahkan `comment : =INET1` dan `OK`.

```yaml
# ...atau paste script dibawah ini :

/ip route
add check-gateway=ping comment="IP TUJUAN TUNNEL" distance=1 dst-address=103.80.83.32/32 gateway=INET1
add distance=1 gateway=INET1 routing-mark=INET
add check-gateway=ping distance=3 gateway=INET1
add check-gateway=ping distance=1 gateway=172.31.252.6 routing-mark=JSN
add check-gateway=ping distance=1 gateway=172.31.252.6
add check-gateway=ping distance=2 gateway=172.31.255.1 routing-mark=JSN
add check-gateway=ping distance=3 gateway=172.31.255.1
```


- [x] Pada router Cyber

Buka menu `IP` `Routes` pada tab `Routes`
* add `[+]` new route `Dst. Address: ip-publik-mitra` kemudian `Gateway: ip-ipip-mitra` `Distance: 3` dan `Apply`. Kemudian Copy ubah `Distance: 1` dan `Routing Mark: INET1` kemudian `OK`.

##### Yang kedua kita konfigurasi Routing Rules

- [x] Pada router Mitra

Untuk table `main` ip lokal, gunakan `script` dibawah ini dan pastekan ke `terminal`:

```yaml
/ip route rule
add dst-address=192.168.0.0/16 table=main
add dst-address=172.16.0.0/12 table=main
add dst-address=10.0.0.0/8 table=main
```

Kemudian kita buat rule untuk ip publik `Indihome` dan juga `JSN`.

Silakan buka menu `IP` `Routes` kemudian pilih tab `Rules`
* add `[+]` new rule `Src. Address: 36, 71, 32, 222` `Action: lookup` dan `Table: INET` kemudian berikan Komen `"INET1"` lalu `OK`.
* add `[+]` new rule `Src. Address: ip-publik` `Action: lookup` dan `Table: JSN` lalu `OK`.


### 13. Ganti port Winbox

Ganti port untuk remote winbox sesuai dengan `SOP`. Caranya masuk ke menu `IP` kemudian `Services` lalu `Klik` 2 kali pada `winbox` pada kolom `port:` isikan `65432` lalu `OK`.


### 14. Buat User NOC

Buat user khusus **NOC** untuk monitoring dan penenganan jika suatu saat mitra komplain terjadi masalah. Dan berikan pengertian jug akepada mitra untuk tidak men-disable atau menghapus user tersebut.


### Selesai

Sekarang coba lakukan remote ke perangkat Mitra menggunakan ip publik yang sudah kita berikan tadi.




> **Note :** 
>> - Jalur yang dipakai untuk lewat BW adalah IPIP.
>> - Default route & peer DNS pada Indi disable.
>> - Jika IPIP putus-nyambung, cek lagi routingnya.
