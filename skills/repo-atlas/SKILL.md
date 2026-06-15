---
name: repo-atlas
description: สร้าง "แผนที่ระบบ" (atlas) ของ repo ทั้งหมดในโฟลเดอร์ด้วยวิธี Interface Card + Two-sided Matching — สกัด contract ของแต่ละ repo ว่าเปิดอะไรให้ใช้/ไปเรียกอะไรของใคร แล้วจับคู่หลักฐานสองฝั่งให้ได้ relationship ที่ยืนยันจากทั้งต้นทางและปลายทาง จากนั้นจัด group, วาด diagram ละเอียดรายกลุ่ม, และทำ risk register รายกลุ่มพร้อมหลักฐาน file:line ทุกข้อ ใช้เมื่อผู้ใช้ขอวิเคราะห์ความสัมพันธ์ของ repo/service ทั้งหมด, แยกกลุ่ม/domain, หาความเสี่ยงรายกลุ่ม, ทำแผนผังระบบก่อน migrate/rewrite หรือถามว่า "service ไหนคุยกับ service ไหน"
---

สร้างแผนที่ระบบ (atlas) ของ repo ทั้งหมดในโฟลเดอร์นี้: ภาพรวมทั้งระบบ → จัดกลุ่ม → แผนที่ละเอียดรายกลุ่ม → ความเสี่ยงรายกลุ่ม

## แนวคิดหลัก: เส้นต้องเกิดจากการจับคู่สองฝั่ง

ความผิดพลาดที่อันตรายที่สุดของการ map ระบบคือ "เส้นปลอม" — ลากเส้นเพราะชื่อมันคล้าย หรือเพราะระบบแบบนี้ "ปกติต้องคุยกัน" เอกสารนี้จะถูกใช้ตัดสินใจตอน migrate เส้นปลอมเส้นเดียวทำให้ทั้งเอกสารเชื่อไม่ได้

วิธีของ skill นี้จึงไม่ใช่การไล่ grep หา URL ทั่วโฟลเดอร์ แต่คือ:

1. มองแต่ละ repo เป็น **สัญญาสองด้าน** — สกัดเป็น Interface Card: ด้าน **Provides** (เปิดอะไรให้คนอื่นใช้) และด้าน **Consumes** (ไปเรียกใช้อะไรของใคร)
2. เอา card ทุกใบมา **จับคู่ตรงกลาง** — เส้นเกิดก็ต่อเมื่อ Consumes ของฝั่งหนึ่ง match กับ Provides ของอีกฝั่ง

การเดาจึงถูกกันด้วยโครงสร้างของวิธีการเอง: เส้นที่ไม่มีหลักฐานทั้งสองฝั่งจะ "จับคู่ไม่ติด" โดยอัตโนมัติ และของที่จับคู่ไม่ติดก็ไม่ใช่ขยะ — มันคือรายการ unknown ที่มีค่า (caller ภายนอก? endpoint ตาย? service ที่อยู่นอกโฟลเดอร์?)

ระดับความเชื่อมั่น ใช้กำกับทุกเส้นและทุก risk:
- 🟢 **สองฝั่งยืนยัน** — ฝั่งเรียกมีโค้ดเรียกจริง และฝั่งรับมี route/consumer ชื่อ-path ตรงกันจริง
- 🟡 **ฝั่งเดียว** — เจอหลักฐานแค่ด้านเดียว (เช่น env key ชี้ไป แต่ยังไม่เทียบ route ปลายทาง, ค่าที่ inject ตอน runtime)
- ⚪ **ข้อสงสัย** — ตั้งข้อสังเกตได้แต่หลักฐานไม่พอ → เขียนลง section "ข้อสงสัย" เท่านั้น **ห้ามวาดเป็นเส้นใน diagram**

---

## Phase 1 — สำรวจสำมะโน (Census)

นับ repo จริงทุกโฟลเดอร์ (ไม่นับ `.claude`, `docs`, โฟลเดอร์ที่ไม่มีโค้ด) แล้วบันทึกขั้นต่ำต่อ repo: ชื่อ, stack/ภาษา, entrypoint, ประเภทคร่าวๆ (frontend / API service / worker-consumer / cronjob / gateway)

