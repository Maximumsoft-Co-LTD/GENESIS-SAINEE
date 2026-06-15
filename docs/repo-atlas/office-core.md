# กลุ่ม office-core — สมองกลาง/แอดมิน

> วิเคราะห์: 2026-06-12 | commit: 40367af | [← กลับหน้าปก](README.md)

---

## ก) บทบาทของกลุ่ม

**เงินไหลผ่านกลุ่มนี้: ✅ ใช่ — และเป็นจุดอนุมัติเงินของทั้งระบบ** แอดมินกดอนุมัติฝาก (`/DepositAddCredit`), อนุมัติ/ยกเลิกถอน (`/withdraw-confirm*`), ปรับเครดิตมือ (`/credit-*`), อนุมัติบัญชีธนาคารสมาชิก (`/bank-member/approve`) ผ่านหน้าเว็บ office-v10x ทั้งหมดวิ่งเข้า office-api-v10 ซึ่งเป็น **hub กลางแบบ multi-tenant**: ถือ connection ไปยัง DB ของทุก tenant (โหลดจาก collection `database_config` ใน OFFICE DB ตอน boot — `db/db.go:121`) และเป็น **RabbitMQ publisher อย่างเดียว** (ไม่มี `ch.Consume` เลยทั้ง repo) — สั่งงาน worker กลุ่มอื่นผ่าน queue: ถอนอัตโนมัติ (`WITHDRAW_<bank>` → [bank-automation](bank-automation.md)), เติมเครดิต (`<SERVICE>` → [statement-pipeline](statement-pipeline.md)), report/side-effect (`STATEMENT_<SERVICE>`, `OUTSTANDING_<SERVICE>`)

| สมาชิก | บทบาท | stack | port |
|---|---|---|---|
| **office-api-v10** | back-office hub API ~900 routes ใน ~45 route group — deposit/withdraw/credit/bank/member/report/sms/config ทุกอย่างของแอดมิน; รับ callback การเงินจากภายนอกหลายเส้นแบบ **ไม่มี auth** | Go (Gin) + Mongo (raw) + Redis + RabbitMQ (pub-only) | `7777` hardcode (`server.go:106`) |
| **office-v10x** | หน้าเว็บแอดมิน (SPA) — 98 page routes, ยิง API **626 unique paths** ไปที่ office-api-v10; multi-tenant ผ่าน path segment `/<endpoint>/<webService()>` | Vite 4 + Vue 3 + Pinia + naive-ui (ไม่ใช่ Nuxt) | dev `8088` (`vite.config.ts:13`), prod `8080` static `serve -s` (`Dockerfile:23`) |
| **cronjob-smsauto** | cron ตลาด SMS — อ่าน config แคมเปญจาก OFFICE DB (`smsauto_statement`) ที่แอดมินตั้งผ่าน office, วน member ทุก tenant แล้วยิง SMSKUB/YingSMS; ไม่มี RabbitMQ | Go + gocron + Mongo + Redis (DB 5 hardcode) | env `PORT=5050` (`.env:3`) — compose map `4044:4044` ขัดกัน 🟡 (`docker-compose.yml:10`) |
| **go-linebot-system** | LINE Bot OA ระบบพนันมวย (เปิดคู่/รับเดิมพัน/ฝาก-ถอนผ่านแชท) multi-tenant — เขียน `deposit_statement` ใน tenant DB เดียวกับ office โดยตรง | Go (Gin) + Mongo (raw) + Redis + gocron | `4067` hardcode (`server.go:40`) |

ตัวเชื่อมภายในกลุ่มไม่ใช่ HTTP ระหว่างกันเป็นหลัก แต่เป็น **DB ร่วม**: ทั้ง cronjob-smsauto และ go-linebot-system ใช้ pattern `ResourceService` เดียวกับ office-api-v10 — boot แล้วอ่าน `database_config` จาก OFFICE DB เพื่อต่อ DB ทุก tenant (`cronjob-smsauto/_config/db.go:62-106`, `go-linebot-system/_config/db.go:65`)

---

## ข) แผนผังกลุ่ม

