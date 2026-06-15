## Card: sub_statement

identity: Go 1.21, Gin | entrypoint `_cmd/main.go` → `server.go:StartServer()` | port=$PORT (env, Prometheus metrics exposed via Gin) | Dockerfile + docker-compose.yml present

provides:
  http:    (Gin server started but NO application routes registered — only Prometheus middleware) | server.go:37-47
  consume: exchange=STATEMENT (or STATEMENT_$SERVICE when GROUP==NEWTOPUP, fanout, durable) / queue=FRONTENDSUB (or FRONTENDSUB_$SERVICE, exclusive, auto-ack=true) | handlers: DEPOSIT→notification.Deposit+promotion.Deposit+covid.Deposit+mission.Deposit, WITHDRAW→notification.Withdraw+promotion.Withdraw+covid.Withdraw, RE-DEPOSIT→promotion.Deposit+covid.Deposit, MEMBER→member.Member | pub/amqp.go:51-118

consumes:
  http-out: FIREBASEURL env + /notify/deposit/$username.json PUT | notification/deposit.go:75
  http-out: LINENOTI env + /deposit POST (Line notification) | notification/deposit.go:99
  http-out: hardcoded https://console.sms-kub.com/api/campaigns POST (SMS ticket notification) | promotion/deposit.go:1206 (via CallAPIALL with 10s timeout)
  http-out: hardcoded http://sendsms.lupin.host/api/create_sms_log POST (log after SMS send, fire-and-forget via go axios.Post) | promotion/deposit.go:1226
  data:     $MONGODB_DB_NAME.deposit_statement | R (GetMany by datetime range in reCreateMemberStack) | helper/helper.go:138-154
  data:     $MONGODB_DB_NAME.member_stack | R/W (Upsert per user per day, UpdateOne stack_dynamic, GetAndUpdate for float ticket stacking) | promotion/deposit.go:174, promotion/withdraw.go:64, promotion/deposit.go:1305, 1365, 1408
  data:     $MONGODB_DB_NAME.stack_configs | R (GetMany status=1) | promotion/deposit.go:102
  data:     $MONGODB_DB_NAME.config_system | R (GetOne for checkin type, SMS config) | promotion/deposit.go:96, promotion/deposit.go:1154
  data:     $MONGODB_DB_NAME.self_stat | R (GetManyLimitOne by username+datetime range; used to check turnover before continuing checkin stack) | promotion/deposit.go:356
  data:     $MONGODB_DB_NAME.ticket_config | R/W (GetMany active configs, UpdateOne $inc total) | promotion/deposit.go:645, promotion/deposit.go:609
  data:     $MONGODB_DB_NAME.wallet_statement | R/W (CreateStatement ticket pay, AggregateStatement user/all receive per day) | promotion/deposit.go:710, 733, 1010
  data:     $MONGODB_DB_NAME.member_account | R (GetOne for phone number in SMS flow) | promotion/deposit.go:1170
  data:     Redis DB=$REDIS_DB (env) key=`config_system` (GET/SET; used by RedisConfigSystem helper) | helper/redisService.go:50-60, promotion/deposit.go:96

observations:
  - auto-ack=true on STATEMENT queue consumer — any panic in handler (notification, promotion, covid, mission chains) causes silent message loss with no DLQ | pub/amqp.go:83
  - Gin HTTP server is started but zero application routes are registered — the port is open but serves only Prometheus `/metrics`; the comment `// StartServer open port 5050` is inconsistent with $PORT env | server.go:33
  - RECREATE=TRUE env path calls reCreateMemberStack which does a full scan of deposit_statement by datetime range — no pagination, can OOM on large tenants | server.go:66-68, server.go:121-166
  - promotion.Deposit calls `helper.RedisConnect()` indirectly via `helper.RedisConfigSystem`; Redis client created per-message, not pooled — creates a new connection on every DEPOSIT event | promotion/deposit.go:96
  - self_stat collection is READ by sub_statement (turnover check) but there is no evidence sub_statement WRITES to self_stat — creates an implicit hard dependency on whichever service produces self_stat | promotion/deposit.go:356
  - http://sendsms.lupin.host/api/create_sms_log called with `go axios.Post(...)` — fire-and-forget with no timeout and no error handling; goroutine leak on slow/dead host | promotion/deposit.go:1226
  - Firebase PUT helper (PutFirebase / helper.go) previously logged raw []byte body as decimal-encoded line; fixed in repo but still uses `go-axios` with no explicit timeout context | helper/helper.go (PutFirebase)
  - RabbitMQ reconnection (pub/connection.go:65-79) calls `StartAMQP` which blocks on `<-forever` — after reconnect the goroutine can only run StartAMQP once because `recon = false` is set immediately, so a second disconnection is not handled | pub/connection.go:65-79
  - DynamicStacks (promotion/deposit.go:212) swallows `errCon` from AggregateStatement on member_stack without returning — continues with empty memberStack, potentially writing wrong stack value | promotion/deposit.go:252-256
  - `_config/db.go:77` uses `_ = client.Connect(ctx)` — error from Connect is silently discarded; only Ping failure surfaces the problem | _config/db.go:77
  - connectTimeout = 5 seconds for MongoDB in _config/db.go:17 — very short for initial cold-start under load
  - CallAPIALL (SMSKUB) has a 10-second timeout at promotion/deposit.go:1264; this is good, but it is called synchronously inside the AMQP consumer goroutine — a slow SMS gateway blocks processing of all subsequent messages
  - go.mod requires go 1.21 and mongo-driver v1.13.1 — significantly newer than go_report (go 1.15); the two services have divergent dependency trees for the same underlying collections
