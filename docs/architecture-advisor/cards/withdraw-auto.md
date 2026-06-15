## Card: withdraw-auto

identity: Go 1.23 | _cmd/main.go → app.go → rabbitmqpub/manager.go | no HTTP port (consumer-only) | Dockerfile present, .github/workflows/build.yaml present

provides:
  consume (primary — QueueManager path, default when EVENTWITHDRAW unset):
    queue: WITHDRAW_{BankNumber} | BankWorker.ProcessMessage → bank dispatch switch | rabbitmqpub/worker.go:146
    queue: CHANGE_ACCOUNT_{BankNumber} | BankWorker.ProcessMessage (isChangeAccount=true) | rabbitmqpub/worker.go:147-148
    manual-ack=true, prefetch=1 | rabbitmqpub/manager.go:101-102
    ack AFTER ProcessMessage completes | rabbitmqpub/manager.go:218-219
  consume (legacy QRPAY path — EVENTWITHDRAW=QRPAY):
    queue: env QUEUE_NAME | rabbitmqpub/amqp.go legacy StartAMQP | rabbitmqpub/amqp.go:119
    AUTO-ACK=TRUE — message acked before bank call executes | rabbitmqpub/amqp.go:119
  consume (QRPAY_NEW path — EVENTWITHDRAW=QRPAY_NEW):
    queue: env QUEUE_NAME | rabbitmqpub/amqp-qrpay.go StartAMQPQrpay | rabbitmqpub/amqp-qrpay.go (auto-ack path)
  publish:
    exchange: STATEMENT (or STATEMENT_{SERVICE} if GROUP=NEWTOPUP) | fanout | type=WITHDRAW | service/report/publish.go:58
    queue: BANK_STATEMENT_WITHDRAW (direct, reply-to=BANK_STATEMENT_WITHDRAW) | service/report/publish.go:128
    queue: WITHDRAW_{BankNumber} (retry re-enqueue, durable) | service/rabbit/retry_withdraw.go:47
    hardcoded CloudAMQP URL used for CreateQueueWithdraw in non-prod | service/rabbit/retry_withdraw.go:15