จำนวนที่นับได้ในขั้นนี้คือ **เลขตรวจสอบ** ของทุกขั้นถัดไป — ตาราง inventory, จำนวน card, จำนวนสมาชิกรวมทุก group ต้องเท่ากับเลขนี้เสมอ ถ้าไม่เท่าแปลว่าหล่นหาย

## Phase 2 — สกัด Interface Card ทีละ repo

หัวใจของทั้ง skill — งานนี้แยกอิสระต่อ repo จึง**ควร spawn subagent ขนานกัน** (1 ตัวต่อ repo หรือ batch ละ 3–5 repo ตามขนาด) ให้แต่ละ subagent อ่านไฟล์จริงแล้วคืน card ตาม template:

```
## Card: <repo>
identity: stack | module name | entrypoint | port (จากไฟล์ ไม่ใช่เดา) | Dockerfile/CI

provides:                        # เปิดอะไรให้โลกใช้
  http:    METHOD /path | auth middleware อะไร | หลักฐาน file:line   (จากจุด register route จริง)
  consume: exchange/queue/channel ที่ subscribe | handler | file:line
  cron:    schedule ที่ตั้งเอง | file:line

consumes:                        # ไปใช้อะไรของคนอื่น
  http-out: env key + base URL + path ที่เรียก | file:line   (จาก client code จริง ไม่ใช่แค่ประกาศ env)
  publish:  exchange/queue ที่ส่งเข้า | payload คร่าวๆ | file:line
  data:     DB ชื่ออะไร + collection/table ไหน + อ่านหรือเขียน (R/W) | file:line
  external: third-party (bank, payment provider, LINE, SMS, S3, ...) | file:line

observations:                    # ของแปลกที่เห็นระหว่างอ่าน — จดดิบๆ พร้อม file:line ยังไม่ต้องตัดสิน
  - เช่น secret hardcode, HTTP client ไม่มี timeout, error ถูกกลืน, โค้ดซ้ำกับ repo อื่น
```

กฎของ card:
- ทุกบรรทัดต้องมี `file:line` — สกัดจากจุดที่โค้ด**ทำจริง** (จุด register route, จุดเรียก client, จุด publish) ไม่ใช่จากชื่อไฟล์, README, หรือ comment
- ระวัง false positive: โค้ดใน test, ค่าที่ถูก comment ทิ้ง, dead code ที่ไม่ถูกเรียก — ถ้าไม่แน่ใจให้ติดธง 🟡 มาใน card เลย
- ช่อง `observations` ให้จด**ทุกอย่างที่สะดุดตา**ตอนอ่าน — ตอนนี้ยังไม่ต้องกรอง การคัดและจัดระดับเป็นงานของ Phase 5 (ถ้าไม่จดตอนนี้ จะต้องกลับมาอ่านไฟล์ซ้ำรอบสอง)

## Phase 3 — จับคู่ (Matching)

งานนี้ทำที่ main agent (เป็นการ join ข้อมูล ไม่ใช่การอ่านโค้ดเพิ่ม):

| ชนิดเส้น | เงื่อนไขจับคู่ 🟢 | ถ้าได้แค่ฝั่งเดียว |
|----------|-------------------|---------------------|
| api | `http-out` ของ A มี path ตรงกับ `provides.http` ของ B | 🟡 — env key สื่อชื่อแต่ path ไม่เทียบได้ |
| event | `publish` ของ A ชื่อ exchange/queue ตรงกับ `consume` ของ B (เผื่อ pattern suffix เช่น `_<TENANT>`) | 🟡 — เจอ publisher แต่ไม่เจอ consumer หรือกลับกัน |
| shared-data | ≥2 repo ประกาศ DB+collection เดียวกัน และอย่างน้อยหนึ่งฝั่ง W | 🟡 — ชื่อ DB สื่อ domain เดียวกันแต่ collection ไม่ตรง |

