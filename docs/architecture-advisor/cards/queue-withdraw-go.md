## Card: queue-withdraw-go

identity: Go 1.22.4 | _cmd/main.go → queuework/startQueue.go → pub/amqp.go | no HTTP port (consumer-only) | Dockerfile present (implied by CI), no explicit CI file found in repo root

provides:
  consume:
    queue: {QUEUE_NAME}_{SERVICE} (env-composed) | durable=false | auto-ack=false | prefetch=1 | pub/amqp.go:46-48,64,82
    dispatch by d.Type:
      ORIGINWITHDRAW → origin.Withdraw(resource, username, amount, bankNumber, type, bonusID) | pub/amqp.go:121-123
      WALLETWITHDRAW → wallet.WithdrawWalletV2(resource, bobyReceive) + Redis wallet_withdraw_{username} del | pub/amqp.go:124-144
      EVENTACTION → event.EventWallet(resource, bobyReceive) | pub/amqp.go:145-147
      WHEELEVENT → wheel.CallBackWheel(resource, bobyReceive) | pub/amqp.go:148-150
    processing timeout: context.WithTimeout 60s per message | pub/amqp.go:101
    ack AFTER done (manual ack on success) | pub/amqp.go:195
    nack requeue=false on 60s timeout — message DROPPED (no DLQ) | pub/amqp.go:191
  publish (RPC reply):
    queue: d.ReplyTo (caller-supplied reply-to queue) | CorrelationId echoed | Expiration=60000ms | pub/amqp.go:160-170
  publish (statement fan-out):
    exchange: STATEMENT_{SERVICE} (fanout) | routing key: DYNAMICQUEUE_{SERVICE} | helper/wallet.go:92-105
  publish (dynamic queue direct):
    queue: DYNAMICQUEUE_{SERVICE} | type from covid.Type | Expiration=60000ms | origin/withdraw.go:1237-1250

consumes:
  http-out: credit/balance check — SERVICE_API env (via helper.FilterAPI) | Timeout: 10s (axios InstanceConfig) | helper/agent.go:384-385
  http-out: DeleteCredit — SERVICE_API env | helper.CallAPI | &http.Client{} NO TIMEOUT | helper/helper.go:161
  http-out: AgentAmount (balance pre-check) — SERVICE_API env | helper.CallAPI | &http.Client{} NO TIMEOUT | helper/helper.go:161
  http-out: withdraw-auto trigger POST to {OFFICE_API}/api/auto/createQueueWithdraw/{SERVICE} — helper.HttpHandle | &http.Client{} NO TIMEOUT | helper/helper.go:213, origin/withdraw.go:1195-1196
  http-out: monitoring ping GET https://monitor.thezeus.online/api/webQueue/{SERVICE}/{type} — hardcoded URL, goroutine fire-and-forget | pub/amqp.go:120
  http-out: Telegram alert — hardcoded bot token | helper/helper.go:384
  data: MONGODB_DB_NAME env (MongoDB) | withdraw_statement collection (R/W) | origin/withdraw.go:1007,1031
  data: MONGODB_DB_NAME env (MongoDB) | deposit_statement collection (R/W) | origin/withdraw.go:975,996,1018
  data: MONGODB_DB_NAME env (MongoDB) | member_account collection (R/W) | origin/withdraw.go:949,1044
  data: MONGODB_DB_NAME env (MongoDB) | wallet_statement collection (R/W) | wallet/withdraw.go:877,888,900
  data: MONGODB_DB_NAME env (MongoDB) | event_statement collection (W) | wallet/withdraw.go:937
  data: MONGODB_DB_NAME env (MongoDB) | member_affiliated collection (R) | helper/helper.go:128
  data: MONGODB_DB_NAME env (MongoDB) | member_log collection (W) | helper/helper.go:201
  data: REDIS_URL + REDIS_DB env (Redis) | key: wallet_withdraw_{username} (R/W/Del) — upstream dedup guard for WALLETWITHDRAW | pub/amqp.go:135-144, helper/redis.go
  external: RabbitMQ — RABBITMQURL env | streadway/amqp v1.0.0 | pub/connection.go
  external: Jaeger tracing — JAEGER_AGENT_HOST env | jaeger-client-go | (startQueue.go setup)

