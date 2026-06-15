## Card: kinglot-seamless

identity: Go 1.22, Iris v12 | kinglot.StartServer() or rabbitmqlotto.ConnectRabbit() depending on SERVICE_METHODS env | port :8000 | Dockerfile present, docker-compose.yml present

provides:
  http:
    GET  /metrics                                    | no auth (Prometheus) | server.go:63
    GET  /api/v1/                                    | method.Authentication (Bearer AUTHEN env) | server.go:68
    POST /api/v1/                                    | method.Authentication | server.go:69
    POST /api/v1/login                               | method.Authentication | server.go:70
    GET  /api/v1/balance                             | method.Authentication | server.go:71
    POST /api/v1/placebet                            | method.Authentication | server.go:72
    POST /api/v1/voidbet                             | method.Authentication | server.go:73
    POST /api/v1/settlebet                           | method.Authentication | server.go:74
    POST /api/v1/unsettle                            | method.Authentication | server.go:75
    GET  /agent/userinfo                             | IP allowlist CORS (CF-Connecting-IP) | server.go:85
    GET  /agent/deposit                              | IP allowlist | server.go:86
    GET  /agent/withdraw                             | IP allowlist | server.go:87
    GET  /agent/                                     | IP allowlist | server.go:89
    POST /agent/                                     | IP allowlist | server.go:90
    POST /agent/login                                | IP allowlist | server.go:91
    GET  /agent/balance                              | IP allowlist | server.go:92
    GET  /agent/add_member                           | IP allowlist | server.go:97
    GET  /agent/login (game login)                   | IP allowlist | server.go:98
    GET  /agent/games                                | IP allowlist | server.go:100
    GET  /agent/rounds                               | IP allowlist | server.go:101
    POST /agent/prebet                               | IP allowlist | server.go:102
    POST /agent/bet                                  | IP allowlist | server.go:103
    GET  /agent/history                              | IP allowlist | server.go:104
    GET  /agent/reward                               | IP allowlist | server.go:105
    GET  /agent/yiki                                 | IP allowlist | server.go:106
    POST /agent/yiki                                 | IP allowlist | server.go:107
    POST /agent/refund                               | IP allowlist | server.go:108
    POST /agent/callbacklotto                        | IP allowlist | server.go:109
    GET  /agent/seamless/transactionlotto            | IP allowlist | server.go:110
    GET  /agent/seamless/sumtransaction              | IP allowlist | server.go:111
    GET  /agent/seamless/useractivetransaction       | IP allowlist | server.go:112
    GET  /agent/gamesbyname                          | IP allowlist | server.go:113
    GET  /agent/seamless/transactionlottogroupbytype | IP allowlist | server.go:114
    GET  /agent/seamless/transactionlottoround       | IP allowlist | server.go:115
    GET  /agent/reward_bygroup                       | IP allowlist | server.go:116
    GET  /agent/group                                | IP allowlist | server.go:117
    POST /agent/check_bet_set                        | IP allowlist | server.go:119
    GET  /agent/report_winloss_summary               | IP allowlist | server.go:120
    POST /agent/userbalanceminimul                   | IP allowlist | server.go:121
    POST /agent/updatedomainagentlotto               | IP allowlist | server.go:123
    GET  /agent/seamless/sumtransactioncommission    | IP allowlist | server.go:125
    POST /agent/alert_ratepayment                    | IP allowlist | server.go:126
    POST /agent/placebet  (queue method)             | IP allowlist | server.go:129
    POST /agent/settlebet (queue method)             | IP allowlist | server.go:130
    POST /agent/unsettle  (queue method)             | IP allowlist | server.go:131
    POST /agent/voidbet   (queue method)             | IP allowlist | server.go:132
    POST /agent/noti_reward                          | IP allowlist | server.go:135
  consume:
    queue CALLBACK_LOTTO_{SERVICE} | PLACEBET → PlaceBetQueue (WITHDRAW), SETTLEBET → SettleBetQueue (DEPOSIT), UNSETTLE → UnSettleBetQueue (WITHDRAW), VOIDBET → VoidBetQueue (DEPOSIT) | rabbitmqlotto/worker.go:35, 103-172
    (activated when SERVICE_METHODS=queue)

