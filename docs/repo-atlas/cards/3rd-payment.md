# Card: 3rd-payment
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go + Gin + MongoDB (raw driver) + Redis (thorlock) + RabbitMQ (streadway/amqp) + OpenTelemetry (otelgo) — `go.mod` `module app`
- **module**: `app` — entrypoint import `import "app"` (`_cmd/main.go:4`)
- **entrypoint**: `_cmd/main.go` → `main()` เรียก `app.StartServ()` (`_cmd/main.go:24-30`); `StartCronjob()` ถูก comment ทิ้งไว้ (`_cmd/main.go:31`)
- **port**: `:8081` ทั้ง dev (`app.go:147 r.Run(":8081")`) และ production (`app.go:158 srv := &http.Server{Addr: ":8081"}`) — Dockerfile ไม่มี `EXPOSE`, docker-compose ไม่ map ports
- **Dockerfile**: build `golang:1.25-alpine3.22` → run `alpine:3.22`, `CGO_ENABLED=0 go build -o app _cmd/main.go`, `CMD ["./app"]`, copy `non_sans.ttf` (Dockerfile:1-13). platform=arm64 บังคับ
- **CI**: `.github/workflows/build.yaml` self-hosted runner → Kaniko/JFrog (`thezeustech.jfrog.io`), tag uat/dev ตาม branch (build.yaml:1-40)

## provides

### http
Base group `/api/v2` (rv2) นอกจากระบุไว้เป็นอื่น. middleware auth = `IPWhitelistMiddleware` (เปิดเฉพาะ `CHECK_IPWHITELIST=="TRUE"`, `middleware/ipwhitelist.go:22`) มิฉะนั้น `c.Next()` ผ่านหมด