consumes:
  http-out: callAuthBank — `PROXY_APP_BANK` env (default https://fast.lupin.host) | Timeout: 20s | service/callAuthBank/callAuthBank.go:51-52
  http-out: bank API calls — timeout varies per controller:
    gateway.func.go POST path: &http.Client{} no timeout | controller/bankgateway/gateway/gateway.func.go:37
    gateway.func.go GET path: Timeout: 30s | controller/bankgateway/gateway/gateway.func.go:187-188
    kbank/appapi: mixed — Timeout: 30s (main calls) | controller/kbank/appapi/appapi.func.go:119-120
    kbank/appapi: no timeout on some paths | controller/kbank/appapi/appapi.func.go:284, 486
    ktb/appapi: no timeout on some paths | controller/ktb/appapi/appapi.func.go:163, 233
    scb/appapi (_appapi.func.go): &http.Client{} no timeout on 9 of 11 calls | controller/scb/appapi/_appapi.func.go:31-535
    kkp/appapi: no timeout on primary calls | controller/kkp/appapi/appapi.func.go:83, 326
    gsb/appapi: Timeout: configurable or 30s | controller/gsb/appapi/func.go:229
    ttb/netbank: Timeout: 30s | controller/ttb/netbank/netbank.func.go:760-761
    tmn/tmnGateway: Timeout: 30s | controller/thirdgateway/main.go:218-219
    speedpay: Timeout: 30s | service/speedpay/main.go:140-141
    selenium HTTP client: Timeout: 300s | service/selenium/selenium.go:302
  http-out: Firebase status updates — &http.Client{} no timeout | service/firebase/upWithdraw.go:34,75,147,214,247
  http-out: k8s Helm action — &http.Client{} no timeout | service/k8sService/main.go:34
  http-out: office.repository POST — Timeout: 2min | repository/office.repository.go:44
  data: DATABASE env (MongoDB) | database_config collection (R) — multi-tenant routing | db/conn.go:263
  data: DATABASE env (MongoDB) | bank_config collection (R) — QueueManager refresh every 60s | rabbitmqpub/manager.go:120
  data: DATABASE env (MongoDB) | withdraw_statement collection (R/W) | repository/isdb.repository.go
  data: DATABASE env (MongoDB) | withdraw_statement_qrpay collection (R/W) | repository/payment.repository.go:60
  data: DATABASE env (MongoDB) | queue_payment collection (W) | repository/payment.repository.go:101
  data: DATABASE env (MongoDB) | service_qrpayment collection (R) | repository/payment.repository.go:28
  data: DATABASE env (MongoDB) | summary_deposit collection (R) | repository/payment.repository.go:92
  external: Firebase RTDB — bank worker status updates | service/firebase/upWithdraw.go
  external: Selenium WebDriver (remote or local ChromeDriver) | service/selenium/selenium.go
  external: Appium AppDriver (remote) | service/appdriver/appdriver.client.go
  external: Sentry DSN env — error capture | _cmd/main.go (sentry.Init)
  external: LINE Notify — transfer alerts | service/lineNotify/line-notify.go
  external: CloudAMQP (HARDCODED, non-prod) | rabbitmqpub/conn.go:34

observations:
  - MONEY RISK — AUTO-ACK BEFORE BANK CALL (legacy path): StartAMQP (QRPAY mode) sets autoAck=true at amqp.go:119; message is acked from broker BEFORE the bank withdrawal executes. If the process crashes mid-transfer, the message is permanently lost. | rabbitmqpub/amqp.go:119
  - MONEY RISK — NO DLQ ANYWHERE: No QueueDeclare in either consumer path passes x-dead-letter-exchange args. Failed/crashed messages have no recovery queue. | rabbitmqpub/manager.go:101 (nil args), rabbitmqpub/amqp.go:97 (nil args)
  - MONEY RISK — NO IDEMPOTENCY KEY: ProcessMessage generates no dedup key before dispatching to bank controller. Relies entirely on upstream not sending duplicate messages. | rabbitmqpub/worker.go:146-241
  - MONEY RISK — IN-MEMORY TRANSFER LOCK LOST ON RESTART: lockWithdraw map in _cmd/main.go:52 is per-process; crash or pod restart resets all per-bank transfer locks. | _cmd/main.go:52, rabbitmqpub/worker.go:241
  - HARDCODED CREDENTIAL: CloudAMQP URL with username+password hardcoded for MODE != production. | rabbitmqpub/conn.go:34, service/rabbit/retry_withdraw.go:15
  - TRUEWALLET DISABLED: truewallet.StartServ case commented out — bank code TRUEWALLET falls through to default (sentry.CaptureException). | rabbitmqpub/worker.go:210-211
  - MIXED TIMEOUTS ON BANK API CALLS: Most bank controllers (ttb, tmn, scb primary, speedpay) use 30s; several SCB, KTB, KKP, and gateway paths use &http.Client{} with no timeout — can hang indefinitely. | controller/scb/appapi/_appapi.func.go:31-478, controller/ktb/appapi/appapi.func.go:163,233, controller/kkp/appapi/appapi.func.go:83,326
  - REENQUEUE PATH: service/rabbit/retry_withdraw.go:47 can re-publish a message to WITHDRAW_{BankNumber} queue — combined with no idempotency this is a double-execution risk. | service/rabbit/retry_withdraw.go:47
  - DB CONNECT TIMEOUT: 5s constant for MongoDB connection (connectTimeout) — very tight under load. | db/conn.go:17
  - K8S HELM LIFECYCLE: QueueManager.Cleanup calls k8sService.CallActionHelm to uninstall legacy single-bank pods when upgrading to multi-queue mode. | rabbitmqpub/manager.go:481
  - FAILED BANK COOLDOWN: Banks that fail InitDrivers enter exponential cooldown: 5min × retryCount, max 30min. | rabbitmqpub/manager.go:324-331
  - QueueManager refreshes active bank configs from DB every 60s and reconciles workers (add new, remove stale) without service restart. | rabbitmqpub/manager.go:120
  - LOCAL CHROMEDRIVER: Restricted to BAY, UOB, KBANK only; other banks with empty UrlSeleniumWithdraw skip driver init silently. | rabbitmqpub/worker.go:88-93
