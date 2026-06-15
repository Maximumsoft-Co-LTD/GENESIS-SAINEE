## Card: que_payment

identity: Go 1.25 | Worker process (no HTTP server) | no port | Dockerfile present, docker-compose.yml present

provides:
  http: (none — pure worker/consumer process)

  consume:
    queue "QUE_PAYMENT_<PAYMENT_NAME>" (env) | durable=false, no DLQ configured | handlers: QueTypeSwitchV2 dispatching to DEPOSIT / WITHDRAW / PAYOUT / REFUND / REPORT_DATA_CURRENT / REPORT_DATA_OLD / MANUAL_CONFIRM_WITHDRAW / REFUND-WITHDRAW-OFFICE | rabbitmqpub/amqp.go:61–68, 310–341
    queue "QUE_PAYMENT_<SYSTEM_CODE>" (legacy v1, StartAMQP) | durable=false | rabbitmqpub/amqp.go:232–239

  cron:
    every 20s: StartServV2 polls MongoDB que_payment for pending queue items | quepub/cornjob.go:41
    every 30min: AutomationRetryCallbackStatus — retries failed deposit/withdraw callbacks (TYPE=QUE_CRONJOB + PAYMENT_NAME=AUTOMATION_RETRY_CALLBACK_STATUS) | app.go:128–136, cronjob/automation_retry_callback_status.go:34
    periodic: RecoverBankSummary (TYPE=QUE_CRONJOB + PAYMENT_NAME=BANK_SUMMARY_RECOVER) | app.go:118–126
    every 20s: ServRetryOrder polls and retries failed orders | app.go:154–165

consumes:
  http-out: urlCallback (from DB config) + /api/Payment/PaymentCreateStatement/:service (deposit callback to office) | services/callback/main.go
  http-out: urlCallback + /api/WithdrawCallback (withdraw callback to office) | services/callback/main.go
  http-out: 40+ payment provider APIs (per-provider services packages) — see observations for timeout issues
  publish:  reply-to queue (d.ReplyTo, anonymous) with RPC response | rabbitmqpub/amqp.go:201–211
  data:     DB_DBNAME (env) + collection qr_payment      | R/W | repository (via PaymentRepository.GetDepositStatement, UpdateIsSuccessDepositStatementV2)
  data:     DB_DBNAME (env) + collection withdraw_statement | R/W | repository
  data:     DB_DBNAME (env) + collection que_payment     | R/W | repository (GetListQueAllV2, UpdateQue)
  data:     Redis (REDIS_URL / REDIS_DB env) — used for idempotency locks in AutomationRetryCallbackStatus (KEY_LOCK_DEPOSIT_STATEMENT, KEY_LOCK_WITHDRAW_STATEMENT, 10min TTL) | cronjob/automation_retry_callback_status.go:27–30
  external: Sentry (SENTRY_DSN env) for error tracking | _cmd/main.go:33–37
  external: Telegram Bot (TELEGRAM_ERROR_BOT_TOKEN / TELEGRAM_ERROR_CHAT_ID env) | app.go:38–50

observations:
  - CRITICAL: Queue declared with durable=false — broker restart loses all in-flight payment jobs | rabbitmqpub/amqp.go:63 (second arg `false`)
  - CRITICAL: No Dead Letter Queue configured on QUE_PAYMENT_* — failed/rejected messages are either silently dropped (Nack requeue=false) or re-queued indefinitely (Nack requeue=true) with no poison-message handling | rabbitmqpub/amqp.go:122 (Nack false,false) and :210 (Nack false,true)
  - CRITICAL: Hardcoded MongoDB credentials in docker-compose.yml: "mongodb://useradmin:Aa112233@3.1.8.89:27017/abaoffice_db" — production-looking IP with plaintext password | docker-compose.yml:11
  - CRITICAL: Multiple payment provider HTTP clients have no timeout — askmepay/func.go:34 (&http.Client{}), blackdog/main.go:45, gbetpay/func.go:36, jaijaipay/func.go:50, mtmpay/func.go:45, mtpay/main.go:32, papayapay/func.go:33, peer2pay/func.go:45, worldpay/func.go:55, zappay/func.go:48, zpay/main.go:29 — a hung bank API will block a worker goroutine indefinitely and starve the queue
  - HIGH: Idempotency on deposit is a is_success==1 check in the handler (controllers/deposit.go:78,117) with no distributed lock around the check-and-update — concurrent duplicate queue deliveries can both pass the guard simultaneously
  - HIGH: Reply publish failure falls back to Nack-with-requeue (rabbitmqpub/amqp.go:210) but original message is already processed — requeue causes double-processing of money operations
  - HIGH: TYPE env var branches into QUE_RABBIT vs QUE_CRONJOB — if TYPE is unset, neither branch runs and the service starts silently doing nothing | app.go:95–166
  - MEDIUM: Legacy StartAMQP (v1) still present with auto-Ack on error path — errors in QueTypeSwitch are logged but message is still acked, silently losing failed payment jobs | rabbitmqpub/amqp.go:262–279 🟡
  - MEDIUM: StartServV2 cronjob and RabbitMQ consumer run in same process when TYPE=QUE_RABBIT — StartServV2 goroutine is always launched alongside rabbit consumer, may double-process items | app.go:148–165
  - MEDIUM: Sentry DSN is required at startup — if SENTRY_DSN is empty or invalid, `log.Fatalf` kills the process | _cmd/main.go:37
  - MEDIUM: Redis connection string and DB number logged in plain text at startup ("Connected to Redis at %s (DB: %s)") | app.go:86
  - INFO: Shutdown manager pattern used correctly — OTel, Telegram notifier, MongoDB, Redis, RabbitMQ all register shutdown hooks | app.go:34,49,71,88,101
  - INFO: OTel trace propagation from AMQP headers correctly implemented (Extract before span, Inject on reply) | rabbitmqpub/amqp.go:133,200
