# Card: go_sms
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go — dual-mode: Gin HTTP API (รับ SMS/slip จาก forwarder) หรือ RabbitMQ subscriber (เลือกด้วย env `GROUP`/`TYPE`) — MongoDB multi-tenant, RabbitMQ (ทั้ง streadway/amqp legacy และ rabbitmq/amqp091-go ผ่าน que.Manager), Jaeger, Prometheus
- **module**: `sms` (go.mod:1, go 1.24.0 — go.mod:3)
- **entrypoint**: `_cmd/main.go:30` — `GROUP` ว่าง → `controller.ServAPI` (HTTP mode, main.go:34-40); `GROUP` มีค่า → `controller.AppSubscribe` (subscriber mode, main.go:41-46); config ผ่าน cleanenv (config/config.go:24, config/type.go:3-19)
- **port**: `:8000` (controller/api.go:61-64; legacy server.go:53 ก็ :8000) — Dockerfile ไม่มี EXPOSE
- **Dockerfile**: golang:1.24-alpine (docker.mirror.hashicorp.services) → alpine:3.23, COPY `public.pem` + `log-linebot/log.json`, `CMD ["./app"]` (Dockerfile:1-14)
- **CI**: `.github/workflows/build.yaml`

## provides
### http (ทุก route อยู่ใต้ prefix `/global` — controller/api.go:56; เพิ่ม `/metrics` Prometheus — api.go:27)
| METHOD | path | auth | evidence |
|---|---|---|---|
| GET | `/global/v1` | ไม่มี | sms/route.go:19 |
| GET | `/global/healthcheck/:service` | ไม่มี | sms/route.go:20 |
| POST | `/global/createSMS` | ไม่มี | sms/route.go:21; handler sms/sms.go:31 |
| GET | `/global/sms/:nameserver` | ไม่มี (เช็ค ApiKey ใน query กับ `database_config` ภายใน handler) | sms/route.go:22; sms/sms.go:69 |
| GET | `/global/smsSendAPI/:server` | ไม่มี (สำหรับ APP) | sms/route.go:23 |
| POST | `/global/smsNewTopup/:nameserver` | ไม่มี | sms/route.go:24; sms/smsnewtopup.go:18 |
| POST | `/global/smsNewTopupJSON/:nameserver` | ไม่มี | sms/route.go:25; sms/smsnewtopup.go:97 |
| POST | `/global/log-check-sms/:service` | ไม่มี | sms/route.go:26; sms/checked.go:22 |
| POST | `/global/sendLineStatement/:server` | `middleware.AuthHeader()` = IP allowlist จาก header `CF-Connecting-IP` | sms/route.go:27; middleware/authHeader.go:11-73 |

### consume (subscriber mode — ทุกตัว **AutoAck=true**, reconnect ทุก 3s)
| exchange/queue (exact) | handler | evidence |
|---|---|---|
| exchange `PUSHBULLET` (fanout) → queue `PUSHBULLET<SERVICE>` (ไม่มี `_` คั่น) เมื่อ `TYPE=PUSHBULLET` | `handlePushbullet` → insert statement (truewallet) + `PublishBankStatement` + `PublishAddCredit` | controller/que-sub.go:150-167, :252-312 |
| exchange `SMS` (fanout) → queue `SMSQUEUE<SERVICE>` เมื่อ `GROUP=TOPUP` | `handleTopup` → เช็ค APIKey → `CreateOne(log_sms)` → `FilterSMS` (parse SMS ธนาคาร → statement → publish) | controller/que-sub.go:170-188, :314-361 |
| exchange `SMS_NEWTOPUP_<SERVICE>` (fanout) → queue ชื่อเดียวกัน เมื่อ `GROUP=NEWTOPUP` | Channel `LINEBOT` → `handleLineBot`; `LINEBOT-OTP`/`LINEBOT-LOG-MESSAGE` → `handleLineBotOTP`; อื่น ๆ → `handleTopup` | controller/que-sub.go:190-217 |
| exchange `LINEBOT_NEWTOPUP_<SERVICE>` (fanout) → queue auto-generate (exclusive) เมื่อ env `GROUP_LINEBOT` ไม่ว่าง | `handleLineBotDom` → hash central → insert `bank_statement` → publish STATEMENT + AddCredit | controller/que-sub.go:219-237, :455-621 |
| exchange `REGISTER_PUSHMOBILE` (fanout) → queue auto-generate เมื่อ `GROUP=PUSHMOBILEKEY` | `handlePushKeyToken` → update `bank_config` device/acc + `bank_list` ของ service | controller/que-sub.go:239-250, :363-396 |

