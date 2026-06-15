# Card: withdraw-auto
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go 1.23 (`go.mod:3`), MongoDB (mongo-driver v1.17.3), RabbitMQ (`streadway/amqp` v1.1.0), Selenium (`tebeka/selenium`), Appium (custom appdriver), Sentry, AWS S3 — `go.mod:5-27`
- **module**: `app` (module name แบบ local ไม่ใช่ URL) — `go.mod:1`
- **entrypoint**: `_cmd/main.go` — init โหลด `.env` ถ้า `MODE != production` (`_cmd/main.go:14-23`), main เลือก mode จาก `EVENTWITHDRAW` แล้วเรียก `app.StartServ` / `StartServQRPAY` / `StartServQRPAYNEW` (`_cmd/main.go:54-64`)
- **port**: ไม่พบ — ไม่มี HTTP server (ไม่มี `ListenAndServe`/gin/`EXPOSE`/PORT ใน code หรือ Dockerfile). เป็น RabbitMQ consumer worker ล้วน
- **Dockerfile/CI**: multi-stage `golang:1.23.4-alpine3.20` builder → `alpine:3.20` runtime ติดตั้ง `chromium chromium-chromedriver`, COPY `privateKbankApp.pem`/`publicKbankApp.pem`/`publicKTBApp.pem`, `CMD ["./app"]` (`Dockerfile:1-15`). CI: `.github/workflows/cicd.yml` env `NAME: withdraw`, self-hosted runner → Kaniko → JFrog, เขียน .pem จาก secrets ตอน build (`cicd.yml:38-40`)

## provides

### http
ไม่พบ — ไม่มี HTTP route/handler ใด ๆ (เป็น consumer)

### consume — RabbitMQ
URL มาจาก `dbConfig.RabbitMqURL` (MongoDB `database_config`) — `rabbitmqpub/conn.go:36`; ถ้า `MODE != production` hardcode CloudAMQP URL (`conn.go:34`)

| exchange/queue (EXACT + pattern) | handler | evidence |
|---|---|---|
| `WITHDRAW_{BankNumber}` (default withdraw mode) | `BankWorker.ProcessMessage` → dispatch ตาม bankCode | queue name สร้างที่ `rabbitmqpub/manager.go:437,446,453,460`; consume `manager.go:393-401`; routing `worker.go:197-235` |
| `CHANGE_ACCOUNT_{BankNumber}` (deposit / change_account) | same handler, `IsChangeAccount=true`, table `withdraw_statement_change_account` | queue `manager.go:435,451,456`; flag `worker.go:148,156-161`; repo `repository/isdb.repository.go:64-71` |
| single-bank mode: ใช้ env `BANK_NUMBER` → queue เดียว (`WITHDRAW_`/`CHANGE_ACCOUNT_`) | `refreshSingleBank` | `manager.go:432-438`, `manager.go:241-260` |
| multi-queue mode (ไม่มี `BANK_NUMBER`): ดึง `bank_config` ที่ active ทุกตัว refresh ทุก 60s สร้าง queue ต่อ bank | `doRefreshBankConfigs` + ticker | `manager.go:110-131`, `manager.go:262-358` |
| `QRPAYV2_WITHDRAW_{svQrConfig.KeyCode}` (mode `EVENTWITHDRAW=QRPAY_NEW`) | `startWithdrawQRPAY` → `qrpayment.StartServ` | `rabbitmqpub/amqp-qrpay.go:36-68,97-99` |
| `QRPAY_WITHDRAW_{BANK_NUMBER}` (mode `EVENTWITHDRAW=QRPAY`, legacy alone) | `ReadyWithdrawAloneAll` → `scb.StartServ` | `rabbitmqpub/rabbit-alone.go:211-236,289` |
| `OTP_{BANK}_{PhoneNumber}` — consume OTP จาก SMS gateway: `OTP_KBANK_`, `OTP_BAY_`, `OTP_KTB_`, `OTP_SCB_`, `OTP_TTB_`, `OTP_UOB_` | per-bank `*WebReciveOtp` (consume ใน netbank.rabbit.go) | `controller/kbank/netbank/netbank.rabbit.go:32`; `bay/...rabbit.go:33,100`; `ktb/...rabbit.go:31`; `scb/...rabbit.go:32`; `ttb/...rabbit.go:30`; `uob/...rabbit.go:38` |

- bankCode ที่ route ได้: SCB, KTB, BAY, KBANK, BCA, MANDARI(livin), TTB, UOB, KKP, GSB, LNH/LHB/LH, QRCODE, PAYMENT/PEER2PAY/UMPAY; โหมดพิเศษ: `GatewayConfig.IsWithdraw`→thirdgateway, `WithdrawMode==BANK_GATEWAY`→bankgateway, `BankBusiness`→bankbusiness — `worker.go:197-231`
- TRUEWALLET ถูก comment ออก 🟡 `worker.go:210-211`

