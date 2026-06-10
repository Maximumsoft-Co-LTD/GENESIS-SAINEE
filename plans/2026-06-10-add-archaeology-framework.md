# Plan: เพิ่ม 5-Prompt Framework ลง research/system-archaeology.md

## Context
ผู้ใช้ได้รับเฟรมเวิร์ก "4-Layer Top-Down System Archaeology" (5 prompts สำเร็จรูป)
สำหรับขุดโครงสร้าง legacy code เพื่อ migrate/maintain ระบบเก่า — เหมาะกับ monorepo
15 projects ที่ทีมกำลังดูแล ผู้ใช้ต้องการเก็บเฟรมเวิร์กนี้ **ตามต้นฉบับ** ลงไฟล์
research ที่มีอยู่ พร้อม **แทรกคอมเมนต์/ข้อเสนอแนะของผม** เข้าไปด้วย เพื่อให้
ไฟล์เดียวมีทั้ง "วิธีทำมือ" และ "ข้อควรระวังเฉพาะ context ของทีม"

ปัจจุบัน `research/system-archaeology.md` มีแค่วิธีใช้ `/archaeology` skill แบบ
อัตโนมัติ — ยังขาดเฟรมเวิร์กทำมือทีละ prompt ที่ใช้ได้แม้ใน chat เปล่าๆ ไม่มี
filesystem access

## ไฟล์ที่แก้
- `sainee-logs/research/system-archaeology.md` (ไฟล์เดียว, append section ใหม่)

## โครงสร้างที่จะเพิ่ม (ต่อท้าย section "นำไปใช้" เดิม)

เพิ่ม section ใหม่ `## 5-Prompt Framework (Top-Down ทำมือ)` ประกอบด้วย:

1. **คำนำสั้นๆ** — บอกว่าเฟรมเวิร์กนี้เหมาะกับ "ป้อน code เอง / chat ไม่มี
   filesystem" ส่วนใน Claude Code ที่มี `/archaeology` ให้ใช้เป็น
   **checklist ว่า output ต้องครอบคลุมอะไร**

2. **5 Layers ตามต้นฉบับ (verbatim)** — คงเนื้อหา prompt ทั้ง 5 ไว้ครบ:
   - Layer 1: สแกนโครงสร้าง + Entry Points (Prompt 1)
   - Layer 2: API Contracts & Data Flow (Prompt 2)
   - Layer 3: Business Logic & Edge Cases (Prompt 3)
   - Layer 4: Data & Integration (Prompt 4)
   - Layer 5: Keep/Drop Summary (Prompt 5)
   - + ทริก 1 Repo / 1 Chat

   แต่ละ Layer เก็บ "สิ่งที่ต้องป้อน" + ตัว prompt ไว้ในรูป code block เพื่อ
   ก๊อปไปใช้ได้ทันที

3. **คอมเมนต์ผม** — แทรกเป็น blockquote callout (`> 💬 ความเห็น:`) ใต้ layer
   ที่เกี่ยวข้อง:
   - ใต้คำนำ → จุดแข็งโดยรวม + "ของคุณมี `/archaeology` แล้ว ใช้เป็น checklist"
   - **Layer 0 ใหม่ (แทรกก่อน Layer 1)** → Runtime/Ops reality: env vars,
     Dockerfile/k8s, config dev/uat/prod ต่างกัน
   - ใต้ Layer 2 → เพิ่ม async paths (RabbitMQ/Redis consumer/producer,
     callback) — เคสทีมมี queue เยอะ
   - ใต้ Layer 3 → ระวัง pseudocode ภาษาไทยกำกวม → เสริม decision table /
     given-when-then
   - ใต้ Layer 5 → ย้ำว่าเป็นขั้นเปลี่ยน "อ่านโค้ด" → "ตัดสินใจสถาปัตยกรรม"

## หลักการเขียน
- เนื้อหา prompt ต้นฉบับ = คงไว้ ไม่ตัดทอน
- คอมเมนต์ผม = แยกให้เห็นชัดด้วย `>` blockquote + emoji 💬 เพื่อไม่ปนกับต้นฉบับ
- ใช้ภาษาไทยตลอด ให้สอดคล้องกับไฟล์เดิม

## ขั้นตอนหลังแก้ไฟล์
- commit: `git add research/system-archaeology.md && git commit -m "research: add 5-prompt top-down archaeology framework + comments"`
- README ไม่ต้องแก้ (link ไป system-archaeology.md มีอยู่แล้ว)

## Verification
- เปิด `research/system-archaeology.md` ตรวจว่า:
  - 5 prompts ครบ ก๊อปจาก code block ไปใช้ได้
  - คอมเมนต์ผมแยกชัดจากต้นฉบับ (blockquote)
  - Layer 0 อยู่ก่อน Layer 1
- `git log --oneline -1` เห็น commit ล่าสุด