```mermaid
flowchart LR
    subgraph CORE["office-core"]
        FE["office-v10x<br/>(Vue3 SPA :8088/:8080)"]
        API["office-api-v10<br/>(Go hub :7777, ~900 routes)"]
        SMS["cronjob-smsauto<br/>(gocron :5050)"]
        LB["go-linebot-system<br/>(LINE bot มวย :4067)"]
        ODB[("Mongo OFFICE<br/>database_config, bank_config,<br/>smsauto_statement, log_sms_auto")]
        TDB[("Mongo tenant ทุก service<br/>config_system, member_account,<br/>deposit/withdraw_statement")]
        RDS[("Redis")]
    end

    FE -->|"626 paths /api/* + Bearer JWT<br/>(dev: localhost:7777, prod: origin+/api ผ่าน ingress)"| API
    API -->|"R/W"| ODB
    API -->|"R/W (283 จุด config_system, 132 member_account)"| TDB
    API -->|"cache config, lock ถอน (thorlock)"| RDS
    SMS -->|"R smsauto_statement / W log_sms_auto"| ODB
    SMS -->|"R member_account, report_active_user_*"| TDB
    SMS -->|"dedup sentPhoneNumber_* TTL 24h (DB 5)"| RDS
    LB -->|"R database_config, line_config"| ODB
    LB -->|"R/W member_account, deposit_statement,<br/>bet_statement, round_statement"| TDB
    LB -->|"configService_(key)"| RDS

    %% inbound ข้ามกลุ่ม
    TOPUP["topupserie"] -.->|"POST /api/auto/createQueueWithdraw<br/>/api/redis-peer2pay<br/>/api/slip-verify-deposit-statement-by-member"| API
    QWG["queue-withdraw-go"] -.->|"POST /api/auto/createQueueWithdraw"| API
    AFF["affiliate_standalone"] -.->|"POST /api/ManageWithdrawStatementRemain-auto-cornjobs<br/>(JWT key ตายตัวตรงกันสองฝั่ง)"| API
    P3["3rd-payment"] -.->|"↩ /api/WithdrawCallback<br/>/api/Payment/PaymentCreateStatement<br/>/api/Payment/CallBackWithdrawAmount<br/>/api/Xpay/CreateStatement"| API
    QP["que_payment"] -.->|"↩ /api/WithdrawCallback + /api/Payment/*"| API

    %% outbound ข้ามกลุ่ม
    API -.->|"PAYMENT_API /api/v2/get-bankconfig,<br/>get-payment-auto-withdraw, check_balance_p2p, bot/*"| P3
    API -.->|"/api/get-list-gateway, /api/get-connected-by-account,<br/>/api/disconnect-default-geteway"| G3["3rd-gateway"]
    API -.->|"/api/coupon/*"| CPN["coupon-service"]
    API -.->|"/api/get-channel, /api/change-channel,<br/>/api/save-device-botapp"| HC["hash-central"]
    API ==>|"⟿ exchange STATEMENT_(SVC) fanout<br/>type BANK/WITHDRAW/DEPOSIT/MIGRATECOUPONV2<br/>+ queue OUTSTANDING_(SVC)"| RPT["go_report / sub_statement"]
    API ==>|"⟿ queue (SVC)<br/>body=ObjectID / UPDATECONFIG"| GD["go_deposit"]
    API ==>|"⟿ queue WITHDRAW_(bankNumber)<br/>+ CHANGE_ACCOUNT_(bankNumber)"| WA["withdraw-auto"]

    %% external
    EXT1(["supercompany / sms-kub / oauth.game-auth.com<br/>LINE / Telegram / Facebook CAPI / EasySlip / S3 / Firebase"])
    EXT2(["console.sms-kub.com /api/messages<br/>YingSMS /v1/campaign"])
    EXT3(["api.line.me /v2/bot/*<br/>allapi.topupoffice.com /api/GetBankListConfig<br/>agent API (api_url จาก DB) 🟡"])
    EXT4(["Firebase RTDB x2 (withdraw-monitor + kill-switch)<br/>Sentry / LiveChat / S3 logo"])
    API -.-> EXT1
    SMS -.-> EXT2
    LB -.-> EXT3
    FE -.-> EXT4

    classDef faded fill:#f0f0f0,stroke:#bbb,color:#888
    classDef ext fill:#fff8e6,stroke:#d4a017,color:#7a5c00,stroke-dasharray: 5 5
    class TOPUP,QWG,AFF,P3,QP,G3,CPN,HC,RPT,GD,WA faded
    class EXT1,EXT2,EXT3,EXT4 ext
```

### Sequence: แอดมินอนุมัติถอนเงิน

> หมายเหตุยืนยันจากโค้ด: callback `POST /api/WithdrawCallback` มาจากฝั่ง **3rd-payment / que_payment** (payment-gateway mode) — ส่วน **withdraw-auto** (bank bot) รายงานกลับด้วยการ publish `STATEMENT_<SVC>` + เรียก `CallbackApiWithdraw + "/withdraw_statement"` (URL runtime จาก `database_config` ซึ่ง **ไม่ใช่** route ของ office — `withdraw-auto/service/report/callback.go:45-58`)

```mermaid
sequenceDiagram
    autonumber
    actor Admin
    participant FE as office-v10x
    participant API as office-api-v10 (:7777)
    participant MQ as RabbitMQ
    participant WA as withdraw-auto (กลุ่ม bank-automation)
    participant P3 as 3rd-payment / que_payment

    Admin->>FE: กดอนุมัติถอน (CardWithdrawDetail.vue:470-579)
    FE->>API: POST /api/withdraw-confirmauto/{svc} (Bearer JWT, perm WITHDRAW_AUTO)<br/>withdraw/index.ts:411 → route/withdraw.go:36
    API->>API: 💾 update withdraw_statement + ตัด bank_config.balance<br/>(withdraw.go:2880-2884 — ไม่อยู่ใน transaction ⚠️)
    alt ถอนผ่าน bot ธนาคาร
        API-)MQ: ⟿ queue WITHDRAW_{bankNumber} body={service,id}<br/>helper/queue.go:96-141 (เรียกจาก withdraw.go:1607,1776,2001,2873,3106)
        MQ-)WA: consume WITHDRAW_{bankNumber} (manager.go:393-401)
        WA->>WA: โอนเงินจริง (Selenium/Appium/Bank API)
        WA-)MQ: ⟿ STATEMENT_{SVC} type=WITHDRAW (service/report/publish.go:35-68)
        WA--)API: ↩ POST {CallbackApiWithdraw}/withdraw_statement (runtime URL 🟡 — ไม่พบ route นี้ใน office)
    else ถอนผ่าน payment gateway
        API->>P3: → PAYMENT_API /api/v2/get-payment-auto-withdraw (withdraw.go:4278)
        P3--)API: ↩ POST /api/WithdrawCallback (ไม่มี auth/signature ⚠️)<br/>3rd-payment service/callback/main.go:27 → route/withdraw.go:53
        API->>API: 💾 filter is_success≠1 → mark สำเร็จ (withdraw.go:3210-3220 — idempotency บางส่วน)
    end
```

---

## ค) ตาราง edge

### ภายในกลุ่ม

