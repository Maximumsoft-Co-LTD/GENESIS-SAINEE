## Card: go-agent-rocketwin

identity: Go 1.24, Gin | agentnewtopup.StartServer() or rabbitmqlotto.ConnectRabbit() depending on SERVICE_METHODS env | port :8000 | Dockerfile present, docker-compose.yml present, Dockerfile-g present

provides:
  http:
    GET  /agent/                                     | IP allowlist CORS (CF-Connecting-IP) | route.go:24
    GET  /agent/userinfo                             | IP allowlist | route.go:27
    GET  /agent/deposit                              | IP allowlist | route.go:28
    GET  /agent/withdraw                             | IP allowlist | route.go:29
    GET  /agent/add_member                           | IP allowlist | route.go:30
    POST /agent/del_member                           | IP allowlist | route.go:31
    POST /agent/usergamedelete                       | IP allowlist | route.go:32
    POST /agent/usersessiondelete                    | IP allowlist | route.go:33
    POST /agent/updatetypenotplayuser                | IP allowlist | route.go:34
    GET  /agent/brandgame                            | IP allowlist | route.go:37
    GET  /agent/getgame                              | IP allowlist | route.go:38
    GET  /agent/playgame                             | IP allowlist | route.go:39
    GET  /agent/searchgame                           | IP allowlist | route.go:40
    GET  /agent/listgame                             | IP allowlist | route.go:41
    POST /agent/getdetail                            | IP allowlist | route.go:43
    GET  /agent/wallet                               | IP allowlist | route.go:45
    GET  /agent/transactionType                      | IP allowlist | route.go:46
    GET  /agent/createToken                          | IP allowlist | route.go:47
    GET  /agent/transfer/transaction                 | IP allowlist | route.go:48
    GET  /agent/outstanding/:username                | IP allowlist | route.go:51
    POST /agent/outstanding                          | IP allowlist | route.go:52
    GET  /agent/getratelimit                         | IP allowlist | route.go:57
    GET  /agent/getbuffergame                        | IP allowlist | route.go:58
    GET  /agent/checkstatus                          | IP allowlist | route.go:59
    GET  /agent/checkbalanceagent                    | IP allowlist | route.go:60
    POST /agent/userbalanceminimul                   | IP allowlist | route.go:63
    POST /agent/transactionsbo                       | IP allowlist | route.go:64
    GET  /agent/reporttransactionsport               | IP allowlist (also VPN bypass path) | route.go:67
    POST /agent/reporttransactionsport               | IP allowlist | route.go:68
    GET  /agent/getdomainagent                       | IP allowlist | route.go:70
    GET  /agent/updatedomainagent                    | IP allowlist | route.go:71
    GET  /agent/uploadfilegame                       | IP allowlist | route.go:72
    GET  /agent/preset-pg                            | IP allowlist | route.go:73
    GET  /agent/seamless/transaction                 | IP allowlist (also VPN bypass path) | route.go:79
    GET  /agent/seamless/sumtransaction              | IP allowlist (also VPN bypass path) | route.go:80
    GET  /agent/seamless/useractivetransaction       | IP allowlist | route.go:81
    GET  /agent/seamless/transactiongroupbytype      | IP allowlist | route.go:82
    GET  /agent/seamless/sumtransactioncommission    | IP allowlist (also VPN bypass: "88.216.56.60") | route.go:85
    POST /api/userbalanceminimul                     | IP allowlist | route.go:90
    POST /api/transactionsbo                         | IP allowlist | route.go:91
    GET  /agent/login                                | IP allowlist | route.go:101
    GET  /agent/games                                | IP allowlist | route.go:104
    GET  /agent/rounds                               | IP allowlist | route.go:105
    GET  /agent/yiki                                 | IP allowlist | route.go:106
    POST /agent/yiki                                 | IP allowlist | route.go:107
    GET  /agent/group                                | IP allowlist | route.go:108
    POST /agent/prebet                               | IP allowlist | route.go:109
    GET  /agent/history                              | IP allowlist | route.go:111
    POST /agent/refund                               | IP allowlist | route.go:112
    GET  /agent/reward                               | IP allowlist | route.go:113
    GET  /agent/seamless/transactionlottogroupbytype | IP allowlist | route.go:119
    GET  /agent/seamless/transactionlottoround       | IP allowlist | route.go:120
    GET  /agent/reward_bygroup                       | IP allowlist | route.go:121
    POST /agent/check_bet_set                        | IP allowlist | route.go:122
    GET  /agent/report_winloss_summary               | IP allowlist | route.go:123
    GET  /agent/balance                              | IP allowlist | route.go:129
    POST /agent/updatedomainagentlotto               | IP allowlist | route.go:132
    POST /agent/access_lotto_seamless                | IP allowlist | route.go:138
    POST /agent/bet                  (kinglot v2)    | IP allowlist | route.go:143
    POST /agent/placebet             (kinglot v2)    | IP allowlist | route.go:144
    POST /agent/settlebet            (kinglot v2)    | IP allowlist | route.go:145
    POST /agent/voidbet              (kinglot v2)    | IP allowlist | route.go:146
    POST /agent/unsettle             (kinglot v2)    | IP allowlist | route.go:147
    POST /agent/settlebetslow        (kinglot v2)    | IP allowlist (also VPN bypass path) | route.go:150
    POST /agent/callbacklotto        (kinglot v2)    | IP allowlist | route.go:153
    GET  /v2/agent/balance                           | IP allowlist | route.go:159
    POST /v2/agent/bet                               | IP allowlist | route.go:160
    POST /v2/agent/placebet                          | IP allowlist | route.go:161
    POST /v2/agent/settlebet                         | IP allowlist | route.go:162
    POST /v2/agent/voidbet                           | IP allowlist | route.go:163
    POST /v2/agent/unsettle                          | IP allowlist | route.go:164
    POST /v2/agent/settlebetslow                     | IP allowlist | route.go:167
    POST /v2/agent/callbacklotto                     | IP allowlist | route.go:170
  consume:
    queue CALLBACK_LOTTO_{SERVICE}  | PLACEBET → placeBetQueue (WITHDRAW wallet), SETTLEBET → settleBetQueue (DEPOSIT wallet), UNSETTLE → unSettleBetQueue (WITHDRAW wallet), VOIDBET → voidBetQueue (DEPOSIT wallet) | service/rabbitmqlotto/worker.go:55, 95-165
    (activated when SERVICE_METHODS=queue, else HTTP mode is used)

