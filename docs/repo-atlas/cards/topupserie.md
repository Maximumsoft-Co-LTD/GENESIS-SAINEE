# Card: topupserie
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go 1.21, Gin (`github.com/gin-gonic/gin v1.7.4`), MongoDB (`go.mongodb.org/mongo-driver v1.16.0`), Redis (`go-redis/redis/v8`), RabbitMQ (`streadway/amqp`), OpenTelemetry + Jaeger | go.mod:1-40
- **module**: `topupserie` | go.mod:1
- **entrypoint**: `_cmd/main.go` → `topupserie.StartServer()` | _cmd/main.go:3-8 ; StartServer ที่ server.go:22
- **port**: อ่านจาก env `PORT` ตอนสร้าง `http.Server{Addr: ":"+PORT}` | server.go:82-84 ; ไม่มี EXPOSE ใน Dockerfile (Dockerfile:1-13) — ค่า runtime คือ **5052** จาก docker-compose port mapping `5052:5052` (docker-compose.yml:11-12, docker-compose-run.yml:6) 🟡 (runtime-injected)
- **Dockerfile**: multi-stage `golang:1.21-alpine` → `alpine:3.13`, build `_cmd/main.go`, copy `./static`, `CMD ["./app"]` | Dockerfile:1-13
- **CI**: `.github/workflows/pipeline.yml` ใช้ reusable workflow `Maximumsoft-Co-LTD/ci-cd-workflows/...reusable-docker-build-deploy.yml@v1.1.3` image_name=topupserie → JFrog | pipeline.yml:13-29 ; `build.yaml.disabled` (ปิดแล้ว)
- **bootstrap**: CORS → Prometheus → `db.CreateResource()` (Mongo) → `CreateMQ()` (Rabbit) → `RedisConnect()` → `MultiRoute()` (Route + RouteApexPro) → `service.ChangeOTPLink()` → OTel init → graceful shutdown | server.go:24-113
- **multi-host route**: ลงทะเบียน 2 ชุดเส้นทางพร้อมกัน — มาตรฐาน (`Route`) และเวอร์ชัน APEX path ภาษาสวีเดน (`RouteApexPro`) บน engine เดียวกัน | server.go:153-157, route.go:16, route_apex_pro.go:15

## provides
### http
หมายเหตุ auth: `middleware.AuthHeader(typeAuth)` โดย `typeAuth` = env `TYPEAUTH` (ถ้าไม่ใช่ `"authorization"` จะ fallback เป็น `"apikey"`) | route.go:22-25. โหมด apikey เทียบ header `apikey` กับ env `SECRET_KEY` ตรง ๆ (middleware/authHeader.go:128-139). `middleware.AccessIP()` เป็น no-op (โค้ดถูกคอมเมนต์ทั้งหมด) | middleware/accessIP.go:18-69 🟡