จากนั้นวิเคราะห์ **ของที่จับคู่ไม่ติด** — ส่วนนี้ห้ามทิ้ง:
- `consumes` ที่ไม่มีใครรับ → external service หรือ service ที่อยู่นอกโฟลเดอร์ → ลงตาราง External
- `provides` ที่ไม่มีใครเรียก → endpoint ตาย? caller อยู่นอกโฟลเดอร์ (เช่น frontend ที่ inject URL ตอน deploy)? → ลงตาราง Unknown caller

ผลลัพธ์ Phase นี้: **edge list กลาง** (from | to | ชนิด | ชื่อจริงของ path/queue/DB | หลักฐานสองฝั่ง | conf) ที่ทุก Phase ถัดไปอ้างอิง

## Phase 4 — จัดกลุ่ม (Grouping)

จัดจากข้อมูล ไม่ใช่จากชื่อ repo:
- เริ่มจาก **cluster ของเส้น 🟢** — repo ที่มีเส้นถึงกันหนาแน่น (api + event + shared-data รวมกัน) คือกลุ่มธรรมชาติ
- ใช้ business domain เป็นตัวตั้งชื่อกลุ่มและตัดสินกรณีก้ำกึ่ง
- ทุก repo มี **primary group เดียว** — repo ที่เส้นกระจายหลายกลุ่ม (hub) ให้จัดตามหน้าที่หลัก แล้วติดป้าย **boundary repo** พร้อมระบุกลุ่มอื่นที่มันแตะ
- repo ที่เส้นน้อยจนจัดไม่ได้ → กลุ่ม "ยังไม่ชัด" พร้อมระบุว่าขาดหลักฐานอะไร — การยอมรับว่าไม่รู้ดีกว่ายัดเข้ากลุ่มเพราะชื่อคล้าย

ส่งมอบ: ตาราง Group | สมาชิก | เหตุผลที่จัด (อ้าง edge จริง) | boundary repos

## Phase 5 — แผนที่รายกลุ่ม (Atlas page ต่อกลุ่ม)

ผลิต 1 หน้า ต่อ 1 กลุ่ม (ขนานด้วย subagent ต่อกลุ่มได้ — ส่ง card ของสมาชิก + edge list ที่เกี่ยวข้องให้):

**(ก) บทบาทของกลุ่ม** — รับผิดชอบอะไร, **เงินไหลผ่านไหม** (ตัวคูณความเสี่ยง), สมาชิกแต่ละตัวทำหน้าที่อะไร

**(ข) Diagram รายกลุ่ม (Mermaid)** — ละเอียดกว่าภาพรวมคนละระดับ:
- ทุกเส้นภายในกลุ่ม label ด้วย**ของจริง**: `POST /api/v2/...`, ชื่อ exchange/queue จริง, `DB: ชื่อจริง` — ห้ามใช้คำลอยๆ อย่าง "calls API"
- เส้นทึบ = sync HTTP, เส้นประ = async, repo นอกกลุ่มที่ interface ด้วย = node สีจางขอบเส้นประ, external = สไตล์ ext
- flow ที่**ลำดับเวลาสำคัญ** (callback จาก provider, การ approve แล้วยิง queue) ให้เพิ่ม `sequenceDiagram` แยกอีกอัน เพราะ graph บอกได้แค่ว่าใครคุยกับใคร แต่บอกไม่ได้ว่าอะไรเกิดก่อนหลัง

**(ค) ตาราง edge ภายในกลุ่ม** — ยกจาก edge list กลาง พร้อมหลักฐานสองฝั่ง

**(ง) Flow สำคัญ 1–3 เส้น** — เลือกเส้นที่เงิน/ข้อมูลสำคัญไหลผ่าน เขียนทีละ hop: `→` sync, `⟿` queue, `↩` callback, `💾` DB write

**(จ) Risk Register** — เอา `observations` ดิบจาก card ของสมาชิกมาคัด + ไล่เช็คเพิ่มตาม checklist:

| หมวด | สิ่งที่ต้องไล่ดูจริงในโค้ด |
|------|---------------------------|
| Security | secret/credential ติด repo, crypto อ่อน (MD5/SHA1/ECB), endpoint ธุรกรรมที่ไม่มี auth, callback ที่ไม่ verify signature, JWT secret ใช้ซ้ำข้าม service |
| Reliability | HTTP client ไม่มี timeout, consumer ไม่มี retry/DLQ/ack, callback ซ้ำแล้วเงินเบิ้ล (idempotency), จุดที่ตายแล้วลามทั้งกลุ่ม |
| Data | หลาย repo เขียน collection เดียวกัน, ย้ายเงินนอก transaction, race ตอน update balance พร้อมกัน |
| Maintainability | โค้ดซ้ำข้าม repo (แก้ที่เดียวอีกที่ค้าง), dependency เก่า, ไม่มี test, dead code หลอกคนอ่าน |
| Observability | error ถูกกลืน, จุดเงินไหลที่ไม่มี log — เงินหายแล้ว trace ไม่ได้ |

ทุกข้อรายงานเป็น: `# | ความเสี่ยง | ระดับ | หลักฐาน file:line | ผลกระทบถ้าเกิด | ข้อเสนอ`
ระดับ: 🔴 เงินหาย/โดน exploit ได้ | 🟠 ระบบล่ม/ข้อมูลเพี้ยน | 🟡 debt

และอีกสอง section ที่บังคับมี เพราะบอกคนอ่านว่า "เช็คแล้ว" ต่างจาก "ลืมเช็ค":
- **ตรวจแล้วผ่าน** — จุดที่ไล่ดูแล้วเขาทำถูก (เช่น มี idempotency key จริง ที่ file:line)
- **ข้อสงสัย (ยังไม่ยืนยัน)** — น่าจะเสี่ยงแต่หลักฐานไม่พอ + ต้องไปดูที่ไหนถึงจะยืนยันได้ (เช่น k8s manifest นอก repo)

**(ฉ) Unknown ของกลุ่ม** — สิ่งที่ trace ไม่ถึงจาก repo ในมือ + แหล่งที่ต้องไปหาต่อ

## Phase 6 — หน้าปก (Synthesis)

1. ตาราง inventory ครบทุก repo (เทียบเลข Census)
2. **Global diagram**: subgraph ต่อกลุ่ม แสดง**เฉพาะเส้นข้ามกลุ่ม** — เส้นในกลุ่มอยู่ในหน้าของกลุ่มแล้ว ใส่ซ้ำตรงนี้ภาพจะรกจนใช้ไม่ได้
3. **Top risks ทั้งระบบ** — รวม 🔴 จากทุกกลุ่มไว้ที่เดียว เรียงตามผลกระทบ
4. **Hub / SPOF** — repo ที่เส้นเข้าออกเยอะสุด และถ้าตายอะไรตายตาม
5. **แผนที่ความไม่รู้** — Unknown caller, unmatched consumes, ค่าที่ inject ตอน runtime — เพื่อให้ทีมรู้ขอบเขตของหลักฐานชุดนี้

---

## Output

```
docs/repo-atlas/
├── README.md        # หน้าปก (Phase 6) + ลิงก์ไปทุกหน้า group
├── cards/<repo>.md  # Interface Card ดิบของแต่ละ repo (Phase 2) — เก็บไว้เป็นหลักฐานอ้างอิง
└── <group-slug>.md  # แผนที่รายกลุ่ม (Phase 5) ลิงก์กลับ README และลิงก์ข้ามไปกลุ่มที่ interface ด้วย
```

- หัวไฟล์ทุกไฟล์ระบุวันที่วิเคราะห์ + commit hash (`git rev-parse --short HEAD`) — คนอ่านทีหลังต้องรู้ว่าหลักฐานอิงโค้ด ณ จุดไหน
- ปิดท้ายใน chat: จำนวนกลุ่ม, top 3 risks, จุดที่อยากให้คนรีวิวต่อ
- ถ้ารันซ้ำ: เริ่มจาก Census ใหม่เสมอ (repo อาจเพิ่ม/ลด) แต่ card เดิมใช้เทียบ diff ได้ว่าอะไรเปลี่ยน
