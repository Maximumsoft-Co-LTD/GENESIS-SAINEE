# System Archaeology — 3rd-payment, que_payment, GOTOPOPOFFICE, office-abatech, topupserie_mootui, telesale-service-mootui
**Maximumsoft Co., LTD | วิเคราะห์เมื่อ: 2026-06-10**

---

## 📊 สรุปภาพรวม Health Score

```
telesale-service-mootui  ████████████████████ 9/10 ✅ ใช้เป็น template
3rd-payment              ██████████████       7/10 🟡 แก้ auth + secret ก่อน
que_payment              ████████████         6/10 🟡 แก้ nack logic
GOTOPOPOFFICE            ██████████           5/10 🟠 upgrade deps เร่งด่วน
topupserie_mootui        ██████████           5/10 🟠 หลาย critical issue
office-abatech           ████████             4/10 🔴 EOL stack + key leak
```

**สรุปข้อสังเกตระดับ Architecture:**
- ระบบทั้ง 6 ทำงานร่วมกันเป็น ecosystem: `topupserie` (member-facing) → `3rd-payment` (payment hub) → `que_payment` (async processor) → `GOTOPOPOFFICE` (office) ← `office-abatech` (UI)
- `telesale-service-mootui` สะอาดที่สุด — ใช้เป็น template สำหรับ service ใหม่
- **Critical path:** ถ้า `3rd-payment` ล่ม → ทั้งระบบ payment หยุด → `que_payment` queue ค้าง → ต้องมี retry + circuit breaker

---

## 📂 Layer 0: Runtime / Ops Reality

### Deploy Strategy (ทุก project)
ทุก project ใช้ **Docker multi-stage build** (Go Alpine) → push image ไป **JFrog Artifactory** (`thezeustech.jfrog.io/docker-local/`) → deploy บน **Kubernetes** ผ่าน GitOps webhook

| Project | Image | Port | Go/Node | หมายเหตุ |
|---------|-------|------|---------|---------|
| `3rd-payment` | `all-in-one-payment` | `:8081` | Go 1.26 | Dockerfile platform=**arm64** ⚠️ |
| `GOTOPOPOFFICE` | `api-topupoffice` | `:7777` | Go 1.18 | amd64 |
| `que_payment` | `queue-payment` | ไม่มี HTTP | Go 1.24 | amd64 |
| `topupserie_mootui` | `thezeusthech/topupserie` | `:5052` | Go 1.18/Dockerfile 1.19 ⚠️ | amd64 |
| `telesale-service-mootui` | — | `:5053` | Go 1.24.5 | amd64 |
| `office-abatech` | Node 16 + http-server | `:8080` | Node 16 EOL ⚠️ | SPA mode |

### Environment-Controlled Behavior
| Env Var | Project | เปลี่ยนอะไร |
|---------|---------|------------|
| `MODE` (DEV/PRODUCTION) | GOTOPOPOFFICE | MongoDB TLS |
| `TYPE` (QUE_RABBIT/QUE_CRONJOB) | que_payment | โหมดทำงานทั้งหมด |
| `TYPEAUTH` (authorization/apikey) | topupserie_mootui | auth middleware |
| `PROJECT` | topupserie_mootui, 3rd-payment | response format ตาม platform |
| `PAYMENT_NAME` | que_payment | queue name + cronjob logic |

### 🔐 Secrets ที่พบ (ควร rotate ถ้ายังใช้อยู่)
- MongoDB credentials อยู่ใน `.env` plaintext ทุก project
- `PAYMENT_DETAIL_PRIVATE_KEY` — RSA private key ใน `.env` ของ 3rd-payment ⚠️
- JWT Secret: `ACCESS_SECRET=@ABAOFFICER`, `ACCESS_SECRET=demo_service` (อ่อนแอ)
- `SECRET_KEY_2FA`, LINE Channel Secret, `HOTLINE_API_KEY`
- Redis/RabbitMQ ไม่มี password ใน dev