| METHOD | path | auth middleware | evidence (file:line) |
|--------|------|-----------------|----------------------|
| POST | /login, /login_line, /loginbytoken, /loginbypin | AccessIP เท่านั้น (no auth จริง) | route.go:38-42 |
| POST | /logout, /is_login, /set_pin, /change_pin, /forget_pin, /reset_password | AccessIP + AuthHeader | route.go:43-48,52 |
| POST | /register, /register_v3 | AccessIP + RateLimit("register") **ไม่มี AuthHeader** | route.go:54-55 |
| POST | **/withdraw** | AccessIP + AuthHeader → `controller.Withdraw` | route.go:57 |
| POST | **/peer2pay_withdraw** | **ไม่มี auth** (AuthHeader คอมเมนต์ไว้) | route.go:70 🟡 |
| POST | /otp, /otpv2, /otp_peer2pay | RateLimit("otp")/none → ส่ง OTP | route.go:67-69 |
| POST | **/qrpay, /qrpay_v2, /qrpay_umpay, /peer2pay_pay** (deposit) | AuthHeader (qrpay_v2 + RateLimit) | route.go:74-78 |
| POST | **/deposit_auto, /deposit_auto_v3** | AccessIP + AuthHeader | route.go:166-167 |
| POST | **/g2e_pay** (init) | AccessIP + AuthHeader | route.go:83 |
| POST | **/g2e_pay_confirm** (payment callback) | **AccessIP เท่านั้น, ไม่มี AuthHeader** | route.go:84 |
| POST | **/confirm_statement** | AccessIP + AuthHeader → `ComfirmStatement` | route.go:79 |
| POST | /slip_qrcode (slip verify) | AccessIP + AuthHeader | route.go:82 |
| POST | /decimal, /check_deposit, /check_enable_deposit | AccessIP + AuthHeader | route.go:73,140,95 |
| POST | /addbank, /addbank_v2, /set/bank, /auto/edit_bank | AuthHeader | route.go:89-90,269,383 |
| POST | /get/wallet, /get/affiliate (alias→Wallet), /wallet_history | AccessIP + AuthHeader → `controller.Wallet` | route.go:214-215,105 |
| POST | /get/turnover, /get/check_bonus, /get/bonus, /set/bonus, /bonus, /bonus_free | AuthHeader | route.go:210,213,212,267,101,100 |
| POST | **/game_balance, /callback_wheel, /callback_wheel_multi** | **ไม่มี auth** | route.go:112,114-115 |
| POST | /playgame_spin | AuthHeader | route.go:113 |
| POST | /covid/transfer_credit, /covid/credit, /covid/outstanding, /covid/withdraw-due | AuthHeader (กลุ่ม /covid) | route.go:403-417 |
| POST | /set/exchange, /set/buy_shop_item, /shop_exchange | AuthHeader (shop→withdraw) | route.go:276-277,156 |
| GET/POST | /game/* (listgame, accessGame, login_game, favorite, history) | บางส่วน AuthHeader บางส่วนเปิด | route.go:308-376 |
| POST | /game/bet_lotto, /game/cancel_bill, /game/access_lotto, /game/statement_lotto | AuthHeader (+RateLimit บางตัว) | route.go:336-350 |
| GET | /game/reward_lotto, POST /game/reward_lotto (รับรางวัล) | AuthHeader | route.go:361-363 |
| POST | /auto/register, /auto/add_bank, /auto/deposit_decimal, /auto/qrpay_verify, **/auto/withdraw** | AuthHeader (กลุ่ม /auto) | route.go:381-396 |
| POST | /office/systemConfig, /office/flushall-redis | **ไม่มี auth** | route.go:287-288 |
| GET | /office/reset_redis, /office/get_redis, /office/checkOTP, /office/reset_redisgame | **ไม่มี auth** | route.go:290-294 |
| POST | /set/event, /set/statement, /set/password, /set/profile_url | AuthHeader | route.go:265-270 |
| POST | /auth/captcha_verify | none → reCAPTCHA verify | route.go:300 |
| GET | /v1/version, /v1/demo | none | route.go:29-30 |
| GET/POST | /v1/demo_queue, /v1/test-recovery, /v1/backup | AccessIP (no-op) | route.go:31-33 |
| GET | /service | `AuthHeaderN8n()` (header X-N8N-API-Key = env N8N_API_KEY) | route.go:49, middleware/n8n.go:10-21 |
| GET | /external/external-cashback-commission | `AuthHeaderExternal()` (AES token-by-day) | route.go:450, middleware/authHeader.go:172-216 |
| POST | /lotto/* (login, register, otp, affiliate, member-token) | ส่วนใหญ่ไม่มี auth / บาง endpoint AuthHeader | route.go:422-440 |
| POST | /default/report | **ไม่มี auth** | route.go:444 |
| * | RouteApexPro: เส้นทาง mirror ภาษาสวีเดน (/logga-in, /dra-tillbaka=withdraw, /qrcode-pay-v2, /hjul=callback_wheel, /kontor/*=office) | route_apex_pro.go:25-319 |

### consume (queues subscribed)
| exchange/queue | handler | evidence |
|----------------|---------|----------|
| (reply-queue ชั่วคราว, anonymous) | RPC-style: `QueueDeclare("")` + `Consume(q.Name, auto-ack)` รอ response ตาม CorrelationId แล้ว Cancel | service/mainService.go:434-516 (PublishQueue), 645-714 (WheelMulti) |

ไม่พบ background consumer/worker ถาวร — service นี้เป็น **publisher + RPC client** เท่านั้น ไม่มี long-lived subscriber ที่ผูกกับ named queue/exchange

### cron
| schedule | does what | evidence |
|----------|-----------|----------|
| startup (ครั้งเดียว ไม่ใช่ schedule) | `service.ChangeOTPLink(resource)` อัปเดต link provider OTP (smskub) ใน collection `config_system` | server.go:64, service/mainService.go:3640-3671 |

ไม่พบ in-process scheduler/ticker/gocron จริง — ชื่อ `*_cron_job_*` เป็นแค่ Redis key/config label (controller/history.go:464-469, service/randomService.go:107-110) งาน cron จริงน่าจะอยู่นอก service นี้

## consumes
### http-out
| env key | base URL value | paths called | evidence |
|---------|----------------|--------------|----------|
| SERVICE_API | 🟡 runtime-injected | /userinfo, /preset-pg, /add_member, /seamless, agent game info/login/logout/balance | service/agentService.go:95,125,958,1190; controller/game.go:3326,3616,4210...; controller/withdraw.go:73; service/mainService.go:862-864 |
| SERVICE_API_OLD | 🟡 runtime-injected | agent API เก่า (เลือกตาม FILTERAPIDATE) | service/mainService.go:864, FilterAPI ~line 905 |
| SUPERCOM_URL | 🟡 runtime-injected | /api/web/{group}/{web}/game, /game/tag, /game/jackpot, /api/web_banner/{theme}, /api/block_list_user/check, /api/block_list_bank/check, /api/office/bank_code | controller/game.go:7618-7663; controller/office.go:979; controller/register.go:4219,4266,4310; service/mainService.go:4059; helper/repo.go:697 |
| API_QR | 🟡 runtime-injected | /hash, /init-payment | controller/deposit.go:115,186,229 |
| API_QR_V2 | 🟡 runtime-injected | /pay/V2 | controller/deposit.go:306,425 |
| API_QR_ALL | 🟡 runtime-injected | /v2/get-bankconfig, /v2/payin, /v2/get-bank-support | controller/deposit.go:520,854,1446,4054; controller/umpay.go:25 |
| URL_OFFICE_API | 🟡 runtime-injected | /api/auto/createQueueWithdraw/{service}, peer2pay key | controller/withdraw.go:522; controller/deposit.go:4140; service/mainService.go:3726,3748 |
| URL_OFFICE_API_PUBLIC | 🟡 runtime-injected | office public API | service/mainService.go:4111 |
| CHECK_BANK_API | 🟡 runtime-injected | /check?bank=&number=&service= | helper/repo.go:121; controller/register.go:3619,3789; service/mainService.go:4113,4218 |
| COUPON_SERVICE | 🟡 runtime-injected | coupon service endpoints | controller/coupon.go:806,1419,1533,2679 |
| MINIGAME_API | 🟡 runtime-injected | /get_url + minigame | controller/minigame.go:258,316 |
| APIWHEEL | 🟡 runtime-injected | wheel API | controller/wheel.go:125 |
| FIREBASEURL | 🟡 runtime-injected | /notify/event/{user}.json (Firebase RTDB) | service/mainService.go:1947,3001,3004; controller/office.go:199 |
| FIREBASEURL_withdraw | 🟡 runtime-injected | firebase withdraw notify | controller/member.go:939,1187; controller/register.go:615,1342,1817,2129 |
| URL_CHECK_SLIP | 🟡 runtime-injected | /api/slip-verify-deposit-statement-by-member/{service} | controller/slipQrcode.go:379 |
| MIGRATE_URL | 🟡 runtime-injected | user migration | service/loginService.go:122 |
| CLOUDFLARE_ENDPOINT | 🟡 runtime-injected (โค้ด IP-allowlist ถูกคอมเมนต์) | /ipkube.json | middleware/accessIP.go:53 🟡 |
| END_PGSMART | 🟡 runtime-injected | PG smart cutoff time | controller/game.go:8235 |
| — (hardcoded) | `http://18.138.225.205:{3110,3043,3114,3033}` | /create-user-phone/, /create-user-all/{user} | service/loginService.go:133-139,193 |
| — (hardcoded) | `https://affiliate.powerdiamonds.xyz` | /api/Survey (postback เมื่อ GROUP==ABA) | controller/register.go:677,1412,1834,2151,2312,4050 |
| — (hardcoded) | `https://t.firsttrackers.com` | /postback (affiliate tracking) | controller/register.go:4054 |
| — (hardcoded) | `https://notify-api.line.me/api/notify` | LINE Notify (alert DB/error) | _config/db.go:138; repo/repo.go:333; service/mainService.go:2936; controller/game.go:5767 |
| — (hardcoded) | `https://www.google.com/recaptcha/api/siteverify` | reCAPTCHA verify | controller/captcha.go:33,48 |

### publish
ทุก routing key/exchange ต่อท้ายด้วย `_<SERVICE>` (env `SERVICE`)
| exchange/queue (exact string) | payload | evidence |
|-------------------------------|---------|----------|
| queue `DYNAMICQUEUE_<SERVICE>` (default exchange) | `model.SendDataQueue` RPC | service/mainService.go:479-492 (PublishQueue), call sites controller/covid/member.go:353, register.go:270,331 |
| queue `ORIGINWITHDRAW_<SERVICE>` | SendDataQueue (transfer covid→main) | controller/covid/register.go:129 |
| queue `WALLETWITHDRAW_<SERVICE>` | SendDataQueue (shop exchange→withdraw) | controller/promotion/shop.go:754,1161 |
| queue `WALLETWITHDRAW_<SERVICE>` type=`EVENTACTION` | SendDataQueue (mini event withdraw) — routingKey EVENTACTION ถูก map queueName→WALLETWITHDRAW | service/mainService.go:475-478; controller/miniEvent.go:223 |
| queue `WHEELEVENT_<SERVICE>` | SendDataQueue per spin (wheel multi) | service/mainService.go:680-692,772-784; call sites controller/wheel.go (PublishQueueWheelMulti/V2) |
| queue named=`<SERVICE>` (durable) type=`WALLET-TO-GAME`/`""` | task id/username string (text/plain) — createQueueAddCredit (เติมเครดิตหลัง deposit) | controller/deposit.go:3134-3166; callers bonus.go:682,961,1013, deposit.go:1917,2068,2354, slipQrcode.go:176, service/multipleDeposit.go:231,317,451,698 |
| exchange `STATEMENT_<SERVICE>` (fanout, durable) type=`DEPOSIT` | `model.DepositStatement` | service/mainService.go:1697-1707 (PublishDepositStatement); callers cashbackSelfStat.go:856, checkin.go:601, commissionSelfStat.go:674 |
| exchange `STATEMENT_<SERVICE>` (fanout) type=`<typeName>` เช่น `RE-DEPOSIT` | DepositStatement | service/mainService.go:1736-1746 (PublishSubStatement); controller/office.go:936 |
| exchange `STATEMENT_<SERVICE>` (fanout) type=`MEMBER` | `model.MemberAccount` | controller/register.go:4077-4087 (PublishReigsterStatement) |
| exchange `STATEMENT_<SERVICE>` (fanout) type=`LOGIN`/`REGISTER`/`BONUS_FREE`/`TRANSFER_CASHBACK` | []byte report | service/mainService.go:1778-1788 (PublishReport); callers member.go:1365,1598,1884,1987, register.go:1457, bonusFree.go:200, cashbackSelfStat.go:978 |
| queue `<routingKey>_<SERVICE>` บน RabbitMQ คนละ host (env `RABBITMQURLLOG`) channel เช่น QUEUE_ADD_CREDIT, WITHDRAW, STAKE_POOL, REGISTER_STATEMENT, SHOP_EXCHANGE_TO_WITHDRAW, TRANSFER_COVID_WALLET, MULTI_QUEUE_ADD_CREDIT, CALL_BACK_WHEEL, WITHDRAW_POCKET_WALLET | `model.SendQueueLog` + insert collection `queue_log` ใน DBLog | service/mainService.go:526-626; call sites grep PublishQueueLog (~14 จุด) |

### data
MongoDB connect ผ่าน env `MONGODB_ENDPOINT` + `MONGODB_DB_NAME` (db หลัก) และ `MONGODB_ENDPOINT_LOG` + `MONGODB_DB_NAME_LOG` (DBLog, optional) | _config/db.go:42-92. ชื่อ DB เป็น runtime (🟡). Redis: env `REDIS_URL`, `REDIS_DB` | _config/rd.go:57-76. ไม่มี ORM ใช้ raw driver ผ่าน `repo.*`/`repoLog.*` collection helper.

| DB name | collection | R/W | evidence |
|---------|------------|-----|----------|
| MONGODB_DB_NAME 🟡 | member_account | R/W | grep collection ~174 จุด เช่น repo.* call sites controller/member.go, register.go |
| MONGODB_DB_NAME 🟡 | deposit_statement | W (CreateStatement) | service/mainService.go:1815; controller/deposit.go |
| MONGODB_DB_NAME 🟡 | withdraw_statement | R/W | controller/withdraw.go:510,524 |
| MONGODB_DB_NAME 🟡 | wallet_statement / wallet_statement_lotto | R/W | grep ~63/4 จุด |
| MONGODB_DB_NAME 🟡 | event_statement, event_config, generate_statement | R/W | grep ~98/22/16 จุด |
| MONGODB_DB_NAME 🟡 | confirm_statement | W | controller/deposit.go:3178 |
| MONGODB_DB_NAME 🟡 | bank_statement, bank_statement_3d, bank_account_register, bank_list | R/W | controller/deposit.go:3122; grep |
| MONGODB_DB_NAME 🟡 | member_affiliated, affiliate_stack, affiliate_level, statement_affiliate_lotto/day | R/W | grep ~45/3/4/6/3 |
| MONGODB_DB_NAME 🟡 | member_bonus, member_promotion, bonus_config, member_stack, member_log | R/W | grep ~18/17/5/15/9 |
| MONGODB_DB_NAME 🟡 | config_system, config_history | R/W (ChangeOTPLink update) | service/mainService.go:3640-3671; grep ~28/14 |
| MONGODB_DB_NAME 🟡 | game_list, game_group, game_type, games_user, games_history, game_*_delete | R/W | controller/game.go; grep |
| MONGODB_DB_NAME 🟡 | staking_pool, staking_statement | R/W | grep ~5/8 |
| MONGODB_DB_NAME 🟡 | shop_item, shop_address, shop_statement, shop_exchange | R/W | controller/promotion/shop.go |
| MONGODB_DB_NAME 🟡 | coupon_code, coupon_detail | R/W | controller/coupon.go |
| MONGODB_DB_NAME 🟡 | statement_lotto, statement_lotto_list, lotto_betdetail, lotto_callback, result_lotto | R/W | controller/lotto, game.go |
| MONGODB_DB_NAME 🟡 | mission_statement, ranking_config, ticket_config, campaign | R/W | grep |
| MONGODB_DB_NAME 🟡 | self_stat, self_stat_realtime, cashback_user, commission_user | R/W | grep ~11/12/4/4 |
| MONGODB_DB_NAME 🟡 | g2epay_statement, ads_statement, sms_log, phone_number_otp, agent_log | R/W | grep |
| MONGODB_DB_NAME 🟡 | report_active_user_day, report_active_user_month | R/W | grep ~4/4 |
| MONGODB_DB_NAME_LOG 🟡 (DBLog) | queue_log | W | service/mainService.go:591 (repoLog.CreateStatement) |
| Redis (REDIS_URL, DB=REDIS_DB) | game cache DB=2, config/session/rate-limit keys | R/W | _config/rd.go:57-91; middleware/rateLimit.go |

### external
| third-party | purpose | auth | evidence |
|-------------|---------|------|----------|
| LINE Notify (notify-api.line.me) | แจ้งเตือน DB error / alert | Bearer token **hardcoded** `pKahNdB8...` | _config/db.go:134-141 |
| Google reCAPTCHA | verify captcha | secret **hardcoded** `6LdjBBci...` | controller/captcha.go:40,48 |
| Firebase RTDB | notify event/withdraw | URL จาก env | service/mainService.go:3004; member.go:939 |
| affiliate.powerdiamonds.xyz / t.firsttrackers.com | affiliate survey/postback | query param (no auth) | controller/register.go:677,4054 |
| 18.138.225.205:30xx | create-user (migration/login) | none | service/loginService.go:133-193 |
| Payment gateway (API_QR*) | QR/payin/bankconfig | none เห็นใน HttpHandle | controller/deposit.go |
| SUPERCOM (game/banner/blocklist) | data เกม + block list | none | controller/game.go, register.go |
| Office API (URL_OFFICE_API) | createQueueWithdraw, peer2pay | header `apiKey`/`api_key` = env SECRET_KEY | controller/withdraw.go:522-524 |

## observations
- **Hardcoded secret — LINE Notify token** `pKahNdB8I8ifXmaeekmD66EbvXSgKMF5hBYUt5bgiNa` commit ลงโค้ด | _config/db.go:134
- **Hardcoded secret — reCAPTCHA secret key** `6LdjBBciAAAAAHDQysRO-Bo03L1Zyol6uAXyVHp8` (env `RECAPTCHA_SECRET_KEY` ถูกคอมเมนต์ทิ้ง) | controller/captcha.go:39-40
- **Hardcoded production IP** `18.138.225.205` 4 พอร์ตในโค้ด login/migration | service/loginService.go:133-139,193
- **JWT secret fallback อ่อน**: ถ้า `ACCESS_SECRET` ว่าง ใช้ค่า `"secret"` | service/authJWT.go:35-40 ; lotto token อายุ **8765 ชม. (~1 ปี)** | service/authJWT.go:68
- **apikey auth = string เทียบตรง**: header `apikey` เทียบ env `SECRET_KEY` แบบ plain (โค้ด HMAC signature ถูกคอมเมนต์) และ `color.Red(mySecret)`/`color.Cyan(authorization)` พิมพ์ secret ลง log | middleware/authHeader.go:128-139
- **`AccessIP()` เป็น no-op**: ตรรกะ IP-allowlist ถูกคอมเมนต์ทั้งหมด แต่ยังถูกเรียกหน้า endpoint สำคัญ (ให้ความรู้สึกปลอดภัยลวง) | middleware/accessIP.go:18-69
- **Money endpoint ไม่มี auth**: `/peer2pay_withdraw` (AuthHeader คอมเมนต์) | route.go:70 ; `/game_balance`, `/callback_wheel`, `/callback_wheel_multi` ไม่มี auth | route.go:112-115 ; `/g2e_pay_confirm` payment callback ไม่มี auth | route.go:84 ; `/default/report`, `/office/*` (flushall-redis, reset_redis) ไม่มี auth | route.go:288-294,444
- **`AuthHeaderExternal` logic เพี้ยน**: `strings.HasPrefix(authHeader, "")` คืน true เสมอ และ `TrimPrefix(..., "")` ไม่ตัดอะไร — เช็ค prefix ใช้งานไม่ได้จริง; keySource AES `"tyJo8xRuyA"` hardcoded | middleware/authHeader.go:175-184
- **ไม่มี balance lock / transaction**: deposit/withdraw/wallet ใช้ pattern อ่าน statement → publish queue → update; ไม่มี Mongo transaction หรือ distributed lock — เสี่ยง race/double-credit (เติมเครดิตผ่าน fire-and-forget `go createQueueAddCredit`) | controller/deposit.go:3132-3176; controller/withdraw.go:510-534
- **ไม่มี idempotency key**: `CorrelationId = time.Now().Unix()` (วินาที) อาจชนกันถ้ามีหลาย request ในวินาทีเดียว, ใช้เป็น key จับคู่ RPC response | service/mainService.go:469,598,678,770
- **Swallowed errors เยอะ**: `json.Unmarshal(body, obj)` ไม่เช็ค error ใน GetRedis/externalCALL | _config/rd.go:31,43; _config/db.go:164. `bodyQueue, _ := json.Marshal(...)` ละ error ทุกที่ | service/mainService.go:472,606
- **RabbitMQ reconnect loop ไม่มี backoff cap**: `CreateMQ` วน `for{}` sleep 1s ตลอดไป, `maxRetry++` ไม่เคยถูกใช้เป็นเงื่อนไขหยุด (dead code), `return nil` หลัง for ไม่มีทางถึง | _config/mq.go:14-41
- **MQReconnector ทำงานครั้งเดียว**: ตั้ง `recon=false` หลัง reconnect รอบแรก ทำให้ goroutine จบ ไม่ดักการตัดครั้งถัด ๆ ไป | _config/mq.go:43-53
- **HTTP client ไม่มี timeout**: `HttpHandle` ใช้ `&http.Client{}` ไม่มี Timeout (เฉพาะ Transport.IdleConnTimeout) | service/mainService.go:1481,1501; เช่นเดียวกัน middleware/accessIP.go:92,113 และ `externalCALL` _config/db.go:152 (ไม่มี timeout เลย) — 15 จุดใช้ `http.Client{}` เปล่า
- **PublishQueueWheelMulti panic เสี่ยง**: `respQueueAll[len(respQueueAll)-1]` index เมื่อ slice ว่าง (ทุก response timeout) → index out of range | service/mainService.go:716,805
- **`quit := make(chan os.Signal)` unbuffered** อาจพลาดสัญญาณ (Go vet เตือน signal.Notify บน unbuffered channel) | server.go:99
- **CORS เปิดกว้างสุด** `Allow-Origin: *`, `Allow-Methods/Headers: *`, OPTIONS ตอบ 200 ทุกกรณี | server.go:117-145
- **Resp(401) ไม่ส่ง body**: เมื่อ status 401 ฟังก์ชัน `Resp` `return` ก่อนเขียน JSON → client ได้ response ว่าง | service/mainService.go:69-72
- **Dependency เก่า/EOL**: `dgrijalva/jwt-go v3.2.0` (archived, มี CVE GO-2020-0017), Gin v1.7.4 (เก่า), `go-redis/redis v6` + `v8` ใช้ปนกัน, `streadway/amqp` (deprecated → ควรย้าย rabbitmq/amqp091-go) | go.mod:7-21
- **Dead code / debug routes**: `/v1/demo`, `/demo/agent`, `testRateLimit.go`, route comment จำนวนมาก (`// main.POST(...)`), `controller.RecoveryMember`/`GetBackupMember` เปิดด้วย AccessIP no-op | route.go:30-33,178-180,305; controller/testRateLimit.go
- **เส้นทางซ้ำซ้อน 2 ชุด (Route + RouteApexPro)** map controller เดียวกันด้วย path คนละภาษา เพิ่ม attack surface และภาระดูแล (เช่น `/withdraw` กับ `/dra-tillbaka`) | route_apex_pro.go:44
- **`InitConfig(resource)` ถูกเรียก 2 ครั้ง** (ทั้งใน Route และ RouteApexPro) ตอน bootstrap | route.go:20, route_apex_pro.go:18
