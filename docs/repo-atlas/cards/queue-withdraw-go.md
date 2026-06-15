# Card: queue-withdraw-go
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack:** Go 1.22.4 (`go.mod:3`), streadway/amqp v1.0.0, mongo-driver v1.10.2, go-redis/v8, jaeger-client-go (opentracing), go-axios, telegram-bot-api (`go.mod:5-18`)
- **module:** `queuework` (`go.mod:1`) — ชื่อ module ชนกับ queue-dynamic-go
- **entrypoint:** `_cmd/main.go:10` → `queuework.StartQueue()` → `startQueue.go:14-28` (connect Mongo → connect RabbitMQ → `pub.StartAMQP`)
- **port:** ไม่พบ — pure RabbitMQ worker ไม่มี HTTP server (ไม่มี gin/net.Listen ใน executing code; `_cmd/main.go:8-11` มีแค่ StartQueue)
- **Dockerfile:** `Dockerfile:1-11` golang:1.22-alpine3.20 → alpine:3.20, build `_cmd/main.go`, ดาวน์โหลด rds-combined-ca-bundle.pem ตอน build (`Dockerfile:10`)
- **CI:** `.github/workflows/build.yaml:4` image ชื่อ `topupserie-withdraw` (ชื่อ image ไม่ตรงชื่อ repo), Kaniko → JFrog (`build.yaml:53-65`), deploy ผ่าน `API_UPDATE_IMAGE` webhook (`build.yaml:86`)

## provides

### http
ไม่พบ — pure worker

### consume
| exchange/queue (EXACT) | handler | evidence |
|---|---|---|
| queue: `$QUEUE_NAME + "_" + $SERVICE` (default exchange; .env ตั้ง `QUEUE_NAME=WALLETWITHDRAW` → เช่น `WALLETWITHDRAW_DEMO-DEV`), durable=false, prefetch=1, manual ack | dispatch ตาม `d.Type` | `pub/amqp.go:45-52` (QueueDeclare), `pub/amqp.go:78-86` (Consume), `.env:60` (QUEUE_NAME=WALLETWITHDRAW), `.env:14` (SERVICE=DEMO-DEV) |
| └ Type=`ORIGINWITHDRAW` | `origin.Withdraw` — ถอนเครดิตหลัก (ตรวจ statement, ตัดเครดิต agent, สร้าง withdraw_statement, ถอนออโต้) | `pub/amqp.go:121-123`, `origin/withdraw.go:26` |
| └ Type=`WALLETWITHDRAW` | `wallet.WithdrawWalletV2` — ถอนจากกระเป๋ากิจกรรม/แต้ม เข้าบัญชีธนาคาร + ลบ redis key `wallet_withdraw_<username>` | `pub/amqp.go:124-144`, `wallet/withdraw.go:409` |
| └ Type=`EVENTACTION` | `event.EventWallet` — เติม POINT/CREDIT จากกิจกรรม | `pub/amqp.go:145-147`, `event/wallet.go:17` |
| └ Type=`WHEELEVENT` | `wheel.CallBackWheel` — จ่ายรางวัลกงล้อ | `pub/amqp.go:148-150`, `wheel/withdraw.go` |
| RPC reply: publish ผลลัพธ์กลับไปที่ `d.ReplyTo` (default exchange) พร้อม CorrelationId, Expiration 60000ms | — | `pub/amqp.go:160-170` |

### cron
ไม่พบ

## consumes

