# ขั้นตอนการติดตั้ง DNS Server (Bind9)

### 1. อัปเดตระบบให้เป็นเวอร์ชันล่าสุด

```bash
sudo apt update && sudo apt upgrade -y
```

### 2.ตั้งค่า Static IP และติดตั้งแพ็คเกจ

วิธีการตั้งค่า Static IP adddress บน Linux ubuntu 22.04 LTS [Link](https://github.com/teerakanotk/ubuntu/blob/main/docs/static-ip.md)

ติดตั้งแพ็คเกจ bind9

```bash
sudo apt install bind9
```

ติดตั้งแพ็คเกจ `dnsutils` สำหรับทดสอบและแก้ไขปัญหา

```bash
sudo apt install dnsutils
```

### 3. ตั้งค่า Caching nameserver

กรณีที่ DNS Server ภายในองค์กรไม่สามารถค้นหาชื่อโดเมนที่ผู้ใช้งานร้องขอได้ เซิร์ฟเวอร์จะทำการ `ส่งต่อ (forward)` คำขอเหล่านั้นไปยัง `IP address` ที่กำหนดเอาไว้ภายใน **forwarders**

```bash
sudo nano /etc/bind/named.conf.options
```

ในส่วน forwarders ให้เปลี่ยนจาก `0.0.0.0` เป็นที่อยู่ไอพีแอดเดรสของ dns server ที่ต้องการส่งต่อ เช่น ผู้ให้บริการอินเทอร์เน็ต (ISP)

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

         forwarders {
                1.1.1.1;
                1.0.0.1;
         };

        //=========================================================>
        // If BIND logs error messages about the root key being exp>
        // you will need to update your keys.  See https://www.isc.>
        //=========================================================>
        dnssec-validation auto;

        listen-on-v6 { any; };
};
```

### 4. ตั้งค่า Forward zone File

```bash
sudo nano /etc/bind/named.conf.local
```

เพิ่มการตั้งค่าต่อไปนี้

```
zone "ctsurin.local" {
  type master;
  file "/etc/bind/db.ctsurin.local";
};
```

> เปลี่ยน ctsurin.local เป็นชื่อโดเมนที่ต้องการ

คัดลอกไฟล์ `/etc/bind/db.local` แล้วตั้งชื่อเป็น `/etc/bind/db.ctsurin.local`

```bash
sudo cp /etc/bind/db.local /etc/bind/db.ctsurin.local
```

จากนั้นแก้ไขข้อความภายในไฟล์ `/etc/bind/db.ctsurin.local`

```bash
sudo nano /etc/bind/db.ctsurin.local
```

> Note:<br>
> เปลี่ยน `localhost` เป็น ชื่อโดเมนที่ต้องการ<br>
> เปลี่ยน `127.0.0.1` เป็น IP address ของเครื่องเซิร์ฟเวอร์

```
;
; BIND data file for ctsurin.local
;
$TTL    604800
@       IN      SOA     ctsurin.local. root.ctsurin.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@       IN      NS      ns.ctsurin.local.
@       IN      A       10.11.254.10
@       IN      AAAA    ::1
ns      IN      A       10.11.254.10
```

### 5. ตั้งค่า Reverse zone File

```bash
sudo nano /etc/bind/named.conf.local
```

เพิ่มการตั้งค่าต่อไปนี้ที่ด้านล่างของ `zone "ctsurin.local"`

```
zone "254.11.10.in-addr.arpa" {
  type master;
  file "/etc/bind/db.10";
};
```

> Note:<br>
> `254.11.10` มาจากเลข 3 ชุดแรกของ IP มาสลับลำดับจากหลังมาหน้า เช่น `192.168.100.0` จะได้ `100.192.168.in-addr.arpa`

คัดลอกไฟล์ `/etc/bind/db.127` แล้วตั้งชื่อเป็น `/etc/bind/db.10`

> `db.10` มาจากเลขชุดแรกของ IP เช่น `192.168.100.0` จะได้เป็น `db.192`

```bash
sudo cp /etc/bind/db.127 /etc/bind/db.10
```

แก้ไขไฟล์ `/etc/bind/db.10` เหมือนกันกับ `Forward zone`

```bash
sudo nano /etc/bind/db.10
```

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

```bash
sudo systemctl restart bind9.service
```

> Note:<br>
> ควรเพิ่มเลข Serial ขึ้นทุกครั้งๆ ละ 1 เมื่อมีการแก้ไขไฟล์ `db.ctsurin.local` หรือ `db.10`

<br>

## ทดสอบ

### 1. ตั้งค่า resolv ของเครื่อง DNS Server

เปิดไฟล์ `/etc/resolv.conf` 

```bash
sudo nano /etc/resolv.conf
```

จากนั้นแก้ไขในส่วน `search` โดยเปลี่ยน `.` เป็นชื่อโดเมน `FQDN`

```
nameserver 127.0.0.53
search ctsurin.local
```

จากนั้นใช้คำสั่ง `resolvectl status` เพื่อตรวจสอบการตั้งค่าว่า ถูกต้องหรือไม่

```bash
resolvectl status
```

ผลลัพธ์:

```bash
Global
       Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
resolv.conf mode: stub

Link 2 (ens33)
    Current Scopes: DNS
         Protocols: +DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 10.11.254.10
       DNS Servers: 10.11.254.10
        DNS Domain: ctsurin.local
```

### 2. ทดสอบการทำงานของ Caching Nameserver

ใช้คำสั่ง dig เพื่อตรวจสอบเวลาการตอบสนอง (Query Time) ของการค้นหาชื่อโดเมนนอกองค์กร

```bash
dig ubuntu.com
```

ผลลัพธ์:

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

### 3. PING

ทดลองใช้คำสั่ง PING ไปยังโดเมน

```bash
ping ctsurin.local
```

ผลลัพธ์:

```
PING ctsurin.local (10.11.254.10) 56(84) bytes of data.
64 bytes from ns.ctsurin.local (10.11.254.10): icmp_seq=1 ttl=64 time=0.089 ms
64 bytes from ns.ctsurin.local (10.11.254.10): icmp_seq=2 ttl=64 time=0.164 ms
64 bytes from ns.ctsurin.local (10.11.254.10): icmp_seq=3 ttl=64 time=2.22 ms
64 bytes from ns.ctsurin.local (10.11.254.10): icmp_seq=4 ttl=64 time=0.055 ms
64 bytes from ns.ctsurin.local (10.11.254.10): icmp_seq=5 ttl=64 time=0.052 ms
^C
--- ctsurin.local ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4021ms
rtt min/avg/max/mdev = 0.052/0.516/2.220/0.852 ms
```

> Note:<br>
> อย่าลืมกำหนด dns มายังเครื่องเซิร์ฟเวอร์ มิเช่นนั้นจะไม่สามารถค้นหาที่อยู่โดเมนภายในองค์กรได้
