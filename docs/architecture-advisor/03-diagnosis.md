# Structural Diagnosis

> วิเคราะห์ ณ commit: `40367af` · วันที่: 2026-06-15
> Phase 4: รวม risk จาก Phase 3 → ปัญหาเชิงโครงสร้าง → Architecture Drivers

---

## 1. ปัญหาเชิงโครงสร้าง

| # | ปัญหา | อาการ / risk ที่โยง | ผลกระทบต่อการพัฒนา/ดำเนินงาน |
|---|-------|-------------------|-------------------------------|
| **P-01** | **office-api-v10 เป็น Hub เดียวที่รับ callback ทุกสาย** | R-015: 30+ unauthenticated routes; R-045: AMQP publish swallow error; R-046: double-credit; H-02, H-03, H-04, H-05, H-06, H-07 | ถ้า office-api-v10 ล่ม: deposit + withdrawal callbacks ทั้งหมดหยุด ไม่มี queue buffer. feature ใหม่ทุกอย่างต้องแก้ที่นี่ → bottleneck development |
| **P-02** | **Non-durable queues + ไม่มี DLQ บนทุก financial flow** | R-022, R-023, R-024, Q-01 ถึง Q-12 | RabbitMQ restart = in-flight payment/deposit/withdraw/bet-settle events สูญถาวร ไม่มีทางกู้คืน. ไม่มี dead-letter → ไม่รู้ว่า message fail |
| **P-03** | **Withdraw pipeline มีหลายจุดที่เงินหายได้** | R-001 (ctx not passed), R-002 (5 writes no txn), R-003 (auto-ack), R-027 (in-memory lock) | withdraw ล้มเหลวแบบ partial: เงินหักแต่ไม่โอน, โอนแต่ไม่บันทึก — ตรวจหลังข้อเท็จจริงยาก เพราะ ไม่มี DLQ, ไม่มี idempotency |
| **P-04** | **Bet/lotto wallet ไม่มี idempotency enforcement + advisory lock แตก** | R-004, R-005, R-006, R-037, R-038 | duplicate callback = double debit/credit wallet; process crash = lock ค้าง = user ทำ bet ไม่ได้; ยังมี 2 versions (go-agent-rocketwin + kinglot-seamless) ที่ consume queue เดิมกัน |
| **P-05** | **Shared MongoDB collections ไม่มีเจ้าของชัดเจน** | withdraw_statement (6 writers), bank_statement (4 writers), wallet_statement (3 writers), shared in data table | schema drift — service A เพิ่ม field ที่ service B ไม่รู้ → silent read failure. ไม่มีใครรับผิดชอบ migration; go_report เขียน back-office collections (R-036) |
| **P-06** | **Credential/secret hardcode ทั่วทุก service** | R-018 (AWS IAM x2 repos), R-025 (RabbitMQ x3), R-026 (Telegram), R-065 (Redis), R-066 (MongoDB), R-067 (cron secret) + อีกหลายจุด | ทุก commit/push = credential leak ใน git history. rotation ต้อง code change + redeploy ทุกครั้ง. ไม่มีทางบังคับ rotation policy |
| **P-07** | **Auth ไม่สม่ำเสมอ — pattern 3 อย่างใน codebase เดียว** | R-004 (optional env), R-014 (ไม่มีเลย), R-015 (routes ปนกัน), R-020 (fallback "secret") | เพิ่ม endpoint ใหม่ → ไม่รู้ว่า pattern ไหนถูก. code review ต้องเช็คทุก route ด้วยตา. รับมาจาก 25 repos ที่ไม่มี shared auth library |
| **P-08** | **โค้ด/logic ซ้ำข้าม repos** | R-060 (go-agent-rocketwin + kinglot identical), R-061 (40+ payment providers ซ้ำ), cronjob-smsauto + go_sms ต่างก็อ่าน config_system สำหรับ SMS | แก้ bug ที่เดียว อีก 1-2 ที่ยังเหลือ. ไม่ชัดว่า deployment ไหน active → risky hotfix |
| **P-09** | **Multi-tenancy routing ผ่าน database_config collection — single point of config failure** | S-07, R-045 (AMQP publish error swallow), H-12 | database_config corrupt = office-api-v10, go-linebot-system, SCHEDULE_SERVICE_BANK สูญเสีย per-tenant connections ทั้งหมด. ไม่มี fallback |
| **P-10** | **Frontend config hardcoded ใน source — ไม่มี environment separation** | R-063 (office-v10x), theme-violet-game dev URL hardcoded, hydra-hyperx-web localhost:5053 | build สำหรับต่าง environment ต้องแก้ source; dev URL leak เข้า production build ได้; CI/CD ไม่สามารถ inject config ได้ |
| **P-11** | **Observability ไม่สม่ำเสมอ — บาง service มี OTel, ส่วนใหญ่ไม่มี** | 3rd-gateway OTel commented out; auto-ack = error สูญหาย ไม่รู้; AMQP publish error swallow; go_report Go 1.15 | ไม่รู้ว่าระบบพังที่ไหน ถ้า deposit ไม่เข้าต้อง debug blindly |

