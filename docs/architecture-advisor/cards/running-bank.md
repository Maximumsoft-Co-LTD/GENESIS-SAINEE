## Card: running-bank
identity: Go 1.21 | module `app` | entrypoint `_cmd/main.go` → `app.StartService()` → `runbankall.StartServ()` | NO HTTP server in main loop (polling worker); secondary HTTP handlers exist under `controller/runbank/` but are not started from main | Dockerfile: yes | CI: yes (.github/workflows/)

provides:
  http:    (secondary, not started by default) GET/POST handlers under `controller/runbank/` for manual bank-run control | controller/runbank/runbank.main.go (not wired in app.go)
  consume: none (no RabbitMQ consumer; produces only)
  cron:    implicit infinite polling loop: sleep KBANK 150-200s, SCB 180-250s, GSB 300-400s, others 90-160s | controller/runbankAll/runbank-all.main.go:65-85

consumes:
  http-out: `HASH_CENTRAL` env + `/api/hashlayout-list` POST (dedup check per statement batch) | service/hashCentral/main.go:35 — `http.Client{}` with NO timeout set on client struct 🟡
  http-out: bank statement APIs (KBANK, SCB, GSB, BBL, others) — credentials loaded from `DatabaseConfig.BankAPIURL` / `BankAPIToken` per tenant | controller/runbankAll/runbank-all.main.go (various bank fetch functions)
  http-out: `SLIP_ACCEPT` env (optional slip check endpoint) — no timeout on client | service/hashCentral/main.go (CallSlipCheck)
  http-out: Firebase monitor PATCH (optional, `FIREBASE_URL` / LINE Notify) | firebase/ and linenotification/
  publish:  exchange `STATEMENT` (fanout, durable) — `db.BankStatement` JSON | report/publish.go:34 — queue name varies: `STATEMENT` or `STATEMENT_<service>` when GROUP=NEWTOPUP
            queue `<service>` (direct, durable, persistent) — `_id.Hex()` string for go_deposit worker | addCredit.go:42-76 and controller/runbankAll/runbank-all.main.go:414-435
  data:     officeDB / `database_config` R (all tenant configs) | repository/repoAbaoffice.go:GetBankConfig
            tenantDB / `bank_statement` R+W (insert new statements, update status) | repository/repoAbaoffice.go
            tenantDB / `bank_config` R (per-bank credentials) | repository/repoAbaoffice.go
  external: LINE Notify `https://notify-api.line.me/api/notify` POST | report/publish.go:68 (no timeout on http.Client)
            Bank statement APIs (KBANK API, SCB Business API, GSB API, etc.)

observations:
  - `repository/repoAbaoffice.go` GetBankConfig calls `panic(err)` on Mongo query failure — crashes the entire polling loop for all tenants | repoAbaoffice.go (cited in CLAUDE.md)
  - `http.Client{}` in service/hashCentral/client.go:12 has NO Timeout field — a hung hash-central HTTP call blocks the entire polling loop indefinitely (single-threaded) | service/hashCentral/client.go:12
  - Single-threaded polling loop: one hung bank API stalls ALL tenant polling; no per-tenant goroutine isolation | controller/runbankAll/runbank-all.main.go:45-93
  - Error handling pattern: `if err := runningBank(...); err != nil { fmt.Printf("Error running bank: %v\n", err) }` — errors logged and dropped, no propagation | runbank-all.main.go:87-89
  - `report/publish.go` opens a new `amqp.Dial` per statement publish — no connection pooling, O(N) connections per poll cycle | report/publish.go:17
  - `addCredit.go` also opens a fresh `amqp.Dial` per credit message | addCredit.go:16
  - Dedup via hash-central is OPTIONAL: if `HASH_CENTRAL` env is empty the call silently returns the full list unfiltered — duplicate statements enter `bank_statement` | service/hashCentral/main.go:12-14
  - No idempotency on statement insert — a retry after partial failure can insert duplicates | controller/runbankAll/runbank-all.main.go
  - Go version 1.21 (oldest in monorepo) — no range-over-int, no slices.* stdlib
  - `rand.Seed(time.Now().UnixNano())` called every loop iteration (deprecated in Go 1.20+) | runbank-all.main.go:66-84
  - `ioutil.ReadAll` / `ioutil.ReadFile` used (deprecated since Go 1.16) | report/publish.go:93 and others
  - Only 2 test files; polling loop has no integration test harness
  - `crypto/sha1` imported in runbank-all.main.go:12 — SHA-1 hash used internally for statement dedup (same algo as hash-central)
  - LINE Notify `http.Client` in report/publish.go:75 has no Timeout set
