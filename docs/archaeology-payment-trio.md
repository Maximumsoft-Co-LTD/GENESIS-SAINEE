# 🏺 System Archaeology — 3rd-payment, que_payment, 3rd-gateway

> วิเคราะห์จากการอ่านไฟล์จริง (2026-06-12) + อ้างอิง `docs/repo-map.md`
> หลักฐานแข็ง = อ่านโค้ดจริงพร้อม file:line | (?) = ยังไม่ยืนยัน 100%
> 📄 **ฉบับละเอียดเต็ม** (route ครบ 132 เส้น, provider verification matrix ~45 เจ้า, data dictionary, state machine, cron/handler detail, TrueWallet API inventory): `docs/archaeology-payment-trio.html`

---

## 🧭 Scope

- **โหมด:** วิเคราะห์ทั้ง 3 service แบบ deep-dive (Layer 0–6) แต่เนื่องจาก 3rd-payment (~71k บรรทัด, 313 ไฟล์) และ que_payment (~34k บรรทัด, 245 ไฟล์) ใหญ่มาก → เจาะเฉพาะ **4 flow หลัก**: ฝากเงิน (payin), callback ฝาก, ถอนเงิน (sync create → async process), เชื่อม/โอนเงิน TrueWallet (3rd-gateway)
- **3rd-gateway** (~5k บรรทัด) เล็กพอ → อ่านครอบคลุมเกือบทั้งหมด
- **Out of scope รอบนี้:** รายละเอียดครบทุก provider (มี ~67 ตัว — อ่านตัวแทน cloudpay/anypay/peer2pay), โมดูล dashboard + Telegram bot, bank-transfer-gateway module เชิงลึก, config ฝั่ง Indo, repo ปลายทาง (office-api-v10, go_report) นอกเหนือจากจุดเชื่อมต่อ

**ภาพรวม 1 บรรทัด:** `3rd-payment` = REST API aggregator คุยกับ payment provider ~67 เจ้า (สร้าง order ฝาก/ถอน + รับ callback) → ส่งงาน async ผ่าน RabbitMQ ให้ `que_payment` (worker ประมวลผล + callback กลับ office) ส่วน `3rd-gateway` = ตัวเก็บ/ใช้ credential กระเป๋า TrueWallet สำหรับโอนเงินอัตโนมัติ (ลูกค้าหลักคือ withdraw-auto)

---

## 📂 Layer 0: Runtime / Ops Reality

### 3rd-payment
- **Deploy:** Docker (Alpine) → CI build.yaml → Kaniko → JFrog → K8s GitOps; HTTP server **port 8081** (`app.go:150`)
- **Env-dependent behavior:**
  - `MODE != "production"` → โหลด `.env` (`_cmd/main.go:16`) และ **บังคับ RabbitMQ เป็น `amqp://guest:guest@127.0.0.1:5672`** (`service/rabbit/que-payment.go:48-50`)
  - `CHECK_IPWHITELIST=TRUE` → เปิด IP whitelist ที่ callback (`middleware/ipwhitelist.go:23`)
  - `MERCHANT_CODE` มีค่า → เปลี่ยนชื่อ queue เป็น `QUE_MERCHANT_<code>` แทน `QUE_PAYMENT_<paymentName>` (`que-payment.go:32-38`)
- **Connection strings:** MongoDB Atlas dev `test02.9oh7e.mongodb.net` / DB `demo_office` (docker-compose — **มี password อยู่ในไฟล์**), Redis (`REDIS_URL/PSW/DB`), RabbitMQ URL **เก็บใน MongoDB ต่อ payment config** (`service_qrpayment.rabbitmq_url`), OTel/Jaeger (`OTEL_CONNECT`), Telegram bot 2 ชุด (alert + oncall), `ENCRYPTION_ENDPOINT` (encryption service ภายนอก — ใช้กับ CLOUDPAY/VISAPAY/JIBPAYX)
- **Blast Radius:**
  - **Upstream (ใครเรียกเข้า):** office-api-v10 (`PAYMENT_API` 🟢), SCHEDULE_SERVICE_BANK (`PAYMENT_API` 🟡), payment providers ภายนอก ~67 เจ้า (webhook callbacks), topupserie ผ่าน office (?)
  - **Downstream (เรียกออก):** provider APIs ~67 เจ้า (HTTPS), go_report (`API_REPORT` HTTP 🟢), RabbitMQ ⟿ `QUE_PAYMENT_<NAME>` → **que_payment** (🟢 ชื่อ queue ตรงกับ consumer ฝั่ง que_payment), `ENCRYPTION_ENDPOINT`, Telegram API

### que_payment
- **Deploy:** Docker multi-stage (Go 1.25-alpine) — **ไม่ใช่ HTTP server, ไม่มี port** เป็น pure worker
- **จุดสำคัญ:** deploy แบบ **1 instance ต่อ 1 provider** — `PAYMENT_NAME` กำหนดชื่อ queue ที่ consume (`QUE_PAYMENT_<PAYMENT_NAME>`) และ `TYPE` เลือกโหมด `QUE_RABBIT` (consumer) หรือ `QUE_CRONJOB` (`app.go:95-167`)
- **Secret รั่วในไฟล์:** docker-compose hardcode `mongodb://useradmin:Aa112233@3.1.8.89:27017` ⚠️
- **Blast Radius:**
  - **Upstream:** 3rd-payment (ผ่าน RabbitMQ RPC — มี ReplyTo + CorrelationId)
  - **Downstream:** provider APIs (เหมือน 3rd-payment — services/ ซ้ำกัน ~60 ตัว), **office-api-v10** ผ่าน HTTP callback (`api/WithdrawCallback`, `api/Payment/PaymentCreateStatement/<service>`, `api/Payment/CallBackWithdrawAmount/<service>` — `services/callback/main.go:41-47`), `ENCRYPTION_ENDPOINT`, Telegram, Sentry

