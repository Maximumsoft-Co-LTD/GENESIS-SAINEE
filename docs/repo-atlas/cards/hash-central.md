# Card: hash-central
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go 1.25.5 + Gin + MongoDB (raw driver) + Redis (go-redis v9 + thorlock/redislock) + OpenTelemetry (otelgo v0.0.7) + Prometheus (ginmetrics) — `go.mod:1-25`
- **module**: `app` (`go.mod:1`)
- **entrypoint**: `_cmd/main.go:20-26` — `TYPE==""` → `app.StartService()` (HTTP server), `TYPE=="CRONJOB"` → `app.StartServiceCronjob()` (gocron worker ไม่เปิด HTTP)
- **port**: `:8001` hardcode ใน `app.go:148-151` (`srv := &http.Server{Addr: ":8001"}`); `/metrics` ginmetrics เปิดบน router เดียวกัน (`app.go:82`)
- **Dockerfile**: build `golang:1.25.5-alpine3.23` → run `alpine:3.23`, `TZ=Asia/Bangkok`, รัน test ชุด Channel ใน build stage, `CMD ["./app"]`, ไม่มี EXPOSE (`Dockerfile:1-24`)
- **CI**: `.github/workflows/workflow.yaml` — Kaniko → JFrog, branch→tag (dev/uat/v*)
- **บังคับตอน boot**: ต้องมี `QR_DECODER_URL` มิฉะนั้น app ไม่ start (`service/qrclient/choose.go:15-18`, `app.go:69-73`); Mongo + Redis connect ไม่ได้ = ไม่ start (`app.go:59-63,97-113`)

## provides

### http
ทุก route ใต้ `/api` (group `app.go:145`) ยกเว้น webhook ที่ root (`routes/pxy-man.go:11-17`). Auth มีจุดเดียว: group `/api/v2` ฝั่ง proxy ใช้ `middlewares.AuthRequired` เทียบ header `api-key` กับ env `API_KEY_PROXY` (`routes/main.go:67-68`, `middlewares/auth.go:14-38`). **route อื่นทั้งหมดไม่มี auth**

| METHOD | path | auth | evidence |
|---|---|---|---|
| POST | /api/hashlayout | ไม่มี | routes/main.go:32 |
| POST | /api/hashlayout-list | ไม่มี | routes/main.go:33 |
| POST | /api/get-channel | ไม่มี | routes/main.go:34 |
| POST | /api/change-channel | ไม่มี | routes/main.go:35 |
| POST | /api/bank-account | ไม่มี | routes/main.go:52 |
| POST | /api/slip-verify | ไม่มี | routes/main.go:53 |
| POST | /api/line-bot-config | ไม่มี | routes/main.go:54 |
| POST | /api/v2/slip-verify | ไม่มี (group v2 ของ slip ไม่ติด middleware) | routes/main.go:55-57 |
| POST | /api/v2/use-slip-verify | ไม่มี | routes/main.go:58 |
| GET | /api/v2/profile-slip | ไม่มี | routes/main.go:59 |
| POST | /api/con-proxy-test | ไม่มี | routes/main.go:65 |
| POST | /api/v2/get-proxy | api-key | routes/main.go:68-69 |
| POST | /api/v2/callback-proxy | api-key | routes/main.go:70 |
| POST | /api/v2/status/proxy-address | api-key | routes/main.go:71 |
| POST | /api/v2/status/proxy-config | api-key | routes/main.go:72 |
| POST | /api/v2/get-bank-config | api-key | routes/main.go:73 |
| POST | /api/v2/edit-address/bank-config | api-key | routes/main.go:74 |
| POST | /api/v2/clear-bank-config | api-key | routes/main.go:75 |
| POST | /api/v2/change/status-proxy | api-key | routes/main.go:76 |
| POST | /api/v2/change/ip-proxy | api-key | routes/main.go:77 |
| POST | /api/v2/open-close/proxy-config | api-key | routes/main.go:78 |
| POST | /api/v2/open-close/proxy-address | api-key | routes/main.go:79 |
| POST | /api/v2/open-close/proxy-address/bank-status | api-key | routes/main.go:80 |
| POST | /api/v2/open-close/proxy-config/bank-status | api-key | routes/main.go:81 |
| POST | /api/v2/reset-proxy | api-key | routes/main.go:82 |
| POST | /api/v2/update-status-find | api-key | routes/main.go:83 |
| GET | /api/v2/reset-factory | api-key | routes/main.go:84 |
| POST | /api/get-device-botapp | ไม่มี | routes/main.go:88 |
| POST | /api/save-device-botapp | ไม่มี | routes/main.go:89 |
| POST | /api/upload-withdraw-img | ไม่มี | routes/main.go:90 |
| POST | /api/bot/gen-device | ไม่มี | routes/main.go:95 |
| GET | /api/bot/get-data-bank | ไม่มี | routes/main.go:96 |
| POST | /api/bot/update-queue | ไม่มี | routes/main.go:97 |
| POST | /api/bot/update-status-gen-device | ไม่มี | routes/main.go:98 |
| POST | /api/bot/check-sms-status | ไม่มี | routes/main.go:99 |
| POST | /api/bot/check-status-gen-device | ไม่มี | routes/main.go:100 |
| POST | /api/bot/get-data-bank-by-office | ไม่มี | routes/main.go:101 |
| POST | /api/bot/update-image-base64 | ไม่มี | routes/main.go:102 |
| POST | /api/bot/cancel-gen-device | ไม่มี | routes/main.go:103 |
| POST | /api/save-face-data | ไม่มี | routes/main.go:107 |
| POST | /api/get-face-data | ไม่มี | routes/main.go:108 |
| POST | /webhook (SCB device hook) | ไม่มี | routes/pxy-man.go:13 |
| POST | /webhook-by-proxy | ไม่มี | routes/pxy-man.go:14 |
| POST | /get-device-generate | ไม่มี | routes/pxy-man.go:15 |
| POST | /update-device-generate | ไม่มี | routes/pxy-man.go:16 |
| GET | /metrics (Prometheus) | ไม่มี | app.go:82 |