| METHOD | path | auth middleware | evidence (file:line) |
|--------|------|-----------------|----------------------|
| POST | /api/v2/payin | ไม่มี | routes/main.go:54 |
| POST | /api/v2/callback-payin/:payment_code | IPWhitelist | routes/main.go:56 |
| POST | /api/v2/callback-payin/:payment_code/callback/qrStatus | IPWhitelist (peer2pay) | routes/main.go:57 |
| POST | /api/v2/callback-one-link/:payment_name/:payment_code | IPWhitelist | routes/main.go:58 |
| POST | /api/v2/callback-one-link/:payment_name/:payment_code/deposit | IPWhitelist | routes/main.go:59 |
| POST | /api/v2/callback-one-link/:payment_name/:payment_code/withdraw | IPWhitelist | routes/main.go:60 |
| POST | /api/v2/create-withdraw | ไม่มี | routes/main.go:61 |
| POST | /api/v2/confirm-withdraw | ไม่มี | routes/main.go:62 |
| POST | /api/v2/reconfirm-withdraw | ไม่มี | routes/main.go:63 |
| POST | /api/v2/refund-withdraw-payment | ไม่มี | routes/main.go:64 |
| POST | /api/v2/callback-payout/:payment_code | IPWhitelist | routes/main.go:65 |
| POST | /api/v2/callback-payout/autopeer/:payment_code | IPWhitelist | routes/main.go:66 |
| POST | /api/v2/callback-payout/umpay/:payment_code | IPWhitelist | routes/main.go:67 |
| POST | /api/v2/check-uuid/umpay/:payment_code/:uuid | ไม่มี | routes/main.go:68 |
| POST | /api/v2/callback-payout/:payment_code/withdraw | IPWhitelist (hengpay) | routes/main.go:69 |
| POST | /api/v2/adjust-balance | ไม่มี | routes/main.go:70 |
| POST | /api/v2/balance | ไม่มี (เช็คยอด) | routes/main.go:88 |
| GET | /api/v2/check/balance | ไม่มี | routes/main.go:89 |
| POST | /api/v2/withdraw/payment | ไม่มี | routes/main.go:104 |
| POST | /api/v2/cancel/payout | ไม่มี | routes/main.go:105 |
| POST | /api/v2/submit-withdraw | ไม่มี | routes/main.go:107 |
| POST | /api/v2/callback-transaction/:payment_code | ไม่มี | routes/main.go:106 |
| POST | /api/v2/virify/slip | ไม่มี (verify-slip first2pay) | routes/main.go:144 |
| POST | /api/v2/callback/verify_slip/:payment_code/:order_no | ไม่มี | routes/main.go:145 |
| GET | /api/v2/get-order/:order_no | ไม่มี | routes/main.go:30 |
| POST | /api/v2/create-order | ไม่มี | routes/main.go:31 |
| POST | /api/v2/new-order-payin | ไม่มี | routes/main.go:32 |
| POST | /api/v2/get-order-transaction | ไม่มี | routes/main.go:33 |
| POST | /api/v2/get-payment-list-by-name | ไม่มี | routes/main.go:36 |
| POST | /api/v2/create-config-payment | ไม่มี | routes/main.go:38 |
| POST | /api/v2/update-config-payment | ไม่มี | routes/main.go:39 |
| POST | /api/v2/create-indo-config-payment | ไม่มี | routes/main.go:40 |
| GET | /api/v2/get-config-payment/:payment_code | ไม่มี | routes/main.go:42 |
| GET | /api/v2/test-payment/:payment_code | ไม่มี | routes/main.go:43 |
| POST | /api/v2/check/order/:payment_code | ไม่มี | routes/main.go:71 |
| POST | /api/v2/check/ordercall/:payment_code/:service | ไม่มี (ปรับสถานะ) | routes/main.go:72 |
| POST | /api/v2/check/neworder/:payment_code/:order | ไม่มี | routes/main.go:73 |
| POST | /api/v2/confirm-transaction-payin | ไม่มี (manual confirm office) | routes/main.go:74 |
| POST | /api/v2/confirm-transaction-payin-slip | ไม่มี | routes/main.go:75 |
| POST | /api/v2/confirm-bank-deposit | ไม่มี | routes/main.go:120 |
| POST | /api/v2/cancel-bank-deposit | ไม่มี | routes/main.go:121 |
| POST | /api/v2/change-status-deposit-payment | ไม่มี | routes/main.go:85 |
| POST | /api/v2/check_balance_p2p | ไม่มี | routes/main.go:158 |
| GET | /api/v2/p2cpay/channels/:payment_code | ไม่มี | routes/main.go:160 |
| POST | /api/v2/create-p2p-qrpayment | ไม่มี | routes/main.go:154 |
| POST | /api/v2/request-resend-callback/:payment_code/:order | ไม่มี | routes/main.go:80 |
| POST | /api/v2/retry-payment-sendffice | ไม่มี | routes/main.go:141 |
| POST | /api/v2/corepay-user-register | ไม่มี | routes/main.go:111 |
| GET | /api/v2/dashboard/balancePayment | ALLOW_IP check ใน handler | routes/main.go:127, controller/dashboard/main.go:22 |
| POST | /api/v2/dashboard/dashboard | AUTH_KEY ใน handler | routes/main.go:130, controller/dashboard/main.go:130 |
| POST | /api/v2/bot/webhook | ไม่มี (telegram) | routes/main.go:166 |
| POST | /peer2pay/v3/payin | ไม่มี | routes/main.go:175 |
| POST | /peer2pay/v3/callback-payin/:payment_code | IPWhitelist | routes/main.go:176 |
| POST | /peer2pay/v3/callback-payout/:payment_code | IPWhitelist | routes/main.go:178 |
| POST | /peer2pay/v3/create-withdraw | ไม่มี | routes/main.go:183 |
| POST | /api/xpay/private/callback-payin/:payment_code | IPWhitelist | routes/main.go:195 |
| POST | /bank-gateway/verify-transfer | ไม่มี | routes/bank-transfer-gateway.go:48 |
| POST | /bank-gateway/confirm-transfer | ไม่มี | routes/bank-transfer-gateway.go:49 |
| POST | /bank-gateway/verify-and-confirm-transfer | ไม่มี | routes/bank-transfer-gateway.go:50 |
| POST | /bank-gateway/enc-aes | ไม่มี | routes/bank-transfer-gateway.go:53 |
| POST | /bank-gateway/call-by-req | ไม่มี (proxy ทั่วไป) | routes/bank-transfer-gateway.go:54 |
| ANY | /call-pm/:paymentCode/*byPath | ไม่มี (generic proxy ไป provider) | routes/call-by-payment.go:40, app.go:144 |

หมายเหตุ: เส้นทาง provides ทั้งหมดมีอีกหลายสิบ route (statement/report/dashboard/bot) — ดู routes/main.go:30-167 ครบถ้วน; ที่ลิสต์ครอบคลุม payin/payout/callback/balance/verify-slip/adjust ครบตามต้องการ

### consume (queues subscribed)
| exchange/queue | handler | evidence |
|----------------|---------|----------|
| reply queue (anonymous, exclusive) | รับ RPC reply จาก que_payment service | service/rabbit/que-payment.go:75-105 (`ch.QueueDeclare("")` + `ch.Consume` รอ reply ตาม CorrelationId) |

ไม่พบ consumer แบบ long-running ต่อ named queue — บริการนี้เป็นฝั่ง publisher/RPC client เท่านั้น

### cron
| schedule | does what | evidence |
|----------|-----------|----------|
| ไม่พบ (disabled) | `StartCronjob()` เรียก `controller.CronjobStatement` แต่ถูก comment ใน main | _cmd/main.go:31, app.go:184-202 |

`CronjobStatement` ยังถูกผูกเป็น HTTP POST `/api/v2/create-member-data` แทน (routes/main.go:110)

## consumes

### http-out
| env key | base URL value (🟡 = runtime/commented) | paths called | evidence |
|---------|----------------------------------------|--------------|----------|
| (DB) ServicePayment.HostSmartPay | 🟡 runtime จาก collection service_qrpayment | provider endpoint ต่อราย (เช่น `/v1/qrCode/create`) | controller/anypay/main.go:42, controller/call-by-payment.go:71 |
| PAYMENT_DETAIL_URL | 🟡 runtime-injected | สร้างหน้า payment detail (coalesce กับ serviceConfig.PaymentDetailURL) | controller/deposit.go:2602, 1954, 3304, 5077 |
| DOMAIN_WEB | 🟡 runtime-injected | fallback ของ payment detail URL | controller/deposit.go:5077,5135,5191 |
| ENCRYPTION_ENDPOINT | 🟡 runtime-injected | encrypt service (cloudpay/visapay) | controller/cloudpay/func.go:117-118, controller/visapay/func.go:117-118 |
| URL_VERIFYSLIP | 🟡 runtime-injected | POST verify slip ธนาคาร | service/verifySlip/verifySlip.go:15,27 |
| (DB) urlCallback ของ office | 🟡 runtime จาก config | `/api/WithdrawCallback`, `/api/Payment/PaymentCreateStatement/{svc}`, `/api/Payment/CallBackWithdrawAmount/{svc}`, `/api/Xpay/CreateStatement/{svc}`, `/api/{svc}/payment-update-balance` | service/callback/main.go:27,67,103,147,189 |
| (DB) SettingForward.UriForwardPayin/Payout | 🟡 runtime | forward callback ไป URL ลูกค้า (resty retry 3) | controller/callback.go:16058,16064 |
| BOT_TELEGRAM_TOKEN / BOT_TELEGRAM_URL_WEBHOOK | 🟡 runtime | `https://api.telegram.org/bot{token}/setWebhook` / sendMessage / sendPhoto | app.go:206, controller/botTelegram.go (`/sendMessage`, `/sendPhoto`) |
| (none) | `https://translate.googleapis.com/translate_a/single` | แปลภาษา | controller/main.go:2295 |
| (none) | `https://scanslipapi.first2pay.io/v1/scan_slip_verify` + `/scan_slip_confirm` | verify slip first2pay | controller/first2pay (hardcoded) |
| (none) | `https://api.payonex.asia/transactions/%s`, `https://gateway.smilepayz.com`, `https://superichpay.com/api/redis/%s` | askmepay providers | controller/askmepay (hardcoded) |
| API_REPORT | 🟡 commented out | report service | controller/callback.go:16033 (comment) |

### publish
| exchange/queue (exact string) | payload | evidence |
|-------------------------------|---------|----------|
| `QUE_MERCHANT_{MERCHANT_CODE}` (ถ้า MERCHANT_CODE ตั้งค่า) | JSON {id, service, type, type_report, date, payment_code, merchant_code} | service/rabbit/que-payment.go:31-35,140 + ch.Publish routing key = queueName (149) |
| `QUE_PAYMENT_{PaymentName}` (ถ้าไม่มี MERCHANT_CODE) | เหมือนข้างบน; Type = `DEPOSIT` / `REPORT_DATA_CURRENT` | service/rabbit/que-payment.go:36, controller/callback.go:15089 (DEPOSIT), :268 (REPORT_DATA_CURRENT) |

- exchange = `""` (default exchange), routing key = queueName, mandatory/immediate = false (que-payment.go:146-149)
- broker URL = `serviceConfig.RabbitmqURL` (runtime), แต่ override เป็น `amqp://guest:guest@127.0.0.1:5672` เมื่อ MODE != production (que-payment.go:47)

### data
MongoDB เดียว (database = env `DB_DBNAME`, conn = `DB_ENDPOINT`) ผ่าน raw driver (db/connect.db.go:38-75). collection literals ที่พบ (R/W):

| DB name | collection | R/W | evidence |
|---------|-----------|-----|----------|
| {DB_DBNAME} | qr_payment | R/W (54 จุด) | controller/callback.go:214 (UpdateOneDocument), :229 (GetOneDocument) |
| {DB_DBNAME} | service_qrpayment | R (34 จุด, config payment) | repository/main.go:162, controller/call-by-payment.go:33 |
| {DB_DBNAME} | service_qrpayment_temp | R/W | grep (2 จุด) |
| {DB_DBNAME} | withdraw_statement | R/W (33 จุด) | controller/withdraw.go |
| {DB_DBNAME} | confirm_bank_deposit | R/W (9 จุด) | controller |
| {DB_DBNAME} | bank_summary | R/W (7 จุด) | controller |
| {DB_DBNAME} | telegram_order_statement | R/W (6 จุด) | controller/callback.go (updateTelegramOrderLog) |
| {DB_DBNAME} | adjust_statement | R/W (5 จุด) | controller/adjustBalance.go |
| {DB_DBNAME} | telegram_connect_provider | R/W (3) | controller |
| {DB_DBNAME} | logs_callback_payment | W (3) | controller |
| {DB_DBNAME} | transaction_xpay_fail | W (2) | controller |
| {DB_DBNAME} | que_payment | W (1) | repository/main.go:32 |
| {DB_DBNAME} | qr_payment_confirm | R/W (1) | controller |
| {DB_DBNAME} | report_payment | R/W (1) | model/report_payment.go, controller/report.go |
| {DB_DBNAME} | summary_deposit / logs_inc_payment / confirm_statement / bank_account | R/W (1 ละ) | grep |

Redis (thorlock distributed lock) — `REDIS_URL` / `REDIS_PSW` (service/thorlock/couterRedis.go:31-32); init lock TTL 180s (app.go:80)

### external (payment providers)
ทุก provider เป็น package ภายใต้ `controller/{name}/` มี Deposit/Withdraw/Balance/Callback. base URL ส่วนใหญ่มาจาก `serviceConfig.HostSmartPay` (DB runtime 🟡); auth = MerchantID/MerchantSecret/ApiKey จาก config DB ต่อราย (controller/anypay/main.go:18-21,49). ผู้ให้บริการที่พบ (PAYMENT_NAME const):

ANYPAY, ANYPAYTH, APAY24, ASKMEPAY, ASKPAY, AUTOPEERPAY, AZPAY, BIBPAY, BIGPAY, BITPAYZ, BLACKDOG, CAPITALPAY, CLOUDPAY, COMPAY, COREPAY, CUBIXPAY, CUTPAYZ, DANARAPAY, DPAY, E2P, FIRST2PAY, FLAZZPAY, GBETPAY, GM2PAY, HASHPAYS, HENGPAY, HUBPAY, HYRONPAY, IMPAY, JAIJAIPAY, JIBPAYX, K2PAY, KISSPAY, LUCKYTHAI, MAANPAY, MINERAPAY, MTMPAY, MTPAY, MYPAYS24, OKEYSPAY, ONEDAYPAY, ONEPAY, P2CPAY, P2WPAY, PAPAYAPAY, PAYKRUB, PEER2PAY, SONICPAY, SPEEDPAY, SUDAHPAY, SUGARPAY, SWIFTPAY, THUNDERPAY, TRUSTPAY, UMPAY, VISAPAY, WEALTHWAVE, WORLDPAY, WOWPAY, XPAY, XUPERMA, ZAPPAY, ZEENYPAY, ZPAY, BANKTRANSFER(+bk55) — รวม ~66 provider dir (controller/*/) — enumerate จาก `find controller -maxdepth 1 -type d`

