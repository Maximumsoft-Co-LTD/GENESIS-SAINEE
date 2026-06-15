# Migration Roadmap

> วิเคราะห์ ณ commit: `40367af` · วันที่: 2026-06-15
> สำหรับ **Option 1: Domain Service Consolidation + A2a Merged agent-svc**
> Constraints: 5 fullstack · downtime OK (1–2 min) · transformative · สิ้น มิ.ย. 2026 = design done

---

## Quick Wins — ทำได้ทันทีในโค้ดปัจจุบัน (ก่อนเริ่ม migration ใดๆ)

> ไม่ต้องรอ redesign เสร็จ ทำได้พรุ่งนี้ ลด risk 🔴 ทันที

| # | งาน | แก้ risk | Effort | DoD |
|---|-----|---------|--------|-----|
| QW-1 | **Rotate AWS IAM key** (AKIA5MF2WON2MLFDZG2I) + ย้ายไป AWS Secrets Manager / env injection | R-018 | S | New key ไม่อยู่ใน source; old key revoke; git-filter-repo ลบ history |
| QW-2 | **เปลี่ยน `durable=false` → `true`** บน QUE_PAYMENT, WALLETWITHDRAW, DYNAMICQUEUE, CALLBACK_LOTTO | R-022–R-024 | S (4 files) | Queue survive RabbitMQ restart; smoke test send/receive |
| QW-3 | **เพิ่ม DLQ** (`x-dead-letter-exchange`) บนทุก financial queue | R-022–R-024 | S | Failed messages land in DLQ; observable via management UI |
| QW-4 | **uncomment unique index** บน `transaction_statement` ใน hash-central | R-021 | S | Concurrent deposit test ไม่ duplicate |
| QW-5 | **บังคับ AUTHEN env** ใน kinglot-seamless: `if cfg.AUTHEN == "" { log.Fatal(...) }` | R-004 | S | kinglot startup fail fast ถ้าไม่ตั้ง AUTHEN; /api/v1/placebet ต้องมี token |
| QW-6 | **ลบ CHECK_IPWHITELIST bypass** ใน 3rd-payment | R-007 | S | env ไม่มีผลต่อ whitelist enforcement |
| QW-7 | **เพิ่ม Timeout: 30s** บน http.Client ทุกตัวที่ไม่มี (3rd-payment, que_payment, queue-withdraw-go, go_sms, SCHEDULE_SERVICE_BANK) | R-050 | S (grep-and-fix) | `http.Client{Timeout: 30 * time.Second}` ทุกที่ |
| QW-8 | **ย้าย RabbitMQ credentials** ออกจาก sms/publish.go → env var; ลบ legacy path | R-025 | S | sms/publish.go ไม่มี URL hardcode; legacy path ถูกลบ |
| QW-9 | **rotate Telegram bot token** SCHEDULE_SERVICE_BANK + ย้ายไป env | R-026 | S | token ไม่อยู่ใน source |
| QW-10 | **Pass `ctx` เข้า origin.Withdraw และ wallet.WithdrawWalletV2** | R-001 | M | ctx cancel ทำให้ debit abort; test timeout scenario |
| QW-11 | **เพิ่ม MongoDB transaction** รอบ 5 writes ใน wallet.WithdrawWalletV2 | R-002 | M | partial failure rollback; integration test |
| QW-12 | **เพิ่ม auth middleware** บน /api/v2/adjust-balance, /bank-gateway/call-by-req (ลบหรือ IP-restrict), /api/redis/flushall (coupon-service) | R-008, R-009, R-011 | S–M | endpoint ปิดจาก public; penetration test |

> **ลำดับ:** QW-1 → QW-2+3 → QW-5 → QW-6 → QW-8+9 → QW-10+11 (concurrent) → QW-4, QW-7, QW-12

---

## Phase 0 — Platform Foundation (สัปดาห์ที่ 1–2)

**ทำอะไร:** สร้าง infrastructure ที่ทุก service จะใช้ร่วม — ไม่ย้าย service ใดก่อน

### 0a. Secrets Management
- ตั้ง AWS Secrets Manager (หรือ Vault) สำหรับ production + staging
- กำหนด naming convention: `genesis/<env>/<service>/<key>`
- Migrate ทุก secret จาก source/docker-compose: DATABASE_URL, RABBITMQ_URL, REDIS_URL, JWT_SECRET, LINE tokens, payment API keys
- **ผล:** ไม่มี secret ใหม่ใน source code; credential rotation ไม่ต้อง redeploy

