# Card: SCHEDULE_SERVICE_BANK
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go 1.18 (`go.mod:3`), MongoDB (mongo-driver v1.11.2), RabbitMQ (`streadway/amqp` v1.0.0), gocron (`go-co-op/gocron` v1.37.0), Telegram bot API — `go.mod:5-23`
- **module**: `servicebank` — `go.mod:1`
- **entrypoint**: `_cmd/main.go` — init โหลด `.env` ถ้า `MODE != production` (`_cmd/main.go:13-22`), main เรียก `servicebank.StartServ()` (`_cmd/main.go:30`)
- **port**: ไม่พบ — ไม่มี HTTP server (ไม่มี `ListenAndServe`/gin/`EXPOSE`/PORT). เป็น cron scheduler ล้วน
- **Dockerfile/CI**: multi-stage `golang:1.18-alpine3.16` builder → `alpine:3.13` runtime, สร้าง `cookies/` + `balance_bank_config/`, `CMD ["./app"]` (`Dockerfile:1-12`). หมายเหตุ: builder go1.18 แต่ runtime go.mod 1.18 ตรงกัน. CI: `.github/workflows/build.yaml` env `NAME: service-bank`, self-hosted → Kaniko → JFrog → webhook update image (`build.yaml:4,54,80`)

## provides

### http
ไม่พบ — ไม่มี HTTP route/handler (เป็น scheduler)

### consume — RabbitMQ
ไม่พบ — service นี้เป็นฝั่ง **publish** อย่างเดียว ไม่มี consumer

### cron — gocron (timezone UTC)
| schedule | what | evidence |
|---|---|---|
| ทุก 3 นาที, SingletonMode, StartImmediately | `proccessAllDbService` — loop ทุก service ใน `database_config` แล้วยิงงานขนานกันด้วย WaitGroup | `app.go:52-54,91-138` |

งานย่อยที่รันในแต่ละรอบ (ต่อ service, goroutine ขนาน) — `app.go:99-135`:
1. `ctlUpdateStatusWithdraw.UpdateStatusCurrentWithdraw` — callback สถานะถอน payment
2. `ctlTriggerBankAccount.ProccessRecreate` — ส่ง bank_statement → RabbitMQ สร้างเครดิต
3. `ctlChangeAccount.ChangeAccWithdraw` — สร้างรายการโยกเงิน
4. `ctlNotificationAccount.LineNotificationAccessWaitGroup` — แจ้งเตือน limit/whitelist/ban
5. `ctlWithdrawTransferSCB.CreateWithdrawTransferBussinessSCB` → sleep 1 นาที → `ctlGetStatementTransferSCB.GetStatementTransferBussinessSCB` (SCB Business: create→confirm แล้ว verify, ต้องเรียงลำดับ)
6. `ctlSwitchAccount.SwitchAccountChange` — สลับการแสดงบัญชี (1 ครั้ง/รอบ)

- โค้ดเดิม env-driven (`SWITCH_ACCOUNT`/`CHANGE_ACCOUNT`/`NOTIFICATION`/`TYPE_EVENT`) ถูก comment ออกทั้งหมด 🟡 — `app.go:39-87`, `_cmd/main.go:29`

## consumes

### http-out
ทุก client เป็น `net/http` — **CallApiV2 (SCB Business) และ AllApi/CallApi ไม่มี timeout**; bank-transfer มี 200s, ban-check ไม่มี

| env key | base URL (🟡 = runtime/hardcode) | paths | evidence |
|---|---|---|---|
| `SCB_BUSINESS_API` | runtime | `/api/scbanywhere/transfer-create`, `/api/scbanywhere/transfer-confirm`, `/api/scbanywhere/transfer-check` | `controller/ctl-withdraw-transfer-scb-business/main.go:202,246,289`; `controller/ctl-get-statement-transfer-scb-business/main.go:172,220` |
| `PAYMENT_API` | runtime | `/api/v2/callback-status-current/withdraw` | `controller/ctl-update-status-withdraw/ctl-update-status-withdraw.main.go:41` |
| `URL_SCB_APP` | runtime | `/api/get-balance`, `/api/get-statement`, `/api/get-profile` (ban check) | `controller/ctl-notification-account/noti-ban-bank.go:86,116,118`; `controller/notification-bank/func.go:19,45-47` |
| `URL_ALL_API` | runtime | `GetSwitchAccountByBankCode` | `services/all-api/all-api.main.go:16,21,29` |
| `BOT_TELEGRAM_TOKEN` | runtime | telegram bot adapter (limit/whitelist noti) | `config/gbal.go:11`; `controller/ctl-notification-account/ctl-notification-account.main.go:267,332,407` |
| `BOT_TOKEN` + `GROUP_NOTI_BAN` | runtime → `https://api.telegram.org/bot{token}/sendMessage` | telegram ban noti | `noti-ban-bank.go:150-151,173` |
| (hardcode) | `https://kbank.thezeus.online/kbank_app/`, `https://scb-app.lupin.host/api/` 🟡 | `get-balance` (เช็ค balance KBANK/SCB) | `helper/bank-tranfers.go:13-14,42,65` |
| (hardcode) | `https://notify-api.line.me/api/notify` 🟡 | LINE notify | `controller/ctl-notification-account/ctl-notification-account.main.go:458`; `controller/notification-bank/func.go:85` |
| (hardcode) | `https://withdraw-monitor-new.firebaseio.com/`, `https://monitor.thezeus.online/api/netbank/withdraw`, `https://go-care-679ce-default-rtdb.firebaseio.com/Withdraw/` 🟡 ไม่มี auth | firebase realtime DB (อ่าน/เขียน balance) | `services/firebase/upWithdraw.go:38,75,146,162,282` |
| `configSystem.DatabaseConfig.OfficeAPI` (จาก DB) | runtime | ส่งเป็น field `office_api` ใน payload callback | `ctl-update-status-withdraw.main.go:48` |