| from | to | ชนิด | path / queue / DB จริง | หลักฐานสองฝั่ง (file:line) | conf |
|---|---|---|---|---|---|
| office-v10x | office-api-v10 | HTTP sync | base dev `http://localhost:7777` + `/api` / prod `location.origin+/api`; 626 paths เช่น `POST /DepositAddCredit/{svc}`, `/withdraw-confirmauto/{svc}`, `/credit-return/{svc}`, `/bank-member/approve/{svc}` | FE: `base-url.ts:26,60`, `deposit/index.ts:350`, `withdraw/index.ts:411`, `credit/index.ts:27`, `members/index.ts:1249` ↔ API: `route/deposit.go:35`, `route/withdraw.go:36`, `route/credit.go:19`, `route/approveBank.go:21` | 🟢 (prod base inject ผ่าน ingress นอก repo 🟡) |
| office-api-v10 | cronjob-smsauto | INTRA-DB (config → cron) | แอดมินตั้งแคมเปญ → office **W** `smsauto_statement` (OFFICE DB) → cron **R** ทุกรอบ | office `controller/sms.go:102,117` ↔ cron `service/smsService.go:315` | 🟢 |
| cronjob-smsauto | office-api-v10 | INTRA-DB (log ← cron) | cron **W** `log_sms_auto` → office อ่านทำรายงาน SMS | cron `service/smsService.go:507-523` ↔ office `controller/sms.go:380,461` | 🟢 |
| cronjob-smsauto | (DB กลาง+tenant) | INTRA-DB | **R** `database_config` (boot ต่อทุก tenant), tenant `member_account`, `report_active_user_day/month`, `config_system` (SMS credential) | cron `_config/db.go:28-30,62-106`; `smsService.go:36,115,191,334` ↔ office โหลด `database_config` ชุดเดียวกัน `db/db.go:121` | 🟢 |
| go-linebot-system | (DB กลาง+tenant) | INTRA-DB | **R** `database_config`, `line_config`; tenant **R/W** `member_account`, `deposit_statement` (สร้างรายการฝาก/คืนเครดิต), `bet_statement`, `round_statement` | linebot `_config/db.go:65`, `Init.go:39,111`, `repositoryDB.go:533`, `handleAdmin.go:662,689` ↔ office อ่าน/เขียน collection เดียวกัน (`deposit.go`, card §data) | 🟢 |

### Interface ข้ามกลุ่ม — ขาเข้า (คนอื่นเรียก office-api-v10)

| from (กลุ่ม) | path | auth ฝั่ง office | หลักฐานสองฝั่ง | conf |
|---|---|---|---|---|
| topupserie ([customer-wallet](customer-wallet.md)) | `POST /api/auto/createQueueWithdraw/{svc}`, `POST /api/redis-peer2pay`, `POST /api/slip-verify-deposit-statement-by-member/{svc}` | **ไม่มี auth ทั้ง 3 เส้น** | topupserie `controller/withdraw.go:522`, `service/mainService.go:3726,3748`, `controller/slipQrcode.go:379` ↔ office `route/withdraw.go:59`, `route/statement.go:22`, `route/deposit.go:39` | 🟢 |
| queue-withdraw-go ([customer-wallet](customer-wallet.md)) | `POST /api/auto/createQueueWithdraw/{svc}` (ส่ง header `apiKey` แต่ office ไม่ตรวจ) | ไม่มี auth | qwg `wallet/withdraw.go:394-395,841-842`, `origin/withdraw.go:1195-1196` ↔ office `route/withdraw.go:59` | 🟢 |
| affiliate_standalone ([affiliate-promotion](affiliate-promotion.md)) | `POST /api/ManageWithdrawStatementRemain-auto-cornjobs/{svc}` | `CheckkeyCronJobNextDay` — JWT key hardcode `2ac714e8d1451244432013c2d7aa2fd7` **ตรงกันทั้งสองฝั่ง** | aff `service/affiliateRealtimeV2.go:1209` ↔ office `route/withdraw.go:22` + `auth.go:617-620` | 🟢 |
| 3rd-payment ([payment-gateway](payment-gateway.md)) | `↩ POST /api/WithdrawCallback`, `/api/Payment/PaymentCreateStatement/{svc}`, `/api/Payment/CallBackWithdrawAmount/{svc}`, `/api/Xpay/CreateStatement/{svc}` | **ไม่มี auth/signature** | 3rd-payment `service/callback/main.go:27,67,103,147` ↔ office `route/withdraw.go:53`, `route/payment.go:17-19` | 🟢 (URL ฝั่ง caller มาจาก DB runtime 🟡) |
| que_payment ([payment-gateway](payment-gateway.md)) | `↩ /api/WithdrawCallback`, `/api/Payment/PaymentCreateStatement/{svc}`, `/api/Payment/CallBackWithdrawAmount/{svc}` | ไม่มี auth | que_payment `services/callback/main.go:41-47,91,130,169` ↔ office route เดียวกัน | 🟢 |
| ~~go-agent-rocketwin~~ ([game-lotto](game-lotto.md)) | ~~`POST /api/manage/game/domain`~~ — **ตัดออก: ไม่ใช่ edge เข้า office** | — | route นี้**ไม่มี**ใน `office-api-v10/route/*` (grep ทั้งโฟลเดอร์ = 0); ตัว call จริงยิงไป `construct.APIURL` ของ KingLot (fallback `https://api.u4win.com` — `go-agent-rocketwin/controller/kinglot/game.go:409`) โดย env `APIOFFICE` ถูกใช้เป็นแค่ค่า `domain` ใน payload (`exernal.go:56-63,70`) | ❌ แก้แล้ว |

### Interface ข้ามกลุ่ม — ขาออก (office-api-v10 เรียก/publish)

