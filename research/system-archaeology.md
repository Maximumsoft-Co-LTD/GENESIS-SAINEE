# System Archaeology

**Daily log:** [2026-06-09](../daily/2026-06-09.md)

---

## คำถามที่ต้องการตอบ
- โครงสร้างของ project ที่ไม่มีเอกสารเป็นยังไง?
- business flow หลักคืออะไร?
- service แต่ละตัว communicate กันยังไง?

---

## คืออะไร
เทคนิค reverse-engineer โครงสร้าง project จาก code โดยไม่ต้องพึ่งเอกสาร  
Claude จะอ่าน code แล้ววิเคราะห์ flow, dependency, business logic พร้อมกันด้วย multi-agent

---

## วิธีใช้ + ผลที่ได้

| ขั้นตอน | command/action | output |
|--------|---------------|--------|
| 1. ระบุ project | `/archaeology <project>` | — |
| 2. Claude วิเคราะห์ | multi-agent อ่าน code | — |
| 3. ได้ผลลัพธ์ | — | `.html` + `.md` |

**ตัวอย่างที่รัน:** projects 1,7,14,15,16 ใน monorepo  
**เวลาที่ใช้:** ~13.5 นาที (ช้าเพราะอ่านไฟล์ code เยอะ)

---

## เปรียบเทียบกับ pattern อื่น

| Pattern | ใช้เมื่อ | Output |
|---------|---------|--------|
| Archaeology | ต้องการเข้าใจ project เดียวลึกๆ | `.html` + `.md` ต่อ project |
| Control Plane | ต้องการ overview หลาย repo พร้อมกัน | dashboard รวม |
| Analyze Flow | ต้องการดู business flow เฉพาะจุด | flow diagram |

---

## ข้อสังเกต / Gotchas
- ช้าเพราะ Claude ต้องอ่านไฟล์ code จำนวนมาก
- ควรขอ `.md` ควบคู่กับ `.html` เสมอ เพราะรอ HTML นานและดูในกรณีฉุกเฉินได้เร็วกว่า
- ใช้คู่กับ `/control-plane` เพื่อได้ทั้ง overview และ deep-dive

---

## นำไปใช้
- [x] วิเคราะห์ 5 projects ใน monorepo → ได้ `archaeology_5projects.html` + `.md`
- [ ] ลองใช้กับ `indo/` sub-monorepo
