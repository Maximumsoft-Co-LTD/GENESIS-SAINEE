# Card: queue-dynamic-go
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack:** Go 1.19 (`go.mod:3`), streadway/amqp v1.0.0, mongo-driver v1.11.4, go-redis/v8, OpenTelemetry OTLP gRPC v1.24.0, telegram-bot-api, ginkgo/gomega tests (`go.mod:5-23`)
- **module:** `queuework` (`go.mod:1`) — **ชื่อ module เดียวกับ queue-withdraw-go** (fork จาก template เดียวกัน)
- **entrypoint:** `_cmd/main.go:11` → `queuework.StartQueue()` → `startQueue.go:17-44` (Mongo → init OTel provider → RabbitMQ → `pub.StartAMQP`)
- **port:** ไม่พบ — pure RabbitMQ worker ไม่มี HTTP server (docker-compose ประกาศ `5052:5052` 🟡 แต่ไม่มีโค้ด listen, `docker-compose.yml:10`)
- **Dockerfile:** `Dockerfile:1-13` golang:1.19.2-alpine3.16 (mirror.gcr.io) → alpine:3.13
- **CI:** `.github/workflows/pipeline.yaml:14-18` ใช้ reusable workflow `Maximumsoft-Co-LTD/ci-cd-workflows@v1.1.3`, image ชื่อ `queue-dynamic`; build.yaml เก่าถูก disable (`.github/workflows/build.yaml.disabled`) 🟡

## provides

### http
ไม่พบ — pure worker

### consume
| exchange/queue (EXACT) | handler | evidence |
|---|---|---|
| queue: `$QUEUE_NAME + "_" + $SERVICE` (default exchange), durable=true, prefetch=1, **auto-ack=true** — ไม่มี .env ใน repo (gitignore) 🟡 ค่า QUEUE_NAME มาจาก runtime; ฝั่ง producer ใน queue-withdraw-go ยิงเข้า `DYNAMICQUEUE_<SERVICE>` (`queue-withdraw-go/helper/wallet.go:92`, `queue-withdraw-go/origin/withdraw.go:1237`) | dispatch ตาม `BobyReceive.Type` | `pub/amqp.go:33-41` (QueueDeclare durable), `pub/amqp.go:57-65` (Consume auto-ack) |
| └ Type=`COVIDAFFILIATE` | `covid.AffiliateSystem` — อัพเดตสถิติ hydra/covid affiliate + ticket realtime | `pub/amqp.go:81-83`, `covid/affiliated.go:23-42` |
| └ Type=`AFFILIATE_LOTTO` (จาก affiliate_standalone ตาม comment) | `covid.AffStatementLotto` — update ads_statement ของ lotto | `pub/amqp.go:84-85` |
| └ Type=`COVID_CREDIT` (จาก sub_statement ตาม comment) | `covid.CovidCreditHandle` — deposit/withdraw กระเป๋าโควิด | `pub/amqp.go:86-87` |
| └ Type=`COVID_ACCOUNT` (จาก topupserie ตาม comment) | `covid.CovidAccountHandle` — เปิด/ปิดบัญชีโควิด | `pub/amqp.go:88-89` |
| └ Type=`COVID_OUTSTANDING` | `covid.CovidOutstandingHandle` — จ่ายยอดค้างทั้งหมด | `pub/amqp.go:90-91` |
| └ Type=`COVID_RESET` (จาก affiliate_standalone ตาม comment) | `covid.CovidResetCredit` — รีเซ็ตรายอาทิตย์ | `pub/amqp.go:92-93` |
| └ Type=`HYDRA_UFA_AFFILIATE` | `covid.AffiliateSystemHydraUFA` — update ads_statement ของ ufa | `pub/amqp.go:94-95` |
| └ `DynamicQueue=true` (field ใน body) | `queue.ReceiveQueueLog` — upsert `queue_log` | `pub/amqp.go:96-98`, `queue/log.go:14-46` |
| RPC reply: ถ้า `d.ReplyTo != ""` publish response กลับ default exchange พร้อม CorrelationId, Expiration 60000 | — | `pub/amqp.go:103-120` |

