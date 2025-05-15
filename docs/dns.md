# คู่มือการติดตั้ง DNS Server บน Linux Ubuntu 22.04 LTS (CLI)

## บทนำ  
เอกสารนี้อธิบายขั้นตอนการติดตั้งและตั้งค่า DNS Server บน Ubuntu 22.04 LTS โดยใช้ Command Line Interface (CLI) เหมาะสำหรับผู้ดูแลระบบที่ต้องการควบคุมการกำหนดค่าระบบเครือข่ายด้วยตนเอง

---

## ขั้นตอนการติดตั้ง

### 1. อัปเดตแพตช์ความปลอดภัยของระบบ

```bash
sudo apt update -y && sudo apt upgrade -y
```

### 2. ปิดการตั้งค่า IP อัตโนมัติจาก Cloud-init

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```