### publish — RabbitMQ
URL จาก `dbConfig.RabbitMqURL` (MongoDB `database_config`)

| exchange/queue (exact) | payload | evidence |
|---|---|---|
| queue ชื่อ `{service}` (default exchange) | `_id.Hex()` ของ bank_statement (text/plain, persistent) — สร้างเครดิตฝาก | `controller/ctl-trigger-bank-account/ctl-trigger-bank-account.main.go:103-128` |
| queue `CHANGE_ACCOUNT_{BankNumber}` (default exchange) | `models.ReqData{Service,ID}` JSON — รายการโยกเงิน | `controller/ctl-change-account/ctl-change-account.main.go:278-337`, `188-189` |

### data — MongoDB (raw driver, no ORM)
office DB จาก env `DB_OFFICE`+`DB_ENDPOINT`/`DB_ENDPOINT_ALLINONE` (`db/conn.go:39-46`); multi-tenant: `CreateConnectionOnce` วน `database_config` สร้าง connection ต่อ service จาก `db_host_name` (`db/conn.go:115-166`); `connectTimeout=5s`, ctx 60s

| collection | R/W | evidence |
|---|---|---|
| `database_config` | R | `db/conn.go:118`; `repository/office.repository.go` (GetDatabaseConfigs) |
| `bank_config` / `bank_config_other` | R/W (อ่าน config, เปลี่ยน status/is_show/times_block) | `controller/ctl-change-account/...:44,48`; repo อ่านหลายจุด; `GroupByBankConfig` ใช้ scb_bussiness creds |
| `withdraw_statement` | R/W (SCB Business: อ่าน status 6/15, อัพ status 16/15/1/13/is_success + scb_bussiness) | `repository/isdb.repository.go:236-308` |
| `withdraw_statement_change_account` | W (insert รายการโยก) + R (เช็ครายการค้าง) | `ctl-change-account.main.go:225,267`; `repository/isdb.repository.go:103,111,128` |
| `bank_statement` | R (status 0 ย้อน 5 ชม. → recreate credit) | `ctl-trigger-bank-account.main.go:42,73` |
| `bank_statement_withdraw_whitelist` | R/W (สร้าง unique index hash, insertMany) | `ctl-notification-account.main.go:336,350-355` |
| `bank_deposit_summary` | R/W | `repository` GetBankDeposiSummarytByBanknumber / InsertBankDepositSummary |
| `bank_list` | R/W (status) | `repository` GetBankListByBankId / UpStatusBankListByBankId |
| `bank_account_switch` | R | `repository` GetBankAccountSwitchByServiceAndBankCode |
| `config_system` | R | `repository` GetConfigSystem |
| `working_log` | W | `ctl-notification-account.main.go:156`; `ctl-switch-account.main.go:110` |
| local file `balance_bank_config/balance_bank_all.json` | R/W (cache balance) | `ctl-change-account.main.go:381,397,491-528` |

### external — banks + credentials
| bank/api | purpose | auth (เก็บที่ไหน) | evidence |
|---|---|---|---|
| **SCB Business / SCB Anywhere** (`SCB_BUSINESS_API`) | สร้าง+ยืนยัน+ตรวจสอบการโอนถอนเงินจริง (maker/approver) | `bankConfig.ScbBussiness.{MakerUsername,MakerPassword,ApproverUsername,ApproverPassword}` + `PhoneNumber` ส่งเป็น **plaintext JSON** จาก `bank_config` (MongoDB) | `ctl-withdraw-transfer-scb-business/main.go:113-132`; model `models/bank-config.model.go:9-13` (`maker_password`/`approver_password` json:"-" แต่เก็บ plaintext ใน DB) |
| SCB app (`scb-app.lupin.host` / `URL_SCB_APP`) | เช็ค balance/profile/statement เพื่อตรวจบัญชีโดน ban | `bankConfig.AccessToken`/`PinBank`/`CitizenId`/`DateOfBirth` ส่งใน header/body | `helper/bank-tranfers.go:53-75`; `noti-ban-bank.go:86-118` |
| KBANK app (`kbank.thezeus.online`) | เช็ค balance | `AccessToken`/`PinBank` plaintext JSON | `helper/bank-tranfers.go:30-50` |
| Firebase RTDB | อ่าน balance withdraw จาก monitor | ไม่มี auth (URL เปิด) | `services/firebase/upWithdraw.go`; `ctl-change-account.main.go:346` |
| Telegram / LINE Notify | แจ้งเตือน limit/ban/whitelist | `BOT_TELEGRAM_TOKEN`/`BOT_TOKEN`/`GROUP_NOTI_BAN`, LINE token จาก `config_system.access_token_line` | `noti-ban-bank.go:150-153`; `ct l-notification-account.main.go:451-470` |