### 3rd-gateway
- **Deploy:** Docker (Go 1.23.3, TZ Asia/Bangkok), HTTP **port 8000** hardcode (`app.go:65`)
- **Env:** `AES_KEY_FIX32` (กุญแจ AES-GCM เข้ารหัส credential — **SPOF**), `PRIV_KEYSET`/`PUB_KEYSET` (Google Tink), `PROXY_URL` (proxy สำหรับยิง TrueWallet), Mongo, Redis (ประกาศแต่แทบไม่ใช้), **OTel ถูก comment ปิด** (`app.go:39-48`)
- **Blast Radius:**
  - **Upstream:** withdraw-auto (`URL_GATEWAY` 🟢), office-api-v10 (`GATEWAY_API` 🟡), running-bank (`URL_GATEWAY` 🟡)
  - **Downstream:** TrueWallet API ผ่าน proxy `https://api.tmn.one/proxy.dev.php/tmn-mobile-gateway/...`, **`https://dp.thezeus.tech/api/initWithdraw`** (K8s Helm installer — ยิงทุกครั้งที่ ConnectGateway สำเร็จ พร้อม **API key hardcode ในโค้ด** `install-helm.go:14`)

---

## 📂 Layer 1: โครงสร้าง & Entry Points

| | 3rd-payment | que_payment | 3rd-gateway |
|---|---|---|---|
| **Stack** | Go 1.25.5, Gin 1.11, mongo-driver 1.17.4, go-redis v8, streadway/amqp, otelgo, resty, shopspring/decimal (ใช้บางจุด), go-qrcode | Go 1.25.5, mongo-driver, go-redis **v9**, streadway/amqp, resty, samber/lo, decimal (ใช้แค่ anypay), Sentry | Go 1.23, Gin, **go-mongox** (ORM), **Google Tink** (AEAD/KMS), zap, go-redis v8 |
| **บทบาท** | Payment aggregator (sync API) | Payment worker (async) + cronjobs | Wallet-gateway credential vault + transfer executor |
| **Entry** | `_cmd/main.go` → `app.StartServ()` (`StartCronjob()` ถูก comment) | `_cmd/main.go` → ShutdownManager → `app.StartServ()` แยกโหมดด้วย `TYPE` | `_cmd/main.go` → `app.go` |
| **Module name** | `app` | `app` | `app` |

**Dependency ที่ควรระวัง:**
- `streadway/amqp` — archived/unmaintained (ควรเป็น `rabbitmq/amqp091-go`)
- `skip2/go-qrcode` — pseudo-version ปี 2020
- ทั้ง 3 repo ตั้งชื่อ module ว่า `app` เหมือนกันหมด — import path ชนกันถ้ารวม workspace
- ไม่มี test เลย (ยกเว้น `observability/classifier/classifier_test.go` ใน que_payment)

---

## 🔄 Layer 2: API Contracts & Data Flow

### 3rd-payment — รวม **~106 routes** (`routes/main.go` + 3 ไฟล์ route)

> เส้นทางเยอะมาก สรุปกลุ่ม core ที่ใช้งานจริง + ตัวแทน — provider routes ที่เหลือเป็น pattern เดียวกัน

| Method | Path | หน้าที่ | Auth/MW | Input | Output | สถานะ | พร้อม migrate? |
|---|---|---|---|---|---|---|---|
| POST | `/api/v2/payin` | **สร้าง order ฝาก (QR)** — flow หลัก | ไม่มี auth ❗ | ReqDeposit (service, payment_code, username, amount, office_api) | order_no, url_qrcode/qr_text, countdown | ใช้งาน | ⚠️ logic ดี แต่ต้องเพิ่ม auth + แตก switch provider |
| POST | `/api/v2/callback-payin/:payment_code` | **รับ webhook ฝากจาก provider** | IPWhitelist (เปิดด้วย env) | payload ต่างกันต่อ provider | ack | ใช้งาน | ❌ เขียนใหม่ — switch 16k บรรทัด |
| POST | `/api/v2/create-withdraw` | **สร้างรายการถอน** | ไม่มี auth ❗ | bank_code/number, amount | order_no | ใช้งาน | ⚠️ เหมือน payin |
| POST | `/api/v2/confirm-withdraw`, `/reconfirm-withdraw`, `/refund-withdraw-payment` | ยืนยัน/ทำซ้ำ/คืนเงินถอน | ไม่มี | order_no | status | ใช้งาน | ⚠️ |
| POST | `/api/v2/callback-payout/:payment_code` (+variant autopeer/umpay/hengpay) | รับ webhook ถอน | IPWhitelist | per-provider | ack | ใช้งาน | ❌ |
| POST | `/api/v2/create-config-payment`, `/update-config-payment`, GET `/get-config-payment/:code` | CRUD config provider (เก็บ secret ลง Mongo) | ไม่มี ❗ | ServicePayment (93 fields) | config | ใช้งาน | ⚠️ ต้องเข้ารหัส secret + auth |
| POST | `/api/v2/balance`, GET `/check/balance` | เช็คยอดคงเหลือ provider | ไม่มี | payment_code | balance | ใช้งาน | ✅ |
| POST | `/api/v2/{deposit,topup,withdraw,payout}/statement(V2)` | ดึงประวัติ | ไม่มี | filter | list | ใช้งาน | ✅ query ตรงไปตรงมา |
| POST | `/api/v2/confirm-transaction-payin(-slip)`, `/confirm-bank-deposit`, `/cancel-bank-deposit` | ยืนยัน/ยกเลิกฝาก manual (โดย office) | ไม่มี | order_no, slip | status | ใช้งาน | ⚠️ |
| POST | `/api/v2/adjust-balance`, `/change-status-deposit-payment` | แก้ยอด/สถานะ manual | ไม่มี ❗❗ | amount/status | — | ใช้งาน | ⚠️ อันตราย — ต้องมี auth + audit |
| POST | `/api/v2/request-resend-callback/:code/:order`, `/retry-payment-sendffice` | retry callback | ไม่มี | — | — | ใช้งาน | ✅ |
| `/peer2pay/v3/*` (10 เส้น) | กลุ่ม PEER2PAY แยก prefix (payin/payout/QR/history) | IPWhitelist เฉพาะ callback | — | — | ใช้งาน | ⚠️ ซ้ำกับ /api/v2 |
| `/bank-gateway/*` (15 เส้น) | โมดูล bank-transfer gateway (group/connect/verify-transfer/enc-aes) | ไม่มี | — | — | ใช้งาน(?) | ⚠️ ยังไม่ยืนยัน caller |
| `/api/v2/dashboard/*` (6), `/api/v2/bot/*` (4) | dashboard + Telegram bot webhook | AUTH_KEY+ALLOW_IP (dashboard) | — | — | ใช้งาน | ⚠️ |
| ANY | `/call-pm/:paymentCode/*byPath` | **proxy ยิงอะไรก็ได้ไปหา provider** | ไม่มี ❗ | arbitrary | arbitrary | ใช้งาน(?) | ❌ SSRF-prone |
| GET | `/api/v2/update-report-test`, `/filter-report-test2` | test routes ทิ้งไว้ใน prod | ไม่มี | — | — | **dead/test** | ❌ ลบ |

