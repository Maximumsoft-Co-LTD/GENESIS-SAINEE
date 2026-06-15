---
name: architecture-advisor
description: วิเคราะห์ความเสี่ยงของทุก repo ในโฟลเดอร์แบบละเอียด (risk register พร้อมหลักฐาน file:line ทุกข้อ), สรุปภาพรวม architecture ปัจจุบันของ repo ที่เชื่อมกัน, แล้ว **เสนอ architecture ใหม่หลายทางเลือกพร้อม trade-off + migration roadmap** จบในเอกสารชุดเดียว ใช้เมื่อผู้ใช้ขอ "ดู/list risk ทั้งหมดของทุก repo", "อยากเห็นภาพรวม architecture ของระบบ", "แนะนำ/ออกแบบ architecture ใหม่", "ควร rewrite/migrate/ปรับโครงสร้างยังไง", หรือถามว่าระบบนี้ควรไปทางไหน — รองรับทั้งการวิเคราะห์ใหม่ทั้งหมดและการ reuse ผลเดิมเมื่อมี — **ออกแบบสถาปัตยกรรมเป้าหมายและแผนย้าย** ไม่ได้หยุดแค่แผนที่ระบบเดิม
---

ทำเอกสารชุดเดียวที่ตอบ 3 คำถาม: **(1) ระบบนี้มีความเสี่ยงอะไรบ้าง? (2) ตอนนี้หน้าตาเป็นยังไง? (3) ควรไปทางไหนต่อ?** — โดยข้อ 3 ต้องตั้งอยู่บนหลักฐานจากข้อ 1–2 ไม่ใช่ความเห็นลอยๆ

## หลักการที่ทำให้เอกสารนี้เชื่อถือได้

เอกสารนี้จะถูกใช้ตัดสินใจเรื่องใหญ่ (rewrite/migrate/ลงทุนคน) ดังนั้นกฎเหล็กคือ:

- **ทุก risk และทุกเส้นความสัมพันธ์ต้องมีหลักฐาน `file:line` จากโค้ดที่ทำงานจริง** — ไม่ใช่จากชื่อไฟล์ README comment หรือ "ระบบแบบนี้ปกติต้องเป็นแบบนี้" หลักฐานเดียวที่ผิดทำให้ทั้งเอกสารเชื่อไม่ได้
- **แยก "รู้แน่" ออกจาก "เดา" เสมอ** ใช้ระดับความเชื่อมั่น 🟢 ยืนยันสองฝั่ง / 🟡 หลักฐานฝั่งเดียว / ⚪ ข้อสงสัย (เขียนได้แต่ห้ามวาดเป็นเส้นใน diagram)
- **ข้อเสนอ architecture ใหม่ต้อง trace กลับไปหา risk หรือ pain point ที่มีหลักฐานได้เสมอ** — ทุกการเปลี่ยนแปลงต้องตอบได้ว่า "แก้ปัญหาข้อไหนที่เจอใน Phase 3"

---

## ก่อนเริ่ม — ตัดสินใจ fresh หรือ reuse (เช็คครั้งเดียว แล้วเดินหน้า)

**ขั้นแรก (บังคับ):** เช็ค 2 เงื่อนไขนี้ตามลำดับ:

1. **args มีคำว่า "เริ่มใหม่" / "fresh" / "ไม่ reuse" / "start fresh"** → **หยุดที่นี่ ไปทำ Phase 0 เลย** ไม่ต้องเช็คอะไรต่อ
2. **ไม่มีไฟล์ `docs/architecture-advisor/01-current-state.md`** → ไปทำ Phase 0 เลย (ไม่ต้องเช็คอะไรเพิ่ม)

ถ้าผ่านทั้งสองเงื่อนไข (args ปกติ + ไฟล์มีอยู่) ค่อยเทียบ commit hash ที่หัวไฟล์กับ `git rev-parse --short HEAD`:
- **ตรง** → ข้ามไป Phase 4 ได้เลย (Phase 1–3 ยังใช้ได้)
- **ไม่ตรง** → ทำ Phase 0–3 ใหม่ทั้งหมด

ถ้ามี `docs/repo-groups.md` ให้**ใช้เป็น grouping ตั้งต้น (canonical)** ใน Phase 2 — แต่ยังต้องตรวจว่า code evidence ขัดกับ grouping นี้ตรงไหน (boundary repo, orphan) และ reconcile เลขรวม repo

