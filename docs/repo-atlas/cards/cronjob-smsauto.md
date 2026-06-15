# Card: cronjob-smsauto
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack:** Go 1.19 (`go.mod:3`), Gin v1.8.1, gocron v1.18.0, go-redis v6 (ไม่ใช่ v8), mongo-driver v1.11.1, go-axios (`go.mod:5-16`) — ไม่มี RabbitMQ
- **module:** `smsauto` (`go.mod:1`)
- **entrypoint:** `_cmd/main.go:8` → `smsauto.StartServer()` → `server.go:15-35` (godotenv → gin → multi-tenant Mongo → `Route()` → `controller.InitCron()` → `r.Run()`)
- **port:** gin `r.Run()` ไม่ระบุ port ในโค้ด (`server.go:34`) → ใช้ env `PORT=5050` (`.env:3`); docker-compose ประกาศ `4044:4044` ไม่ตรงกับ .env 🟡 (`docker-compose.yml:10`)
- **Dockerfile:** `Dockerfile:1-12` golang:1.19.2-alpine3.16 → alpine:3.13 ผ่าน harbor proxy `habor-proxy.analytichpxv3.online`
- **CI:** `.github/workflows/build.yaml:4` image `cronjob-smsauto`, Kaniko → JFrog

## provides

### http
| METHOD | path | auth | evidence |
|---|---|---|---|
| GET | `/test/demo` | ไม่มี auth (CORS allow `*`) — logic จริงถูก comment เหลือตอบ "Success CanclePlay" เปล่า ๆ 🟡 | `route.go:13`, `controller/testfunc.go:10-27`, `server.go:40-42` |
| GET | `/test/demo2?service=<name>` | ไม่มี auth — เรียก `HandleGetMemberVip` จริง คืนชื่อ+เบอร์โทรลูกค้า VIP ของ tenant ที่ระบุ | `route.go:14`, `controller/testfunc.go:29-44` |

### consume
ไม่พบ — ไม่มี RabbitMQ ใน repo (ไม่มี amqp ใน go.mod)

### cron
| schedule | what | evidence |
|---|---|---|
| ทุก 10 นาที (gocron, TZ=UTC) | `HandleSmsAutoTypeOneNodeposit` — ดึง config type "1" จาก OFFICE DB → หา member สมัครแล้วไม่ฝากในช่วงเวลา → ยิง SMS (SMSKUB หรือ YingSMS) → insert `log_sms_auto` | `controller/smsauto.go:541-544`, handler `controller/smsauto.go:18-144` |
| ทุก 10 นาที | `HandleSmsAutoTypeOneSendPromotion` — config type "3" ส่งโปรโมชั่นให้สมาชิกใหม่ | `controller/smsauto.go:547`, handler `controller/smsauto.go:146-271` |
| ทุก 3 ชั่วโมง | `HandleSmsAutoTypeTwoOffline` — config type "2" ตามลูกค้าเลิกเล่น N วัน (aggregate `report_active_user_day`) | `controller/smsauto.go:550`, handler `controller/smsauto.go:273-401` |
| ทุกวัน 23:59 (UTC) | `HandleSmsAutoTypeThreeVip` — config type "vip" เทียบ `report_active_user_month` เดือนก่อน vs เดือนนี้ หาลูกค้า VIP หาย | `controller/smsauto.go:553`, handler `controller/smsauto.go:403-538` |

## consumes

### http-out
| env key / source | base URL | paths | evidence |
|---|---|---|---|
| hardcoded (override ได้จาก `config_system.sms.api` ใน DB tenant 🟡 runtime) | `https://console.sms-kub.com` | POST `/api/messages` (ส่งทันที), POST `/api/campaigns` (ตั้งเวลา) — header `token: <config_system.sms.password>` | `service/smsService.go:385-407`, `service/mainService.go:110-124` (CallAPIALL) |
| `config_system.sms_mkt.link` (จาก DB tenant 🟡 runtime) | YingSMS | POST `/v1/campaign?service=hpx` — header `X-Auth-Token: <sms_mkt.password>` | `service/smsService.go:439-467`, `service/mainService.go:126-143` (CallAPIYingSms) |
| hardcoded 🟡 dead code (ฟังก์ชันไม่ถูกเรียกที่ไหน) | `http://sendsms.lupin.host` | POST `/api/sms_log` (`SendMessage`) | `service/smsService.go:250-276`; ไม่มี caller (grep ทั้ง repo) |
| hardcoded 🟡 dead code | `https://abaplan.sunnygreen.online` | GET `/api/smsauto/smsauto_statement?type=` (`GetSmsAutoStatement`, axios timeout 10 นาที) | `service/smsService.go:278-291`; ไม่มี caller |

