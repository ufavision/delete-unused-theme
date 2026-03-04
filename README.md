# delete-unused-themes.sh

สคริปต์ลบ WordPress Theme ที่ไม่ได้ใช้งานออกจากทุกเว็บใน server โดยอัตโนมัติ รองรับการประมวลผลแบบ parallel และมีระบบ log ครบถ้วน

---

## ทำอะไร?

- ค้นหาเว็บ WordPress ทั้งหมดใน `/home/*/public_html/*/`
- ทำงาน 2 Phase:
  - **Phase 1** — ตรวจสอบว่าเว็บไหนมี Theme ที่ต้องลบ
  - **Phase 2** — ลบ Theme เฉพาะเว็บที่พบ
- รัน parallel อัตโนมัติตาม CPU และ RAM ที่มี (สูงสุด 20 jobs)
- **ไม่ลบ** Theme ที่กำลัง Active อยู่ — จะข้ามและแจ้งเตือนแทน
- บันทึก log ทุกขั้นตอนไว้ที่ `/var/log/wp-delete-unused-themes.log`

### Theme ที่กำหนดไว้ในสคริปต์

| ประเภท | Theme |
|---|---|
| ❌ ลบ | `twentytwentythree`, `twentytwentyfour` |
| ✅ เก็บไว้ | `twentytwentyfive`, `blocksy`, `blocksy-child` |

> ต้องการเพิ่ม/ลด Theme ให้แก้ตัวแปร `DELETE_THEMES` และ `KEEP_THEMES` ในไฟล์สคริปต์

## คำสั่งในการัน
---
```bash
bash <(curl -s https://raw.githubusercontent.com/ufavision/delete-unused-theme/main/delete-unused-themes.sh)
```
---
## ข้อกำหนด

| รายการ | รายละเอียด |
|---|---|
| OS | Linux (CentOS, Ubuntu, AlmaLinux ฯลฯ) |
| Shell | Bash 4.0 ขึ้นไป |
| สิทธิ์ | root หรือ user ที่มีสิทธิ์อ่าน/เขียน `/home` |
| เครื่องมือ | [WP-CLI](https://wp-cli.org/) ต้องติดตั้งและเรียกใช้ได้ด้วยคำสั่ง `wp` |
| เหมาะกับ | เซิร์ฟเวอร์ cPanel / DirectAdmin ที่มีหลาย user |

---

## ตัวอย่าง Output

```
======================================
 DELETE UNUSED THEMES
 เริ่มเวลา      : 2025-01-01 10:00:00
 CPU Cores     : 4 Core
 Total RAM     : 8192 MB
 Auto MAX_JOBS : 4
 ลบ Theme      : twentytwentythree twentytwentyfour
 เก็บ Theme     : twentytwentyfive blocksy blocksy-child
======================================
พบ WordPress ทั้งหมด: 10 เว็บ

 PHASE 1: กำลังตรวจสอบ...
✅ OK (ไม่มี Theme ที่ต้องลบ): user1/site1
⚠️  NEEDS DELETE: user2/site2
...

 PHASE 2: กำลังลบ Theme จาก 3 เว็บ...
✅ DELETED: user2/site2 → twentytwentythree
⚠️  SKIPPED (Active Theme): user3/site3 → twentytwentyfour
❌ FAILED DELETE: user4/site4 → twentytwentythree

======================================
 สรุปผลรวม
 รวมทั้งหมด               : 10 เว็บ
 ✅ ไม่มี Theme ที่ต้องลบ   : 7 เว็บ
 ✅ ลบสำเร็จ               : 2 เว็บ
 ❌ ลบไม่สำเร็จ            : 1 เว็บ
 ⚠️  ข้าม (Active Theme)   : 1 เว็บ
 เวลาที่ใช้                : 1 นาที 23 วินาที
 Log อยู่ที่               : /var/log/wp-delete-unused-themes.log
======================================
```

---

## ดู Log

### ดู log แบบ real-time (ขณะสคริปต์กำลังรัน)

```bash
tail -f /var/log/wp-delete-unused-themes.log
```

### ดู log ย้อนหลัง 100 บรรทัด

```bash
tail -n 100 /var/log/wp-delete-unused-themes.log
```

### ดู log ทั้งหมด

```bash
cat /var/log/wp-delete-unused-themes.log
```

### กรองเฉพาะ error

```bash
grep "❌" /var/log/wp-delete-unused-themes.log
```

### กรองเฉพาะเว็บที่ถูกข้าม (Active Theme)

```bash
grep "SKIPPED" /var/log/wp-delete-unused-themes.log
```

### กรองตามวันที่

```bash
grep "2025-01-01" /var/log/wp-delete-unused-themes.log
```
