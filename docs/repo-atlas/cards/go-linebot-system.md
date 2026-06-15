# Card: go-linebot-system
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go 1.19 (go.mod:3) / module `linebot` (go.mod:1) — Gin (server.go:10,19), MongoDB driver raw (no ORM), go-redis/v8, gocron, dgrijalva/jwt-go (เก่า) — เป็น LINE Bot OA สำหรับระบบพนันมวย (คู่มวย/เดิมพัน/รายงาน) แบบ multi-tenant
- **module**: `linebot` (go.mod:1) ; entry `_cmd/main.go:8` เรียก `linebotsystem.StartServer()`
- **entrypoint**: `StartServer()` ที่ server.go:15 — โหลด .env (server.go:16), สร้าง gin + CORS (server.go:19-20), เชื่อม Mongo (server.go:22), init config (server.go:32), register route (server.go:34)
- **port**: `:4067` ฮาร์ดโค้ด (server.go:40) ; docker-compose map `4067:4067` (docker-compose.yml:8)
- **Dockerfile/CI**: Dockerfile FROM golang:1.18-alpine3.16 builder → alpine:3.13, copy `FlexLine` → `./flexline` (Dockerfile:1-12, ไม่ EXPOSE/ไม่กำหนด PORT). CI `.github/workflows/build.yaml`: self-hosted → Kaniko build → JFrog → repository_dispatch ไป K8s manifest (`URL_K8S_LINEBOT_MUAY`, build.yaml ใน repo)

## provides
### http
| METHOD | path | auth | evidence |
|--------|------|------|----------|
| POST | /webhook/:key | ไม่มี (LINE webhook, ไม่ verify signature) | route.go:18 → controller.Webhook (webhook.go:15) |
| POST | /redis/reset_redis | ไม่มี (เช็คแค่ body.Service) | route.go:22 → Init.go:17 |
| POST | /redis/reset_redis_key | ไม่มี | route.go:23 → Init.go:77 |
| POST | /report/summary | AuthHeader (JWT Bearer HS256) | route.go:27 → handleReport SummaryPerDay ; middleware authHeader.go:26 |
| POST | /report/summary_user | AuthHeader | route.go:28 |
| POST | /report/round | AuthHeader | route.go:29 |
| POST | /report/round_detail | AuthHeader | route.go:30 |
| POST | /report/member_credit | AuthHeader | route.go:31 |
| POST | /report/history_user | AuthHeader | route.go:32 |
| (NoRoute) | * | - | server.go:35-38 ตอบ 404 |

หมายเหตุ LINE webhook: `/webhook/:key` รับ events จาก LINE Platform โดยตรง (webhook.go:18-25), แยก source.type เป็น "group"/"user" (webhook.go:61-131), จัดการ message text/image/postback (filterType webhook.go:142). `:key` = ชื่อ service/tenant ใช้ค้น redis `configService_<key>` (webhook.go:34).

### consume
| exchange/queue | handler | evidence |
|---|---|---|
| ไม่พบ (ไม่มี RabbitMQ/AMQP — go.mod ไม่มี amqp; ใช้แค่ Redis เป็น cache มิใช่ pub/sub) | - | go.mod:5-15 |

### cron
| schedule | what | evidence |
|---|---|---|
| ทุก 5 นาที (gocron) | `serviceWebHook` — วน config ทุก service, ยิง POST `https://api.line.me/v2/bot/channel/webhook/test` เพื่อทดสอบ webhook ของแต่ละ LINE channel | Init.go:281-320 (CronWebHook ถูกเรียกที่ route.go:15) |

