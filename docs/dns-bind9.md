# วิธีการติดตั้ง DNS Server บน Linux ubuntu 22.04 LTS (CLI)

### 1. อัปเดตระบบและติดตั้งแพตช์ความปลอดภัยให้เป็นเวอร์ชันล่าสุด

```bash
sudo apt update && sudo apt upgrade -y
```

### 2.ตั้งค่า Static IP และติดตั้งแพ็คเกจ ```bind9```

[วิธีการกำหนดค่า Static IP address](https://github.com/teerakanotk/ubuntu/blob/main/docs/static-ip.md)

ติดตั้งแพ็คเกจ bind9

```bash
sudo apt install bind9
```

(optional) ติดตั้งแพ็คเกจ ```dnsutils``` สำหรับทดสอบและแก้ไขปัญหา

```bash
sudo apt install dnsutils
```

### 3. รายละเอียด Config file

ไฟล์กำหนดค่าของ DNS จะถูกเก็บไว้ในไดเรกทอรี ```/etc/bind```

- ```/etc/bind/named.conf.options``` สำหรับกำหนดค่า forwarders, acl

- ```/etc/bind/named.conf.local``` สำหรับกำหนด zone

### 4. ตั้งค่า Caching nameserver

กรณีที่ DNS Server ภายในองค์กรไม่สามารถค้นหาชื่อโดเมนที่ผู้ใช้งานร้องขอได้ เซิร์ฟเวอร์จะทำการ **ส่งต่อ (forward)** คำขอเหล่านั้นไปยัง **DNS Server ที่กำหนดเอาไว้ภายใน ```forwarders```

เปิดไฟล์ ```/etc/bind/named.conf.options```

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

ลบ```//```ออก จากนั้นแก้ไขในส่วนของ forwarders โดยเปลี่ยนจาก ```0.0.0.0``` เป็นไอพีแอดเดรส DNS Server ที่ต้องการ 

```bash
forwarders {
  1.1.1.1;
  1.0.0.1;
};
```

```bash
sudo systemctl restart bind9
```

### 5. ตั้งค่า Forward zone File

เปิดไฟล์ ```/etc/bind/named.conf.local``` 

```bash
sudo nano /etc/bind/named.conf.local
```

เพิ่มการตั้งค่าต่อไปนี้

```bash
zone "ctsurin.com" {
  type master;
  file "/etc/bind/db.ctsurin.com";
};
```

note:
- เปลี่ยน ```ctsurin.com``` เป็นชื่อโดเมนที่ต้องการ

คัดลอกไฟล์ ```/etc/bind/db.local``` แล้วตั้งชื่อเป็น ```/etc/bind/db.ctsurin.com```

```bash
sudo cp /etc/bind/db.local /etc/bind/db.ctsurin.com
```

แก้ไขไฟล์ ```/etc/bind/db.ctsurin.com``` โดยเปลี่ยน ```localhost``` เป็น ctsurin หรือชื่อโดเมนที่ต้องการ ตามด้วย ```. (จุด)``` หลังชื่อโดเมน และเปลี่ยน ```127.0.0.1``` เป็น IP address ของเครื่องเซิร์ฟเวอร์ ตัวอย่าง

```bash
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

```bash
sudo systemctl restart bind9
```

### 6. ตั้งค่า Reverse zone File

การตั้งค่า Reverse Zone เพื่อให้ DNS สามารถแปลง IP Address กลับเป็นชื่อโดเมน ให้แก้ไขไฟล์

```bash
sudo nano /etc/bind/named.conf.local
```

เพิ่มการตั้งค่าต่อไปนี้

```bash
zone "254.11.10.in-addr.arpa" {
  type master;
  file "/etc/bind/db.10";
};
```

note:
- เปลี่ยน 254.11.10 เป็นเลข 3 ชุดแรกของ IP มาสลับลำดับจากหลังมาหน้า เช่น 192.168.100.0 จะได้ 100.192.168.in-addr.arpa
- เปลี่ยน db.10 เป็น db.<ip> เช่น 192.168.100.0 จะได้ db.192

คัดลอกไฟล์ ```/etc/bind/db.127``` แล้วตั้งชื่อเป็น ```/etc/bind/db.10```

```bash
sudo cp /etc/bind/db.127 /etc/bind/db.10
```

แก้ไขไฟล์ภายใน โดยการตั้งค่าเหมือนกันกับ Forward zone
```bash
sudo nano /etc/bind/db.10
```

```bash
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
- ควรเพิ่ม Serial ทุกครั้งที่มีการอัพเดทหรือแก้ไขไฟล์ โดยเพิ่มทีละ 1 หน่วย

```bash
sudo systemctl restart bind9.service
```
