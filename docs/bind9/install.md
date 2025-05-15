# ขั้นตอนการติดตั้ง DNS Server (Bind9)

## Install

### 1. อัปเดตระบบและติดตั้งแพตช์ความปลอดภัยให้เป็นเวอร์ชันล่าสุด

```bash
sudo apt update && sudo apt upgrade -y
```

### 2.ตั้งค่า Static IP และติดตั้งแพ็คเกจ `bind9`

[วิธีการกำหนดค่า Static IP address](https://github.com/teerakanotk/ubuntu/blob/main/docs/static-ip.md)

ติดตั้งแพ็คเกจ bind9

```bash
sudo apt install bind9
```

(optional) ติดตั้งแพ็คเกจ `dnsutils` สำหรับทดสอบและแก้ไขปัญหา

```bash
sudo apt install dnsutils
```

### 3. ตั้งค่า Caching nameserver

กรณีที่ DNS Server ภายในองค์กรไม่สามารถค้นหาชื่อโดเมนที่ผู้ใช้งานร้องขอได้ เซิร์ฟเวอร์จะทำการ **ส่งต่อ (forward)** คำขอเหล่านั้นไปยัง **DNS Server ที่กำหนดเอาไว้ภายใน `forwarders`**

```bash
sudo nano /etc/bind/named.conf.options
```

```
options {
        directory "/var/cache/bind";

        // If there is a firewall between you and nameservers you w>
        // to talk to, you may need to fix the firewall to allow mu>
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders>
        // Uncomment the following block, and insert the addresses >
        // the all-0's placeholder.

        // forwarders {
        //        0.0.0.0;
        // };

        //=========================================================>
        // If BIND logs error messages about the root key being exp>
        // you will need to update your keys.  See https://www.isc.>
        //=========================================================>
        dnssec-validation auto;

        listen-on-v6 { any; };
};
```

ในส่วนของ forwarders เปลี่ยนจาก `0.0.0.0` เป็นไอพีแอดเดรส DNS Server ที่ต้องการ

```
forwarders {
  1.1.1.1;
  1.0.0.1;
};
```

### 4. ตั้งค่า Forward zone File

Forward zone คือ โซนที่ใช้เก็บ record สำหรับแปลงชื่อ ให้กลายเป็น IP

```bash
sudo nano /etc/bind/named.conf.local
```

เพิ่มการตั้งค่าต่อไปนี้

```
zone "ctsurin.com" {
  type master;
  file "/etc/bind/db.ctsurin.com";
};
```

note:

- เปลี่ยน `ctsurin.com` เป็นชื่อโดเมนที่ต้องการ

คัดลอกไฟล์ `/etc/bind/db.local` แล้วตั้งชื่อเป็น `/etc/bind/db.ctsurin.com`

```bash
sudo cp /etc/bind/db.local /etc/bind/db.ctsurin.com
```

เปิดไฟล์ `/etc/bind/db.ctsurin.com`

```bash
sudo nano /etc/bind/db.ctsurin.com
```

```
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     localhost. root.localhost. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      localhost.
@       IN      A       127.0.0.1
@       IN      AAAA    ::1
```

- เปลี่ยน `localhost` เป็น `ชื่อโดเมน`
- เปลี่ยน `127.0.0.1` เป็น `IP address ของเครื่องเซิร์ฟเวอร์`

```
;
; BIND data file for ctsurin.com
;
$TTL    604800
@       IN      SOA     ctsurin.com. root.ctsurin.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@       IN      NS      ns.ctsurin.com.
@       IN      A       10.11.254.10
@       IN      AAAA    ::1
ns      IN      A       10.11.254.10
```

### 5. ตั้งค่า Reverse zone File

Reverse zone คือ โซนที่ใช้เก็บ record สำหรับแปลง IP Address ให้กลายเป็นชื่อโดเมน (ตรงกันข้ามกับ Forward Zone ที่แปลงชื่อเป็น IP)

```bash
sudo nano /etc/bind/named.conf.local
```

เพิ่มการตั้งค่าต่อไปนี้

```
zone "254.11.10.in-addr.arpa" {
  type master;
  file "/etc/bind/db.10";
};
```

note:

- 254.11.10 เป็นเลข 3 ชุดแรกของ IP มาสลับลำดับจากหลังมาหน้า เช่น 192.168.100.0 จะได้ 100.192.168.in-addr.arpa
- db.10 เป็น db.<ip> เช่น 192.168.100.0 จะได้ db.192

คัดลอกไฟล์ `/etc/bind/db.127` แล้วตั้งชื่อเป็น `/etc/bind/db.10`

```bash
sudo cp /etc/bind/db.127 /etc/bind/db.10
```

แก้ไขไฟล์ `/etc/bind/db.10` เหมือนกันกับ `Forward zone`

```
;
; BIND reverse data file for local 10.11.254.XXX net
;
$TTL    604800
@       IN      SOA     ns.ctsurin.com. root.ctsurin.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.
10      IN      PTR     ns.ctsurin.com.
```

note:

- เลข Serial ควรเพิ่มขึ้นทุกครั้งเมื่อมีการแก้ไขไฟล์ `/etc/bind/db.ctsurin.com` และ `/etc/bind/db.10` โดยเพิ่มทีละ 1 หน่วย

```bash
sudo systemctl restart bind9.service
```

## Test

### 1. resolve.conf

แก้ไขไฟล์ `/etc/resolve.conf` โดยตั้งค่าดังนี้

```
nameserver 127.0.0.53
search ctsurin.com
```

จากนั้นใช้คำสั่ง `resolvectl status`

```bash
Global
       Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
resolv.conf mode: stub

Link 2 (ens33)
    Current Scopes: DNS
         Protocols: +DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 10.11.254.10
       DNS Servers: 10.11.254.10
        DNS Domain: ctsurin.com
```

### 2. ทดสอบการทำงานของ Caching Nameserver

ใช้คำสั่ง dig เพื่อตรวจสอบเวลาการตอบสนอง (Query Time) ของการค้นหาชื่อโดเมนนอกองค์กร

```bash
dig ubuntu.com
```

```
; <<>> DiG 9.18.30-0ubuntu0.22.04.2-Ubuntu <<>> ubuntu.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 22137
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;ubuntu.com.                    IN      A

;; ANSWER SECTION:
ubuntu.com.             60      IN      A       185.125.190.29
ubuntu.com.             60      IN      A       185.125.190.20
ubuntu.com.             60      IN      A       185.125.190.21

;; Query time: 228 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Thu May 15 15:42:59 UTC 2025
;; MSG SIZE  rcvd: 87
```

### 3. ทดลอง PING ไปยังโดเมนหลัก

```bash
ping ctsurin.com
```

From server:

```
noll@dns:~$ ping ctsurin.com
PING ctsurin.com(ip6-localhost (::1)) 56 data bytes
64 bytes from ip6-localhost (::1): icmp_seq=1 ttl=64 time=0.077 ms
64 bytes from ip6-localhost (::1): icmp_seq=2 ttl=64 time=0.053 ms
64 bytes from ip6-localhost (::1): icmp_seq=3 ttl=64 time=0.063 ms
^C
--- ctsurin.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2039ms
rtt min/avg/max/mdev = 0.053/0.064/0.077/0.009 ms
```

From client:

```
Pinging ctsurin.com [10.11.254.10] with 32 bytes of data:
Reply from 10.11.254.10: bytes=32 time<1ms TTL=64
Reply from 10.11.254.10: bytes=32 time<1ms TTL=64
Reply from 10.11.254.10: bytes=32 time=1ms TTL=64
Reply from 10.11.254.10: bytes=32 time<1ms TTL=64

Ping statistics for 10.11.254.10:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 1ms, Average = 0ms
```

หมายเหตุ:

- เครื่อง client อย่าลืมตั้งค่า dns มายังเซิร์ฟเวอร์ ไม่อย่างนั้นจะมองไม่เห็น domain
