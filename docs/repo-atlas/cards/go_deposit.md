# Card: go_deposit
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go (worker / ไม่มี HTTP server) — RabbitMQ consumer (streadway/amqp), MongoDB (raw driver), Redis (go-redis/v9), Sentry, Jaeger, LINE Notify
- **module**: `godeposit` (go.mod:1, go 1.24.6 — go.mod:3)
- **entrypoint**: `_cmd/main.go:17` → Sentry init (main.go:19) → Redis connect (main.go:36) → `rabbitmq.ConnectRabbit` (main.go:44) → `rabbitmq.Worker` loop (main.go:50) | godotenv โหลด `.env` เมื่อ `SERVICE` ว่าง (main.go:32-34)
- **port**: ไม่พบ — เป็น pure RabbitMQ worker ไม่มี gin/net.Listen ใด ๆ (`_cmd/main.go:17-52` ไม่มี HTTP server; Dockerfile ไม่มี EXPOSE)
- **Dockerfile**: golang:1.24.6 builder → alpine:3.22, `CMD ["./app"]`, ดาวน์โหลด rds-combined-ca-bundle.pem ตอน build (Dockerfile:1-11)
- **CI**: `.github/workflows/pipeline.yml` (build.yaml.disabled ถูกปิดใช้)
- **บทบาท**: จุดรวมของ pipeline ฝากเงิน — กินคิว statement → จับคู่สมาชิก → เติมเครดิตผ่าน game agent API → publish ต่อให้ go_report / sub_statement

## provides
### http
ไม่พบ — repo นี้ไม่เปิด HTTP endpoint (มีแต่ consumer + goroutine loop)

### consume
| exchange/queue (exact) | handler | evidence |
|---|---|---|
| queue = env `QUEUE_NAME` (.env:15 = `DEPOSIT`; แต่ producer ฝั่ง go_sms publish เข้า queue ชื่อ = `<SERVICE>` — sms/que-pub.go:154-156 ของ go_sms) — durable=true, Qos prefetch=1, manual ack (`d.Ack(false)` หลัง process) | แยกตาม `d.Type`: `WALLET-TO-GAME` → `WorkerWalletSystem` (เมื่อ DepositType==2), `UPDATECONFIG` → reload `database_config`+`config_system`, อื่น ๆ (body = bank_statement `_id` hex) → `WorkerDeposit` | rabbitmq/worker.go:61-68, :71-76, :78-86, :106-132 |

### cron (goroutine loop — ไม่ใช่ cron lib)
| schedule | what | evidence |
|---|---|---|
| loop ไม่รู้จบ, sleep สุ่ม 1-7 นาที/รอบ | `CheckStatus` — ดึง `deposit_statement` ย้อนหลัง 48 ชม. status 5/6 (เงินค้าง) → เช็ค balance ที่ agent → ลบ user game → `DepositAgent` เติมเครดิตซ้ำ | deposit/checkstatus.go:17-29, :48, :82, :105-109 |
| ทุก 60 วิ (ping 5 ครั้ง, fail ครบ → panic) | `HealthCheckDatabase` ping Mongo ทุก connection | db/db.go:201-224; เรียกจาก rabbitmq/worker.go:44 |

## consumes
### http-out
| env key / source | base URL | paths | evidence |
|---|---|---|---|
| `configDatabase.APIURL` (จาก DB `database_config` — runtime 🟡) | runtime 🟡 | game agent: `/userbalanceminimul` (agent.go:318), DepositAgent/GetBalanceAgent/DeleteUserGame ฯลฯ ผ่าน `GetAPIAgent` | deposit/agent.go:255-260, :318; deposit/checkstatus.go:47-48 |
| hardcode ใน `GetAPIAgent` สำหรับ service เก่า | `https://918kissauto.api-node.com/newAPI2/agent`, `http://autogroup-918kiss.api-th.com/newAPI/api`, `http://allbet24hr.api-node.com/newAPI/agent7`, `http://pussy888fun.api-node.com/newAPI/agent*`, `https://pussy888win.api-node.com/newAPI/agent-new`, `http://pussy888play.api-node.com/api-new` ฯลฯ | game agent API ต่อ service | deposit/agent.go:28-97 |
| hardcode (SUPERCOMPANY wallet) | `https://sc-prd.asiawallet.net/api` (prod) / `https://supercompany-dev.thesonicblue.xyz/api` (dev) | wallet system | deposit/agent.go:571-573, :592-594 |
| hardcode | `https://notify-api.line.me/api/notify` | แจ้ง error DB connect (LINE Notify) | helpermethod/helper.go:11-23 |
| hardcode | `https://affiliate.powerdiamonds.xyz/api/refer_deposit?...` | tracking affiliate clickid (GET, no timeout) | deposit/deposit.go:1100-1119 |
| `firebaseURL` (จาก config DB — runtime 🟡) | runtime 🟡 | `PATCH /notify/dynamic/<username>.json` (fire-and-forget `go CallAPI`, timeout 10s) | deposit/firbase.go:19-27, :33-34 |
| `configDatabase.CallbackAPI` (จาก DB — runtime 🟡) | runtime 🟡 | callback deposit statement ไประบบลูกค้า + retry รายการ fail | deposit/addcredit.go:50-59; deposit/callback.go |
| hardcode (Sentry) | `https://80e85c485deee4561d73697c304d31d7@o4506229598846976.ingest.sentry.io/4506229940486144` | error tracking | _cmd/main.go:20 |