---

## 2. ตาราง Structural Problems สรุป

| # | ปัญหา | Risk ที่โยงถึง | ความเร่งด่วน |
|---|-------|---------------|-------------|
| P-01 | office-api-v10 monolith hub | R-015, R-045, R-046 | 🔴 ต้องแก้ก่อน deploy feature ใหม่ |
| P-02 | Non-durable queues / No DLQ | R-022, R-023, R-024 | 🔴 แก้ได้ทันทีโดยไม่ต้อง redesign |
| P-03 | Withdraw pipeline unsafe | R-001, R-002, R-003, R-027 | 🔴 เงินหายได้ทุกวัน |
| P-04 | Bet wallet unsafe | R-004, R-005, R-006, R-037, R-038 | 🔴 เงินหายได้ทุก bet |
| P-05 | Shared DB ไม่มีเจ้าของ | withdraw_statement 6 writers | 🟠 long-term debt; schema drift |
| P-06 | Credentials hardcoded | R-018, R-025, R-026 | 🔴 ต้อง rotate ทันที |
| P-07 | Auth inconsistency | R-004, R-014, R-015, R-020 | 🔴 ลด attack surface ก่อน |
| P-08 | Code duplication | R-060, R-061 | 🟡 maintenance risk |
| P-09 | Config single point of failure | database_config collection | 🟠 reliability risk |
| P-10 | Frontend config hardcoded | R-063, R-068 | 🟡 แก้ได้ระหว่าง sprint |
| P-11 | Observability ไม่ครบ | auto-ack, error swallow | 🟠 ไม่รู้ว่าพังจนลูกค้าบอก |

---

## 3. Architecture Drivers (เกณฑ์ตัดสินใจสำหรับ Phase 5)

สิ่งที่สกัดได้จาก code evidence — ยังต้องยืนยัน priority กับทีม:

| Driver | จาก Evidence | ความสำคัญ (เดา) |
|--------|-------------|----------------|
| **D-01: เงินต้องไม่หาย** | R-001–R-003, R-005, R-006, R-012, R-013, R-021 — หลายจุดที่ partial failure = money loss | 🔴 Critical |
| **D-02: ระบบต้องไม่ล่มทั้งหมดจาก single failure** | P-01 (office hub), P-02 (non-durable), P-09 (config SPOF) | 🔴 Critical |
| **D-03: ลด credential exposure ทันที** | R-018, R-025 — AWS IAM, RabbitMQ ใน git history | 🔴 ต้องทำก่อน Phase 5 เสร็จ |
| **D-04: Security baseline ขั้นต่ำ** | R-007–R-017 — unauthenticated financial endpoints | 🔴 Critical |
| **D-05: รองรับ multi-tenant growth** | database_config pattern, SCHEDULE_SERVICE_BANK per-tenant loops | 🟠 Important |
| **D-06: Developer velocity — feature ใหม่ใช้เวลาน้อยลง** | P-08 (code dup), P-07 (auth inconsistency), P-10 (frontend hardcoded) | 🟠 Important |
| **D-07: Observability — รู้ว่าพังที่ไหน** | P-11 | 🟠 Important |

---

## 4. คำถามที่ต้องถามก่อน Phase 5

ก่อนเสนอ target architecture ต้องรู้ข้อมูลนี้จากทีม เพราะ trade-off ต่างกันมาก:

**ทีมและ capacity:**
1. ทีม backend มีกี่คน? (ส่งผลต่อ: รวม service vs แยก service)
2. มีทีม infra/platform แยกต่างหากไหม? (ส่งผลต่อ: ใครดูแล shared infra)

**Timeline:**
3. มี deadline คร่าวๆ ไหม? เช่น "ต้องแก้ 🔴 ภายใน Q3"
4. มีช่วงที่ห้ามทำ downtime ไหม? (เช่น promotions, high-traffic periods)

**Constraints:**
5. ยอมรับ downtime สั้นๆ (1-2 นาที) สำหรับ migration ได้ไหม?
6. มี budget สำหรับ infra เพิ่มไหม? (เช่น RabbitMQ cluster, Redis sentinel)
7. มี service ไหนที่ "ห้ามแตะ" ใน 6 เดือนข้างหน้า? (เช่น กำลัง rewrite อยู่)

**Tech strategy:**
8. ต้องการ option แบบ "แก้ที่จุดอันตรายก่อนโดยไม่ redesign" ด้วยไหม (conservative) หรือ ต้องการ target ที่ clean ขึ้นพร้อม migration plan (transformative)?
9. อยากรวม go-agent-rocketwin กับ kinglot-seamless ไหม? (ตอนนี้ duplicate)
10. `3rd-gateway` (TrueMoneyWallet) เป็น service สำคัญไหม หรือ phasing out?

---

> ไฟล์ถัดไป: **04-target-options.md** (Phase 5) จะเขียนหลังได้รับ constraints จากทีม
> ไฟล์ที่มีแล้ว: `01-current-state.md`, `02-risks.md`, `03-diagnosis.md`, `cards/` (25 ไฟล์)
