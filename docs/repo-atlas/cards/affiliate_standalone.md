# Card: affiliate_standalone
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go + Gin + MongoDB (raw driver) + RabbitMQ (streadway/amqp) + gocron scheduler (`github.com/go-co-op/gocron`) — `service/cronService.go:160`
- **module**: `affiliate` — import ใน `_cmd/main.go:4` (`import "affiliate"`)
- **entrypoint**: `_cmd/main.go:8` `affiliate.StartServer()` → `server.go:22` `func StartServer()`
- **port**: **4042** ฮาร์ดโค้ดใน `server.go:64` (`Addr: ":4042"`) — ไม่อ่านจาก env, ไม่มี EXPOSE ใน Dockerfile
- **Dockerfile**: `Dockerfile` — builder `golang:1.23-alpine` → `alpine:3.13`, build `_cmd/main.go`, `CMD ["./app"]` (ไม่มี EXPOSE/port)
- **CI**: `.github/workflows/build.yaml` (Kaniko → registry) — มีไฟล์อยู่

## provides

### http
Route หลักลงทะเบียนใน `route.go` (group `/v1` `route.go:70`, group `/test` `route.go:126`). ไม่มี auth middleware ใด ๆ บน group — ทุก endpoint เปิดโล่ง (`route.go` ไม่มี `r.Use(...)` auth; CORS ถูกคอมเมนต์ `server.go:27`)

| METHOD | path | auth | evidence |
|--------|------|------|----------|
| GET | /v1/demo | none | route.go:72 |
| POST | /v1/reset_cron | none | route.go:73 |
| POST | /v1/aff_manual | none | route.go:74 |
| GET | /v1/pay_manual | none | route.go:75 |
| GET | /v1/pay_manual_restat | none | route.go:76 |
| POST | /v1/pay_manual_realtime | none | route.go:77 |
| POST | /v1/pay_manual_from_self_stat_realtime | none | route.go:78 |
| GET | /v1/pay_wallet_from_aff_statement | none | route.go:79 |
| POST | /v1/resetcashback | none | route.go:80 |
| POST | /v1/aff_by_self_stat_id | none | route.go:81 |
| POST | /v1/callback_sport | none | route.go:88 |
| POST | /v1/callback_muay | none | route.go:89 |
| POST | /v1/callback_affiliate | none | route.go:93 |
| POST | /v1/cashback_manual | none | route.go:95 |
| POST | /v1/hydra_re_stat | none | route.go:97 |
| POST | /v1/hydra_re_register | none | route.go:98 |
| POST | /v1/HydraReportActiveDay | none | route.go:99 |
| POST | /v1/clear_wallet_aff (💰 เคลียร์ wallet affiliate) | none | route.go:101 |
| POST | /v1/HydraReportUserActiveDay | none | route.go:102 |
| POST | /v1/reset_selfstat | none | route.go:104 |
| POST | /v1/paid_cashback (💰 จ่าย cashback) | none | route.go:105 |
| POST | /v1/pay_cashback_realtime (💰) | none | route.go:106 |
| GET | /v1/calculate_affilateufabet | none | route.go:108 |
| GET | /v1/pay_affilateufabet (💰 จ่าย affiliate → wallet) | none | route.go:109 |
| POST | /v1/cal_affilatebyID (💰) | none | route.go:111 |
| POST | /v1/clear_voidbet_allinone | none | route.go:113 |
| POST | /v1/ticket_reset_date | none | route.go:115 |
| POST | /v1/manual/covid_report_day | none | route.go:117 |
| POST | /v1/manual-tournament | none | route.go:119 |
| GET | /v1/report_incorrect_lotto | none | route.go:121 |
| POST | /v1/manual-calculate-self-stat | none | route.go:123 |
| POST | /test/create_member | none | route.go:128 |
| POST | /test/create_deposit (💰 สร้าง deposit statement) | none | route.go:129 |
| POST | /test/sleep_service | none | route.go:130 |
| POST | /test/sleep_service2 | none | route.go:131 |
| GET | /test/callback | none | route.go:132 |
| GET | /test/commission | none | route.go:133 |
| POST | /test/covid/reset_credit (💰 reset credit) | none | route.go:136 |
| POST | /test/covid/report | none | route.go:137 |
| POST | /test/affiliate_cal/manual (💰 จ่าย affiliate รายuser) | none | route.go:138 |
| POST | /test/affiliate_cal_all/manual (💰 จ่าย affiliate ทุกuser) | none | route.go:139 |
| GET | /test/aff_by_date | none | route.go:140 |