### cron
ไม่พบ cron จริง — แต่ multi-queue mode มี ticker refresh `bank_config` ทุก 60 วินาที (`manager.go:120`); failed-bank cooldown `5min × retryCount` (max 30min) (`manager.go:325-328`)

## consumes

### http-out
client ส่วนใหญ่เป็น `net/http` default **ไม่มี timeout** (firebase, recaptcha, callAuthBank มี 20s)

| env key | base URL (🟡 = runtime/hardcode) | paths | evidence |
|---|---|---|---|
| `BANK_AUTH` | fallback hardcode `https://bank-auth-372sumveiq-as.a.run.app/api` 🟡 | `/ktb`,`/kbank`,`/kkp` + `/app/auth`,`/netbank/auth`,`/sign` | `service/callAuthBank/callAuthBank.go:16,20,89,93,161-171` |
| `URL_GATEWAY` | runtime | thirdgateway withdraw | `controller/thirdgateway/main.go:26` |
| `URL_PAYMENT` | runtime | `/api/v2/check_balance_p2p`,`/api/WithdrawQyPaymentByService`,`/api/v2/create-withdraw`,`/api/ConfirmWithdrawQrpayment`,`/api/v2/confirm-withdraw` | `service/speedpay/main.go:34,36,75,77,113` |
| `URL_BANK_BUSINESS` | runtime | bank business transfer | `controller/bankbusiness/business/business.main.go:28` |
| `URL_PAYMENT` / `DOMAIN_CALLBACK_PAYMENT` | runtime | payment curl | `controller/payment/curl/curl.main.go:74` |
| `GSB_LOCAL` / `GSB_SERVICE` | runtime override | GSB app api | `controller/gsb/appapi/func.go:37-38,112-113,157-158,207-208`, `func_by_gsb_service_thai.go:327` |
| `KPLS_LOCAL` / `KPLUS_SERVICE` | runtime | KBANK K+ service | `controller/kbank/appapi/appapi.func.go:168,214,269,453,596` |
| `KTB_SERVICE` / `SCB_SERVICE` | runtime | KTB/SCB service | `controller/ktb/appapi/service.go:24`, `controller/scb/appapi/scb-service.go:139` |
| `LNH_SERVICE` | runtime | LNH service | `controller/lnh/appapi/func.go:95` |
| `URL_K8S` / `K8S_KEY` / `PROJECT` | runtime | uninstall helm release ของ pod single-bank เก่า | `service/k8sService/main.go:24-26`, `manager.go:465-485` |
| `URL_BYPASS` | runtime | bypass recaptcha audio | `helper/recaptcha.go:177` |
| `OCR_SERVER` | fallback `http://13.212.68.203:8071/file` 🟡 | OCR | `service/tesseract/tesseract.main.go:88-90` |
| `PROXY_APP_BANK` | fallback hardcode `https://fast.lupin.host` 🟡 | `/api/job`,`/api/device-forward`,`/api/tranfer-status` (SCB appproxy) | `repository/office.repository.go:34-36`; `controller/scb/appproxy/appproxy.go:121,157`, `bkbwait.go:233` |
| (hardcode) | `https://withdraw-monitor-new.firebaseio.com/`, `https://monitor.thezeus.online/api/netbank/withdraw`, `https://go-care-679ce-default-rtdb.firebaseio.com/Withdraw/` 🟡 ไม่มี auth | firebase realtime DB | `service/firebase/upWithdraw.go:22,59,130,146,239` |
| (hardcode) | `https://checkbank.thesonicblue.xyz/insertBank` 🟡 | push check bank (JWT HS256 secret `ASka203Vj6` hardcode) | `service/report/callback.go:33,37` |
| (hardcode) | `https://notify-api.line.me/api/notify` 🟡 | LINE notify screenshot | `service/lineNotify/line-notify.go:16,118` |
| (hardcode) | `https://rt10.kasikornbank.com/kplus-service`, `https://android.clients.google.com/...`, `https://api.newnext.krungthai.com/`, `https://www.ktbnetbank.com/...` 🟡 | bank/Google GCM endpoints จริง | `service/kplusgenerator/shared/client.go:25-26`, `gcm.go:45,85`, `service/ktbgenerate/Client.go:12`, `service/captcha/callSound.go:40` |
| `CallbackApiWithdraw` (จาก dbConfig) | runtime | `/withdraw_statement` callback กลับ office | `service/report/callback.go:45-58` |

