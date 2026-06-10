# System Archaeology 5-Layer — 3rd-payment, GOTOPOPOFFICE, office-abatech, que_payment, telesale-service-mootui, topupserie_mootui
**Maximumsoft Co., LTD | วิเคราะห์เมื่อ: 2026-06-10 (Session ที่ 2)**

---

## 📊 Health Score Overview

| โปรเจค | Stack | Health | ปัญหาหลัก |
|--------|-------|--------|-----------|
| `3rd-payment` | Go, Gin, MongoDB | **4.5/10** 🔴 | `callback.go` 16K บรรทัด, 69 payment providers ในไฟล์เดียว, 0 test |
| `GOTOPOPOFFICE` | Go, Gin, MongoDB, Redis | **5/10** 🟠 | JWT lib deprecated, ไม่มี DB transaction, race condition บน credit |
| `office-abatech` | Nuxt 2, Vue 2, Element UI | **5.5/10** 🟠 | CLAUDE_API_KEY ใน .env, Vue 2/Node 16/Firebase 8 ล้าสมัย |
| `que_payment` | Go, RabbitMQ, MongoDB | **5.5/10** 🟠 | Switch 50+ cases, application-level locking, ไม่มี circuit breaker |
| `telesale-service-mootui` | Go, Gin, MongoDB, Redis | **6.5/10** 🟢 | Clean hexagonal arch แต่ไม่มี auth middleware, webhook ไม่มี signature |
| `topupserie_mootui` | Go, Gin, MongoDB, Redis, RabbitMQ | **6.5/10** 🟢 | 40+ endpoint ซ้ำซ้อน, hardcoded secrets, JWT deprecated |

---

## 📂 Layer 0 — Runtime / Ops

### 3rd-payment
- **Port:** 8081 | **Build:** Alpine multi-stage, CGO_ENABLED=0
- **Registry:** thezeustech.jfrog.io
- **Key Env:** `DB_ENDPOINT` (MongoDB Atlas), `SYSTEM_CODE`, `MODE`, `REDIS_URL`, `RABBIT_MQ_*`, `OTEL_CONNECT`, `PAYMENT_DETAIL_PRIVATE_KEY` (RSA key ใน plaintext)
- **Upstream:** office system, web frontend, mobile apps
- **Downstream:** MongoDB Atlas, Redis (ThorLock), RabbitMQ, 40+ external payment APIs, Telegram, OpenTelemetry

### GOTOPOPOFFICE
- **Port:** 7777 | **Docker port mapping:** 4440:7777
- **Key Env:** `DATABASE` (MongoDB+Atlas+DocumentDB), `ACCESS_SECRET=@ABAOFFICER`, `GROUP`, `SERVICE`, `DOCKER` (proxy), `WHEELAPI`, `PAYMENT_API`, `HOTLINE_API_URL`, `FIREBASE`
- **Upstream:** web dashboard, LINE chatbot, mobile
- **Downstream:** MongoDB Atlas, Redis, RabbitMQ (STATEMENT_[service]), AWS S3, QR Payment, Telesale API, SMS API, Jaeger

### office-abatech
- **Port:** 8080 | **Mode:** SPA (SSR disabled)
- **Key Env:** `CLAUDE_API_KEY` ⚠️, `AUTHLOGIN=LINE`, `SERVICE`, `WEBNAME`, `LOGO`
- **API Base:** `/api` (reverse proxy → GOTOPOPOFFICE :7777)
- **Firebase:** `withdraw-monitor-new.firebaseio.com` + `g2e-billpayment.asia-southeast1.firebasedatabase.app`

### que_payment
- **Port:** ไม่มี HTTP server | **Mode:** Worker service (TYPE env)
- **Key Env:** `TYPE=QUE_RABBIT|QUE_CRONJOB`, `PAYMENT_NAME`, `DB_ENDPOINT`, `REDIS_URL`, `RABBIT_MQ_DEVELOPMENT`, `OTEL_CONNECT`, `SENTRY_DSN`
- **Upstream:** RabbitMQ (queue: `QUE_PAYMENT_<NAME>`) หรือ MongoDB polling
- **Downstream:** MongoDB, Redis, 50+ payment provider APIs, GOTOPOPOFFICE callback