Route ตามเงื่อนไข `LOTTO_TYPE` env (`server.go:52`):
- ถ้า `LOTTO_TYPE != "AGENT"` → `RountLotto` ลงทะเบียน POST `/v1/callback_lotto` (`route.go:147-149`)
- ถ้า `LOTTO_TYPE == "AGENT"` → `RountLottoV2` (`route.go:152-161`): POST `/v1/callback_lotto`, POST `/v1/callback_mini_game`, PATCH `/v1/re_cal_mini_game`, GET `/v1/sum_report_daymonth`, GET `/v1/fix_sum_report`, GET `/v1/cron_calculate`, GET `/v1/cron_aff_realtime` — ทั้งหมด none auth

### consume
- **ไม่พบ** — grep `Consume`/`QueueDeclare` ไม่เจอใน code ที่ทำงาน (repo นี้เป็น publisher อย่างเดียว, มีแต่ `MQCh.Publish`)

### cron
gocron scheduler สร้างใน `service/cronService.go:198` (`InitCron`), `StartAsync()` ที่ `cronService.go:206`. แตกสาขาตาม `config.AgentType` (จาก collection `config_system`) และ env `BANDGAME`/`LOTTO_TYPE`.

| schedule | does what | evidence |
|----------|-----------|----------|
| every 30m | RealtimeDepositWithdrawUFA (สาขา UFA) | cronService.go:218 |
| every 15m | LoadTransactionAgentUFATransfer | cronService.go:220 |
| 01:00 daily | CronCalculateTransactionAgentUFATransfer | cronService.go:221 |
| 02:00 daily | CalculateV2UFATransfer | cronService.go:222 |
| 02:30 daily | HandlePayToWalletUFATransfer (💰) | cronService.go:223 |
| 00:30 daily | AffiliateRealtimeUfabet (สาขา UFABET/BETFLIX) | cronService.go:233 |
| 01:30 daily | PayToWalletV2 (💰) | cronService.go:234 |
| 23:59 daily | Calculate (สาขา MUAY) | cronService.go:238 |
| 01:15 daily | PayToWalletV2 (💰 MUAY) | cronService.go:239 |
| every 4m | AffiliateRealtimeV2 (สาขา default) | cronService.go:247 |
| cron "20 * * * *" | RealtimeWithdraw (💰) | cronService.go:248 |
| cron "0 8 * * *" | RealtimeDeposit | cronService.go:249 |
| every 10m | CronCallbackSport | cronService.go:252 |
| every 5m | CreatetransectionSport | cronService.go:255 |
| 23:59 daily | CalculateV2 | cronService.go:258 |
| 00:30 daily | HandlePayToWallet (💰) | cronService.go:259 |
| 01:15 daily | PayToWalletV2 (💰) | cronService.go:260 |
| 02:00 daily | CronUpdateDuration | cronService.go:261 |
| every 3h | GamesHitsDay / UpdateGameRocketWin | cronService.go:264-265 |
| 01:00 daily | UpdateBrandGameRocketWin | cronService.go:266 |
| cron "45 * * * *" | SummaryReportHour (สาขา LOTTO_TYPE=AGENT) | cronService.go:267 |
| 00:10 daily | HandleResetCashback (💰) | cronService.go:271 |
| 01:10 daily | HandleResetCommission (💰) | cronService.go:275 |
| 00:30 daily | TicketService | cronService.go:278 |
| 00:30 daily | TournamentServiceV2 | cronService.go:281 |
| Monday 00:30 | ResetCreditCovid (💰) | cronService.go:285 |
| 00:45 daily | DeleteAgentLog | cronService.go:288 |
| every 5m | BankNotify | cronService.go:291 |
| 23:30 daily (1 run) | SetDefaultBonusID | cronService.go:293 |
| 00:00:01 daily | autoCarryNextDay (เรียก OfficeAPI ถอนยกวัน) | cronService.go:295 |
| every 30m (rand start) | CheckBankOutstanding "OLD" | cronService.go:299 |
| every 10m | CheckBankOutstanding "NEW" | cronService.go:300 |

หมายเหตุ: `StartServer` ยังเรียก `go service.AgentResetCreditZero(...)` ตอนบูต (`server.go:48`) และ `Route()` เรียก script เลย: `service.DeleteStatementAffiliateDay(resource)` (`route.go:33`), `service.HandleUpdateSelfStat(resource)` (`route.go:38`), `controller.WalletValidation` (`route.go:43`), `controller.TicketMigrate` (`route.go:44`) — รันทันทีตอน startup

## consumes