| to (กลุ่ม) | ชนิด | path / queue จริง | หลักฐานสองฝั่ง | conf |
|---|---|---|---|---|
| 3rd-payment ([payment-gateway](payment-gateway.md)) | HTTP sync | `PAYMENT_API` → `/api/v2/get-bankconfig`, `/api/v2/get-payment-auto-withdraw`, `/api/v2/check_balance_p2p`, `/api/v2/callback-payin/{}-PEER2PAY`, `/api/v2/bot/*` | office `withdraw.go:4278`, `payment.go:48,241,1392`, `statement.go:1608` ↔ 3rd-payment card §provides `/api/v2/*` | 🟢 |
| 3rd-gateway ([payment-gateway](payment-gateway.md)) | HTTP sync | `GATEWAY_API` → `/api/get-list-gateway`, `/api/get-connected-by-account`, `/api/disconnect-default-geteway` | office `bankGateway.go:59,128,326+` | 🟢 (env ไม่มีใน .env, fallback hardcode 🟡) |
| coupon-service ([affiliate-promotion](affiliate-promotion.md)) | HTTP sync | `COUPON_SERVICE` → `/api/coupon/*`, `/api/statement/{}` | office `controller/extention.go:1095+` | 🟢 |
| hash-central | HTTP sync | `HASH_CENTRAL_API` → `/api/get-channel`, `/api/change-channel`, `/api/save-device-botapp` | office `bank.go:1668`, `optionBank.go:58,138`, `runbankFunc.go:626` | 🟢 |
| go_report + sub_statement ([statement-pipeline](statement-pipeline.md)) | ⟿ MQ | exchange `STATEMENT_<SVC>` (fanout) type `BANK`/`WITHDRAW`/`DEPOSIT`/`MIGRATECOUPONV2` + queue `OUTSTANDING_<SVC>` type `OUTSTANDINGUPDATE` | office `helper/queue.go:162-327, :359-371` ↔ go_report `rabbitmq/subscribe.go:60-104, :181-184` + sub_statement `pub/amqp.go:44-58` | 🟢 |
| go_deposit ([statement-pipeline](statement-pipeline.md)) | ⟿ MQ | queue `<SVC>` (durable) body=ObjectID hex (เติมเครดิต) / Type=`UPDATECONFIG` (สั่ง reload) | office `helper/queue.go:11-50, :56-90`, `runbankFunc.go:111` ↔ go_deposit `rabbitmq/worker.go:61-86, :106-132` | 🟢 |
| withdraw-auto ([bank-automation](bank-automation.md)) | ⟿ MQ | queue `WITHDRAW_<bankNumber>` + `CHANGE_ACCOUNT_<bankNumber>` body={service,id} | office `helper/queue.go:96-141`, `bank.go:2159-2160`, `withdraw.go:2264` ↔ withdraw-auto `rabbitmqpub/manager.go:437-460, :393-401` | 🟢 |

### External (นอกระบบ)

| ใคร | ปลายทาง | หลักฐาน |
|---|---|---|
| office-api-v10 | supercompany (`SUPERCOM_API` — permission fallback), sms-kub (`SMS_API`), `oauth.game-auth.com` (login gateway), LINE OAuth/Notify, Telegram Bot, Facebook CAPI, EasySlip, AWS S3 (key hardcode), Firebase RTDB (withdraw monitor), game agents (game24hr/918kiss/allbet) | `auth.go:179-181`, `controller/sms.go`, `callbackLogin.go:333,685`, `pixel.go:659`, `verifySlip.go:162`, `config.go:977`, `withdraw.go:3136`, `deposit.go:1701-1746` |
| cronjob-smsauto | `console.sms-kub.com` `/api/messages`, `/api/campaigns` (header `token` = password จาก `config_system` ราย tenant); YingSMS `/v1/campaign?service=hpx` (`X-Auth-Token`) | `service/smsService.go:385-407, :439-467`, `service/mainService.go:110-143` |
| go-linebot-system | `api.line.me` `/v2/bot/*` (reply/push/profile/webhook test); `allapi.topupoffice.com` `/api/GetBankListConfig`; agent API + web main API จาก `database_config.api_url`/`api_fontend_url` runtime 🟡 | `lineMessage.go:26-139`, `Init.go:312`, `mainService.go:211`, `agentService.go:22-492` |
| office-v10x | Firebase RTDB 2 โปรเจกต์ (withdraw-monitor + **kill-switch** redirect google.com), Sentry (DSN hardcode), LiveChat, S3 logo | `plugins/firebase.ts:13-34`, `mainLayout.vue:350-356`, `main.ts:60-78` |

---

## ง) Key flows

**สัญลักษณ์:** `→` HTTP sync | `⟿` queue | `↩` callback | `💾` DB

### Flow 1 — แอดมินอนุมัติถอนเงิน (เส้นเงินออกหลักของระบบ)

```
แอดมิน (office-v10x CardWithdrawDetail.vue:470-579)
  → POST /api/withdraw-confirmauto/{svc}  (withdraw/index.ts:411 → route/withdraw.go:36, perm WITHDRAW_AUTO)
  💾 office-api: update withdraw_statement + ตัด bank_config.balance (withdraw.go:2880-2884 — UpdateOne แยกกัน ไม่มี transaction)
  ── สาย bot ธนาคาร ──
  ⟿ queue WITHDRAW_<bankNumber> {service,id}  (helper/queue.go:96-141)
  → withdraw-auto consume (manager.go:393-401) → โอนจริงผ่าน Selenium/Appium/Bank API
  ⟿ withdraw-auto ตอบกลับทาง STATEMENT_<SVC> type=WITHDRAW (publish.go:35-68) → go_report/sub_statement
  ↩ + POST {CallbackApiWithdraw}/withdraw_statement (runtime URL 🟡 ไม่ใช่ route office)
  ── สาย payment gateway ──
  → office-api เรียก PAYMENT_API /api/v2/get-payment-auto-withdraw (withdraw.go:4278)
  ↩ 3rd-payment/que_payment ยิงกลับ POST /api/WithdrawCallback (ไม่มี auth — route/withdraw.go:53)
  💾 office-api filter is_success≠1 → mark สำเร็จ (controller/withdraw.go:3210-3220)
```

### Flow 2 — แอดมินเติมเครดิตจากรายการฝาก (ผ่าน queue `<SERVICE>`)