### publish
| exchange/queue (exact) | payload | evidence |
|---|---|---|
| exchange `STATEMENT` (fanout, durable) — ถ้า `GROUP=NEWTOPUP` → `STATEMENT_<SERVICE>` | Type=`DEPOSIT`, body = `DepositStatementModel` JSON, persistent | deposit/publish.go:33-36, :64-74 (เรียกจาก deposit/addcredit.go:30) |
| queue `OUTSTANDING` (default exchange, routing key) — ถ้า `GROUP=NEWTOPUP` → `OUTSTANDING_<SERVICE>` | Type=`OUTSTANDINGUPDATE`, body = `StatementModel` JSON (ยิงทุกครั้งที่ statement จบแบบไม่สำเร็จ ~20 จุด) | deposit/publish.go:96-99, :116-126; deposit/deposit.go:1157-1170 |
| RabbitMQ URL: `configDatabase.RabbitMQURL` จาก DB; override ด้วย env `RABBIT_MQ_DEVELOPMENT` | dial ใหม่ทุกครั้งที่ publish (amqp.Dial ต่อ message) | rabbitmq/connection.go:37-41; deposit/publish.go:20, :82 |

### data
| DB | collection | R/W | evidence |
|---|---|---|---|
| office DB (hardcode DocumentDB `mongodb://abaoffice:zxcvasdf789@mongodb-aba...:27017` / MOOTUI / ESCOBAR; ถ้า `GROUP=TOPUP|NEWTOPUP` ใช้ env `MONGODB_ENDPOINT`; db = env `DB_OFFICE` default `abaoffice_db`) | `database_config` | R (config ต่อ service + reload ตอน UPDATECONFIG) | db/db.go:64-77, :127; rabbitmq/worker.go:113 |
| service DB (`configDatabase.DbHostname` จาก database_config) | `bank_statement` | R/W (อ่าน statement, update status/comment) | deposit/repository.go:108; deposit/deposit.go:73-76, :115-117 |
| service DB | `deposit_statement` | R/W (เช็คซ้ำ, insert, update status, CheckStatus loop) | deposit/checkstatus.go:24; new_task.go:112, :154; repository.go (19 จุด) |
| service DB | `member_account` | R/W (จับคู่สมาชิก, update static) | deposit/repository.go:171, :193 |
| service DB | `generate_statement` | R (จับคู่ยอดทศนิยม/QR/refCode) | deposit/repository.go:222 |
| service DB | `wallet_statement`, `member_promotion`, `bonus_config`, `member_log`, `games_history`, `game_list`, `game_group`, `bank_list`, `config_system` | R/W | grep Collection(...) ทั้ง repo; rabbitmq/worker.go:50 (`config_system`) |
| Redis (env `REDIS_URL`, `REDIS_PSW`, `REDIS_DB`) | lock กันรายการซ้ำ: key `INSERT_STATEMENT_<hash>` + `<hash>` (SetNX TTL จาก env `DEFAULT_TIME_EXPIRE_REDIS` ชม., default 3) | R/W | redis/connection.go:17-23; deposit/deposit.go:213-248 |

### external
| third-party | purpose | auth | evidence |
|---|---|---|---|
| Game agent APIs (918kiss/allbet/pussy888/rocketwin ฯลฯ) | เติม/เช็คเครดิตเกม | ตาม agent (ใน URL/body) | deposit/agent.go:28-97, :255 |
| LINE Notify | แจ้ง dev เมื่อ DB error | Bearer token hardcode | helpermethod/helper.go:23 |
| Sentry | error tracking | DSN hardcode | _cmd/main.go:20 |
| Firebase RTDB (URL จาก config) | push สถานะ deposit ให้ frontend | ไม่มี auth header (ใช้ .json endpoint) | deposit/firbase.go:27, :42 |
| affiliate.powerdiamonds.xyz | แจ้งยอดฝาก affiliate | ไม่มี auth | deposit/deposit.go:1100-1108 |