### http-out
| env key / source | base URL | paths | evidence |
|---|---|---|---|
| hardcoded | `https://monitor.thezeus.online` | GET `/api/webQueue/<SERVICE>/<type>` ทุก message (fire-and-forget goroutine) | `pub/amqp.go:120` |
| `SERVICE_API` (🟡 .env ว่าง, docker-compose ใส่ `https://fullslot-test.api-node.com`) | agent API | GET `/withdraw?key=&point=`, GET `/deposit?key=&point=`, GET `/add_member?`, GET `/userinfo?key=`, GET `/transfer/transaction`, GET `/seamless/sumtransaction`, GET `/seamless/sumtransactioncommission`, POST `/usergamedelete`, GET `/outstanding/<user>`, POST `/outstanding` | `helper/agent.go:111,171,199,275,315,369,415,509`, `origin/withdraw.go:1340-1344,1362-1366`, `docker-compose.yml:14` |
| hardcoded ใน `GetAPIAgent` (เลือกตาม SERVICE + date_regis) | `http://autogroup-918kiss.api-th.com`, `http://allbet24hr.api-node.com`, `http://allbet.abagroup.api-url.com`, `http://pussy888fun.api-node.com`, `http://pussy888.abagroup.api-url.com`, `http://autogroup-pussy888win.api-th.com`, `http://pussy888play.api-node.com` (สุ่มเลือกด้วย `rand.Intn`) | `/newAPI/agent*`, `/newAPI/api*` | `helper/agent.go:618-685`, เรียกใช้จริงที่ `origin/withdraw.go:42` |
| Mongo `config_system.db_config.office_api` (runtime จาก DB 🟡) | office API ของแต่ละ tenant | POST `/api/auto/createQueueWithdraw/<SERVICE>` (สั่งถอนออโต้, auth header `apiKey` = `db_config.callback_api_key`) | `wallet/withdraw.go:394-395,841-842`, `origin/withdraw.go:1195-1196`, `helper/helper.go:239` |
| `FIREBASEURL` (🟡 .env ว่าง) / `FIREBASEURLMONITOR`=`https://withdraw-monitor-new.firebaseio.com` | Firebase RTDB | PUT `<SERVICE>/notification/TRANSECTION_WITHDRAW.json`, GET/PUT `/notify/event/<user>.json`, `/notify/slipwithdraw/slip.json` | `helper/noti.go:44-91`, `helper/helper.go:288-293`, `slip/slipWithdraw.go:440-470`, `.env:54-55` |
| hardcoded | `https://notify-api.line.me/api/notify` (LINE Notify, Bearer token จาก `config_system.line_notify_access_token.withdraw`) | POST | `helper/helper.go:307-321`, `wallet/withdraw.go:284-287` |
| Telegram Bot API (ผ่าน tgbotapi) | bot token hardcode `7546574244:AAHX...` | sendMessage ไป `config_system.telegram_noti.withdraw_noti.chat_id` | `helper/helper.go:382-398`, `wallet/withdraw.go:288-292` |
| `SOCKETIOURL` (🟡 ไม่มีใน .env) | socket notify | GET `/mail/socket/auto/dynamic?...` | `helper/helper.go:295-300` |
| `APIWHEEL`=`http://wheel.api-node.com/login/` (🟡 ประกาศใน .env แต่ไม่พบการอ่านใน .go), `API_QR`=`http://payment.api-node.com/v1` (🟡 เช่นกัน) | — | — | `.env:39,51` (ไม่มี `os.Getenv("APIWHEEL"/"API_QR")` ในโค้ด) |

### publish
| exchange/queue (EXACT) | payload | evidence |
|---|---|---|
| exchange `STATEMENT_<SERVICE>`, routing key `DYNAMICQUEUE_<SERVICE>`, Type=`DEPOSIT`, persistent | DepositQueModel (สำเนา deposit_statement) → ผู้บริโภคคือ queue-dynamic-go | `helper/wallet.go:92-105` (PublishStatement), เรียกที่ `wallet/withdraw.go:659-660` |
| default exchange → queue `DYNAMICQUEUE_<SERVICE>`, Type=covid.Type, Expiration 60000 | CovidBody | `origin/withdraw.go:1224-1250` (PublishCovid) |
| `d.ReplyTo` (RPC response) | ResponseQueue JSON | `pub/amqp.go:160-170` |

