# Card: go-agent-rocketwin
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go 1.24, Gin (HTTP), MongoDB (raw driver), Redis (go-redis v8), RabbitMQ (streadway/amqp), ClickHouse (sqlx), OpenTelemetry/Prometheus — `go.mod:1-30`
- **module**: `agentnewtopup` — `go.mod:1`
- **entrypoint**: `_cmd/main.go:14` `func main()` — แยกโหมดตาม `SERVICE_METHODS`: `"queue"`→`rabbitmqlotto.ConnectRabbit(resource)` (consumer), `"api"`/default→`agentnewtopup.StartServer()` (HTTP) — `_cmd/main.go:35-43`
- **บทบาท**: เป็น "เกมเอเย่นต์" 2 ฝั่งในตัวเดียว — (1) seamless wallet สำหรับเกม rocketwin (controller/rocketwin) (2) seamless lotto agent สำหรับ kinglot (controller/kinglot + controller/v2/kinglot) ที่ตัด/เพิ่มเครดิตจริงผ่าน rocketwin wallet API
- **port**: ฟัง `:8000` ฮาร์ดโค้ด — `server.go:102` `Addr: fmt.Sprintf(":8000")`; docker map `8167:8000` — `docker-compose.yml:8`
- **Dockerfile**: golang:1.24-alpine3.22 → alpine:3.22, `CMD ["./app"]`, ไม่มี EXPOSE — `Dockerfile:1-11`
- **CI**: `.github/workflows/pipeline.yml` reusable workflow, `image_name: agent-newtopup` — `pipeline.yml:1,18`; `build.yaml.disabled` ปิดอยู่

## provides

### http — gin (`route.go`)
กลุ่ม `/agent` (rocketwin — game seamless wallet): | METHOD | path | auth | evidence |
- GET `/agent/` — none(CORS เท่านั้น) — `route.go:24`
- GET `/agent/userinfo` (GetBalance) — `route.go:27`
- GET `/agent/deposit` (CreateTransaction "up") — `route.go:28`
- GET `/agent/withdraw` (CreateTransaction "down") — `route.go:29`
- GET `/agent/add_member` — `route.go:30`
- POST `/agent/del_member` — `route.go:31`
- POST `/agent/usergamedelete` — `route.go:32`
- POST `/agent/usersessiondelete` — `route.go:33`
- POST `/agent/updatetypenotplayuser` — `route.go:34`
- GET `/agent/brandgame|getgame|playgame|searchgame|listgame` — `route.go:37-41`
- POST `/agent/getdetail` — `route.go:42`
- GET `/agent/wallet|transactionType|createToken|transfer/transaction` — `route.go:44-47`
- GET `/agent/outstanding/:username` ; POST `/agent/outstanding` — `route.go:51-52`
- GET `/agent/getratelimit|getbuffergame|checkstatus|checkbalanceagent` — `route.go:57-60`
- POST `/agent/userbalanceminimul` (UserBalanceMimimal callback) — `route.go:63`
- POST `/agent/transactionsbo` (CreateTransactionSbo callback) — `route.go:64`
- GET/POST `/agent/reporttransactionsport` — `route.go:67-68`
- GET `/agent/getdomainagent|updatedomainagent|uploadfilegame|preset-pg` — `route.go:70-73`

กลุ่ม `/agent/seamless` (รายงาน transaction): GET `/transaction`,`/sumtransaction`,`/useractivetransaction`,`/transactiongroupbytype`,`/sumtransactioncommission` — `route.go:79-85`

กลุ่ม `/api`: POST `/api/userbalanceminimul`, POST `/api/transactionsbo` — `route.go:90-91`