### Blast Radius (upstream ↔ downstream)
```
office-abatech (Frontend SPA)
    └──► GOTOPOPOFFICE :7777   (back-office API)
    └──► 3rd-payment :8081     (payment statements)

topupserie_mootui :5052 (Member-facing API)
    └──► 3rd-payment (API_QR_ALL)       เรียก payment
    └──► GOTOPOPOFFICE (SERVICE_API)    เรียก agent service
    └──► External: wheel, supercom, LINE

3rd-payment :8081
    └──► MongoDB (payment DB)
    └──► Redis (thorlock, cache)
    └──► RabbitMQ (publish event)
    └──► External: 40+ payment providers
    └──► Telegram Bot API, payment-detail-inquiry

que_payment (no HTTP)
    ◄── RabbitMQ (consume จาก 3rd-payment / topupserie)
    └──► MongoDB (write result + callback to office)

telesale-service-mootui :5053
    └──► MongoDB (telesale DB)
    └──► Redis (reserve cache)
    └──► External: VOIP providers
    ◄── GOTOPOPOFFICE (import telesale package)
```

---

## 📂 Layer 1: Tech Stack & Entry Points

| Project | Go/Node | Gin | MongoDB Driver | Dependency น่าสังเกต |
|---------|---------|-----|----------------|---------------------|
| `3rd-payment` | Go 1.26 | 1.11.0 | 1.17.4 | chromedp, ClickHouse, QUIC |
| `GOTOPOPOFFICE` | Go 1.18 ⚠️ | **1.6.3** ⚠️ | **1.10.2** ⚠️ | dgrijalva/jwt-go CVE ⚠️ |
| `que_payment` | Go 1.24 | 1.10.1 | 1.17.4 | Sentry (DSN ว่าง) |
| `topupserie_mootui` | Go 1.18 ⚠️ | 1.7.4 ⚠️ | — | dgrijalva/jwt-go ⚠️, redis v6+v8 ผสม |
| `telesale-service-mootui` | Go 1.24.5 ✅ | 1.10.1 | 1.17.4 | สะอาด ไม่มี legacy |
| `office-abatech` | Node 16 EOL ⚠️ | Nuxt 2 EOL ⚠️ | — | node-sass deprecated |

**Entry Points:**
- Go services: `_cmd/main.go` → `StartServer()` / `StartServ()`
- telesale: `cmd/http/main.go` (clean architecture)
- office-abatech: Nuxt `pages/` (file-based routing, SPA)

---

## 🔄 Layer 2: API Contracts & Data Flow

### 3rd-payment (`/api/v2/*`) — ~60+ endpoints

| กลุ่ม | Endpoints หลัก | Auth |
|-------|---------------|------|
| Payment flow | `/payin`, `/create-withdraw`, `/confirm-withdraw` | ❌ ไม่มี |
| Callbacks | `/callback-payin/:code`, `/callback-payout/:code` | IP Whitelist เท่านั้น |
| One-link callback | `/callback-one-link/:name/:code` | IP Whitelist |
| Statements | `/deposit/statement`, `/withdraw/statement`, `/topup/statement` | ❌ ไม่มี |
| Config | `/create-config-payment`, `/update-config-payment` | ❌ ไม่มี |
| Balance | `/balance`, `/check/balance` | ❌ ไม่มี |
| Dashboard | `/dashboard/balancePayment` | ❌ ไม่มี |
| Slip verify | `/virify/slip`, `/virify/slip-image` | ❌ ไม่มี |
| Peer2Pay v3 | duplicate ของ v2 | IP Whitelist บาง route |

**⚠️ ปัญหาใหญ่: เกือบทุก endpoint ไม่มี auth**

### GOTOPOPOFFICE (`/api/*`) — ~32 domain packages

| Domain | Route | Auth |
|--------|-------|------|
| member | `/member/list` | JWT AuthRequiredDB ✅ |
| deposit | `/GetDepositStatementListByFilter/:service` | mixed |
| bank | bank management | JWT |
| agent | agent management | JWT |
| SystemConfig | system config | JWT |
| ...32 domains | ... | JWT mostly |

### topupserie_mootui — Multi-platform routing