### telesale-service-mootui
- **Port:** 5053 | **Build:** Alpine multi-stage, Go 1.24.5
- **Key Env:** `APP_SERVICE`, `HTTP_PORT`, `DATABASE_URI` (MongoDB Atlas), `REDIS_ADDRESS`
- **Upstream:** web/mobile frontend
- **Downstream:** MongoDB, Redis, YaleCom VoIP API (`apiDomain/robocall/send`)

### topupserie_mootui
- **Port:** 5052/5053 | **Build:** Alpine, Go 1.18
- **Key Env:** `SERVICE`, `DATABASE_URI`, `RABBITMQURL`, `REDIS_URL`, `SERVICE_API`, `API_QR`, `API_QR_ALL`, `APIWHEEL`, `ACCESS_SECRET=demo_service`, `JAEGER_AGENT_HOST`
- **Upstream:** browser/app clients
- **Downstream:** MongoDB, Redis, RabbitMQ, SERVICE_API (office), API_QR_ALL (payment), APIWHEEL, LINE OAuth, Jaeger

---

## 📂 Layer 1 — Tech Stack & Entry Points

| โปรเจค | Go/Node | Framework | DB | Cache | Queue | Notable Dep |
|--------|---------|-----------|-----|-------|-------|-------------|
| `3rd-payment` | Go 1.25.5 | Gin 1.11.0 | MongoDB 1.17.4 | Redis v8 | RabbitMQ | otelgo (custom), shopspring/decimal |
| `GOTOPOPOFFICE` | Go 1.18 | Gin 1.6.3 | MongoDB 1.10.2 | Redis v8 | RabbitMQ | ⚠️ `dgrijalva/jwt-go` deprecated |
| `office-abatech` | Node 16 / Nuxt 2 | Vue 2 | — | — | — | ⚠️ firebase ^8.2.7, node-sass, datamaps |
| `que_payment` | Go 1.25.5 | ไม่มี HTTP | MongoDB 1.17.4 | Redis v9 | RabbitMQ | otelgo, shopspring/decimal |
| `telesale-service-mootui` | Go 1.24.5 | Gin 1.10.1 | MongoDB 1.17.4 | Redis v9 | ไม่มี | Hexagonal arch |
| `topupserie_mootui` | Go 1.18 | Gin 1.7.4 | MongoDB 1.10.2 | Redis v8 | RabbitMQ | ⚠️ `dgrijalva/jwt-go`, Jaeger |

---

## 📂 Layer 2 — API Contracts & Async Paths

### 3rd-payment — Endpoints สำคัญ

| Method | Path | Purpose | Auth | Migrate? |
|--------|------|---------|------|----------|
| POST | `/api/v2/payin` | สร้าง QR/bank deposit | No | ⚠️ ต้องเพิ่ม auth |
| POST | `/api/v2/callback-payin/:payment_code` | รับ callback จาก provider | IP Whitelist | ⚠️ ต้องแยก logic |
| POST | `/api/v2/create-withdraw` | สร้างรายการถอน | No | ⚠️ ต้องเพิ่ม auth |
| POST | `/api/v2/callback-payout/:payment_code` | รับ callback ถอน | IP Whitelist | ⚠️ ต้องแยก logic |
| POST | `/api/v2/confirm-withdraw` | ยืนยันการถอน | No | ✅ |
| GET | `/api/v2/get-order/:order_no` | ดูสถานะ order | No | ✅ |
| POST | `/api/v2/virify/slip` | ตรวจสอบสลิป | No | ⚠️ ต้องเพิ่ม auth |
| POST | `/api/xpay/private/callback-payin/:payment_code` | Xpay callback | IP Whitelist | ✅ |

**Async:** RabbitMQ `QUE_PAYMENT_*` (producer), Telegram bot webhook

### GOTOPOPOFFICE — 172 files, 31+ domain packages