กลุ่ม `/agent` (kinglot lotto): | METHOD | path | auth | evidence |
- GET `/agent/login` (kinglot.Login) — `route.go:101`
- GET `/agent/games|rounds|yiki|group|history|reward` ; POST `/agent/yiki|prebet|refund` — `route.go:104-113`
- GET `/agent/seamless/transactionlottogroupbytype|transactionlottoround|reward_bygroup|report_winloss_summary` ; POST `/agent/check_bet_set` — `route.go:119-123`
- **callback balance**: GET `/agent/balance` (kinglot.GetBalance) — `route.go:129`
- POST `/agent/updatedomainagentlotto` — `route.go:132`
- POST `/agent/access_lotto_seamless` (seamless.AccessSeamless) — `route.go:138`

**callback v2 lotto (seamless wallet — เคลื่อนเงินจริง)** — auth: ไม่มี signature/auth ที่ handler มีแต่ CORS IP allowlist:
- POST `/agent/bet` (kinglotv2_bet.GetBet) — `route.go:142`
- POST `/agent/placebet` (kinglotv2_bet.PlaceBet) — `route.go:143`
- POST `/agent/settlebet` (kinglotv2_settlebet.SettleBet) — `route.go:144`
- POST `/agent/voidbet` (kinglotv2_voidbet.VoidBetLotto) — `route.go:145`
- POST `/agent/unsettle` (kinglotv2_unsettle.UnSettleLotto) — `route.go:146`
- POST `/agent/settlebetslow` (kinglotv2_settlebet.SettleBetSlow) — `route.go:149`
- POST `/agent/callbacklotto` (kinglotv2_callbacklotto.CallbackLotto) — `route.go:152`

กลุ่ม `/v2/agent` (ทำซ้ำชุด callback v2): GET `/v2/agent/balance` ; POST `/v2/agent/bet|placebet|settlebet|voidbet|unsettle|settlebetslow|callbacklotto` — `route.go:159-170`

### consume — RabbitMQ (เฉพาะโหมด `SERVICE_METHODS=queue`)
| queue (EXACT) | รูปแบบ | handler | evidence |
- `CALLBACK_LOTTO_<SERVICE>` (เช่น `CALLBACK_LOTTO_ROCKETBET`) — RPC consumer, manual ack, prefetch=1, `auto-ack=false`, durable=false — `service/rabbitmqlotto/worker.go:55-90`
- dispatch ตาม `d.Type` (AMQP Type header): `PLACEBET`→placeBetQueue, `SETTLEBET`→settleBetQueue, `UNSETTLE`→unSettleBetQueue, `VOIDBET`→voidBetQueue, default→error — `worker.go:106-142`
- ตอบกลับ publish ไป `d.ReplyTo` พร้อม `CorrelationId`, Expiration 60000ms — `worker.go:149-159`
- placeBetQueue ตัดเครดิต (WITHDRAW/PLACEBET) — `worker.go:171-208`; settleBetQueue เพิ่ม (DEPOSIT/SETTLEBET) — `worker.go:210-245`; unSettleBetQueue (WITHDRAW) — `worker.go:247-317`; voidBetQueue (DEPOSIT) — `worker.go:319-352`

### cron — ไม่ใช่ cron แต่เป็น background worker (เริ่มใน StartServer เมื่อ `LOTTOKEY != ""`)
| worker | ทำอะไร | evidence |
- `ScanQueueSettleBetClickhouse` — สแกน redis key `queue_settle_bet_clickhouse_*` batch 500 → insert ClickHouse `payload_lotto` — `worker.go:37-77`, `settlebet/worker.go:32,191`
- `ScanQueueSettleBetFast` — สแกน `queue_settle_bet_fast_work_*` → settleBetFastwork (เพิ่มเครดิตชนะ) — `worker.go:79-117`, `settlebet/worker.go:33,272`
- `ScanQueueSettleBetSlow` — pop `queu_settle_bet_slow_work` (สังเกตสะกดผิด) reconcile — `worker.go:278-367` ; **NB**: `SettleBetSlowwork` ถูก comment ปิดใน slow scan — `settlebet/worker.go:363`
- เริ่มที่ `server.go:71-73`

## consumes

