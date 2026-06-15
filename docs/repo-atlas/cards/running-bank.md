# Card: running-bank
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go 1.21 + MongoDB (driver v1.10.2) + Redis (go-redis v8) + RabbitMQ (streadway/amqp) — `go.mod:1-27`. ไม่มี HTTP server ที่รันจริง (Gin อยู่ใน dep แต่ entry path ไม่เปิด listener)
- **module**: `app` (`go.mod:1`)
- **entrypoint**: `_cmd/main.go:20-22` → `app.StartService()` (`app.go:11-31`) → `runbankall.StartServ()` = **infinite polling loop** (`controller/runbankAll/runbank-all.main.go:45-95`); ไม่มี graceful shutdown
- **port**: ไม่พบ — เป็น worker ล้วน; `app.go:13-16` router ของ gin ถูก comment ทิ้ง; docker-compose ไม่ map port
- **Dockerfile**: build `golang:1.25.5-alpine` (mirror hashicorp) → run `alpine:3.13`, ดาวน์โหลด `rds-combined-ca-bundle.pem` ตอน build, `CMD ["./app"]`, ไม่มี EXPOSE (`Dockerfile:1-15`)
- **CI**: `.github/workflows/build.yaml` — Kaniko → JFrog, image `running-scb-deposit`, dev/uat/v* (`build.yaml:4,26-35,54`)
- **จังหวะ loop**: sleep สุ่มตาม `BANK_CODE` — KBANK 150-200s, SCB 180-250s, GSB 300-400s, อื่น ๆ 90-160s (`runbank-all.main.go:64-92`)

## provides

### http
ไม่พบ — entry path ไม่ register route ใด ๆ (`app.go:11-31` เรียกแค่ `runbankall.StartServ`); โค้ด HTTP handler ใน `controller/runbank/` ไม่ถูกเรียกจาก main (dead code 🟡)

### consume
ไม่พบ — repo นี้เป็น RabbitMQ **producer** อย่างเดียว (ไม่มี `ch.Consume` ใน source)

### cron
| schedule | what | evidence |
|---|---|---|
| infinite loop + sleep สุ่ม 90-400s ตาม BANK_CODE | ดึง `database_config` ทุก tenant → ต่อ tenant: ดึง `bank_config` (filter `bank_code=$BANK_CODE`) → ยิง bank scraper API → dedup (SHA1 hash ±1นาที/±1วัน + Redis + hash-central) → insert `bank_statement` → publish RabbitMQ | controller/runbankAll/runbank-all.main.go:51-93,154-312 |
| ทุกรอบ loop (เฉพาะ BANK_CODE=SCB) | `ReCreateQueueAddCredit` — re-enqueue statement ที่ `status=0` ย้อนหลัง 5 ชม. | controller/runbankAll/runbank-all.main.go:96-125,217 |
| ช่วง 01:00-01:20 UTC ของวัน | `ClearBank3d` — DeleteMany `bank_statement_3d` เก่ากว่า 2 วัน | controller/runbankAll/runbank-all.main.go:809-833 |

## consumes

### http-out
URL scraper ทั้งหมดมาจาก env; credentials ธนาคาร (username/password/PIN/access_token/citizen_id) มาจาก document `bank_config` ใน Mongo ของแต่ละ tenant