### 0b. RabbitMQ Cluster
- Setup 3-node RabbitMQ cluster (HA queue mirroring)
- กำหนด policy: `financial.*` exchanges → durable=true, ha-mode=all, dead-letter-exchange=`dlx.financial`
- Monitor: management plugin + Prometheus exporter
- **ผล:** broker restart ไม่ทำ in-flight message สูญ (แก้ P-02)

### 0c. Redis Sentinel
- Setup Redis Sentinel (1 master + 2 replica)
- กำหนด TTL policy สำหรับ session keys, distributed locks, dedup keys
- **ผล:** Redis failover ไม่กระทบ lock/session

### 0d. platform-lib skeleton (Go module)
สร้าง repository `genesis/platform-lib` พร้อม package structure:
```go
// platform-lib/money/tx.go — MongoDB transaction wrapper
func WithTransaction(ctx context.Context, session mongo.Session, fn func(ctx context.Context) error) error

// platform-lib/money/idempotency.go — UUID-based idempotency key + unique index helper
func EnsureIdempotency(ctx context.Context, coll *mongo.Collection, key string) (bool, error)

// platform-lib/queue/channel.go — channel factory
func NewDurableChannel(conn *amqp.Connection, queueName string, opts DurableOpts) (*amqp.Channel, error)
// DurableOpts: durable=true, DLX=dlx.financial, prefetch=1

// platform-lib/auth/middleware.go — Gin JWT middleware (golang-jwt/jwt)
func Required(secret string) gin.HandlerFunc

// platform-lib/http/client.go — http.Client factory
func New(timeout time.Duration) *http.Client  // default 30s

// platform-lib/observe/otel.go — OTel setup
func Init(serviceName, collectorURL string) (func(), error)
```

**DoD Phase 0:** Cluster running; secrets migrated; platform-lib v0.1.0 tagged; Quick Wins ทั้งหมดเสร็จ

| แก้ risk | effort |
|---------|--------|
| P-06 credentials hardcoded | M (infra setup) |
| P-02 non-durable queues | S (config policy) |
| QW-1–QW-12 | S–M (code patches) |

---

## Phase 1 — Fix Money Flows (สัปดาห์ที่ 3–5)

**ทำอะไร:** แก้จุดที่เงินหายได้ในโค้ดปัจจุบัน ก่อน migrate service ใดๆ — ใช้ platform-lib/money และ platform-lib/queue

### 1a. queue-withdraw-go
- ใช้ platform-lib/money/tx.go แทน 5 sequential writes (R-002)
- Pass ctx เข้า origin.Withdraw + wallet.WithdrawWalletV2 (R-001)
- ย้าย in-memory lock → platform-lib/money Redis distributed lock (R-027)
- เพิ่ม DLQ consumer: log + alert on dead-letter messages
- **Migration pattern:** patch-in-place; test ด้วย duplicate message scenario

### 1b. kinglot-seamless
- บังคับ unique index บน wallet_statement_lotto.Hash (R-006)
- แทน MongoDB advisory lock ด้วย platform-lib/money Redis lock + TTL (R-038)
- บังคับ AUTHEN env; เพิ่ม circuit breaker บน lotto provider HTTP call
- **DoD:** duplicate placebet → 2nd request rejected ด้วย unique index error

### 1c. go-agent-rocketwin
- เหมือน 1b (R-005, R-037)
- Document ว่า SERVICE_METHODS env กำหนดว่า binary ไหน active — บังคับให้ไม่ deploy ทั้งสองพร้อมกันใน service เดียวกัน

### 1d. hash-central
- uncomment unique index (QW-4)
- เปลี่ยน hash algorithm จาก SHA-1 → SHA-256 (R-058) + dual-write ช่วง migration เพื่อ backward compatibility
- เพิ่ม auth middleware บน /api/hashlayout-list (internal service token)

**DoD Phase 1:** ไม่มีจุดใน financial pipeline ที่ partial failure ทำให้ยอดเงินไม่สมดุลได้

---

## Phase 2 — Security Baseline (สัปดาห์ที่ 5–7)

**ทำอะไร:** ปิด unauthenticated endpoints ที่เปิดอยู่ ใช้ platform-lib/auth middleware

### 2a. API Gateway (Kong / Traefik)
- ติดตั้ง gateway ด้านหน้า office-api-v10, topupserie, agent services
- กำหนด route policies: JWT validation, rate limiting per service
- ย้าย CORS policy ไปที่ gateway (ลบ CORS wildcard จาก individual services)
- **ผล:** service-to-service calls ผ่าน internal network; external-only routes ผ่าน gateway