### http-out
| env key | base URL | paths | evidence |
- `APIKEY`+`TYPE` → rocketwin wallet API (`https://api.rocketwin.net/api` / mahabackend / rocketwindemo) — `service/rocketwin_service/init.go:9-31`
  - POST `v1/wallet/gen_token_statement` (CreateToken) — `rocketwin_service/createToken.go:24`
  - POST `v1/wallet/transaction` (CreateTransaction) — `rocketwin_service/createTransaction.go:28`
  - GET `v1/wallet/balance` (GetWallet) — `rocketwin_service/getwallet.go:34`
  - GET `/v1/wallet/balance`, POST `/v1/wallet/transaction`, `/v2/external/wallet/balance`, `/v1/wallet/gen_token_statement` (legacy queue path MonacoRequestAPI) — `service/rabbitmqlotto/callrocketwin.go:56,111,134,154`
- `TYPELOTTO` → kinglot lotto API (`https://api.u4win.com` / `api.lotto-commission.net` / `api-227cdacd.nip.io` / `api-34.143.236.165.sslip.io`) — `service/kinglot_service/init.go:9-27`
  - POST `api/games/games` (BetGame) — `kinglot_service/bet.go:22`
  - GET `api/external/game/recepsiton/secret-key` (GetSecretKeyLotto) — `kinglot_service/loginlotto.go:21`
  - GET `api/external/game/get-token` (GetTokenLotto) — `kinglot_service/loginlotto.go:51`
  - GET `api/games/rounds` (GetRoundCode) — `kinglot_service/roundcode.go:20`
  - POST `/api/manage/game/domain` (InitConfig/UpdateDomainLotto) — `controller/kinglot/exernal.go:38,71`
- `APIOFFICE` → ใช้เป็น domain ลงทะเบียน callback กับ lotto + verify ผ่าน `checkDomain` GET `<domain>/agent/` — `controller/kinglot/exernal.go:56-63,90-112`
- `APICALLBACKLOTTO` → POST `<APICALLBACKLOTTO>/v1/callback_lotto` (ส่ง round_id ไป aff service หลัง callbacklotto) — `controller/v2/kinglot/callbacklotto/callbacklotto.go:65,86`
- `APICALLBACK` → POST `<APICALLBACK>api/v1/callback_lotto` (legacy, ในโค้ด comment block) — `controller/rocketwin/transaction.go:718,724`
- `APICUSTOMER` → POST `/v1/external-setting/update` (update domain customer) — `controller/rocketwin/external.go:51,67`; HttpHandle prepends `construct.APIURL` — `controller/rocketwin/callapi.go:253`
- `APIMINIGAME` → POST `<APIMINIGAME>/api/get_summary_stat` (turnover minigame) — `controller/rocketwin/transaction.go:1629,1644`
- `SEAMLESS_API` → POST `<SEAMLESS_API>/get_url` (ขอ url เข้าเกม lotto, ส่ง `secret_key=LOTTOKEY`) — `controller/seamless/access_seamless.go:62,74`
- `FIREBASEURL` → PUT/GET Firebase realtime (`/notify/creditagent/<service>.json`) — `controller/rocketwin/wallet.go:276-284`; queue noti firebase ถูก comment — `rabbitmqlotto/func.go:267-275`
- LINE Notify (hardcoded token) → POST `https://notify-api.line.me/api/notify` — `service/rocketwinService.go:271,286`; alert credit agent — `rabbitmqlotto/callrocketwin.go:199`

### publish — ไม่พบ (repo นี้เป็น RPC consumer ฝั่ง reply เท่านั้น; publish reply ไป `d.ReplyTo` — `worker.go:149`)