| Group | Endpoints หลัก | Auth |
|-------|---------------|------|
| `/` main | login, register, withdraw, history, deposit | AuthHeader |
| `/get/` | commission, cashback, reward | AuthHeader |
| `/set/` | event, bonus, bank | AuthHeader |
| `/office/` | systemConfig, **flushall-redis** | **❌ ไม่มี auth** ⚠️ |
| `/game/` | listgame, accessGame, lotto | mixed |
| `/covid/` | credit, transfer sub-wallet | AuthHeader |
| `/lotto/` | login, bet, history | AuthHeader |

### telesale-service-mootui (`/api/v1/*`)

| Method | Path | หน้าที่ |
|--------|------|--------|
| CRUD | `/v1/provider` | VOIP providers |
| CRUD | `/v1/provider-extension` | extensions |
| CRUD | `/v1/extension-operator` | operator mapping |
| CRUD | `/v1/group` | กลุ่ม telesale |
| CRUD | `/v1/reserve` | จอง extension |
| CRUD | `/v1/call` | call records |
| POST | `/v2/call` | MakeCallV2 |
| GET | `/v1/report` | report |
| POST | `/v1/provider/callback/:provider` | VOIP webhook |

### Async Paths

| Project | ประเภท | Queue/Key | Payload | ส่งต่อ |
|---------|-------|-----------|---------|-------|
| `que_payment` | RabbitMQ Consumer | `QUE_PAYMENT_{PAYMENT_NAME}` | `{id, service, type, payment_code}` | deposit/withdraw per provider |
| `que_payment` | Cronjob | — | — | RecoverBankSummary, RetryCallback, UpdateRedis |
| `3rd-payment` | RabbitMQ Publisher | — | — | publish → que_payment |
| `topupserie_mootui` | RabbitMQ | AMQP direct | deposit event | → que_payment |
| `GOTOPOPOFFICE` | RabbitMQ | streadway/amqp | notifications | Telegram/SMS |

---

## 🧠 Layer 3: Business Logic & Edge Cases

### 1. que_payment — `DepositAgent()`
**Pseudocode:**
```
รับ {id, service, type, paymentCode}
→ ดึง deposit statement จาก DB
→ ถ้า IsSuccess==1 → return error (idempotency check)
→ switch currency: BAHT → depositAgentForThaiBAHT()
→ route ไป provider.Deposit() ตาม paymentCode
→ อัพเดตสถานะ + callback ไป office
```
**Edge Cases:**
- Typo: "succested" แทน "succeeded" — บ่งบอก lack of review
- `d.Nack(false, false)` เมื่อ unmarshal error = **drop message ถาวร** ⚠️
- ไม่มี idempotency key ที่แข็งแกร่ง — ถ้า crash กลางทางอาจ process ซ้ำ

### 2. 3rd-payment — `CallbackDeposit()`
**Decision Table:**
| IP ใน whitelist? | Statement มีอยู่? | IsSuccess? | ผลลัพธ์ |
|-----------------|-----------------|------------|---------|
| No | — | — | 403 |
| Yes | No | — | error log |
| Yes | Yes | 1 | return OK (idempotent) ✅ |
| Yes | Yes | 0 | process + publish + callback |

**Technical Debt:** `RouteByPeer2Pay` ซ้ำ routes กับ `/api/v2` — duplicate code

### 3. topupserie_mootui — Bank filtering
**Pseudocode:**
```
ดึง banklist (status=1, bank_code IN [KBANK,SCB])
→ sort ด้วย weight (MANUAL=10, KBANK=5, อื่น=0)
→ limit 1 → เลือกบัญชีน้ำหนักสูงสุด
→ ดึง auto/truewallet แยก
→ return {auto, truewallet, bank}
```
**Edge Cases:**
- ใช้ `try.This()` (panic/recover) ซ่อน error จริงๆ ⚠️
- Weight aggregation ใน MongoDB — ยากจะ unit test

### 4. telesale-service-mootui — `MakeCall()`
**Pseudocode:**
```
รับ operatorID, reserveID
→ ดึง reserve + verify owner
→ ตรวจ status ≠ Canceled
→ ดึง extension ที่ assign ให้ operator
→ เรียก VOIP provider.Dial()
→ บันทึก call record
```
**Edge Cases:** ถ้า VOIP provider fail → call record ไม่ถูกสร้าง → ไม่มี retry ⚠️
**โปรด note:** ใช้ domain errors (`ErrReserveNotFound`, etc.) — clean มาก ✅

