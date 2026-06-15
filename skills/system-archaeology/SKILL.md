---
name: system-archaeology
description: ขุดวิเคราะห์ repo/service แบบ Top-Down System Archaeology 7 layers (deploy/ops, tech stack & entry points, API contracts & data flow, business logic & edge cases, data & integration, keep/drop summary, feature deep-dive) โดยอ่านไฟล์จริง ห้ามเดา — ใช้เมื่อต้องการเข้าใจระบบเดิมเชิงลึกก่อน migrate/rewrite หรือทำ knowledge transfer ของ service ใดservice หนึ่ง
---

Perform a Top-Down System Archaeology on `$ARGUMENTS` using the layers below **sequentially**.
For each layer, READ the relevant files first, then answer the questions.

ถ้า `$ARGUMENTS` ว่าง ให้ถามผู้ใช้ก่อนว่าจะวิเคราะห์ repo/service ไหน (หรือ path ใด)
ถ้า file ใดไม่มีอยู่ใน repo ให้ข้ามและระบุว่า "ไม่พบ" — ไม่ต้องหยุดรอ
Output: สรุปแต่ละ Layer แยก section ชัดเจน **ไม่ต้องสร้าง HTML**

**🎯 เป้าหมาย (Definition of Done):** เอกสารที่ได้ต้องทำให้ **นักพัฒนาคนใหม่อ่านแล้วแก้ฟีเจอร์ได้ โดยไม่ต้องไล่อ่านโค้ดทั้งระบบเอง** และทีมต้องเห็นชัดว่าจุดไหน "รู้แน่ (verified)" จุดไหน "ยังเป็นปริศนา (unknown)"

**หลักการ:** ไล่เป็นเฟสจาก **ภาพกว้าง → รายละเอียด** เสมอ (Scope → Structure → API → Data → Logic) — ห้ามกระโดดลงรายละเอียดก่อนกำหนดขอบเขต

---

## 🧭 Phase: Scope (ทำก่อนทุก Layer — บังคับ)

> กำหนดขอบเขตก่อน ไม่งั้นจะหลงทางในโค้ดเป็นแสนบรรทัด

ก่อนเริ่ม Layer 0 ตอบให้ชัด (ถ้าไม่ชัดให้ถามผู้ใช้):

- **โหมดไหน?** วิเคราะห์ **ทั้ง service** หรือ **เฉพาะฟีเจอร์/flow เดียว** (เช่น เฉพาะ "ฝากเงิน")
- **ลึกแค่ไหน?** survey เร็ว (Layer 0–2) หรือ deep-dive ครบ (ถึง Layer 6)
- **ขนาด repo** — ถ้าใหญ่ (>50k บรรทัด เช่น office-api-v10, 3rd-payment) ให้ **เลือก flow สำคัญ 3–5 เส้นเท่านั้น** ไม่ต้องครบทุก endpoint
- ระบุ **สิ่งที่จะ "ไม่" ครอบคลุม** ในรอบนี้ (out of scope) ให้ชัด

สรุป scope เป็น 2–3 บรรทัดก่อน แล้วค่อยลงมือ

---

## 📂 Layer 0: Runtime / Ops Reality
Read: `.env`, `Dockerfile`, `docker-compose.yml`, k8s manifests, config files

- ระบบนี้ deploy ยังไง?
- มี config ไหนที่เปลี่ยนพฤติกรรมตาม environment (dev/uat/prod)?
- มี secret หรือ connection string ชี้ไปที่ไหนบ้าง?
- **Blast Radius:** service นี้ถูกเรียกจาก service ไหนบ้าง? และเรียก service ไหนออกไปบ้าง? (upstream / downstream)

---

## 📂 Layer 1: สแกนโครงสร้างและระบุ Entry Points
Read: folder tree, `go.mod` / `package.json` / `pom.xml`, `main.go` / `main.js` / `main.py`

- Repo นี้ใช้ Tech Stack อะไร (เวอร์ชันไหน) และทำหน้าที่หลักอะไรในระบบ?
- จุดไหนคือ Entry Point (จุดเริ่มต้นการทำงาน)?
- มี Library หรือ Dependency ตัวไหนที่สำคัญ หรือดูเก่า/แปลกปลอมที่ควรระวัง?

---

## 🔄 Layer 2: API Contracts & Data Flow
Read: router files, controller files, handler files

