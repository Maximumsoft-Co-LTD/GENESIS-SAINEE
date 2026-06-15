# Card: office-api-v10
> เธงเธดเน€เธเธฃเธฒเธฐเธซเน: 2026-06-12 | commit: 40367af

## identity
- **stack:** Go (Gin), MongoDB (raw driver), Redis (go-redis/v8), RabbitMQ (streadway/amqp), Prometheus โ€” เน€เธเนเธ back-office/hub API เนเธเธ multi-tenant
- **module:** `goofficev10` (go.mod:1, go 1.18 โ€” go.mod:3)
- **entrypoint:** `_cmd/main.go` โ’ `config.InitEnvConfig()` (main.go:10) โ’ `goofficev10.StartServer()` (main.go:14)
- **bootstrap:** `server.go:19` `StartServer()` โ€” เนเธซเธฅเธ” .env (godotenv), เน€เธเธทเนเธญเธก Mongo OFFICE db เธเธฒเธ `MONGODB_DB_NAME` (server.go:24), เธชเธฃเนเธฒเธ connection เธ—เธธเธ tenant เธเธฒเธ collection `database_config` (`db.CreateConnectionOnce` โ€” db/db.go:121), เธ•เนเธญ Redis (server.go:40), เธฅเธเธ—เธฐเน€เธเธตเธขเธ ~45 route group (server.go:51-92)
- **port:** `7777` โ€” hardcoded `r.Run(":7777")` (server.go:106). เนเธกเนเธญเนเธฒเธเธเธฒเธ env PORT
- **Dockerfile:** build เธเธฒเธ `golang:1.22.3-alpine3.20` โ’ runtime `alpine:3.20`, เธ”เธถเธ `rds-combined-ca-bundle.pem`, TZ Asia/Bangkok, `CMD ["./app"]` (Dockerfile). เนเธกเนเธกเธต EXPOSE (port เน€เธเธดเธ”เนเธเนเธเนเธ” 7777)
- **CI:** เนเธกเนเธเธเนเธเธฅเนเนเธ `.github/workflows/` เธ—เธตเนเธญเนเธฒเธเนเธ”เนเนเธเธฃเธญเธเธเธตเน โ€” เธฃเธฐเธเธธ "เนเธกเนเธเธ" เธฃเธฒเธขเธฅเธฐเน€เธญเธตเธขเธ”
- **เนเธซเธกเธ”เธเธดเน€เธจเธฉ:** เธ–เนเธฒ env `STATUS_REPORT=TRUE` เธเธฐ register เน€เธเธเธฒเธฐ `exportMember.RouteExportDataMember` เนเธ—เธ route เธ—เธฑเนเธเธซเธกเธ” (server.go:49-51)

## provides

### http
เธ—เธธเธ route เธญเธขเธนเนเนเธ•เน prefix `/api`. middleware เธซเธฅเธฑเธเธเธทเธญ `middlewares.AuthRequiredDB(resource, permission)` (JWT HS256 เธ”เนเธงเธข `ACCESS_SECRET` โ€” auth.go:63 + เธ•เธฃเธงเธ permission + fallback เน€เธฃเธตเธขเธ SUPERCOM_API). เธเธฅเธธเนเธกเธ—เธตเนเนเธเนเธ•เธฑเธงเนเธเธฃ group `global`/`glocal`/`callback`/`webhook` = **เนเธกเนเธกเธต auth middleware** (เธชเธณเธเธฑเธเธกเธฒเธ).

