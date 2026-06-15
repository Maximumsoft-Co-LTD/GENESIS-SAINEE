# Card: coupon-service
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go + Gin + MongoDB (raw driver) + Redis (go-redis/v8) + Prometheus (gin middleware). มี config RabbitMQ (streadway/amqp) แต่**ไม่ถูกเรียกใช้** (ดู observations)
- **module**: `coupon` — `_cmd/main.go:4` (`import coupon "coupon"`), package server (`server.go:1` `package server`)
- **entrypoint**: `_cmd/main.go:8` `coupon.Server()` → `server.go:25` `func Server()`
- **port**: `os.Getenv("PORT")` default **8080** — `server.go:55-58`, listen `server.go:62-64`
- **Dockerfile**: `Dockerfile` — builder `golang:1.20-alpine3.19` → `alpine:3.19`, build `_cmd/main.go`, `CMD ["./app"]` (ไม่มี EXPOSE)
- **CI**: `.github/workflows/build.yaml` "CI/CD Pipeline" — Kaniko (`gcr.io/kaniko-project/executor`) build → trigger edit image (GitOps)

## provides

### http
ลงทะเบียนใน `internal/route/router.go`. มี `r.Use(CORS)` (allow `*`, `server.go:51`/`server.go:90`) และ Prometheus middleware (`server.go:48-49`). Auth: เฉพาะ group `/api/access` ใช้ `AccessIpOfficeMiddleware` (เช็ค IP จาก env `ACCESS_IP_OFFICE`). group อื่นใช้ query `service` แทน auth (ไม่มี JWT/token)