| env key | ใช้ทำอะไร | paths/วิธีส่ง | evidence |
|---|---|---|---|
| `URL_GATEWAY` | gateway statement | POST `/api/get-transactions` + header `Client-ID`/`API-Key` จาก config_system | runbank-all.main.go:875,1120-1123 |
| `URL_SCB_APP` | SCB app scraper | POST JSON ReqAuth (pin, citizen_id, device_id, access_token) | runbank-all.main.go:920-928,1073-1126 |
| `SCB_BUSINESS_API` | SCB Business (NETBANKWEB) | POST `/api/scbanywhere/statement` (maker_username/maker_password) | runbank-all.main.go:932,1179-1233 |
| `URL_KTB_APP` | KTB app scraper | POST ReqAuth | runbank-all.main.go:909 |
| `URL_GSB_APP` | GSB (MyMo) scraper | POST ReqAuth | runbank-all.main.go:898 |
| `URL_LNH_APP` | LH Bank scraper | POST ReqAuth | runbank-all.main.go:887 |
| `URL_KBANK_APP` | KBANK app scraper (ต้อง SmsStatus==1) | POST ReqAuth | runbank-all.main.go:971 |
| `URL_KBANK_BIZ` | KBANK BIZ | POST ReqAuth | runbank-all.main.go:945 |
| `URL_BANK_BUSINESS` | KTB/KBANK business | POST `/api/v1/get-transactions` | runbank-all.main.go:1007 |
| `URL_BAY` | BAY netbank | **GET — user/pass ใน query string** `?user=..&pass=..&number=..` | runbank-all.main.go:963 |
| `URL_BAY_BIZ` | BAY BIZ | **GET — user/pass ใน path** `{url}{user}/{pass}/{number}` | runbank-all.main.go:955 |
| `URL_TTB` | TTB netbank | GET — user/pass ใน path | runbank-all.main.go:991 |
| `URL_KKP_APP` | KKP | GET — user/pass ใน path | runbank-all.main.go:999 |
| `URL_TRUEWALLET` | TrueWallet | GET — user/pass ใน path | runbank-all.main.go:983 |
| `PROXY_APP_URL` | proxy job service (เฉพาะ TYPE_RUN=SCB + TokenProxyBank) | POST `/api/job`, `/api/device-forward` (ส่ง pin+device_id, header `token` = CallbackAPIKey) | runbank-all.main.go:1241-1242,1279 |
| `HASH_CENTRAL` | hash-central dedup | POST `/api/hashlayout-list` (ข้าม pipeline ถ้า env ว่าง) | service/hashCentral/main.go:10,35; runbank-all.main.go:228-247,1345-1366 |
| `SLIP_ACCEPT` | slip-check service (เรียกจาก `HashSlipCheck` ซึ่งปัจจุบันถูก comment ที่ call-site 🟡) | POST slip payload | service/slipcent/main.go:16; runbank-all.main.go:253 |
| (hardcode) | LINE Notify `https://notify-api.line.me/api/notify` | alert ต่อ DB ไม่ติด + whitelist withdraw alert | db/conFrontend.go:192; report/publish.go:74 |
| 🟡 dead code: `URL_SCB`, `URL_KBANK` (ใช้ใน getStatement.go / getStatementNewVersion.go / controller/runbank ที่ไม่ถูกเรียกจาก entry path) | — | — | getStatement.go:80-115; getStatementNewVersion.go:55-497 |

### publish
RabbitMQ URL 🟡 มาจาก field `rabbitmq_url` ใน document `database_config` (per-tenant, runtime) — db/model.go:24

| exchange/queue (exact) | payload | evidence |
|---|---|---|
| queue ชื่อ = `{service}` (tenant name), durable, default exchange `""` | ObjectID hex ของ `bank_statement` ที่เพิ่ง insert (text/plain, persistent) → ให้ consumer เติมเครดิต | runbank-all.main.go:400-444 (QueueDeclare:414, Publish:430), call-site :122,270 |
| exchange `STATEMENT` (fanout, durable) — หรือ `STATEMENT_{service}` เมื่อ `GROUP=NEWTOPUP` | JSON ของ BankStatement ทั้ง document (Type: "BANK", persistent) | report/publish.go:30-63, call-site runbank-all.main.go:271 |
| 🟡 `addCredit.go:12 createQueueAddCredit` (lowercase) — dead code ไม่ถูกเรียก | — | addCredit.go:12-87 |

