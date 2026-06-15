## Card: 3rd-payment

identity: Go 1.25 | Gin HTTP server | port 8081 (app.go:150) | Dockerfile present, docker-compose.yml present

provides:
  http:
    POST /api/v2/payin                                  | no auth middleware (open)                                      | routes/main.go:53
    POST /api/v2/callback-payin/:payment_code           | IPWhitelistMiddleware (DB lookup, bypass-able via env CHECK_IPWHITELIST != "TRUE") | routes/main.go:55
    POST /api/v2/callback-payin/:payment_code/callback/qrStatus | IPWhitelistMiddleware                              | routes/main.go:56
    POST /api/v2/callback-one-link/:payment_name/:payment_code  | IPWhitelistMiddleware                              | routes/main.go:57–59
    POST /api/v2/create-withdraw                        | no auth middleware                                             | routes/main.go:60
    POST /api/v2/confirm-withdraw                       | no auth middleware                                             | routes/main.go:61
    POST /api/v2/callback-payout/:payment_code          | IPWhitelistMiddleware                                          | routes/main.go:64
    POST /api/v2/callback-payout/autopeer/:payment_code | IPWhitelistMiddleware                                          | routes/main.go:65
    POST /api/v2/callback-payout/umpay/:payment_code    | IPWhitelistMiddleware                                          | routes/main.go:66
    POST /api/v2/adjust-balance                         | no auth middleware                                             | routes/main.go:69
    POST /api/v2/confirm-transaction-payin              | no auth middleware                                             | routes/main.go:74
    POST /api/v2/re-create-statement/:service/:paymentCode/:orderNo | no auth middleware                             | routes/main.go:46
    POST /api/v2/create-config-payment                  | no auth middleware                                             | routes/main.go:37
    POST /api/v2/update-config-payment                  | no auth middleware                                             | routes/main.go:38
    POST /api/v2/submit-withdraw                        | no auth middleware                                             | routes/main.go:103
    POST /api/v2/retry-payment-sendffice                | no auth middleware                                             | routes/main.go:132
    POST /bank-gateway/verify-transfer                  | no auth middleware                                             | routes/bank-transfer-gateway.go:49
    POST /bank-gateway/confirm-transfer                 | no auth middleware                                             | routes/bank-transfer-gateway.go:50
    POST /bank-gateway/verify-and-confirm-transfer      | no auth middleware                                             | routes/bank-transfer-gateway.go:51
    POST /bank-gateway/call-by-req                      | no auth middleware (open HTTP proxy endpoint)                  | routes/bank-transfer-gateway.go:55
    ANY  /call-pm/:paymentCode/*byPath                  | no auth middleware (wildcard proxy to payment providers)        | routes/call-by-payment.go:39

  consume: (none — no RabbitMQ consumer; publishes only)

  cron: none active (StartCronjob() only called via commented-out line in _cmd/main.go:6)

consumes:
  http-out: RabbitmqURL (from mongo service_payment config) + queue RPC reply-to pattern | service/rabbit/que-payment.go:56
  http-out: urlCallback (dynamic from DB config) + /api/Payment/PaymentCreateStatement/:service | service/callback/main.go:106
  http-out: urlCallback + /api/WithdrawCallback                                           | service/callback/main.go:30
  http-out: urlCallback + /api/Payment/CallBackWithdrawAmount/:service                   | service/callback/main.go:66
  http-out: urlCallback + /api/:service/payment-update-balance                           | service/callback/main.go:191
  http-out: Telegram API https://api.telegram.org/bot.../setWebhook (BOT_TELEGRAM_TOKEN)  | app.go:207
  publish:  queue "QUE_PAYMENT_<PAYMENT_NAME>" or "QUE_MERCHANT_<MERCHANT_CODE>"        | service/rabbit/que-payment.go:140–156
  data:     DB_DBNAME (env) + collection qr_payment      | R/W | repository/main.go:60,120
  data:     DB_DBNAME (env) + collection withdraw_statement | R/W | repository/main.go:152,313
  data:     DB_DBNAME (env) + collection que_payment     | W   | repository/main.go:32
  data:     DB_DBNAME (env) + collection adjust_statement | W   | repository/main.go:231
  data:     Redis (REDIS_URL env) via thorlock distributed lock (ThorRedis) | app.go:85
  external: ~50 third-party payment providers (askmepay, bigpay, peer2pay, hengpay, speedpay etc.) — HTTP from individual controller packages
  external: Telegram Bot API (BOT_TELEGRAM_TOKEN / TELEGRAM_ERROR_BOT_TOKEN env)         | app.go:41

observations:
  - CRITICAL: Callback signature verification is payment-provider-specific and inconsistently applied — umpay verifies HMAC (controller/callback.go:1178–1186), anypayth verifies webhook sig (controller/callback.go:1645–1657), k2pay verifies (1952–1965), but many providers (speedpay, peer2pay, jaijaipay) use only IP whitelist as the sole guard which can be disabled by CHECK_IPWHITELIST != "TRUE" | middleware/ipwhitelist.go:23–27
  - CRITICAL: IPWhitelist guard is globally bypassable at runtime via CHECK_IPWHITELIST env var — setting it to anything except "TRUE" makes all callback endpoints unauthenticated | middleware/ipwhitelist.go:23–24
  - CRITICAL: No auth middleware on /api/v2/adjust-balance, /api/v2/create-config-payment, /api/v2/update-config-payment, /api/v2/submit-withdraw, /api/v2/confirm-transaction-payin — money-affecting endpoints exposed without authentication | routes/main.go:37–38,69,74,103
  - CRITICAL: /bank-gateway/call-by-req is an open unauthenticated HTTP proxy endpoint that accepts arbitrary domain+method+header+body from caller, enabling SSRF | routes/bank-transfer-gateway.go:55–74
  - CRITICAL: /call-pm/:paymentCode/*byPath wildcard route uses no auth middleware — acts as open proxy to all downstream payment providers | routes/call-by-payment.go:39
  - CRITICAL: Hardcoded MongoDB credentials in docker-compose.yml: "mongodb+srv://root:Zxcvasdf789@test02..." | docker-compose.yml (env DB_ENDPOINT)
  - CRITICAL: No auth on /api/v2/create-indo-config-payment, /api/v2/update-indo-config-payment — indo payment config writable without auth | routes/main.go:39–40
  - HIGH: Multiple payment provider HTTP clients use &http.Client{} with no Timeout — askmepay/func.go:38, danarapay/func.go:37, gbetpay/func.go:36, jaijaipay/func.go:50, mtmpay/func.go:45, papayapay/func.go:33, peer2pay/func.go:42, worldpay/func.go:54, zappay/func.go:47, banktransfer/bk55/main.go:19, helper/callApi.go:19 — goroutine leak risk under slow/hung bank APIs
  - HIGH: RabbitMQ connection is created per-request in CallRabbitDepositAndWithdraw (new amqp.Dial each call) — no connection pooling; under load this exhausts RabbitMQ connection limits | service/rabbit/que-payment.go:56
  - HIGH: Reply-queue consumer uses auto-ack=true (ch.Consume ...true...) — if process crashes after dequeue but before processing, message is silently lost | service/rabbit/que-payment.go:98
  - MEDIUM: 40+ payment provider packages duplicated almost identically between 3rd-payment/controller/ and que_payment/services/ — identical logic maintained in two places, drift already visible (askmepay absent from que_payment services list)
  - MEDIUM: Telegram webhook endpoint /api/v2/bot/webhook registered but route is via rv2Bot group with no auth — relies only on Telegram's secret_token header validation | routes/main.go:159
  - MEDIUM: StartCronjob() is dead code — commented out in _cmd/main.go:6; however CronjobStatement is also exposed as HTTP POST /api/v2/create-member-data without auth | routes/main.go:106
  - INFO: Thorlock distributed Redis lock used for deposit/withdraw idempotency — correct approach, but callback.go:1078 (is_success==1 check) is race-prone before lock acquired 🟡
  - INFO: MongoDB connection pool options commented out (min/maxPoolSize) — using default Go driver pool under unknown load | db/connect.db.go:49
