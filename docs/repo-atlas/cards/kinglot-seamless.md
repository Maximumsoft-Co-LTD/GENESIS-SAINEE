# Card: kinglot-seamless
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go 1.22, **Iris v12** (HTTP — ต่างจากเพื่อนที่ใช้ Gin), MongoDB (raw driver), RabbitMQ (streadway/amqp), OpenTelemetry/Prometheus; ไม่มี Redis, ไม่มี ClickHouse — `go.mod:1-24`, `server.go:14-17`
- **module**: `kinglot` — `go.mod:1`
- **entrypoint**: `_cmd/main.go:14` `func main()` — เลือกโหมดตาม `SERVICE_METHODS`: `"queue"`→`rabbitmqlotto.ConnectRabbit(database)` (consumer/worker ตัดเครดิต), `"api"`→`kinglot.StartServer()` (HTTP + RPC publisher) — `_cmd/main.go:24-31`
- **บทบาท**: seamless wallet สำหรับ lotto/เกม kinglot — รับ callback จาก lotto provider (placebet/settle/void/unsettle) แล้วเคลื่อนเครดิตใน wallet ของตัวเอง (collection `wallet_statement`/`agent_credit`/`member_account`), เป็นทั้ง agent proxy ไป lotto API (`api.u4win.com` ฯลฯ) และ wallet เจ้าของยอด
- **port**: ฟัง `:8000` ฮาร์ดโค้ด — `server.go:135` `app.Listen(":8000")`; docker map `8168:8000` — `docker-compose.yml:7-8`
- **Dockerfile**: golang:1.22.3-alpine3.20 → alpine:3.14 (ผ่าน harbor proxy `habor-proxy.analytichpxv3.online`), `CMD ["./app"]`, ไม่มี EXPOSE — `Dockerfile:1-9`
- **CI**: `.github/workflows/pipeline.yml` reusable, `image_name: kinglot-seamless` — `pipeline.yml:1,18`; `build.yaml.disabled` ปิด

## provides

### http — Iris (`server.go`)
มี middleware `method.New().ServeHTTP` (prometheus/log) + `CORS` (IP allowlist) — `server.go:23-34`

กลุ่ม `/api/v1` (มี `method.Authentication` = ตรวจ header `Authorization: Bearer <AUTHEN>`, ถ้า `AUTHEN==""` ข้าม) — `server.go:62-72`, `method/middleware.go:9-25`:
| METHOD | path | handler | evidence |
- GET/POST `/api/v1/` (GetAPI) — `server.go:65-66`
- POST `/api/v1/login` (callback.GetValidateToken) — `server.go:67`
- GET `/api/v1/balance` (callback.GetBalance — **callback balance**) — `server.go:68`
- POST `/api/v1/placebet` (callback.PlaceBet — ตัดเครดิต WITHDRAW) — `server.go:69`
- POST `/api/v1/voidbet` (callback.VoidBet) — `server.go:70`
- POST `/api/v1/settlebet` (callback.SettleBet — เพิ่มเครดิต DEPOSIT) — `server.go:71`
- POST `/api/v1/unsettle` (callback.UnSettleBet) — `server.go:72`

กลุ่ม `/agent` (**ไม่มี Authentication middleware** — CORS IP เท่านั้น) — `server.go:75`:
- GET `/agent/userinfo|deposit|withdraw` (wallet) — `server.go:82-84`
- GET/POST `/agent/` (GetAPI) — `server.go:86-87`
- POST `/agent/login` (callback.GetValidateToken) — `server.go:88`
- GET `/agent/balance` (callback.GetBalance — **callback**) — `server.go:89`
- GET `/agent/add_member` (callback.Addmember) — `server.go:95`
- GET `/agent/login` (game.Login) — `server.go:96` *(ซ้ำ path กับ POST /agent/login)*
- GET `/agent/games|rounds|history|reward|yiki|gamesbyname|group` ; POST `/agent/prebet|bet|yiki|refund` (game.*) — `server.go:97-119`
- POST `/agent/callbacklotto` (callback.LottoRoundResult — **callback ผลรางวัล**) — `server.go:104`
- GET `/agent/seamless/transactionlotto|sumtransaction|useractivetransaction|transactionlottogroupbytype|transactionlottoround|reward_bygroup|sumtransactioncommission` — `server.go:105-122`
- POST `/agent/check_bet_set` ; GET `/agent/report_winloss_summary` — `server.go:115-116`
- POST `/agent/userbalanceminimul` (callback.GetUserBalanceMimimal) — `server.go:117`
- POST `/agent/updatedomainagentlotto` (callback.UpdateDomainLotto) — `server.go:119`
- POST `/agent/alert_ratepayment` — `server.go:121`
- POST `/agent/noti_reward` (callback.NotiRewardCallback) — `server.go:130`