| METHOD | path | auth middleware | evidence (file:line) |
|--------|------|-----------------|----------------------|
| โ€” | **เธเธฅเธธเนเธก CRUD เธ—เธตเน auth เธเธฃเธ (AuthRequiredDB="")** | | |
| * | `/api/agent/*` (18 routes) | AuthRequiredDB | route/Agent.go:12-13 |
| * | `/api/configsystem + config_system (95 routes)` | เธชเนเธงเธเนเธซเธเน AuthRequiredDB; `app` group เธเธฒเธเธชเนเธงเธเนเธกเน auth | route/configsystem.go:12-13 |
| * | `/api/report/* (85 routes)` | api=AuthRequiredDB, global=เนเธกเน auth | route/report.go:13-14 |
| * | `/api/bankconfig-* (85 routes)` | api=AuthRequiredDB, callback/global=เนเธกเน auth | route/bank.go:12-16 |
| * | `/api member/* (63 routes)` | api=AuthRequiredDB | route/member.go:13-14 |
| * | `/api employee/* (54 routes)` | route/api=AuthRequiredDB | route/employee.go:14-15 |
| * | `/api sms/* (36 routes)` | AuthRequiredDB | route/sms.go:12 |
| * | `/api telesale/* (33 routes)` | api=AuthRequiredDB, global=เนเธกเน auth | route/telesale.go:13-14 |
| * | `/api extension/* (32 routes)` | AuthRequiredDB | route/extension.go:12-60 |
| * | `/api config/* (25 routes)` | api=AuthRequiredDB, global=เนเธกเน auth | route/config.go:13-14 |
| * | `/api shop/* (20 routes)` | AuthRequiredDB | route/shop.go:13 |
| * | `/api ticket/* (15 routes)` | AuthRequiredDB | route/ticket.go:13 |
| * | `/api game/* (13 routes)` | AuthRequiredDB | route/game.go:13 |
| * | `/api/bank-line-connect, /:service/line-connect/* (15 routes)` | controller-based (resty) | route/bank-line-connect.go:19 |
| * | `/api mission/bonus/wheel/ranking/tournament/pixel/checkin/wallet/whitelistIP/log` (CRUD groups) | AuthRequiredDB | route/*.go |
| GET | `/api/bankGateway/* (9 routes)` | AuthRequiredDB | route/bankGateway.go:12 |
| โ€” | **DEPOSIT (transaction-critical)** | | |
| GET | `/api/GetDepositStatement/:service` ... (เธชเธ–เธดเธ•เธดเธเธฒเธ ~14 GET) | AuthRequiredDB="" | route/deposit.go:15-37 |
| POST | `/api/DepositAddCredit/:service` | AuthRequiredDB="" | route/deposit.go:35 |
| POST | `/api/ConfirmMoveStatement/:service` | AuthRequiredDB="" | route/deposit.go:34 |
| POST | `/api/deposit-remain-move/:service` | AuthRequiredDB="" | route/deposit.go:31 |
| POST | `/api/slip-verify-deposit-statement/:service` | AuthRequiredDB="" | route/deposit.go:38 |
| POST | `/api/slip-verify-deposit-statement-by-member/:service` | **เนเธกเนเธกเธต auth (global)** | route/deposit.go:39 |
| โ€” | **WITHDRAW (transaction-critical)** | | |
| POST | `/api/withdraw-confirm/:service` | AuthRequiredDB="WITHDRAW_AUTO" | route/withdraw.go:35 |
| POST | `/api/withdraw-confirmauto/:service` | AuthRequiredDB="WITHDRAW_AUTO" | route/withdraw.go:36 |
| POST | `/api/withdraw-again/:service` | AuthRequiredDB="WITHDRAW_AUTO" | route/withdraw.go:38 |
| POST | `/api/withdraw-confirmmanual/:service` | AuthRequiredDB="" (api group) | route/withdraw.go:39 |
| POST | `/api/withdraw-confirmedit/:service` | AuthRequiredDB="" | route/withdraw.go:40 |
| POST | `/api/withdraw-cancel/:service` | AuthRequiredDB="" | route/withdraw.go:37 |
| POST | `/api/withdraw-hidden/:service` | AuthRequiredDB="" | route/withdraw.go:41 |
| POST | `/api/confirm-withdraw-edit-change-account-status/:service` | AuthRequiredDB="WITHDRAW_AUTO" | route/withdraw.go:43 |
| POST | `/api/test-withdraw/:service`, `/api/test-withdraw-truewallet/:service` | AuthRequiredDB="" | route/withdraw.go:46-47 |
| POST | `/api/ManageWithdrawStatementRemain-auto-cornjobs/:service` | `CheckkeyCronJobNextDay` (JWT key hardcoded) | route/withdraw.go:22 |
| POST | `/api/WithdrawCallback` | **เนเธกเนเธกเธต auth (global)** | route/withdraw.go:53 |
| POST | `/api/auto/createQueueWithdraw/:service` | **เนเธกเนเธกเธต auth (global)** | route/withdraw.go:59 |
| POST | `/api/WithdrawStatementBotMode/:service` | **เนเธกเนเธกเธต auth (global)** | route/withdraw.go:64 |
| POST | `/api/checkWithdrawStatementBotMode/:service` | **เนเธกเนเธกเธต auth (global)** | route/withdraw.go:65 |
| POST | `/api/CallbackWithdrawStatementBotMode/:service` | **เนเธกเนเธกเธต auth (global)** | route/withdraw.go:66 |
| โ€” | **CREDIT (transaction-critical)** | | |
| POST | `/api/credit-check/:service` | AuthRequiredDB="" | route/credit.go:18 |
| POST | `/api/credit-return/:service` | AuthRequiredDB="TRANSECTION_MANAGECREDIT_RETURN" | route/credit.go:19 |
| POST | `/api/credit-delete/:service` | AuthRequiredDB="" | route/credit.go:20 |
| POST | `/api/credit-freebonus/:service` | AuthRequiredDB="" | route/credit.go:21 |
| POST | `/api/credit-editturn`, `/credit-point`, `/set-point`, `/free-wallet`, `/credit-delete-wallet`, `/credit-return-wallet`, `/deposit-add-credit-real-wallet` /:service | AuthRequiredDB="" | route/credit.go:22-29 |
| POST | `/api/return-withdraw-statement-credit/:service/:withdrawID` | AuthRequiredDB="" | route/credit.go:30 |
| GET | `/api/get-credit-list, get-bonus-list, get-turn-list, get-point-list, get-ticket-list, get-aff-list /:service` | AuthRequiredDB="" | route/credit.go:32-37 |
| โ€” | **APPROVE BANK (transaction-critical)** | | |
| GET | `/api/bank-member/:service`, `/api/bank-memberv2/:service` | AuthRequiredDB="" | route/approveBank.go:17-18 |
| POST | `/api/bank-member/approve/:service` | AuthRequiredDB="" | route/approveBank.go:21 |
| POST | `/api/bank-member/approve-all/:service` | AuthRequiredDB="" | route/approveBank.go:22 |
| POST | `/api/bank-member/reject/:service`, `/reject-all/:service` | AuthRequiredDB="" | route/approveBank.go:23-24 |
| โ€” | **STATEMENT / CREATE (transaction-critical)** | | |
| POST | `/api/CreateStatement/:service` | AuthRequiredDB="TRANSECTION_INSERT" | route/statement.go:25 |
| POST | `/api/:service/VerifyBeforeCreateStatement` | AuthRequiredDB="TRANSECTION_INSERT" | route/statement.go:26 |
| POST | `/api/CreateStatementSlip/:service` | AuthRequiredDB="TRANSECTION_INSERT" | route/statement.go:28 |
| POST | `/api/CreateStatementTrueWallet/:service` | **เนเธกเนเธกเธต auth (global)** | route/statement.go:30 |
| POST | `/api/QRCODE/CreateStatement/:service` | **เนเธกเนเธกเธต auth (global)** | route/statement.go:31 |
| POST | `/api/redis-peer2pay` | **เนเธกเนเธกเธต auth (global)** โ€” เธเธทเธ access_key/secret_key | route/statement.go:22 |
| POST | `/api/callback-peer2pay` | AuthRequiredDB="" (api group) | route/statement.go:21 |
| โ€” | **PAYMENT callbacks (transaction-critical)** | | |
| POST | `/api/Payment/PaymentCreateStatement/:service` | **เนเธกเนเธกเธต auth (glocal)** | route/payment.go:17 |
| POST | `/api/Payment/CallBackWithdrawAmount/:service` | **เนเธกเนเธกเธต auth (glocal)** | route/payment.go:18 |
| POST | `/api/Xpay/CreateStatement/:service` | **เนเธกเนเธกเธต auth (glocal)** | route/payment.go:19 |
| POST | `/api/callback-update-status/withdraw` | **เนเธกเนเธกเธต auth (glocal)** | route/payment.go:20 |
| POST | `/api/:service/payment/bot-telegram/* (botTelegram group)` | controller เธ•เธฃเธงเธ permission เธ เธฒเธขเนเธ | route/payment.go:57 |
| โ€” | **BANK callbacks (transaction-critical)** | | |
| POST | `/api/withdraw_proxy_bank` | **เนเธกเนเธกเธต auth (callback)** | route/bank.go:101 |
| POST | `/api/change_account_proxy_bank` | **เนเธกเนเธกเธต auth (callback)** | route/bank.go:108 |
| POST | `/api/bank-account-install-withdraw` | **เนเธกเนเธกเธต auth (callback)** | route/bank.go:110 |
| POST | `/api/firebase-noti-bank_config/:service` | **เนเธกเนเธกเธต auth (global)** | route/bank.go:136 |
| POST | `/api/bank-config-set-access-token/:service` | **เนเธกเนเธกเธต auth (global)** | route/bank.go:141 |
| โ€” | **MEMBER bank webhook (transaction-critical)** | | |
| POST | `/api/webhook/verify-bank/:username` | **เนเธกเนเธกเธต auth (webhook group)** | route/member.go:102-103 |
| โ€” | **EXTERNAL SERVICE (money-moving)** | | |
| POST | `/api/external-service/bonus/:service` | **เนเธกเนเธกเธต auth** (เธญเธขเธนเนเธเนเธญเธ `api.Use(AuthRequiredDB)` โ€” เนเธเธเนเธเธเธฑเธชเธเธฃเธต) | route/externalService.go:12-13 |
| GET/PATCH/POST | `/api/external-service/* (เธญเธทเนเธเน)` | AuthRequiredDB="" | route/externalService.go:15-20 |
| โ€” | **CONFIG / MIGRATE (admin, เนเธกเน auth)** | | |
| GET | `/api/GetDataDefaultTemplate/:service`, `/api/DF_OFFICE/:service` | **เนเธกเนเธกเธต auth (global)** | route/config.go:46-47 |
| POST | `/api/init-service`, `/api/init-service-callback` | **เนเธกเนเธกเธต auth (global)** โ€” เธชเธฃเนเธฒเธ/เธขเนเธฒเธข service เนเธซเธกเน | route/config.go:48-49 |
| POST | `/api/update-configsystem/:service` | **เนเธกเนเธกเธต auth (global)** | route/migrate.go:15 |
| POST | `/api/clear-script-status-game-report/:service` | **เนเธกเนเธกเธต auth (global)** | route/migrate.go:17 |
| GET | `/api/clear-redis-service/:key` | **เนเธกเนเธกเธต auth (global)** | route/report.go:28 |
| POST | `/api/dashboard-summary-refresh` | **เนเธกเนเธกเธต auth (global)** | route/report.go:97 |
| GET | `/api/report-reset-outstanding` | **เนเธกเนเธกเธต auth (global)** | route/report.go:120 |
| GET | `/api/count-enable-unlock-detail` | **เนเธกเนเธกเธต auth (global)** | route/report.go:128 |
| โ€” | **LOGIN platform (auth เธ เธฒเธขเนเธ controller)** | | |
| GET | `/api/login-platform`, `/api/make-link-login`, `/api/register-platform` | เนเธกเนเธกเธต middleware (callbackLogin.go) | route/callbackLogin.go:13-15 |
| POST | `/api/login-platform-verify` | เนเธกเนเธกเธต middleware | route/callbackLogin.go:16 |
| โ€” | **RUNBANK** | | |
| POST | `/api/runbank-office` | เนเธกเนเธกเธต middleware | route/runbank.go:21 |
| โ€” | **VERIFY SLIP / WALLET / WHITELIST / MIGRATE** | | |
| * | `/api/verifyslip/* (9), /api/wallet/* (4), /api/whitelistIP/* (4), /api/migrate/* (4)` | เธเธเธเธฑเธ api/global | route/verifyslip.go, wallet.go, whitelistIP.go, migrate.go |

> เธฃเธงเธก route เธ—เธฑเนเธเธฃเธฐเธเธ ~900+ registration (เธเธฒเธเธเธฒเธฃเธเธฑเธ `.GET/.POST/...` เธ•เนเธญเนเธเธฅเน โ€” configsystem 95, report 85, bank 85, member 63, employee 54, withdraw 39, payment 38, sms 36, statement 34, telesale 33, extension 32 เธฏเธฅเธฏ)

### consume (queues subscribed)
| exchange/queue/channel | handler | evidence |
|------------------------|---------|----------|
| เนเธกเนเธเธ โ€” repo เธเธตเนเน€เธเนเธ **publisher เธญเธขเนเธฒเธเน€เธ”เธตเธขเธง** เนเธกเนเธกเธต `ch.Consume(...)` เนเธ” เน | โ€” | grep `Consume(` เธ—เธฑเนเธ repo = เนเธกเนเธเธ |

### cron
| schedule | does what | evidence |
|----------|-----------|----------|
| เนเธกเนเธกเธต cron scheduler เนเธเธ•เธฑเธง (เนเธกเนเธกเธต gocron/ticker AddFunc) | โ€” | grep gocron/cron./AddFunc = เนเธกเนเธเธ |
| ๐ก endpoint cron เนเธเธเน€เธฃเธตเธขเธเธเธฒเธเธ เธฒเธขเธเธญเธ: `POST /api/ManageWithdrawStatementRemain-auto-cornjobs/:service` เธเนเธญเธเธเธฑเธเธ”เนเธงเธข `CheckkeyCronJobNextDay` (เน€เธ—เธตเธขเธ JWT key เธ•เธฒเธขเธ•เธฑเธง) | เธเธฑเธ”เธเธฒเธฃเธ—เธธเธเธ–เธญเธเธขเธเธขเธญเธ”เธเนเธฒเธกเธงเธฑเธ | route/withdraw.go:22 ; auth.go:614-627 |
| `time.Sleep` เนเธเนเนเธเธซเธฅเธฒเธขเธเธธเธ” (เนเธกเนเนเธเน cron เนเธ—เน) | เธซเธเนเธงเธเน€เธงเธฅเธฒ | controller/payment.go:1523, statement.go:2813 |

## consumes

### http-out
| env key | base URL value (๐ก = injected/commented) | paths called | evidence |
|---------|------------------------------------------|--------------|----------|
| `SUPERCOM_API` | `https://supercompany-dev.thesonicblue.xyz/api` (.env) | `/office/{PROJECT}/member/{id}`, `/office/getpermission-member?service=`, `/office/{}/member/login-platform`, `/office/{}/member/{}/get-service-by-username`, `/office/{}`, `/office`, `/icon/preset/office/{}/{}`, `/icon/preset/office/{}` | auth.go:179-181; supercomOffice.go:18,36; iconPreset.go:75 |
| `PAYMENT_API` | `https://payment-uat.themoonnight.com` (.env); fallback hardcode `https://payment-3rd-dev.themoonnight.com` | `/api/v2/get-bankconfig`, `/api/v2/get-payment-auto-withdraw`, `/api/v2/check_balance_p2p`, `/api/v2/callback-payin/{}-PEER2PAY`, `/api/v2/bot/send-message`, `/api/v2/bot/update-log-telegram`, `/api/v2/bot/log-telegram-list` | withdraw.go:4278; payment.go:48,241,1392+; statement.go:1608 |
| `SMS_API` | `https://uat.sms-kub.com/api` (.env) | `/campaigns`, `/campaigns/`, `/campaigns/status/`, `/campaigns/retarget/`, `/check-sender/`, `/configs/rate_message`, `/phone`, `/senders/usable`, `/report-tracking/`, `/user`, `/users` | controller/sms.go (เธซเธฅเธฒเธขเธเธธเธ”) |
| `COUPON_SERVICE` | `https://coupon-service-dev.thesonicblue.xyz/api` (.env) | `/coupon/{}?service=`, `/coupon/code/{}`, `/coupon/prefix/{}`, `/coupon?service=`, `/statement/{}?...` | controller/extention.go:1095+ |
| `HASH_CENTRAL_API` | `https://hash-central-dev.uppicture.online` (.env) | `/api/get-channel`, `/api/change-channel`, `/api/save-device-botapp` | bank.go:1668; optionBank.go:58,138; runbankFunc.go:626 |
| `PROXY_APP_URL` | `https://4j5j6s7m-7778.asse.devtunnels.ms` (.env ๐ก devtunnel) | `/api/bank-account`, `/api/bank-account/`, `/api/bank-account/update-current`, `/api/bank-account-already`, `/api/job`, `/api/job-status` | accountOption.go:115; bank.go:1751,2949,3000,3441 |
| `URLBILLPAYMENT` | `https://console-blpmv3.me` (hardcode fallback) | `/api/GetBillPaymentByService`, `/api/BillPaymentByID?id=`, `/api/ConfirmStatus4BillPayment?id=` | report.go:2164-2241 |
| `GATEWAY_API` | ๐ก เนเธกเนเธกเธตเนเธ .env; fallback `https://payment-3rd-dev.themoonnight.com` | `/api/get-list-gateway`, `/api/get-connected-by-account`, `/api/disconnect-default-geteway` | bankGateway.go:59,128,326+ |
| `URLK8S` | ๐ก เนเธกเนเธกเธตเนเธ .env; hardcode `http://18.141.146.113:6000/api/...` (เธชเนเธงเธเนเธซเธเน comment) | `/api/install-helm-withdraw`, `/api/install-helm-withdraw-change-account`, `/api/uninstall-helm-...`, `/api/initWithdraw` | bank.go:174,965,2045; payment.go:334,496 |
| `LOGIN_GATEWAY_URL` | `https://oauth.game-auth.com` (.env) | `/?callback=` | callbackLogin.go:333,373,480 |
| `ENDPOINT_AUTH_VPN` | `http://localhost:8882` (.env ๐ก) | VPN auth (vpn/vpn.go) | vpn/vpn.go:97,117 |
| `EASYSLIP_API` | ๐ก เนเธกเนเธกเธตเนเธ .env | `/api/v1/verify`, `/api/v1/verify/trueWallet` | helper/verifySlip/verifySlip.go:162,282 |
| `URL_SLIP_VERIFY` | ๐ก เนเธกเนเธกเธตเนเธ .env (config struct) | hash-central slip verify | verifySlip.go:391,434 |
| `OTP_SCB_URL` | ๐ก เนเธกเนเธกเธตเนเธ .env | `/request-otp`, `/receive-otp` | bank.go:2613,2683 |
| `TELESALE_DOMAIN` | ๐ก เนเธกเนเธกเธตเนเธ .env; fallback `https://telesale-service.thesonicblue.xyz/api` | `/api`, `/api/telesale/members-summary/`, `/api/telesale/configs/` | telesale.go:52,2255,4686 |
| `GSB_SERVICE` | ๐ก เนเธกเนเธกเธตเนเธ .env | `/generate_otp`, `/confirm_otp` | bank.go:3172,3196 |
| `LINE_CONNECT_API` | ๐ก เนเธกเนเธกเธตเนเธ .env; fallback ngrok `https://6d61-34-87-96-153.ngrok-free.app` | line connect update bank | bank.go:4565,4740 |
| `HOTLINE_API_URL` / `HOTLINE_API_KEY` | ๐ก เนเธกเนเธกเธตเนเธ .env | `/api/v1{path}` (Bearer apiKey) | bank-line-connect.go:30,50 |
| `MAINTOPUP` | ๐ก เนเธกเนเธกเธตเนเธ .env | `/api/SaveTrueWalletAccount` | bank.go:170; config.go:2106 |
| `THIRD_P2P_KEY` | `M1JEUDJQQkFTRTY0Cg` (.env, base64-ish key) | เธชเนเธเนเธ body check_balance_p2p | payment.go:1448 |
| โ€” hardcoded URLs เธญเธทเนเธ (เนเธกเนเธเนเธฒเธ env) | `https://checkbank.thesonicblue.xyz/check`, `/insertBank` | เธ•เธฃเธงเธเธเธฑเธเธเธต | approveBank.go:483,518; bank.go:2427,2491 |
| โ€” | `https://api.line.me/oauth2/v2.1/token`, `/v2/profile` | LINE OAuth login | callbackLogin.go:685,736; employee.go:2449 |
| โ€” | `https://seamless.game24hr.com/api/v1/topup/*` | wheel/topup config | controller/wheel.go (เธซเธฅเธฒเธขเธเธธเธ”) |
| โ€” | `http://*.api-th.com / *.api-node.com / *.abagroup.api-url.com /newAPI/agent*` | game agent (918kiss/pussy888/allbet) | deposit.go:1701-1746 |
| โ€” | `https://graph.facebook.com/v19.0/{}/events` | FB Pixel CAPI | pixel.go:659 |
| โ€” | `https://notify-api.line.me/api/notify` | LINE Notify | report/publish.go:69 |
| โ€” | `https://monitor-customer-service.thezeus.online/api/recive/notify` | monitor notify | configsystem.go:3981 |
| โ€” | `{database_config.api_url}/kontor/system-config`, `/office/systemConfig` (๐ก runtime DB value `APIFONTENDURL`) | เนเธเนเธ frontend เนเธซเน refresh config | payment.go:128-140 |

### publish
RabbitMQ broker URL เธกเธฒเธเธฒเธเธเนเธฒ `dbConfig.Rabbitmq` (เธเธดเธฅเธ”เนเนเธ `database_config` เธ•เนเธญ tenant โ€” runtime injected ๐ก), เนเธกเนเนเธเน env เน€เธ”เธตเธขเธง.

| exchange/queue (exact string) | payload summary | evidence |
|-------------------------------|-----------------|----------|
| exchange `STATEMENT_<SERVICE>` (fanout, durable) Type=`BANK` | BankStatement JSON | helper/queue.go:162,180 (PublishBankStatement); report/publish.go:29 |
| exchange `STATEMENT_<SERVICE>` Type=`WITHDRAW` | withdraw statement JSON | helper/queue.go:212,230 (PublishWithdrawBankStatement) |
| exchange `STATEMENT_<SERVICE>` Type=`DEPOSIT` | deposit statement JSON | helper/queue.go:260,278 (PublishDepositStatement) |
| exchange `STATEMENT_<SERVICE>` Type=`MIGRATECOUPONV2` | coupon migrate JSON | helper/queue.go:309,327 (PublishMigrateCouponV2) |
| queue `OUTSTANDING_<SERVICE>` (default exchange, routing key) Type=`OUTSTANDINGUPDATE` | statement JSON | helper/queue.go:359,371 (PublishOutstanding) |
| queue `<SERVICE>` (default exchange, durable) body=ObjectID hex Type=text/plain (UPDATECONFIG เธชเธณเธซเธฃเธฑเธ restart) | add-credit task / restart deposit | helper/queue.go:11-50 (CreateQueueAddCredit), :56-90 (PublishRestartDeposit); controller/runbankFunc.go:111 |
| queue `WITHDRAW_<bankNumber>` (default exchange) body={service,id} JSON | เธชเธฑเนเธเธ–เธญเธเน€เธเธดเธ auto เธเนเธฒเธ bot/proxy | helper/queue.go:96-141 (CreateQueueWithdraw); withdraw.go:1607,1776,2001,2873,3106; เน€เธฃเธตเธขเธ bank.go:2160 |
| queue `CHANGE_ACCOUNT_<bankNumber>` (default exchange) body={service,id} JSON | เธชเธฑเนเธเนเธขเธเธเธฑเธเธเธต auto | bank.go:2159-2160; withdraw.go:2264 |

### data
Multi-tenant: `resource[service]` = tenant DB (เธเธทเนเธญเธเธฒเธ `database_config.db_name`), `resource["OFFICE"]` = central office DB (`MONGODB_DB_NAME`, .env=`demo-dev_office`). เนเธกเนเธกเธต ORM, เนเธเน raw mongo driver.

| DB | collection | R/W | evidence |
|----|-----------|-----|----------|
| OFFICE (central) | `database_config` | R (เนเธซเธฅเธ” tenant เธ—เธฑเนเธเธซเธกเธ”เธ•เธญเธ boot + lookup service) | db/db.go:121; bank.go:2155 |
| OFFICE | `bank_config` | R/W (เธญเธฑเธเน€เธ”เธ• balance, status, credential) | bank.go (137 เธเธธเธ”); withdraw.go:2882 |
| OFFICE | `employees` | R/W (login, re-2fa) | helper.go:1339; auth flow |
| OFFICE | `roles`, `department`, `system_config`, `options_bank`, `bank_config_other`, `external_service_config`, `expense_cost(_template)`, `user_vip_group`, `report_summaryall`, `smsauto_statement`, `sms_sender`, `log_sms_auto`, `bank_account_change_auto`, `bank_account_switch`, `bank_account_register`, `bank_list_image`, `balance_credit_agent`, `demo_account`, `work_logs` | R/W | grep collection usage (controller/*) |
| tenant `<service>` | `config_system` | R/W (283 เธเธธเธ” โ€” config เธซเธฅเธฑเธ) | controller/* |
| tenant | `member_account` | R/W (132 เธเธธเธ”) | controller/* |
| tenant | `withdraw_statement` | R/W (77 เธเธธเธ” โ€” เธชเธ–เธฒเธเธฐเธ–เธญเธ, balance) | withdraw.go |
| tenant | `deposit_statement` | R/W (64 เธเธธเธ”) | deposit.go |
| tenant | `bank_statement`, `bank_statement_3d`, `bank_statement_withdraw_whitelist` | R/W | runbankFunc.go:94; statement.go |
| tenant | `wallet_statement`, `statement_real_wallet` | R/W | credit.go |
| tenant | `withdraw_amount`, `withdraw_statement_change_account` | R/W | withdraw.go:2148 |
| tenant | `event_config`, `event_statement`, `bonus_config`, `member_bonus`, `member_promotion`, `member_affiliated`, `ticket_config`, `ranking_config`, `self_stat_ranking/realtime`, `report_active_user_day/month`, `report_bonus_day`, `config_history`, `game_group`, `game_list`, `games_history`, `games_report`, `bank_list`, `bank_list_qrpay`, `line_config`, `line_message`, `line_pic`, `shop_*`, `mission_statement`, `multiple_deposit`, `campaign`, `ads_collection`, `article`, `member_stack`, `member_overdue`, `stack_configs`, `generate_statement`, `statement_affiliate_lotto` | R/W | grep collection usage |
| Redis | tenant `resource[].RDB` + `helper.RedisConnect()` (env `REDIS_URL`, `REDIS_DB`) | cache config_system, lock เธ–เธญเธเน€เธเธดเธ (thorlock), peer2pay key | db/rd.go:56-74; runbank.go:11; withdraw.go:1590 |

### external
| third-party | purpose | auth method | evidence |
|-------------|---------|-------------|----------|
| AWS S3 (ap-southeast-1, bucket rocketwinoffice) | เธญเธฑเธเนเธซเธฅเธ”/เธญเนเธฒเธเนเธเธฅเนเธ เธฒเธ | **static credentials hardcode** | config.go:977; helper.go:666; uploadFile/uploadfiles.go:20 |
| LINE (OAuth + Notify + Messaging) | login เธเธเธฑเธเธเธฒเธ, เนเธเนเธเน€เธ•เธทเธญเธ, line-connect | OAuth token / Bearer line token | callbackLogin.go:685; report/publish.go:69; bank-line-connect.go |
| Telegram Bot | เนเธเนเธเน€เธ•เธทเธญเธ/เธชเนเธ order | token env `TELEGRAM_TOKEN_SECRET` (.env) | callbackLogin.go:635; payment bot routes |
| Facebook Graph (Pixel CAPI) | track event | access token (config) | pixel.go:659 |
| EasySlip | เธ•เธฃเธงเธเธชเธฅเธดเธ | env `EASYSLIP_API` (Bearer) | verifySlip.go:162 |
| game seamless game24hr / 918kiss / pussy888 / allbet | game agent topup/config | URL เธ•เธฒเธขเธ•เธฑเธง | wheel.go; deposit.go:1701 |
| Firebase Realtime DB (withdraw-monitor) | log เธชเธ–เธฒเธเธฐเธ–เธญเธ | URL เธ•เธฒเธขเธ•เธฑเธง | withdraw.go:3136 |

## observations
- **Hardcoded MongoDB credential เนเธ source (non-prod):** `mongodb+srv://root:Zxcvasdf789@test02.9oh7e.mongodb.net/test` เธเธฑเธเนเธเนเธเนเธ” db/db.go:85, :199 (เนเธฅเธฐ .env เธเธฃเธฃเธ—เธฑเธ” MONGODB_ENDPOINT) โ€” เธฃเธฑเนเธง credential
- **Hardcoded AWS keys เนเธ source:** `[REDACTED_AWS_KEY_ID]` / secret `[REDACTED_AWS_SECRET]` (config.go:977-978, helper.go:666-667) เนเธฅเธฐเธญเธตเธเธเธธเธ” `[REDACTED_AWS_KEY_ID_2]`/`KAzp5OUR0CTkcyM+...` (uploadFile/uploadfiles.go:20-21) โ€” live AWS credential เนเธ repo
- **.env เธ–เธนเธ commit เธเธฃเนเธญเธก secret เธเธฃเธดเธ:** ACCESS_SECRET=`@OFFICEV10`, CHANNELSECRET, SECRET_KEY_2FA=`9kake...`, LOGIN_SECRET_KEY, TELEGRAM_TOKEN_SECRET=`7414505541:AAG...`, THIRD_P2P_KEY (.env เธ—เธฑเนเธเนเธเธฅเน)
- **JWT key เธชเธณเธซเธฃเธฑเธ cron endpoint เธเธฑเธเธ•เธฒเธขเธ•เธฑเธง:** `secretKey := "2ac714e8d1451244432013c2d7aa2fd7"` เนเธฅเธฐเน€เธ—เธตเธขเธเธเธฑเธ `"147ce3cba244318c00c298680ca1da08"` (auth.go:617-620) โ€” เนเธเธฃเธฃเธนเนเธเนเธฒเธเธตเนเน€เธฃเธตเธขเธ endpoint เธเธฑเธ”เธเธฒเธฃเธ—เธธเธเธ–เธญเธเนเธ”เน
- **money-moving / callback endpoints เนเธกเนเธกเธต auth middleware:**
  - `POST /api/WithdrawCallback` (withdraw.go:53) โ€” เธญเธฑเธเน€เธ”เธ• withdraw_statement status=success เธ•เธฒเธก body, เนเธกเนเธกเธต signature/HMAC verify (handler controller/withdraw.go:3202)
  - `POST /api/auto/createQueueWithdraw/:service` (withdraw.go:59) โ€” เธชเธฃเนเธฒเธ queue เธ–เธญเธเน€เธเธดเธ, เนเธกเนเธกเธต auth
  - `POST /api/external-service/bonus/:service` (externalService.go:12) โ€” เนเธเธเนเธเธเธฑเธชเธเธฃเธต, เธฅเธเธ—เธฐเน€เธเธตเธขเธเธเนเธญเธ `api.Use(AuthRequiredDB)` เธเธถเธเนเธกเนเธเนเธฒเธ middleware
  - `POST /api/Payment/PaymentCreateStatement/:service`, `/Payment/CallBackWithdrawAmount/:service`, `/Xpay/CreateStatement/:service`, `/callback-update-status/withdraw` (payment.go:17-20) โ€” เธชเธฃเนเธฒเธ statement/credit เธเธฒเธ callback, เนเธกเนเธกเธต auth
  - `POST /api/withdraw_proxy_bank`, `/change_account_proxy_bank`, `/bank-account-install-withdraw` (bank.go:101,108,110) โ€” callback bank, เนเธกเนเธกเธต auth
  - `POST /api/CreateStatementTrueWallet/:service`, `/QRCODE/CreateStatement/:service` (statement.go:30-31) โ€” เธชเธฃเนเธฒเธ statement เธเธฒเธ เนเธกเนเธกเธต auth
  - `POST /api/webhook/verify-bank/:username` (member.go:103) โ€” เนเธกเนเธกเธต auth, เนเธกเนเธกเธต signature verify
  - `POST /api/redis-peer2pay` (statement.go:22) โ€” **เธชเนเธเธเธทเธ access_key/secret_key เธเธญเธ peer2pay** เนเธ”เธขเนเธกเนเธกเธต auth (statement.go:3012-3037)
- **WithdrawCallback เธกเธต idempotency เธเธฒเธเธชเนเธงเธ** (filter `is_success != 1` + เน€เธเนเธ `IsSuccess==1` เธเธทเธเธ—เธฑเธเธ—เธต โ€” withdraw.go:3211-3221) เนเธ•เน **เนเธกเนเธกเธต signature verification** เน€เธฅเธข
- **CreateStatementPayment / payment callbacks เธกเธต permission check เธ–เธนเธ comment เธ—เธดเนเธ** (payment.go:1396-1403) โ’ เน€เธซเธฅเธทเธญ unauthenticated
- **balance update เนเธกเนเธญเธขเธนเนเนเธ transaction:** withdraw.go:2880-2884 เธญเธฑเธเน€เธ”เธ• `bank_config.balance = balance - amount` เธ”เนเธงเธข UpdateOne เนเธขเธเธเธฒเธเธเธฒเธฃเธญเธฑเธเน€เธ”เธ• withdraw_statement (เนเธกเนเธกเธต multi-doc transaction); เธกเธต Redis lock เน€เธเธเธฒเธฐ flow cancel (thorlock) เนเธ•เนเนเธกเนเธเธฃเธญเธ balance เธ—เธฑเนเธเธซเธกเธ”
- **HTTP clients เธชเนเธงเธเนเธซเธเนเนเธกเนเธกเธต timeout:** `&http.Client{}` เนเธ accountOption.go:79, bank.go:512,694,4519,4702, bankGateway.go:69,142,225 เธฏเธฅเธฏ โ€” เธกเธตเนเธเน pixel.go:660 เธ—เธตเนเธ•เธฑเนเธ Timeout 15s; `vicanso/go-axios` (axios.Get/Post) เนเธเน default เธ—เธฑเนเธงเธ—เธฑเนเธ repo
- **weak crypto:** เนเธเน `crypto/md5` (callbackLogin.go:921,952; config.go:673) เนเธฅเธฐ `crypto/sha1` (helper.go:1117,1410; bank.go:2111; credit.go:191) เนเธเธเธฒเธฃ hash/เธชเธฃเนเธฒเธ key โ€” เนเธกเนเน€เธซเธกเธฒเธฐเธเธฑเธ security context
- **CORS เน€เธเธดเธ”เธเธงเนเธฒเธเธชเธธเธ”:** `Access-Control-Allow-Origin: *` + Methods/Headers `*` (server.go:111-113)
- **swallowed errors เน€เธขเธญเธฐเธกเธฒเธ:** queue publish helper เธเธทเธ `return` เน€เธเธตเธขเธ เน เน€เธกเธทเนเธญ amqp.Dial/Channel เธฅเนเธกเน€เธซเธฅเธง (queue.go:13-22, 102-110; runbankFunc.go:113-124) โ€” เธเนเธญเธเธงเธฒเธก queue เธ–เธญเธ/เน€เธเธฃเธ”เธดเธ•เธญเธฒเธเธซเธฒเธขเนเธ”เธขเนเธกเนเธกเธต log; `axios.Post(... )` เธเธฅเธฅเธฑเธเธเนเธ–เธนเธเธ—เธดเนเธ (payment.go:128-130, 138-140)
- **REDIS_URL เธญเนเธฒเธ password เธงเนเธฒเธเน€เธชเธกเธญ:** `Password: ""` hardcode (db/rd.go:68) เนเธกเนเธกเธต env REDIS_PSW เธเนเนเธกเนเธ–เธนเธเนเธเนเนเธเธเธธเธ”เธเธตเน
- **PROXY_APP_URL เนเธ .env เธเธตเน devtunnels.ms** (`https://4j5j6s7m-7778.asse.devtunnels.ms`) โ€” endpoint dev เธชเนเธงเธเธ•เธฑเธงเธซเธฅเธธเธ”เน€เธเนเธฒ config
- **LINE_CONNECT_API fallback เน€เธเนเธ ngrok URL เธ•เธฒเธขเธ•เธฑเธง** `https://6d61-34-87-96-153.ngrok-free.app` (bank.go:4567,4742) โ€” dead/throwaway endpoint
- **dead code ๐ก:** `InitMoveDataServiceWait` (config.go:1932) เนเธกเนเธ–เธนเธ route; route เธซเธฅเธฒเธขเธเธฃเธฃเธ—เธฑเธ”เธ–เธนเธ comment (telesale.go:18-47, report.go:122, migrate.go:16, bank.go:104); `Re2FAOffice`, `ReIsLoginTelegramLine` เธ–เธนเธ comment เนเธ server.go:96-99; เธเธฑเธเธเนเธเธฑเธ startup `FixIsLoginTelegramLine` เธ—เธณ UpdateMany reset secret เธเธญเธเธเธเธฑเธเธเธฒเธ TELEGRAM/LINE เธ—เธธเธเธเธฃเธฑเนเธเธ—เธตเน boot (server.go:101; helper.go:1333) โ€” side-effect เธ•เธญเธ start เธ—เธธเธเธเธฃเธฑเนเธ
- **JWT lib เน€เธเนเธฒ/เธเนเธณ:** เนเธเนเธ—เธฑเนเธ `dgrijalva/jwt-go v3.2.0` (deprecated, เธกเธต CVE) เนเธฅเธฐ `golang-jwt/jwt v3.2.2` เธเธฃเนเธญเธกเธเธฑเธ (go.mod:11,16); auth เนเธเน dgrijalva (auth.go:21)
- **go.mod เธเธฃเธฐเธเธฒเธจ go 1.18** เนเธ•เน Dockerfile build เธ”เนเธงเธข go 1.22.3 โ€” เน€เธงเธญเธฃเนเธเธฑเธเนเธกเนเธ•เธฃเธ
- **AuthHeader (เนเธกเนเนเธเน DB) เนเธกเน return เธซเธฅเธฑเธ AbortWithStatusJSON เน€เธกเธทเนเธญ ExtractToken error** (auth.go:54-58) โ€” เนเธเนเธ”เน€เธ”เธดเธเธ•เนเธญเนเธ panic เธ—เธตเน `bearToken[1]` (auth.go:60)
- **fallback URL เธเธฑเธเธ•เธฒเธขเธ•เธฑเธงเน€เธกเธทเนเธญ env เธงเนเธฒเธ** เธเธฃเธฐเธเธฒเธขเธ—เธฑเนเธง payment.go/credit.go (เน€เธเนเธ `https://payment-3rd-dev.themoonnight.com`, `https://sc-prd.asiawallet.net/api`, `https://supercompany-dev...`) โ€” prod เธญเธฒเธเธขเธดเธเนเธ dev เนเธ”เธขเนเธกเนเธฃเธนเนเธ•เธฑเธงเธ–เนเธฒ env เนเธกเนเธ–เธนเธเธ•เธฑเนเธ
- **เนเธกเนเธกเธต consumer**: repo เน€เธเนเธ hub เธ—เธตเน publish event เน€เธเนเธฒ `STATEMENT_<SERVICE>` / `OUTSTANDING_<SERVICE>` / `<SERVICE>` / `WITHDRAW_<bank>` / `CHANGE_ACCOUNT_<bank>` เนเธซเน service เธเธฅเธฒเธขเธ—เธฒเธ (deposit/withdraw worker) เธกเธฒ consume โ€” เธเธนเธเธเธฑเธเธเธฑเธ naming convention เธเธญเธ tenant service
