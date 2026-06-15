## Card: coupon-service

identity: Go 1.20 | _cmd/main.go → coupon.Server() | PORT env (default 8080) | Dockerfile present, docker-compose.yml present, no CI workflow found

provides:
  http:
    GET  /api/coupon/                        | no auth middleware | internal/route/router.go:25
    POST /api/coupon/                        | no auth middleware | internal/route/router.go:26
    GET  /api/coupon/prefix/:prefix          | no auth middleware | internal/route/router.go:27
    GET  /api/coupon/:id                     | no auth middleware | internal/route/router.go:28
    GET  /api/coupon/code/:id                | no auth middleware | internal/route/router.go:29
    POST /api/coupon/ref_bonus               | no auth middleware | internal/route/router.go:30
    PATCH /api/coupon/:id                    | no auth middleware | internal/route/router.go:31
    DELETE /api/coupon/:id                   | no auth middleware | internal/route/router.go:32
    GET  /api/statement/:id                  | no auth middleware | internal/route/router.go:36
    GET  /api/redeem/:code                   | no auth middleware | internal/route/router.go:41
    POST /api/redeem/                        | no auth middleware | internal/route/router.go:42
    DELETE /api/redis/flushall               | no auth middleware | internal/route/router.go:47
    GET  /api/redis/                         | no auth middleware | internal/route/router.go:48
    DELETE /api/redis/                       | no auth middleware | internal/route/router.go:49
    GET  /api/access/                        | middleware.AccessIpOfficeMiddleware         | internal/route/router.go:54
    PUT  /api/access/                        | middleware.AccessIpOfficeMiddleware         | internal/route/router.go:55
    GET  /api/test/demo                      | no auth middleware | internal/route/router.go:59
    GET  /api/test/redis                     | no auth middleware | internal/route/router.go:60

  consume: (none — no RabbitMQ consumer registered; MQ declared in config but not used at runtime)

  cron: (none)

consumes:
  http-out: (none — no external HTTP calls found in controller or service code)

  data:
    MongoDB env MONGODB_ENDPOINT + MONGODB_NAME | collection: coupon_config (R/W) | controller/redeem/repo.go:32,55,83,143,222; _config/db.go:37
    MongoDB env MONGODB_ENDPOINT + MONGODB_NAME | collection: coupon_statement (R/W) | controller/redeem/repo.go:100,119,167,191
    Redis env (REDIS_URL) | key: "code-<service>-<couponCode>" with 2h TTL for MANYTIME coupons | controller/redeem/repo.go:22,38

  external: (none)

observations:
  - CRITICAL: ALL coupon management endpoints (create, update, delete, GET by code/id/prefix) have NO authentication middleware — any unauthenticated caller can create/delete coupons or enumerate all codes | internal/route/router.go:25-32
  - CRITICAL: POST /api/redeem/ (coupon redemption — credits balance) has NO authentication | internal/route/router.go:42
  - CRITICAL: DELETE /api/redis/flushall — wipes entire Redis instance, NO auth middleware | internal/route/router.go:47
  - CRITICAL: Only AccessIpOfficeMiddleware (IP allowlist) protects /api/access/ — all other endpoints are fully open to internet with no credential check | internal/middleware/access_ip.go (inferred from router.go:54-55)
  - Double-redemption protection analysis: CreateRedeem calls checkRedeemCouponStatement (DB read), then createRedeemCouponStatement (DB write), then updateRedeemCouponConfig ($inc) — these three ops are NOT atomic; a concurrent race window exists between the check and the insert, allowing double redemption under parallel requests | controller/redeem/service.go:44-68, controller/redeem/repo.go:156-196
  - No Redis distributed lock (SETNX/RedLock) around the check-then-insert redeem flow | controller/redeem/service.go:31-75
  - redisAddQueue in service.go uses SetNX for serializing per-queue processing, but this mechanism is commented out in the router (routes /api/test/queue-en and queue-de are commented) — the current production path in CreateRedeem calls handleRedeemCoupon directly without queue serialization | controller/redeem/handler.go:93; router.go:60-66
  - No idempotency key on POST /api/redeem/ — caller must handle duplicates; there is NO unique index enforcement visible at API layer | controller/redeem/handler.go:69-96
  - coupon_statement uses a compound hash field (username+couponID) for dedup | controller/redeem/repo.go:98,163 — but no unique MongoDB index declaration is visible in code; if index is missing, race condition allows duplicates
  - MQ (RabbitMQ) is declared as a Resource field but CreateMQ() is NEVER called in server.go — MQ resource is always nil at runtime | server.go:1-110, _config/mq.go
  - CORS allows "*" on all origins | server.go:93-108
  - No request-body size limit set on gin.Default() | server.go:46
  - MigrateTypeWithdraw runs as a goroutine on every server start with no idempotency guard | internal/route/router.go:21
  - go.mod uses gopkg.in/mgo.v2 (unmaintained MongoDB driver) as indirect dep alongside official mongo-driver | coupon/go.mod:16