### data
| DB | collection | R/W | evidence |
|---|---|---|---|
| MongoDB เดี่ยว (`MONGODB_ENDPOINT`/`MONGODB_DB_NAME`, pool 1-5) | `wallet_statement` | R/W (FindOne, UpdateMany, InsertOne) | `_config/db.go:44-54`, `wallet/withdraw.go:877,888,900`, `wheel/withdraw.go:215-246` |
| | `withdraw_statement` | R/W (InsertOne, UpdateOne status=12) | `wallet/withdraw.go:912`, `origin/withdraw.go:958,1007,1031`, `slip/slipWithdraw.go:146,274` |
| | `deposit_statement` | R/W | `wallet/withdraw.go:950`, `origin/withdraw.go:95,350,796,812,975,996,1018` |
| | `event_statement` | W | `wallet/withdraw.go:937`, `event/wallet.go:47,90`, `wheel/withdraw.go:257` |
| | `member_account` | R | `wallet/withdraw.go:863`, `origin/withdraw.go:949,1044` |
| | `member_log` | W | `helper/helper.go:187-205` |
| | `config_system` | R | `wallet/withdraw.go:282,380`, `origin/withdraw.go:549`, `helper/redis.go:75` |
| | `member_affiliated` | R | `helper/helper.go:128`, `origin/withdraw.go:271` |
| | `lotto_callback` | R (aggregate) | `helper/helper.go:361` |
| | `callback_min_amount` | W | `origin/withdraw.go:655` |
| Redis (`REDIS_URL`, `REDIS_DB`, no password) | key `wallet_withdraw_<username>` (DEL), `config-system` (GET/SET TTL 1h) | R/W/D | `helper/redis.go:89-99`, `pub/amqp.go:134-144`, `helper/redis.go:61-87` |

### external
| third-party | purpose | auth | evidence |
|---|---|---|---|
| LINE Notify | แจ้งเตือนรายการถอน | Bearer token จาก config_system | `helper/helper.go:307-321` |
| Telegram Bot API | แจ้งเตือนรายการถอน | bot token hardcode ในโค้ด | `helper/helper.go:383` |
| Firebase Realtime DB | แจ้งเตือน withdraw monitor / slip | URL .json ไม่มี auth ใน URL | `helper/noti.go:73-93`, `slip/slipWithdraw.go:440-470` |
| monitor.thezeus.online | นับ traffic ทุก message | ไม่มี auth | `pub/amqp.go:120` |
| Game agent APIs (918kiss/allbet/pussy888/...) | เช็คเครดิต + ตัดเครดิตเกม | query string `?key=<username>` ไม่มี auth header | `helper/agent.go:111-509,618-685` |
| Jaeger (config FromEnv, sampler const=1) | tracing | — | `jaeger/jaeger.go:27-33` |