```
แอดมิน (office-v10x DepositView.vue / DepositTransection.vue)
  → POST /api/DepositAddCredit/{svc}  (deposit/index.ts:350 → route/deposit.go:35)
  💾 office-api บันทึก/ผูก statement แล้ว
  ⟿ queue <SERVICE> body = ObjectID hex ของ statement (helper/queue.go:11-50 CreateQueueAddCredit)
  → go_deposit consume (rabbitmq/worker.go:61-86, manual ack) → WorkerDeposit เติมเครดิตที่ agent
  💾 go_deposit update deposit_statement
  ⟿ go_deposit publish STATEMENT_<SVC> type=DEPOSIT + OUTSTANDING_<SVC> (deposit/publish.go:33-126)
  → go_report สร้าง report / sub_statement แจกโบนัส-โปรโมชั่น
```

### Flow 3 — SMS marketing cron (ไม่มีเงิน แต่แตะ PII ทุก tenant)

```
แอดมินตั้งแคมเปญใน office-v10x → office-api 💾 W smsauto_statement (controller/sms.go:102)
cronjob-smsauto ทุก 10 นาที (gocron UTC — smsauto.go:541-547)
  💾 R smsauto_statement (OFFICE) → R tenant member_account / report_active_user_* (smsService.go:36,115)
  💾 R config_system ของ tenant เอา SMS credential (smsService.go:334)
  → Redis dedup key sentPhoneNumber_<SVC>_<date> TTL 24h (smsauto.go:106-114 — ไม่ atomic)
  → POST console.sms-kub.com/api/messages (token header) หรือ YingSMS /v1/campaign (smsService.go:385-467)
  💾 W log_sms_auto พร้อม username_sms/password_sms plaintext (smsauto.go:74-75, smsService.go:507-523)
```

---

## จ) Risk Register

### 🔒 Security

| # | ความเสี่ยง | ระดับ | หลักฐาน | ผลกระทบ | ข้อเสนอ |
|---|---|---|---|---|---|
| S1 | **callback การเงินไม่มี auth/signature**: `POST /api/WithdrawCallback` (mark ถอนสำเร็จ), `/api/Payment/PaymentCreateStatement/{svc}`, `/api/Payment/CallBackWithdrawAmount/{svc}`, `/api/Xpay/CreateStatement/{svc}`, `/api/callback-update-status/withdraw` | 🔴 | `route/withdraw.go:53`, `route/payment.go:17-20`, handler `controller/withdraw.go:3202` (ไม่มี HMAC/signature ใด ๆ) | ใครเข้าถึง network ปลอม callback ปิดรายการถอน/สร้าง statement ฝากปลอม → เงิน/เครดิตเสียหายตรง ๆ | บังคับ HMAC signature + shared secret ต่อ partner, จำกัด source IP ที่ ingress |
| S2 | **สั่งถอน/แจกโบนัสได้โดยไม่ auth**: `POST /api/auto/createQueueWithdraw/{svc}` (สร้างคิวถอน), `POST /api/external-service/bonus/{svc}` (ลงทะเบียนก่อน `api.Use(AuthRequiredDB)`), `/api/CreateStatementTrueWallet`, `/api/QRCODE/CreateStatement`, `/api/webhook/verify-bank/:username`, bank callbacks `/api/withdraw_proxy_bank` ฯลฯ | 🔴 | `route/withdraw.go:59`, `route/externalService.go:12-13`, `route/statement.go:30-31`, `route/member.go:103`, `route/bank.go:101,108,110` | สร้างรายการเงินเข้า-ออกได้จากภายนอก | ย้ายเข้า group ที่มี auth, อย่างน้อยตรวจ `apiKey` ที่ caller (queue-withdraw-go) ส่งมาอยู่แล้วแต่ office ไม่เคยตรวจ |
| S3 | **`POST /api/redis-peer2pay` คืน `access_key`/`secret_key` ของ peer2pay โดยไม่มี auth** | 🔴 | `route/statement.go:22`, handler `controller/statement.go:3012-3037` | ขโมย credential payment ไปสั่งจ่ายเองได้ | ปิด endpoint หรือใส่ auth + ไม่คืน secret ตรง ๆ |
| S4 | **AWS keys + Mongo credential hardcode ใน source**: `AKIA5MF2WON2MLFDZG2I`/secret, อีกชุดใน uploadFile; `mongodb+srv://root:Zxcvasdf789@...` — credential เดียวกันโผล่ใน `.env` ของ cronjob-smsauto และ go-linebot-system ด้วย | 🔴 | office `config.go:977-978`, `helper.go:666-667`, `uploadFile/uploadfiles.go:20-21`, `db/db.go:85,199`; cron `.env:2`; linebot `.env:1` + `docker-compose.yml:10-16` | live credential รั่วทั้ง S3 และ DB กลาง (ทุก tenant) แค่อ่าน repo | rotate ทันที, ย้ายเข้า secret manager, ลบออกจาก git history |
| S5 | **LINE webhook ไม่ verify signature** — `/webhook/:key` ประมวล events ตรง ๆ ไม่เช็ค `X-Line-Signature` ทั้งที่ `secret_key` เก็บไว้ใน `line_config` แล้วแต่ไม่เคยใช้; สิทธิ์ admin เช็คแค่ LINE userId ใน body ที่ปลอมได้ → trigger ฝาก/คืนเครดิต/ถอน (สร้าง `deposit_statement` + `AgentDeposit`) | 🔴 | linebot `route.go:18`, `webhook.go:15,87-117`, `handleAdmin.go:662,669`, `models/lineConfig.go:31` | ปลอม event เป็นแอดมิน → เติมเครดิต/ถอนเงินใน tenant DB จริง | verify signature ด้วย channel secret ก่อน parse body |
| S6 | **JWT key cron hardcode ใช้ร่วมข้าม repo**: `2ac714e8d1451244432013c2d7aa2fd7` (เทียบกับ `147ce3cba2...`) ป้องกัน endpoint จัดการทุนถอนข้ามวัน — ค่าเดียวกันฝังใน affiliate_standalone | 🟠 | office `auth.go:617-620`, `route/withdraw.go:22` ↔ aff `service/affiliateRealtimeV2.go:1209` | key รั่วครั้งเดียว = เรียก endpoint เงินได้ตลอด (rotate ไม่ได้โดยไม่ deploy 2 repo) | ย้ายเป็น env + key rotation |
| S7 | **`.env` commit พร้อม secret จริง**: `ACCESS_SECRET=@OFFICEV10` (สั้น/เดาง่าย), `SECRET_KEY_2FA`, `TELEGRAM_TOKEN_SECRET`, `THIRD_P2P_KEY`; linebot `ACCESS_SECRET=ABAsercretPayload`=`API_TOKEN` | 🟠 | office `.env` ทั้งไฟล์ (card §observations); linebot `.env:4,7` | ปลอม JWT ของทั้ง office และ linebot report ได้ | rotate + ออกจาก repo |
| S8 | **`/test/demo2?service=X` เปิดสาธารณะ คืนชื่อ+เบอร์โทรลูกค้า VIP ทุก tenant** (PII leak) + CORS `*` | 🟠 | cron `route.go:14`, `controller/testfunc.go:29-44`, `server.go:40-42` | PII ลูกค้า high-value รั่วทั้งระบบ ใช้ทำ phishing/SIM-swap ได้ | ลบ endpoint ทดสอบออกจาก prod |
| S9 | **SMS credential ถูกเขียนลง `log_sms_auto` เป็น plaintext** (`username_sms`/`password_sms` ทุก log) | 🟠 | cron `controller/smsauto.go:74-75,201-202,329-330,467-468` | ใครอ่าน collection log ได้ = ได้ credential ผู้ให้บริการ SMS ทุก tenant | ตัด field ออกจาก log |
| S10 | **CORS `*` ทุก service ในกลุ่ม** รวม endpoint การเงิน | 🟡 | office `server.go:111-113`; linebot `server.go:46-49`; cron `server.go:40-42` | ประกอบร่างกับ S1/S2 ให้โจมตีจาก browser ได้ | allowlist origin |
| S11 | **สิทธิ์ฝั่ง FE เป็นแค่การซ่อน UI**: decode JWT เองใน browser, `checkPermission` เทียบ store ใน memory, โหมด dev มี `bypass` คืนสิทธิ์เต็ม; token เก็บ localStorage (XSS-stealable) — แปลว่า endpoint เงินที่ backend ไม่ auth (S1/S2) ไม่มีชั้นป้องกันอื่นเลย | 🟠 | FE `router/index.ts:916-930`, `helper-function.ts:466-474, :372`, `api-config.ts:22` | ข้าม UI ยิง API ตรงได้เสมอ | ถือว่า backend คือ boundary เดียว → ปิดช่อง S1/S2 ก่อน |
| S12 | **permission check ใน payment callback ถูก comment ทิ้ง** | 🟠 | office `controller/payment.go:1396-1403` | เส้นสร้าง statement payment เหลือ unauthenticated | เปิด check กลับ/เขียนใหม่ |
| S13 | **endpoint admin/migrate ไม่ auth**: `/api/init-service`, `/api/update-configsystem/{svc}`, `/api/clear-redis-service/:key` | 🟠 | `route/config.go:48-49`, `route/migrate.go:15`, `route/report.go:28` | สร้าง/แก้ config tenant หรือเคลียร์ cache จากภายนอก | ใส่ auth + แยก network ภายใน |

