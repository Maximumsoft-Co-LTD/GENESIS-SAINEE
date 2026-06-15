# กลุ่ม: payment-gateway

> วิเคราะห์: 2026-06-12 | commit: 40367af | [← กลับหน้าปก](README.md)

**สมาชิก:** `3rd-payment` (Go/Gin, port 8081) · `que_payment` (Go worker, ไม่มี HTTP) · `3rd-gateway` (Go/Gin, port 8000)

กลุ่มที่เกี่ยวข้อง: [office-core](office-core.md) · [bank-automation](bank-automation.md) · [customer-wallet](customer-wallet.md)

---

## (ก) บทบาทของกลุ่ม

**เงินจริงทุกบาทที่เข้า-ออกระบบผ่าน payment provider ภายนอกไหลผ่านกลุ่มนี้ตรง ๆ** — ทั้งฝั่งฝาก (payin: ลูกค้าโอน/สแกน QR ผ่าน provider → provider ยิง callback กลับมา → เครดิตเข้าระบบ) และฝั่งถอน (payout: ระบบสั่ง provider โอนเงินออกให้ลูกค้า) กลุ่มนี้คุยกับ payment provider ภายนอก **~66 ราย** (ANYPAY, COREPAY, SPEEDPAY, PEER2PAY, XPAY, UMPAY ฯลฯ) บวก TrueMoney Wallet ผ่าน `api.tmn.one`