consumes:
  http-out:
    APIKEY=APIKEY env  | Rocketwin/Mahagame/Demo provider
      GET  APIURL/v1/wallet/balance                  | service/rocketwin_service/getwallet.go:36-47
      POST APIURL/v1/wallet/transaction              | service/rocketwin_service/createTransaction.go:29
      POST APIURL/api (various game routes)          | service/rocketwin_service/init.go:11-18
      GET  APIURL/v1/wallet/balance (queue mode)     | service/rabbitmqlotto/callrocketwin.go:56
      POST APIURL/v1/wallet/transaction (queue)      | service/rabbitmqlotto/callrocketwin.go:134
      POST APIURL/v1/wallet/gen_token_statement      | service/rabbitmqlotto/callrocketwin.go:154
      GET  APIURL/v2/external/wallet/balance         | service/rabbitmqlotto/callrocketwin.go:111
    Line Notify (hardcoded token) | https://notify-api.line.me/api/notify | service/rocketwinService.go:271-283
    Line Notify (hardcoded token) low-credit alert   | service/rabbitmqlotto/callrocketwin.go:196
    APIMINIGAME env + /api/get_summary_stat          | kinglot/service called from lotto seamless commission route (not in rocketwin route.go — 🟡 appears to be in kinglot controller, not rocketwin)
    Clickhouse (env LOTTOKEY gates this)             | _config/db_clickhouse.go | server.go:59
  data:
    MongoDB (env MONGODB_ENDPOINT, DB env MONGODB_DB_NAME)
      collection "wallet_statement_lotto"  | R/W | service/rabbitmqlotto/func.go:297, 273, 227
      collection "wallet_statement"        | R   | service/rabbitmqlotto/func.go:38
      collection "service_log"             | W/D | service/rabbitmqlotto/func.go:495-509
      collection "callback_min_amount"     | R/W | service/rabbitmqlotto/func.go:354-413
      collection "outstanding" (🟡 implied by route, not confirmed in read files)
    Redis (env REDISURL via service/redisService.go 🟡 file not read)
  publish:
    replies to RPC reply-to queue (d.ReplyTo) after PLACEBET/SETTLEBET/UNSETTLE/VOIDBET processing | service/rabbitmqlotto/worker.go:149-160

observations:
  - CRITICAL: Hardcoded Line Notify token "pKahNdB8I8ifXmaeekmD66EbvXSgKMF5hBYUt5bgiNa" in service/rocketwinService.go:283 — leaked token, any caller can post Line notifications to this group
  - CRITICAL: Hardcoded Line Notify token "EO8qk0z0chWl8yoAUMMIACLrlzu7ebHzxk3f67hEljz" (credit alert group) in service/rabbitmqlotto/callrocketwin.go:195 — same risk
  - CRITICAL: No idempotency key on wallet debit/credit for PLACEBET/SETTLEBET/VOIDBET/UNSETTLE when called from queue — walletStatement.Hash = "{TxID}{TypeDesp}{RecalTimes}" is stored in the document but there is NO unique index enforcement shown; duplicate queue messages can double-debit/double-credit | service/rabbitmqlotto/func.go:185
  - CRITICAL: service_log distributed locking pattern (CreateServiceLog then defer DelServiceLog) is a naive advisory lock — no TTL, no atomic CAS. If the process crashes after CreateServiceLog but before DelServiceLog, the lock is never released and subsequent operations for that username will retry indefinitely (retry.Attempts(99)) | service/rabbitmqlotto/func.go:83-95
  - MISSING HTTP timeout: HttpHandleAgent uses http.Client{} with no Timeout | service/rocketwinService.go:85 — used for game-provider calls
  - IP-based CORS allowlist is hardcoded as Go map literals with per-tenant GCP IPs — requires code redeploy to add new tenants | server.go:147-198
  - VPN IP "88.216.56.60" hardcoded as a special bypass for /agent/settlebetslow and commission report routes | server.go:221
  - Queue consumer declares queue as durable=false — queue is lost on RabbitMQ restart; in-flight bet settlements would be lost with no DLQ configured | service/rabbitmqlotto/worker.go:57
  - MQReconnector sets recon=false after first reconnect — only reconnects once, then the goroutine exits; a second RabbitMQ disconnect is not handled | _config/mq.go:51
  - go-agent-rocketwin is a combined rocketwin (slot/game) + kinglot (lotto) agent in one binary — dual routing responsibilities make versioning and scaling complex
  - Clickhouse dependency (LOTTOKEY env) is optional; its absence is silently skipped but ScanQueueSettleBet workers are not started, which means lotto settle-queue processing is silently disabled without LOTTOKEY | server.go:58-73
