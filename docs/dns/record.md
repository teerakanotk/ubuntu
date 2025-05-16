# ประเภทของเรคคอร์ด DNS ที่พบบ่อย

### A Record

ใช้สำหรับ IPv4 โดยจับคู่โดเมนกับ IP Address

```
www     IN      A       10.11.254.20
```

ตัวอย่าง: `www.ctsurin.com` จะชี้ไปที่ `10.11.254.20`

### CNAME Record

สร้างชื่อแทน (alias) ให้กับ A record ที่มีอยู่

`ไม่สามารสร้าง CNAME ชี้ไปยัง CNAME อื่นได้`

```
web     IN      CNAME       www
```

ตัวอย่าง: `web.ctsurin.com` จะชี้ไปที่ `www.ctsurin.com`

### MX Record

ระบุว่า อีเมลของโดเมนนั้นควรส่งไปที่ใด

`ต้องชี้ไปที่ A Record เท่านั้น`

```
@       IN  MX  1   mail.ctsurin.com
mail    IN  A       10.11.254.18
```

ตัวอย่าง: โดเมนหลักจะใช้ mail server ที่ `mail.ctsurin.com` (`10.11.254.18`)

### NS Record

ใช้ระบุว่า DNS Server ใดบ้างที่ให้บริการข้อมูลโซนนี้

`ต้องชี้ไปที่ A Record เท่านั้น`

```
@   IN  NS  ns.ctsurin.com
@   IN  A   10.11.254.10
```