### ⚙️ Reliability

| # | ความเสี่ยง | ระดับ | หลักฐาน | ผลกระทบ | ข้อเสนอ |
|---|---|---|---|---|---|
| R1 | **ตัด balance นอก transaction**: `bank_config.balance = balance - amount` เป็น UpdateOne แยกจาก update `withdraw_statement`; Redis lock (thorlock) ครอบเฉพาะ flow cancel | 🔴 | office `withdraw.go:2880-2884`, lock เฉพาะ `withdraw.go:1590` | crash/race ระหว่างสองสเต็ป = ยอดเงินกับสถานะไม่ตรงกัน (เงินจริง) | Mongo multi-doc transaction หรือ lock ครอบทั้ง flow |
| R2 | **queue publish กลืน error เงียบ**: amqp.Dial/Channel fail → `return` เฉย ๆ ไม่มี log/retry — ข้อความสั่งถอน/เติมเครดิตหายโดยไม่มีใครรู้ | 🟠 | office `helper/queue.go:13-22, :102-110`, `runbankFunc.go:113-124` | แอดมินกดอนุมัติแล้วแต่คำสั่งไม่ถึง worker — รายการค้าง | log + error response + outbox pattern |
| R3 | **HTTP client ไม่มี timeout เกือบทั้งกลุ่ม**: office `&http.Client{}` หลายสิบจุด; cron client เปล่าใน CallAPIALL/CallAPIYingSms | 🟠 | office `accountOption.go:79`, `bank.go:512,694,4519,4702`, `bankGateway.go:69,142,225`; cron `mainService.go:116,135` | ปลายทางค้าง = goroutine/cron ค้างไม่จำกัด | ตั้ง timeout กลาง |
| R4 | **cron `panic(err)` ใน HTTP client โดยไม่มี recover** — SMS API ล่มทำ process ตายทั้งตัว | 🟠 | cron `service/mainService.go:119,138` | แคมเปญ SMS ทุก tenant หยุดเพราะ tenant เดียว | คืน error แทน panic |
| R5 | **linebot ฝากเงินแบบ create-then-update ไม่ atomic ไม่มี idempotency**: สร้าง `deposit_statement` → `AgentDeposit` → ถ้า update fail ค่อย `AgentWithdraw` คืน | 🟠 | linebot `handleAdmin.go:662,669,~700` | window ที่เครดิตเข้าแล้วแต่ statement ค้าง process | idempotency key + saga ที่ตรวจสอบได้ |
| R6 | **tenant connection สร้างครั้งเดียวตอน boot**: tenant ต่อไม่ได้แค่ print แล้วข้าม → `resource[service]` nil → nil map access ใน cron; linebot `ConfigInit` panic ทั้ง process ถ้า set redis fail ของ tenant เดียว | 🟡 | cron `_config/db.go:96-99`, `controller/smsauto.go:42`; linebot `Init.go:135,142`, `server.go:23-27` | เพิ่ม tenant ต้อง restart; tenant เสียตัวเดียวพา service ล่ม | เช็ค nil + lazy reconnect |
| R7 | **`FixIsLoginTelegramLine` reset secret 2FA ของพนักงาน TELEGRAM/LINE ทุกครั้งที่ boot** (side-effect ตอน start) | 🟡 | office `server.go:101`, `helper.go:1333` | restart ทีไรพนักงานต้องผูกใหม่ | ย้ายเป็น migration ครั้งเดียว |

