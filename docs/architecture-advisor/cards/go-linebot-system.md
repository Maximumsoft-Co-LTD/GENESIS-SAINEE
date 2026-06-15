## Card: go-linebot-system

identity: Go 1.19 | _cmd/main.go → linebotsystem.StartServer() | hardcoded port 4067 | Dockerfile present, docker-compose.yml present, no CI workflow found

provides:
  http:
    POST /webhook/:key              | no auth middleware (CRITICAL: no LINE signature verification) | route.go:18
    POST /redis/reset_redis         | no auth middleware                                            | route.go:22
    POST /redis/reset_redis_key     | no auth middleware                                            | route.go:23
    POST /report/summary            | middleware.AuthHeader() (JWT)                                 | route.go:27
    POST /report/summary_user       | middleware.AuthHeader() (JWT)                                 | route.go:28
    POST /report/round              | middleware.AuthHeader() (JWT)                                 | route.go:29
    POST /report/round_detail       | middleware.AuthHeader() (JWT)                                 | route.go:30
    POST /report/member_credit      | middleware.AuthHeader() (JWT)                                 | route.go:31
    POST /report/history_user       | middleware.AuthHeader() (JWT)                                 | route.go:32

  consume: (none — no RabbitMQ consumer declared)

  cron:
    serviceWebHook every 5 minutes — calls LINE API to test each bot's webhook URL | controller/Init.go:282-285
    schedule: s.Every(5).Minutes().Do(serviceWebHook, dataService) | controller/Init.go:284

consumes:
  http-out:
    https://api.line.me/v2/bot/channel/webhook/test | POST, Bearer <AccessToken from DB> | controller/Init.go:307-310
    env DOMAIN_CUSTOMER + path — generates JWT-signed deep-link URLs for member access | service/authJWT.go:48
    env-keyed API URL (configAgent.ApiUrl from database_config collection) + various paths via CallApiMain | service/mainService.go:151-198
    https://allapi.topupoffice.com/api/GetBankListConfig | GET, no auth, hardcoded URL | service/mainService.go:211

  data:
    MongoDB env MONGODB_ENDPOINT + MONGODB_DB_NAME (OFFICE db) | collection: database_config, question, config_system, line_config, member_account, member_log | _config/db.go:29-56
    MongoDB per-service (multi-tenant) — additional DBs loaded dynamically from database_config collection at startup | _config/db.go:62-107
    Redis env (REDIS_URL) — configService_<key>, configAgent_<key> cached | service/redisService.go (inferred from controller/Init.go:57-68)

  external:
    LINE Messaging API (api.line.me) — send push messages, reply messages | service/lineMessage.go, service/lineOA.go
    LINE Bot webhook test endpoint | controller/Init.go:307

observations:
  - CRITICAL: LINE webhook endpoint POST /webhook/:key has NO X-Line-Signature HMAC-SHA256 verification — any attacker can POST forged events to this endpoint | route.go:18, webhook.go:15-140
  - CRITICAL: The BotConfig struct stores SecretKey in MongoDB (models/lineConfig.go:31) but it is NEVER used for signature validation anywhere in the codebase | grep result: zero matches for hmac/sha256/X-Line-Signature
  - CRITICAL: /redis/reset_redis and /redis/reset_redis_key have NO authentication — anyone can flush/delete arbitrary Redis keys | route.go:22-23
  - Multi-tenant key routing via URL path param /:key — if key is guessable/predictable, cross-tenant event injection is possible | webhook.go:18
  - LINE AccessToken and SecretKey stored in MongoDB (not env) — compromise of DB = compromise of all LINE bots | models/lineConfig.go:29-33
  - CORS allows "*" all origins, all headers | server.go:44-47
  - JWT for report endpoints uses env ACCESS_SECRET; uses deprecated github.com/dgrijalva/jwt-go v3.2.0 (CVE-2020-26160) | go.mod:6, service/authJWT.go:9
  - Port 4067 hardcoded in server.go (no PORT env fallback) | server.go:40
  - ConfigInit panics on missing DB config — no graceful startup failure | controller/Init.go:113
  - Context cancel with defer inside a for loop in ConfigInit — deferred cancel may not fire until function returns, leaking contexts | controller/Init.go:128-129
  - ExternalCALL in service/callback.go has 30s timeout (acceptable), but _config/db.go's externalCALL has NO timeout | _config/db.go:150
  - ioutil.ReadAll used (deprecated since Go 1.16) | service/callback.go:6
  - go.mod dependency golang.org/x/crypto v0.5.0 — outdated, several CVEs fixed in later versions | go.mod
