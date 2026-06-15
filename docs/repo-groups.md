# Repo Grouping (team-defined / canonical)

> Grouping ที่ทีมกำหนดเอง ใช้เป็น **ตั้งต้น (canonical)** ของการวิเคราะห์ — skill `architecture-advisor` / `repo-atlas` ควรใช้ grouping นี้เป็นจุดเริ่ม ไม่ derive กลุ่มเองจากศูนย์ แต่ยังต้องตรวจว่า code evidence ขัดกับ grouping นี้ตรงไหน (boundary repo, shared-data ข้ามกลุ่ม)
>
> วิเคราะห์ ณ commit: `40367af` · อัปเดต: 2026-06-15 · รวม 25 repo (24 อยู่ใน 11 กลุ่ม + 1 ยังไม่จัดกลุ่ม)
> ชื่อในตารางคือ **ชื่อโฟลเดอร์จริง** (ต่างจากที่พิมพ์ครั้งแรกเล็กน้อย เช่น `affiliate_standalone`, `hydra-affilaite`)

| # | Group | Repos | จำนวน |
|---|-------|-------|-------|
| 1 | **payment** | `3rd-payment`, `que_payment`, `3rd-gateway` | 3 |
| 2 | **deposit** | `go_deposit`, `running-bank`, `hash-central`, `go_sms` | 4 |
| 3 | **withdraw** | `withdraw-auto`, `queue-withdraw-go` | 2 |
| 4 | **affiliate** | `affiliate_standalone`, `hydra-affilaite`, `queue-dynamic-go`, `hydra-hyperx-web` | 4 |
| 5 | **report** | `go_report`, `sub_statement` | 2 |
| 6 | **sms** | `cronjob-smsauto` | 1 |
| 7 | **agent** | `go-agent-rocketwin`, `kinglot-seamless` | 2 |
| 8 | **office** | `office-api-v10`, `office-v10x` | 2 |
| 9 | **customer** | `topupserie`, `theme-violet-game` | 2 |
| 10 | **line** | `go-linebot-system` | 1 |
| 11 | **coupon** | `coupon-service` | 1 |
| — | **(ยังไม่จัดกลุ่ม)** | `SCHEDULE_SERVICE_BANK` | 1 |

**รวม: 25 repo** (เลขตรวจสอบ — ทุก Phase ที่อ้าง repo ต้อง reconcile กับเลขนี้)

## หมายเหตุที่ต้องยืนยันตอนวิเคราะห์

- **`SCHEDULE_SERVICE_BANK` ยังไม่มีกลุ่ม** — ชื่อสื่อถึง bank schedule (น่าจะ deposit หรือ report) แต่ยังไม่ยืนยันจากโค้ด → Phase 2 ต้องจัดกลุ่มจาก edge จริง แล้วเสนอกลุ่มให้ทีมยืนยัน
- **`go_sms` (กลุ่ม deposit) vs `cronjob-smsauto` (กลุ่ม sms)** — งาน SMS แยกอยู่สองกลุ่มโดยตั้งใจ ตรวจว่าทั้งสองตัวคุยกัน/แชร์ data กันไหม (อาจเป็น boundary)
- **`theme-violet-game`** — เดิมระบุว่า "theme ต่างๆ" แต่ในโฟลเดอร์มีธีมเดียว ถ้ามีธีมเพิ่มภายหลังให้เพิ่มในกลุ่ม customer