### cron
ไม่พบ — มีแต่ `service.ReAdsStatement` ที่ถูก comment ไว้ 🟡 (`startQueue.go:40`, ตัวฟังก์ชัน `service/reHydra.go:12-39` one-shot ไม่ใช่ cron)

## consumes

### http-out
| env key | base URL | paths | evidence |
|---|---|---|---|
| `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` | Telegram Bot API (ผ่าน tgbotapi lib) | sendMessage ตอน connect สำเร็จ และทุก ๆ retry 180 ครั้ง | `pub/connection.go:50-57,62-71`, `helper/helper.go:93-101` |
| `OTEL_CONNECT` | OTLP gRPC collector (insecure, WithBlock) | trace export | `otel/provider.go:39-43` |
| (ไม่พบ HTTP client อื่น — ไม่มี axios/net/http ใน executing code) | — | — | grep ทั้ง repo พบเฉพาะข้างต้น |

### publish
| exchange/queue (EXACT) | payload | evidence |
|---|---|---|
| `d.ReplyTo` (RPC response, default exchange) | ResponseQueue JSON | `pub/amqp.go:105-115` |
| ไม่พบการ publish ไป queue อื่น | — | — |

### data
| DB | collection | R/W | evidence |
|---|---|---|---|
| MongoDB เดี่ยว (`MONGODB_ENDPOINT`/`MONGODB_DB_NAME`, pool max=1, godotenv.Load) | `ads_statement` | R/W (สถิติ affiliate: inc adv_list.*, list_topup, list_first_deposit ฯลฯ) | `_config/db.go:38-51`, `covid/affiliated.go:202-260,319-339,636-693,774-807,1111-1522,3452-3945,4030-4204`, `service/reHydra.go:36` |
| | `deposit_statement` | R/W (set hydra_status, covid_status) | `covid/affiliated.go:267-274,700-707,902-927,1093,1228,3543,3825-3874` |
| | `withdraw_statement` | R/W | `covid/affiliated.go:346-352,801-807,1245-1316,1394,3627` |
| | `wallet_statement` | R/W (กระเป๋า COVID_WALLET / ticket) | `covid/affiliated.go:1821-1944,2056-2232,2404-2859,2982-3400`, `helper/helper.go:67` |
| | `campaign` | R | `covid/affiliated.go:49,188,300,368,621,754,826,1029,1296,3957-3967,4066-4134` |
| | `config_system` | R | `covid/affiliated.go:2329,2436,3001` |
| | `ticket_type` / `ticket_config` / `ticket_pay` | R/W | `covid/affiliated.go:1635-1639,1879-1932,1897` |
| | `queue_log` | R/W (update status=1 หรือ insert status=2) | `queue/log.go:23,38` |
| Redis (`REDIS_URL`, `REDIS_DB`, no password) | cache `campaign`, covid config (SetEX) | R/W | `service/redisService.go:50-60`, `covid/affiliated.go:2320,2995-3008,3958-3974` |

### external
| third-party | purpose | auth | evidence |
|---|---|---|---|
| Telegram Bot API | แจ้งสถานะ connect/retry RabbitMQ | env `TELEGRAM_BOT_TOKEN` | `pub/connection.go:50-67` |
| OTel collector | tracing | insecure gRPC | `otel/provider.go:39-43` |