**Key domains:** member (36 ep), deposit (28 ep), withdraw (30 ep), statement, bonus, credit, telesale, ranking, report, SMS, bank

**Async:** RabbitMQ `STATEMENT_[service]` (fanout exchange) สำหรับ deposit/withdraw events

### office-abatech — Frontend API calls

| หน้า | Endpoint ที่เรียก | Purpose |
|-----|----------------|---------|
| login.vue | `/login`, `/register`, `/LoginUserPass` | Auth |
| MemberList.vue | `/GetMemberAll`, `/GetMemberSearch`, `/RegisterMember`, `/DeleteMemberByID` | CRUD สมาชิก |
| DashboardAll.vue | `/GetDashboardSummaryALL` | Analytics 14 วัน |
| BonusList.vue | `/GetBonusAll`, `/UpdateBonus` | Bonus config |
| Payment/AccountPayment.vue | `/GetBankCodeList` | ข้อมูลธนาคาร |

### que_payment — Queue Worker

| Type | Queue/Trigger | Payload | Action |
|------|--------------|---------|--------|
| RabbitMQ Consumer | `QUE_PAYMENT_<NAME>` | `ReqQue{id,service,type,payment_code}` | ส่งต่อ deposit/withdraw ไป provider |
| CronJob 20s | MongoDB `que_payment` status=0 | WithdrawStatement | ประมวลผลถอนเงิน (max 10 concurrent) |
| CronJob 5m | WithdrawStatement status=16 | RetryOrderCallbackAgain | retry callback office |
| CronJob 20s | service_payment | UpdateRedis | sync Redis cache |

### telesale-service-mootui — 31 Endpoints (Clean CRUD)

| Group | Endpoints | Pattern |
|-------|-----------|---------|
| provider | 7 ep | CRUD + callback webhook |
| provider-extension | 6 ep | CRUD |
| extension-operator | 6 ep | CRUD |
| group | 3 ep | Campaign groups |
| reserve | 7 ep | Member booking |
| call | 6 ep | VoIP call management |
| report | 3 ep | Analytics |

**Async:** ไม่มี — synchronous ทั้งหมด, MongoDB transactions

### topupserie_mootui — 80+ Endpoints (3 platforms)

**Multi-platform routing:** `route.go` (Main) + `route_apex_pro.go` (Apex Pro) + `route_allinone_transfer.go`

| Domain | Endpoints | สถานะ |
|--------|-----------|-------|
| Auth/Login | 8 ep | Active |
| Deposit | 12 ep (QR, auto, manual, slip) | ⚠️ 5 variants ซ้ำซ้อน |
| Withdrawal | 3 ep | Active |
| Bonus/Cashback/Commission | 12 ep | ⚠️ naming ซ้ำ (get/ prefix) |
| Bank management | 6 ep | Active |
| Promotions | 10+ ep | Active |
| Game/Wheel/Staking | 8 ep | Active |
| Admin (`/office/*`) | 4 ep | ⚠️ ไม่มี auth |

---

## 📂 Layer 3 — Business Logic & Edge Cases

### 3rd-payment — Top Functions

**`CallbackDeposit()` (callback.go ~16K บรรทัด)** ⚠️ CRITICAL
- Switch/case 69 payment providers → each has custom signature verification (MD5/SHA256/RSA)
- Redis ThorLock (180s) ป้องกัน duplicate callback
- Edge: Peer2Pay ใช้ suffix "-PEER2PAY" ใน payment_code, OneLink ดู body field ตัดสิน type

**`Deposit()`** — สร้าง QR image ด้วย `fogleman/gg` + Thai font `non_sans.ttf`, cache order ใน Redis (TTL = countdown_time)

**Status codes:** 0=pending, 1=success, 2=submitted, 3=failed, 99=manual review

### GOTOPOPOFFICE — Top Functions

**`DepositAddCredit()`** — ⚠️ ไม่มี MongoDB transaction → race condition ถ้า concurrent requests
- Flow: find member → validate amount → create statement → update credit → Redis invalidate → publish RabbitMQ
- **เสี่ยง:** credit update ไม่ atomic

