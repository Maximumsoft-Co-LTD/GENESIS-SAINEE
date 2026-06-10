# CLAUDE.md — sainee-logs

repo นี้คือ personal dev log ของ Sainee

## ⚠️ กฎสำคัญที่สุด: ก่อน commit ต้องถามก่อนเสมอ

- **ห้าม** `git commit` หรือ `git push` เองโดยไม่ได้รับอนุญาตจาก user
- แก้ไฟล์ / สร้างไฟล์ ทำได้ตามปกติ — แต่ขั้นตอน commit/push ต้องหยุดถามก่อน
- ก่อนถาม ให้สรุปสั้นๆ ว่า: จะ commit ไฟล์ไหนบ้าง + commit message ว่าอะไร
  แล้วรอให้ user ยืนยันก่อนค่อยรัน

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
- ถ้า research ลึก ให้บอก user ว่าควรแยกไฟล์ไว้ใน `research/`
- ถ้ามี plan จาก plan mode ที่อยากเก็บ ให้ก๊อปลง `plans/YYYY-MM-DD-ชื่องาน.md`

## โครงสร้าง repo
- `daily/` — log รายวัน
- `research/` — research ลึกๆ แยกไฟล์ (ไม่มีวันที่ — ดูบริบทจาก daily)
- `plans/` — แผนงานเต็มๆ จาก plan mode