### consume
ไม่พบ — ไม่มี RabbitMQ ใน repo นี้ (ไม่มี import amqp ใน go.mod / source)

### cron
| schedule | what | evidence |
|---|---|---|
| ทุก 20 นาที (เมื่อ `TYPE=CRONJOB`) | `ProcessFindProxy` — ดึง proxy list จากอุปกรณ์ XPROXY แล้ว sync ลง `proxy_address` | controller/find_proxy.go:24, routes/main.go:161-164, app.go:190-200 |
| ทุก 30 นาที (CRONJOB) | `ProcessUpdateProxyAuto` | controller/find_proxy.go:25 |
| ทุก 10 นาที (CRONJOB) | `ProcessUpdateBankUseCurrent` | controller/find_proxy.go:26 |
| background goroutine ทุก `KTB_POOL_TOPUP_INTERVAL_SECONDS` (default 30s) ใน HTTP mode | `RunPoolTopup` pre-warm KTB device pool (เมื่อ `KTB_DEVICE_POOL=true` และ `KTB_POOL_TARGET_HEALTHY>0`) | routes/main.go:40-50, controller/ktb_device_pool.go:267-296 |
| 🟡 `RouteServCronjob` → `CronjobCheckProxy` ถูก comment ทิ้ง | — | app.go:199, routes/main.go:156-159 |

## consumes