**callback v2 ผ่าน RabbitMQ RPC (publisher)** — handler `method.ReciveRequest(key)` แปลง HTTP→queue แล้วรอ reply (sync RPC):
- POST `/agent/placebet` → key `"PLACEBET"` — `server.go:125`
- POST `/agent/settlebet` → key `"SETTLEBET"` — `server.go:126`
- POST `/agent/unsettle` → key `"UNSETTLE"` — `server.go:127`
- POST `/agent/voidbet` → key `"VOIDBET"` — `server.go:128`
- (สังเกต: `/agent/placebet` ฯลฯ ทั้ง 4 ถูก map ทับด้วย ReciveRequest หลังบรรทัด 90-93 ที่ comment ปิดตัว direct handler ไว้ — `server.go:90-93`)

### consume — RabbitMQ (โหมด `SERVICE_METHODS=queue` → `rabbitmqlotto.WorkerLotto`)
| queue (EXACT) | รูปแบบ | handler | evidence |
- `CALLBACK_LOTTO_<SERVICE>` (เช่น `CALLBACK_LOTTO_KINGLOT`) — RPC consumer, manual ack, prefetch=1, durable=false — `rabbitmqlotto/worker.go:33-66`
- dispatch ตาม `d.Type`: `PLACEBET`→`callback.PlaceBetQueue` (ตัดเครดิต) , `SETTLEBET`→`SettleBetQueue` (เพิ่ม), `UNSETTLE`→`UnSettleBetQueue`, `VOIDBET`→`VoidBetQueue`, default→error — `worker.go:95-178`
- reply publish ไป `d.ReplyTo` + `CorrelationId`, Expiration 60000ms — `worker.go:182-192`

### cron — ไม่พบ

## consumes

### http-out
| env key | base URL | paths | evidence |
- `TYPE`+`APIKEY` → kinglot lotto API: `https://api.u4win.com` / `api.lotto-commission.net` / `api-227cdacd.nip.io` / `api.228feca5.nip.io` / `api.lotto-thorv2.com` — `callback/init.go:20-46`
  - POST `/api/v1/lobby/get/token` (GetValidateToken login) — `callback/login.go:48`
  - GET `/api/external/game/recepsiton/secret-key`, GET `/api/external/game/get-token` (game.Login) — `game/login.go:35,60`
  - POST `/api/games/games` (GetBet/PreBet, game.GetGame/GetGameByName/yiki) — `game/bet.go:105,259`, `game/game.go:45,83,169,255`
  - POST `/api/games/refund` (game.Refund) — `game/bet.go:301`
  - POST `/api/manage/game/domain` (InitConfig/UpdateDomainLotto, ส่ง `secret_key=APIKEY`) — `callback/bet.go:1781,1817`
- `APIOFFICE` → domain ลงทะเบียน + verify GET `<domain>/agent/` (checkDomain) — `callback/bet.go:1803-1808,1842`
- `APICALLBACK` → POST `<APICALLBACK>/v1/callback_lotto` (ส่ง round_id หลัง LottoRoundResult) — `callback/bet.go:922,955`
- `APIMINIGAME` → POST `<APIMINIGAME>/api/get_summary_stat` (turnover) — `callback/bet.go:1576`
- `FIREBASEURL` → PUT/GET Firebase (`/notify/ratelotto/<roundcode>.json`) — `method/service.go:262-269`, `callback/linenoti.go`
- LINE Notify / Telegram (alert) — `callback/linenoti.go`

