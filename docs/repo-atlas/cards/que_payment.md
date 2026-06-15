# Card: que_payment
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go (worker / ไม่มี HTTP server) — RabbitMQ consumer + cronjob worker, MongoDB, Redis, OpenTelemetry (otelgo), Sentry, Telegram alert
- **module**: `app` (go.mod:1, go 1.25.5 — go.mod:3)
- **entrypoint**: `_cmd/main.go` → `app.StartServ()` (app.go:22) | godotenv โหลด `.env` เมื่อ `MODE!=production` (`_cmd/main.go:18`)
- **port**: ไม่พบ HTTP listen port — เป็น worker ล้วน เลือก mode ด้วย env `TYPE` (`QUE_RABBIT` / `QUE_CRONJOB`) (app.go:95, app.go:117). ไม่มี Dockerfile EXPOSE
- **Dockerfile**: golang:1.25-alpine3.22 builder → alpine:3.22, `CMD ["./app"]` (Dockerfile:1-13). ดาวน์โหลด rds-combined-ca-bundle.pem (Dockerfile:11)
- **CI**: `.github/workflows/build.yaml` NAME=queue-payment, self-hosted → Kaniko → JFrog (build.yaml:4)
- **bootstrap**: OTel init (app.go:30), Telegram notifier (app.go:39), Mongo connect 30s timeout (app.go:52-54), MongoDB event writer `monitor_payment` (app.go:63), Redis connect (app.go:80)

## provides
### http
ไม่พบ — repo นี้ไม่เปิด HTTP endpoint ใด ๆ (ไม่มี gin/route/server) — ทำงานผ่าน RabbitMQ + cronjob เท่านั้น

### consume (queues subscribed)
| exchange/queue (exact string) | handler | evidence |
|---|---|---|
| `QUE_PAYMENT_` + env `PAYMENT_NAME` (V2, default exchange `""`, durable=false, autoAck=false, Qos prefetch=1) | `StartAMQPV2WithContext` → `QueTypeSwitchV2` | rabbitmqpub/amqp.go:62, :84, :150 |
| `QUE_PAYMENT_` + env `SYSTEM_CODE` (V1 legacy `StartAMQP`) | `QueTypeSwitch` | rabbitmqpub/amqp.go:233, :269 |

**Message routing ใน QueTypeSwitchV2 (rabbitmqpub/amqp.go:310-341)** — แยกด้วย field `type` ใน JSON body `{id,service,type,type_report,date,payment_code}` (amqp.go:33-40):
- `DEPOSIT` → `controllers.DepositAgentV2` (amqp.go:312)
- `REFUND` → `controllers.RefundMoneyV2` (amqp.go:315)
- `WITHDRAW` / `PAYOUT` → `controllers.WithdrawAndPayOutV2` (amqp.go:325-328)
- `REPORT_DATA_CURRENT` / `REPORT_DATA_OLD` → `controllers.UpdateReport` (amqp.go:332)
- `MANUAL_CONFIRM_WITHDRAW` → `controllers.ManualConfirmWithdraw` (amqp.go:335)
- `REFUND-WITHDRAW-OFFICE` → `controllers.RefundWithdrawOffice` (amqp.go:338)
- ตอบกลับ publish ไป `d.ReplyTo` (default exchange) ด้วย CorrelationId (amqp.go:208) — RPC pattern