consumes:
  http-out:
    Kinglot lotto provider API (TYPE env selects URL):
      TYPE=LENLEK or default  → https://api.u4win.com          | callback/init.go:22
      TYPE=LENLEK-COMMISSION or LOTTO-COMMISSION → https://api.lotto-commission.net | callback/init.go:28
      TYPE=LENLEK-STAGING     → https://api-227cdacd.nip.io    | callback/init.go:31
      TYPE=LOTTO-STAGING or LOTTO-COMMISSION-STAGING → https://api.228feca5.nip.io | callback/init.go:33
      TYPE=LOTTO-SCB          → https://api.lotto-thorv2.com   | callback/init.go:35
      Outbound call POST APIURL+"/api/manage/game/domain" (register callback domain) | callback/bet.go:1781
    APICALLBACK env + /v1/callback_lotto  (async POST after lotto round result) | callback/bet.go:955
    APIMINIGAME env + /api/get_summary_stat  (POST for commission summary)        | callback/bet.go:1576
    Line Notify (hardcoded token "jH1kTGoE1A3edAGwzc8TkqZc2fPQ3XYak3yNl2Ezj4z") for alert_ratepayment | callback/bet.go:1735
  data:
    MongoDB (env MONGODB_ENDPOINT, DB env MONGODB_DB_NAME) pool max 50 conn:
      collection "wallet_statement_lotto"  | R/W  | callback/bet.go:840, callback/db.go (implied by CreateTransactionWallet)
      collection "service_log"             | W/D  | callback/bet.go:697-712 (CreateServiceLog / DelServiceLog)
      collection "callback_min_amount"     | R/W  | callback/bet.go:972, 991-1003
      collection "member_account"          | R    | callback/wallet.go:120 (GetMember)
      collection "lotto_callback"          | R/W  | callback/bet.go:1116, 1276
      collection "member_affiliated"       | R    | callback/bet.go:1068
      collection "agent_credit"            | R/W  | callback/wallet.go:371-406
    RabbitMQ (env RABBITMQURL) | db/mq.go:18
  publish:
    RPC reply-to queue (d.ReplyTo) with bet result after PLACEBET/SETTLEBET/UNSETTLE/VOIDBET | rabbitmqlotto/worker.go:179-189

observations:
  - CRITICAL — NO IDEMPOTENCY KEY ENFORCED ON BET OPERATIONS: wallet_statement_lotto Hash field = "{TxID}{TypeDesp}{RecalTimes}" is written per document but there is NO unique index check before insert; duplicate PLACEBET callbacks from the provider will result in duplicate debits from the user's wallet | callback/bet.go:780
  - CRITICAL — WALLET HOLDER IS THIS SERVICE: kinglot-seamless owns the authoritative wallet balance in MongoDB "wallet_statement_lotto". The current balance is derived by reading the latest document (GetWallet queries by username/type_name/is_check), NOT from a single balance field — this is a balance reconstruction pattern that is prone to race conditions between concurrent callbacks | callback/db.go (GetWallet pattern consistent with go-agent-rocketwin rabbitmqlotto/func.go:33-45)
  - CRITICAL — DISTRIBUTED LOCK IS NOT ATOMIC: CreateServiceLog (insert service_log) is used as a mutex guard before balance operations with retry.Attempts(99). If the HTTP process crashes after insert but before DelServiceLog, the advisory lock is never released; future placebet/deposit/withdraw for that user will spin 99 retries then fail | callback/bet.go:692-713
  - CRITICAL — AUTHENTICATION IS OPTIONAL: method.Authentication only enforces Bearer token if AUTHEN env is non-empty; if AUTHEN is unset (common in dev/uat), ALL /api/v1 routes including /api/v1/placebet (direct wallet debit) are completely unauthenticated | method/middleware.go:12-13
  - CRITICAL — IP ALLOWLIST IS HARDCODED with 30+ GCP IPs including staging and third-party provider IPs | server.go:171-225; must redeploy to add/remove IPs
  - No DLQ configured on CALLBACK_LOTTO_{SERVICE} queue (durable=false, no dead-letter exchange) — RabbitMQ restart drops all queued bet callbacks | rabbitmqlotto/worker.go:37-43
  - agent_credit table maintains a separate running agent balance that is updated in two separate writes (UpdateAgentCredit then UpdateMemberCredit) with no transaction — a crash between the two writes leaves balances inconsistent | callback/wallet.go:399-405
  - LOTTO CALLBACK ASYNC FIRE-AND-FORGET: After storing LottoRoundResult, the service fires POST to APICALLBACK asynchronously in a goroutine; errors are only printed (fmt.Println), not retried or logged persistently | callback/bet.go:950-965
  - Hardcoded Line Notify token "jH1kTGoE1A3edAGwzc8TkqZc2fPQ3XYak3yNl2Ezj4z" in callback/bet.go:1735 — leaked credential
  - MongoDB connection pool maxPoolSize=50 (db/connect.go:43) which is higher than other services — may indicate this service handles high concurrency, but no monitoring of pool exhaustion
  - go.mod uses iris/v12 v12.1.8 — Iris 12.1 is an older release; current is 12.2.x
  - Queue consumer uses Reconnector that sets recon=false after first reconnect — only handles one reconnect cycle per goroutine invocation; subsequent RabbitMQ disconnects are not recovered | rabbitmqlotto/connect.go:82
  - OVERLAP NOTE: kinglot-seamless and go-agent-rocketwin BOTH implement the CALLBACK_LOTTO_{SERVICE} queue consumer with nearly identical code — they appear to be two generations of the same service; which one is active in production depends on SERVICE_METHODS env and deployment configuration. Running both concurrently on the same SERVICE would result in double-processing of bet callbacks.