### publish — RabbitMQ (โหมด api/RPC publisher, `method.PublishQueue`)
| queue (EXACT) | payload | evidence |
- publish ไป routing key `CALLBACK_LOTTO_<SERVICE>` (default exchange ""), AMQP `Type=<PLACEBET|SETTLEBET|UNSETTLE|VOIDBET>`, `ReplyTo=<temp queue>`, `CorrelationId=<UnixNano>`, body `model.BetPayload`, Expiration 60000ms — `method/service.go:130-148`
- สร้าง reply queue ชั่วคราว (durable=false, autoDelete=true, exclusive=true) แล้ว consume รอ correlation id ตรง — `method/service.go:67-82,95-105,170-176`
- รูปแบบ: เป็น RPC sync — HTTP request บล็อกจนได้ reply หรือ channel ปิด

### data — MongoDB (DB จาก `MONGODB_DB_NAME`, raw driver, timeout 60s, pool 1-50)
| collection | R/W | evidence |
- `wallet_statement` — R (GetWallet: filter `type_name=AGENT, is_check=0` sort datetime desc) / W (CreateTransactionWallet insert + UpdateWalletIsCheck) — `callback/db.go:38,55`, `db.go:238`
- `agent_credit` — R/W (ยอดเครดิตเอเย่นต์รวม: GetAgentCredit/CreateAgentCredit/UpdateAgentCredit) — `callback/db.go:74,86,98`
- `member_account` — R/W (GetMember / UpdateMemberCredit balance) — `callback/db.go:110,137`
- `lotto_callback` — W (InsertMany ผลรางวัล, unordered) — `callback/db.go:152`
- `service_log` — W/Del (lock per username/CREDIT) — `callback/db.go:342,354`
- `callback_min_amount` (UserMimimal) — `callback/bet.go:969+`, `db.go:276,320`

### external — game/lottery providers
| provider | purpose | auth | evidence |
- kinglot lotto API (u4win/lotto-commission/nip.io/lotto-thorv2) — login/bet/round/refund/domain | header `Authorization: Bearer <key>` + body/query `secret_key=APIKEY`; เพิ่ม `Referer: lot` เมื่อ TYPE=LOTTO-STAGING | `game/bet.go:105`, `callback/login.go:48`, `method/callapi.go:41-43`
- Firebase realtime DB (`FIREBASEURL`) | none | `method/service.go:262`
- Telegram/LINE alert | token (ดู linenoti.go) | `callback/linenoti.go`