สร้างตารางสรุป Endpoints ทั้งหมด:
| HTTP Method | Path | หน้าที่ (business) | Auth/Middleware | Input | Output | สถานะ (ใช้งาน/dead) | พร้อม migrate? |

Legend พร้อม migrate: ✅ copy ได้เลย | ⚠️ ต้องปรับ (ระบุสั้นๆ) | ❌ เขียนใหม่

**และ** สรุป Async paths:
| ประเภท | Topic/Queue/Key | รับ payload อะไร | ส่งต่อไปไหน |
(RabbitMQ consumer/producer, Redis pub/sub, cron job)

**Inter-Service Flows (top 3-5 user journeys สำคัญ):**
Read: service client files, HTTP call files, กลับไปดู Blast Radius จาก Layer 0

เลือก flows ที่ user ทำบ่อยที่สุด (เช่น login, deposit, withdraw, game-launch) แล้วเขียน call chain:

| Flow | Call Chain | หมายเหตุ |
|------|-----------|----------|

รูปแบบ call chain:
- `→` = synchronous HTTP call
- `⟿` = async (RabbitMQ / cron)
- `[POST /path]` = endpoint ที่ถูกเรียก

ตัวอย่าง:
`Frontend → [POST /login] → customer-api → wallet-service → Redis(set session) → response`
`[POST /deposit] → deposit-service ⟿ queue:notify → notify-gateway → Firebase`

---

## 🧠 Layer 3: Business Logic & Edge Cases
Read: service files, domain logic files, files with heavy if-else

สำหรับแต่ละ function/method **สำคัญ** (top 3-5 ตัวเท่านั้น ไม่ต้องครบทุก function) ให้ออก:

1. **Pseudocode** — อธิบายตรรกะเป็นภาษาไทย
2. **Decision Table** — เงื่อนไขทุกกิ่ง (given/when/then)
3. **Edge Cases** — null checks, exception handling ที่ซ่อนอยู่
4. **Technical Debt** — Workaround หรือ code ที่ไม่ควรลอกไป v11

---

## 🗄️ Layer 4: Data & Integration
Read: model files, schema files, **migration files** (`migrations/`, `*.sql`, schema versioning), DB query files, Redis/external API calls

- Data Model มีความสัมพันธ์กับ collection/table อื่นอย่างไร? (วาด ER สั้นๆ ถ้าช่วยให้เข้าใจ)
- **Migrations = "ชั้นดินประวัติศาสตร์":** ไล่ดู migration ตามลำดับเวลา — schema วิวัฒนาการมายังไง? field/table ไหนเพิ่มทีหลัง, ไหนถูก deprecate, ไหนเป็นซากที่ไม่ใช้แล้ว? (ถ้าไม่มี migration system ให้ระบุว่า "ไม่พบ — schema อยู่ใน code/implicit")
- มี **validation rules** อะไรบังคับที่ชั้น data (required, unique, enum, regex)?
- ใช้ Cache (Redis) ในจังหวะไหน? นโยบาย TTL/invalidate เป็นอย่างไร?
- เชื่อมต่อ Third-party หรือ service ภายนอกอย่างไร? (Auth แบบไหน, ส่งข้อมูลอะไร)

---

## 🧹 Layer 5: Keep / Drop Summary

สรุปรายงานสั้นๆ:

**🟢 รู้แน่ vs 🔴 ปริศนา (Known vs Mystery Map):**
ตารางบอกชัดว่าหลังวิเคราะห์แล้ว อะไร verified จากโค้ดจริง อะไรยังเดา/ไม่รู้ — เพื่อให้ทีมรู้ว่าต้องไปขุดต่อตรงไหน
| หัวข้อ | สถานะ | หลักฐาน / สิ่งที่ยังไม่รู้ |
|--------|-------|---------------------------|
| (เช่น) flow ฝากเงิน | 🟢 รู้แน่ | trace ครบจาก controller → queue → consumer |
| (เช่น) ใครเรียก endpoint X | 🔴 ปริศนา | ไม่พบ caller ใน repo นี้ — อาจอยู่ repo อื่น/external |

**Health Score: X/10**
(หักคะแนนจาก: test coverage, complexity, dependency เก่า, security gaps)

**Route Reusability Summary:**
| Path | พร้อม migrate? | เหตุผล / สิ่งที่ต้องปรับ |
|------|----------------|--------------------------|