### cron (เมื่อ env `TYPE=QUE_CRONJOB`)
| schedule | does what | evidence |
|---|---|---|
| ทันที + ทุก 20s ticker | `quepub.StartServV2` — poll `que_payment` (status 0) แล้วส่งคิวกลับเข้า RabbitMQ ผ่าน `rabbit.CallRabbitWithdraw` (max 10 workers) | quepub/cornjob.go:38-64, :66; app.go:157-165 |
| ทันที + ทุก 5m ticker | `cronjob.ServRetryOrder` — ดึง withdraw status=16 ค้าง 5h-3d → ยกเลิก+callback office | cronjob/retry-order.go:32, :40, :61 |
| รายวัน 23:59:59 | `cronjob.StartUpdateRedis` — sync `service_qrpayment` → Redis key `payment_bank_support` | cronjob/updateredis.go:37, :59, :122; app.go:146 |
| ทันที + ทุก 10m ticker (เมื่อ `PAYMENT_NAME=BANK_SUMMARY_RECOVER`) | `cronjob.RecoverBankSummary` — ปลดล็อก `bank_summary` ที่ is_active=1 ค้าง >15 นาที (Redis lock `recover_bank_summary_lock`) | cronjob/recover_bank_summary.go:20, :30, :103; app.go:118-127 |
| ทันที + ทุก 30m ticker (เมื่อ `PAYMENT_NAME=AUTOMATION_RETRY_CALLBACK_STATUS`) | `cronjob.AutomationRetryCallbackStatus` — retry callback deposit (3h) + withdraw (6h) ไป office (Redis lock per order) | cronjob/automation_retry_callback_status.go:34, :36, :77, :144; app.go:128-137 |

## consumes
### http-out
URL ปลายทางส่วนใหญ่ดึงจาก **MongoDB config** (`service_qrpayment` → `HostSmartPay`, `APIQrpay`, `OfficeAPI` ฯลฯ) ไม่ใช่ env

| env key / source | base URL value | paths called | evidence |
|---|---|---|---|
| `objWdx.OfficeAPI` (จาก DB) | runtime 🟡 | `api/WithdrawCallback`, `api/Payment/PaymentCreateStatement/{service}`, `api/Payment/CallBackWithdrawAmount/{service}` | services/callback/main.go:41-47, :91, :130, :169 |
| `serviceConfig.HostSmartPay` (DB) | runtime 🟡 | provider-specific เช่น speedpay `/api/defray/V2` (speedpay/main.go:61), corepay `/trade/pay`, `/merchant/balance` (corepay/main.go:13,:58) | corepay/main.go:13, speedpay/main.go:61 |
| `serviceConfig.APIQrpay + PathCallbackWithdraw` (DB) | runtime 🟡 | callback URL ส่งให้ provider | speedpay/main.go:73 |
| ไม่มี env URL hardcode สำหรับ provider — ทั้งหมดมาจาก `models.ServicePayment` (DB) | — | — | models/service_payment.go:5-59 |

### publish
| exchange/queue (exact string) | payload | evidence |
|---|---|---|
| `QUE_PAYMENT_` + env `PAYMENT_NAME` (default exchange `""`, routing key = queue name) | `{id, type}` (objQue.ID.Hex(), objQue.Type) — cronjob ส่งคิวเข้า worker | services/rabbit/que-payment.go:87-97 |
| `d.ReplyTo` (default exchange, RPC reply) | `{code, message}` (ResQue) Expiration=1 | rabbitmqpub/amqp.go:208 |
| RabbitMQ URL: `RABBIT_MQ` (prod) / `RABBIT_MQ_DEVELOPMENT` (non-prod) / fallback `amqp://guest:guest@127.0.0.1:5672` | — | rabbitmqpub/conn.go:56-62; rabbit/que-payment.go:27-30 |