### que_payment — ไม่มี HTTP endpoint (worker)

### 3rd-gateway — 7 endpoints (`routes/main.go:26-40`)

| Method | Path | หน้าที่ | Auth | Input | Output | สถานะ | พร้อม migrate? |
|---|---|---|---|---|---|---|---|
| POST | `/api/register` | ลงทะเบียน service client → ออก ClientID+ApiKey | **ไม่มี** ❗ | name, service, project, domain | credential (เข้ารหัสด้วย key ที่ client ส่งมา) | ใช้งาน | ⚠️ ต้องมี admin auth/rate limit |
| GET | `/api/get-list-gateway` | list gateway + การเชื่อมต่อของ caller | Client-Id + API-Key header | — | configs + connections | ใช้งาน | ✅ |
| POST | `/api/get-connected-by-account` | ดู connection รายตัว + balance | เดียวกัน | gateway_code, bank_number/code | connection | ใช้งาน | ✅ |
| POST | `/api/connect-geteway` *(สะกดผิด)* | เชื่อมบัญชี TrueWallet (login PIN → เก็บ token เข้ารหัส) | เดียวกัน | connect_data (phone, keyId, loginToken, pin, deviceId) | balance + info | ใช้งาน | ⚠️ ตัด helm call ออก/ทำใหม่ |
| POST | `/api/disconnect-default-geteway` | ถอด connection ($pull จาก list_connect) | เดียวกัน | gateway_code, bank | ok | ใช้งาน | ✅ |
| POST | `/api/fund-transfer` | **โอนเงินออกจาก wallet** (P2P หรือเข้าบัญชีธนาคาร) | เดียวกัน | from/to account, amount | status, balance | ใช้งาน | ❌ มี bug กลับด้าน (ดู Layer 3) |
| POST | `/api/get-transactions` | ดึง statement วันนี้จาก wallet | เดียวกัน | bank_number/code | transactions | ใช้งาน | ✅ |

### Async paths

| ประเภท | Topic/Queue/Key | รับ payload | ส่งต่อไปไหน |
|---|---|---|---|
| RabbitMQ **producer** (3rd-payment, `service/rabbit/que-payment.go`) | `QUE_PAYMENT_<PaymentName>` หรือ `QUE_MERCHANT_<code>` — RPC pattern (ReplyTo + CorrelationId, timeout 120s) | `{id, service, type: REPORT_DATA_CURRENT/WITHDRAW/..., type_report, date, payment_code}` | → que_payment |
| RabbitMQ **consumer** (que_payment, `rabbitmqpub/amqp.go:48-222`) | `QUE_PAYMENT_<PAYMENT_NAME>` (durable=false ❗, prefetch=1, manual ack, **ไม่มี DLQ**, nack(requeue=true) วนไม่จำกัด) | ReqQue: type = DEPOSIT / WITHDRAW / PAYOUT / REFUND / REPORT_DATA_CURRENT / REPORT_DATA_OLD / MANUAL_CONFIRM_WITHDRAW / REFUND-WITHDRAW-OFFICE | dispatch → provider service → HTTP callback office + reply ReplyTo |
| Cron (que_payment, `TYPE=QUE_CRONJOB`) | `StartServV2` ทุก 20 วิ | poll collection `que_payment` (งานถอนค้าง) | spawn ≤10 workers → provider.Withdraw() |
| Cron | `ServRetryOrder` ทุก 5 นาที | withdraw status=16 ค้าง >3 วัน | expire → status=4 (cancel) |
| Cron | `AutomationRetryCallbackStatus` ทุก 30 นาที + Redis lock `*_statement_callback:<id>` TTL 10 นาที | รายการสำเร็จแล้วแต่ `callback_status != 1` เกิน 3 ชม. | retry HTTP callback → office |
| Cron | `StartUpdateRedis` ทุกวัน 23:59:59 | provider list | Redis key `payment_bank_support` |
| Cron (one-shot) | `BANK_SUMMARY_RECOVER` | rebuild `bank_summary` จาก logs | MongoDB |

### Inter-Service Flows

| Flow | Call Chain | หมายเหตุ |
|---|---|---|
| ฝากเงิน QR | `office-api-v10 → [POST /api/v2/payin] → 3rd-payment → 🔒Redis lock → provider API → 💾qr_payment → resp(QR)` แล้ว `provider ↩ [POST /callback-payin/:code] → 3rd-payment ⟿ QUE_PAYMENT_<NAME> → que_payment → [POST api/Payment/PaymentCreateStatement/<svc>] → office` | callback office ทำโดย que_payment ไม่ใช่ 3rd-payment |
| ถอนเงิน | `office → [POST /create-withdraw] → 3rd-payment → 💾withdraw_statement` → `[POST /confirm-withdraw]` ⟿ queue → `que_payment → provider.Withdraw() → ↩[POST api/WithdrawCallback] → office` | บาง provider ตัด balance จาก `bank_summary` ก่อนยิง แล้ว refund ถ้า fail |
| รายงาน | `3rd-payment → [HTTP API_REPORT] → go_report` (🟢 จาก repo-map) | แยกจากเส้น queue |
| ถอนผ่าน TrueWallet | `withdraw-auto → [POST /api/fund-transfer] → 3rd-gateway → InitAccount(decrypt token) → TrueWallet API (verify→confirm) → update balance → resp` | credential เข้ารหัส AES-GCM ใน `gateway_connect` |
| เชื่อมบัญชี wallet | `withdraw-auto/office → [POST /api/connect-geteway] → 3rd-gateway → TrueWallet PIN login → 💾gateway_connect + customer_registry → [POST dp.thezeus.tech/api/initWithdraw]` (helm auto-provision, error ถูกกลืน) | ยิง external installer ทุกครั้งที่ connect สำเร็จ |