### cron
| schedule | what | evidence |
|---|---|---|
| ticker 30s (subscriber mode) | `loopCheckConnection` → sync `database_config` (status=1) → add/remove RabbitMQ bundle connection ต่อ service | controller/manage-que.go:43-66, :68-103; app-serv.go:35-38 |

## consumes
### http-out
| env key | base URL | paths | evidence |
|---|---|---|---|
| `HASH_CENTRAL` (ว่าง = ข้าม 🟡) | runtime | `POST /api/hashlayout` — ขอ hash กันซ้ำของ statement | service/hashCentral/main.go:13-44 |
| `SLIP_ACCEPT` (ว่าง = ข้าม 🟡) | runtime | POST slip verify | service/hashCentral/main.go:46-52, :82 |
| hardcode | `https://monitor.thezeus.online` | `POST /api/sms/service` (Firebase monitor, fire-and-forget `go UpdateFirebaseMonitor`) | sms/firebase.go:33-36; que-sub.go:592 |
| hardcode | `https://allapi.topupoffice.com/api/` | `GET GetRabByName`, `GET GetVersionRabbit` (lookup RabbitMQ URL ต่อ group; auth = custom encrypted header) | helper/all_api.go:12, :39-56 |

### publish (เส้นทางหลักผ่าน que.Manager — DSN ต่อ service มาจาก `database_config.RabbitmqURL`)
| exchange/queue (exact) | payload | evidence |
|---|---|---|
| exchange `STATEMENT` (fanout, durable) — **ไม่มี suffix _SERVICE ฝั่งนี้** แต่ DSN เป็น RabbitMQ ของ service นั้น | Type=`BANK`, body = `StatementModel` JSON (bank statement จาก SMS/linebot) | sms/que-pub.go:55-75; เรียกจาก que-sub.go:308, :615; sms/sms.go:1662 |
| queue `<SERVICE>` (default exchange, routing key = ชื่อ service, durable, Priority=9) | body = ObjectID hex ของ bank_statement → ให้ go_deposit เติมเครดิต | sms/que-pub.go:150-163; que-sub.go:309, :616; sms/sms.go:1663 |
| exchange `SMS_NEWTOPUP_<service>` (fanout, durable, Expiration=120000) | Type=`SMS`, body = `SMSRequest` (forward SMS เข้าคิวกลางให้ subscriber ประจำ service) | sms/que-pub.go:100-127; sms/smsnewtopup.go:46, :127; sms/linebot.go:65 |
| exchange `SMS` (fanout, Expiration=120000) | Type=`SMS`, body = `SMSRequest` (TOPUP legacy path) | sms/que-pub.go:77-98; sms/sms.go:98 |
| queue `OTP_<BANKCODE>_<PHONE>` (default exchange, durable) | OTP จาก SMS ธนาคาร (legacy `CreateQueueOTP` ยังถูกเรียกจริง) | sms/publish.go:159-236; sms/sms.go:1501 |
| queue `<service>_OTP_LOGIN_<BANKCODE>_<PHONE>` (durable, x-message-ttl 60s) | OTP login SCB app (legacy fn + hardcode CloudAMQP DSN ใน fn) | sms/publish.go:353-433; sms/sms.go:1718 |
| RabbitMQ DSN ที่ใช้: hardcode `fixRabbitNewRocketwin` (CloudAMQP amqps://xhnvvjnt:...@exotic-blond-turkey.rmq6.cloudamqp.com), `fixRabbitLineBotDom` (amqps://uqxhvlkz:...@funny-black-bobcat.rmq7.cloudamqp.com), `fixRabbitDefault` (amqp://abaofficerabbitmq:Zxcvasdf789@52.77.27.18:5672) + env `RABBITMQ_ENDPOINT` + per-service จาก DB | — | controller/manage-que.go:18-40 |

### data
| DB | collection | R/W | evidence |
|---|---|---|---|
| office (hardcode `mongodb://abaoffice:zxcvasdf789@mongodb-aba...:27017`; `GROUP` = TOPUP/NEWTOPUP/PUSHMOBILEKEY/LINEBOT_NEWTOPUP → env `MONGODB_ENDPOINT`, db = `DB_OFFICE`) | `bank_config` | R/W (จับคู่เบอร์/บัญชีรับ SMS, update device/balance) | db/db.go:67-76; sms/repository.go:89-220, :460-520 |
| office | `database_config` | R (config + RabbitMQ URL ต่อ service, sync ทุก 30s) | sms/repository.go:221; manage-que.go:71-79 |
| office | `log_sms`, `log_check_sms`, `bank_deposit_summary`, `account_linebot` | R/W | sms/repository.go:77, :421, :522, :583 |
| service DB (ต่อ tenant จาก `database_config`) | `bank_statement` | W (insert statement พร้อม hash dedup `$or` hash/hash_add_one_min/hash_sub_one_min) | sms/repository.go:236-287 |
| service DB | `bank_statement_3d`, `bank_list` | R/W | grep Collection(...) |

### external
| third-party | purpose | auth | evidence |
|---|---|---|---|
| hash-central service | สร้าง hash layout กัน statement ซ้ำ | ไม่มี (Content-Type อย่างเดียว) | service/hashCentral/main.go:39 |
| CloudAMQP ×2 + RabbitMQ EC2 52.77.27.18 | message broker กลาง | credentials hardcode | controller/manage-que.go:20-22 |
| monitor.thezeus.online | monitor dashboard | ไม่มี | sms/firebase.go:33 |
| allapi.topupoffice.com | discovery RabbitMQ URL | custom `auth` header (helper.Encrypt) | helper/all_api.go:39-56 |

## observations
1. **Hardcoded broker credentials 3 ชุดใน executing path**: CloudAMQP `xhnvvjnt:ItKB96u5...`, `uqxhvlkz:1SWaKwBF...`, EC2 `abaofficerabbitmq:Zxcvasdf789@52.77.27.18` — controller/manage-que.go:20-22; ซ้ำใน legacy sms/publish.go:240, :300, :356
2. **Hardcoded office DocumentDB credentials** — db/db.go:67, :129
3. **AutoAck=true ทุก consumer** — handler error แล้ว message หายถาวร (handler error แค่ `log.Printf` — que-sub.go:133-135); ไม่มี retry/DLQ — que-sub.go:86, :157, :176, :197, :225
4. **HTTP ingestion เกือบทั้งหมดไม่มี auth**: `/global/smsNewTopup/:nameserver` ฯลฯ เปิดรับ SMS ที่กลายเป็นรายการฝากเงินจริง — กันด้วย ApiKey ใน payload เฉพาะบาง path (que-sub.go:325-333) ; `/sendLineStatement` ใช้ IP allowlist จาก header `CF-Connecting-IP` ซึ่ง **client ปลอมได้ถ้าไม่ผ่าน Cloudflare** และค่าว่าง/`" "`/localhost ผ่านได้เลย — middleware/authHeader.go:13-18, :63-66
5. **Idempotency ของ statement พึ่ง hash จาก hash-central**: ถ้า env `HASH_CENTRAL` ว่าง จะข้ามและคืน statement เดิมโดยไม่มี hash (`return stm, nil`) — service/hashCentral/main.go:14-18; เคส hash ว่างถูกเช็คใน que-sub.go:524-533 แต่ path `FilterSMS` ใน sms.go ต้องตรวจแยก; dedup ทำผ่าน upsert `$setOnInsert` + `$or` hash ±1 นาที — sms/repository.go:256-262
6. **เวลา statement ยอมรับช่วง [-48h, now]** — SMS เก่ากว่า 2 วันถูกทิ้งเงียบ (span log อย่างเดียว) — controller/que-sub.go:546-555
7. **Legacy duplicate code ยังอยู่**: `rabbitmq/` + `sms/publish.go` (streadway) ทำงานเดียวกับ `service/que` + `que-pub.go` (amqp091) — ของเก่าถูกเรียกจาก `_cmd/main.go` ที่ comment ทิ้ง (main.go:50-68) แต่ `CreateQueueOTP`/`CreateQueueOTPLoginAPPSCB` (เก่า, dial ต่อครั้ง) **ยังถูกเรียกจริง** — sms/sms.go:1501, :1718
8. **`amqp.Dial` ต่อ message ใน legacy publish fn** ไม่มี timeout/retry, error ถูกกลืน (`if err != nil { return }` ไม่ log) — sms/publish.go:16-20, :41-44, :63-66
9. **sleep แบบ fix ใน hot path**: `waitOrDelay 200ms` ทุก message (que-sub.go:303-305, :357-359), `time.Sleep(100ms)` ก่อน publish (que-sub.go:612), start delay 30s ก่อน subscribe (app-serv.go:44)
10. **`UpdateBalance` / `CreateOrUpdateDepositSummary` / `UpdateStatementTime` error ไม่ถูกเช็ค** ใน insertStatementLinebotToService — que-sub.go:558, :619-620
11. **jaeger/init.go ประกาศ `package sms`** ทั้งที่อยู่ dir jaeger; `tracingJaeger.Init` ถูกเรียก+close ต่อ message (alloc สูง) — controller/que-sub.go:253-254, :315-316
12. **db layer import service layer** (`sms/db` ← `sms/service/que` ผ่าน `CreateConnectionWithQueManager`) — controller/api.go:41
13. **ไม่มี test ใด ๆ ใน repo**