**`SlipverifyDepositStatement()`** — เรียก external OCR service (hash-central) timeout 60s

### telesale-service-mootui — Top Functions

**`CallService.MakeCall()`** — ✅ ใช้ MongoDB transaction ครบ
- Validate reserve owner → check extension/provider active → transaction: increment counters + create call + call YaleCom API

**`ReserveService.Reserve()`** — ✅ transaction: check dup → increment group counters → create reserve → Redis invalidate

### topupserie_mootui — Top Functions

**Bank Condition Filtering** — Logic สำคัญสำหรับ routing payment methods:
- ConditionType 0 = ทุกคน, 1 = rank-based (StartRanking≤rank≤EndRanking), 2 = group-based

**Rate Limiting** — Redis-based, key: `start/count/ban-{path}-{ip}`, fail open เมื่อ Redis down

**Bonus Distribution** — credit_free (ถอนไม่ได้ตรงๆ), ต้องทำ turnover ตาม min_turnover ก่อน

---

## 📂 Layer 4 — Data & Integration

### Collections สำคัญ

| Collection | โปรเจค | Purpose |
|-----------|--------|---------|
| `service_payment` | 3rd-payment | Config ทุก payment provider (59 fields!) |
| `qr_payment` | 3rd-payment | Deposit orders (QR/bank) |
| `withdraw_statement` | 3rd-payment, que_payment | Withdrawal records |
| `bank_summary` | 3rd-payment, que_payment | Balance tracking per provider |
| `que_payment` | que_payment | Queue task items (status 0=pending, 2=processing, 1=done) |
| `member_account` | GOTOPOPOFFICE, topupserie | User profiles + bank list |
| `config_system` | topupserie | Master config (100+ fields) |
| `banklist` | topupserie | Payment methods + condition routing |
| `provider`, `reserve`, `call` | telesale | VoIP campaign data |

### Redis TTL Policy

| โปรเจค | Key Pattern | TTL | Purpose |
|--------|------------|-----|---------|
| 3rd-payment | `lock:{orderNo}` | 180s | Idempotency lock |
| 3rd-payment | `order_{orderNo}` | countdown_time | QR order cache |
| 3rd-payment | `balance_{payment_code}` | 300s | Balance cache |
| GOTOPOPOFFICE | `member:{username}` | 24h | Member profile |
| GOTOPOPOFFICE | `session:{token}` | 1h | Auth session |
| topupserie | `user:{username}:token` | 1h | JWT token |
| topupserie | `bank:list:{username}` | 5m | Filtered banks |
| topupserie | `ban-{ip}` | 30m | Rate limit ban |

### Inter-Service Call Chain

```
ฝากเงิน (QR):
User → topupserie [POST /qrpay]
     → 3rd-payment [POST /api/v2/payin] → สร้าง QR image
     → External provider (Hengpay/Peer2Pay/etc)
     → Callback → 3rd-payment [POST /callback-payin/:code] + ThorLock
     → GOTOPOPOFFICE webhook (UriForwardPayin) → อัปเดต wallet
     → RabbitMQ STATEMENT_[service] → consumer updates

ถอนเงิน:
GOTOPOPOFFICE [POST /withdraw] → create withdraw_statement
     → publish RabbitMQ QUE_PAYMENT_* → que_payment consumer
     → que_payment: get bank_summary + is_active lock
     → call provider payout API (e.g. SmartPay /api/defray/V2)
     → callback → GOTOPOPOFFICE อัปเดต status

Telesale Call:
telesale [POST /api/v2/call] + MongoDB transaction
     → YaleCom [GET /robocall/send?Ref1=callID]
     → webhook [POST /api/v1/provider/callback/yalecom]
     → update Call record + recording URL
```

---

## 📂 Layer 5 — Keep / Drop Summary

### สิ่งที่ควร KEEP ✅