## observations
- **secrets committed:** `.env` ถูก commit พร้อม MongoDB Atlas prod-like credentials `mongodb+srv://root:Zxcvasdf789@test02.9oh7e...` (`.env:7`), `MONGODB_PASSWORD=Aa112233` (`.env:3`), `ACCESS_SECRET=ABAsercretPayload` (`.env:10`)
- **hardcoded Telegram bot token** `7546574244:AAHXLS_9dP1Wj3IZyApkUzB8TIr1talel6g` ใน `helper/helper.go:383` (GetBotTokenGeneral)
- **hardcoded JWT ใน comment** `helper/helper.go:168` มี Bearer token จริง (มี password และข้อมูลลูกค้าใน payload) ถูก comment ไว้ 🟡
- **wallet withdraw ไม่มี lock/idempotency:** `wallet/withdraw.go:435-457` อ่านยอดจาก `wallet_statement` ล่าสุด (FindOne sort datetime:-1, `wallet/withdraw.go:870-882`) แล้ว insert รายการหักเงิน (`wallet/withdraw.go:562`) แบบ read-then-write ไม่มี transaction/findAndModify — กัน double-spend ด้วย Redis key `wallet_withdraw_<username>` ที่ "ลบ" หลังประมวลผลเท่านั้น (`pub/amqp.go:134-144`; ฝั่ง set อยู่ repo อื่น) + prefetch=1 (`pub/amqp.go:63-67`) — ถ้ารัน replica >1 ตัวบน queue เดียวกันจะแข่งกันได้
- **hash กันซ้ำ resolution แค่ระดับนาที:** hash = sha1(amount+type+user+TimeFormat("minute")) (`wallet/withdraw.go:540-541`) และ `TimeFormat("minute")` ใส่ Hour ซ้ำสองครั้งแทน (bug: `fmt.Sprint(timeNow.Hour())+fmt.Sprint(timeNow.Hour())+fmt.Sprint(timeNow.Minute())` `helper/helper.go:280`)
- **ack/retry/DLQ:** มี manual ack + Nack(requeue=false) เมื่อ timeout 60s (`pub/amqp.go:188-197`) แต่ Nack ไม่ requeue และไม่มี DLQ ประกาศ → message หายเงียบ; งานที่ timeout ยังรันต่อใน goroutine (ไม่ถูก cancel จริง — ctx ไม่ได้ส่งเข้า handler, `pub/amqp.go:104-186`) → เสี่ยงถอนซ้ำเมื่อ message ถูกประมวลผลค้างหลัง Nack
- **queue ไม่ durable** (`pub/amqp.go:47` durable=false) — RabbitMQ restart แล้วข้อความถอนเงินหาย
- **Unmarshal error แล้วไปต่อ:** `pub/amqp.go:107-111` json.Unmarshal พังแค่ print แล้วประมวลผล struct ว่างต่อ (comment `// continue`)
- **swallowed errors:** insert `event_statement` fail แค่ log แล้วไปต่อ (`wallet/withdraw.go:786-794`); insert `deposit_statement` fail ก็ publish ไป DYNAMICQUEUE ต่อ (`wallet/withdraw.go:650-660` — return ถูก comment ที่ 657); `PublishStatement`/`PublishCovid` กลืน error ทุกจุด (`helper/wallet.go:82-84,106-109`, `origin/withdraw.go:1227-1229`)
- **Reconnector เรียก StartAMQP ซ้อน recursive** (`pub/connection.go:66-81`) และ retry connect สูงสุด 200 ครั้งแล้ว return เงียบ ๆ process ค้างไม่ตาย (`pub/connection.go:38-41`); `conn.Config.Heartbeat` ถูก set หลัง Dial ซึ่งไม่มีผล + อ่าน conn ก่อนเช็ค err → panic ได้ถ้า Dial fail (`pub/connection.go:43-44`)
- **เลือกบัญชีธนาคารตัวสุดท้ายใน loop ไม่ใช่ตัวที่ user เลือก** (`wallet/withdraw.go:481-490` วน BankList ทับค่า currentBank เรื่อย ๆ)
- **agent URL เลือกแบบ `rand.Intn`** ระหว่างสองโดเมน (`helper/agent.go:632-643,653-664`) — ผลถอนไม่ deterministic
- **no timeout:** axios.Get/Post ที่ `helper/agent.go`, `origin/withdraw.go:1344,1366`, `pub/amqp.go:120` ใช้ default instance ไม่ตั้ง timeout; `helper.CallAPI` (`helper/helper.go:161`) http.Client{} ไม่มี Timeout
- **เทียบ queue-dynamic-go:** module ชื่อ `queuework` เดียวกัน (`go.mod:1` ทั้งคู่), โครงโค้ดเดียวกันชัดเจน — `_cmd/main.go` ต่างกัน 1 บรรทัด, `_config/db.go` ต่างกัน 10/82 บรรทัด (~94% เหมือน), `pub/connection.go` skeleton เดียวกัน (ฝั่ง dynamic เพิ่ม Telegram noti), `pub/amqp.go` โครง StartAMQP/QueueDeclare/Qos/Consume + RPC reply เหมือนกันแต่ dispatch คนละชุด handler, `helper.GetUserPocket`/`Bod`/`Eod`/`TimeFormat` ซ้ำกันแบบ copy (`queue-withdraw-go/helper/wallet.go:62-77` ↔ `queue-dynamic-go/helper/helper.go:60-78`) — เป็น fork คนละ worker จาก template เดียวกัน ไม่ใช่ repo ซ้ำทั้งก้อน
- **CI image ชื่อ `topupserie-withdraw`** ไม่ตรงชื่อ repo (`.github/workflows/build.yaml:4`) — สับสนเวลา trace deploy
- **deps เก่า:** streadway/amqp (archived, no maintenance) `go.mod:12`, jaeger-client-go (deprecated → OTel) `go.mod:13`, ioutil (deprecated) `helper/helper.go:178`
- **dead env:** `APIWHEEL`, `API_QR`, `WALLETWITHDRAW=200`, `PERSEN_AFFILIATED` ฯลฯ ประกาศใน `.env:20-52` แต่ไม่มี `os.Getenv` ที่อ่านใน repo นี้ 🟡