### 💾 Data

| # | ความเสี่ยง | ระดับ | หลักฐาน | ผลกระทบ | ข้อเสนอ |
|---|---|---|---|---|---|
| D1 | **dedup SMS พึ่ง Redis ตัวเดียวและไม่ atomic** (Get→append→Set) + cron type1/type3 รันชนกันทุก 10 นาที; Redis flush = ส่งซ้ำทั้ง batch | 🟡 | cron `controller/smsauto.go:106-114` | SMS ซ้ำ = ต้นทุน + ลูกค้ารำคาญ | SETNX ราย phone หรือ persist ลง Mongo |
| D2 | **3 repo เขียน collection เดียวกันโดยไม่มี contract**: office, linebot (deposit_statement W), cron (R member_account) แตะ tenant DB ตรง ๆ — schema เปลี่ยนฝั่งเดียวพังเงียบ | 🟠 | office card §data; linebot `repositoryDB.go:533`; cron `smsService.go:36` | data corruption ข้าม service ตรวจยาก | รวม write path ผ่าน API เดียว หรือ schema registry |
| D3 | **Redis ไม่มี password ทั้งกลุ่ม**: office hardcode `Password:""` (ทั้งที่มี env `REDIS_PSW`), cron DB 5 hardcode, linebot `Password:""` + REDIS_URL เป็น IP สาธารณะใน compose | 🟡 | office `db/rd.go:68`; cron `service/redis.go:40-48`; linebot `redisService.go:77,89`, `docker-compose.yml` | cache/lock/dedup ถูกแก้จากภายนอกได้ถ้า network ไม่ปิด | เปิด auth + ใช้ env |

### 🧰 Maintainability

| # | ความเสี่ยง | ระดับ | หลักฐาน | ผลกระทบ | ข้อเสนอ |
|---|---|---|---|---|---|
| M1 | JWT lib `dgrijalva/jwt-go v3.2.0` (deprecated, มี CVE) ใช้คู่กับ `golang-jwt/jwt` — office และ linebot | 🟡 | office `go.mod:11,16`, `auth.go:21`; linebot `go.mod:6` | ช่องโหว่ lib + สับสนสอง lib | migrate เป็น golang-jwt v5 |
| M2 | weak crypto `md5`/`sha1` ใน flow auth/key | 🟡 | office `callbackLogin.go:921,952`, `helper.go:1117,1410`, `credit.go:191` | hash ปลอมแปลงได้ใน security context | เปลี่ยน SHA-256/HMAC |
| M3 | fallback URL dev/ngrok/devtunnels ฝังตายตัว — prod อาจยิงไป dev เงียบ ๆ ถ้า env ไม่ครบ (`payment-3rd-dev...`, ngrok `bank.go:4567,4742`, `PROXY_APP_URL` devtunnels) | 🟠 | office `payment.go`, `bank.go:4567,4742`, `.env` | ข้อมูลเงินจริงไหลไป endpoint dev/บุคคลที่สาม | fail-fast เมื่อ env ว่าง |
| M4 | cron 4 handler copy-paste ซ้ำ ~90%; dead code หลายชุด (sendsms.lupin.host, abaplan.sunnygreen.online); office dead routes/comment เพียบ | 🟡 | cron `smsauto.go:59-527`, `smsService.go:250,278,321`; office `telesale.go:18-47` | แก้ bug ต้องแก้ 4 ที่ | refactor รวม |
| M5 | FE: `xlsx ^0.17.0` (มีช่องโหว่), node:19 EOL, `eslint` เป็น runtime dep, `AUTHSERVICE`+secret ตายอยู่ใน bundle | 🟡 | FE `package.json:69,40`, `Dockerfile:1`, `stores/auth.ts:25` | supply-chain + bundle leak | อัปเดต dep / ตัด dead code |
| M6 | config ขัดกันเอง: cron PORT 5050 vs compose 4044; office go.mod 1.18 vs Docker go 1.22; env typo `MONGODB_OFFICE_DB_NANE` | 🟡 | cron `.env:3` vs `docker-compose.yml:10`, `_config/db.go:28`; office `go.mod:3` vs Dockerfile | deploy ผิดพอร์ต/ผิด env เงียบ ๆ | จัดมาตรฐาน config |

### 👁 Observability

| # | ความเสี่ยง | ระดับ | หลักฐาน | ผลกระทบ | ข้อเสนอ |
|---|---|---|---|---|---|
| O1 | error ถูกกลืนทั่วกลุ่ม: office ทิ้งผล `axios.Post`; cron ignore err ด้วย `_ :=` + InsertLogSMS fail แค่ println; linebot agent call log แล้วเดินต่อ | 🟠 | office `payment.go:128-130,138-140`; cron `smsauto.go:42,132-134`; linebot `agentService.go:90`, `callback.go:31-32` | ความผิดพลาดเรื่องเงิน/SMS หายเงียบ ตรวจสอบย้อนหลังไม่ได้ | error logging กลาง + alert |
| O2 | **kill-switch ภายนอกคุมหน้า office ทั้งระบบ**: Firebase RTDB `topup-group` status==0 → redirect ทุกคนไป google.com (apiKey hardcode) — ไม่มี log ว่าใครสั่ง | 🟠 | FE `plugins/firebase.ts:28-34`, `mainLayout.vue:350-356` | ผู้คุม RTDB ปิด back-office ได้เงียบ ๆ | ย้าย flag เข้า backend ที่ audit ได้ |
| O3 | Sentry DSN hardcode + `tracePropagationTargets` ยังเป็น placeholder `yourserver.io`; office ไม่มี metric/trace ที่ครอบ queue publish | 🟡 | FE `main.ts:64,69`; office `helper/queue.go` (ไม่มี instrument) | trace ขาดช่วงตรงจุดเงินไหล | เก็บ DSN ใน env + instrument publisher |
| O4 | AuthHeader ไม่ return หลัง `AbortWithStatusJSON` → โค้ดเดินต่อ panic ที่ `bearToken[1]` — panic โผล่เป็น 500 ปนสัญญาณ auth fail | 🟡 | office `auth.go:54-60` | log ปนเปื้อน + ช่อง DoS เบา ๆ | เติม `return` |