---

## Phase 0 — Scope & Census (บังคับ ทำก่อนทุกอย่าง)

นับ repo จริงทุกโฟลเดอร์ (ไม่นับ `.claude`, `docs`, โฟลเดอร์ที่ไม่มีโค้ด) บันทึกขั้นต่ำต่อ repo: ชื่อ, stack/ภาษา, entrypoint, ประเภท (frontend / API service / worker-consumer / cronjob / gateway / mobile / lib)

**เลขที่นับได้คือเลขตรวจสอบของทุก Phase ถัดไป** — ตาราง inventory, จำนวน risk ที่ครอบคลุม, จำนวนสมาชิกใน target design ต้องอ้างกลับมาที่เลขนี้ ถ้าไม่ครบแปลว่าหล่น

ถ้า repo เยอะ (>15) หรือ user ระบุขอบเขต ให้สรุป scope 2–3 บรรทัดก่อน: จะลงลึกทุก repo หรือโฟกัสกลุ่มที่เงิน/ข้อมูลสำคัญไหลผ่าน และอะไร out of scope รอบนี้

---

## Phase 1 — สกัดหลักฐานทีละ repo (Interface Card + Observations)

งานนี้แยกอิสระต่อ repo — ใช้ **`Workflow` tool** สร้าง fan-out script ด้วย `pipeline()` (batch 2–3 repo ต่อ agent ตามขนาด) **ไม่ใช้ `Agent` tool ตรงๆ** เพราะ 15+ calls พร้อมกันทำให้ permission prompt ท่วม และขาด resume/progress capability ถ้าถูกหยุดกลางทาง

ตัวอย่างโครงสร้าง Workflow script สำหรับ Phase 1:

```javascript
export const meta = {
  name: 'arch-advisor-p1',
  description: 'Extract Interface Cards for all repos',
  phases: [{ title: 'Extract Cards' }],
}
// REPOS มาจาก Phase 0 census — ส่งใน args หรือ hardcode หลัง Phase 0
const cards = await pipeline(
  REPOS,
  repo => agent(`Extract Interface Card for ${repo}: <card instructions>`, {
    label: `card:${repo}`,
    phase: 'Extract Cards',
  })
)
// แต่ละ agent เขียน card ลง docs/architecture-advisor/cards/<repo>.md ด้วย Write tool
```

แต่ละ agent ใน Workflow อ่านไฟล์จริงแล้วเขียน card ในรูปแบบนี้ลง `docs/architecture-advisor/cards/<repo>.md`:

```
## Card: <repo>
identity: stack | module/entrypoint | port (จากไฟล์ ไม่ใช่เดา) | Dockerfile/CI

provides:                       # เปิดอะไรให้คนอื่นใช้
  http:    METHOD /path | auth middleware | file:line   (จากจุด register route จริง)
  consume: exchange/queue/channel ที่ subscribe | handler | file:line
  cron:    schedule | file:line

consumes:                       # ไปใช้อะไรของคนอื่น
  http-out: env key + base URL + path ที่เรียก | file:line   (จาก client code จริง)
  publish:  exchange/queue ที่ส่งเข้า | file:line
  data:     DB + collection/table | R/W | file:line
  external: third-party (bank/payment/LINE/SMS/S3...) | file:line

observations:                   # ของแปลกที่เห็นตอนอ่าน — จดดิบๆ พร้อม file:line ยังไม่ต้องตัดสิน
  - secret hardcode / no timeout / error ถูกกลืน / โค้ดซ้ำ repo อื่น / dep เก่า ...
```

กฎ: ทุกบรรทัดมี `file:line` จากจุดที่โค้ด**ทำจริง** ระวัง false positive จาก test / โค้ด comment ทิ้ง / dead code (ถ้าไม่ชัดติดธง 🟡) และช่อง `observations` ให้จด**ทุกอย่างที่สะดุดตา** — การคัด/จัดระดับเป็นงานของ Phase 3 (ถ้าไม่จดตอนนี้ต้องกลับมาอ่านไฟล์ซ้ำ)

---

## Phase 2 — จับคู่ความสัมพันธ์ → Current Architecture (ทำที่ main agent)