| Component | โปรเจค | เหตุผล |
|-----------|--------|-------|
| OTel + Telegram + ClickHouse observability | 3rd-payment, que_payment | Production-grade monitoring |
| ThorLock (Redis distributed lock) | 3rd-payment | ป้องกัน duplicate callback ได้ดี |
| Hexagonal architecture | telesale | Clean, testable, extensible |
| MongoDB transaction pattern | telesale | Atomic counter updates |
| Multi-tenancy via ResourceService map | GOTOPOPOFFICE | Core pattern ของทั้ง platform |
| Bank condition filtering logic | topupserie | Core business rule |
| Graceful shutdown + ShutdownManager | que_payment | Clean goroutine management |

### สิ่งที่ควร REWRITE 🔴

| Component | โปรเจค | เหตุผล |
|-----------|--------|-------|
| `callback.go` 16K monolith | 3rd-payment | ไม่ maintainable |
| `SwitchGetBalancePayment` 400+ switch | que_payment | ต้องเป็น interface |
| Credit update logic | GOTOPOPOFFICE | ต้องใช้ MongoDB transaction |
| `/office/*` admin routes | topupserie | ต้องเพิ่ม auth |
| Vue 2 → Vue 3 migration | office-abatech | Framework EOL |
| Response format branching | topupserie | PROJECT env var routing = unmaintainable |

### Technical Debt Priority Matrix

| Priority | Issue | โปรเจค | Effort | Risk |
|----------|-------|--------|--------|------|
| 🔴 P0 | Secrets ใน .env/git | ทุกโปรเจค | 1-2 days | Data breach |
| 🔴 P0 | `CLAUDE_API_KEY` ใน env block | office-abatech | 1 hr | Key leak ทุก user |
| 🔴 P0 | ไม่มี auth บน `/office/flushall-redis` | topupserie | 2 hrs | Redis flush ไม่ต้อง login |
| 🔴 P0 | JWT lib deprecated CVE | GOTOPOP, topup | 4 hrs | Security patches |
| 🔴 P0 | ไม่มี DB transaction บน credit/deposit | GOTOPOPOFFICE | 1 wk | Race condition ยอดเงิน |
| 🟠 P1 | Webhook ไม่มี signature verify | telesale | 2 hrs | Spoofed callbacks |
| 🟠 P1 | ไม่มี auth middleware | telesale | 2 days | Unauthorized access |
| 🟠 P1 | RabbitMQ Nack(requeue=false) | que_payment | 4 hrs | Silent message loss |
| 🟠 P1 | แยก callback.go → 69 packages | 3rd-payment | 3-4 wks | Maintenance |
| 🟡 P2 | สร้าง PaymentGateway interface | que_payment | 1-2 wks | Extensibility |
| 🟡 P2 | Vue 2 → Nuxt 3 migration | office-abatech | 6-8 wks | Framework EOL |
| 🟡 P2 | Endpoint consolidation | topupserie | 1 wk | DX improvement |
| 🟡 P2 | เพิ่ม test coverage (0% ทุกโปรเจค) | ทุกโปรเจค | ongoing | Regression risk |

---

## 🚀 Recommended Phases

**Phase 1 — Security (สัปดาห์ 1-2)**
- ย้าย secrets → AWS Secrets Manager / K8s secrets
- Fix `CLAUDE_API_KEY` exposure ใน office-abatech
- เพิ่ม auth บน `/office/*` routes
- Migrate JWT lib

**Phase 2 — Stability (สัปดาห์ 3-6)**
- MongoDB transactions บน deposit/withdraw (GOTOPOPOFFICE)
- RabbitMQ requeue logic (que_payment)
- Webhook signature verification (telesale)
- Auth middleware (telesale)

**Phase 3 — Maintainability (เดือน 2-3)**
- แยก `callback.go` → provider packages + interface
- `PaymentGateway` interface สำหรับ que_payment
- Consolidate duplicate endpoints (topupserie)

**Phase 4 — Modernization (เดือน 4-6)**
- Migrate office-abatech → Nuxt 3 / Vue 3 / TypeScript
- เพิ่ม test coverage ≥60% บน critical paths
- API versioning scheme

---

*Generated by `/archaeology-5layer` | 2026-06-10*