### publish — RabbitMQ
| exchange/queue (exact) | payload | evidence |
|---|---|---|
| queue `WITHDRAW_{BankNumber}` (re-withdraw / retry) | `models.ReqData{Service,ID,RewithdrawUnit+1}` | `controller/{bay,kbank,ktb,scb}/netbank` `CreateQueueWithdraw`; เช่น `controller/scb/netbank/netbank.main.go:246-247`, `netbank.rabbit.go:126`; `repository/isdb.repository.go:181-187` |
| exchange `STATEMENT` หรือ `STATEMENT_{SERVICE}` (fanout, ถ้า `GROUP=NEWTOPUP`) type=`WITHDRAW` | `models.WithdrawStatement` JSON persistent | `service/report/publish.go:35-68` (เรียกตอน `UpStatusSuccess` `repository/isdb.repository.go:264`) |
| routing key `BANK_STATEMENT_WITHDRAW` (default exchange) | `{service,bank_number,key_session,cookies}` | `service/report/publish.go:128-137` |

### data — MongoDB (raw driver, no ORM)
connection จาก env `DB_ENDPOINT`(+`DB_DBNAME`)/`DB_ENDPOINT_ALLINONE`, multi-tenant ผ่าน `database_config.db_host_name` ต่อ service — `db/conn.go:39-73,117-162`; `connectTimeout=5s` connect, query ctx 60s (`db/conn.go:34-37`)

| DB/collection | R/W | evidence |
|---|---|---|
| `database_config` | R | `repository/office.repository.go:82,89`, `db/conn.go:120,168,215` |
| `bank_config` / `bank_config_other` | R/W (อ่าน config, เขียน credential/access_token) | `office.repository.go:128,166,278`, `service/bankCredential/bankCredential.go:160`, `repository/isdb.repository.go:411` |
| `withdraw_statement` | R/W (อ่านรายการถอน, อัพเดท status/is_success/comment) | `repository/isdb.repository.go:48,56,83,161,...` |
| `withdraw_statement_qrpay` | R/W | `isdb.repository.go:42,45`, `office.repository.go:310,344` |
| `withdraw_statement_change_account` | R/W | `isdb.repository.go:51,67` |
| `salary_statement` | R/W (mode PAYROLL) | `isdb.repository.go:54` |
| `withdraw_amount` | R/W (aggregate + insert เพิ่มทุน) | `isdb.repository.go:434,473` |
| `working_log` | W | `isdb.repository.go:487` |
| `queue_payment` | W | `isdb.repository.go:494` |
| `config_system` | R | `isdb.repository.go:510` |
| `bank_list_image` | R/W | `office.repository.go:190,198,216,224,233` |
| `summary_deposit` | R | `office.repository.go:322` |

### external — banks + credentials
| bank/api | purpose | auth (เก็บที่ไหน) | evidence |
|---|---|---|---|
| SCB (EasyWeb netbank `https://www.scbeasy.com/...`, SCB Anywhere ผ่าน proxy `fast.lupin.host`, scb-service) | login/โอนเงินถอน | `bankConfig.Password`/`PinBank`/`AccessToken`/`DeviceId` จาก `bank_config` (MongoDB) | `controller/scb/netbank/netbank.main.go:18`, `appproxy/*` |
| KBANK (K+ generator + netbank + appapi) | โอน/auth | `bankConfig.PinBank`/`Password`/`AccessToken`; pem keys (`privateKbankApp.pem`) | `controller/kbank/appapi/appapi.func.go:158,200,251,466,527,573`, `Dockerfile:11-12` |
| KTB, BAY, UOB, KKP, GSB, LNH, TTB | netbank/appapi โอนถอน | `bankConfig` credentials | `controller/{ktb,bay,uob,kkp,gsb,lnh,ttb}/...` |
| BCA, MANDARI(livin), TRUEWALLET | Appium mobile automation (`UrlSeleniumWithdraw`+`DeviceId`) | `bankConfig.Password`/`PinBank` ส่งเข้า app field | `controller/bca/appium/appium.func.go:136,407,718,940`; `worker.go:64-81` |
| bank-auth service (`BANK_AUTH`) | sign/auth ส่ง `pin`,`card_id`,`device_id` | ส่ง credential ออกนอก | `service/callAuthBank/callAuthBank.go:178-191` |
| AWS S3 `withdraw-slip` (ap-southeast-1) | อัพสลิป | **AWS key/secret hardcode ใน source** | `service/uploadfiles/uploadfiles.go:20-21,47`; `service/minestone/mimestone.img.go:17-18` |