**เริ่มด้วยการโหลด cards:** อ่าน card ทุกไฟล์ใน `docs/architecture-advisor/cards/` กลับเข้า context ด้วย Read tool ก่อนทำขั้นตอนใดๆ — ห้าม match จากความจำ ต้อง match จากข้อมูลในไฟล์เท่านั้น

เส้นเกิดได้ก็ต่อเมื่อ **Consumes ของฝั่งหนึ่ง match กับ Provides ของอีกฝั่ง**:

| ชนิดเส้น | เงื่อนไข 🟢 | ถ้าได้ฝั่งเดียว → 🟡 |
|----------|-----------|---------------------|
| api | `http-out` ของ A มี path ตรงกับ `provides.http` ของ B | env key สื่อชื่อแต่เทียบ path ไม่ได้ |
| event | `publish` ของ A ชื่อ exchange/queue ตรงกับ `consume` ของ B (เผื่อ suffix เช่น `_<TENANT>`) | เจอ publisher แต่ไม่เจอ consumer หรือกลับกัน |
| shared-data | ≥2 repo ประกาศ DB+collection เดียวกัน และอย่างน้อยหนึ่งฝั่ง W | ชื่อ DB สื่อ domain เดียวกันแต่ collection ไม่ตรง |

ของที่**จับคู่ไม่ติดห้ามทิ้ง**: `consumes` ที่ไม่มีใครรับ → external/นอกโฟลเดอร์ ; `provides` ที่ไม่มีใครเรียก → endpoint ตาย? caller นอกโฟลเดอร์? บันทึกเป็น Unknown

ส่งมอบ Phase นี้:
1. **Edge list กลาง** (from | to | ชนิด | ชื่อจริง path/queue/DB | หลักฐานสองฝั่ง | conf)
2. **Grouping** — เลือกวิธีตามที่มี:
   - **มี `docs/repo-groups.md`** → ใช้เป็น canonical grouping ตั้งต้น แล้วตรวจกับ edge จริง (boundary repo, orphan)
   - **ไม่มี `docs/repo-groups.md`** → derive จาก cluster ของ edge 🟢: repo ที่แชร์ queue/DB เดียวกันหรือเรียก API กันบ่อยคือ cluster เดียว → ตั้งชื่อกลุ่มชั่วคราวตาม business domain → **เสนอ grouping นี้ให้ user ยืนยันใน chat ก่อนเข้า Phase 3** เพราะ Phase 3 รายงานรายกลุ่ม ถ้ากลุ่มผิดต้องทำซ้ำ
3. **Current architecture diagram (Mermaid)** — subgraph ต่อกลุ่ม, label เส้นด้วยของจริง (`POST /api/...`, ชื่อ queue, `DB:ชื่อ`), เส้นทึบ=sync เส้นประ=async, external=สไตล์ ext
4. **Hub / SPOF** — repo ที่เส้นเข้าออกเยอะสุด ถ้าตายอะไรตายตาม

---

## Phase 3 — Risk Audit แบบละเอียด (หัวใจของคำขอ "list risk ทั้งหมด")

เอา `observations` ดิบจากทุก card มาคัด **แล้วไล่เช็คเพิ่มทุกหมวดตาม checklist** — ใช้ Read tool อ่าน `references/risk-checklist.md` ก่อนทำ Phase นี้ เพราะมันคือรายการสิ่งที่ต้องไปดูจริงในโค้ดต่อหมวด (Security / Reliability / Data / Maintainability / Observability) พร้อมตัวอย่าง pattern ที่อันตราย

รายงานทุก risk เป็นตารางเดียวรวมทั้งระบบ + แยกย่อยรายกลุ่ม:

```
# | Repo/กลุ่ม | ความเสี่ยง | หมวด | ระดับ | conf | หลักฐาน file:line | ผลกระทบถ้าเกิด | ข้อเสนอแก้
```

- **ระดับ:** 🔴 เงินหาย/โดน exploit ได้ | 🟠 ระบบล่ม/ข้อมูลเพี้ยน | 🟡 debt ที่ควรแก้
- **conf:** 🟢 ยืนยันจากโค้ด | 🟡 หลักฐานฝั่งเดียว | ⚪ ข้อสงสัย
- เรียงตามระดับ (🔴 ก่อน) แล้วตามผลกระทบ