### data (MongoDB — DB name จาก env `DB_DBNAME`/`DB_ENDPOINT`, prod ใช้ `DB_ENDPOINT` ตรง; docker-compose: `abaoffice_db`)
| collection | R/W | evidence |
|---|---|---|
| `que_payment` | R/W (queue หลัก, status, recover) | repository/payment.go:32,:39,:86; paymentV2.go:238,:263; recover_bank_summary.go:95 |
| `qr_payment` | R/W (deposit statement) | repository/payment.go:96,:105,:113; paymentV2.go:45,:55,:109,:117 |
| `service_qrpayment` | R (config payment) | repository/payment.go:123,:132; paymentV2.go:20,:28,:36 |
| `withdraw_statement` | R/W (รายการถอน, comment, callback_status) | repository/payment.go:199,:206,:232; paymentV2.go:65,:73,:100,:124,:271,:372,:386,:418 |
| `bank_summary` | R/W (ยอดกระเป๋า `$inc current_balance`, lock is_active) | repository/payment.go:145,:154,:161,:177,:187; paymentV2.go:131,:161,:173,:200,:211 |
| `logs_que_payment` | W (timeline คิว) | paymentV2.go:228,:253; paymentV2.go:337 (InsertLogQue) |
| `logs_inc_payment` | W (audit การ inc ยอด) | deposit.go:381,:423; paymentV2.go:176,:191 |
| `logs_callback_office` | W (log การ callback office) | paymentV2.go:345 (InsertLogCallbackOffice) |
| `logs_callback_payment` | R (checkbank) | paymentV2.go:359 |
| `report_payment` | W (upsert/bulkWrite รายงานราย วัน, MongoDB transaction) | paymentV2.go:587,:640,:687; report.go:32-39 |
| `monitor_payment` | W (OTel error events, TTL 7 วัน) | observability/mongodb/writer.go:18,:52 |
- Redis: key `payment_bank_support` (updateredis.go:59), `recover_bank_summary_lock` (recover_bank_summary.go:20), `deposit_statement_callback:` / `withdraw_statement_callback:` lock (automation_retry_callback_status.go:28-29), `trace:cb:{orderNo}:{paymentCode}` traceparent cache TTL 30m (withdraw.go:376-379). Conn: `REDIS_URL`/`REDIS_PSW`/`REDIS_DB` (redis/connection.go:16-23)

### external (payment providers + office + alert)
| third-party | purpose | auth | evidence |
|---|---|---|---|
| **62+ payment providers** (แต่ละตัวเป็น package ใน `services/`): anypay, anypayth, apay24, askmepay, askpay, autopeerpay, azpay, beepay, bibpay, bigpay, bitpayz, blackdog, capitalpay, cloudpay, compay, corepay, cubixpay, cutpayz, dpay, e2p, first2pay, gbetpay, gm2pay, hengpay, hubpay, hyronpay, impay, jaijaipay, jibpayx, k2pay, kisspay, luckythai, maanpay, minerapay, mtmpay, mtpay, mypays24, onedaypay, onepay, p2cpay, p2wpay, papayapay, paykrub, peer2pay, sonicpay, speedpay, sugarpay, swiftpay, thunderpay, trustpay, umpay, visapay, wealthwave, worldpay, wowpay, xpay, zappay, zeenypay, zpay + `service/sudahpay` | ฝาก/ถอน/เช็คยอด ผ่าน HTTP | MerchantNo/SecretKey/AccessKey/sign จาก DB config | controllers/withdraw.go:7-67, :446-749; controllers/deposit.go:122-343; controllers/balance.go:55-120 |
| **Office backend** (URL = `OfficeAPI` ใน statement) | callback ผลฝาก/ถอน | ไม่มี auth header (POST JSON ตรง) | services/callback/main.go:33-73 |
| **Telegram Bot API** | แจ้ง error severity≥2 | `TELEGRAM_ERROR_BOT_TOKEN` | observability/telegram/notifier.go:47-72; app.go:39-43 |
| **Sentry** | crash reporting | `SENTRY_DSN` | _cmd/main.go:33-35 |
| **OTel collector / Jaeger** | tracing | `OTEL_URL`/`JAEGER_URL` | app.go:30; notifier.go:70 |