### data
| DB | collection | R/W | evidence |
|---|---|---|---|
| Mongo กลาง (`DB_ENDPOINT`/`DB_DBNAME` — abaoffice) | database_config | R (status=1, ทุก loop) | repository/repoAbaoffice.go:62-83; db/conFrontend.go:107-110 |
| Mongo กลาง | bank_config | R (per service+BANK_CODE) / W (`access_token`,`device_id`,`auth_status`,`balance`,`online_updated`) | repository/repoAbaoffice.go:86-123,236-280 |
| Mongo กลาง | bank_deposit_summary | W (upsert ยอดต่อวัน) | repository/repoAbaoffice.go:282-327 |
| Mongo กลาง | linenotify_accesstoken | R | repository/repoAbaoffice.go:51-59 |
| Mongo tenant (per-service ผ่าน `DBHostName` จาก database_config 🟡 runtime) | bank_statement | R/W (InsertMany + FindOneAndUpdate balance) | runbank-all.main.go:109,457,396 |
| Mongo tenant | bank_statement_3d | R/W/D (dedup window 3 วัน) | runbank-all.main.go:458,481,616,825 |
| Mongo tenant | bank_statement_withdraw_whitelist | W (+unique index hash) | runbank-all.main.go:337-356 |
| Mongo tenant | config_system | R (token LINE, gateway api key) | runbank-all.main.go:176,202,347 |
| Mongo tenant | generate_statement | R (จับคู่ยอดฝากที่ผู้ใช้ generate) | runbank-all.main.go:588 |
| Mongo tenant | Log_working | R (หา channel change log) | runbank-all.main.go:540 |
| Mongo tenant | bank_list | W (updated_at) | runbank-all.main.go:795 |
| Redis (`REDIS_URL`/`REDIS_DB`/`REDIS_PSW`, optional) | key `{service}_{bankCode}_{bankNumber}_DEPOSIT` TTL 24h — dedup ข้าม loop | service/mredis/mredis.go:62-70; runbank-all.main.go:494-528 |

### external
| third-party | purpose | auth | evidence |
|---|---|---|---|
| ธนาคาร SCB / KBANK / KTB / GSB / BAY / TTB / KKP / LNH / TrueWallet (ผ่าน scraper services ภายใน) | ดึง bank statement โดยใช้ credentials จริงของบัญชี (user/pass/PIN/citizen_id/device_id) | credentials จาก `bank_config` ใน Mongo, ส่ง plaintext JSON หรือฝังใน URL | runbank-all.main.go:872-1018,1073-1126 |
| LINE Notify | alert DB connect fail + withdraw นอก whitelist | Bearer token — ตัวหนึ่ง **hardcode ใน source** อีกตัวจาก config_system | db/conFrontend.go:188-192; report/publish.go:66-101 |
| hash-central (ภายใน) | dedup statement | ไม่มี auth header | service/hashCentral/main.go:35 |
| RabbitMQ (per-tenant) | enqueue เติมเครดิต + broadcast statement | URL จาก DB (มัก embed user:pass) | runbank-all.main.go:402; report/publish.go:17 |