## observations
- **(1)** เงินจริง — SCB Business โอนแบบ create→confirm โดยไม่มี idempotency key/dedupe ที่ฝั่ง API call; กันซ้ำด้วยการ **pre-mark status ใน DB ก่อนเรียก API** (16 ก่อน create, 15 ก่อน confirm) เท่านั้น — `controller/ctl-withdraw-transfer-scb-business/main.go:233,246,276,289`
- **(2)** เงินจริง — ถ้า `transfer-confirm` สำเร็จที่ธนาคารแต่ DB/รอบถัดไปพลาด ไม่มี rollback/compensation; ทุก error path แค่ตั้ง status=12 และพิมพ์ log — `ctl-withdraw-transfer-scb-business/main.go:290-303`
- **(3)** `CallApiV2` (ตัวที่ยิง SCB Business transfer-create/confirm/check เงินจริง) ใช้ `&http.Client{}` **ไม่มี timeout** → ถ้า SCB ค้าง goroutine ค้างได้ไม่จำกัด — `helper/main.go:195-208`; ยังมี DISC-002/DISC-004 docs ยืนยันปัญหา timeout/delay
- **(4)** scheduler มี SingletonMode ทุก 3 นาที แต่ลำดับ create→`time.Sleep(1*time.Minute)`→verify ทำใน goroutine เดียว (`app.go:124-129`) — ถ้า create ของหลาย service/bank ช้าพร้อมกัน รอบจะลากเกิน 3 นาทีและกินทรัพยากร
- **(5)** SCB Business credentials (maker/approver password) เก็บ **plaintext** ใน `bank_config.maker_password`/`approver_password` (bson) แม้ json จะ `"-"` — `models/bank-config.model.go:11,13`; ถูก marshal ส่งออกทาง HTTP plaintext — `main.go:113-119`
- **(6)** Hardcoded production bank endpoints ใน source: `kbank.thezeus.online`, `scb-app.lupin.host` — `helper/bank-tranfers.go:13-14`
- **(7)** Swallowed error: `UpdateStatusCurrentWithdraw` เรียก `GetConfigSystem()` แล้วทิ้ง err (`err` ถูก shadow ทันทีบรรทัดถัดมา) — `ctl-update-status-withdraw.main.go:33-34`; `CreateQueueAddCredit` คืน void พิมพ์ error อย่างเดียว — `ctl-trigger-bank-account.main.go:89-133`
- **(8)** `ctl-change-account` re-queue `CHANGE_ACCOUNT_{BankNumber}` (โอนเงินจริง) ด้วย `scheduleFunc`/timer — กันรายการค้างด้วยเช็ค status 2/5/12 ก่อน insert (`ctl-change-account.main.go:238-244`) แต่ random amount และ hash layout เป็น guard เดียวกันซ้ำ → ถ้า hash ชนกันข้ามรอบเสี่ยงสร้างซ้ำ
- **(9)** `firebase.GetFirebaseByBankWithdraw` ใช้ balance จาก Firebase (ไม่มี auth, แก้ได้จากภายนอก) มาเป็นเงื่อนไขตัดสินใจโยกเงินจริง — `ctl-change-account.main.go:346,431`
- **(10)** `panic(err)` ใน startup path ถ้า `CreateConnectionOnce` ล้มเหลว → ทั้ง process ตาย (`app.go:32`); `ctl-switch-account.StartServ` ก็ `panic` (`ctl-switch-account.main.go:18`)
- **(11)** เขียน balance cache ลงไฟล์ใน container (`balance_bank_config/balance_bank_all.json`) — state ไม่ persist ข้าม pod restart, race ได้เพราะหลาย goroutine/ service เขียนไฟล์เดียวกัน — `ctl-change-account.main.go:397,491-507`
- **(12)** deps เก่า: `streadway/amqp v1.0.0` (deprecated), mongo-driver v1.11.2, go 1.18 (EOL) — `go.mod:3,8,23`
- **(13)** มีโค้ดซ้ำซ้อน 2 ชุดสำหรับ ban check (`controller/notification-bank/func.go` vs `ctl-notification-account/noti-ban-bank.go`) ตรรกะต่างกันเล็กน้อย (func.go เช็ค statement ด้วย, noti-ban ไม่เช็ค) — เสี่ยง drift