### 5. GOTOPOPOFFICE — Multi-tenant ResourceService
**Pattern:**
```go
type Resource struct { DB *mongo.Database; RDB *redis.Client }
type ResourceService map[string]Resource  // keyed by service name
```
ทุก handler รับ `ResourceService` → ดึง resource ตาม service name ใน request

---

## 🗄️ Layer 4: Data & Integration

### Collections หลัก

| Project | Collections สำคัญ |
|---------|------------------|
| `3rd-payment` | payment_config, payment_statement_deposit/withdraw, bank_summary, ip_whitelist, order |
| `GOTOPOPOFFICE` | database_config (multi-tenant), member, deposit, withdraw, config_system |
| `topupserie_mootui` | banklist, member, statement, config_system (59KB model!), covid_wallet |
| `telesale` | provider, extension, extension_operator, group, reserve, call, session |
| `que_payment` | ใช้ collections เดียวกับ 3rd-payment |

### Redis Cache Policy

| Project | Key Pattern | TTL | Invalidate |
|---------|------------|-----|------------|
| `3rd-payment` | `thorlock:*` (distributed lock) | 180s | auto-expire |
| `3rd-payment` | bank support list | custom | `/clear-redis` endpoint |
| `GOTOPOPOFFICE` | config_system | manual | `/office/reset_redis` |
| `topupserie_mootui` | config cache | — | `/office/flushall-redis` ⚠️ no auth |
| `telesale` | group reserve | TTL ใน GroupService | cancel reserve |

### Third-party Integrations

| Integration | ใช้ใน | Auth Method |
|------------|-------|------------|
| 40+ Payment Providers | 3rd-payment, que_payment | API Key / HMAC |
| Telegram Bot API | 3rd-payment, GOTOPOPOFFICE | Bot Token |
| LINE Messaging | GOTOPOPOFFICE, topupserie | Channel Secret |
| VOIP Providers | telesale | callback token |
| AWS S3 | GOTOPOPOFFICE | AWS SDK |
| hash-central | GOTOPOPOFFICE, topupserie | API URL |
| OpenTelemetry/Jaeger | 3rd-payment, topupserie, GOTOPOPOFFICE | OTEL endpoint |
| ClickHouse | 3rd-payment | DB connection (analytics) |

---

## 🧹 Layer 5: Keep / Drop Summary

### 3rd-payment — 7/10
| Debt | ความเสี่ยง | แก้เมื่อ |
|------|-----------|---------|
| ไม่มี auth บน main endpoints | สูง ⚠️ | ก่อน migrate |
| RSA private key ใน .env plaintext | สูง ⚠️ | ก่อน migrate |
| Dockerfile target arm64 (ควรเป็น amd64) | กลาง | ก่อน deploy |
| RouteByPeer2Pay duplicate routes | ต่ำ | หลัง |
| mgo.v2 dependency เก่า (unused?) | ต่ำ | หลัง |

✅ **ลอก:** payment routing, thorlock distributed lock, callback idempotency, IP whitelist middleware, Telegram error notifier
🗑️ **โละ:** RouteBankTransferGateway (commented out), peer2pay/v3 duplicate routes

---

### GOTOPOPOFFICE — 5/10
| Debt | ความเสี่ยง | แก้เมื่อ |
|------|-----------|---------|
| `dgrijalva/jwt-go` CVE | สูง ⚠️ | ก่อน migrate |
| Gin 1.6.3 outdated | สูง | ก่อน migrate |
| Go 1.18 out of support | กลาง | ก่อน migrate |
| ไม่มี test files เลย | กลาง | หลัง |

✅ **ลอก:** ResourceService multi-tenant pattern, JWT auth middleware, Prometheus setup, domain-per-feature structure
🗑️ **โละ:** covid/ package (legacy), Jaeger-client-go (แทนด้วย OTel), demo/ package