## observations
1. **Hardcoded Mongo credentials** ใน source: `mongodb://abaoffice:zxcvasdf789@mongodb-aba.cluster-...docdb.amazonaws.com:27017` + MOOTUI/ESCOBAR variants — db/db.go:64-68; และ `.env` ที่ commit ลง repo มี Atlas root password — .env:2-3, :11
2. **Hardcoded LINE Notify token** `pKahNdB8I8ifXmaeekmD66EbvXSgKMF5hBYUt5bgiNa` — helpermethod/helper.go:23; **Sentry DSN hardcode** — _cmd/main.go:20; JWT จริง (มี password/OTP ของ user) ค้างใน comment — deposit/firbase.go:41
3. **failOnError เป็น no-op** (body ถูก comment ทิ้ง) — channel/queue declare/consume ล้มเหลวจะเดินต่อเงียบ ๆ แล้วค่อยตายตอน `for d := range msgs` — rabbitmq/worker.go:16-20, :57, :69, :76, :88; deposit/publish.go:12-16; new_task.go:20-24
4. **Ack หลัง process แต่ไม่มี retry/DLQ**: `d.Ack(false)` ถูกเรียกเสมอแม้ WorkerDeposit ล้มเหลวภายใน (ทุก error path เป็น return เฉย ๆ) → message หาย ไม่ requeue, ไม่มี dead-letter — rabbitmq/worker.go:127-132
5. **Consume ตั้ง no-wait=true** ผิดตำแหน่ง (ช่อง args ของ amqp Consume คือ noWait=true) — ถ้า broker ตอบ error จะไม่รู้ — rabbitmq/worker.go:78-86
6. **ความเสี่ยงจับคู่ผิดคน (เงินเข้าผิดบัญชี)**: `GetMember` fallback หลายชั้น — จับคู่ด้วยเลขบัญชี 4-8 หลักท้าย + ยอดทศนิยม + เวลา (`generate_statement`) — ถ้าเจอ member ตรงเงื่อนไขเพียง 1 คนจะเติมเครดิตทันที (deposit/member.go:20-110, repository.go:GetMemberByBankNumber) ; เคสชื่อบัญชีซ้ำ/บัญชีเพิ่มใหม่กันไว้เฉพาะเมื่อ env `CHECK_BANK=v2` เท่านั้น — deposit/member.go:113-118
7. **Idempotency พึ่ง Redis SetNX อย่างเดียวสำหรับเคส INSERT_STATEMENT** (`AccBankCode=="XXXXXXXXX"`): hash = วันที่+ยอด+เลขบัญชี (ไม่มีเวลาในแบบ withTime=false) TTL 3 ชม. — ฝากยอดเท่ากันจากบัญชีเดียวกันวันเดียวกัน > 1 ครั้งภายใน TTL จะถูกบล็อกเป็น duplicate / หลัง TTL หมดจะผ่าน — deposit/deposit.go:212-248, :1172-1179; ถ้า Redis ล่ม `SetNX` error → return เงียบ message ถูก ack ทิ้ง — deposit/deposit.go:229-232
8. **เช็ครายการซ้ำใน deposit_statement (`CheckDupDeposit`/`FindStatementInDepositStatementByID`) แล้ว mark status=2 ให้คนตรวจ** — แต่ขั้น "เช็คแล้ว insert" ไม่ใช่ atomic ระหว่าง 2 worker (ป้องกันด้วย Qos=1 ต่อ instance เท่านั้น ถ้า scale หลาย pod ชน) — deposit/deposit.go:112-120, :190-209
9. **Error ถูกกลืนทั่วไฟล์**: log ด้วย `fmt.Println` แล้วเดินต่อ เช่น `checkWhitelist` error แล้วทำต่อ (deposit.go:96-99), `CheckStatusDepositBank` error แล้วทำต่อ (deposit.go:123-126), `GetMember` err ถูกอ่านหลังใช้ `memberSource` ไปแล้ว (deposit.go:151-168 — ใช้ค่า member ก่อนเช็ค err ที่ :170)
10. **ไม่มี timeout**: `trackingDeposit` ใช้ `http.Client{}` ไม่ตั้ง Timeout (deposit/deposit.go:1108), LINE Notify ก็ไม่ตั้ง (helpermethod/helper.go:16); agent CallAPI บางตัว timeout 10s (firbase.go:33-35)
11. **panic ใน production path**: HealthCheckDatabase ping fail 5 ครั้ง → `panic(err)` ทั้ง process (ตั้งใจให้ pod restart แต่ฆ่า in-flight message) — db/db.go:207-211
12. **Reconnect แล้วเรียก Worker ซ้อน**: `Reconnector` → `Connect` + `Worker(...)` ใหม่ โดย goroutine `CheckStatus` ของรอบเก่าไม่ถูกหยุด → CheckStatus loop ซ้อนหลายตัวหลัง reconnect — rabbitmq/connection.go:82-95; worker.go:98
13. **MaxPoolSize=1 (office) / 3 (service)** — คอขวดบน critical path — db/db.go:82, :156
14. **sleep แบบ hardcode ใน loop**: retry get statement ด้วย sleep 5s ครั้งเดียว (deposit.go:68), CheckStatus sleep 1s/รายการ (checkstatus.go:43), maxRetry=500×2s ใน ConnectRabbit แล้ว return เงียบ (connection.go:55-58, main.go:45-48 — process จบโดยไม่ error)
15. **`new_task.go` / `updateconfig/` เป็น `package main` script ปนใน repo** มี service name hardcode `PUSSY888FUN` — new_task.go:32, :47
16. **ไม่มี test ใด ๆ ใน repo** (ไม่มีไฟล์ `*_test.go`)