## observations
1. **Hardcoded MongoDB credential ใน docker-compose.yml** — `DB_ENDPOINT=mongodb://useradmin:Aa112233@3.1.8.89:27017/abaoffice_db` (docker-compose.yml:12) — username/password/host รั่วในไฟล์ใน repo
2. **Hardcoded BANK_NUMBER=2420260295** ใน docker-compose.yml:14
3. **RabbitMQ guest credential fallback hardcode** `amqp://guest:guest@127.0.0.1:5672` (rabbitmqpub/conn.go:61, rabbit/que-payment.go:29)
4. **Callback ไป office ไม่มีการ verify/sign ใด ๆ** — POST JSON ตรงไป URL ที่เก็บใน DB (`OfficeAPI`) ฝั่งรับไม่มีลายเซ็น (services/callback/main.go:33-73) — ปลอม callback ได้ถ้ารู้ URL
5. **balance update ไม่มี DB-level lock / ไม่ atomic แท้จริง** — `UpdateSummaryByServiceV2` ทำ GetOneDocument แล้วค่อย `$inc` แยก step (paymentV2.go:161,:185); การล็อกใช้ `bank_summary.is_active` flag (compare-and-set ผ่าน MatchedCount, paymentV2.go:211-214) ไม่ใช่ transaction — มี race window
6. **Withdraw flow: refund ยอดเมื่อ provider error แต่ถ้า refund เองล้มเหลวจะคืนเงินไม่ครบ** — เช็ค error ซ้อน error (withdraw.go:278-304) ถ้า `UpdateSummaryByServiceV2` refund fail จะ return error โดยเงินถูกหักไปแล้ว
7. **ไม่มี idempotency key ที่ระดับ message** — กันซ้ำด้วย status check (`IsSuccess==1`, withdraw.go:367; deposit.go:117) เท่านั้น ถ้า message มาซ้ำพร้อมกันก่อน status เปลี่ยน อาจ process ซ้ำ
8. **Weak crypto — corepay ใช้ AES-CBC + MD5 sign** — `EncryptAESCorepay` ใช้ CBC ไม่มี MAC (services/corepay/func.go:98-113), sign ใช้ MD5 สองชั้น (`Md5Sign`, func.go:91-96) — MD5 แตกง่าย, CBC ไม่มี integrity
9. **callback client มี retry 3 ครั้งแต่ไม่มี idempotency ฝั่งรับ** — newCallbackClient retry (services/callback/main.go:24-30) อาจยิง callback ซ้ำ
10. **Goroutine callback แบบ fire-and-forget ไม่เช็ค error** — `go callback.CallbackAllOffice(...)` (quepub/cornjob.go:243,:349; withdraw.go:333) — error ถูกทิ้ง
11. **MakeRequest legacy callback timeout 60s แต่ตัวอื่นไม่มี context cancel** — services/callback/main.go:202-218 ใช้ http.Client ตรง ไม่ผูก ctx
12. **dead code V1 จำนวนมาก** — `StartAMQP`/`QueTypeSwitch`/`QueRunning`/`UpdateAndcallback` (V1) ยังอยู่คู่ V2 (rabbitmqpub/amqp.go:225-308; quepub/cornjob.go:248-352) — ความเสี่ยงสับสน/แก้ไม่ครบสองที่
13. **Provider code อาจซ้ำกับ 3rd-payment** — package names ใน `services/` (corepay, speedpay, wowpay, ฯลฯ) ควรเทียบกับ 3rd-payment ว่า logic ฝาก/ถอน duplicate หรือไม่ (parent compare). corepay มีทั้ง encrypt(CBC)+MD5 sign+bankcode map ครบในตัว (func.go ทั้งไฟล์)
14. **DepositAgentV2 switch ขนาดใหญ่ทุก case เรียก `depositAgentBySpeedpay` เหมือนกันหมด** (deposit.go:122-343) — switch 200+ บรรทัดที่ไม่จำเป็น (ทุก provider ใช้ logic เดียวกัน) บ่งชี้ technical debt
15. **MongoDB connection timeout connectTimeout=5s แต่ disconnect ใช้ค่าเดียวกัน** (db/conn.go:14-15) — ไม่มี pool size config (commented out, db/conn.go:48-49)
16. **failOnError ใช้ log.Fatalf ใน path ที่ recoverable** — `StartAMQP` (V1) call failOnError ทำให้ทั้ง process ตาย (rabbitmqpub/amqp.go:27-31) — V2 แก้แล้วแต่ V1 ยังเสี่ยง
17. **เก็บ traceparent ลง Redis โดยไม่ encrypt** (withdraw.go:378) — low risk แต่ leak trace topology