### ✅ ตรวจแล้วผ่าน (ของที่ดูเสี่ยงแต่มีการป้องกันจริง)

- **`/api/WithdrawCallback` มี idempotency บางส่วน** — query ด้วย filter `is_success: {$ne: 1}` และเช็ค `IsSuccess==1` แล้วคืน "รายการนี้สำเร็จแล้ว" ทันที (ยืนยันจากโค้ดจริง `controller/withdraw.go:3210-3220`; card อ้าง 3211-3221 คลาดเคลื่อน 1 บรรทัด) — replay callback เดิมซ้ำไม่ทำเงินซ้ำ แต่**ไม่ช่วย**กรณีปลอม orderNo ที่ยังไม่สำเร็จ
- **route ส่วนใหญ่ (~กลุ่ม `api`) มี JWT auth + permission ครบ** — `middlewares.AuthRequiredDB` ครอบ CRUD เกือบทั้งหมด และจุดเงินสำคัญฝั่งแอดมินมี permission เฉพาะ: `withdraw-confirm/confirmauto` = `WITHDRAW_AUTO` (`route/withdraw.go:35-36`), `credit-return` = `TRANSECTION_MANAGECREDIT_RETURN` (`route/credit.go:19`), `CreateStatement*` = `TRANSECTION_INSERT` (`route/statement.go:25-28`)
- **FE จัดการ session หมดอายุถูกต้อง** — 401/4011 ลบ `auth_token` + redirect `/login` (`api-config.ts:44-57`)
- **go_deposit ฝั่ง consumer ใช้ manual ack + Qos=1** (`rabbitmq/worker.go:61-68`) — ข้อความเติมเครดิตที่ถึง broker แล้วไม่หายกลางทางง่าย ๆ (จุดอ่อนอยู่ฝั่ง publish ของ office — R2)
- **cron SMS มี dedup รายวันจริง** (Redis TTL 24h) — ใช้งานได้ในเคสปกติ แม้ไม่ atomic (D1)

### ❓ ข้อสงสัย (ยังไม่ยืนยัน)

- **prod ingress/base URL ของ office-v10x อยู่นอก repo** — โค้ดใช้ `location.origin + "/api"` (`base-url.ts:60`) การ map `/api` → office-api-v10:7777 อยู่ใน k8s manifest repo ที่ไม่ได้อยู่ในโฟลเดอร์นี้ → ยืนยันไม่ได้ว่า prod มี WAF/IP filter ช่วยปิด S1/S2 หรือไม่
- **`CallbackApiWithdraw` ของ withdraw-auto ชี้ไปไหน** — เป็น URL runtime จาก `database_config` ยิง `POST .../withdraw_statement` (`withdraw-auto/service/report/callback.go:45-58`) ซึ่ง**ไม่มี** route นี้ใน office-api-v10 — น่าจะชี้ API ฝั่งเว็บลูกค้า ([customer-wallet](customer-wallet.md)) แต่ยังไม่มีหลักฐานสองฝั่ง
- **agent API ของ go-linebot-system** (`database_config.api_url`, `api_fontend_url`) เป็นค่า runtime — ยังจับคู่กับ repo ใดในระบบไม่ได้ 🟡
- **`PROXY_APP_URL` devtunnels และ `LINE_CONNECT_API` ngrok** ถูกใช้ใน prod จริงหรือเป็นซากของ dev — ตัดสินจากโค้ดอย่างเดียวไม่ได้

---

## ฉ) Unknown ของกลุ่ม

1. **caller ภายนอกของ no-auth callbacks** — `/api/WithdrawCallback`, `/api/Payment/*`, `/api/auto/createQueueWithdraw` ถูกยิงจาก URL ที่เก็บใน DB runtime ของ 3rd-payment/que_payment/topupserie — ในเมื่อไม่มี auth ก็ **พิสูจน์ไม่ได้ว่ามีแค่พวกนั้นที่เรียก**; ต้องดู access log prod ประกอบ
2. **k8s manifests / ingress** — CI ทุก repo ยิง repository_dispatch ไป manifest repo ภายนอก (เช่น `URL_K8S_LINEBOT_MUAY`) — network policy, ingress path mapping, secret จริงใน prod อยู่นอกขอบเขตที่ตรวจได้
3. **`URLK8S` = `http://18.141.146.113:6000`** (install/uninstall helm withdraw — `bank.go:174,965,2045`, `payment.go:334,496` ส่วนใหญ่ comment) — IP สาธารณะของ control plane อะไรบางอย่าง ไม่รู้ยังมีชีวิตหรือไม่ และ office เรียกแบบไม่มี auth ที่มองเห็นในโค้ด
4. **`DOMAIN_CUSTOMER` ของ linebot = `http://192.168.1.101:6900/rpt`** (`.env:3`) — เครื่องภายในวง LAN ใคร ไม่ทราบ
5. **เจ้าของ Firebase RTDB kill-switch** (`topup-group-default-rtdb`) และ withdraw-monitor (`aba-group-new`) — สิทธิ์เขียนอยู่กับใครไม่ปรากฏใน repo
