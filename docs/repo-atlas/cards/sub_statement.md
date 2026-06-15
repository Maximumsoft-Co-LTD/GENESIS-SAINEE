# Card: sub_statement
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go — Gin (เปิด port ไว้เสิร์ฟ Prometheus /metrics เท่านั้น) + RabbitMQ consumer (streadway/amqp) — MongoDB, Redis, OpenTelemetry, Prometheus — subscriber ของ STATEMENT exchange ฝั่ง "side-effect" (notification/promotion stack/covid/mission/member)
- **module**: `statementPub` (go.mod:1, go 1.21 — go.mod:3)
- **entrypoint**: `_cmd/main.go:7` → `statementPub.StartServer()` (server.go:33) → gin + prometheus → `db.CreateResource()` → `resource.CreateMQ()` → `pub.ConnectRabbit` + `pub.StartAMQP` (server.go:44-47, :55-61, :90-91); mode พิเศษ `RECREATE=TRUE` → `reCreateMemberStack` replay deposit ตามช่วงวันที่ (server.go:66-68, :121-165)
- **port**: env `PORT` (server.go:45) — `.env:12 PORT=6000`; ไม่มี route ลงทะเบียนนอกจาก middleware metrics (`p.Use(r)` server.go:40-42) — Dockerfile ไม่มี EXPOSE
- **Dockerfile**: golang:1.21-alpine (habor-proxy.analytichpxv3.online) → alpine:3.14, `CMD ["./app"]` (Dockerfile:2-12)
- **CI**: `.github/workflows/build.yaml`

## provides
### http
| METHOD | path | auth | evidence |
|---|---|---|---|
| GET | `/metrics` (จาก gin prometheus middleware) | ไม่มี | server.go:40-42; metrics/prometheus.go |
| — | ไม่มี business route อื่น (gin.New ไม่มี handler ลงทะเบียน) | — | server.go:37-47 |

### consume
| exchange/queue (exact) | handler | evidence |
|---|---|---|
| exchange `STATEMENT` (fanout, durable) — ถ้า `GROUP=NEWTOPUP` → `STATEMENT_<SERVICE>` ; bind เข้า queue `FRONTENDSUB` / `FRONTENDSUB_<SERVICE>` (durable=false, exclusive=true), **autoAck=true** | แยกตาม `d.Type`: `DEPOSIT` → `notification.Deposit` + `promotion.Deposit` + `covid.Deposit` + `mission.Deposit`; `WITHDRAW` → `notification.Withdraw` + `promotion.Withdraw` + `covid.Withdraw`; `RE-DEPOSIT` → `promotion.Deposit` + `covid.Deposit`; `MEMBER` → `member.Member` (affiliate/webafffiliate/followmember ถูก comment ทิ้ง 🟡) | pub/amqp.go:44-58, :61-77, :79-88, :92-118 |

### cron
ไม่พบ — ไม่มี scheduler ใน executing code (มีเพียง one-shot `reCreateMemberStack` เมื่อ `RECREATE=TRUE` — server.go:66-68)

## consumes
### http-out
| env key | base URL | paths | evidence |
|---|---|---|---|
| `FIREBASEURL` (ว่าง = ข้าม 🟡) | runtime | `PUT /notify/deposit/<username>.json`, `PUT /notify/withdraw/<username>.json` (fire-and-forget `go helper.CallAPI`, timeout 5s) | notification/deposit.go:60-75; notification/withdraw.go:48-60; helper/helper.go:79-83 |
| `LINENOTI` (.env:6 ว่าง 🟡) | runtime | `POST <LINENOTI>/deposit`, `POST <LINENOTI>/withdraw` (แจ้ง LINE user, fire-and-forget) | notification/deposit.go:86-99; notification/withdraw.go:73 |
| hardcode | `https://console.sms-kub.com/api/campaigns` | POST ส่ง SMS แจก ticket/รางวัล (token จาก ticket_config ใน DB) | promotion/deposit.go:1206, :1252-1262 |
| hardcode | `http://sendsms.lupin.host/api/create_sms_log` | log SMS ที่ส่งสำเร็จ (axios.Post fire-and-forget) | promotion/deposit.go:1226 |
| hardcode 🟡 (อยู่ใน `followmember`/`webafffiliate` ซึ่งจุดเรียกถูก comment ใน pub/amqp.go:103-105) | `https://escobar.warmlight.online/api`, `https://v2mootuis.warmlight.online/api`, `https://affiliate.powerdiamonds.xyz/api/refer_deposit`, `/api/refer_withdraw` | affiliate/follow feed | followmember/followmember.go:73-75; webafffiliate/deposit.go:26, :49 |
| `SERVICEAPI` (.env:8 = `http://autogroup-joker.api-th.com/newAPI/api`) 🟡 — ไม่พบจุดใช้ใน executing path ที่ active | — | — | .env:8 |

### publish
| exchange/queue (exact) | payload | evidence |
|---|---|---|
| queue `DYNAMICQUEUE_<SERVICE>` (default exchange, routing key; Type = ชื่อ queue, Expiration=60000, CorrelationId=unix) | `CovidBody` `{type, covid_credit, deposit_statement, withdraw_statement}` — ค่าคอมมิชชั่น covid/affiliate ให้ queue-dynamic-go ประมวลผล | covid/publish.go:33-55; เรียกจาก covid/deposit.go:101, covid/withdraw.go:101 |
| ใช้ channel เดียวจาก `resource.MQCh` (RabbitMQ URL = env `RABBIT_ENDPOINT` — .env:5) | — | _config/mq.go:18-29 |