---

## 🧠 Layer 3: Business Logic & Edge Cases (top 5)

### 1) `controller.Deposit` — 3rd-payment `controller/deposit.go:103` ✅ อ่านจริง

**Pseudocode:**
1. bind + validate (service/office_api/datetime/payment_code ห้ามว่าง)
2. ถ้า PaymentName=PEER2PAY → สลับ payment_code เป็น "PEER2PAY" ชั่วคราวเพื่อดึง config แล้วสลับกลับ + `BypassPeer2Pay(accessKey, secretKey)` (เชื่อ key จาก request ❗)
3. ดึง config จาก `service_qrpayment` → เช็ค service whitelist
4. **dedup:** ถ้ามี order เก่า (payment_code+service+username+amount ใกล้กัน) → คืน order เดิม `NewOrder=false`
5. **🔒 Redis lock** key=`<code>:<username>:<amount>` TTL **35 วินาที** — ไม่ได้ lock → ปฏิเสธ
6. แยก channel `DEPOSIT_BY_MEMBER` / `DEPOSIT_BY_AGENT` (จาก scope หรือ service==username) → เช็ค `DepositAgentOnly` / `IsNotTopUp`
7. เช็ค min/max → แปลง order_no เป็น SHA256 ตามจำนวนหลักเฉพาะ provider (E2P=10, CLOUDPAY=20, CUTPAYZ=16, ANYPAY=15, TRUSTPAY=random 10)
8. `SwitchHealthCheck` ยิง provider ก่อน → COREPAY มี flow สมัครสมาชิกแทรก → switch ตาม PaymentName เรียก provider → insert `qr_payment` → ตอบ QR

**Decision Table:**
| Given | When | Then |
|---|---|---|
| field ขาด | validate | 400 "ไม่พบข้อมูล" |
| config ไม่มี/ปิด | GetConfigPaymentV2 err | "ช่องทางฝากปิดชั่วคราว" |
| service ไม่อยู่ whitelist | CheckServiceWhitelist | "ไม่สามารถใช้งาน Payment นี้" |
| มี order ค้าง (เงื่อนไขเดียวกัน) | dedup | คืน order เดิม (ไม่สร้างใหม่) |
| lock ไม่ได้ | ObtainLock | "ไม่สามารถทำรายการได้ในขณะนี้" |
| member ฝากแต่ config agent-only (หรือกลับกัน) | check Option | ปฏิเสธ |
| amount นอก min/max | check | บอกช่วงที่ถูกต้อง |
| HENGPAY/IMPAY | — | บังคับ `qr_image` (`deposit.go:215-217`) |
| health check fail | SwitchHealthCheck | "ไม่สามารถใช้งาน payment ได้" |

**Edge cases / Debt:** lock 35s ไม่มี renewal — งานช้ากว่านั้น = ซ้ำได้; dedup ใช้ amount เป็น float ใน lock key (`%.2f`); MAANPAY/ZAPPAY/XPAY กลุ่มหนึ่งใช้ `AmountActual` แทน `AmountCreate` (`deposit.go:156-158`) — กติกาฝังในโค้ด

### 2) `controller.CallbackDeposit` — 3rd-payment `controller/callback.go` (ไฟล์เดียว ~16,000 บรรทัด ❗)

**Pseudocode:** อ่าน raw body + log ทุกอย่าง → IP whitelist (ถ้าเปิด) → ดึง config → **switch ยักษ์ตาม PaymentName** แต่ละ provider parse payload + verify เอง (บางเจ้า HMAC `x-signature` (ANYPAYTH), บางเจ้า RSA (CLOUDPAY/VISAPAY), **หลายเจ้าไม่ verify อะไรเลยนอกจาก IP**) → เทียบ TransactionRef กับที่เก็บตอน payin → status 99 (รอ manual) return เร็ว → สร้าง log `logs_callback_payment` → ⟿ publish RabbitMQ (RPC รอตอบ ≤120s) → update telegram log
**Edge:** ถ้า RabbitMQ fail แค่ log แล้วไปต่อ (`callback.go:274-282`) — order สำเร็จใน 3rd-payment แต่ office ไม่รู้ จนกว่า cron retry ฝั่ง que_payment จะช่วย; เทียบ amount แบบ `float64 ==` (เช่น `callback.go:1544`); parse เวลาเป็น `time.Local` ปนกับ UTC (`callback.go:1633`)
**Debt:** นี่คือไฟล์ที่ห้ามลอกไป v11 มากที่สุด — ต้องแตกเป็น interface ต่อ provider

### 3) `controller.CreateWithdraw` — 3rd-payment `controller/withdraw.go:38`

**Pseudocode:** validate → config → เช็คยอด: จาก `bank_summary` (ถ้าไม่มี → สร้างแล้ว `time.Sleep(5s)` ❗) หรือถ้า `UseDirectBalance` → ยิง balance API ตรง → ถ้า amount > balance ปฏิเสธ → เช็ค cooldown ระหว่าง payout (`NextTimePayoutByHashLayout`) → สร้าง `withdraw_statement` (status=0, hash unique) → switch provider → ตอบ order_no
**Edge:** `balance.Data.Token[0]` ไม่เช็คความยาว array (panic ได้); LUCKYPAY/SWIFTPAY ห้ามทศนิยม (`withdraw.go:111`)

### 4) `WithdrawBySpeedPay` (pattern หลักของ ~60 provider) — que_payment `controllers/withdraw.go:220-290`

**Pseudocode (6 ขั้นซ้ำทุก provider):**
1. `GetBalance()` จาก provider → 2. เทียบ limit/ยอด → 3. **หักเงินจาก `bank_summary` ก่อน** → 4. ยิง `provider.Withdraw()` → 5. ถ้า fail → **บวกเงินคืน** + status=13/12 → 6. สำเร็จ → status flow 16(processing)→15(awaiting)→1(success) + `callback.CallbackAllOffice()`

**Decision Table:**
| Given | When | Then |
|---|---|---|
| balance provider < amount | check | ปฏิเสธ ไม่หักเงิน |
| amount < MinWithdraw | check | ปฏิเสธ |
| provider ตอบ error | หลังหักเงินแล้ว | refund summary + status=13 |
| provider ตอบสำเร็จ | — | status=16/15 รอ callback ยืนยัน → 1 |
| ค้าง status=16 เกิน 3 วัน | cron ServRetryOrder | status=4 (expired) |
| callback office ล้มเหลว | resty retry 3 ครั้ง + cron 30 นาที | callback_status ค้าง 0 จน retry สำเร็จ |