## consumes
### http-out
| env key | base URL | paths | evidence |
|---|---|---|---|
| (ฮาร์ดโค้ด) | https://api.line.me | /v2/bot/channel/webhook/test (Init.go:312), /v2/bot/profile/{userID} (lineMessage.go:26), /v2/bot/group/{groupID}/member/{userID} (lineMessage.go:67), /v2/bot/message/reply (lineMessage.go:89), /v2/bot/message/push (lineMessage.go:115), /v2/bot/chat/loading/start (lineMessage.go:139) | LINE Messaging API |
| (ฮาร์ดโค้ด) | https://allapi.topupoffice.com | /api/GetBankListConfig (mainService.go:211) | ดึงรายชื่อธนาคาร, cache redis `listbank_all` 24h |
| `objAgent.ApiUrl` (จาก collection database_config field `api_url`, set ที่ Init.go:217) 🟡 runtime | <ApiUrl> | /userinfo, /user/cost, /users-wallet, /add_member, /open_round, /update_round, /summarymuay, /ex-summary-match, /latest_round, /betmuay, /deposit, /withdraw | agentService.go:22,51,85,106,138,195,249,329,358,412,462,492 — เกม/กระเป๋าเงิน agent |
| `objAgent.ApiFontendUrl` (database_config `api_fontend_url`, Init.go:215) 🟡 runtime | <ApiFontendUrl> | /decimal (lineOAQuestion.go:691), /withdraw (871), /addbank (1092), + dynamic path (1159,1220) ผ่าน `CallApiMain` (mainService.go:151) | API หลักของเว็บ (ฝาก/ถอน/เพิ่มบัญชี) เซ็น JWT ด้วย `API_TOKEN` |
| `DOMAIN_CUSTOMER` | env | ใช้ประกอบ URL รายงานใน GenerateToken (`%s/%s?token=%s`) | authJWT.go:48 ; .env:3 = http://192.168.1.101:6900/rpt |

### publish
| exchange/queue | payload | evidence |
|---|---|---|
| ไม่พบ | - | ไม่มี publisher (ไม่มี amqp ใน go.mod) |

### data
| DB | collection | R/W | evidence |
|---|---|---|---|
| Mongo (OFFICE = MONGODB_DB_NAME) | database_config | R (`status:1`) — สร้าง connection ต่อ tenant | _config/db.go:65, Init.go:39,111 |
| Mongo (per-tenant `resource[service]`) | question | R | Init.go:154 |
| | config_system | R | Init.go:165, handleAdmin.go:533 |
| | line_config | R (LINE channel config) | Init.go:176 |
| | member_account | R/W (owner/admin/report/member, อัปเดต Line_display_name) | Init.go:196, message.go:25,36, lineMessage.go:52, mainService.go:267, repositoryDB.go:591 |
| | message | W (เก็บ log ข้อความกลุ่ม) | message.go:60, webhook.go:71 |
| | member_log | W (admin log) | mainService.go:324 |
| | member_affiliated | R/W | repositoryDB.go:28,59,60,81, lineOAQuestion.go:1465 |
| | round_statement | R/W (เปิด/จบ/อัปเดตคู่มวย) | repositoryDB.go:104,116,131,145,317,340, handleReport.go:673,1523,1686 |
| | bet_statement | R/W (รายการเดิมพัน + aggregate รายงาน) | repositoryDB.go:155,288,429,518,649 |
| | deposit_statement | R/W (ฝาก/คืนเครดิต RETURN_CREDIT) | repositoryDB.go:533, handleAdmin.go:662,689,787,814 |

### external
| LINE API / third-party | purpose | auth |
|---|---|---|
| LINE Messaging API (api.line.me) | reply/push message, ดึง profile, loading, ทดสอบ webhook | Bearer **AccessToken** ดึงจาก `LineConfig.MultiLinebot[].AccessToken` ซึ่งเก็บใน Mongo collection `line_config` (models/lineConfig.go:19,30 ; ใช้ที่ lineMessage.go:28-29,90-92 ฯลฯ). **Channel SecretKey** ก็เก็บใน BotConfig (lineConfig.go:31) แต่ **ไม่ถูกนำมาใช้ verify ลายเซ็น webhook** |
| allapi.topupoffice.com | รายชื่อ/ตั้งค่าธนาคาร | ไม่มี auth header (mainService.go:211) |
| Agent API (database_config.api_url) | wallet/เกมมวย/ฝาก-ถอน | query string `key_agent`/`key` = ApiKey (agentService.go) |
| Web Main API (api_fontend_url) | ฝาก/ถอนทศนิยม, เพิ่มบัญชี | JWT HS256 เซ็นด้วย `API_TOKEN`, ส่ง header `authorization: Bearer` (mainService.go:165-176) |