credential เก็บ/เข้ารหัส: AES-CFB ด้วย env `BANK_CREDENTIAL_KEY`, hash SHA1 (password/pin/device) เก็บใน `bank_config.secret_token`/`pin_bank`/`password` — `service/bankCredential/bankCredential.go:36,59,87-116,146-160`

## observations
- **(1)** Hardcoded AWS access key + secret ใน source: `AKIA57MPBYFHKEWZ25LN` / `KAzp5OUR0CTkcyM+...` — `service/uploadfiles/uploadfiles.go:20-21` และ `service/minestone/mimestone.img.go:17-18`
- **(2)** Hardcoded JWT signing secret `"ASka203Vj6"` (HS256) สำหรับ push check bank — `service/report/callback.go:33`
- **(3)** Hardcoded production RabbitMQ CloudAMQP URL พร้อม username/password ใน source (ใช้เมื่อ `MODE!=production`) — `rabbitmqpub/conn.go:34`, `service/rabbit/retry_withdraw.go:15`
- **(4)** เงินจริง — retry/rewithdraw re-queue ไป `WITHDRAW_{BankNumber}` เมื่อ status 12 โดยไม่มี idempotency key, อิง `RewithdrawUnit < 3/rewithdrawAgain` เป็น guard เท่านั้น → เสี่ยงโอนซ้ำถ้าธนาคารโอนสำเร็จแต่ web ไม่ตอบ — `repository/isdb.repository.go:168-192`, `controller/scb/netbank/netbank.main.go:246-247`
- **(5)** `processLoop` ack message (`Delivery.Ack(false)`) ทุกครั้งหลัง `ProcessMessage` แม้ withdraw error → ข้อความถูก ack ทิ้งเมื่อ error (อาศัย DB status เป็นแหล่งความจริงแทน) — `rabbitmqpub/manager.go:218-221`
- **(6)** `failOnError` ใช้ `log.Fatalf` (ทำทั้ง process ตาย) ถูกเรียกจาก consumer หลายจุดรวมถึง OTP consumer — `controller/kbank/netbank/netbank.rabbit.go:17-20,25,28,39,49`
- **(7)** HTTP client หลายตัวไม่มี timeout บน money/monitor path: firebase (`service/firebase/upWithdraw.go:34,75,116,147`), recaptcha bypass (`helper/recaptcha.go:176`), report.Publish dial ไม่ตั้ง timeout
- **(8)** Firebase Realtime DB endpoint เปิด public ไม่มี auth token (`...firebaseio.com/.json` PATCH ตรง) — รั่ว balance/bank info — `service/firebase/upWithdraw.go:22,59,239`
- **(9)** Swallowed error บน money/credential ops: `primitive.ObjectIDFromHex(reqData.ID)` ทิ้ง err (`worker.go:171`, `amqp-qrpay.go:82`); `InsertWithDrawAmount` error แค่ `fmt.Println` (`isdb.repository.go:295,322,351`)
- **(10)** multi-queue mode: ทุก bank แชร์ `msgChan` buffer=1 + single `processLoop` goroutine + QoS(1) → ทุกธนาคารถอน serialize กันหมด, ธนาคารช้าตัวเดียว block ทั้ง worker — `manager.go:76,102,205-239`
- **(11)** plaintext credential ออกนอกระบบ: ส่ง `pin`/`password`/`device_id` เป็น JSON ไปยัง `BANK_AUTH` / bank service และพิมพ์ `reqData`/payload ลง stdout หลายจุด (`worker.go:170`, `rabbit-alone.go:121`)
- **(12)** dev fallback bank base URLs ชี้ production จริงแบบ hardcode (`bank-auth-372sumveiq-as.a.run.app`, `fast.lupin.host`, `rt10.kasikornbank.com`, `api.newnext.krungthai.com`) — `callAuthBank.go:20`, `office.repository.go:36`, `kplusgenerator/shared/client.go:25`
- **(13)** local ChromeDriver จำกัดเฉพาะ BAY/UOB/KBANK; bank อื่นที่ `UrlSeleniumWithdraw` ว่างจะ skip driver เงียบ ๆ (return nil ไม่ error) → worker active แต่ถอนไม่ได้ — `worker.go:88-94`
- **(14)** worker เรียก `k8sService` uninstall helm release ของ pod เก่าอัตโนมัติเมื่อ migrate (มี side-effect ต่อ k8s cluster) — `manager.go:353,465-485`
- **(15)** deps เก่า/เสี่ยง: `dgrijalva/jwt-go v3.2.0` (unmaintained, มี CVE), `streadway/amqp` (deprecated → rabbitmq/amqp091) — `go.mod:8,21`