**Edge/Debt:** หัก-คืนเงิน `bank_summary` ไม่ atomic กับการยิง provider (crash ระหว่างกลาง = เงินหาย/เกินใน summary); status เป็น magic number ไม่มี enum (`0,1,4,12,13,15,16`); เงินทั้งหมดเป็น `float64`; `_ =` ทิ้ง error ~30 จุด

### 5) `ConnectGateway` + `FundTransfer` — 3rd-gateway `controller/gateway.go:226,408` ✅ อ่านจริง

**ConnectGateway pseudocode:** auth ผ่าน middleware (Client-Id+API-Key → โหลด `customer_registry`) → หา `gateway_config` ตามสิทธิ์ (`allow_all_customers` / `group_ids` / `allowed_customer_ids` − `disallowed_customer_ids`, `gateway.go:252-263`) → `gwContext.ConnectGateway()` = login TrueWallet ด้วย PIN (hash SHA256(TmnID+Pin)) → ได้ AccessToken+ShieldID → เข้ารหัส AES-GCM ด้วย `AES_KEY_FIX32` → upsert `gateway_connect` → update/push `list_connect` ใน `customer_registry` (**2 ขั้น ไม่ atomic**, `gateway.go:305-350`) → **ยิง `CallK8SHelmByWithdraw` ไป dp.thezeus.tech — error แค่ println** (`gateway.go:359-362`)

**🐛 Bug ยืนยันแล้ว — `gateway-hub/tmn-one/main.go:172`:**
```go
resp.Status = confirmResult
if strings.Contains(resp.Status, "success") {   // ← กลับด้าน!
    return resp, fmt.Errorf("error confirm wallet")
}
```
`FundTransfer` (โอน wallet→wallet) คืน **error เมื่อโอนสำเร็จ** — เทียบ `FundTransferAccount` บรรทัด 201 ที่ใช้ `!strings.Contains` ถูกต้อง ผลคือ caller (withdraw-auto) เห็น 500 + `IsConfirm=true` ทั้งที่เงินออกแล้ว → เสี่ยง**โอนซ้ำ**ถ้า caller retry โดยไม่ดู IsConfirm

**Edge อื่น:** 401 จาก TrueWallet → `loginReset()` retry — เสี่ยง recursion ถ้า login fail ซ้ำ (`tmn-one/client.go:173-190`); PIN เก็บ plaintext ใน memory

---

## 🗄️ Layer 4: Data & Integration

### ER ย่อ (Mongo — ทุก service ใช้ DB เดียวกัน domain `demo_office`)

```
service_qrpayment (config ต่อ provider, 93 fields รวม secret plaintext)
   1 ──< qr_payment        (ฝาก: order_no, hash unique sparse, status, amount*, callback_status)
   1 ──< withdraw_statement (ถอน: hash unique, status magic numbers, before/after_credit)
   1 ──< bank_summary       (ยอดคงเหลือต่อ service+currency — ถูกหัก/คืนโดย que_payment)
qr_payment/withdraw_statement ──< logs_callback_payment, logs_que_payment (audit)
que_payment (collection คิวงานถอน — poll โดย cron StartServV2)
telegram_log_statement, adjust_statement, transaction_xpay_fail, confirm_bank_deposit

3rd-gateway (DB แยก domain):
gateway_config 1──< gateway_connect >──1 customer_registry (embed list_connect[])
                     └ credentials_token (AES-GCM, bson:"-" ปิดจาก JSON แต่อยู่ใน DB)
```

- **Migrations:** **ไม่พบทั้ง 3 repo** — schema เป็น implicit ใน struct; index สร้าง inline ตอน insert (เช่น hash unique sparse ใน `repository` ของ 3rd-payment) → "ชั้นดินประวัติศาสตร์" ดูได้จาก field ที่งอกใน `ServicePayment` 93 fields (FeeQrpay, HostSmartPayExternal, Option.*, DepositType.* ฯลฯ = ซากวิวัฒนาการต่อ provider)
- **Validation ชั้น data:** แทบไม่มี — พึ่ง unique index `hash` อย่างเดียว; status ไม่มี enum constraint; ไม่มี required ที่ DB
- **Redis:** 3rd-payment = distributed lock (35s), cache bank support list, cache token BigPay, trace correlation `trace:cb:<txn>:<code>`; que_payment = lock retry callback (10 นาที), `payment_bank_support` (refresh รายวัน); 3rd-gateway = ประกาศ thorlock แต่**ไม่ได้ใช้ใน route ไหนเลย**
- **Third-party auth:** ต่อ provider 3 แบบ — Bearer/API key, HMAC-SHA256, RSA sign (โดย**ส่ง private key ไปให้ `ENCRYPTION_ENDPOINT` ภายนอกช่วย sign/decrypt** — CLOUDPAY/VISAPAY/JIBPAYX ❗); TrueWallet ใช้ AES-CBC ชั้น app ทับ TLS + device fingerprint (DeviceID default = SHA256(เบอร์โทร))
- **Secret at rest:** credential provider ทั้งหมด (SecretKey, MerchantToken, IVCode...) เก็บ **plaintext ใน `service_qrpayment`**; 3rd-gateway ดีกว่า (AES-GCM) แต่ key เดียวจาก env

---

## 🧹 Layer 5: Keep / Drop Summary

### 🟢 รู้แน่ vs 🔴 ปริศนา