| METHOD | path | auth | evidence |
|--------|------|------|----------|
| GET | /api/coupon/ | none (query `service`) | router.go:25, handler.go:14 |
| POST | /api/coupon/ | none | router.go:26 |
| GET | /api/coupon/prefix/:prefix | none | router.go:27 |
| GET | /api/coupon/:id | none | router.go:28, handler.go:37 |
| GET | /api/coupon/code/:id | none | router.go:29 |
| POST | /api/coupon/ref_bonus | none | router.go:30 |
| PATCH | /api/coupon/:id | none | router.go:31 |
| DELETE | /api/coupon/:id | none | router.go:32 |
| GET | /api/statement/:id | none (query service/username/code) | router.go:37, statement/handler.go:11 |
| GET | /api/redeem/:code | none (query service,username) — เช็คคูปองก่อน redeem | router.go:43, redeem/handler.go:13 |
| **POST** | **/api/redeem/** | **none** — 💰 ใช้คูปองจริง (redeem) | router.go:44, redeem/handler.go:69 |
| DELETE | /api/redis/flushall | none — ⚠️ flush Redis ทั้งหมด | router.go:49 |
| GET | /api/redis/ | none | router.go:50 |
| DELETE | /api/redis/ | none | router.go:51 |
| GET | /api/access/ | AccessIpOfficeMiddleware (IP check) | router.go:56 |
| PUT | /api/access/ | AccessIpOfficeMiddleware (IP check) | router.go:57 |
| GET | /api/test/demo | none | router.go:62 |
| GET | /api/test/redis | none | router.go:63 |
| POST | /api/migrate/min_condition | none | router.go:68 |
| GET | /metrics | none (Prometheus) | service/prometheus/prometheus.go:12,58 |

### consume
- **ไม่พบ** — ไม่มี `.Consume(`/`QueueDeclare` ที่ทำงานจริง. มีแค่ `_config/mq.go` (`CreateMQ`/`MQReconnector`) แต่ `server.go` ไม่เรียก `CreateMQ()` เลย → RabbitMQ ไม่ถูกต่อ (dead config)

### cron
- **ไม่พบ** scheduler — มีเพียง `go migrate.MigrateTypeWithdraw(resource)` รันครั้งเดียวตอน startup ใน router (`router.go:22`)

## consumes

### http-out
- **ไม่พบ** outbound HTTP client ใด ๆ (grep `http.Get/Post/NewRequest/Client`/`resty` = ว่าง). coupon-service เป็นปลายทางที่ถูกเรียก ไม่เรียกใครต่อ

### publish
- **ไม่พบ** — ไม่มี `MQCh.Publish` ที่ทำงาน (MQ ไม่ถูกต่อ)

### data
DB = `os.Getenv("MONGODB_NAME")`, endpoint `MONGODB_ENDPOINT` (`_config/db.go:36-37`). pool min 3 / max 10 (`_config/db.go:41-42`). ใช้ผ่าน `service/repo/repo.go`.

| collection | R/W | evidence |
|-----------|-----|----------|
| coupon_config | R/W | controller/coupon/service.go:65(W),88(R),110(R),190(W update),752(W bulk); redeem/repo.go:32,55,83,143(R); redeem/service.go:222(W $inc) |
| coupon_statement | R/W | redeem/service.go:191(W create); redeem/repo.go:100,119,167(R) |
| access_ip | R/W | internal/middleware/access_ip.go:65(R); controller/access/service.go:15(R),37(W create),48(W update) |

Redis (`REDIS_URL`/`REDIS_DB`, `_config/rd.go:48-50`): cache `code-<service>-<couponCode>` → coupon ObjectID, TTL 2h เฉพาะ type MANYTIME (`redeem/repo.go:22,38`). มีโครง redis queue (LPush/BRPop/SetNX) ใน `redeem/service.go:77-147` แต่ไม่ถูกใช้ใน flow จริง (ดู observations)

### external
- **ไม่พบ** third-party (ไม่มี Telegram/SMS/payment ภายนอก). พึ่งพาแค่ MongoDB + Redis + Prometheus

## observations
- **redeem/service.go:56-69 (handleRedeemCoupon)** — ลำดับ redeem: `createRedeemCouponStatement` (insert coupon_statement) → แล้วค่อย `updateRedeemCouponConfig` ($inc receive) **ไม่ใช่ transaction** → ถ้า insert สำเร็จแต่ update ล้ม จะนับ receive ไม่ตรง (statement ถูกสร้างแต่ counter ไม่เพิ่ม)
- **service/repo/index.go:40-41,66** — กันซ้ำ (double-redeem) พึ่ง unique index `hash_unique` บน `coupon_statement.hash` (= username+couponID); **แต่ `router.go:21` `// repo.InitCollection(resource)` ถูกคอมเมนต์ออก** → index ไม่ถูกสร้างอัตโนมัติ ถ้า DB จริงไม่มี index นี้ การยิง POST /api/redeem พร้อมกันสองครั้งจะ insert ซ้ำได้ (race / double-redeem)
- **redeem/service.go:131-152 (checkRedeemCouponConfig) + 198-225 (updateRedeemCouponConfig)** — guard `$expr: {$lt: ["$receive","$limit"]}` ทำใน 2 query แยก (check แล้วค่อย update) ไม่ atomic; แม้ update ใช้ filter `$lt receive limit` ช่วยกันเกิน limit ระดับ DB ได้บางส่วน แต่ระหว่าง check→createStatement→update มี gap → เกิน limit ได้ถ้า index ไม่บังคับ
- **POST /api/redeem/ (handler.go:69) ไม่มี auth** — endpoint ใช้คูปองจริงเปิดโล่ง ใครก็ยิงได้ รู้แค่ service+couponID+code+username
- **DELETE /api/redis/flushall (router.go:49) ไม่มี auth** — flush Redis ทั้ง DB ผ่าน HTTP โดยไม่ตรวจสอบสิทธิ์ → DoS/cache poisoning
- **internal/middleware/access_ip.go:20-21** — auth เป็นการเทียบ client IP ตรง ๆ กับ env `ACCESS_IP_OFFICE`; bypass ได้ถ้า client IP = `0.0.0.0` (`client != "0.0.0.0"`); `c.ClientIP()` เชื่อ X-Forwarded-For จึงปลอม header ได้
- **server.go:35-37** — ถ้า `CreateResource()` error แค่ `time.Sleep(5s)` แล้ว `return` (server ไม่สตาร์ท เงียบ ๆ ไม่มี log ชัด); error ถูกกลืน
- **_config/mq.go ทั้งไฟล์** — โค้ด RabbitMQ (CreateMQ/MQReconnector/CloseMQ) มีอยู่แต่ `server.go` ไม่เคยเรียก → dead code; `Resource` struct มี field MQ/MQCh แต่ nil เสมอ 🟡
- **_config/mq.go:14-39** — เช่นเดียวกับ affiliate: retry loop ไม่มีขอบเขต (infinite) ถ้า RabbitMQ ตาย (แต่ไม่ถูกเรียกจึงไม่มีผล)
- **server.go:50, CORS server.go:88-92** — `Access-Control-Allow-Origin: *` + allow methods/headers `*` ทั้ง service
- **redeem/service.go:77-147** — `redisAddQueue`/`redisReceiveQueue` (โครง queue กันชนด้วย Redis SetNX) เป็น dead code: `CreateRedeem` (handler.go:93) เรียก `handleRedeemCoupon` ตรง ๆ ไม่ผ่าน queue → การกัน concurrency ที่ตั้งใจไว้ไม่ถูกใช้งานจริง
- **redeem/handler.go:101-110 (GetRedis) + router.go:62-63** — test endpoint (`/api/test/redis`, `/api/test/demo`) ถูก deploy จริง ไม่มี guard
- **service/version/version.go** — มีแค่ `package version` ว่างเปล่า (ไม่มี version constant) — dead file
- **_config/db.go:39-44** — `mongo.NewClient` + `client.Connect` (deprecated ใน driver ใหม่ ควรใช้ `mongo.Connect`); ไม่มี TLS (โค้ด ca.pem ถูกคอมเมนต์ db.go มี template `connectionStringTemplate` ไม่ถูกใช้)
- **deps เก่า** — Go 1.20 (Dockerfile), `gopkg.in/mgo.v2/bson` ใช้ปนกับ official driver ใน middleware (`access_ip.go:12`), `github.com/streadway/amqp` (deprecated)
- **redeem/repo.go:50-53** — fallback: ถ้า id ที่ cache เป็น zero จะ delete `_id` filter แล้วค้นด้วย code field — logic ซับซ้อนเสี่ยง cache คืนค่าผิด coupon ข้าม service ได้ถ้า key ชน (แต่ key รวม service ไว้แล้ว)