### http-out
| env key / source | base URL | paths | evidence |
|---|---|---|---|
| (hardcode) | `https://api.newnext.krungthai.com` | `/v2/auth/prelogin/grant?grant_type=client_credentials` (OAuth grant + ECDSA/HMAC signing), `/api/qr/v1/transaction-details` (ตรวจสลิป KTB) | controller/slip_verify_ktb.go:73,529 |
| `KTB_PROXY_URL` 🟡 (optional, มี user:pass ฝังใน URL) | proxy สำหรับ KTB ทั้ง 2 call; transport-fail → retry direct | controller/ktb_proxy.go:18 |
| `SLIP2GO_API_KEY` (Bearer) | `https://connect.slip2go.com` (hardcode) | `/api/verify-slip/qr-code/info` — fallback เมื่อ KTB fail | service/verifySlip/slip2go.go:22,37-55 |
| (hardcode, apiKey จาก collection `line_slip_config`) | `https://developer.easyslip.com` | `/api/v1/me`, `/api/v1/verify`, `/api/v1/verify/truewallet` | controller/slip_api_easyslip.go:19,37,81,90 |
| `QR_DECODER_URL` | maan-qrdecoder service | decode QR จากภาพสลิป (timeout 15s/20s) | service/qrclient/choose.go:15, service/qrclient/http_client.go:43-58 |
| `SCB_SERVICE` | SCB bank service | POST `/api/first-login` (จาก webhook device-generate flow) | controller/device-generate.go:46,282 |
| `GSB_SERVICE` | GSB (MyMo) bank service | POST `/api/first_login` (ส่ง access_token, pin, citizen_id, phone_number) | controller/bot-gen-device.go:316-327,366-372 |
| `SMS_URL` | SMS gateway | POST `/global/log-check-sms/HASH_CENTRAL` | controller/bot-gen-device.go:493-497 |
| 🟡 `data.OfficeAPI` (จาก document ใน Mongo, runtime) | office API per-tenant | POST `/api/bank-config-set-access-token/{service}` | controller/bot-gen-device.go:343 |
| 🟡 `bankAccount.DomainURL` (จาก collection `bank_account`, runtime) | bank scraper ปลายทาง | POST scan slip — ส่ง `access_token` ของบัญชีไปยัง URL ที่อยู่ใน DB | controller/slip_verify_hash.go:771-779,1083 |
| (hardcode, apiKey จาก `proxy_config` ใน DB) | `https://proxy.webshare.io` | GET `/api/v2/proxy/list/` | controller/function_proxy.go:26-33 |
| (hardcode) | `https://fasteasy.scbeasy.com:8443` | GET `/v2/customer/countries` — probe ทดสอบว่า proxy ใช้ได้ | helper/main.go:62-75 |
| 🟡 `proxyConfig.ProxyConnect.Host` (จาก DB, runtime) | XPROXY device API `http://host:port` | ดึง/หมุน proxy list (cron) | controller/find_proxy.go:46-49 |
| `TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID` | `https://api.telegram.org` | `/bot{token}/sendMessage` — infra alert | service/telegram/client.go:23-24,63 |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP collector (no-op ถ้าว่าง) | trace/metric export | service/telemetry/telemetry.go:20 |
| (hardcode creds) | AWS S3 `ap-southeast-1` bucket `withdraw-slip` / `uploadfile-customer` | PutObject ภาพ withdraw + face data | controller/botapp.go:89-98,158-176 |

### publish
ไม่พบ — ไม่มี RabbitMQ publish

### data
DB: MongoDB เดียว `DB_ENDPOINT`/`DB_NAME` (`db/connect.go:42-43`); Redis `REDIS_URL`/`REDIS_DB`/`REDIS_PSW` (`app.go:104-109`)

| collection | R/W | evidence |
|---|---|---|
| bank_proxy | R/W | controller/function_proxy.go:54,78,116,262,519 |
| proxy_config | R/W | controller/function_proxy.go:143,167; controller/find_proxy.go:36,106 |
| proxy_address | R/W (+bulk) | controller/find_proxy.go:94-103; service/xproxy/const.go:7 |
| bank_config | R/W (xproxy) | service/xproxy/const.go:8 |
| bank_config_service | W (เก็บ access_token+pin+device_id, unique index hash) | controller/secert-enc.go:43-50 |
| auth_bank | R/W | controller/bank_auth.go:20,60,112 |
| transaction_statement | R/W (dedup หลัก) | controller/hash.go:25,481,489; controller/slip_verify_hash.go:460,549 |
| log_hash | W | controller/hash.go:26,357 |
| channel / request_hash / logs | R/W | controller/request_hash.go:14-15,132 |
| slip_verify | R/W | controller/slip_verify_hash.go:88,446,528,828 |
| bank_account_slip | R/W (ชื่อบัญชี + self-heal) | controller/slip_verify_hash.go:84,879,893,906 |
| bank_account | R (สุ่มบัญชีตรวจสลิป) | repo/repo.go:286-294 |
| line_slip_config | C/U/D | controller/slip_verify_hash.go:1132,1137,1142 |
| log_slip_verify | W | controller/slip_verify_hash.go:1068 |
| slip_verify_failures | W (async, gated `SLIP_FAILURE_CAPTURE_ENABLED`) | controller/slip_failure_capture.go:23,48 |
| monitor_ktb_raw | W (TTL `KTB_RAW_LOG_TTL_DAYS`) | controller/slip_verify_ktb_log.go:25,44 |
| monitor_slipverify | W (TTL `SLIP_VERIFY_LOG_TTL_DAYS`) | controller/slip_verify_observe.go:31,153,163 |
| device_generate / log_device_generate | R/W (mongox) | controller/device-generate.go:36-37,51-52 |
| bot_gen_devices | R/W | controller/bot-gen-device.go:60-541 (หลายจุด) |
| botapp_devices | R/W | controller/botapp.go:49,57,80 |
| **Redis keys** | `ktb:auth`, `ktb:device-id`, `ktb:devices`, `ktb:devices:healthy` (pool), `slip_verify:cache:*` (TTL 18000s), `qr_decode:cache:*` (TTL 86400s), distributed lock (thorlock) | app.go:123-143; controller/slip_verify_ktb.go:44-70; controller/slip_verify_cache.go:51; controller/qr_decode_cache.go:70; service/thorlock/client.go:31-34 |