| สมาชิก | บทบาท |
|---|---|
| **3rd-payment** | "หน้าด่าน" sync API — รับคำสั่ง payin/create-withdraw จาก service อื่น, รับ **callback จาก provider ทุกราย** (`/api/v2/callback-payin/:payment_code`, `/api/v2/callback-payout/...`), ถือ config provider ใน `service_qrpayment`, แล้วโยนงานหนัก (เครดิตยอด, callback office) เข้าคิว RabbitMQ ให้ que_payment. มี provider package ~66 ตัวใต้ `controller/{name}/` (3rd-payment/routes/main.go:54-69, service/rabbit/que-payment.go:31-36) |
| **que_payment** | worker async ล้วน (ไม่มี HTTP server เลย) — consume คิว `QUE_PAYMENT_{PAYMENT_NAME}` แยกตาม type: `DEPOSIT`/`WITHDRAW`/`REFUND`/`REPORT_*` (rabbitmqpub/amqp.go:310-341), เป็นคนปรับยอดกระเป๋า `bank_summary` (`$inc current_balance`), เรียก provider ฝั่งถอน, และยิง callback ผลลัพธ์ไป office. มี cronjob ซ้อน: poll คิวค้าง ทุก 20s, retry order ทุก 5m, recover bank_summary, retry callback (quepub/cornjob.go:38-66, cronjob/*) |
| **3rd-gateway** | connector เฉพาะทาง **TrueMoney (TMN)** — เก็บ credential TMN ของลูกค้า (encrypted) ใน `gateway_connect`, ให้บริการ `api/fund-transfer` (โอนเงินจริงจาก TrueMoney wallet → บัญชีปลายทาง) และ `api/get-transactions` แก่ withdraw-auto / running-bank / office; ตอน connect gateway จะสั่ง helm install worker บน K8S ผ่าน `dp.thezeus.tech` (3rd-gateway/routes/main.go:31-39, gateway.go:359) |

จุดสำคัญเชิงสถาปัตยกรรม: 3rd-payment กับ que_payment **ใช้ MongoDB database เดียวกัน** (`DB_DBNAME`) และอ่าน/เขียน collection ชุดเดียวกัน (`qr_payment`, `withdraw_statement`, `bank_summary`, `service_qrpayment`, `que_payment`) — เป็น shared-data coupling ไม่ใช่แค่ message coupling และ**โค้ด provider ถูก copy ซ้ำสองชุด** (ดูผลเทียบใน ข้อสังเกตท้าย (ค))

---

## (ข) แผนผังกลุ่ม

```mermaid
graph LR
    subgraph payment-gateway
        TP["3rd-payment<br/>:8081 Gin<br/>~66 provider pkg"]
        QP["que_payment<br/>worker (no HTTP)<br/>TYPE=QUE_RABBIT/QUE_CRONJOB"]
        TG["3rd-gateway<br/>:8000 Gin<br/>TMN connector"]
        DB[("MongoDB {DB_DBNAME}<br/>qr_payment · withdraw_statement<br/>bank_summary · service_qrpayment<br/>que_payment")]
        TGDB[("MongoDB<br/>gateway_config · gateway_connect<br/>customer_registry")]
    end

    subgraph callers["ผู้เรียกเข้า (กลุ่มอื่น)"]
        OFFICE["office-api-v10"]
        TOPUP["topupserie"]
        WDAUTO["withdraw-auto"]
        RUNBANK["running-bank"]
        SCHED["SCHEDULE_SERVICE_BANK"]
    end

    subgraph external["ภายนอก"]
        PROV["payment providers ~66 ราย<br/>(HostSmartPay จาก DB)"]
        TMN["api.tmn.one<br/>(TrueMoney)"]
        K8S["dp.thezeus.tech<br/>/api/initWithdraw"]
        TELE["Telegram Bot API"]
        F2P["scanslipapi.first2pay.io"]
    end

    TOPUP -->|"POST /api/v2/payin<br/>/v2/get-bankconfig /v2/get-bank-support"| TP
    OFFICE -->|"PAYMENT_API: /api/v2/get-bankconfig<br/>/api/v2/callback-payin/{}-PEER2PAY<br/>/api/v2/check_balance_p2p /api/v2/bot/*"| TP
    SCHED -.->|"❌ /api/v2/callback-status-current/withdraw<br/>(route ถูก comment — dead)"| TP
    PROV -->|"callback: /api/v2/callback-payin/:payment_code<br/>/api/v2/callback-payout/:payment_code"| TP
    TP -->|"HTTP: deposit/withdraw/balance<br/>(base = HostSmartPay)"| PROV
    TP -->|"publish QUE_PAYMENT_{PaymentName}<br/>หรือ QUE_MERCHANT_{MERCHANT_CODE}"| QP
    QP -->|"RPC reply (d.ReplyTo + CorrelationId)"| TP
    QP -->|"withdraw/balance call"| PROV
    TP <-->|"R/W"| DB
    QP <-->|"R/W"| DB
    TP -->|"callback: /api/WithdrawCallback<br/>/api/Payment/PaymentCreateStatement/{svc}<br/>/api/Xpay/CreateStatement/{svc}"| OFFICE
    QP -->|"callback: api/WithdrawCallback<br/>api/Payment/PaymentCreateStatement/{svc}<br/>api/Payment/CallBackWithdrawAmount/{svc}"| OFFICE
    TP -->|"sendMessage/sendPhoto"| TELE
    TP -->|"scan_slip_verify"| F2P
    WDAUTO -->|"POST /api/fund-transfer<br/>(URL_GATEWAY)"| TG
    RUNBANK -->|"POST /api/get-transactions"| TG
    OFFICE -->|"/api/get-list-gateway<br/>/api/get-connected-by-account<br/>/api/disconnect-default-geteway"| TG
    TG -->|"login/balance/transfer<br/>api.php + proxy.dev.php"| TMN
    TG -->|"helm install/uninstall<br/>(api-key hardcoded)"| K8S
    TG <-->|"R/W"| TGDB
```

### Sequence: deposit ผ่าน provider callback (เงินเข้า)

```mermaid
sequenceDiagram
    autonumber
    participant TS as topupserie
    participant TP as 3rd-payment :8081
    participant PV as provider (เช่น COREPAY)
    participant MQ as RabbitMQ
    participant QP as que_payment
    participant DB as MongoDB {DB_DBNAME}
    participant OF as office-api-v10

    TS->>TP: POST /api/v2/payin (routes/main.go:54)
    TP->>DB: insert qr_payment
    TP->>PV: create order (base = serviceConfig.HostSmartPay จาก service_qrpayment)
    PV-->>TP: QR / payment URL
    TP-->>TS: payment detail URL
    Note over PV: ลูกค้าจ่ายเงินที่ provider
    PV->>TP: POST /api/v2/callback-payin/:payment_code (routes/main.go:56)<br/>⚠️ ไม่ verify signature — IPWhitelist เท่านั้น (ปิด default)
    TP->>DB: update qr_payment (callback.go:214)
    TP->>MQ: publish QUE_PAYMENT_{PaymentName} type=DEPOSIT<br/>(service/rabbit/que-payment.go:36, callback.go:15089)
    MQ->>QP: consume QUE_PAYMENT_{PAYMENT_NAME} (amqp.go:62, prefetch=1, manual ack)
    QP->>QP: QueTypeSwitchV2 → DepositAgentV2 (amqp.go:312)
    QP->>DB: $inc bank_summary.current_balance + log logs_inc_payment<br/>(paymentV2.go:161-191)
    QP->>OF: POST {OfficeAPI}/api/Payment/PaymentCreateStatement/{service}<br/>(services/callback/main.go:91) — เครดิตเข้าบัญชีสมาชิก
    QP-->>MQ: RPC reply → d.ReplyTo + CorrelationId (amqp.go:208)
    MQ-->>TP: reply (que-payment.go:75-105)
```

### Sequence: withdraw ผ่าน que_payment (เงินออก)

```mermaid
sequenceDiagram
    autonumber
    participant OF as office-api-v10
    participant TP as 3rd-payment :8081
    participant MQ as RabbitMQ
    participant QP as que_payment
    participant DB as MongoDB {DB_DBNAME}
    participant PV as provider

    OF->>TP: POST /api/v2/create-withdraw (routes/main.go:61)
    TP->>DB: insert withdraw_statement + que_payment (repository/main.go:32)
    TP->>MQ: publish QUE_PAYMENT_{PaymentName} type=WITHDRAW
    MQ->>QP: consume → WithdrawAndPayOutV2 (amqp.go:325-328)
    QP->>DB: หักยอด bank_summary (UpdateSummaryByServiceV2, paymentV2.go:161)
    QP->>PV: สั่งโอน เช่น corepay /trade/pay, speedpay /api/defray/V2<br/>(corepay/main.go:13, speedpay/main.go:61)
    alt provider error
        QP->>DB: refund ยอดคืน (withdraw.go:278-304)<br/>🔴 ถ้า refund fail = เงินหายเงียบ
    end
    PV->>TP: POST /api/v2/callback-payout/:payment_code (routes/main.go:65)
    TP->>OF: POST {urlCallback}/api/WithdrawCallback (service/callback/main.go:27)
    QP->>OF: POST {OfficeAPI}/api/WithdrawCallback (services/callback/main.go:41-47)
```

---

## (ค) ตาราง edge

| # | from | to | ชนิด | path / queue / collection จริง | หลักฐานสองฝั่ง (file:line) | conf |
|---|---|---|---|---|---|---|
| 1 | 3rd-payment | que_payment | queue | `QUE_PAYMENT_{PaymentName}` หรือ `QUE_MERCHANT_{MERCHANT_CODE}` (default exchange, payload `{id,service,type,...}`) | pub: 3rd-payment/service/rabbit/que-payment.go:31-36,140-149 ↔ sub: que_payment/rabbitmqpub/amqp.go:62-150 | 🟢 |
| 2 | que_payment | 3rd-payment | queue (RPC reply) | `d.ReplyTo` + CorrelationId | pub: que_payment/rabbitmqpub/amqp.go:208 ↔ sub: 3rd-payment/service/rabbit/que-payment.go:75-105 | 🟢 |
| 3 | 3rd-payment | que_payment | shared DB | MongoDB `{DB_DBNAME}`: `qr_payment`, `service_qrpayment`, `withdraw_statement`, `bank_summary`, `que_payment` R/W ทั้งสองฝั่ง | 3rd-payment/controller/callback.go:214, repository/main.go:32,162 ↔ que_payment/repository/payment.go:32-232, controllers/paymentV2.go:45-211 | 🟢 |
| 4 | 3rd-payment | office-api-v10 | HTTP callback (no-auth) | `/api/WithdrawCallback`, `/api/Payment/PaymentCreateStatement/{svc}`, `/api/Payment/CallBackWithdrawAmount/{svc}`, `/api/Xpay/CreateStatement/{svc}`, `/api/{svc}/payment-update-balance` (URL จาก DB config) | 3rd-payment/service/callback/main.go:27,67,103,147,189 ↔ office ฝั่งรับ ดู [office-core](office-core.md) | 🟢 ข้ามกลุ่ม |
| 5 | que_payment | office-api-v10 | HTTP callback (no-auth) | `api/WithdrawCallback`, `api/Payment/PaymentCreateStatement/{service}`, `api/Payment/CallBackWithdrawAmount/{service}` (URL = `OfficeAPI` ใน statement) | que_payment/services/callback/main.go:41-47,91,130,169 ↔ [office-core](office-core.md) | 🟢 ข้ามกลุ่ม |
| 6 | office-api-v10 | 3rd-payment | HTTP | env `PAYMENT_API`: `/api/v2/get-bankconfig`, `/api/v2/callback-payin/{}-PEER2PAY`, `/api/v2/check_balance_p2p`, `/api/v2/bot/*` | ฝั่งรับ: 3rd-payment/routes/main.go:56,158,166 ↔ ฝั่งเรียก ดู [office-core](office-core.md) | 🟢 ข้ามกลุ่ม |
| 7 | topupserie | 3rd-payment | HTTP | env `API_QR_ALL`: `/v2/get-bankconfig`, `/v2/payin`, `/v2/get-bank-support` | ฝั่งรับ: 3rd-payment/routes/main.go:54 ↔ [customer-wallet](customer-wallet.md) | 🟢 ข้ามกลุ่ม |
| 8 | office-api-v10 | 3rd-gateway | HTTP | `/api/get-list-gateway`, `/api/get-connected-by-account`, `/api/disconnect-default-geteway` (AuthClientMiddleware api_key) | ฝั่งรับ: 3rd-gateway/routes/main.go:33,34,36 ↔ [office-core](office-core.md) | 🟢 ข้ามกลุ่ม |
| 9 | running-bank | 3rd-gateway | HTTP | `POST /api/get-transactions` (ดึง statement TMN วันนี้) | ฝั่งรับ: 3rd-gateway/routes/main.go:39 → gateway.go:467 ↔ [bank-automation](bank-automation.md) | 🟢 ข้ามกลุ่ม |
| 10 | withdraw-auto | 3rd-gateway | HTTP | env `URL_GATEWAY` → `POST /api/fund-transfer` | ฝั่งเรียก: withdraw-auto/controller/thirdgateway/main.go:26 (URL_GATEWAY), :165 (`URL3rdGateway + "/api/fund-transfer"`) ↔ ฝั่งรับ: 3rd-gateway/routes/main.go:38 → gateway.go:408 | 🟢 **VERIFY แล้ว — ยืนยัน** |
| 11 | SCHEDULE_SERVICE_BANK | 3rd-payment | HTTP | env `PAYMENT_API` → `/api/v2/callback-status-current/withdraw` | ฝั่งรับ: **route ถูก comment ทิ้ง** — 3rd-payment/routes/main.go:85 `// rv2.POST("/callback-status-current/withdraw", ...)` | 🔴 **VERIFY แล้ว — edge ตาย** (ผู้เรียกจะได้ 404) |
| 12 | providers ~66 ราย | 3rd-payment | HTTP callback เข้า | `/api/v2/callback-payin/:payment_code`, `/api/v2/callback-payout/:payment_code`(+variant umpay/autopeer/hengpay), `/api/v2/callback-one-link/...`, `/peer2pay/v3/callback-*`, `/api/xpay/private/callback-payin/:payment_code` | 3rd-payment/routes/main.go:56-69,176-178,195 (ฝั่ง provider ตรวจไม่ได้ — ภายนอก) | 🟢 |
| 13 | 3rd-payment | providers | HTTP ออก | base = `serviceConfig.HostSmartPay` (runtime จาก collection `service_qrpayment`) + hardcoded บางราย (payonex/smilepayz/superichpay) | controller/anypay/main.go:42, controller/call-by-payment.go:71, controller/askmepay | 🟢 |
| 14 | que_payment | providers | HTTP ออก | base = `serviceConfig.HostSmartPay` เช่น corepay `/trade/pay`, `/merchant/balance`, speedpay `/api/defray/V2` | que_payment/services/corepay/main.go:13, speedpay/main.go:61 | 🟢 |
| 15 | 3rd-gateway | api.tmn.one (TrueMoney) | HTTP ออก | `https://api.tmn.one/api.php` + `https://api.tmn.one/proxy.dev.php/tmn-mobile-gateway/` (login pin, balance, transactions, p2p/account transfer) | tmn-one/main.go:24-25, client.go:43,140 | 🟢 ภายนอก |
| 16 | 3rd-gateway | dp.thezeus.tech (K8S) | HTTP ออก | `POST https://dp.thezeus.tech/api/initWithdraw` (helm install/uninstall worker, header `api-key` hardcoded) | service/helminstall/install-helm.go:13-26 ← เรียกจาก gateway.go:359 | 🟢 ภายนอก |
| 17 | 3rd-payment | Telegram / first2pay / Google Translate / URL_VERIFYSLIP | HTTP ออก | `api.telegram.org/bot{token}/...`, `scanslipapi.first2pay.io/v1/scan_slip_verify`, `translate.googleapis.com` | app.go:206, controller/first2pay, controller/main.go:2295, service/verifySlip/verifySlip.go:15-27 | 🟢 ภายนอก |
| 18 | que_payment | Telegram / Sentry / OTel | alert/observability | `TELEGRAM_ERROR_BOT_TOKEN`, `SENTRY_DSN`, `OTEL_URL` | observability/telegram/notifier.go:47-72, _cmd/main.go:33-35, app.go:30 | 🟢 ภายนอก |

### ข้อสังเกต: provider code ซ้ำสองชุด (เทียบจริงแล้ว)

เทียบ `corepay` และ `speedpay` ระหว่าง `3rd-payment/controller/{name}/` กับ `que_payment/services/{name}/` — **เป็น copy-paste ชุดเดียวกันที่แตก fork แล้ว drift**:

- โครงสร้างไฟล์เหมือนกันเป๊ะ (`func.go` / `main.go` / `model.go`) และชื่อฟังก์ชันชุดเดียวกัน: `Md5Sign`, `CreateCorePaySign`, `CreateCorePayData`, `EncryptAESCorepay`, `PKCS7Pad/Unpad`, `MakeRequestCorepay`, `BankDataCorepay`
- ฝั่ง que_payment เป็น subset เฉพาะ Withdraw/GetBalance (corepay/main.go: 84 บรรทัด) ส่วน 3rd-payment มีครบ Deposit/Withdraw/Balance/Enquiry/TradeUser (452 บรรทัด)
- **drift จริงที่พบ:** speedpay bank map ต่างกันแล้ว — 3rd-payment ใช้ `T_BAY`/`T_TTB` (3rd-payment/controller/speedpay/func.go:24,45) แต่ que_payment แก้เป็น `T_KRUNGSRI`/`T_TMB` (que_payment/services/speedpay/func.go:25,47) → ถ้า provider เปลี่ยน code แล้วแก้ฝั่งเดียว ฝากกับถอนจะ map คนละธนาคาร

---

## (ง) Key flows

สัญลักษณ์: `→` HTTP call · `⟿` async (queue) · `↩` callback/reply · `💾` เขียน DB

### Flow 1: Payin + provider callback (เครดิตเงินเข้า)

1. topupserie `→` 3rd-payment `POST /api/v2/payin` (routes/main.go:54) `💾` insert `qr_payment`
2. 3rd-payment `→` provider (base = `HostSmartPay` จาก `service_qrpayment`) สร้าง order/QR `↩` payment URL กลับให้ topupserie
3. ลูกค้าจ่ายเงินฝั่ง provider → provider `→` 3rd-payment `POST /api/v2/callback-payin/:payment_code` (routes/main.go:56) — **ไม่มี signature check, IPWhitelist ปิด default** (callback.go:112-310, ipwhitelist.go:22-25)
4. 3rd-payment `💾` update `qr_payment` (callback.go:214) แล้ว `⟿` publish `QUE_PAYMENT_{PaymentName}` type=`DEPOSIT` (que-payment.go:36, callback.go:15089) — กันซ้ำด้วย `IsSuccess==1` เท่านั้น (callback.go:15068)
5. que_payment consume (amqp.go:62, prefetch=1) → `DepositAgentV2` (amqp.go:312) `💾` `$inc bank_summary.current_balance` + audit `logs_inc_payment` (paymentV2.go:161-191)
6. que_payment `↩` callback office `POST {OfficeAPI}/api/Payment/PaymentCreateStatement/{service}` (services/callback/main.go:91) — **office เครดิตเข้าบัญชีสมาชิกจากจุดนี้** (no-auth, retry 3 ครั้ง main.go:24-30)
7. que_payment `↩` RPC reply เข้า `d.ReplyTo` (amqp.go:208) ← 3rd-payment รอด้วย CorrelationId (que-payment.go:75-105)
8. (ถ้ามี SettingForward) 3rd-payment `↩` forward callback ไป URL ลูกค้า (callback.go:16058-16064)

### Flow 2: Withdraw/payout ผ่านคิว

1. office `→` 3rd-payment `POST /api/v2/create-withdraw` → `confirm-withdraw` (routes/main.go:61-62) `💾` `withdraw_statement` + `que_payment` (repository/main.go:32)
2. 3rd-payment `⟿` `QUE_PAYMENT_{PaymentName}` type=`WITHDRAW` — ถ้า message หลุด มี cronjob ฝั่ง que_payment (TYPE=QUE_CRONJOB) poll `que_payment` status=0 ทุก 20s ยิงคิวซ้ำ (quepub/cornjob.go:38-66)
3. que_payment → `WithdrawAndPayOutV2` (amqp.go:325-328): `💾` หักยอด `bank_summary` (lock ด้วย flag `is_active` compare-and-set — ไม่ใช่ transaction, paymentV2.go:211-214)
4. que_payment `→` provider สั่งโอน (corepay `/trade/pay` main.go:13, speedpay `/api/defray/V2` main.go:61)
5. ถ้า provider ตอบ error → refund ยอดคืน — **ถ้า refund เอง fail เงินถูกหักทิ้งไว้** (withdraw.go:278-304) 🔴
6. provider โอนเสร็จ `↩` 3rd-payment `POST /api/v2/callback-payout/:payment_code` (routes/main.go:65) `💾` update `withdraw_statement`
7. ทั้งสองฝั่ง `↩` office: 3rd-payment ผ่าน `service/callback/main.go:27` (`/api/WithdrawCallback`), que_payment ผ่าน `services/callback/main.go:41-47` — มี cronjob retry callback ค้าง 6h (automation_retry_callback_status.go:34-144)
8. order ค้าง status=16 เกิน 5h → `ServRetryOrder` ยกเลิก + callback office (cronjob/retry-order.go:32-61)

### Flow 3: TMN fund-transfer (ถอนผ่าน TrueMoney — เงินจริงออกทันที)

1. withdraw-auto consume คิว `WITHDRAW_{BankNumber}` → routing เข้า `thirdgateway.StartServ()` เมื่อ `GatewayConfig.IsWithdraw` (withdraw-auto/rabbitmqpub/worker.go)
2. withdraw-auto `→` 3rd-gateway `POST {URL_GATEWAY}/api/fund-transfer` (thirdgateway/main.go:26,165) — auth = `AuthClientMiddleware` เทียบ `api_key` ตรง ๆ ใน DB (customer-client.go:26)
3. 3rd-gateway หา `gateway_connect` ของ customer (typeAcc=withdraw, gateway.go:428) → ถอดรหัส credentials_token (AES-256-GCM key=`AES_KEY_FIX32`) → init TMN
4. ถ้า `to_account_bank_code=="TRUEWALLET"` → `FundTransfer` (p2p) — **logic กลับด้าน: คืน error เมื่อ success** (tmn-one/main.go:172-174) 🔴; ไม่งั้น `FundTransferAccount` (main.go:201 — เงื่อนไขถูก) `→` `api.tmn.one` โอนจริง
5. `💾` update `credentials_token` + `total_balance` กลับ `gateway_connect` (gateway.go:461-463)
6. withdraw-auto สร้างสลิป + update สถานะ withdraw_statement ต่อ (ดู [bank-automation](bank-automation.md))
   - หมายเหตุ: เส้นทาง `controller/tmn/tmnGateway` ใน withdraw-auto เรียก `{URL_GATEWAY}/fund-transfer` **ไม่มี prefix `/api`** (tmnGateway/func.go:37) — ใช้ได้ก็ต่อเมื่อ env ใส่ `/api` มาแล้ว ไม่งั้นเป็น path ตาย (ดู "ข้อสงสัย")

---

## (จ) Risk Register

### หมวด 1 — การยืนยันตัวตน / ช่องทางเข้า (Authentication & Exposure)

| # | ความเสี่ยง | ระดับ | file:line | ผลกระทบ | ข้อเสนอ |
|---|---|---|---|---|---|
| 1.1 | callback จาก provider **ไม่ verify signature/HMAC ใด ๆ** — อ่าน body แล้ว update `qr_payment` + ส่งคิวเครดิตทันที | 🔴 | 3rd-payment/controller/callback.go:112-310 | ใครยิง callback ปลอมได้ = สั่งเครดิตเงินปลอมเข้าระบบ | บังคับ verify signature ต่อ provider (เกือบทุกเจ้ามี sign ใน spec) ก่อนแตะ DB |
| 1.2 | IP whitelist **ปิดโดย default** — `CHECK_IPWHITELIST != "TRUE"` → ผ่านทุก IP; เป็นเกราะเดียวของข้อ 1.1 | 🔴 | 3rd-payment/middleware/ipwhitelist.go:22-25 | callback-payin/payout เปิด public ถ้าลืมตั้ง env | invert default (fail-closed), alert เมื่อ middleware ถูก bypass |
| 1.3 | `/bank-gateway/call-by-req` = **SSRF เต็มรูปแบบ no auth** — รับ `domain_call`+method+header+body จาก client แล้วยิงออกตรง | 🔴 | 3rd-payment/routes/bank-transfer-gateway.go:55-77 | ใช้ server เป็น proxy ยิง internal network / metadata endpoint | ลบ หรือใส่ auth + allowlist domain |
| 1.4 | `/call-pm/:paymentCode/*byPath` generic proxy ไป HostSmartPay ตาม path ที่ client กำหนด, no auth | 🟠 | 3rd-payment/controller/call-by-payment.go:49-80, app.go:144 | เรียก API provider ใด ๆ ด้วย credential ของระบบ (เช่น สั่งถอน) | จำกัด path allowlist + auth |
| 1.5 | endpoint การเงินส่วนใหญ่ของ 3rd-payment **ไม่มี middleware auth เลย** (`/api/v2/create-withdraw`, `/confirm-withdraw`, `/adjust-balance`, `/confirm-transaction-payin` ฯลฯ) | 🔴 | 3rd-payment/routes/main.go:61-64,70,74 | ใครเข้าถึง network ได้ = สั่งถอน/adjust ยอดได้ | service-to-service auth (mTLS/API key) ทั้ง group |
| 1.6 | `api/register` ของ 3rd-gateway เป็น public ไม่มี rate limit; api_key เทียบ plain equality ใน DB query | 🟡 | 3rd-gateway/routes/main.go:31, middleware/customer-client.go:26 | สมัคร client เถื่อน / credential stuffing | rate limit + hash api_key |

### หมวด 2 — ความถูกต้องทางการเงิน (Money Correctness)

| # | ความเสี่ยง | ระดับ | file:line | ผลกระทบ | ข้อเสนอ |
|---|---|---|---|---|---|
| 2.1 | **FundTransfer (TMN p2p) logic กลับด้าน** — `if strings.Contains(resp.Status, "success") { return ..., fmt.Errorf("error confirm wallet") }` คืน error เมื่อโอน*สำเร็จ* (เทียบ FundTransferAccount ที่ใช้ `!Contains`) | 🔴 | 3rd-gateway/tmn-one/main.go:172-174 (เทียบ :201) | **เงินจริง**: โอน TrueMoney p2p สำเร็จแต่ระบบบันทึก fail → จ่ายซ้ำ/ยอดเพี้ยน | แก้เงื่อนไขทันที + เพิ่ม test กับ response จริง |
| 2.2 | **refund fail หลังหักเงิน** — withdraw หักยอดก่อน แล้วถ้า provider error จะ refund; ถ้า `UpdateSummaryByServiceV2` refund fail จะ return error เฉย ๆ โดยเงินถูกหักไปแล้ว | 🔴 | que_payment/controllers/withdraw.go:278-304 | ยอด `bank_summary` หายเงียบ ไม่มี reconciliation | ทำ compensation queue + alert เมื่อ refund fail |
| 2.3 | **ไม่มี idempotency จริง** — กันซ้ำด้วย status check (`IsSuccess==1`) ไม่มี unique index/transaction; callback ซ้ำที่มาพร้อมกันก่อน status เปลี่ยน = credit ซ้ำ (race: GetConfig→Switch→CallRabbit) | 🔴 | 3rd-payment/controller/callback.go:15068; que_payment/controllers/withdraw.go:367, deposit.go:117 | duplicate callback (ซึ่ง provider ส่งซ้ำเป็นปกติ + callback client เอง retry 3 ครั้ง services/callback/main.go:24-30) → เครดิตคู่ | unique index บน order_no+status transition, findOneAndUpdate atomic |
| 2.4 | balance update ไม่ atomic — GetOneDocument แล้วค่อย `$inc` แยก step, lock ด้วย flag `is_active` (compare MatchedCount) ไม่ใช่ Mongo transaction; ฝั่ง 3rd-payment พึ่ง thorlock (Redis) — Redis ล่ม = ไม่มี lock | 🟠 | que_payment/controllers/paymentV2.go:161-214; 3rd-payment/app.go:80-84 | race window ปรับยอดผิด | ใช้ findOneAndUpdate เงื่อนไขครบใน query เดียว หรือ Mongo txn |
| 2.5 | publish RabbitMQ fail แล้ว swallow — `CallRabbitDepositAndWithdraw` error → `Trac.ChildSuccess()` + return 501; deposit รับเงินแล้วแต่คิวไม่ถูกส่ง | 🟠 | 3rd-payment/controller/callback.go:15095 | เงินเข้า provider แล้วแต่ไม่เครดิต (พึ่ง cron poll กู้) | mark `que_payment` status ให้ cron กู้ได้เสมอ + alert |
| 2.6 | callback office ทั้งสองฝั่ง fire-and-forget goroutine ไม่เช็ค error | 🟠 | que_payment/quepub/cornjob.go:243,349; withdraw.go:333 | ผลถอนไม่ถึง office เงียบ ๆ (มี cron retry 6h ช่วยบางส่วน) | เก็บ result + dead-letter |

### หมวด 3 — Secrets / ข้อมูลรั่วใน repo

| # | ความเสี่ยง | ระดับ | file:line | ผลกระทบ | ข้อเสนอ |
|---|---|---|---|---|---|
| 3.1 | MongoDB **root credential** committed: `mongodb+srv://root:Zxcvasdf789@test02.9oh7e.mongodb.net/` | 🔴 | 3rd-payment/docker-compose.yml:11 | เข้าถึง DB ที่ถือทุก statement การเงิน | rotate ทันที + ลบจาก history |
| 3.2 | MongoDB credential committed อีกชุด: `mongodb://useradmin:Aa112233@3.1.8.89:27017/abaoffice_db` (+ `BANK_NUMBER=2420260295`) | 🔴 | que_payment/docker-compose.yml:12,14 | เข้าถึง DB office จริง | rotate + secret manager |
| 3.3 | **K8S API key hardcoded ในโค้ด**: `mXjtZ7q9D0zhG6NN06832mqLCr0KlM` + URL `dp.thezeus.tech/api/initWithdraw` | 🔴 | 3rd-gateway/service/helminstall/install-helm.go:13-14 | สั่ง install/uninstall workload บน cluster ได้ | rotate + ย้ายเข้า env/secret |
| 3.4 | `azpay.json` (Postman collection) committed มี merchant signature hash + HMAC pattern | 🟠 | 3rd-payment/azpay.json:95,129,203 | reverse sign scheme ของ merchant | ลบไฟล์ + rotate merchant secret |
| 3.5 | `config.yaml` ถูก track (gitignore เขียน `./config.yaml` — pattern ผิด); RabbitMQ guest fallback hardcode ทั้งสอง repo | 🟡 | 3rd-payment/.gitignore:2, config.yaml:11; que_payment/rabbitmqpub/conn.go:61 | config/topology หลุด; dev broker ไม่มี auth จริง | แก้ gitignore, ตัด fallback guest |

### หมวด 4 — Cryptography

| # | ความเสี่ยง | ระดับ | file:line | ผลกระทบ | ข้อเสนอ |
|---|---|---|---|---|---|
| 4.1 | TMN wallet token: **AES-CBC ไม่มี MAC + `unpad` ไม่ validate padding → padding oracle** | 🟠 | 3rd-gateway/tmn-one/cryption.go:107-173, :175-179 | ถอดรหัส/forge wallet access token ได้ในเงื่อนไข oracle | เปลี่ยนเป็น AES-GCM (มี EncryptAesGcm อยู่แล้วในโปรเจกต์) |
| 4.2 | `AES_KEY_FIX32` **key เดียวทั้งระบบ** encrypt credentials_token (รวม PIN TrueMoney) ของทุก customer | 🟠 | 3rd-gateway/config/environment.go:20; gateway.go:297; tmn-one/main.go:257-259 | key รั่วครั้งเดียว = ถอด TMN credential ทุกคน | per-record key (envelope ผ่าน KMS — มี EncAeadKMS แล้ว aead.go:80) |
| 4.3 | **DecryptMid เป็น dead code** — `ctx.Next(); return` บนสุดของฟังก์ชัน ทำให้ flow encrypted_random_key ทั้งเส้นไม่ทำงาน | 🟠 | 3rd-gateway/middleware/decrypted.go:21-22 | เข้าใจผิดว่ามี end-to-end encryption ทั้งที่ไม่มี | ลบ หรือเปิดใช้จริง |
| 4.4 | corepay sign = **MD5 สองชั้น** + AES-CBC no MAC (ซ้ำทั้งสอง repo); SHA1 hash statement; PBKDF2 iteration=5 + salt=`"0"` | 🟠 | que_payment/services/corepay/func.go:91-113; 3rd-payment/helper/hash.go:10, controller/banktransfer/encrypt-decrypt.go:141,162 | sign ปลอมง่าย / brute-force เร็ว | ตาม spec provider ถ้าแก้ไม่ได้ อย่างน้อยแก้ฝั่ง internal (PBKDF2/SHA1) |
| 4.5 | Tink keyset โหลดแบบ `insecurecleartextkeyset` — private key plaintext ใน env `PRIV_KEYSET` | 🟠 | 3rd-gateway/service/encryption/aead.go:28,53, hybrid.go:29,53; config/environment.go:21 | key รั่วผ่าน env dump | ใช้ KMS-wrapped keyset ให้สม่ำเสมอ (แบบ EncAeadKMS) |

### หมวด 5 — ความเสถียร / Maintainability

| # | ความเสี่ยง | ระดับ | file:line | ผลกระทบ | ข้อเสนอ |
|---|---|---|---|---|---|
| 5.1 | **callback.go ไฟล์เดียว 16,182 บรรทัด** คุม callback ทุก provider | 🟠 | 3rd-payment/controller/callback.go | แก้ provider หนึ่งกระทบทุกตัว, review ไม่ไหว | แตกไฟล์ตาม provider + interface กลาง |
| 5.2 | **provider code duplicate 2 ชุดและ drift แล้ว** — speedpay bank map ต่างกัน (`T_BAY` vs `T_KRUNGSRI`, `T_TTB` vs `T_TMB`) | 🟠 | 3rd-payment/controller/speedpay/func.go:24,45 ↔ que_payment/services/speedpay/func.go:25,47 | แก้ฝั่งเดียว → ฝาก/ถอน map คนละธนาคาร | แยก provider SDK เป็น shared module เดียว |
| 5.3 | HTTP client ไม่มี timeout ทั้งสาม repo (`&http.Client{}` เปล่า) | 🟡 | 3rd-payment/helper/callApi.go:18, routes/bank-transfer-gateway.go:73; 3rd-gateway/tmn-one/main.go:135, client.go:59, install-helm.go:27 | provider ค้าง = goroutine/worker ค้าง | ตั้ง Timeout + context ทุก client |
| 5.4 | edge ตาย: route `/api/v2/callback-status-current/withdraw` ถูก comment แต่ SCHEDULE_SERVICE_BANK ยังชี้มา | 🟡 | 3rd-payment/routes/main.go:85 | scheduler ยิง 404 เงียบ ๆ — งาน update สถานะ withdraw ไม่เกิด | เปิด route คืนหรือถอดฝั่งเรียก |
| 5.5 | dead code V1 คู่ V2 ใน que_payment (`StartAMQP`/`QueTypeSwitch`/`QueRunning`) + `failOnError` ใช้ `log.Fatalf` ใน path V1; dead code ก้อนใหญ่ใน 3rd-payment routes | 🟡 | que_payment/rabbitmqpub/amqp.go:225-308, :27-31; 3rd-payment/routes/main.go:18-25,188-214 | แก้ไม่ครบสองที่ / process ตายทั้งตัว | ลบ V1 เมื่อยืนยันไม่มี deploy ใช้ |
| 5.6 | 3rd-gateway **ปิด OTel/tracing ทั้งหมด** ใน service ที่โอนเงินจริง; helm install error แค่ Println ไม่ rollback | 🟡 | 3rd-gateway/app.go:43-48,54; gateway.go:359-362 | เงินออกแต่ไม่มี trace; connect ค้างสถานะครึ่งทาง | เปิด observability + rollback connect |
| 5.7 | Dockerfile 3rd-payment บังคับ `--platform=arm64` ทั้ง build/run | 🟡 | 3rd-payment/Dockerfile:1,7 | deploy บน amd64 ต้อง emulate (ช้า/พัง) | ใช้ TARGETPLATFORM |

### ✅ ตรวจแล้วผ่าน (ของที่ทำถูก)

| รายการ | หลักฐาน |
|---|---|
| 3rd-gateway `EncryptAesGcm` ใช้ AES-256-GCM ถูกต้อง — AEAD, nonce random, ตรวจ key len=32 | 3rd-gateway/service/encryption/aes.go:13-43 |
| `EncAeadKMS` ใช้ GCP KMS envelope encryption ถูกแบบ (ต่างจาก keyset cleartext ข้อ 4.5) | 3rd-gateway/service/encryption/aead.go:80-132 |
| que_payment consumer ใช้ **manual ack (autoAck=false) + Qos prefetch=1** — message ไม่หายเมื่อ worker ตายกลางคัน | que_payment/rabbitmqpub/amqp.go:84,150 |
| `report_payment` upsert ใช้ **MongoDB transaction + bulkWrite** จริง (จุดเดียวในกลุ่มที่ใช้ txn) | que_payment/controllers/paymentV2.go:587-687, report.go:32-39 |
| TMN request encryption ใช้ IV random ต่อ request (แม้ CBC) | 3rd-gateway/tmn-one/cryption.go:24-30 |
| cron retry callback มี Redis lock ราย order กันยิงซ้ำพร้อมกัน | que_payment/cronjob/automation_retry_callback_status.go:28-29,77 |
| audit trail การ inc ยอดแยก collection `logs_inc_payment` + `logs_que_payment` timeline | que_payment/controllers/deposit.go:381,423; paymentV2.go:228-253 |

### ❓ ข้อสงสัย (ยังไม่ยืนยัน)

- **tmnGateway path ไม่ตรง convention** — withdraw-auto/controller/tmn/tmnGateway/func.go:37 เรียก `{URL_GATEWAY}/fund-transfer` (ไม่มี `/api`) ขณะ thirdgateway/main.go:165 เรียก `{URL_GATEWAY}/api/fund-transfer`; สองเส้นนี้ใช้ env ตัวเดียวกัน (`URL_GATEWAY` — main.go:26 vs tmnGateway/main.go:34) จึงถูกพร้อมกันไม่ได้ — เส้นใดเส้นหนึ่งตายหรือ env ของ deployment ต่างกัน ต้องดูค่า env จริงใน manifest
- **que_payment V1 queue `QUE_PAYMENT_{SYSTEM_CODE}`** (amqp.go:233) ยังมี deployment ที่รันโหมดนี้อยู่หรือไม่ — ไม่พบ publisher ฝั่ง 3rd-payment ที่ใช้ SYSTEM_CODE
- **`/api/v2/dashboard/*` auth ด้วย ALLOW_IP/AUTH_KEY ใน handler** (controller/dashboard/main.go:22,130) — ความแข็งแรงของ AUTH_KEY ยังไม่ได้ตรวจ
- **API_REPORT** ถูก comment ใน callback.go:16033 — report service เคยเป็น dependency แต่สถานะปัจจุบันไม่ทราบ

---

## (ฉ) Unknown / ต้องตามต่อ

| ประเด็น | สิ่งที่รู้ | สิ่งที่ไม่รู้ |
|---|---|---|
| **ENCRYPTION_ENDPOINT** | 3rd-payment เรียกใช้ตอน encrypt payload ของ cloudpay/visapay (controller/cloudpay/func.go:117-118, controller/visapay/func.go:117-118) — ค่าเป็น runtime env | เป็น service ไหน (ไม่อยู่ใน 25 repo นี้?) ใครถือ key, ถ้าล่ม cloudpay/visapay จ่ายไม่ได้ทั้งคู่ |
| **HostSmartPay** | base URL ของ provider เกือบทุกราย **มาจาก DB runtime** (collection `service_qrpayment` field `HostSmartPay` — controller/anypay/main.go:42; que_payment/models/service_payment.go:5-59) ไม่ใช่ env | ค่า production จริงต่อ provider — ต้อง dump DB ถึงจะ map external dependency ครบ; ใครมีสิทธิแก้ field นี้ = redirect เงินได้ (ผูกกับ risk 1.5 เพราะ `/api/v2/update-config-payment` ไม่มี auth) |
| **BANK_AUTH service** | withdraw-auto มี `service/callAuthBank/` สำหรับ auth ธนาคาร (เกี่ยวพันกับ flow fund-transfer ของกลุ่มนี้ทางอ้อม) | เป็น service ภายนอกหรือ repo ใดใน 25 ตัว — ยังไม่พบฝั่ง serve; ดู [bank-automation](bank-automation.md) |
| **ผู้เรียก callback-status-current** | SCHEDULE_SERVICE_BANK ตั้ง `PAYMENT_API` ชี้มาที่ route ที่ถูก comment (3rd-payment/routes/main.go:85) | ถูกปิดโดยตั้งใจ (ย้าย logic ไปที่อื่น) หรือ regression — ถ้าตั้งใจ ฝั่ง scheduler ควรถอด job ทิ้ง |
| **dp.thezeus.tech** | 3rd-gateway สั่ง helm install "withdraw worker" ผ่าน API นี้เมื่อ customer connect TMN (install-helm.go:12-27) | workload ที่ถูก install คืออะไร (น่าจะเป็น withdraw-auto instance ราย bank?) — ต้องดู manifest repo |