XPAY มี alias หลายตัวภายใน package เดียว: TOPPAY, SPEEDHUBPAY, DPAY, MAXPAY, APXPAY, BADOOPAY, ONEBANKPAY, PAYGT, TLCONNECTPAY, WELLPAYZ, ZAPMAN (controller/xpay grep PAYMENT_NAME_*)

providers ฝั่งอินโดเฉพาะกลุ่ม: FLAZZPAY, MINERAPAY, SUDAHPAY, K2PAY, XUPERMA, DANARAPAY (service/paymentIndo/paymentIndo.go:18-25)

third-party อื่น: Telegram Bot API (api.telegram.org), Google Translate (translate.googleapis.com), first2pay scanslip API, ธนาคารผ่าน URL_VERIFYSLIP, gstark generic proxy (service/gstark)

## observations
- **secrets committed**: `config.yaml` ถูก track ใน git (มี OTel exporter endpoint hardcode `http://34.87.23.21:4317`, prometheus `0.0.0.0:8889`) — config.yaml:11; แต่ใน .gitignore ระบุ `./config.yaml` (รูปแบบ path ผิด จึง track หลุด) .gitignore:2
- **MongoDB creds รั่วใน docker-compose**: `DB_ENDPOINT=mongodb+srv://root:Zxcvasdf789@test02.9oh7e.mongodb.net/` committed (docker-compose.yml:11) — รหัสผ่าน root จริง
- **azpay.json (Postman) committed**: มี merchant signature hash `4589cbad...0cb1f8db` และตัวอย่าง HMAC-SHA256 secret pattern (azpay.json:95,129,203)
- **callback ไม่ verify signature**: `CallbackDeposit` อ่าน body แล้วเข้า SwitchCallbackDeposit → update qr_payment → ส่ง rabbit ทันที ไม่มีการตรวจ HMAC/signature ของ provider (controller/callback.go:112-310) — ป้องกันด้วย IP whitelist เท่านั้น
- **IP whitelist ปิดโดย default**: `CHECK_IPWHITELIST != "TRUE"` → `c.Next()` ผ่านทุก IP (middleware/ipwhitelist.go:22-25) — ถ้า env ไม่ตั้ง callback-payin/payout เปิด public
- **ไม่มี idempotency เชิงโครงสร้าง**: กันซ้ำด้วย `objDep.IsSuccess == 1` เท่านั้น (callback.go:15068) ไม่มี unique index/transaction — callback ที่มาพร้อมกันก่อนสถานะถูก set อาจ credit ซ้ำ (race ระหว่าง GetConfig→Switch→CallRabbit)
- **balance/status update ไม่มี DB transaction**: ใช้ `UpdateOneDocument` รายตัว ไม่มี mongo session/txn (db/repo.go UpdateOneDocument); ใช้ thorlock (Redis lock) แทน — ถ้า Redis ล่ม lock ไม่ทำงาน
- **HTTP client ไม่มี timeout**: `helper.CallAPI` ใช้ `&http.Client{}` ไม่มี Timeout (helper/callApi.go:18); `callDemo` proxy ใช้ `&http.Client{}` เปล่า (routes/bank-transfer-gateway.go:73); `setWebhookWithSecret` ใช้ `http.Post` (app.go:230)
- **/bank-gateway/call-by-req = SSRF**: รับ `domain_call`+method+header+body จาก client แล้วยิงออกตรงๆ ไม่มี auth/whitelist (routes/bank-transfer-gateway.go:55-77)
- **generic proxy /call-pm/:paymentCode/*byPath**: ฟอร์เวิร์ดไป HostSmartPay ตาม path ที่ client กำหนด ไม่มี auth (controller/call-by-payment.go:49-80)
- **swallowed errors**: `CallRabbitDepositAndWithdraw` error → `Trac.ChildSuccess()` แล้ว return code 501 (น่าจะ Fail) (controller/callback.go:15095); read body error เพียง log ไม่ abort (callback.go:133-134)
- **weak crypto**: SHA1 ใช้สร้าง hash statement/order (helper/hash.go:10, service/hashlayout/main.go:20); PBKDF2 iteration = 5 เท่านั้น + salt = `[]byte("0")` (controller/banktransfer/encrypt-decrypt.go:141,162)
- **dev secret default ใน rabbit**: เมื่อ non-production บังคับ `amqp://guest:guest@127.0.0.1:5672` (que-payment.go:47)
- **panic on thorlock fail**: `panic(err)` ถ้า Redis lock connect ไม่ได้ตอน start (app.go:84)
- **error notifier/mongo writer fail-open**: init ล้มเหลวก็ยังรันต่อด้วย instance ว่าง (app.go:127, 138)
- **dead/commented code**: speedpay routes comment ทิ้ง (routes/main.go:18-25), AES key manage ใน bank-gateway comment ทิ้ง (routes/bank-transfer-gateway.go:17-22), RouteBankTransferGateway เก่าใน main.go comment ทั้งก้อน (routes/main.go:188-214)
- **arm64-only Dockerfile**: `--platform=arm64` hardcode ทั้ง build/run stage (Dockerfile:1,7) — deploy บน amd64 ต้อง emulate
- **callback.go ขนาด 16,182 บรรทัด** — ไฟล์เดียวคุม callback ทุก provider, แตะยาก