## observations
- **auto-ack=true** (`pub/amqp.go:60`) — message ถูก ack ทันทีที่ส่งถึง consumer; process ตาย/handler error = ข้อมูลสถิติ affiliate/เครดิตโควิดหายถาวร ไม่มี retry/DLQ ใด ๆ (มี `d.Ack(false)` ถูก comment ไว้ `pub/amqp.go:121` 🟡)
- **error จาก handler ถูกกลืนทั้งหมด:** `AffiliateSystem` แค่ `fmt.Println` err ของ handleHydraStatV2/handleCovidStat/TicketServiceRealtime แล้วตอบ "อัพเดตสำเร็จ" เสมอ (`covid/affiliated.go:26-41`)
- **Unmarshal error ไม่เช็คเลย:** `pub/amqp.go:75` ค่า err จาก json.Unmarshal ไม่ถูกตรวจก่อนใช้ BobyReceive
- **RPC reply error ทำ goroutine consumer ตายเงียบ:** `pub/amqp.go:116-119` ถ้า Publish reply fail จะ `return` ออกจาก for-loop goroutine → service หยุดบริโภค message แต่ process ไม่ตาย (forever channel ค้าง `pub/amqp.go:124-125`)
- **เงินกระเป๋าโควิดไม่มี lock/idempotency:** wallet_statement ใช้ pattern อ่านล่าสุดแล้ว insert แบบเดียวกับ queue-withdraw-go (`helper/helper.go:60-78` GetUserPocket → insert ใน `covid/affiliated.go:1821-1944,2056+`) ไม่มี transaction; พึ่ง prefetch=1 + instance เดียวเท่านั้น (`pub/amqp.go:47-51`)
- **code duplication กับ queue-withdraw-go:** module `queuework` เดียวกัน (`go.mod:1` ทั้งสอง), `_cmd/main.go` ต่างกัน 1 บรรทัด, `_config/db.go` ต่างกัน 10/82 บรรทัด (~94% identical — ฝั่งนี้เพิ่ม godotenv.Load, ลด pool 5→1, timeout 15→5s), `pub/connection.go` โครงเดียวกัน (ฝั่งนี้เพิ่ม Telegram noti + retry ไม่จำกัด vs maxRetry 200), `pub/amqp.go` โครง StartAMQP เดียวกันแต่ dispatch คนละชุด และ `helper/helper.go` ตัด HTTP helpers ออก (`Bod/Eod/TimeFormat/GetUserPocket/ToFixed` copy ตรงกัน `queue-dynamic-go/helper/helper.go:24-91` ↔ `queue-withdraw-go/helper/wallet.go:62-77`, `queue-withdraw-go/helper/helper.go:272-380`) — ประเมินไฟล์โครงสร้างพื้นฐานเหมือนกัน ~70-95% ต่อไฟล์, business logic (covid/ vs origin/wallet/wheel/) คนละชุดโดยสิ้นเชิง
- **`Eod` bug ต่างจาก fork ต้นทาง:** nanosecond = 99 แทน 999ms (`helper/helper.go:31` vs `queue-withdraw-go/helper/helper.go:379` ที่ใช้ `999*int(math.Pow10(6))`) — ขอบเวลา end-of-day สั้นกว่าที่ตั้งใจ
- **covid/affiliated.go ไฟล์เดียว 4,215 บรรทัด** (wc -l) รวม business logic 8 ประเภท message — แก้ไข/ทดสอบยาก
- **OTel fail แล้วไปต่อ + อาจ panic ตอน shutdown:** ถ้า `InitProvider` error จะ print แล้วเรียก `shutdown(ctx)` ทั้งที่ shutdown เป็น nil (`startQueue.go:30-38`) → nil pointer dereference เมื่อ OTEL_CONNECT ต่อไม่ได้
- **gRPC `WithBlock()` + timeout 60s** (`otel/provider.go:37-43`) — ถ้า collector ไม่ขึ้น service start ช้า/ค้าง 60 วิ
- **retry connect RabbitMQ ไม่มีเพดาน** (`pub/connection.go:38-75` วน infinite, แจ้ง Telegram ทุก 180 ครั้ง) — ดีกว่า fork ต้นทางแต่ spam Telegram ได้
- **hardcoded date ใน ReAdsStatement:** `time.Date(2023, 12, 29, ...)` (`service/reHydra.go:15`) — one-off migration code ที่ยังอยู่ในโค้ด (ถูก comment ที่ call-site `startQueue.go:40` 🟡)
- **mongo pool MaxPoolSize=1** (`_config/db.go:50`) — คอขวดเมื่อ handler เดียวยิงหลาย query ต่อ message
- **deps เก่า:** Go 1.19 (EOL), streadway/amqp archived (`go.mod:14`), alpine:3.13 base image EOL (`Dockerfile:6`)
- **ไม่มี .env/.env.example ใน repo** — config ทั้งหมด (QUEUE_NAME, SERVICE, RABBITMQURL, MONGODB_*, REDIS_*, OTEL_CONNECT, TELEGRAM_*, COVID_USE) inject ตอน runtime 🟡 ทราบชื่อ queue จริงได้จากฝั่ง producer เท่านั้น