### data — MongoDB (DB จาก `MONGODB_DB_NAME`, raw driver, timeout 60s)
| collection | R/W | evidence |
- `wallet_statement_lotto` — R/W หลัก (สร้าง/อัปเดต statement bet/settle/void/unsettle) — `bet/bet_util.go:108`, `settlebet/settlbet_utils.go:68,151`, `voidbet/voidbet_utils.go:36`, `worker.go:273,297`
- `wallet_statement` — R (legacy GetWallet balance) — `rabbitmqlotto/func.go:39`, `service/rocketwinService.go`
- `callback_min_amount` — R/W (UserBalanceMimimal) — `rabbitmqlotto/func.go:355,384,408`
- `service_log` — W/Del (lock กันยิงซ้ำต่อ user/CREDIT) — `rabbitmqlotto/func.go:496,506`
- `lotto_callback` — W (ผลรางวัล) — `controller/kinglot/callback.go:202`, `callbacklotto/callbacklotto_utils.go:127`
- `sbo_callback`, `config_system`, `member_log`, `log_alertreportlotto` — `repo/repo.go` + grep
- **ClickHouse** (`CLICKHOUSE_*`, db `lotto_dbF`): `payload_lotto` (insert batch), `callback_lotto` (insert) — `repo/repo_clickhouse.go:95,201`, `settlebet/worker.go:191`, `callbacklotto/callbacklotto_utils.go:50`

### external — game/lottery providers
| provider | purpose | auth | evidence |
- rocketwin wallet (`api.rocketwin.net`) — ตัด/เพิ่มเครดิตผู้เล่นจริง | header `apiKey: <APIKEY>` | `rocketwin_service/getwallet.go:38`, `createTransaction.go:30`
- kinglot lotto API (u4win/lotto-commission/nip.io) — แทงหวย/ดึงรอบ/token | header `Authorization: Bearer <key>` + query `secret_key=<LOTTOKEY>` | `kinglot_service/bet.go:25`, `loginlotto.go:17`
- seamless url provider (`SEAMLESS_API`) | body `secret_key=LOTTOKEY` | `access_seamless.go:69`
- Telegram bot alert (token ฮาร์ดโค้ด) | `service/telegramService.go:11,38`
- LINE Notify (token ฮาร์ดโค้ด) | `service/rocketwinService.go:284`