**Technical Debt Inventory:**
| รายการ | ความเสี่ยง (สูง/กลาง/ต่ำ) | แก้ก่อนหรือหลัง migrate |
|--------|--------------------------|------------------------|

**ต้องลอก (Core Business Value):**
- ...

**ควรโละทิ้ง / เขียนใหม่:**
- ...

**ส่งต่อ sub-agent (ถ้าเจอ):**
| ประเด็น | ส่งต่อให้ |
|---------|----------|
| security issue | `/security-review` |
| undocumented API | prompt สร้าง OpenAPI spec |
| performance bottleneck | prompt วิเคราะห์ query/cache |

---

## 🔍 Layer 6: Feature Deep-Dive

> Layer นี้ต่อจาก Layer 2–4 — เลือก **3–5 ฟีเจอร์หลัก** ที่ผู้ใช้ทำบ่อยที่สุด (เช่น ฝากเงิน, ถอนเงิน, login, เล่นเกม, เรียกดูประวัติ) แล้วขุดให้ลึกแต่ละฟีเจอร์

สำหรับแต่ละฟีเจอร์ ให้เขียนใน format นี้:

---

### 🔹 Feature: [ชื่อฟีเจอร์ ภาษาไทย]

#### 1. ทำอะไรได้บ้าง (What it does)
อธิบายสั้นๆ ว่าฟีเจอร์นี้ทำหน้าที่อะไรในระบบจากมุมของ user และของระบบ

#### 2. จากไหนไปไหน (Full Flow)
ระบุ **ทุก hop** ตั้งแต่ user กด button ไปจนถึง response กลับมา:

```
[ขั้น 1] User/Frontend
  ↓ HTTP POST /path
[ขั้น 2] Service A (repo: xxx)
  - validate input
  - query MongoDB: collection_name
  ↓ HTTP call / RabbitMQ / Redis
[ขั้น 3] Service B (repo: yyy)
  - do something
  ↓ callback
[ขั้น 4] Service A อีกที
  - update record
  - respond
[ขั้น 5] User ได้รับ response
```

ใช้ notation:
- `↓ HTTP POST /path` = sync HTTP call
- `⟿ queue:NAME` = async publish
- `↩ callback POST /path` = webhook/callback กลับมา
- `🔒 Redis lock` = distributed lock
- `💾 MongoDB: collection` = DB operation
- `📦 Redis cache` = cache read/write

#### 3. Repos ที่เกี่ยวข้อง (Cross-repo map)
| ลำดับ | Repo | บทบาทใน flow นี้ | Endpoint / Queue |
|------|------|-----------------|-----------------|

#### 4. Happy Path vs. Edge Cases
| Scenario | ผลลัพธ์ | handled? |
|----------|---------|----------|
| กรณีปกติ (happy path) | ... | ✅ |
| [edge case 1] | ... | ✅ / ⚠️ / ❌ |
| [edge case 2] | ... | ✅ / ⚠️ / ❌ |

#### 5. จุดเสี่ยง / Tech Debt ใน flow นี้
- ...

---

> **ตัวอย่าง flows ที่ควรวิเคราะห์ใน Maximumsoft / genesis context:**
> - `ฝากเงิน QR` — topupserie → 3rd-payment → provider → callback → office → wallet
> - `ถอนเงิน` — topupserie → office-api-v10 → que_payment → provider → callback
> - `Login + เปิดเกม` — frontend → JWT → Redis session → game proxy (kinglot-seamless / go-agent-rocketwin)
> - `บันทึก statement → report` — go_deposit/go_sms/running-bank ⟿ STATEMENT → go_report
> - `Admin อนุมัติฝาก (manual)` — office-v10x → office-api-v10 → RabbitMQ → go_deposit

---

## ข้อกำหนดสำคัญ
- **อ่านไฟล์จริงก่อนตอบทุก Layer** — ห้ามเดาจากชื่อไฟล์/convention อย่างเดียว
- แยกหลักฐานแข็ง (โค้ดจริง) กับอ่อน (string ที่ grep เจอ) — ถ้าไม่แน่ใจให้ทำเครื่องหมาย (?) ไว้
- ระวัง false positive จาก comment / test / dead code
- ถ้า service เชื่อมหลาย repo ให้ใช้ผลจาก skill `repo-mapper` (ถ้ามี `docs/repo-map.md`) ประกอบ