### data
| DB | collection | R/W | evidence |
|---|---|---|---|
| service DB เดียว (env `MONGODB_ENDPOINT` + `MONGODB_DB_NAME` — ไม่มี office DB / multi-tenant) | `member_stack` | R/W (สะสมยอดฝาก/ถอนรายวันต่อ user — upsert) | _config/db.go:50-51; promotion/deposit.go (18 จุด); server.go:146 |
| service DB | `wallet_statement` | R/W (แจก ticket/โบนัสเข้า wallet) | promotion/deposit.go:588-601 (hash dedup), grep 13 จุด |
| service DB | `ticket_config`, `event_config`, `config_system`, `stack_configs` | R (เงื่อนไขโปรโมชั่น/มิชชั่น/covid) | covid/deposit.go:29; mission/deposit.go; grep |
| service DB | `member_affiliated`, `member_account`, `affiliate_transaction` | R/W | covid/deposit.go:55; affiliate/repository.go |
| service DB | `mission_statement`, `self_stat`, `deposit_statement` | R/W (mission progress; replay อ่าน deposit_statement) | mission/deposit.go; server.go:139 |
| Redis (env `REDIS_URL`, `REDIS_DB`) | cache `covid_use_<SERVICE>` (config_system TTL 24h) | R/W | covid/deposit.go:19-39; helper/redisService.go:51-53 |

### external
| third-party | purpose | auth | evidence |
|---|---|---|---|
| console.sms-kub.com | ส่ง SMS แจ้งรางวัล ticket | header `token` (ค่าจาก DB ticket_config) | promotion/deposit.go:1206, :1262 |
| sendsms.lupin.host | เก็บ log SMS | ไม่มี | promotion/deposit.go:1226 |
| Firebase RTDB (URL จาก env) | realtime notify ฝาก/ถอน | ไม่มี (.json endpoint) | notification/deposit.go:75 |

## observations
1. **autoAck=true + queue exclusive/durable=false** — pod restart = โปรโมชั่น/notification/มิชชั่นของ message ที่ค้างหาย ไม่มี retry/DLQ/recovery cron ใด ๆ (ต่างจาก go_report ที่มี cron ชดเชย) — pub/amqp.go:61-68, :79-88
2. **`failOnError` = `log.Fatalf`** ใน StartAMQP — exchange/queue declare พลาด → process ตาย; แต่ `ExchangeDeclare` เองถูก ignore error (`_ =`) — pub/amqp.go:19-23, :51, :69
3. **MQReconnector reconnect connection แต่ไม่ re-subscribe**: `CreateMQ()` ใหม่แล้ว `recon=false` จบ loop — `StartAMQP` (consumer) ไม่ถูกเรียกซ้ำ → หลัง RabbitMQ rotate ตัว consumer ตายถาวรแต่ process ยังอยู่ (health ดูปกติ) — _config/mq.go:43-53; server.go:90-91
4. **ไม่มี dedup ระดับ message ใน promotion.Deposit**: ใช้ ObjectID ใหม่ทุกครั้ง (`primitive.NewObjectID()` — promotion/deposit.go:74-76) — message DEPOSIT ซ้ำ (เช่น replay จาก RECREATE ที่เช็ค hash เฉพาะ `member_stack.statement_deposit.hash` — server.go:146-153) อาจ stack ยอดซ้ำ; hash dedup มีเฉพาะ ticket ใน wallet_statement — promotion/deposit.go:588-601
5. **`RE-DEPOSIT` วิ่ง promotion.Deposit + covid.Deposit ซ้ำกับ `DEPOSIT`** โดยไม่มี flag กันซ้ำใน handler — pub/amqp.go:112-114
6. **เงิน covid/affiliate ถูกส่งต่อผ่าน `DYNAMICQUEUE_<SERVICE>` ด้วย Expiration=60000** — ถ้า queue-dynamic-go หยุด >60 วิ ค่าคอมหายเงียบ (publish error แค่ println) — covid/publish.go:46-51
7. **`db.CreateResource` error ถูกใช้ก่อนเช็ค**: `resource, err := db.CreateResource(); defer resource.Close()` — ถ้า err (resource=nil) จะ panic ที่ defer/CreateMQ ก่อนถึง if err — server.go:55-61, :70-74
8. **`client.Connect` ignore error (`_ =`)** และ connect timeout 5s เท่านั้น — _config/db.go:74-77, :17
9. **Secrets ใน `.env` ที่ commit**: Mongo Atlas `root:Zxcvasdf789@test02...` (.env:3), CloudAMQP `pbntxwou:gSs995FC...@mustang.rmq.cloudamqp.com` (.env:5)
10. **MaxPoolSize=1** ทั้งที่ handler 4 ตัววิ่งตามลำดับต่อ message — คอขวด Mongo — _config/db.go:66
11. **error เกือบทุกจุดเป็น `fmt.Println` แล้วเดินต่อ/return เงียบ** เช่น unmarshal fail ใน covid (covid/deposit.go:46-50), promotion sms error (promotion/deposit.go:1207-1210)
12. **followmember/webafffiliate/affiliate/analytics เป็น dead code** ที่ยังมี URL+logic ค้าง (จุดเรียก comment) — pub/amqp.go:103-106, :111
13. **handler ทำงาน sequential ใน goroutine เดียว** (notification → promotion → covid → mission ต่อ 1 message, autoAck แล้ว) — message ใหญ่ช้าจะ delay ทั้งคิว — pub/amqp.go:92-118
14. **มี test เฉพาะ promotion/test_case บางส่วน** (callapi_test.go, deposit_test.go) — ส่วน covid/mission/member ไม่มี
