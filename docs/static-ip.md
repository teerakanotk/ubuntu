# วิธีการตั้งค่า Static IP Address บน Linux ubuntu 22.04 LTS (CLI)

### 1. อัปเดตแพตช์ความปลอดภัยของระบบ

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. ปิดการตั้งค่า IP อัตโนมัติจาก Cloud-init

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

เพิ่มข้อความต่อไปนี้ลงในไฟล์:

```yaml
network: { config: disabled }
```

### 3. เปลี่ยนชื่อไฟล์ cloud-init เพื่อป้องกันไม่ให้ netplan อ่านการตั้งค่าจากไฟล์ดังกล่าว

```bash
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak
```

### 4. สร้างไฟล์ 01-netcfg.yaml

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

เทมเพลต:

```yaml
network:
  ethernets:
    <interface>:
      dhcp4: false
      addresses:
        - <ip>/<subnet>
      routes:
        - to: default
          via: <gateway>
      nameservers:
        addresses:
          - <primary_dns>
          - <secondary_dns>
        search: [<fqdn>]
  version: 2
```

ตัวอย่าง:

```yaml
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 10.11.254.10/24
      routes:
        - to: default
          via: 10.11.254.2
      nameservers:
        addresses:
          - 1.1.1.1
          - 1.0.0.1
        search: []
  version: 2
```

### 5. จากนั้นใช้คำสั่ง netplan apply เพื่อบันทึกการเปลี่ยนแปลง

```bash
sudo netplan apply
```

### หมายเหตุ

- ens33 คือชื่อของ interface (ตรวจสอบได้โดยใช้คำสั่ง ip a)
