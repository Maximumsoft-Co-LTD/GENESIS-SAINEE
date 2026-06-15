## Card: topupserie

identity: Go 1.21 | _cmd/main.go → topupserie.StartServer() | PORT env (default none — must be set) | Dockerfile present, docker-compose.yml present, no CI workflow found

provides:
  http:
    POST /login                         | middleware.AccessIP()                          | route.go:38
    POST /login_line                    | middleware.AccessIP()                          | route.go:39
    POST /loginbytoken                  | middleware.AccessIP()                          | route.go:40
    POST /loginbypin                    | middleware.AccessIP()                          | route.go:41
    POST /register                      | middleware.AccessIP(), RateLimitExtended       | route.go:54
    POST /register_v3                   | middleware.AccessIP(), RateLimitExtended       | route.go:55
    POST /withdraw                      | middleware.AccessIP(), AuthHeader              | route.go:57
    POST /deposit_auto                  | middleware.AccessIP(), AuthHeader              | route.go:166
    POST /deposit_auto_v3               | middleware.AccessIP(), AuthHeader              | route.go:167
    POST /decimal                       | middleware.AccessIP(), AuthHeader              | route.go:73
    POST /qrpay                         | middleware.AccessIP(), AuthHeader              | route.go:74
    POST /qrpay_v2                      | middleware.AccessIP(), RateLimitExtended, AuthHeader | route.go:77
    POST /qrpay_umpay                   | middleware.AccessIP(), AuthHeader              | route.go:78
    POST /slip_qrcode                   | middleware.AccessIP(), AuthHeader              | route.go:82
    POST /g2e_pay                       | middleware.AccessIP(), AuthHeader              | route.go:83
    POST /g2e_pay_confirm               | no auth middleware                             | route.go:84
    POST /otp                           | RateLimitExtended (no AuthHeader)              | route.go:67
    POST /otpv2                         | no auth middleware                             | route.go:68
    POST /otp_peer2pay                  | no auth middleware                             | route.go:69
    POST /peer2pay_withdraw             | no auth middleware                             | route.go:70
    POST /game/bet_lotto                | RateLimitExtended, AuthHeader                  | route.go:342
    POST /game/bet_lotto (apex-pro)     | AuthHeader only (no RateLimit)                 | route_apex_pro.go:259
    POST /callback_wheel                | no auth middleware                             | route.go:114
    POST /callback_wheel_multi          | no auth middleware                             | route.go:115
    GET  /office/systemConfig           | no auth middleware                             | route.go:287
    GET  /office/reset_redis            | no auth middleware                             | route.go:290
    GET  /office/flushall-redis         | no auth middleware                             | route.go:288
    GET  /kontor/flushall-redis         | no auth middleware                             | route_apex_pro.go:222
    GET  /default/report                | no auth middleware                             | route.go:444
    POST /get/coupon                    | middleware.AccessIP(), AuthHeader              | route.go:226
    POST /get/list_coupon               | middleware.AccessIP(), AuthHeader              | route.go:227
    POST /get/history_coupon            | middleware.AccessIP(), AuthHeader              | route.go:230

  consume:
    RabbitMQ — publish+reply-queue RPC pattern | queue name: "<ROUTINGKEY>_<SERVICE>" (e.g. WALLETWITHDRAW_<SERVICE>) | service/mainService.go:479-481
    RabbitMQ — log queue via RABBITMQURLLOG env | queue name: anonymous reply queue | service/mainService.go:526-540

  cron:
    service.ChangeOTPLink(resource) — goroutine, interval not visible from server.go (called once at boot) | server.go:64

consumes:
  http-out:
    env API_URL (topupserie agent API) + multiple paths | service/agentService.go (pattern via mainService.go CallApiMain) | service/mainService.go:151-198
    https://allapi.topupoffice.com/api/GetBankListConfig | GET, no auth | service/mainService.go:211 🟡 hardcoded external URL
    https://notify-api.line.me/api/notify | POST, Bearer token | _config/db.go:136-143

  publish:
    exchange="" routing-key="<QUEUENAME>_<SERVICE>" | amqp.Publishing with ReplyTo reply queue | service/mainService.go:479-492

  data:
    MongoDB env MONGODB_DB_NAME | collections: member, wallet, statement, config_system, game_config, etc. (inferred from controller ops) | _config/db.go:43
    MongoDB env MONGODB_DB_NAME_LOG (optional) | log collections | _config/db.go:70
    Redis env REDIS_URL | session cache, rate-limit counters, game/config cache | _config/rd.go (inferred)

  external:
    LINE Notify API — push notifications on DB connect failure | _config/db.go:134-143
    G2E payment gateway (G2EPayInit / G2EPayCallback) | route.go:83-84
    Peer2Pay payment gateway | route.go:69-70
    RocketWin/game agent API (URL from env) | service/agentService.go

observations:
  - CRITICAL: Hardcoded LINE Notify access token "pKahNdB8I8ifXmaeekmD66EbvXSgKMF5hBYUt5bgiNa" in production code | _config/db.go:135
  - CRITICAL: /office/systemConfig, /office/flushall-redis, /office/reset_redis, /kontor/system-config, /kontor/flushall-redis — admin endpoints with NO authentication middleware | route.go:287-295, route_apex_pro.go:218-228
  - CRITICAL: /g2e_pay_confirm, /callback_wheel, /callback_wheel_multi — payment/game callbacks with NO auth middleware | route.go:84,114,115
  - CRITICAL: /default/report — reporting endpoint with NO auth middleware | route.go:444
  - CORS allows "*" on all origins, all methods, all headers — no restriction | server.go:119
  - RabbitMQ consume uses auto-ack=true on reply queue (no negative ack on failure) | service/mainService.go:455
  - No DLQ configured for RabbitMQ publish pattern | service/mainService.go:434-522
  - TYPEAUTH env controls auth mode between "apikey" and "authorization" — if env is unset, silently falls back to "apikey" | route.go:22-25
  - Duplicate route registration: same handlers registered under English paths (route.go) AND Swedish-obfuscated paths (route_apex_pro.go) — wider attack surface | route_apex_pro.go
  - /otp and /otpv2 endpoints have no auth, /otp has rate limit but /otpv2 has NONE | route.go:67-68
  - http.Client in _config/db.go externalCALL has no timeout set (uses net/http default = no timeout) | _config/db.go:150
  - ioutil.ReadAll (deprecated since Go 1.16) used | _config/db.go:162
  - go.mod uses deprecated github.com/dgrijalva/jwt-go (known CVE-2020-26160) | go.mod:8
  - Signal channel `quit := make(chan os.Signal)` created as unbuffered — technically unsafe before Notify call | server.go:97