| หัวข้อ | สถานะ | หลักฐาน / สิ่งที่ยังไม่รู้ |
|---|---|---|
| flow ฝาก: payin → provider → callback → queue → que_payment → office | 🟢 รู้แน่ | trace ครบ `deposit.go:103` → `callback.go` → `que-payment.go:32-38` ↔ `amqp.go:48` → `callback/main.go:33-74` |
| flow ถอน + หัก/คืน bank_summary | 🟢 รู้แน่ | `withdraw.go` ทั้งสองฝั่ง + cron |
| bug FundTransfer กลับด้าน | 🟢 รู้แน่ | อ่านจริง `tmn-one/main.go:172` เทียบ `:201` |
| ใครเรียก `/bank-gateway/*` และ `/call-pm/*` | 🔴 ปริศนา | ไม่พบ caller ใน 25 repo — อาจเป็น external/ตายแล้ว |
| `dp.thezeus.tech/api/initWithdraw` ทำอะไรจริง | 🔴 ปริศนา | เห็นแค่ payload ฝั่งส่ง — ตัว installer อยู่นอก repo set |
| `ENCRYPTION_ENDPOINT` คือ service อะไร ใครดูแล | 🔴 ปริศนา | repo-map ยืนยันว่า "ไม่ใช่ hash-central" — รับ private key ผ่าน HTTP |
| ค่า env production จริง (host, whitelist IP) | 🔴 ปริศนา | อยู่ใน k8s manifest นอก repo |
| go_report consume queue `QUE_PAYMENT_*` ด้วยหรือไม่ | 🟡 ค่อนข้างแน่ว่าไม่ | ชื่อ queue ตรงกับ que_payment consumer; 3rd-payment → go_report เป็น HTTP `API_REPORT` |
| SCHEDULE_SERVICE_BANK → 3rd-payment ใช้ endpoint ไหน | 🟡 | เห็นแค่ env `PAYMENT_API` (repo-map 🟡) |

### Health Score

| Service | คะแนน | เหตุผลหัก |
|---|---|---|
| 3rd-payment | **3/10** | ไฟล์เดียว 16k บรรทัด, ไม่มี auth ภายใน, secret plaintext, float เงิน, ไม่มี test, dup ~40-50% |
| que_payment | **4/10** | โครง worker/shutdown/observability ดี แต่ magic status, ไม่มี DLQ, retry ไม่จำกัด, non-atomic balance |
| 3rd-gateway | **5/10** | เล็ก โครงสะอาดกว่า (mongox, Tink, AES-GCM) แต่มี bug จริง, hardcoded key, OTel ปิด, ไม่มี audit log |

### Route Reusability Summary (เฉพาะกลุ่มหลัก)

| Path | พร้อม migrate? | เหตุผล |
|---|---|---|
| `/api/v2/payin`, `/create-withdraw` | ⚠️ | business rule ครบดี แต่ต้องเพิ่ม auth, แตก provider switch เป็น interface, เปลี่ยนเงินเป็น decimal |
| `/api/v2/callback-*` | ❌ | เขียนใหม่เป็น per-provider adapter + signature verify บังคับ |
| statement/history (`/{deposit,withdraw}/statement*`) | ✅ | query ตรงไปตรงมา copy logic ได้ |
| config CRUD (`/create-config-payment`...) | ⚠️ | ต้องเข้ารหัส secret at rest + RBAC |
| `/adjust-balance`, `/change-status-*` | ⚠️ | ต้องมี auth + audit trail ก่อน |
| `/call-pm/*`, test routes | ❌ | ทิ้ง (SSRF/dead) |
| 3rd-gateway ทั้ง 7 เส้น | ✅/⚠️ | โครงดี — แก้ bug :172, ย้าย helm key เข้า env, เพิ่ม rate limit ที่ register |
| que_payment queue topology | ⚠️ | เพิ่ม durable queue + DLQ + max retry; pattern RPC ใช้ต่อได้ |

### Technical Debt Inventory

| รายการ | ความเสี่ยง | แก้เมื่อไร |
|---|---|---|
| 🐛 FundTransfer กลับด้าน (`tmn-one/main.go:172`) — สำเร็จแต่ตอบ error เสี่ยงโอนซ้ำ | **สูง** | **ก่อน — แก้ทันที (1 ตัวอักษร)** |
| Hardcoded API key dp.thezeus.tech (`install-helm.go:14`) + mongo creds ใน docker-compose | สูง | ก่อน — ย้ายเข้า secret manager + rotate |
| Endpoint การเงิน (`/payin`, `/adjust-balance`...) ไม่มี auth — พึ่ง network isolation | สูง | ก่อน/ระหว่าง migrate |
| Secret provider plaintext ใน MongoDB + ส่ง private key ให้ ENCRYPTION_ENDPOINT | สูง | ระหว่าง migrate (เข้ารหัส at rest) |
| float64 สำหรับเงิน + เทียบ `==` | สูง | หลัง (ใน v11 ใช้ decimal ตั้งแต่ต้น) |
| callback.go 16k บรรทัด / switch ~67 provider ซ้ำ 2 repo (dup ~40-50%) | กลาง | หลัง — รื้อเป็น provider interface ใน v11 |
| Queue ไม่ durable, ไม่มี DLQ, requeue ไม่จำกัด | กลาง | หลัง (ออกแบบใหม่ใน v11) |
| หัก/คืน bank_summary ไม่ atomic | กลาง | หลัง |
| Redis lock 35s ไม่มี renewal | กลาง | หลัง |
| timezone Local/UTC ปนกัน, `streadway/amqp` archived, ไม่มี test, OTel ปิดใน 3rd-gateway | ต่ำ-กลาง | หลัง |

### ต้องลอก (Core Business Value)
- กติกาธุรกิจใน Deposit/Withdraw: dedup order, Redis lock, channel member/agent, min/max, cooldown payout, fee/cost/profit breakdown (`amount_create/cost_deposit/cost_withdraw/profit`)
- ความรู้เชิง integration ต่อ provider 67 เจ้า (endpoint, sign scheme, ลูกเล่น order_no, quirks เช่น HENGPAY=qr_image) — **นี่คือทรัพย์สินหลักของระบบ**
- State machine ถอน (16→15→1 / 13→refund / 4=expired) + cron retry callback + bank summary recover
- โมเดล `gateway_config` allow-list หลายชั้น + การเข้ารหัส credential แบบ AES-GCM ของ 3rd-gateway
- RPC pattern ผ่าน RabbitMQ (CorrelationId/ReplyTo) — แนวคิดใช้ได้ แต่ implement ใหม่

### ควรโละทิ้ง / เขียนใหม่
- `callback.go` ทั้งไฟล์ → per-provider adapter + บังคับ signature verify
- switch provider ยักษ์ทั้ง 2 repo → `Provider` interface เดียว ลด dup 40-50%
- `/call-pm/*` proxy, test routes, `StartCronjob` ที่ comment ทิ้ง, `service/sudahpay` (dir หลงโครงสร้าง), thorlock ที่ไม่ได้ใช้ใน 3rd-gateway
- helminstall แบบ hardcode → event/config-driven provisioning