---

### que_payment — 6/10
| Debt | ความเสี่ยง | แก้เมื่อ |
|------|-----------|---------|
| Nack(false,false) = drop message | กลาง | ก่อน migrate |
| StartAMQP() เก่าไม่มี context | กลาง | ก่อน migrate |
| SENTRY_DSN ว่าง | ต่ำ | หลัง |

✅ **ลอก:** ShutdownManager pattern (graceful shutdown ดีมาก), context-aware consumer loop, dual-mode (rabbit/cronjob)
🗑️ **โละ:** StartAMQP() เก่า, `quepub/cornjob.go` (typo ชื่อไฟล์)

---

### office-abatech — 4/10
| Debt | ความเสี่ยง | แก้เมื่อ |
|------|-----------|---------|
| `CLAUDE_API_KEY` ใน env block → ส่งไป browser! | สูง ⚠️ | ทันที |
| Node 16 EOL | สูง ⚠️ | ก่อน migrate |
| Nuxt 2 EOL (Dec 2024) | สูง ⚠️ | ก่อน migrate |
| node-sass deprecated | กลาง | หลัง |
| --legacy-peer-deps ใน Docker | กลาง | หลัง |

✅ **ลอก:** page/component structure, UI patterns, store structure
🗑️ **โละ/เขียนใหม่:** ทั้งหมด → migrate เป็น Nuxt 4/Vue 3 (ดู web-theme-nextgen เป็น template)

---

### topupserie_mootui — 5/10
| Debt | ความเสี่ยง | แก้เมื่อ |
|------|-----------|---------|
| `dgrijalva/jwt-go` CVE | สูง ⚠️ | ก่อน migrate |
| `/office/flushall-redis` ไม่มี auth | สูง ⚠️ | ทันที |
| Go 1.18 + Dockerfile Go 1.19 ไม่ match | กลาง | ก่อน migrate |
| go-redis v6 + v8 ทั้งคู่ | กลาง | ก่อน migrate |
| try.This() ซ่อน errors | กลาง | หลัง |
| ConfigSystem.go 59KB = God Object | กลาง | หลัง |

✅ **ลอก:** multi-platform routing (PROJECT env), rate limit middleware, bank weight algorithm, lotto sub-system
🗑️ **โละ:** covid/ sub-system, manucorporat/try, double Redis import

---

### telesale-service-mootui — 9/10 ✅
| Debt | ความเสี่ยง | แก้เมื่อ |
|------|-----------|---------|
| ไม่มี auth middleware (endpoints เปิดหมด) | สูง ⚠️ | ก่อน prod |
| VOIP provider error ไม่มี retry | กลาง | หลัง |

✅ **ลอก:** ทั้งหมด — clean hexagonal architecture, port/adapter pattern, repository interfaces, domain errors, DI ที่ main.go
🗑️ **โละ:** ไม่มี — ใช้เป็น template สำหรับ service ใหม่ได้เลย

---

## 🚨 Critical Action Items (เรียงตามด่วน)

| # | Action | Project | ความเสี่ยง |
|---|--------|---------|-----------|
| 1 | ย้าย `CLAUDE_API_KEY` ออกจาก nuxt.config.js env block | office-abatech | 🔴 ทันที |
| 2 | เพิ่ม auth ให้ `/office/flushall-redis` | topupserie_mootui | 🔴 ทันที |
| 3 | ย้าย RSA private key ออกจาก .env | 3rd-payment | 🔴 สูง |
| 4 | Upgrade `dgrijalva/jwt-go` → `golang-jwt/jwt/v5` | GOTOPOPOFFICE, topupserie | 🔴 สูง |
| 5 | เพิ่ม auth บน main payment endpoints | 3rd-payment | 🟡 กลาง |
| 6 | แก้ Nack(false,false) → Nack(false,true) หรือ DLQ | que_payment | 🟡 กลาง |
| 7 | Upgrade Gin 1.6.3 + Go 1.18 | GOTOPOPOFFICE, topupserie | 🟡 กลาง |
| 8 | Plan migration office-abatech → Nuxt 4 | office-abatech | 🟡 กลาง |
