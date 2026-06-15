## Card: go_deposit
identity: Go 1.24.6 | module `godeposit` | entrypoint `_cmd/main.go` → `rabbitmq.Worker()` | NO HTTP server (pure RabbitMQ worker) | Dockerfile: yes | CI: yes (.github/workflows/)

provides:
  http:    none
  consume: queue name = `os.Getenv("QUEUE_NAME")` (durable, no-wait=true, auto-ack=false, prefetch=1) | handlers: `WorkerDeposit`, `WorkerWalletSystem`, UPDATECONFIG in-place reload | rabbitmq/worker.go:61-88
  cron:    none declared

consumes:
  http-out: `FIREBASE_URL` env + `/notify/dynamic/<username>.json` PATCH (fire-and-forget goroutine) | deposit/firbase.go:27
  publish:  none (wallet/deposit credit is a direct DB write; no downstream publish confirmed in this service)
  data:     officeDB `abaoffice_db` / `database_config` R | db/db.go:127
            officeDB `abaoffice_db` / `config_system` R | rabbitmq/worker.go:50
            serviceDB (tenant, name from `configDatabase.DbName`) / `bank_statement` R+W | deposit/repository.go (multiple FindOne/UpdateOne)
            serviceDB / `deposit_statement` R+W | deposit/repository.go:64
            serviceDB / `member` R+W | deposit/repository.go
            serviceDB / `wallet_statement` R+W | deposit/repository.go:53
            serviceDB / `bonus_config` R | deposit/repository.go:58
            serviceDB / `member_promotion` R+W | deposit/repository.go:68
  external: Sentry (DSN hardcoded in source) | _cmd/main.go:20-25

observations:
  - CRITICAL: Sentry DSN hardcoded in source: `https://80e85c485deee4561d73697c304d31d7@o4506229598846976.ingest.sentry.io/4506229940486144` | _cmd/main.go:21 — credential in git history
  - CRITICAL: MongoDB office DB connection string hardcoded: `mongodb://abaoffice:zxcvasdf789@mongodb-aba.cluster-c0skt7cppmab.ap-southeast-1.docdb.amazonaws.com:27017/...` | db/db.go:64 — also MOOTUI and ESCOBAR team credentials hardcoded on lines 66-68
  - `failOnError` in rabbitmq/worker.go:16-19 swallows ALL RabbitMQ channel errors silently (`log.Fatalf` is commented out) — consumer can silently stop processing with no alert
  - `db.HealthCheckDatabase` calls `panic(err)` after 5 retries — production-path panic in a long-running worker | db/db.go:214
  - `go CallAPI(...)` in deposit/firbase.go:27 is fire-and-forget (goroutine leak risk on Firebase outage); http.Client has 10s timeout ✓ but no context propagation
  - No idempotency key on `InsertDepositStatement` — duplicate queue delivery (no-wait=true, possible redelivery on reconnect) can double-credit a user | deposit/repository.go:64
  - ZERO test files on critical-path deposit worker
  - RabbitMQ reconnect loop: maxRetry=500, 2s sleep, no jitter | rabbitmq/connection.go:55
  - `WorkerDeposit` function signature `(officeDB, database, configDatabase, _id string, redisClient)` — 5 positional args, no DI struct, hard to test | rabbitmq/worker.go:127
  - `fmt.Println` used for ALL logging including errors — no structured fields, no severity | deposit/deposit.go passim
  - `time.Sleep(5 * time.Second)` inside retry loop at deposit/deposit.go:68 — blocking sleep without context
  - Queue declared with `no-wait=true` (line 46) but errors on `ch.QueueDeclare` are swallowed by `failOnError` — if queue declaration fails, consumer is silently a no-op | rabbitmq/worker.go:61-69