## observations
- **LINE webhook ไม่ verify signature** — `/webhook/:key` (route.go:18, webhook.go:15) ประมวล body ตรงๆ โดยไม่เช็ค header `X-Line-Signature` กับ channel secret (grep "signature"/"X-Line" = ไม่พบเลย); ใครก็ยิง events ปลอมเข้ามาได้ → trigger ฝาก/ถอน/เพิ่มเครดิต
- **Channel AccessToken/SecretKey เก็บใน MongoDB** collection `line_config` field `multi_linebot.access_token`/`secret_key` (lineConfig.go:30-31) — secret_key มีแต่ไม่เคยใช้ (dead, ยืนยันความเสี่ยงข้อบน)
- **secret ฮาร์ดโค้ด/อ่อนแอใน repo**: .env commit จริง — `MONGODB_ENDPOINT=mongodb+srv://root:Zxcvasdf789@...` (.env:1), `ACCESS_SECRET=ABAsercretPayload` (.env:7), `API_TOKEN=ABAsercretPayload` (.env:4); ค่าเดียวกันโผล่ใน docker-compose.yml:10-16 (รวม REDIS_URL=52.77.27.18:6379) → credential รั่ว
- **endpoint การเงินไม่ auth จริง**: ฝาก/คืนเครดิต/ถอน ถูก trigger จาก webhook (ไม่ auth) ผ่าน path admin keyword (handleAdmin.go:662 สร้าง deposit_statement + AgentDeposit handleAdmin.go:669) — สิทธิ์เช็คแค่ LINE userId อยู่ใน owner/admin list (webhook.go:87-117) ซึ่งปลอม userId ได้เพราะไม่ verify signature
- **ลำดับ create-then-update ฝากเงินไม่ atomic**: handleAdmin.go:662 สร้าง deposit_statement → AgentDeposit (669) → ถ้า UpdateOneStatement ล้มเหลว จึง AgentWithdraw คืน (handleAdmin.go ~700) — มี window ที่เครดิตถูกเติมแล้วแต่ statement ค้างสถานะ process; ไม่มี idempotency key/lock
- **JWT ใช้ไลบรารี dgrijalva/jwt-go v3.2.0 (deprecated/มี CVE)** (go.mod:6) ; token report อายุ 15 นาที (authJWT.go:37)
- **เลือก LINE bot ตัวแรกแบบ buggy loop**: ทุกฟังก์ชันส่ง LINE ใช้ `var i bool` แล้ว `if !i { accessToken=v; break }` (lineMessage.go:18-23,57-64,79-86,105-112,129-136) — `i` เป็น false เสมอ จึงหยิบ token แรกที่ iterate map (ลำดับ map ไม่แน่นอน) แม้ตั้งใจรองรับ multi-linebot
- **error ถูกกลืน**: agentService หลายจุด log แล้วเดินต่อ (เช่น AgentAmount agentService.go:90 ไม่เช็ค error เลย); `ExternalCALL` json.Unmarshal สองรอบและ ignore error แรก (callback.go:31-32)
- **CORS เปิดกว้าง** `Access-Control-Allow-Origin: *` ทุก route รวม endpoint การเงิน (server.go:46-49)
- **redis ไม่มี password** (RedisConnect Password:"" redisService.go:77,89); webhook ใช้ `SetNX ttl=0` (ตั้งค่าถาวร) สำหรับ config (redisService.go:62) → config เปลี่ยนแล้วไม่อัปเดตจน reset เอง
- **ConfigInit panic ตอน boot** ถ้า set redis ล้มเหลว (Init.go:135,142) — service ล่มทั้งตัวจาก tenant เดียวพัง; แต่ DB connect fail แค่ log ไม่หยุด (server.go:23-27) แล้วใช้ resource ที่อาจ nil ต่อ
- **GetProfileGroup/Profile error handling อ่อน**: เช็คแค่ `res.Message == "Not found"` (lineMessage.go:35) ไม่เช็ค HTTP status
- **timeout outbound 30s** มีใน ExternalCALL (callback.go:16-18) แต่ LINE/agent call ไม่มี retry/circuit-breaker