observations:
  - MONEY RISK — MESSAGE DROPPED ON 60s TIMEOUT WITH WORK STILL RUNNING: ctx (pub/amqp.go:101) is never passed into origin.Withdraw or wallet.WithdrawWalletV2 (called at lines 123,131). When the 60s timeout fires, d.Nack(false,false) drops the message from the queue (requeue=false, no DLQ), but the spawned goroutine continues executing the debit. If debit completes after nack, credit is taken with no withdraw_statement committed successfully — classic partial-execution money loss. | pub/amqp.go:101,123,131,191
  - MONEY RISK — NO DLQ: Queue declared with nil args (no x-dead-letter-exchange). Nacked and expired messages are permanently lost. | pub/amqp.go:48,191
  - MONEY RISK — QUEUE NOT DURABLE: durable=false — all pending messages lost on RabbitMQ restart. | pub/amqp.go:47
  - MONEY RISK — NON-ATOMIC WALLET DEBIT (5 SEQUENTIAL WRITES, NO MONGO TRANSACTION): wallet.WithdrawWalletV2 performs: resetWalletStatement, insertWalletStatement WITHDRAW, insertWalletStatement DEPOSIT, insertWithdrawStatement, UpdateManyStatement status=1 — no session/transaction wrapping. Partial failure leaves inconsistent balances. | wallet/withdraw.go:523,562,596,699,736
  - MONEY RISK — SHA1 DEDUP KEY WITH MINUTE GRANULARITY: wallet_statement hash uses fmt.Sprintf("%f%s%s%sWITHDRAW", afterCredit, typeWallet, username, helper.TimeFormat("minute")) — two requests for the same user in the same minute produce the same hash. No unique index confirmed from code; collision risk is real. | wallet/withdraw.go:540
  - MONEY RISK — WITHDRAW_STATEMENT INSERTED WITH FAILURE STATUS BEFORE CREDIT DELETION: origin.Withdraw inserts withdraw_statement with status=4 ("ตัดเครดิตไม่สำเร็จ") at line 462, then calls DeleteCredit at line 478. If DeleteCredit succeeds but the subsequent status update (line 497) fails, the statement shows failure but credit was taken. | origin/withdraw.go:462,478,497
  - MONEY RISK — NO TIMEOUT ON DELETECREDIT / AGENTAMOUNT HTTP CALLS: helper.CallAPI and helper.HttpHandle both construct &http.Client{} with no Timeout — can hang indefinitely, blocking the consumer goroutine until the 60s context fires, triggering nack-drop-while-working scenario above. | helper/helper.go:161,213
  - DOUBLE-SPEND GUARD (PARTIAL): origin.Withdraw checks for existing status=3 pending withdraw_statement before proceeding (line 31-37). This is a SELECT without a lock — TOCTOU race is possible under concurrent requests. | origin/withdraw.go:31-37
  - BUG — NIL POINTER DEREFERENCE ON FAILED DIAL: pub/connection.go:44 sets conn.Config.Heartbeat BEFORE the if err == nil check. If amqp.Dial fails, conn is nil and this line panics. | pub/connection.go:44
  - HARDCODED TELEGRAM BOT TOKEN: GetTelegramToken() returns a literal token string — credential exposed in source. | helper/helper.go:384
  - HARDCODED MONITORING URL: monitor.thezeus.online pinged on every message — external dependency, fire-and-forget goroutine, no timeout on axios.Get default. | pub/amqp.go:120
  - REDIS WALLET DEDUP IS UPSTREAM-ONLY: wallet_withdraw_{username} Redis key is checked and deleted inside this service's WALLETWITHDRAW branch (pub/amqp.go:135-144), but the key is SET upstream before enqueueing — if enqueue succeeds but this service crashes before deleting the key, the key TTL determines whether retries are blocked. Redis IdleTimeout=1min; key TTL not visible in this repo. | pub/amqp.go:135-144, helper/redis.go:95
  - ON DB ERROR AT STARTUP: startQueue.go logs error and time.Sleep(5s) then returns — no retry loop; process exits and relies on container restart policy. | startQueue.go:18-22
  - CONNECTION RECONNECT: max 200 retries, 1s backoff in pub/connection.go — process will attempt reconnect for ~3min before giving up. | pub/connection.go
  - MongoDB pool: min=1, max=5, idle timeout=5min, connectTimeout=15s. | _config/db.go:82