## observations
- **Hardcoded LINE Notify token ใน source**: `accesstoken := "pKahNdB8I8ifXmaeekmD66EbvXSgKMF5hBYUt5bgiNa"` — db/conFrontend.go:188 (และ LINE Notify ปิดบริการแล้วตั้งแต่ มี.ค. 2025 → alert path นี้ตายเงียบ)
- **Hardcoded Mongo credentials + IP จริงใน docker-compose**: `mongodb://useradmin:Aa112233@3.1.8.89:27017/abaoffice_db` และ scraper IP `13.229.205.131:8088`, `54.169.56.40:8090` — docker-compose.yml:10-13, docker-compose-jfrog.yml:10-13 (commit อยู่ใน git)
- **Credentials ธนาคารใน URL**: BAY/TTB/KKP/TRUEWALLET ส่ง username/password เป็น path/query ของ GET (โผล่ใน access log ทุก hop) — runbank-all.main.go:955,963,983,991,999
- **`panic(err)` ใน polling loop**: `GetBnkCnf` panic เมื่อ query `bank_config` fail → process ตายทั้ง service (ไม่มี recover ใน loop) — repository/repoAbaoffice.go:99
- **Error ถูกกลืนเป็น `return ..., nil`**: transport error ของ bank call คืน `(db.ResStatementApp{}, nil)` → ถูกตีความว่า "สำเร็จแต่ว่าง" — runbank-all.main.go:1117,1129,1135,1194-1213; error ที่เหลือแค่ `fmt.Println` แล้วเดินต่อ (เช่น :88,232,251)
- **DB connect fail ที่ boot ไม่หยุดโปรแกรม**: `logrus.Error(err)` แล้วใช้ connection ที่พังต่อ (`app.go:18-27`); ใน `CreateConnectionOnce` ก็ comment `return nil, err` ทิ้ง — db/conFrontend.go:155-172
- **Dedup เปราะ**: SHA1 จาก `วันเวลา(นาที)+ยอด+เลขบัญชี+bankCode` พร้อม hash ±1 นาที / -1 วัน ชดเชย clock skew ของธนาคาร — ฝากยอดเท่ากันในนาทีเดียวกันจากบัญชีเดียวกันจะถูกทิ้งเป็น duplicate (มี side-channel `generate_statement` + `MAKE_SURE_*` status 99 ช่วย) — runbank-all.main.go:653-758,777-791
- **Scraping fragility**: ตัด statement เหลือ 70 รายการแรก (`runbank-all.main.go:630-632`), lookback hardcode -5 ชม. (:689), เวลา -7 ชม. manual สำหรับ channel ที่ไม่ใช่ SCB_APP (:668-670), เปลี่ยน BankCode ด้วย switch string (TTB→TMB ฯลฯ :645-652)
- **HTTP client timeout 3 นาที** ต่อ bank call (`repository/repoAbaoffice.go:40`) + loop เป็น sequential ต่อ tenant → bank API ค้างหนึ่งตัวถ่วงทุก tenant; `CurlStatement` ใช้ `http.DefaultClient` **ไม่มี timeout เลย** — runbank-all.main.go:1056
- **RabbitMQ ต่อใหม่ทุก message**: `amqp.Dial` ต่อ 1 publish ทั้ง `CreateQueueAddCredit` (:402) และ `report.Publish` (publish.go:17) — ไม่มี connection reuse; URL มาจาก DB ต่อ tenant
- **race condition บน slice**: `WaitGroubService` append `dataUrlBanks` จากหลาย goroutine โดยไม่มี mutex — runbank-all.main.go:834-856; เช่นเดียวกับ `HashSlipCheck` (:126-152)
- **Dead code ปริมาณมาก compile อยู่**: `getStatement.go`, `getStatementNewVersion.go`, `repoAbaoffice.go`(root), `repoFrontend.go`, `addCredit.go`, `SCB/`, `firebase/`, `jaeger/`, `controller/runbank/`, `controller/lineaccess/`, `service/flash/`, `service/otel/`, `linenotification/` — ไม่ถูกเรียกจาก `StartService` (app.go:11-31); jaeger/otel มีของแต่ entry path ไม่ init เลย → ไม่มี tracing/metrics จริง
- **Go 1.21 + dependency เก่า** (mongo-driver 1.10.2, streadway/amqp archive แล้ว, go-redis v8) — go.mod:3-27; ฐาน runtime `alpine:3.13` (EOL) — Dockerfile:8
- **TLS config สำหรับ DocumentDB เป็น `tls.Config` ว่าง** เมื่อ URI มี "ssl" ใน `CreateConnection` (ไม่ได้โหลด CA จริง — โค้ดโหลด CA ถูก comment) — db/connect.go:44-70

## CONDENSED
- TYPE: Go worker (no HTTP) — multi-tenant bank statement poller → dedup → Mongo → RabbitMQ
- PORT: ไม่พบ
- HTTP-OUT: URL_SCB_APP / URL_KTB_APP / URL_GSB_APP / URL_KBANK_APP / URL_KBANK_BIZ / URL_BAY(_BIZ) / URL_TTB / URL_KKP_APP / URL_TRUEWALLET / URL_LNH_APP / URL_GATEWAY / URL_BANK_BUSINESS / SCB_BUSINESS_API / PROXY_APP_URL / HASH_CENTRAL (/api/hashlayout-list) / SLIP_ACCEPT 🟡 / notify-api.line.me
- QUEUE-OUT: queue `{service}` (default exchange, ObjectID hex) + exchange fanout `STATEMENT` / `STATEMENT_{service}` (BankStatement JSON) — rabbitmq_url จาก DB ต่อ tenant
- QUEUE-IN: ไม่พบ
- DB: Mongo กลาง (database_config, bank_config, bank_deposit_summary) + Mongo tenant (bank_statement, bank_statement_3d, config_system, generate_statement) + Redis dedup 24h
