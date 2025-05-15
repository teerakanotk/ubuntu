# วิธีการติดตั้ง DNS Server บน Linux ubuntu 22.04 LTS (CLI)

### 1. อัปเดตระบบและติดตั้งแพตช์ความปลอดภัยให้เป็นเวอร์ชันล่าสุด

```bash
sudo apt update -y && sudo apt upgrade -y
```

### 2. กำหนดค่า IP Address แบบ Static

ขั้นตอนการกำหนดค่า [คลิกเลย](https://github.com/teerakanotk/ubuntu/blob/main/docs/static-ip.md)

### 3. ติดตั้ง DNS โดยใช้แพ็คเกจ ```bind9```

```bash
sudo apt install bind9
```

ติดตั้งแพ็คเกจ ```dnsutils``` สำหรับทดสอบและแก้ไขปัญหา DNS

```bash
sudo apt install dnsutils
```

### 4. รายละเอียดไฟล์กำหนดค่าระบบ (Configuration files)

ไฟล์กำหนดค่าของ DNS จะถูกเก็บไว้ในไดเรกทอรี ```/etc/bind``` โดยไฟล์หลักที่ใช้ในการกำหนดค่าคือ ```/etc/bind/named.conf```

- ```/etc/bind/named.conf.options``` กำหนดค่าทั่วไป เช่น forwarders, ACL

- ```/etc/bind/named.conf.local``` zone ของโดเมน

- ```/etc/bind/named.conf.default-zones``` zone เริ่มต้น

### 5. ตั้งค่า Caching Nameserver

ตั้งค่า DNS forwarders เนื่องจากในกรณีค้นหาที่อยู่โดเมนภายในองค์กรไม่พบ เซิร์ฟเวอร์จะทำการ forwards ไปยัง DNS Server ของผู้ให้บริการอินเทอร์เน็ต (ISP) หรือ DNS Server ที่ต้องการ

```bash
sudo nano /etc/bind/named.conf.options
```

ยกเลิกคอมเมนต์ และแก้ไขเป็น:

```bash
forwarders {
  1.1.1.1;
  1.0.0.1;
};
```

> เปลี่ยน 1.1.1.1 และ 1.0.0.1 เป็น IP address ที่ต้องการ

รีสตาร์ทเซอร์วิส bind9

```bash
sudo systemctl restart bind9
```