และต้องมีอีกสอง section เพราะบอกคนอ่านว่า "เช็คแล้ว" ต่างจาก "ลืมเช็ค":
- **✅ ตรวจแล้วผ่าน** — จุดที่ไล่ดูแล้วเขาทำถูก (เช่น มี idempotency key จริงที่ file:line) — สำคัญเพราะกันการเสนอแก้สิ่งที่ไม่พัง
- **⚪ ข้อสงสัย (ยังไม่ยืนยัน)** — น่าจะเสี่ยงแต่หลักฐานไม่พอ + ต้องไปดูที่ไหนถึงยืนยันได้ (เช่น k8s manifest นอก repo)

> เป้าหมายของ Phase นี้คือ "ละเอียดที่สุดเท่าที่หลักฐานรองรับ" — list ให้ครบ อย่ากรองออกเพราะดูเล็ก แต่จัดระดับให้ถูกเพื่อให้คนอ่านโฟกัสถูกจุด

---

## Phase 4 — วินิจฉัยปัญหาเชิงสถาปัตยกรรม (สะพานจาก risk → redesign)

risk ใน Phase 3 คืออาการ — Phase นี้หา**โรค**: รวมอาการที่ซ้ำๆ ให้เป็นปัญหาเชิงโครงสร้าง เพื่อให้ข้อเสนอใหม่แก้ที่รากไม่ใช่ที่ปลายเหตุ ไล่ดูอย่างน้อย:

- **Coupling/SPOF** — hub ที่ทุกอย่างวิ่งผ่าน, service ที่ตายแล้วลามทั้งระบบ
- **Shared database** — หลาย service เขียน collection/table เดียวกัน (ใครเป็นเจ้าของข้อมูล?)
- **โค้ด/logic ซ้ำข้าม repo** — แก้ที่เดียวอีกหลายที่ค้าง (เช่น JWT verify, response helper, payment callback ลอกกัน)
- **ขอบเขต service ไม่ตรง domain** — service เดียวทำหลายเรื่องไม่เกี่ยวกัน หรือ domain เดียวกระจายหลาย service
- **Sync chain ยาว/เปราะ** — A→B→C→D แบบ sync ที่ latency/ความล่มสะสม
- **ความไม่สม่ำเสมอ** — auth/observability/error handling ที่แต่ละ repo ทำคนละแบบ

สรุปเป็นตาราง: `ปัญหาเชิงโครงสร้าง | อาการ/risk ที่โยงถึง (อ้าง # จาก Phase 3) | ผลกระทบต่อการพัฒนา/ดำเนินงาน`

**บันทึกก่อนถาม (สำคัญ):** เขียน `01-current-state.md`, `02-risks.md`, `03-diagnosis.md` ลงดิสก์ให้เสร็จก่อน — แล้วค่อยถาม constraints ใน chat ป้องกันงานหายถ้า session หยุดหรือ context compress ระหว่างรอ user ตอบ

จากนั้นสกัด **Architecture Drivers** — เกณฑ์ที่จะใช้ตัดสินใจเลือกแบบใน Phase 5 (เช่น ต้องรองรับ tenant เพิ่ม, ลดเวลา dev ฟีเจอร์ใหม่, เงินต้องไม่หาย, ทีมเล็ก) ถ้า user ไม่ได้บอก constraints (ขนาดทีม, timeline, budget, ห้ามดาวน์ไทม์ไหม) **ให้ถามก่อนเข้า Phase 5** เพราะมันเปลี่ยนคำตอบทั้งหมด

---

## Phase 5 — เสนอ Architecture ใหม่ (หลายทางเลือก + trade-off)

ใช้ Read tool อ่าน `references/target-patterns.md` ก่อน — มันคือ catalog ของ pattern (modular monolith, service grouping, event-driven, shared platform/libs, strangler-fig migration ฯลฯ) ว่าแต่ละแบบเหมาะ/ไม่เหมาะกับ driver แบบไหน และวิธีให้คะแนน

เสนอ **2–3 ทางเลือกที่ต่างกันจริง** (ไม่ใช่แบบเดียวที่ปรับนิดหน่อย) แต่ละทางเลือกมี:

1. **ชื่อ + แนวคิดหลัก** (1–2 ประโยค) — เช่น "รวมเป็น modular monolith", "จัดกลุ่มเหลือ N service ตาม domain + shared platform lib", "คงไว้แต่เพิ่ม event backbone"
2. **Target diagram (Mermaid)** — หน้าตาระบบหลังทำ + map ว่า repo เดิมแต่ละตัวไปอยู่ตรงไหน (รวม/แยก/คงเดิม/ทิ้ง)
3. **แก้ปัญหาข้อไหน** — โยงกลับ Phase 4 ทุกข้อ (อันไหนแก้ได้ อันไหนแก้ไม่ได้)
4. **Trade-off** — ตารางข้อดี/ข้อเสีย/ความเสี่ยงของการย้าย/effort คร่าวๆ (S/M/L)
5. **เหมาะเมื่อ** — เงื่อนไขที่ทำให้ทางเลือกนี้คุ้ม (เช่น "ถ้าทีม <5 คน")

ปิดด้วย **ตารางให้คะแนนเทียบกัน** ตาม Architecture Drivers จาก Phase 4 (ให้น้ำหนัก driver ตามที่ user สนใจ) แล้ว **เลือก 1 ทางเลือกที่แนะนำ** พร้อมเหตุผล — การมีหลายทางเลือกไม่ได้แปลว่าไม่ฟันธง คนอ่านต้องการคำแนะนำที่ชัด แต่เห็นทางที่ไม่ได้เลือกด้วยว่าทำไมถึงไม่เลือก

---

## Phase 6 — Migration Roadmap (ของทางเลือกที่แนะนำ)

แตกเป็น **เฟสที่ส่งมอบทีละชิ้นได้** เรียงตามหลักการ: **risk 🔴 ที่แก้ได้เร็วและคุ้มที่สุดมาก่อน, เปลี่ยนของที่ทุบความเปราะ/SPOF ก่อนของที่แค่สวยขึ้น, แต่ละเฟสปล่อยจริงได้โดยไม่ต้องรอเฟสหลัง**

ต่อเฟส: `ทำอะไร | แก้ risk/ปัญหาข้อไหน | effort | ความเสี่ยงตอนย้าย + วิธีลด (เช่น strangler-fig, dual-write, feature flag) | Definition of Done`

เพิ่ม:
- **Quick wins** — สิ่งที่แก้ได้ทันทีไม่ต้องรอ redesign (เช่น ใส่ timeout, ถอน secret ออกจาก repo) แยกออกมาเพราะทำได้พรุ่งนี้
- **ของที่ต้องตัดสินใจก่อนเริ่ม** — open questions / สิ่งที่ต้องยืนยันจากทีม (โยงกับ ⚪ ใน Phase 3)

---

## Output

```
docs/architecture-advisor/
├── README.md            # หน้าปก: สรุปผู้บริหาร + ลิงก์ทุกหน้า + top risks + ทางเลือกที่แนะนำ
├── 01-current-state.md  # Phase 2: inventory + current diagram + hub/SPOF
├── 02-risks.md          # Phase 3: risk register เต็ม + ตรวจแล้วผ่าน + ข้อสงสัย
├── 03-diagnosis.md      # Phase 4: ปัญหาเชิงโครงสร้าง + architecture drivers
├── 04-target-options.md # Phase 5: ทางเลือก + trade-off + ตารางคะแนน + ทางที่แนะนำ
├── 05-roadmap.md        # Phase 6: migration roadmap + quick wins + open questions
└── cards/<repo>.md      # Phase 1: interface card ดิบ (หลักฐานอ้างอิง)
```

- หัวไฟล์ทุกไฟล์ระบุ **วันที่วิเคราะห์ + commit hash** (`git rev-parse --short HEAD`) — คนอ่านทีหลังต้องรู้ว่าหลักฐานอิงโค้ด ณ จุดไหน
- README ต้องอ่านจบใน 2 นาทีแล้วเห็นภาพ: ระบบมี N repo, ความเสี่ยง 🔴 กี่ข้อ, ปัญหาหลักคืออะไร, แนะนำให้ไปทางไหน, เริ่มจากอะไร
- ปิดท้ายใน chat: top 3 risks, ทางเลือกที่แนะนำ + เหตุผลสั้นๆ, quick wins, และจุดที่อยากให้คนรีวิว/ตัดสินใจต่อ
- ถ้า user ตั้ง constraints มาแล้ว (ทีม/timeline/budget) ให้สะท้อนใน Phase 5–6 ถ้ายังไม่ตั้ง ให้ถามใน Phase 4 ก่อนเสนอแบบ
