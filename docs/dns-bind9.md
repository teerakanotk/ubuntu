# วิธีการติดตั้ง DNS Server บน Linux ubuntu 22.04 LTS (CLI)

### 1. อัปเดตระบบและติดตั้งแพตช์ความปลอดภัยให้เป็นเวอร์ชันล่าสุด

```bash
sudo apt update -y && sudo apt upgrade -y
```

### 2. วิธีกำหนดค่า Static IP address [คลิก](https://github.com/teerakanotk/ubuntu/blob/main/docs/static-ip.md)

### 3. หลังจากกำหนดค่า Static IP เรียบร้อยแล้ว ให้ติดตั้งแพ็คเกจ ```bind9```

```bash
sudo apt install bind9
```

ติดตั้งแพ็คเกจ ```dnsutils``` สำหรับทดสอบและแก้ไขปัญหา

```bash
sudo apt install dnsutils
```

### 4. รายละเอียดไฟล์กำหนดค่าระบบ (Configuration files)

ไฟล์กำหนดค่าของ DNS จะถูกเก็บไว้ในไดเรกทอรี ```/etc/bind``` โดยไฟล์หลักที่ใช้ในการกำหนดค่าคือ ```/etc/bind/named.conf```

- ```/etc/bind/named.conf.options``` กำหนดค่าทั่วไป เช่น forwarders, ACL

- ```/etc/bind/named.conf.local``` zone ของโดเมน

- ```/etc/bind/named.conf.default-zones``` zone เริ่มต้น

### 5. ตั้งค่า Caching nameserver

การตั้งค่า Caching nameserver มีความสำคัญในกรณีที่ DNS Server ภายในองค์กรไม่สามารถค้นหาชื่อโดเมนที่ผู้ใช้งานร้องขอได้ เซิร์ฟเวอร์จะทำการ **ส่งต่อ (forward)** คำขอเหล่านั้นไปยัง **DNS Server ของผู้ให้บริการอินเทอร์เน็ต (ISP)** หรือ **DNS Server อื่นๆ ที่กำหนดไว้**

```bash
sudo nano /etc/bind/named.conf.options
```

ยกเลิกการคอมเมนต์ (ถ้ามี) และเพิ่มส่วนต่อไปนี้ภายใต้ ```options```:

```bash
forwarders {
  1.1.1.1;
  1.0.0.1;
};
```

> เปลี่ยน 1.1.1.1 และ 1.0.0.1 เป็น IP address ที่ต้องการ

หลังจากแก้ไขเสร็จแล้ว restart service bind9 เพื่อให้การตั้งค่ามีผล

```bash
sudo systemctl restart bind9
```

### 6. ตั้งค่า Forward zone File

```bash
zone "ctsurin.com" {
  type master;
  file "/etc/bind/db.ctsurin.com";
};
```

> เปลี่ยน ctsurin.com เป็นชื่อโดเมนที่ต้องการ

สร้างไฟล์สำหรับโซน ```ctsurin.com``` โดยใช้เทมเพลตจาก ```/etc/bind/db.local```

```bash
sudo cp /etc/bind/db.local /etc/bind/db.ctsurin.com
```

แก้ไขไฟล์ ```/etc/bind/db.ctsurin.com``` โดยเปลี่ยน ```localhost``` เป็น ctsurin หรือชื่อโดเมนที่ต้องการ ตามด้วย ```. (จุด)``` หลังชื่อโดเมน และเปลี่ยน ```127.0.0.1``` เป็น IP address ของเครื่องเซิร์ฟเวอร์ เช่น ```10.11.254.10```

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

จากนั้นรีสตาร์ท service bind9 เพื่อให้การตั้งค่ามีผล

```bash
sudo systemctl restart bind9
```

### 7. ตั้งค่า Reverse zone File

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

> - เปลี่ยน 254.11.10 เป็นเลข 3 ชุดแรกของ IP มาสลับลำดับจากหลังมาหน้า เช่น 192.168.100.0 จะได้ 100.192.168.in-addr.arpa
> - เปลี่ยน db.10 เป็น db.<IP_ตัวแรกสุด> เช่น 192.168.100.0 จะได้ db.192

สร้างไฟล์ ```/etc/bind/db.10``` โดยใช้เทมเพลตจาก ```/etc/bind/db.127```

```bash
sudo cp /etc/bind/db.127 /etc/bind/db.10
```

แก้ไขไฟล์ภายใน โดยการตั้งค่าเหมือนกันกับ Forward zone

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

> สำคัญ:<br>
> Serial Number จะต้องเพิ่มค่าขึ้นทุกครั้งที่มีการแก้ไขไฟล์ทั้งในโซน Forward และ Reverse

จากนั้นรีสตาร์ท service bind9 เพื่อให้การตั้งค่ามีผล