### external
| third-party | purpose | auth | evidence |
|---|---|---|---|
| KTB (Krungthai NEXT API) | ตรวจสลิป — ปลอมตัวเป็น mobile app (`User-Agent: okhttp/3.14.9`, `X-Device-Model: Samsung SM-A045F`, `X-Platform: android/12`) | OAuth client_credentials (`KTB_OAUTH_CLIENT_ID/SECRET`) + ECDSA P-256 (`KTB_EC_PRIVATE_KEY_B64`) + HMAC (`KTB_HASH_PASS`) + per-device X-Device-Id pool | controller/slip_verify_ktb.go:73,115-124,529-541; service/ktbcrypto/ |
| slip2go | ตรวจสลิป fallback | Bearer `SLIP2GO_API_KEY` | service/verifySlip/slip2go.go:37-46 |
| EasySlip | ตรวจสลิป/truewallet per-tenant | Bearer token จาก DB (`line_slip_config`) | controller/slip_api_easyslip.go:36,67 |
| GSB (ผ่าน GSB_SERVICE ภายใน) | first_login MyMo — ส่ง PIN/citizen id | ไม่มี auth บน call | controller/bot-gen-device.go:316-327 |
| SCB (fasteasy.scbeasy.com) | proxy liveness probe | ไม่มี | helper/main.go:62 |
| Webshare.io | เช่า proxy pool | `Authorization: <api_key>` จาก proxy_config | controller/function_proxy.go:26-33; helper/main.go:44 |
| Telegram Bot API | infra alert + dedup TTL | bot token env | service/telegram/client.go:23,63 |
| AWS S3 | เก็บภาพ withdraw slip / face data | **static credentials hardcode ใน source** | controller/botapp.go:94-95,158-159 |