### publish
ไม่พบ — ไม่มี message broker

### data
| DB | collection | R/W | evidence |
|---|---|---|---|
| Mongo OFFICE (`MONGODB_OFFICE_DB_NANE` — typo NANE, + `MONGODB_ENDPOINT`) | `database_config` | R (boot: โหลด connection string ของทุก tenant ที่ status=1 → `ResourceService[service]`) | `_config/db.go:28-30,62-106` |
| Mongo OFFICE | `smsauto_statement` | R (config แคมเปญ SMS ราย type) | `service/smsService.go:315`, เรียกจาก `controller/smsauto.go:21,149,276,405` |
| Mongo OFFICE | `smssystem_config` | R 🟡 (เฉพาะ `CheckService` ซึ่งไม่ถูกเรียก) | `service/smsService.go:324` |
| Mongo OFFICE | `log_sms_auto` | W (บันทึก log การส่ง พร้อม username_sms/password_sms ใน document) | `service/smsService.go:507-523`, `controller/smsauto.go:132,259,386,525` |
| Mongo per-tenant (`resource[valueStatement.Service]` จาก database_config) | `member_account` | R (เบอร์โทร, date_regis, topup_status, user_type vip/wisdom) | `service/smsService.go:36,68,143,168` |
| Mongo per-tenant | `report_active_user_day` | R (aggregate หา last active) | `service/smsService.go:115` |
| Mongo per-tenant | `report_active_user_month` | R (top 100 deposit เดือนก่อน/เดือนนี้) | `service/smsService.go:191,208` |
| Mongo per-tenant | `config_system` | R (credential SMS ของ tenant: sms.username/password, sms_mkt.provider/link/password) | `service/smsService.go:334`, ใช้ที่ `controller/smsauto.go:50,177,305,443` |
| Redis (`REDIS_URL`, DB 5 hardcode, no password) | key `sentPhoneNumber_<SERVICE>_<YYYY-MM-DD>` TTL 24h (กันส่ง SMS ซ้ำรายวัน) | R/W | `service/redis.go:40-48`, `controller/smsauto.go:106-114`, `service/smsService.go:22-25,54-58,134-138,157-161` |

### external
| third-party | purpose | auth | evidence |
|---|---|---|---|
| SMSKUB (console.sms-kub.com) | ส่ง SMS / สร้างแคมเปญตั้งเวลา | header `token` = password จาก config_system ของ tenant | `service/smsService.go:385-407`, `service/mainService.go:114` |
| YingSMS (`sms_mkt.link`) | ส่ง SMS campaign (provider="yingsms") | header `X-Auth-Token` | `service/smsService.go:443-467`, `service/mainService.go:133` |

