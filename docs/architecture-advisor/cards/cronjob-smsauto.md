## Card: cronjob-smsauto

identity: Go 1.19, Gin | smsauto.StartServer() → _cmd/main.go:8 | port dynamic (r.Run() no explicit port, falls back to $PORT or 8080) | Dockerfile present, docker-compose.yml present

provides:
  http:
    GET  /test/demo  | no auth | route.go:13
    GET  /test/demo2 | no auth | route.go:14
  cron:
    every 10 min  — HandleSmsAutoTypeOneNodeposit  (type=1, ลูกค้าสมัครแต่ไม่ฝาก)  | controller/smsauto.go:544
    every 10 min  — HandleSmsAutoTypeOneSendPromotion (type=3, ส่งโปรโมชั่น)        | controller/smsauto.go:547
    every 3 hours — HandleSmsAutoTypeTwoOffline  (type=2, ลูกค้าเลิกเล่น)           | controller/smsauto.go:550
    daily 23:59   — HandleSmsAutoTypeThreeVip    (type=vip, VIP top members)         | controller/smsauto.go:553

consumes:
  http-out:
    SMSKUB (default)     | hardcoded "https://console.sms-kub.com/api/messages" (immediate) and "https://console.sms-kub.com/api/campaigns" (scheduled) | service/smsService.go:387,399
    SMSKUB configurable  | configSms.Sms.API + "/messages" or "/campaigns" if set in DB                                                                 | service/smsService.go:388,400
    YingSms              | configSms.SmsMkt.Link + "/v1/campaign?service=hpx"                                                                           | service/smsService.go:443
    Legacy SMS log       | hardcoded "http://sendsms.lupin.host/api/sms_log" (DEAD — commented-out callers)                                             | service/smsService.go:264
    abaplan (legacy)     | hardcoded "https://abaplan.sunnygreen.online/api/smsauto/smsauto_statement?type=" (GetSmsAutoStatement — not called by cron)  | service/smsService.go:284 🟡 dead path
  data:
    OFFICE DB (env MONGODB_OFFICE_DB_NANE) / collection "smsauto_statement"   | R   | service/smsService.go:315
    OFFICE DB / collection "member_account"                                    | R   | service/smsService.go:36, 68, 143, 168
    OFFICE DB / collection "report_active_user_day"                            | R   | service/smsService.go:115
    OFFICE DB / collection "report_active_user_month"                          | R   | service/smsService.go:191, 208
    OFFICE DB / collection "log_sms_auto"                                      | W   | service/smsService.go:518
    OFFICE DB / collection "smssystem_config"                                  | R   | service/smsService.go:324
    OFFICE DB / collection "config_system"                                     | R   | service/smsService.go:332
    Per-tenant DBs (loaded from database_config collection, keyed by Service)  | R   | _config/db.go:65-105
    Redis DB=5 (env REDIS_URL)  key "sentPhoneNumber_{service}_{date}"         | R/W | service/redis.go:41, controller/smsauto.go:109-113

observations:
  - CRITICAL: SMS provider credentials (username, password) are stored in MongoDB collection "config_system" and embedded verbatim into bson.M log payload and written to "log_sms_auto" | controller/smsauto.go:73-74 — provider password stored unredacted in logs
  - HARDCODED Line Notify token present in service/mainService.go:0 (no such call here, but see rocketwin — cronjob-smsauto itself does NOT have a hardcoded token)
  - MISSING HTTP timeout: CallAPIALL and CallAPIYingSms use http.Client{} with no Timeout set | service/mainService.go:113, service/mainService.go:135 — can hang indefinitely
  - CallAPIALL uses panic(err) on HTTP error instead of returning error | service/mainService.go:118 — will crash the process
  - CallAPIYingSms also uses panic(err) on HTTP error | service/mainService.go:137
  - No idempotency key on SMS send — if cron fires twice (e.g. pod restart mid-run), the same batch can be sent twice; Redis dedup key is only checked BEFORE send and set AFTER, with no atomic CAS | controller/smsauto.go:109-113
  - defer redis.Close() inside a loop creates a deferred close that runs at function return, not loop iteration — redis connection leaks until the outer function returns | controller/smsauto.go:109 (and other handlers)
  - go.mod uses Go 1.19 — significantly outdated relative to other repos in the monorepo
  - Port is undeclared — r.Run() without args reads $PORT; if unset defaults to :8080, but not documented
  - OVERLAP RISK with go_sms: Both services read "config_system" / "smssystem_config" to obtain SMS credentials; cronjob-smsauto runs autonomous scheduled sends while go_sms presumably handles on-demand sends — no coordination/locking mechanism exists to prevent concurrent SMS to same phone number beyond the Redis dedup key (which is date-granular, not per-minute)
