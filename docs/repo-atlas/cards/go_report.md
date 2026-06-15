# Card: go_report
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go (worker / ไม่มี HTTP server) — RabbitMQ consumer (streadway/amqp), MongoDB, Redis, gocron, Telegram bot, Jaeger — ตัว sink ของ STATEMENT exchange เพื่อสร้าง report aggregate
- **module**: `goreport` (go.mod:1, go 1.15 — go.mod:3)
- **entrypoint**: `_cmd/main.go:15` — godotenv โหลดเสมอ (main.go:16); ถ้า env `TYPE=GET_OLD` → migrate mode `reportOld.ReportOld()` (main.go:23-24); ปกติ → connect DB/Redis → `CronjobRecovery` goroutine → `SubscribeOutstanding` + `Subscribe` (main.go:40, :54-55)
- **port**: ไม่พบ — ไม่มี HTTP listener ใน executing code (เฉพาะ consumer + cron); Dockerfile ไม่มี EXPOSE
- **Dockerfile**: golang:1.18-alpine3.16 (ผ่าน habor-proxy.analytichpxv3.online) → alpine:3.13, `CMD ["./app"]` (Dockerfile:1-13)
- **CI**: `.github/workflows/build.yaml`; มี `cloudbuild.yaml` (GCP Cloud Build) ด้วย

## provides
### http
ไม่พบ

### consume
| exchange/queue (exact) | handler | evidence |
|---|---|---|
| exchange = env `EXCHANGE_NAME` (.env:14 = `STATEMENT_DEMO-DEV` — pattern `STATEMENT_<SERVICE>`; fanout, durable) bind เข้า queue = env `QUEUE_NAME` (.env:6 = `DYNAMICQUEUE`; durable=false, exclusive=true), Qos=1, **autoAck=true** | แยกตาม `d.Type`: `DEPOSIT` → `ReportActiveUser`; `WITHDRAW` → `ReportWithdrawActive` + `ReportWithdraw`; `BANK` → `ReportStatement`; `MEMBER` → `ReportMember`; `LOGIN` → `ReportMemberLogin`; `MIGRATECOUPONV2` → `MigrateCouponV2` | rabbitmq/subscribe.go:60-69, :71-93, :95-104, :137-159 |
| queue `OUTSTANDING_<SERVICE>` (ไม่มี SERVICE → `OUTSTANDING`; default exchange, durable=false, exclusive=true, **autoAck=true**) | `d.Type == "OUTSTANDINGUPDATE"` → `reportOld.UpdateTodayOutstanding` (push datetime ลง Redis list แล้ว debounce recalculation) | rabbitmq/subscribe.go:181-184, :216-241, :277-281; reportOld/cronjob.go:241-277 |

### cron (gocron — start ใน CronjobRecovery เมื่อ `TYPE != SUB_OLD`)
| schedule | what | evidence |
|---|---|---|
| ตอน start (เมื่อ GROUP ไม่ใช่ TOPUP/AUTO/ABA) | `ScriptDashboard` → `StartRecovery("SCRIPT")` recompute ย้อนหลัง 7 วัน | reportOld/cronjob.go:27-29, :223-233 |
| ทุก 1 ชม. | `StartRecovery("CRONJOB")` → `ReportRemain` ย้อนหลัง 1 วัน (กันรายงานหลุดจาก autoAck) | reportOld/cronjob.go:35, :55-72 |
| ทุกวัน 02:00 | `ManageDuplicateReportActiveUser` — ลบ report ซ้ำ | reportOld/cronjob.go:36 |
| ทุกวัน 02:20 | `UpdateReportNotEqualStatement` | reportOld/cronjob.go:37 |
| ทุก 5 นาที | `outStandingUpdateReport2` — อ่าน Redis list `outstanding_deposit_statement` (สูงสุด 5000) → recompute `OutStandingReportDeposit2` → ลบ list | reportOld/cronjob.go:38, :74-206 |

## consumes
### http-out
| env key / source | base URL | paths | evidence |
|---|---|---|---|
| hardcode 🟡 (จุดเรียก `reportSMSDeposit` ถูก comment ที่ report/deposit.go:243 — ฟังก์ชันไม่ถูกเรียกจาก executing path) | `http://mongo-sms.api-node.com` | `POST /v1/logSMSByPhoneNumber/<SERVICE>` | report/deposit.go:346-352 |
| Telegram Bot API (ผ่าน tgbotapi lib) | api.telegram.org | sendMessage แจ้งผล script/error | report/script.go:2544-2560 |

### publish
ไม่พบ — repo นี้ไม่ publish ไป RabbitMQ (grep `ch.Publish` ไม่มีใน executing code; เป็น consumer-only sink)

