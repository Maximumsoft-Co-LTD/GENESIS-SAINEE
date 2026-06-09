# CLAUDE.md — sainee-logs

repo นี้คือ personal dev log ของ Sainee

## หน้าที่ของ Claude ใน repo นี้

ทุกครั้งที่ทำงานใน session นี้ เมื่อ session จบ ให้ append สรุปลงใน `daily/YYYY-MM-DD.md` (วันปัจจุบัน) ใต้ section `## Claude Sessions` โดยอัตโนมัติ

## format ที่ต้องเพิ่ม

```markdown
### HH:MM — <หัวข้อสั้นๆ ของ session>
- **Prompt หลัก:** <สรุป prompt แรกที่ user ถาม>
- **ผลลัพธ์:** <สรุปสิ่งที่ได้จาก session นี้>
- **สิ่งที่นำไปใช้:** <ถ้ามี — เช่น สร้างไฟล์, แก้ code, research topic>
- **Tokens:** ~<ประมาณ> | **เวลา:** ~<นาที>นาที
```

## กฎการเขียน log
- ใช้ภาษาไทยได้
- สรุปให้กระชับ อ่านแล้วรู้เรื่องภายใน 10 วินาที
- ถ้า session นี้ได้ skill หรือ rule ใหม่ ให้บอก user ด้วยว่าควรสร้างไฟล์ใน `skills/` หรือ `rules/`
- ถ้า research ลึก ให้บอก user ว่าควรแยกไฟล์ไว้ใน `research/`

## โครงสร้าง repo
- `daily/` — log รายวัน
- `skills/` — สิ่งที่เรียนรู้ใหม่
- `rules/` — convention / กฎที่ใช้
- `research/` — research ลึกๆ แยกไฟล์