### http-out
| ปลายทาง | env key | base URL | paths | evidence |
|---------|---------|----------|-------|----------|
| Agent (ผู้ให้บริการเกม) | `SERVICE_API` | runtime env 🟡 | `/userinfo?key=<user>` | service/agentService.go:44 |
| Agent | `SERVICE_API` | runtime env 🟡 | `/balance-history/<user>?time=<date>` | service/agentService.go:78 |
| Agent | `SERVICE_API` | runtime env 🟡 | `/checkstatus` | service/agentService.go:95 |
| Office API | `config_system.db_config.OfficeAPI` (จาก DB) | จาก DB 🟡 | POST `api/ManageWithdrawStatementRemain-auto-cornjobs/<SERVICE>` (💰 ยกถอนยอดข้ามวัน) | service/affiliateRealtimeV2.go:1209 |
| Firebase RTDB | `FIREBASEURL` | runtime env 🟡 | `notify/ticket/event/<user>.json` (PUT/GET) | service/ticketService.go:622, helper.go:GetFirebase/PutFirebase |
| Coupon service | `COUPON_SERVICE` | runtime env 🟡 | POST `/migrate/min_condition` | controller/migrateCoupon.go:31,44 |
| SMS (sms-kub) | hardcoded | `https://console.sms-kub.com` | POST `/api/campaigns` | service/ticketService.go:998 |
| SMS log (lupin) | hardcoded | `http://sendsms.lupin.host` | POST `/api/create_sms_log` | service/ticketService.go:1017 |
| Agent (pussy888 ฯลฯ) | `SERVICE` switch | hardcoded URLs | `http://pussy888fun.api-node.com/newAPI/agent{3,4}` ฯลฯ (เลือกแบบ rand) | service/agentService.go:118-125 |
| Telegram Bot API | `TELEGRAM_UFA_TRANSFER_BOT_TOKEN` | api.telegram.org | แจ้งเตือน agent status | service/affiliateRealtimeV2.go:228-230, helper.go:215 |

หมายเหตุ HTTP client: `helper.CallAPI` ใช้ `&http.Client{}` ไม่มี timeout (`helper/helper.go:79-80`); `HttpHandle` POST ก็ `&http.Client{}` ไม่มี timeout (`service/mainService.go:146`); `CallAPIALL` เช่นกัน (`service/ticketService.go:1034`). เฉพาะ `agentService.go` ใช้ axios timeout 10s (`agentService.go:40,72`)

### publish
ใช้ RabbitMQ default exchange (`""`) routing key = queue name (`service/mainService.go:256-262`), `Expiration: "60000"`

| queue (routing key) | payload | evidence |
|---------------------|---------|----------|
| `DYNAMICQUEUE_<SERVICE>` | model.SendDataQueue (ยอด affiliate/cashback) | controller/callbackLotto.go:111-112, service/affiliateRealtimeManual.go:668, service/affiliateRealtimeV2.go:571/664/758/793/880/2725, service/covidService.go:97, service/utilAffiliateRealtime.go:282 |
| `WALLETWITHDRAW_<SERVICE>` (เมื่อ routingKey="EVENTACTION" → map เป็นชื่อนี้) | model.SendDataQueue (💰 จ่ายเงินเข้า wallet) | service/mainService.go:253-254, เรียกจาก service/statementService.go:359 |

### data
DB = `os.Getenv("MONGODB_DB_NAME")` (`_config/db.go:59`), endpoint `MONGODB_ENDPOINT` (regex แทน readPreference=secondary→primaryPreferred, `_config/db.go:61`). ใช้ผ่าน `repo` helper (`repo/repo.go`). Collections หลัก (นับจาก call sites, R/W):

| collection | R/W | evidence (จำนวน call sites) |
|-----------|-----|------|
| config_system | R | 62 sites เช่น service/cronService.go:142, affiliateRealtimeV2.go:1191 |
| self_stat_realtime | R/W | 53 sites |
| deposit_statement | R/W | 50 sites |
| wallet_statement | R/W | 48 sites (💰) |
| self_stat | R/W | 39 sites |
| ads_statement | R/W | 37 sites |
| member_account | R/W | 32 sites |
| agent_log | R/W | 32 sites |
| member_affiliated | R/W | 24 sites |
| withdraw_statement | R/W | 22 sites (💰) |
| campaign | R/W | 18 sites |
| statement_affiliate_day | R/W/Delete | 12 sites (ถูก DeleteStatementAffiliateDay ตอนบูต route.go:33) |
| stat_lotto, event_statement, statement_lotto, callback_min_amount, stat_day, cashback_user, ads_collection ฯลฯ | R/W | <12 sites แต่ละตัว |

(รายการเต็มมี ~47 collection — ดู grep `repo.*resource, "..."`)