### ส่งต่อ sub-agent
| ประเด็น | ส่งต่อให้ |
|---|---|
| ไม่มี auth บน endpoint การเงิน + hardcoded secrets + ENCRYPTION_ENDPOINT รับ private key | `/security-review` |
| API ~106 เส้นไม่มี spec | prompt สร้าง OpenAPI จาก routes/main.go |
| bank_summary non-atomic + queue ไม่ durable | prompt ออกแบบ transactional outbox / idempotency ใน v11 |

---

## 🔍 Layer 6: Feature Deep-Dive

### 🔹 Feature: ฝากเงินผ่าน QR (Payin)

#### 1. ทำอะไรได้บ้าง
ลูกค้าขอฝากเงิน → ระบบเลือก provider ตาม payment_code สร้าง QR/บัญชีให้โอน → เมื่อจ่ายจริง provider ยิง webhook กลับ → ระบบเติมเครดิตให้ลูกค้าผ่าน office

#### 2. Full Flow
```
[1] Frontend/topupserie → office-api-v10
  ↓ HTTP POST /api/v2/payin  (3rd-payment :8081 — ไม่มี auth, เชื่อ req.OfficeAPI จาก caller)
[2] 3rd-payment (controller/deposit.go:103)
  - validate + whitelist service
  - 💾 MongoDB: qr_payment (เช็ค order ซ้ำ → คืน order เดิม)
  - 🔒 Redis lock <code>:<user>:<amount> 35s
  - SwitchHealthCheck → switch PaymentName
  ↓ HTTPS POST provider (เช่น CLOUDPAY /ty/orderPay — RSA sign ผ่าน ENCRYPTION_ENDPOINT)
[3] Provider → ตอบ qr_url/transaction_ref
[4] 3rd-payment 💾 insert qr_payment (status=0) → ตอบ QR ให้ user
[5] User สแกนจ่าย → Provider
  ↩ callback POST /api/v2/callback-payin/:payment_code  (IP whitelist ± signature ตาม provider)
[6] 3rd-payment (callback.go) — verify, update status, 💾 logs_callback_payment
  ⟿ queue:QUE_PAYMENT_<NAME>  (RPC, รอ reply ≤120s)
[7] que_payment (rabbitmqpub/amqp.go → controllers/deposit.go:94)
  - หัก cost: amount − (cost_deposit + cost_withdraw + profit)
  ↓ HTTP POST https://<office>/api/Payment/PaymentCreateStatement/<service>
[8] office-api-v10 เติมเครดิต → ตอบ code=0
[9] que_payment update callback_status=1 → reply ReplyTo → 3rd-payment ack
```

#### 3. Cross-repo map
| ลำดับ | Repo | บทบาท | Endpoint/Queue |
|---|---|---|---|
| 1 | office-api-v10 | ผู้เรียก + ผู้รับเครดิต | `PAYMENT_API/api/v2/payin`, รับ `api/Payment/PaymentCreateStatement/*` |
| 2 | 3rd-payment | สร้าง order, รับ callback | `/api/v2/payin`, `/callback-payin/:code` |
| 3 | que_payment | ประมวลผล + callback office | consume `QUE_PAYMENT_<NAME>` |
| 4 | go_report | รายงาน | HTTP `API_REPORT` |

#### 4. Happy vs Edge
| Scenario | ผลลัพธ์ | handled? |
|---|---|---|
| ฝากปกติ | ครบ 9 ขั้น | ✅ |
| กดซ้ำเร็ว | dedup + Redis lock | ✅ |
| callback ซ้ำจาก provider | เช็ค is_success/TransactionRef | ⚠️ ต่างกันต่อ provider บางตัวไม่เช็ค |
| RabbitMQ ล่มตอน callback | log แล้วไปต่อ — office ไม่รู้จน cron retry (3 ชม.+) | ⚠️ |
| คนปลอม callback (รู้ IP whitelist ปิด/spoof ได้) | หลาย provider ไม่มี signature | ❌ |
| งานช้ากว่า lock 35s | lock หลุด → สร้างซ้ำได้ | ❌ |

#### 5. จุดเสี่ยง
- เครดิตเข้า–ไม่เข้าขึ้นกับ HTTP callback ที่ retry ด้วย cron — ไม่มี exactly-once guarantee
- `callback_status=1` อาจถูก set แม้ callback จริงล้มเหลว (race ใน que_payment)

---

### 🔹 Feature: ถอนเงิน (Payout)

#### 1. ทำอะไรได้บ้าง
office สั่งถอนเงินลูกค้าผ่าน provider — สร้างรายการแบบ sync แล้วประมวลผลโอนจริงแบบ async พร้อมหักยอดจาก bank_summary

#### 2. Full Flow
```
[1] office-api-v10
  ↓ POST /api/v2/create-withdraw → 3rd-payment
[2] 3rd-payment (withdraw.go:38)
  - เช็คยอด bank_summary (หรือ balance API ตรง) + cooldown
  - 💾 withdraw_statement (status=0, hash unique)
  ↓ POST /confirm-withdraw ⟿ queue:QUE_PAYMENT_<NAME>
[3] que_payment (controllers/withdraw.go)
  - provider.GetBalance() → เช็ค limit
  - 💾 bank_summary −= amount   ← หักก่อนยิง
  ↓ HTTPS provider.Withdraw()
[4] Provider โอนเงิน → ตอบรับ (status=16/15)
  ↩ callback POST /api/v2/callback-payout/:code → 3rd-payment → update status=1, after_credit
[5] que_payment ↩ POST https://<office>/api/WithdrawCallback → office ตัดเครดิตจริง
[6] fail path: refund bank_summary, status=13/12; ค้าง 3 วัน → cron → status=4
```

#### 3. Cross-repo map
| ลำดับ | Repo | บทบาท | Endpoint/Queue |
|---|---|---|---|
| 1 | office-api-v10 | สั่งถอน + รับผล | `/create-withdraw`, รับ `api/WithdrawCallback` |
| 2 | 3rd-payment | สร้าง statement + รับ callback provider | `/callback-payout/:code` |
| 3 | que_payment | โอนจริง + จัดการ balance + retry | `QUE_PAYMENT_<NAME>`, cron ทุก 20วิ/5นาที/30นาที |

