# Architecture Advisor — Executive Summary

> วิเคราะห์ ณ commit: `40367af` · วันที่: 2026-06-15
> Monorepo genesis-all-in-one: **25 repos** · 5 fullstack devs · transformative redesign

---

## ระบบมีอะไรบ้าง

25 repos แบ่งเป็น 11 กลุ่ม + 1 ungrouped:

| กลุ่ม | Repos | ประเภท |
|-------|-------|--------|
| payment | 3rd-payment, que_payment, 3rd-gateway | gateway + worker |
| deposit | running-bank, go_deposit, hash-central, go_sms | poller + worker + dedup |
| withdraw | withdraw-auto, queue-withdraw-go | worker |
| affiliate | affiliate_standalone, hydra-affilaite, queue-dynamic-go, hydra-hyperx-web | API + worker + SPA |
| report | go_report, sub_statement | worker |
| sms | cronjob-smsauto | cronjob |
| agent | go-agent-rocketwin, kinglot-seamless | game agent (dual mode) |
| office | office-api-v10, office-v10x | API + SPA |
| customer | topupserie, theme-violet-game | API + SPA |
| line | go-linebot-system | LINE bot |
| coupon | coupon-service | API |
| *ungrouped* | SCHEDULE_SERVICE_BANK | cronjob → **เสนอเข้า deposit** |

---

## ความเสี่ยง 🔴 (เงินหาย / exploitable) — 28 ข้อ

สามข้อที่ต้องรู้ก่อนอื่น:

1. **`queue-withdraw-go`** — context timeout ไม่ pass เข้า Withdraw → nack-drop ขณะ debit ยังทำงาน → **เงินหักแต่ไม่โอน** (`pub/amqp.go:101,123,131`)
2. **`kinglot-seamless`** — ถ้า `AUTHEN` env ว่าง → `/api/v1/placebet` เปิด unauthenticated → **ใครก็ debit wallet ได้** (`method/middleware.go:12-13`)
3. **`hydra-affilaite + office-api-v10`** — AWS IAM key `AKIA5MF2WON2MLFDZG2I` hardcoded ใน 2 repos → **ต้อง rotate ทันที** (`helper/helper.go:666`)

Risk register ทั้งหมด → [`02-risks.md`](02-risks.md) (73 ข้อ: 28 🔴 + 24 🟠 + 21 🟡)

---

## ปัญหาเชิงสถาปัตยกรรมหลัก — 11 ข้อ

| # | ปัญหา | ระดับ |
|---|-------|-------|
| P-01 | office-api-v10 เป็น monolith hub รับ callback ทุกสาย | 🔴 |
| P-02 | Non-durable queues + ไม่มี DLQ บนทุก financial flow | 🔴 |
| P-03 | Withdraw pipeline มีหลายจุดที่เงินหายได้ | 🔴 |
| P-04 | Bet/lotto wallet ไม่มี idempotency + advisory lock แตก | 🔴 |
| P-05 | Shared MongoDB collections ไม่มีเจ้าของ (withdraw_statement: 6 writers) | 🟠 |
| P-06 | Credentials hardcoded ทั่วทุก service | 🔴 |
| P-07 | Auth ไม่สม่ำเสมอ — 3 pattern ใน codebase เดียว | 🔴 |
| P-08 | Code/logic ซ้ำข้าม repos (40+ payment providers ซ้ำ 2 repos) | 🟡 |
| P-09 | database_config collection เป็น single point of config failure | 🟠 |
| P-10 | Frontend config hardcoded ใน source ทุก repo | 🟡 |
| P-11 | Observability ไม่สม่ำเสมอ — เงิน fail ไม่รู้จนลูกค้าบอก | 🟠 |

รายละเอียด → [`03-diagnosis.md`](03-diagnosis.md)

---

## ทางเลือกที่แนะนำ: Option 1 — Domain Service Consolidation

**ลด 25 repos → 10 deployables** ตาม bounded context + `platform-lib` Go module รวม auth/money/queue/OTel

| เก่า | ใหม่ |
|------|------|
| 25 repos | 10 Go services + 3 SPAs + 1 shared lib |
| ไม่มี DLQ | durable queues + DLQ ทุก financial flow |
| credentials ใน source | Secrets Manager |
| auth 3 แบบ | platform-lib/auth middleware เดียว |
| withdraw_statement 6 writers | withdraw-svc เจ้าของเดียว |
| go-agent-rocketwin ≈ kinglot-seamless | agent-svc merged (1 binary) |

ทางเลือกทั้งหมด + trade-off → [`04-target-options.md`](04-target-options.md)

---

## เริ่มจากอะไร — Quick Wins (ทำได้พรุ่งนี้)

| # | งาน | แก้ |
|---|-----|-----|
| QW-1 | Rotate AWS IAM key `AKIA5MF2WON2MLFDZG2I` | R-018 🔴 |
| QW-2 | `durable=false` → `true` บน QUE_PAYMENT, WALLETWITHDRAW, DYNAMICQUEUE, CALLBACK_LOTTO | R-022–024 🔴 |
| QW-3 | เพิ่ม DLQ บนทุก financial queue | R-022–024 🔴 |
| QW-4 | บังคับ `AUTHEN` env ใน kinglot-seamless | R-004 🔴 |
| QW-5 | ลบ `CHECK_IPWHITELIST` bypass ใน 3rd-payment | R-007 🔴 |
| QW-6 | uncomment unique index บน `transaction_statement` (hash-central) | R-021 🔴 |
| QW-7 | Timeout 30s บน http.Client ทุกตัว | R-050 🟠 |
| QW-8 | ย้าย + rotate RabbitMQ/Telegram credentials ออก source | R-025/026 🔴 |

---

## Migration Timeline (18 สัปดาห์)

```
QW  สัปดาห์ 0    Rotate secrets + durable queues + auth fixes
P0  สัปดาห์ 1–2  Platform foundation (RabbitMQ cluster, Redis Sentinel, Secrets Manager, platform-lib)
P1  สัปดาห์ 3–5  Fix money flows (withdraw-go txn, kinglot idempotency, hash SHA-256)
P2  สัปดาห์ 5–7  Security baseline (API gateway, auth middleware, close unauthenticated endpoints)
P3  สัปดาห์ 8–14 Service consolidation 25→10 (strangler-fig per group)
P4  สัปดาห์ 15–18 Frontend env separation
P5  Continuous    Observability (ควบคู่ Phase 3)
```

Roadmap ละเอียด → [`05-roadmap.md`](05-roadmap.md)

---

## ไฟล์ทั้งหมด

| ไฟล์ | เนื้อหา |
|------|---------|
| [README.md](README.md) | Executive summary (ไฟล์นี้) |
| [01-current-state.md](01-current-state.md) | Inventory 25 repos, edge list, Mermaid diagram, SPOF |
| [02-risks.md](02-risks.md) | Risk register 73 ข้อ + ✅ ผ่าน + ⚪ ข้อสงสัย |
| [03-diagnosis.md](03-diagnosis.md) | 11 ปัญหาโครงสร้าง + Architecture Drivers |
| [04-target-options.md](04-target-options.md) | Option 1 + Option 2 + agent sub-options + scoring |
| [05-roadmap.md](05-roadmap.md) | Migration roadmap 18 สัปดาห์ + quick wins + open questions |
| [cards/](cards/) | 25 Interface Cards (หลักฐานดิบทุก repo) |
