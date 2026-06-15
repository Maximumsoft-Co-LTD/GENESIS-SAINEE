## Card: hash-central
identity: Go 1.25.5 | module `app` | entrypoint `_cmd/main.go` → `app.StartService()` or `app.StartServiceCronjob()` (when TYPE=CRONJOB) | port 8001 | Dockerfile: yes | CI: yes (.github/workflows/)

provides:
  http:    POST /api/hashlayout | no auth | controller/hash.go:29 (routes/main.go:32)
           POST /api/hashlayout-list | no auth | controller/hash.go:83 (routes/main.go:33)
           POST /api/get-channel | no auth | controller/channel.go (routes/main.go:34)
           POST /api/change-channel | no auth | controller/channel.go (routes/main.go:35)
           POST /api/bank-account | no auth | routes/main.go:52
           POST /api/slip-verify | no auth | routes/main.go:53 (SlipController.ScanSlip)
           POST /api/line-bot-config | no auth | routes/main.go:54
           POST /api/v2/slip-verify | no auth | routes/main.go:57 (SlipController.SlipVerify)
           POST /api/v2/use-slip-verify | no auth | routes/main.go:58
           GET  /api/v2/profile-slip | no auth | routes/main.go:59
           POST /api/con-proxy-test | no auth | routes/main.go:65
           POST /api/v2/get-proxy | `api-key` header via middlewares.AuthRequired | routes/main.go:69
           POST /api/v2/callback-proxy | auth | routes/main.go:70
           POST /api/v2/* (12 more proxy management endpoints) | auth | routes/main.go:71-84
           POST /api/get-device-botapp | no auth | routes/main.go:87
           POST /api/save-device-botapp | no auth | routes/main.go:88
           POST /api/upload-withdraw-img | no auth | routes/main.go:89
           POST /api/bot/gen-device | no auth | routes/main.go:95
           POST /api/bot/* (8 more bot endpoints) | no auth | routes/main.go:96-104
           POST /api/save-face-data | no auth | routes/main.go:107
           POST /api/get-face-data | no auth | routes/main.go:108
           POST /webhook | no auth (root level) | routes/pxy-man.go (RoutePxyMan)
           POST /webhook-by-proxy | no auth (root level) | routes/pxy-man.go
           POST /get-device-generate | no auth (root level) | routes/pxy-man.go
           POST /update-device-generate | no auth (root level) | routes/pxy-man.go
  consume: none (no RabbitMQ consumer)
  cron:    TYPE=CRONJOB mode → app.StartServiceCronjob() → controller.CronjobFindProxy() | _cmd/main.go:22-23

consumes:
  http-out: `QR_DECODER_URL` env → maan-qrdecoder POST (QR image decode) | service/qrclient/ (app.go:69)
  http-out: `SCB_SERVICE` env → SCB bank API (webhook forward / slip verify) | controller/bank_proxy.go
  http-out: KTB bank API directly (slip verification + ECDSA-signed prelogin grant) via resty — proxy optional via `KTB_PROXY_URL` env | controller/slip_verify_ktb.go
  http-out: GSB bank API directly (slip verification) | service/verifySlip/
  http-out: `SLIP2GO_API_KEY` env → slip2go fallback service POST (KTB fallback) | controller/slip_verify.go:655
  http-out: Telegram bot API (optional, error notifications) | service/telegram/
  publish:  none
  data:     `DB_NAME` env / `bank_account` R+W | repo/repo.go
            `DB_NAME` / `bank_statement` R (dedup lookup) | controller/hash.go:207-213
            `DB_NAME` / `transaction_statement` R+W (dedup insert + status update) | controller/hash.go:446-512
            `DB_NAME` / `log_hash` W | controller/hash.go:309-358
            `DB_NAME` / `proxy` R+W | controller/proxy_management.go
            `DB_NAME` / `proxy_config` R+W | controller/proxy_management.go
            `DB_NAME` / `proxy_address` R+W | controller/proxy_management.go
            `DB_NAME` / `device_botapp` R+W | controller/botapp.go
            `DB_NAME` / `face_data` R+W | controller/faceapp.go
            `DB_NAME` / `channel` R (channel config for hash selection) | controller/hash.go:207
            `DB_NAME` / `log_channel` W | controller/channel.go
            `DB_NAME` / `bank_account_slip` R+W | controller/slip_verify.go:879
            `DB_NAME` / `slip_verify` R+W (idempotency + orphan recovery) | controller/slip_verify.go:515-593
            `DB_NAME` / `log_slip_verify` W | controller/slip_verify.go:1068
            `DB_NAME` / `line_slip_config` R+W | controller/slip_verify.go:1132
            Redis `slip_verify:<txID>` ObtainLock (30s TTL) | controller/slip_verify.go:516
            Redis `slip_verify:cache:<qr_hash>` (TTL=SLIP_VERIFY_CACHE_TTL_SECONDS, default 18000s) | controller/slip_verify_cache.go
            Redis `qr_decode:cache:<input_hash>` (TTL=QR_DECODE_CACHE_TTL_SECONDS, default 86400s) | controller/qr_decode_cache.go
            Redis `ktb:devices` / `ktb:devices:healthy` sets (when KTB_DEVICE_POOL=true) | app.go:126-128
  external: KTB bank API (ECDSA P-256 signed, credentials from env: KTB_EC_PRIVATE_KEY_B64, KTB_HASH_PASS, KTB_OAUTH_CLIENT_ID, KTB_OAUTH_CLIENT_SECRET) | controller/slip_verify_ktb.go
            GSB bank API | service/verifySlip/
            slip2go third-party fallback service | service/verifySlip/

observations:
  - HASH ALGORITHM: SHA-1 (`crypto/sha1`) used for ALL transaction deduplication hashes (Hash, HashAddOneMin, HashSubOneMin, HashSubOneDay) — SHA-1 is cryptographically broken and collision-vulnerable | controller/hash.go:9,515-520
  - Hash input is: `datetime_format + amount_str + masked_bank_number + bank_code` — low entropy input to SHA-1 increases practical collision risk for same-amount same-time deposits | controller/hash.go:248-274
  - All `/api/hashlayout`, `/api/hashlayout-list`, `/api/slip-verify`, `/api/v2/slip-verify` routes have NO authentication — any caller with network access can submit statements or verify slips | routes/main.go:32-59
  - `/webhook` and `/webhook-by-proxy` (SCB webhook receivers) at root level with no auth | routes/pxy-man.go
  - `InsertTransactionStatement` has a `hash_service` application-level dedup guard (GetOne before insert) but the unique index is COMMENTED OUT — race condition window between check and insert | controller/hash.go:495-500 (commented index lines 496-504)
  - Redis slip_verify lock uses ObtainLock (thorlock distributed mutex, 30s TTL) — correct idempotency pattern ✓ | controller/slip_verify.go:516
  - Orphan recovery for stuck slip_verify records (status=2, age > 5min) is present with transaction_statement cross-check ✓ | controller/slip_verify.go:535-560
  - Global Redis singleton (package-level) in service/redis/ — not constructor-injected, harder to test | CLAUDE.md "Global singletons"
  - `time.Sleep(5s)` and `time.Sleep(1h)` inside proxy validation loop — blocks goroutines, slow tests | controller/proxy_management.go + validate_proxy.go
  - KTB device pool pre-warm goroutine target default 300 may trip KTB anti-abuse ceiling | CLAUDE.md KTB_POOL_TARGET_HEALTHY note
  - KTB credentials (KTB_EC_PRIVATE_KEY_B64, KTB_HASH_PASS, etc.) are env-injected (correct) but app.go does NOT validate them at startup — first request fails with [KTB-AUTH-ENV-MISSING] | app.go:123-143
  - 14 sequential KTB bug-fix workflow runs (.workflow/) signals ongoing fragility in KTB integration | CLAUDE.md "KTB tread-carefully zone"
  - `CORS` middleware in app.go sets `Access-Control-Allow-Origin: *` with no restriction | app.go:29
  - `http.Get(imageUrl)` in controller/slip_verify.go:1196 has no timeout (plain net/http) for CloudFront URL image fetch | controller/slip_verify.go:1196
  - ~13 test files, strong coverage for KTB pool and ktbcrypto; slip_verify main path has good coverage ✓