### external
| third-party | purpose | auth |
|-------------|---------|------|
| Telegram Bot API | แจ้งเตือน agent status ไม่ active | bot token จาก env (affiliateRealtimeV2.go:228) |
| sms-kub.com | ส่ง SMS campaign | token (CallAPIALL header "token", ticketService.go:998) |
| Firebase RTDB | แจ้งเตือน ticket event | URL จาก FIREBASEURL (ไม่มี auth header ชัดเจน) |
| Agent gaming API | ดึง credit/balance/status | key ใน query string (userinfo?key=) |
| Office API | ถอนยอดข้ามวัน | JWT HS256 header `authorization` (affiliateRealtimeV2.go:1213) |

## observations
- **affiliateRealtimeV2.go:1200-1203** — ฮาร์ดโค้ด JWT key+secret: `key := "147ce3cba244318c00c298680ca1da08"`, `secretKey := "2ac714e8d1451244432013c2d7aa2fd7"`, sign HS256 ส่งไปยัง Office API (carry-next-day ถอนเงิน) — secret ในซอร์สโค้ด
- **route.go:88-140 ทั้งหมด** — endpoint การเงินทั้งหมด (callback, pay_manual, clear_wallet, paid_cashback, pay_affilateufabet, create_deposit, covid/reset_credit, affiliate_cal/manual) **ไม่มี auth middleware ใด ๆ** — เปิดโล่งทั้ง /v1 และ /test
- **server.go:27** — `r.Use(CORS)` ถูกคอมเมนต์ออก; ไม่มี CORS/auth global
- **helper/helper.go:79-80** — `CallAPI` ใช้ `&http.Client{}` ไม่มี timeout (รวมถึง mainService.go:146 POST, ticketService.go:1034) → เสี่ยง goroutine ค้างเมื่อปลายทางไม่ตอบ
- **service/mainService.go:142-153** — `HttpHandle` POST ใช้ `panic(err)` เมื่อ request/read ล้มเหลว → panic ทั้ง service จาก HTTP error ภายนอก; `ticketService.go:1035` (`CallAPIALL`) ก็ `panic(err)`
- **helper/helper.go:88-... (mainService GetSha)** — ใช้ `sha1.New()` แฮชค่า (mainService.go:100-104) — SHA1 อ่อน
- **helper/helper.go:95 (comment)** — มี hardcoded Bearer JWT token ของ user จริงอยู่ในคอมเมนต์ (ข้อมูล PII: username/phone/bank ใน payload base64) 🟡
- **service/agentService.go:118-130** — ฮาร์ดโค้ด URL ผู้ให้บริการ (`pussy888fun.api-node.com`, `pussy888.abagroup.api-url.com`) และเลือกแบบ `rand.Intn` — non-deterministic routing
- **server.go:64** — port `:4042` ฮาร์ดโค้ด ไม่อ่านจาก env (ต่างจาก service อื่นที่ใช้ PORT)
- **route.go:33-44** — งานหนัก (DeleteStatementAffiliateDay, HandleUpdateSelfStat, WalletValidation, TicketMigrate) รัน synchronous ทันทีในฟังก์ชัน Route ตอน startup → block การ register route/บูตช้า
- **server.go:31-33** — error จาก `CreateResource()` แค่ `fmt.Println` แล้วไปต่อ (ไม่ return) → ใช้ resource ที่อาจ nil; `CreateMQ` error ก็แค่ print (server.go:35)
- **_config/mq.go:16** — `CreateMQ` retry loop ไม่มีขอบเขตจริง (`maxRetry` แค่นับขึ้น ไม่เคยหยุด) — infinite loop ถ้า RabbitMQ ตายตลอด
- **service/affiliateRealtimeV2.go:1212** — exp JWT 5 นาที + secret ฮาร์ดโค้ด ใช้ครอบ endpoint ถอนเงินจริง (OfficeAPI) → ถ้า secret รั่วสามารถปลอม token ถอนได้
- **service/cronService.go:299** — `CheckBankOutstanding` ใช้ `rand.Intn(60)` กำหนดเวลาเริ่ม (random start) — debug ยาก
- **dead code จำนวนมาก** — route.go มีบรรทัด comment-out หลายสิบบรรทัด (callback_lotto, clear_voidbet, HydraReStatV2/V3); cronService.go มี cron job comment-out จำนวนมาก; controller/test.go, testCronjob.go เป็น test endpoint ที่ deploy จริง (route.go:128-140)
- **MQ publish** — `PublishQueue` Expiration "60000" (60s TTL) — ถ้า consumer ช้า ข้อความจ่ายเงินอาจหมดอายุก่อนถูกประมวลผล (mainService.go:266)
- **deps เก่า** — `github.com/streadway/amqp` (deprecated, ใช้ rabbitmq/amqp091-go แทน), `gopkg.in/mgo.v2` style ในบาง service