## observations
1. **ไม่มี signature/auth verification บน callback ที่เคลื่อนเงิน** — `/agent/placebet|settlebet|voidbet|unsettle` ป้องกันด้วย CORS IP allowlist (`CF-Connecting-IP`) เท่านั้น และ dev/uat เปิด `*` — `server.go:210-231`, `route.go:142-152`. ใครก็ตามที่ปลอม IP/อยู่ใน allowlist ยิง settlebet ได้.
2. **PlaceBet มี Redis lock แต่ settle/void/unsettle ไม่มี** — bet ล็อก `<username>_bet` 25s — `bet/bet.go:80-105`; แต่ `settleBetFastwork`/voidbet/unsettle ไม่ล็อก → settle ซ้ำพร้อมกันบน txid เดียวเป็นไปได้.
3. **Settle idempotency อาศัย hash + อ่านก่อนเขียน (race)** — กันซ้ำด้วย `hash=<txid>SETTLEBET<recaltimes>` แล้ว `checkHash` + เช็ค `status==1` ก่อนสร้าง — `settlebet/settlbet_utils.go:28`, `settlebet/settlebet.go:51-65`. เป็น check-then-act ไม่ atomic (ไม่มี unique index/insert-if-not-exists) → settle ถูกซ้ำได้ถ้ามาพร้อมกัน → จ่าย payout ซ้ำ.
4. **Settle ตัดเงิน rocketwin ก่อน แล้วค่อยอัปเดต DB แบบ async** — `PlaceBet`: CreateTransaction (ตัดเงิน) สำเร็จแล้ว update wallet_statement ใน `go func()` — `bet/bet.go:360-423`. ถ้า goroutine ตาย/DB ล้ม เงินถูกตัดแต่ statement ไม่อัปเดต → balance/ledger ไม่ตรง.
5. **settleBetFastwork: ตีม non-VIOLET ไม่เรียก wallet API จริง** — แค่ `GetWallet` แล้วคำนวณ `balance + payout` เขียน statement โดย**ไม่**สร้าง transaction ที่ rocketwin — `settlebet/settlbet_utils.go:192-214`, `settlebet/settlebet.go:174-215`. หมายความว่าเครดิตที่จ่ายจริงขึ้นกับว่า rocketwin ฝั่งไหนเป็นเจ้าของยอด (ขึ้นกับ THEMEMODE) — เสี่ยงจ่ายไม่ตรงระหว่างโหมด.
6. **legacy queue placeBet ไม่เช็ค balance ใน DEPOSIT path และ unsettle อนุญาต Payout<=0 ผ่านได้** — `CreateStatementWallet` คืน `true,nil` เมื่อ SETTLEBET/UNSETTLE payout<=0 โดยไม่เขียน statement — `rabbitmqlotto/func.go:118-124`.
7. **Hardcoded secrets** — Telegram bot tokens `8013495163:AAE...`/`8004912188:AAE...` — `service/telegramService.go:11,36`; LINE Notify tokens `EO8qk...`, `pKahNd...` — `rabbitmqlotto/callrocketwin.go:195`, `service/rocketwinService.go:284`. และ `.env` commit จริง: `MONGODB_ENDPOINT` พร้อม user/pass, `CLICKHOUSE_PASSWORD=L_~30ld4026VT`, `APIKEY=20zFNEiq...` — `.env:6,40,2`.
8. **HTTP client หลายตัวไม่มี timeout** — `HttpHandle` POST สร้าง `&http.Client{}` ไม่มี timeout — `service/rocketwinService.go:189`; `checkDomain` client มี Transport แต่ไม่มี client.Timeout — `controller/kinglot/exernal.go:84-90`; `ExternalCall` ตั้ง timeout เฉพาะเมื่อ option ส่งมา — `service/http_service/http_service.go:48`. (CreateToken/GetWallet ตั้ง timeout 30s — `enum/rocketwin.go:13-14`).
9. **HttpHandle POST: เช็ค error หลัง client.Do ถูก comment (panic ถูกปิด) → deref resp ที่อาจ nil** — `service/rocketwinService.go:189-196` `if err != nil { fmt.Println; // panic(err) }` แล้ว `defer resp.Body.Close()` ทันที → nil pointer panic ได้เมื่อ network error.
10. **Swallowed errors เพียบ** — `GetFirebase`/`PutFirebase` print error เฉยๆ — `rabbitmqlotto/func.go:457,470`; CallAPI legacy ละเลย unmarshal error; settle clickhouse push อยู่ใน `go func()` log แล้วทิ้ง — `settlebet/settlebet.go:489-498`.
11. **Redis lock unlock ไม่ตรวจ owner ในจุดเรียก** — bet ใช้ `AcquireLock`/`unlock()` (มี UnlockFunc) — `bet/bet.go:80,99`; แต่ lock duration 25s < เวลาตัดเงิน rocketwin ที่ retry token 3 ครั้ง×2s + transaction → lock อาจหมดอายุก่อนทำเสร็จ → ยิงซ้ำได้.
12. **`MONGODB` pool size 1** (Min=Max=1) ใน production — `_config/db.go:62-65` → คอขวด/contention ภายใต้โหลด callback.
13. **CORS allowlist เป็น IP ฮาร์ดโค้ดยาว (~40 IP)** — เปลี่ยน infra ต้องแก้โค้ด — `server.go:147-198`.
14. **`/v2/agent/*` กับ `/agent/*` ผูก handler ชุดเดียวกันซ้ำ 2 path** — เพิ่ม attack surface/ความสับสน — `route.go:142-170`.
15. **deps เก่า/ผสม**: ใช้ทั้ง `streadway/amqp` (archived), `monaco-io/request`, `vicanso/go-axios`, mongo-driver v1.15 — `go.mod`.
