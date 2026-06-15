## Card: SCHEDULE_SERVICE_BANK
identity: Go 1.18, github.com/go-co-op/gocron | _cmd/main.go → servicebank.StartServ() | no HTTP port (scheduler-only, no HTTP server) | Dockerfile present, .github/workflows/build.yaml present

provides:
  http:    none — this service exposes no HTTP endpoints
  consume: none — no RabbitMQ consumers; it only publishes
  cron:    every 3 minutes, SingletonMode, StartImmediately | app.go:53

consumes:
  http-out: env PAYMENT_API + "/api/v2/callback-status-current/withdraw" POST | controller/ctl-update-status-withdraw/ctl-update-status-withdraw.main.go:41
  http-out: env URL_ALL_API + "GetSwitchAccountByBankCode" GET (switch-account rules) | services/all-api/all-api.main.go:17,21
  http-out: env URL_SCB_APP + "/api/get-balance" POST, "/api/get-profile" GET (SCB app proxy) | controller/ctl-notification-account/noti-ban-bank.go:116-118
  http-out: env SCB_BUSINESS_API + "/api/scbanywhere/transfer-create" POST | controller/ctl-withdraw-transfer-scb-business/main.go:246
  http-out: env SCB_BUSINESS_API + "/api/scbanywhere/transfer-confirm" POST | controller/ctl-withdraw-transfer-scb-business/main.go:289
  http-out: env SCB_BUSINESS_API + "/api/scbanywhere/transfer-check" POST | controller/ctl-get-statement-transfer-scb-business/main.go:220
  http-out: hardcoded https://kbank.thezeus.online/kbank_app/get-balance POST (KBank balance) | helper/bank-tranfers.go:13,42
  http-out: hardcoded https://scb-app.lupin.host/api/get-balance POST (SCB balance) | helper/bank-tranfers.go:14,65
  http-out: hardcoded https://notify-api.line.me/api/notify POST (Line Notify alerts) | controller/ctl-notification-account/ctl-notification-account.main.go:457-458
  http-out: hardcoded https://api.telegram.org/bot{BOT_TOKEN}/sendMessage POST (Telegram alerts) | controller/ctl-notification-account/noti-ban-bank.go:173
  http-out: hardcoded https://monitor.thezeus.online/api/netbank/withdraw POST (netbank monitor) | services/firebase/upWithdraw.go:146-147
  publish:  queue name = {service} (per-tenant), exchange = "" (default), durable=true — sends bank_statement _id for credit re-trigger | controller/ctl-trigger-bank-account/ctl-trigger-bank-account.main.go:103-133
  publish:  queue name = CHANGE_ACCOUNT_{bank_number} (per-bank), exchange = "" (default) — sends withdraw change-account job | controller/ctl-change-account/ctl-change-account.main.go:309-340
  data:     DB_OFFICE (env) / collection database_config | R | db/conn.go:41, repository/office.repository.go:34
  data:     DB_OFFICE / collection bank_config | R/W (status, is_show, limit_amount.failed_app) | repository/office.repository.go:42,116,176,194
  data:     DB_OFFICE / collection bank_account_switch | R | repository/office.repository.go:100
  data:     DB_OFFICE / collection bank_deposit_summary | R/W | repository/office.repository.go:64-88
  data:     DB_OFFICE / collection working_log | W | repository/office.repository.go:125
  data:     per-tenant DB (from database_config.db_hostname) / collection bank_statement | R/W (status, status_summary_bank) | controller/ctl-trigger-bank-account/ctl-trigger-bank-account.main.go:42-45, repository/isdb.repository.go:42-78
  data:     per-tenant DB / collection bank_list | R/W (status, notification_date) | repository/isdb.repository.go:83-195
  data:     per-tenant DB / collection withdraw_statement | R/W (status, is_success, comment_timelime, scb_bussiness) | repository/isdb.repository.go:236-309
  data:     per-tenant DB / collection withdraw_statement_change_account | R/W | repository/isdb.repository.go:97-133
  data:     per-tenant DB / collection bank_statement_withdraw_whitelist | R/W | repository/isdb.repository.go:215-233
  data:     per-tenant DB / collection config_system | R (LINE tokens, Telegram chat IDs) | repository/isdb.repository.go:183
  external: Firebase Realtime DB https://withdraw-monitor-new.firebaseio.com PATCH/GET (withdraw status monitor) | services/firebase/upWithdraw.go:38,75,162
  external: Firebase Realtime DB https://go-care-679ce-default-rtdb.firebaseio.com/Withdraw/ PATCH (timeline running) | services/firebase/upWithdraw.go:282
  external: Line Notify API https://notify-api.line.me (bank limit alerts, whitelist alerts) | controller/ctl-notification-account/ctl-notification-account.main.go:457
  external: Telegram Bot API https://api.telegram.org (bank ban/limit alerts) | controller/ctl-notification-account/noti-ban-bank.go:173
  external: SCB App proxy https://scb-app.lupin.host (hardcoded — SCB balance, profile checks) | helper/bank-tranfers.go:14
  external: KBank App proxy https://kbank.thezeus.online (hardcoded — KBank balance checks) | helper/bank-tranfers.go:13
  external: Monitor service https://monitor.thezeus.online (netbank withdraw sync) | services/firebase/upWithdraw.go:146