### 2b. office-api-v10 route cleanup
- เพิ่ม service-to-service token สำหรับ internal callbacks: /api/Payment/*, /api/WithdrawCallback, /api/auto/createQueueWithdraw (R-015)
- ลบหรือ IP-restrict /bank-gateway/call-by-req SSRF proxy (R-009)
- ลบ AUTHLOGIN=="DEMO" bypass (R-051)
- ลบ STATUS_REPORT env switch (R-052)

### 2c. Passwords + JWT
- hydra-affilaite: hash passwords ด้วย bcrypt (R-019)
- ย้ายทุก service ไป platform-lib/auth (golang-jwt/jwt) ลบ dgrijalva ออก (R-053–R-056)
- บังคับ JWT_SECRET ≥32 bytes ใน platform-lib/auth startup check

### 2d. coupon-service
- เพิ่ม auth บนทุก /api/coupon/* + /api/redeem/* routes
- ลบ DELETE /api/redis/flushall จาก production (หรือ admin-only + IP restrict)
- เพิ่ม Redis distributed lock รอบ redeem flow (R-012)

**DoD Phase 2:** OWASP Top 10 critical ปิดครบ; penetration test pass

---

## Phase 3 — Service Consolidation (สัปดาห์ที่ 8–14)

**ทำอะไร:** รวม repos ตาม Option 1 domain map — ใช้ strangler-fig per service group

### ลำดับ consolidation (risk ใหญ่สุดก่อน):

#### 3a. withdraw-svc (สัปดาห์ 8–9)
รวม `withdraw-auto` + `queue-withdraw-go`:
- สร้าง repo ใหม่ `withdraw-svc`
- import platform-lib/money, queue, observe
- ย้าย WALLETWITHDRAW consumer จาก queue-withdraw-go
- ย้าย WITHDRAW_{BankNumber} consumer จาก withdraw-auto
- **Migration:** dual-consume (ของเก่า+ใหม่ run พร้อมกัน) ทีละ queue; switch traffic; retire เก่า
- **ผล:** withdraw_statement เจ้าของเดียว; transaction ครบ; lock ถูก

#### 3b. deposit-svc (สัปดาห์ 9–11)
รวม `running-bank` + `go_deposit` + `hash-central` + `go_sms` + `SCHEDULE_SERVICE_BANK`:
- สร้าง `deposit-svc` พร้อม internal modules: bank_poller, credit_worker, dedup_service, sms_ingest, bank_scheduler
- hash-central เปลี่ยนเป็น internal function (ไม่ expose HTTP แยก)
- go_sms legacy path ถูกลบ; ใช้ queue-based path เท่านั้น
- per-bank goroutine isolation (แก้ R-030)
- **Migration:** strangler-fig — deploy deposit-svc; ค่อย route traffic ทีละ bank
- **ผล:** bank polling per-goroutine; dedup เป็น internal call; SHA-256

#### 3c. payment-svc (สัปดาห์ 10–12)
รวม `3rd-payment` + `que_payment`:
- สร้าง `payment-svc`: HTTP gateway + worker embedded
- 40+ provider packages รวมใน packages/ ไม่ซ้ำ
- RabbitMQ connection pool แทน per-request dial (R-042)
- **Migration:** route payment callbacks ผ่าน payment-svc ใหม่ แทน 3rd-payment; retire เก่า

#### 3d. agent-svc (สัปดาห์ 11–13)
รวม `go-agent-rocketwin` + `kinglot-seamless` (A2a):
- สร้าง `agent-svc` พร้อม rocketwin/ + kinglot/ packages
- CALLBACK_LOTTO consumer เดียว (ไม่ duplicate)
- IP allowlist ย้ายไป database; ไม่ hardcode
- wallet_statement_lotto ownership ชัด; unique index enforce ที่ db layer
- Clickhouse optional ยังคงเดิม (LOTTOKEY env)
- **Migration:** stop kinglot-seamless + go-agent-rocketwin พร้อมกัน; start agent-svc (downtime 1–2 min)

#### 3e. affiliate-svc (สัปดาห์ 12–14)
รวม `affiliate_standalone` + `hydra-affilaite` + `queue-dynamic-go`:
- คง hydra-hyperx-web เป็น SPA แยก แต่ call affiliate-svc โดยตรง
- ลบ dead code localhost:5053 (R-014)
- passwords → bcrypt
- DYNAMICQUEUE consumer rebuild ด้วย platform-lib/queue (manual ack + reconnect loop)

#### 3f. office-svc, report-svc, notification-svc (สัปดาห์ 13–16)
- `office-svc` = office-api-v10 + coupon-service (coupon routes เป็น /api/coupon/*)
- `report-svc` = go_report + sub_statement (ลบ cross-domain writes; go_report ส่ง event แทนเขียนตรง)
- `notification-svc` = go-linebot-system + cronjob-smsauto (LINE signature verify เพิ่ม; SMS credential จาก Secrets Manager)

**DoD Phase 3:** 25 repos → 10 deployables; ทุก service ใช้ platform-lib; ไม่มี shared DB cross-service

---

## Phase 4 — Frontend Modernization (สัปดาห์ 15–18)

**ทำอะไร:** ย้าย config hardcoded ออกจาก source; ลบ localStorage JWT

- `office-v10x`: ย้าย Firebase/Sentry/OAuth URL ไป VITE_* env; JWT → httpOnly cookie
- `theme-violet-game`: ย้าย DEV URL ไป VITE_*; ลบ commented URL ใน source
- `hydra-hyperx-web`: **upgrade Nuxt 2 → Nuxt 3** (ทีละ module ไม่ rewrite จากศูนย์); ลบ localhost:5053 hardcode; JWT → httpOnly cookie
  - ใช้ [Nuxt 3 migration guide](https://nuxt.com/docs/migration/overview): `pages/` + `composables/` + `pinia` แทน Vuex; ทดสอบทีละหน้า
- **Migration:** build pipeline inject VITE_* env; ไม่มี secret ใน dist bundle

---

## Phase 5 — Observability Full Coverage (continuous, ควบคู่กับ Phase 3–4)

- platform-lib/observe ติดตั้งใน ทุก service ใหม่ระหว่าง consolidation
- Jaeger tracing: trace ทุก deposit/withdraw/bet end-to-end
- Prometheus alerts: DLQ depth > 0 → alert; withdraw success rate < 99.9% → alert; agent-svc bet latency > 500ms → alert
- Structured logging ทุก service ด้วย zap (ไม่มี fmt.Println ใน production path)

---

## Open Questions (ต้องตัดสินใจก่อนเริ่ม Phase 3)

| # | คำถาม | สถานะ | ผลต่อ |
|---|-------|--------|-------|
| OQ-1 | `3rd-gateway` ยังใช้อยู่ไหม? | ✅ **ยังใช้** — คงไว้เป็น `gateway-svc` แยก ไม่รวมเข้า payment-svc | Phase 3c ไม่ต้อง migrate 3rd-gateway |
| OQ-2 | SMS/OTP queues ที่ go_sms publish → consumer อยู่ที่ไหน? | ✅ **ยอมรับเป็น orphan** — consumer อาจอยู่นอก monorepo; ทำงานเท่าที่มี; ทำ deposit-svc โดยไม่ต้อง map consumer | Phase 3b: deposit-svc migrate go_sms ตามปกติ; ทำ S-01 เป็น ⚪ accepted |
| OQ-3 | STATEMENT_{service} exchange ที่ office-api-v10 publish — consumer ไม่ยืนยัน | ✅ **ไม่สำคัญ** — ถือว่า dead code หรือ external; ไม่ต้อง map consumer ใหม่ | Phase 3f: office-svc คง publish ไว้; ไม่ต้อง refactor จนกว่าจะรู้ consumer |
| OQ-4 | hydra-hyperx-web (Nuxt 2 EOL) — upgrade หรือ rewrite? | ✅ **Upgrade Nuxt 2 → Nuxt 3** — ไม่ rewrite จากศูนย์; ใช้ migration guide ทีละ module | Phase 4 scope |
| OQ-5 | `database_config` migration → Secrets Manager: dual-read ช่วงไหน? | ✅ **ตัดสินใจแทน (ดูด้านล่าง):** คง database_config ไว้ระหว่าง Phase 3; migrate เป็น Secrets Manager ทีละ service ที่ถูก consolidate | Phase 3 per-service |
| OQ-6 | API Gateway — Kong, Traefik, หรือ custom? | ✅ **ตัดสินใจแทน (ดูด้านล่าง):** ใช้ **Traefik** — ไม่ต้องการทีม infra แยก; config-as-code ใน k8s/docker-compose | Phase 2a |

### OQ-5 Decision: database_config → Secrets Manager (step-by-step)

ไม่ต้อง migrate ทีเดียวทั้งหมด — ทำทีละ service ที่ถูก consolidate:

```
Phase 0  สร้าง Secrets Manager + naming convention genesis/<env>/<service>/<key>
         เพิ่ม secret ใหม่: DATABASE_URL, RABBITMQ_URL, REDIS_URL ต่อทุก service ใหม่
         ยังไม่แตะ database_config collection — service เก่ายังอ่านจากที่เดิม

Phase 3a (withdraw-svc)
         withdraw-svc ใหม่อ่านจาก Secrets Manager ตรง — ไม่ผ่าน database_config
         withdraw-auto + queue-withdraw-go เก่า retire → database_config entry ลบได้

Phase 3b (deposit-svc), 3c (payment-svc), 3d (agent-svc) ...
         ทำเหมือนกัน: service ใหม่ใช้ Secrets Manager; retire เก่า; ลบ entry database_config

Phase 3f (office-svc — เจ้าของ database_config)
         office-svc เปลี่ยนเป็น admin UI สำหรับ manage secrets (หรือใช้ AWS console โดยตรง)
         database_config collection เหลือเฉพาะ per-tenant business config (ไม่ใช่ DB credentials)
```

**ทำไม approach นี้:** ทีม 5 คนไม่ต้อง "big bang migrate" — ทุกครั้งที่ service ถูกรวม โยก secret ไปด้วยเลย ไม่มี dual-read period ซับซ้อน

### OQ-6 Decision: Traefik เป็น API Gateway

เหตุผลที่เลือก Traefik สำหรับทีม 5 fullstack ไม่มีทีม infra:

| เกณฑ์ | Traefik | Kong |
|-------|---------|------|
| Setup | single container + labels/annotations | ต้องการ PostgreSQL + admin setup |
| Config | เป็น code (docker-compose / k8s annotations) | UI หรือ Admin API แยกต่างหาก |
| Learning curve | ต่ำ — เหมือน reverse proxy | กลาง — plugin system, consumer/route/service concepts |
| Ops | zero-config service discovery (Docker/k8s) | ต้องการ operator หรือ admin |
| Auth plugin | JWT validation middleware พร้อมใช้ | JWT plugin พร้อมใช้ |

**การติดตั้งขั้นต่ำ (Phase 2a):**
```yaml
# docker-compose.yml (หรือ k8s IngressRoute)
traefik:
  image: traefik:v3
  command:
    - "--providers.docker=true"
    - "--entrypoints.web.address=:80"
    - "--entrypoints.websecure.address=:443"
  # service labels → route /office/* → office-svc:7777
  # JWT middleware → ตรวจ Authorization header ก่อน forward
```

---

## Summary Timeline (ภาพรวม 18 สัปดาห์)

```
สัปดาห์ 1–2   Phase 0  Platform Foundation (infra + secrets + platform-lib + Quick Wins)
สัปดาห์ 3–5   Phase 1  Money Flow Safety (withdraw-go + kinglot + agent + hash)
สัปดาห์ 5–7   Phase 2  Security Baseline (API gateway + auth + coupon)
สัปดาห์ 8–14  Phase 3  Service Consolidation (25→10 repos, strangler-fig per group)
สัปดาห์ 15–18 Phase 4  Frontend modernization
Continuous     Phase 5  Observability (ควบคู่ Phase 3 ทุก service ใหม่)
```

| Phase | แก้ปัญหา | Effort | ความเสี่ยงตอนย้าย | วิธีลดความเสี่ยง |
|-------|---------|--------|-----------------|----------------|
| QW | R-018,R-022–R-024,R-004,R-007,R-025,R-026 | 2 วัน | ต่ำ | Patch + deploy โดยตรง |
| 0 | P-06, P-02, P-11 | 2 สัปดาห์ | ต่ำ–กลาง | New infra, ไม่ย้าย service เก่า |
| 1 | P-03, P-04 | 3 สัปดาห์ | กลาง | Integration test + DLQ monitor |
| 2 | P-07, P-10 (partial) | 3 สัปดาห์ | ต่ำ | Canary deploy; backward compat token |
| 3 | P-01, P-05, P-08, P-09 | 7 สัปดาห์ | กลาง–สูง | Strangler-fig; dual-consume per service |
| 4 | P-10 | 4 สัปดาห์ | ต่ำ | Feature flag; build-time env injection |
| 5 | P-11 | continuous | ต่ำ | Add-only, non-breaking |