## observations
- **secrets committed:** `.env` commit พร้อม `MONGODB_ENDPOINT=mongodb+srv://root:Zxcvasdf789@test02.9oh7e...` (`.env:2`) — credential เดียวกับ queue-withdraw-go/.env:7
- **SMS credential ถูกเก็บลง log collection:** `log_sms_auto` document มี field `username_sms`/`password_sms` เป็น plaintext (`controller/smsauto.go:74-75,201-202,329-330,467-468`)
- **endpoint ทดสอบเปิดสาธารณะ ไม่มี auth:** `/test/demo2?service=X` คืน username+phone_number ของลูกค้า VIP ทุก tenant (PII leak) (`route.go:14`, `controller/testfunc.go:36-42`, CORS `*` `server.go:40`)
- **`panic(err)` ใน HTTP client:** `CallAPIALL`/`CallAPIYingSms` panic เมื่อ request fail (`service/mainService.go:119,138`) — เรียกจาก cron goroutine → gocron job ตาย/panic ขึ้น process (ไม่มี recover)
- **err จาก http.NewRequest ถูกเขียนทับโดยไม่เช็ค:** `service/mainService.go:113-117` (req,err := NewRequest แล้ว err ถูก reuse ที่ client.Do โดยไม่ตรวจค่าแรก)
- **no timeout บน HTTP client:** `http.Client{}` เปล่าใน CallAPIALL/CallAPIYingSms (`service/mainService.go:116,135`) — SMS API ค้าง = cron run ค้างไม่จำกัด; axios instance ตั้ง timeout 10 นาที เฉพาะ dead code (`service/smsService.go:280-282`)
- **idempotency พึ่ง Redis อย่างเดียว:** กันส่งซ้ำด้วย key `sentPhoneNumber_*` TTL 24h (`controller/smsauto.go:106-114`) — Redis ล้ม/flush = SMS ซ้ำทั้ง batch; การอ่าน-append-เขียน key ไม่ atomic (Get→append→Set, `controller/smsauto.go:110-112`) และ cron 2 ตัว (type1/type3) รันทุก 10 นาทีพร้อมกันอาจเขียนทับกัน
- **Redis DB hardcode = 5** (`service/redis.go:44`) ไม่อ่านจาก env ต่างจาก service อื่นในระบบ
- **swallowed errors:** ผลลัพธ์ `HandleGetMember*` ถูก ignore err (`controller/smsauto.go:42,169,297,434` ใช้ `dataUser, _ :=`); InsertLogSMS fail แค่ println (`controller/smsauto.go:132-134`); `client.Connect` ที่ OFFICE DB ใช้ `_ =` (`_config/db.go:45`)
- **boot ต่อ Mongo ทุก tenant ค้างไว้ตลอด:** `createConnectionOnce` เปิด client ต่อทุกแถวใน `database_config` (pool 1/tenant) ตอน start เท่านั้น (`_config/db.go:62-106`) — tenant ใหม่/เปลี่ยน connection string ต้อง restart; tenant ที่ต่อไม่ได้แค่ print แดงแล้วข้าม (`_config/db.go:96-99`) ทำให้ `resource[service]` เป็น nil → nil map access ตอน cron ถ้า config ชี้ tenant นั้น (`controller/smsauto.go:42` ไม่เช็ค nil)
- **typo env key:** `MONGODB_OFFICE_DB_NANE` (NANE ไม่ใช่ NAME) (`_config/db.go:28`, `.env:1`)
- **cron ตั้ง timezone UTC** (`controller/smsauto.go:541`) แต่ logic เวลาใช้ `time.Now().Local()`/`time.Local` ปน UTC หลายจุด (`controller/smsauto.go:55-57`, `service/smsService.go:358-368`, `service/mainService.go:100-108` Bod2/Eod2 แปลง Local→UTC) — ขึ้นกับ TZ ของ container (docker-compose mount /etc/timezone, `docker-compose.yml:7-8`)
- **dead code:** `SendMessage` (sendsms.lupin.host), `GetSmsAutoStatement` (abaplan.sunnygreen.online), `CheckService` ไม่ถูกเรียกเลย (`service/smsService.go:250,278,321`) 🟡; `/test/demo` ตัว logic ถูก comment ทั้งก้อน (`controller/testfunc.go:12-23`) 🟡
- **โค้ด 4 handler ซ้ำกันเอง ~90%:** บล็อก สร้าง `en` bson.M → เลือก provider → set redis → InsertLogSMS copy-paste 4 ชุดใน `controller/smsauto.go:59-135,186-261,314-388,452-527`
- **port config ขัดกัน:** `.env:3` PORT=5050 vs `docker-compose.yml:10` map 4044:4044 — container จริงฟัง 5050 แต่ compose expose 4044 🟡
- **deps เก่า:** Go 1.19 EOL, go-redis v6 (ไม่รองรับ context), alpine:3.13 EOL (`go.mod:3,9`, `Dockerfile:6`), ioutil deprecated (`service/mainService.go:122,141`)