### data
| DB | collection | R/W | evidence |
|---|---|---|---|
| office (hardcode DocumentDB `mongodb://abaoffice:zxcvasdf789@mongodb-aba...:27017`; `GROUP=TOPUP|NEWTOPUP` → env `MONGODB_ENDPOINT`, db = env `DB_OFFICE`) | `database_config` | R (หา RabbitMQURL/DbHostname ต่อ service) | db/db.go:58-64; db/db.go:184, :252 |
| service DB | `deposit_statement` | R/W (อ่าน + `UpdateStatusReport` set `status_report:1` เป็น dedup gate) | report/repository.go:242-266; report/deposit.go |
| service DB | `withdraw_statement` | R/W | report/withdraw.go; grep 18 จุด |
| service DB | `bank_statement` | R/W (`ReportStatement` mark status_report) | report/statement.go:24-27 |
| service DB | `report_active_user_day/hour/month/week` | W (upsert `$inc`/`$push`) | report/deposit.go:245-255, :330-338 |
| service DB | `report_bonus_day/hour/month/week`, `report_day`, `report_hour` | W (upsert) | report/deposit.go:245-255; report/statement.go:42-43 |
| service DB | `member_account`, `report_sms`, `config_system`, `bonus`, `coupon` | R/W (member report, coupon migrate, script) | report/member.go; report/script.go |
| Redis (env `REDIS_URL`) | list `outstanding_deposit_statement` (คิวงาน recompute outstanding) | R/W | helper/redis.go:184; reportOld/cronjob.go:93-132, :196-201 |

### external
| third-party | purpose | auth | evidence |
|---|---|---|---|
| Telegram bot | แจ้งเตือน script/error เข้า group chat `-4200406589` | bot token hardcode `6792312572:AAGM3ol8PDQ_pA8cRkh7EQk_CgyNkrvRHc8` | report/script.go:2549-2554 |

## observations
1. **autoAck=true ทั้ง 2 consumer** (มี comment ไทยยอมรับเองว่า "อาจต้องปิด auto-ack เป็น false") — process ตายกลาง handler = report หาย ต้องพึ่ง cron recovery รายชั่วโมงชดเชย — rabbitmq/subscribe.go:98, :236
2. **queue exclusive=true + durable=false** — restart/reconnect แล้ว queue หาย message ระหว่างดับสูญทั้งหมด (ออกแบบให้ cron ชดเชย แต่ window สูงสุด 1 ชม.) — rabbitmq/subscribe.go:71-78, :216-223
3. **Idempotency ฝั่ง DEPOSIT/BANK ใช้ `UpdateStatusReport` (set `status_report=1`, ModifiedCount==1 จึงประมวลผล)** — กันซ้ำได้เฉพาะ message ที่ document เดิมยังไม่ mark; แต่ `ReportRemain`/recovery วิ่งซ้ำบนข้อมูลเดิม → มี cron `ManageDuplicateReportActiveUser` 02:00 ไว้ลบรายงานซ้ำทุกวัน (ยอมรับว่าซ้ำเกิดได้) — report/repository.go:242-266; reportOld/cronjob.go:36
4. **`failOnError` = `log.Fatalf`** — declare exchange/queue พลาดครั้งเดียว process ตายทันที (ต่างจาก go_deposit ที่กลืนเงียบ) — rabbitmq/subscribe.go:16-20
5. **Hardcoded Telegram bot token + chat id** ใน source (รวม token ตัว test ใน comment) — report/script.go:2549-2552
6. **Hardcoded office DocumentDB credentials** — db/db.go:58; `.env` commit ลง repo มี Mongo Atlas URI `mongodb+srv://root:root@test...` — .env:5 และ RabbitMQ guest URL — .env:8
7. **`SubscribeOutstanding` ประกาศ `queueExchange := "OUTSTANDING" + service` ก่อนอ่าน env** — ถ้า SERVICE ว่างได้ชื่อ `OUTSTANDING` (ไม่มี `_`) ชนกันข้าม service — rabbitmq/subscribe.go:181-185
8. **sleep 30s ก่อน connect RabbitMQ ทุกครั้ง** (รวมตอน reconnect) — message ค้าง 30s+ ทุก rotate — rabbitmq/connection.go:34-36
9. **maxRetry=100 แล้ว `return` เงียบ** — connect ไม่ได้ process อยู่ต่อแบบไม่มี consumer (main ไม่รู้) — rabbitmq/connection.go:40-43
10. **go.mod ระบุ go 1.15 แต่ Dockerfile build ด้วย 1.18** ผ่าน proxy ส่วนตัว `habor-proxy.analytichpxv3.online` — go.mod:3; Dockerfile:1
11. **handler ทั้งหมดกลืน error เป็น `fmt.Println`** เช่น `outStandingUpdateReport2` redis error แล้วทำต่อด้วยข้อมูลว่าง (reportOld/cronjob.go:96-98, :132-134); `UpdateTodayOutstanding` unmarshal fail → return เงียบ (cronjob.go:246-250)
12. **debounce แบบ race**: `timeCalculated` เป็น global ไม่มี mutex ถูกเขียนจาก consumer goroutine พร้อมกัน — reportOld/cronjob.go:17, :262-268
13. **คิวเดียวประมวลผลทีละ message + sleep 1s/message ใน SUB_OLD mode** — throughput ต่ำมากเมื่อ backlog — rabbitmq/subscribe.go:134-136