## observations
- **Hardcoded AWS access key + secret ใน source**: `AWS_ACCESS_KEY_ID=AKIA57MPBYFHKEWZ25LN`, `AWS_SECRET_ACCESS_KEY=KAzp5OUR0...` — controller/botapp.go:94-95 (ใช้จริงที่ botapp.go:159)
- **Hardcoded RSA private key (PKCS1) ทั้ง key ใน source** ใช้ถอดรหัส face-data payload — service/decrypt/main.go:10-12
- **Bank credentials เก็บ plaintext ใน Mongo**: `/api/bank-account` รับ `access_token` + `pin` + `device_id` แล้ว insert ตรงเข้า `bank_config_service` ไม่เข้ารหัส — controller/secert-enc.go:21-50; เช่นเดียวกับ flow bot-gen-device ที่ส่ง pin/citizen_id เป็น JSON เปล่าไป GSB_SERVICE — controller/bot-gen-device.go:308-327
- **Endpoint เกือบทั้งหมดไม่มี auth** (slip-verify, hashlayout, webhook SCB, bot device, face data, /metrics) — มีแค่ group proxy `/api/v2` ที่เช็ค `api-key` (routes/main.go:68) และเทียบ string ตรง ๆ ไม่ constant-time, ตอบ 500 แทน 401 — middlewares/auth.go:17-24,33-34
- **CORS เปิดกว้าง `*` ทุก method/header** — app.go:29-31
- **KTB integration = การ reverse-engineer app ธนาคาร**: ปลอม device headers + ECDSA signing เลียนแบบ KTB NEXT (slip_verify_ktb.go:529-541) — เปราะต่อ anti-abuse (มีประวัติ 403 storm; 403 → ถอด device ทันที controller/slip_verify_ktb.go:585-593) และ `KTB_POOL_TARGET_HEALTHY` default 300 device/credential ถูกระบุในตัวโค้ดว่า AGGRESSIVE เสี่ยงโดน ceiling (controller/ktb_device_pool.go:267-281)
- **`KTB_PROXY_URL` ฝัง credentials proxy ใน env URL** — controller/ktb_proxy.go:18; ส่วน proxy probe พิมพ์ URL พร้อม user:pass ลง stdout `fmt.Println("Proxy URL", urlProxy)` — helper/main.go:67
- **SSRF/credential-forward โดย design**: `scanSlip` POST `access_token` ของบัญชีธนาคารไปยัง `bankAccount.DomainURL` ที่อ่านจาก DB โดยไม่ validate — controller/slip_verify_hash.go:771-779; เช่นเดียวกับ `data.OfficeAPI` จาก DB — controller/bot-gen-device.go:343
- **`helper.CallAPI` แปลง payload เป็น query string บน URL** (ค่า sensitive ติดไปกับ URL/log ของ peer) — helper/main.go:16-37
- **Webhook `/webhook` (SCB) ไม่มี auth** แค่ regex `account_number=(\d{10})` จาก body แล้ว upsert device_id ลง `device_generate` — ปลอม request ได้อิสระ — controller/device-generate.go:56-110, routes/pxy-man.go:13
- **SHA1 ใช้สร้าง dedup hash ของ statement/proxy** (ชนกันได้ในทางทฤษฎี; ใช้เป็น identity ของเงินฝาก) — controller/find_proxy.go:58-61; SHA256 ใช้ที่ secert-enc.go:38-40
- **`time.Sleep(5 * time.Minute)` ใน request-path loop ของ bank_proxy** — controller/bank_proxy.go:181
- **Error ถูกกลืนหลายจุด**: `fmt.Errorf` ไม่ assign (controller/function_proxy.go:365,414), index-create/insert ไม่เช็ค error (controller/secert-enc.go:49-50), `go Logs(...)` fire-and-forget ทั่ว function_proxy.go
- **Slip-verify cache เป็น kill-switch env ไม่มี stampede lock** (`SLIP_VERIFY_CACHE_DISABLED`, TTL 18000s) — controller/slip_verify_cache.go:51-70
- **Trace มี PII เต็ม (เลขบัญชี) เว้นแต่ตั้ง `SLIP_TRACE_PII_DISABLED`** — service/telemetry/attributes.go:38 และ response body ธนาคารถูก capture เข้า trace (~4KB) by default — service/telemetry/response_body.go
- จุดดี: KTB call มี per-attempt timeout 10s + backoff (slip_verify_ktb.go:27-41), slip2go client timeout 15s (slip2go.go:55), qrclient ตั้ง transport timeout ครบ (http_client.go:43-58), graceful shutdown 30s (app.go:159-186)

## CONDENSED
- TYPE: Go/Gin REST — dedup hash กลาง + ตรวจสลิป (KTB/slip2go/EasySlip) + proxy pool manager + bot device-gen + face data
- PORT: 8001 (app.go:149)
- HTTP-IN: /api/hashlayout(-list), /api/slip-verify (+v2), /api/bank-account, /api/v2 proxy mgmt (api-key), /api/bot/*, /api/save-face-data, /webhook (SCB)
- HTTP-OUT: api.newnext.krungthai.com, connect.slip2go.com, developer.easyslip.com, QR_DECODER_URL, SCB_SERVICE, GSB_SERVICE, SMS_URL, proxy.webshare.io, fasteasy.scbeasy.com, api.telegram.org, S3; 🟡 DomainURL/OfficeAPI จาก DB
- QUEUE: ไม่พบ
- DB: Mongo (transaction_statement, slip_verify, bank_proxy, proxy_config/address, bank_config_service, bot_gen_devices, monitor_*) + Redis (ktb pool, slip/qr cache)