## observations
1. **`/agent/placebet|settlebet|voidbet|unsettle` ไม่มี auth** — กลุ่ม `/agent` **ไม่** ติด `method.Authentication` (ต่างจาก `/api/v1`) — `server.go:75` vs `server.go:63`. callback ที่เคลื่อนเงินกันด้วย CORS IP allowlist เท่านั้น และ dev/uat เปิด `*` — `server.go:147,193-201`.
2. **ไม่มี idempotency จริงบน settle/void/unsettle** — `CreateStatementWallet` คำนวณ `BeforeCredit=result.AfterCredit`, `AfterCredit=±point` แล้ว insert statement ใหม่ทุกครั้ง โดย**ไม่เช็ค hash/txid ซ้ำ**ก่อน — `callback/bet.go:766-814`. `Hash=<txid><typeDesp><recaltimes>` ถูกเขียนแต่ไม่ถูกใช้กันซ้ำ → **ส่ง settlebet ซ้ำ = จ่าย payout ซ้ำได้** (replay ได้เต็มๆ).
3. **Read-modify-write บน agent_credit/member_account ไม่ atomic (race)** — `CreateCreditAgent` อ่าน `agent_credit[0].Balance` แล้ว `credit = balance ± amount` แล้ว `UpdateAgentCredit($set balance)` — `callback/wallet.go:371,396-401`. ทุก bet/settle พร้อมกันบนยูสเดียว/เอเย่นต์เดียว → lost update, ยอดเครดิตเพี้ยน. ไม่มี `$inc`, ไม่มี lock, ไม่มี transaction.
4. **GetWallet อิงรายการล่าสุด (datetime desc) เป็น balance** — `callback/db.go:34-42`. ถ้า 2 request เข้าพร้อมกันอ่าน balance เดียวกันก่อนเขียน → ตัดเงินเกิน/balance ติดลบได้ (ไม่มี lock เลยทั้ง repo — ไม่มี Redis).
5. **ไม่มี Redis lock ใดๆ ทั้ง repo** — กันซ้ำใช้แค่ `service_log` collection (CreateServiceLog + defer DelServiceLog) ซึ่งเป็น log ไม่ใช่ mutex จริง — `callback/bet.go:692-713`, `db.go:342`.
6. **PlaceBet เช็ค balance ก่อน แต่ DEPOSIT (settle/void) ไม่เช็คอะไร** — settle/void สร้างเครดิตได้เสมอเมื่อ payout>0 — `callback/bet.go:526,663`.
7. **Settle: ถ้า `Payout<=0` ตอบ success โดยไม่เขียน statement** — `callback/bet.go:131,448-503`, `worker.go` settle path. ปกติ แต่หมายถึง provider เป็นผู้กำหนด payout ทั้งหมด ไม่มีการตรวจสอบฝั่งเรา.
8. **Hardcoded fallback secret** — ถ้า `APIKEY==""` ใช้ secret ฮาร์ดโค้ด `"RnMptogBXKYmi6eWt9737M00lgKI9TDR"` ยิง login ไป lotto — `callback/login.go:44-45`. และ `.env` commit จริง: `MONGODB_ENDPOINT=mongodb+srv://root:Zxcvasdf789@...`, `APIKEY=XjrZIp2Ez...`, `ACCESSTOKEN_SECRET_*` หลายตัว — `.env:3,7,20-23`.
9. **HTTP client outbound ไม่มี timeout** — `method.HttpHandle` POST สร้าง `&http.Client{}` ไม่มี timeout — `method/callapi.go` (HttpHandle); `checkDomain` มี Transport แต่ client ไม่มี Timeout — `callback/bet.go:1826-1831`; `CallAPI` ใช้ monaco-io/request ไม่ตั้ง timeout — `method/callapi.go:46-54`. provider ค้าง = request ค้างยาว.
10. **RPC publisher อาจค้างถาวร** — `PublishQueue` `for d := range msgs` รอจน correlation id ตรง โดยไม่มี timeout/context; ถ้า consumer ไม่ตอบ HTTP request ค้าง — `method/service.go:170-176`. reply queue เป็น exclusive/autoDelete ต่อ request.
11. **Swallowed errors** — `GetFirebase`/`PutFirebase` print แล้วไปต่อ — `method/service.go:240,253`; `json.Unmarshal(d.Body...)` ใน worker error → `return` ออกจาก goroutine ทั้งก้อน (หยุด consume) — `worker.go:74-77`; `UpdateMemberCredit` error แค่ print — `callback/wallet.go:407`.
12. **worker.go: error ระหว่าง process → `return` ทำให้ consumer ตาย** — Unmarshal/Marshal/Publish fail ทำ `return` ในตัว goroutine consume → หยุดรับ queue ทั้งหมดจนกว่าจะ reconnect — `worker.go:74-77,170-176,189-192`.
13. **same codebase 2 deployment (publisher vs consumer) แชร์ queue name เดียว** — ถ้า env `SERVICE` ไม่ตรงกันระหว่าง publisher/consumer reply จะไม่ถึง (queue `CALLBACK_LOTTO_<SERVICE>` ผูกกับ `os.Getenv("SERVICE")` ทั้งสองฝั่ง) — `method/service.go:131`, `worker.go:33`.
14. **deps เก่ามาก** — Iris v12.1.8 (2020), `streadway/amqp` v1.0.0 (archived), mongo-driver v1.8.1, monaco-io/request v1.0.15 — `go.mod`. Iris ไม่ตรงกับ stack Gin ของเพื่อนร่วมระบบ.
15. **CORS IP allowlist ฮาร์ดโค้ด ~40 IP** เหมือน go-agent-rocketwin แต่ของ kinglot **abort เป็น StatusBadGateway** เมื่อไม่อยู่ใน allowlist และไม่มี bypass path — `server.go:202-206`.