#### 4. Happy vs Edge
| Scenario | ผลลัพธ์ | handled? |
|---|---|---|
| ถอนปกติ | ครบ flow | ✅ |
| ยอดไม่พอ | ปฏิเสธก่อนหัก | ✅ |
| provider fail หลังหักยอด | refund summary | ⚠️ ไม่ atomic — crash กลางทาง = ยอดเพี้ยน |
| callback provider ไม่มา | ค้าง 16 → cron expire 3 วัน → status=4 | ⚠️ ช้ามาก |
| โอนซ้ำ (message requeue ไม่จำกัด) | ไม่มี max retry / DLQ | ❌ พึ่ง is_success เช็คใน handler |

#### 5. จุดเสี่ยง
- การหักเงิน `bank_summary` กับการยิง provider เป็นคนละ operation — ไม่มี transaction/outbox
- float64 ทุกจุดของเส้นเงิน

---

### 🔹 Feature: เชื่อมบัญชี TrueWallet + โอนเงิน (3rd-gateway)

#### 1. ทำอะไรได้บ้าง
service อื่น (หลักคือ withdraw-auto) ฝาก credential TrueWallet ไว้กับ 3rd-gateway (เข้ารหัส) แล้วสั่งโอนเงิน/ดึง statement ผ่าน API เดียว โดยไม่ต้องถือ credential เอง

#### 2. Full Flow
```
[1] withdraw-auto → POST /api/connect-geteway  (Client-Id + API-Key)
[2] 3rd-gateway (gateway.go:226)
  - เช็คสิทธิ์ gateway_config (allow_all / group / allowed-disallowed)
  ↓ HTTPS api.tmn.one (PIN login, AES-CBC ชั้น app, ผ่าน PROXY_URL ได้)
[3] TrueWallet → AccessToken + ShieldID
[4] 3rd-gateway
  - 🔐 AES-GCM(AES_KEY_FIX32) → 💾 gateway_connect.credentials_token
  - 💾 customer_registry.list_connect (set-or-push, ไม่ atomic)
  ↓ POST dp.thezeus.tech/api/initWithdraw (helm auto-provision, api-key hardcode, error ถูกกลืน)
[5] ตอบ balance + ข้อมูลบัญชี

โอนเงิน: POST /api/fund-transfer → InitAccount(decrypt) → verify→confirm ที่ TrueWallet
  → 🐛 path wallet→wallet: สำเร็จแต่ return error (main.go:172)
  → update credentials_token + total_balance → resp
```

#### 3. Cross-repo map
| ลำดับ | Repo | บทบาท | Endpoint |
|---|---|---|---|
| 1 | withdraw-auto | caller หลัก (ยืนยันจาก `controller/thirdgateway` + `URL_GATEWAY`) | `/api/fund-transfer`, `/connect-geteway` |
| 2 | office-api-v10 / running-bank | caller (🟡 ยังไม่ยืนยัน path) | `/get-list-gateway`, `/get-transactions` |
| 3 | external: api.tmn.one | TrueWallet proxy | mobile-gateway APIs |
| 4 | external: dp.thezeus.tech | K8s Helm provisioner | `/api/initWithdraw` |

#### 4. Happy vs Edge
| Scenario | ผลลัพธ์ | handled? |
|---|---|---|
| connect + โอนเข้าบัญชีธนาคาร | ครบ flow, อัปเดต token/balance | ✅ |
| โอน wallet→wallet สำเร็จ | **ตอบ 500 ทั้งที่เงินออก** | ❌ bug :172 |
| ต้อง face verification (-428) | มี flow face.go | ✅ |
| token หมดอายุ (401) | loginReset + retry | ⚠️ เสี่ยง loop |
| helm install fail | println ทิ้ง — withdraw service ไม่ถูก provision เงียบ ๆ | ❌ |
| spam /api/register | ไม่มี rate limit/auth | ❌ |

#### 5. จุดเสี่ยง
- `AES_KEY_FIX32` ตัวเดียวปลดล็อก credential ทุกบัญชี
- ไม่มี audit log ว่าใครสั่งโอนอะไรเมื่อไร

---

### 🔹 Feature: Cron กู้สถานะ & retry callback (que_payment)

#### 1. ทำอะไรได้บ้าง
ตาข่ายนิรภัยของระบบเงิน: poll งานถอนค้าง (ทุก 20 วิ, ≤10 workers), expire งานค้าง 3 วัน (ทุก 5 นาที), retry callback office ที่ไม่สำเร็จ (ทุก 30 นาที + Redis lock 10 นาที), rebuild bank_summary (one-shot), refresh provider list ลง Redis (รายวัน)

#### 2. จุดเสี่ยง
- สอง mechanism retry ซ้อนกัน (ServRetryOrder vs AutomationRetryCallbackStatus) — เส้นแบ่งหน้าที่ไม่ชัด
- ตัวเลขเวลา hardcode ทั้งหมด (3 ชม., 3 วัน, 20 วิ)
- ถ้า cron instance ไม่ถูก deploy (TYPE ผิด) → ไม่มีใคร retry เลย และไม่มี alert ว่า cron หาย

---

## 📌 สรุปสำหรับนักพัฒนาใหม่ (TL;DR)

1. **แก้ฝาก/ถอน** → เริ่มที่ `3rd-payment/controller/deposit.go:103` / `withdraw.go:38`; quirks ต่อ provider อยู่ใน `controller/<provider>/` และ switch ใน `main.go`/`callback.go`
2. **เงินเข้าเครดิตลูกค้า** เกิดที่ que_payment → office callback ไม่ใช่ที่ 3rd-payment — ถ้าเครดิตไม่เข้า ดู `callback_status` + cron retry
3. **config provider = MongoDB `service_qrpayment`** (รวม secret + RabbitMQ URL) — เปลี่ยน behavior ได้โดยไม่ deploy
4. que_payment deploy **ต่อ provider ละ instance** ผ่าน env `PAYMENT_NAME`/`TYPE`
5. **ห้ามลืม:** bug `tmn-one/main.go:172`, ไม่มี auth บน endpoint การเงิน, float64 = เงิน