observations:
  - HARDCODED Telegram bot token in source: "7546574244:AAHXLS_9dP1Wj3IZyApkUzB8TIr1talel6g" leaked in config/gbal.go:17 — CRITICAL secret exposure
  - HARDCODED external service URLs: https://kbank.thezeus.online and https://scb-app.lupin.host in helper/bank-tranfers.go:13-14 — cannot be changed without recompile; breaks multi-environment deployments
  - http.Client{} with NO Timeout in services/all-api/all-api.main.go:31 — goroutine leak risk if URL_ALL_API is unreachable
  - http.Client with CheckRedirect override but NO Timeout in ctl-notification-account/ctl-notification-account.main.go:459 — Line Notify calls can hang indefinitely
  - Money transfer operations (CreateWithdrawStatement, CreateQueueWithdraw) have hash-based idempotency for duplicate detection, but the idempotency window is time-based (hourly bucket) — a retry within the same minute will deduplicate, but re-runs in the next hour bucket will create duplicate records | controller/ctl-change-account/ctl-change-account.main.go:69-88,147-151
  - Swallowed error in ctl-notification-account: SendToLineNotify error return ignored when calling after tlg.SendMessage succeeds, UpdateDateNotification called on wrong condition (err != nil should be err == nil) | ctl-notification-account/ctl-notification-account.main.go:300-305 🟡
  - No DLQ configured on any RabbitMQ publish — if consumer is unavailable, messages in CHANGE_ACCOUNT_{bank_number} queues silently age without retry escalation | controller/ctl-change-account/ctl-change-account.main.go:308-338
  - SCB Business flow has a hardcoded 1-minute sleep between create and verify steps (app.go:127) — blocking a goroutine for every tenant per 3-minute cycle; scales poorly with many tenants
  - Multi-tenant architecture: connects to one "office" DB (DB_OFFICE) for config, then dynamically creates connections to each tenant's own DB (from database_config collection) — connection pool created at startup, held for process lifetime with no reconnect logic | db/conn.go:115-166
  - defer cancel() inside a for-loop in StartServ (ctl-trigger-bank-account) means cancel is deferred to function return not to loop iteration, potentially holding contexts too long | controller/ctl-trigger-bank-account/ctl-trigger-bank-account.main.go:32 🟡
  - gocron version 1.37.0 with SingletonMode prevents overlapping runs — safe for the 3-min cycle, but all 5 parallel goroutine groups (per tenant) run concurrently inside one scheduled job with no per-tenant singleton protection
  - Balance persistence uses local JSON file (balance_bank_config/balance_bank_all.json) — this is not safe in a multi-replica/pod deployment; data will diverge between pods | controller/ctl-change-account/ctl-change-account.main.go:382-395,491-527
  - go.mod uses github.com/streadway/amqp v1.0.0 — this library is deprecated (unmaintained); successor is rabbitmq/amqp091-go | go.mod:23

GROUP-PROPOSAL: payment-banking because this service exclusively manages bank accounts (bank_config, bank_statement, withdraw_statement), re-publishes credit events to per-tenant RabbitMQ queues, orchestrates money transfer (ChangeAccount/SCB Business withdraw), and monitors bank account health — all core banking/payment domain operations sharing the same MongoDB collections as the deposit/withdrawal services.
