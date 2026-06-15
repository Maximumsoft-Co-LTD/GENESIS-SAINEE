# Card: 3rd-gateway
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go + Gin (HTTP API), MongoDB (go-mongox ORM), Redis (go-redis v8, ใช้ใน service/thorlock แต่ยังไม่ wire), Tink (AEAD/Hybrid crypto), fasthttp (เรียก TrueMoney), Zap/otelzap logger
- **module**: `app` (go.mod:1, go 1.23.0 — go.mod:3)
- **entrypoint**: `_cmd/main.go` → `config.InitEnvConfig()` (init) → `app.StartServer()` (main.go:9,:13)
- **port**: **`:8000`** hardcoded (`http.Server{Addr: ":8000"}` — app.go:65) — ไม่ได้อ่านจาก env
- **config loader**: viper, อ่าน `.env` เมื่อ `MODE!=production` ไม่งั้น AutomaticEnv (config/environment.go:33-52)
- **Dockerfile**: golang:1.23.3-alpine3.20 → alpine:3.20, `CMD ["./app"]`, set TZ Asia/Bangkok (Dockerfile:1-11) — ไม่ EXPOSE port
- **CI**: `.github/workflows/build.yaml` NAME=3rd-gateway, self-hosted → Kaniko → JFrog → trigger K8S manifest repo (build.yaml:4,:52-58)
- **OTel ถูกปิด** — InitProvider/RequestLogger ถูก comment ทิ้ง (app.go:43-48, :54)

## provides
### http (prefix `api`, gin — routes/main.go:26-39)
| METHOD | path | auth middleware | evidence (file:line) |
|---|---|---|---|
| POST | `api/register` | **ไม่มี** (public) | routes/main.go:31 → controller/customer.go:25 |
| GET | `api/get-list-gateway` | `AuthClientMiddleware` | routes/main.go:33 → gateway.go:37 |
| POST | `api/get-connected-by-account` | `AuthClientMiddleware` | routes/main.go:34 → gateway.go:165 |
| POST | `api/connect-geteway` (sic) | `AuthClientMiddleware` | routes/main.go:35 → gateway.go:226 |
| POST | `api/disconnect-default-geteway` (sic) | `AuthClientMiddleware` | routes/main.go:36 → gateway.go:366 |
| **POST** | **`api/fund-transfer`** (โอนเงิน/ถอน — money) | `AuthClientMiddleware` | routes/main.go:38 → gateway.go:408 |
| POST | `api/get-transactions` (ดึง statement บัญชี — money) | `AuthClientMiddleware` | routes/main.go:39 → gateway.go:467 |
- commented out: `api/get-credential` → `GetCredentialsToken` (routes/main.go:29; handler ยังอยู่ gateway.go:112) 🟡

**Money endpoint รายละเอียด:**
- `fund-transfer` (gateway.go:408-465): body `{from_account_bank_code, from_account_no, to_account_bank_code, to_account_no, transfer_type, amount}`; หา connect ของ customer (typeAcc=withdraw, gateway.go:428), init gateway (TMN), ถ้า `to_account_bank_code=="TRUEWALLET"` → `FundTransfer` (p2p) ไม่งั้น `FundTransferAccount` (gateway.go:445-460); อัปเดต credentials_token + total_balance กลับ DB (gateway.go:461-463)
- `connect-geteway` (gateway.go:226-364): หลังเชื่อมต่อ → upsert `gateway_connect`, อัปเดต `customer_registry.list_connect`, แล้ว **เรียก K8S helm install** (gateway.go:359)
- `get-transactions` (gateway.go:467-509): ดึง statement วันนี้จาก TMN, อัปเดต credentials_token+balance

### consume (queues)
ไม่พบ — ไม่มี RabbitMQ/Kafka ใน repo นี้

### cron
ไม่พบ — ไม่มี ticker/scheduler

## consumes
### http-out
| env key / source | base URL value | paths called | evidence |
|---|---|---|---|
| **hardcoded** | `https://dp.thezeus.tech/api/initWithdraw` | POST form (install/uninstall helm K8S) | service/helminstall/install-helm.go:14, :12-27 |
| **hardcoded** | `https://api.tmn.one/api.php` (TmnoneEndpoint) | proxy cmd (cached token, shield id, sign256, set token) ของ TrueMoney | tmn-one/main.go:24; tmn-one/client.go:43 |
| **hardcoded** | `https://api.tmn.one/proxy.dev.php/tmn-mobile-gateway/` (WalletEndpoint) | login/pin, balance, transactions, p2p/account transfer, voucher, face/pin verify | tmn-one/main.go:25; tmn-one/client.go:140 |
| **hardcoded** | `https://api.ipify.org?format=json` | check egress IP (`checkIP`) | tmn-one/client.go:110 |
| `PROXY_URL` (env) | runtime 🟡 | ตั้ง fasthttp proxy dialer สำหรับเรียก wallet | tmn-one/client.go:210-223; config/environment.go:23 |

### publish
ไม่พบ (ไม่มี message queue)

### data (MongoDB — DB จาก env `DB_ENDPOINT` + `DB_DBNAME`)
| collection | R/W | evidence |
|---|---|---|
| `gateway_config` | R (กรอง gateway ที่ลูกค้าเข้าถึงได้) | config/entity.go:15; gateway.go:60,:265,:562 |
| `customer_registry` | R/W (register, list_connect upsert/pull) | config/entity.go:16; customer.go:48,:87; gateway.go:340,:346,:401; middleware/customer-client.go:27 |
| `gateway_connect` | R/W (upsert connection, update token+balance) | config/entity.go:17; gateway.go:66,:187,:305,:462,:505,:533 |
- Redis: มี `service/thorlock` (distributed lock + cache go-redis v8, couterRedis.go) แต่ **ไม่พบการเรียกใช้จาก controller/route** — dead/unwired (grep thorlock = เฉพาะภายใน package เอง)

### external
| third-party | purpose | auth | evidence |
|---|---|---|---|
| **TrueMoney Wallet** (`api.tmn.one` ผ่าน gateway-hub/tmn-one) | login ด้วย pin, เช็คยอด, ดึง statement, โอน p2p / โอนเข้าบัญชีธนาคาร, สร้าง voucher, face/pin verify | login_token + pin (hashSha256 TmnID+Pin), signature ต่อ request, X-Shield-Session-Id, X-KeyID | tmn-one/login.go:50-73; tmn-one/transaction.go (ทั้งไฟล์); tmn-one/pin.go:16; tmn-one/cmd.go:45 |
| **K8S deploy API** (`dp.thezeus.tech`) | install/uninstall withdraw worker เมื่อ connect/disconnect gateway | header `api-key` (**hardcoded**) | service/helminstall/install-helm.go:13-26 |
| **GCP KMS** (Tink) | AEAD envelope encryption (EncAeadKMS) | ADC | service/encryption/aead.go:80-132 |
- gateway factory: รองรับ `TMN_ONE` และ `TMN_TWO` (ทั้งคู่ map ไป InitTMN เดียวกัน) — gateway-hub/factory.go:40-47

## observations
1. **Hardcoded K8S API key** — `URLK8S_API_KEY := "mXjtZ7q9D0zhG6NN06832mqLCr0KlM"` และ URL `https://dp.thezeus.tech/api/initWithdraw` ฝังในโค้ด (service/helminstall/install-helm.go:13-14) — secret รั่ว
2. **DecryptMid middleware ถูก disable แบบอันตราย** — `ctx.Next(); return` วางไว้บนสุดของฟังก์ชัน ทำให้โค้ด decrypt ทั้งหมดด้านล่างเป็น dead code (middleware/decrypted.go:21-22) — request body ไม่ถูกถอดรหัส/ตรวจสอบเลย
3. **Auth middleware เทียบ api_key แบบ plain equality ใน DB query** — `credential.api_key` match ตรง ๆ ไม่ hash/constant-time (middleware/customer-client.go:26) — timing/credential stuffing risk; api_key สร้างจาก `GenerateRandomString(32)` (customer.go:63)
4. **register เป็น public endpoint ไม่มี rate limit / auth** (routes/main.go:31) — ใครก็สมัครได้, ส่งคืน encrypted_token (clientID+apiKey) (customer.go:58,:93)
5. **TMN AES-CBC ไม่มี MAC (no AEAD) ใน client request body** — `encryptAES` ใช้ CBC + PKCS7 ไม่มี integrity tag (tmn-one/cryption.go:32-45); key มาจาก SHA-512(loginToken) ตัด 32 byte (cryption.go:17-22), IV random ต่อ request (cryption.go:24-30) — ฝั่ง encrypt body ไป tmn.one ไม่มี authentication
6. **Wallet access token encryption ก็ CBC ไม่มี MAC** — `encryptAndCreateRequest`/`decryptWalletAccessToken` ใช้ AES-CBC, key=SHA512(TmnID) (cryption.go:107-173); `unpad` ไม่ validate padding (cryption.go:175-179) — padding oracle risk
7. **crypto ภายในระบบ (EncryptAesGcm) ใช้ AES-256-GCM ถูกต้อง** — มี AEAD, nonce random, ตรวจ key len=32 (service/encryption/aes.go:13-43) — **คุณภาพดี**; ใช้เก็บ credentials_token ของ TMN ทั้งก้อน struct (tmn-one/main.go:258 `GetCredentialsToken` → EncryptAesGcm AES_KEY_FIX32)
8. **Tink AEAD/Hybrid ใช้ insecurecleartextkeyset.Read** — โหลด keyset แบบ cleartext (ไม่ผ่าน KMS wrap) สำหรับ EncryptedAead/Hybrid (aead.go:28,:53; hybrid.go:29,:53) — private key เก็บ plaintext ใน env `PRIV_KEYSET` (config/environment.go:21) — ความเสี่ยง key รั่ว; แต่ EncAeadKMS (aead.go:80) ใช้ KMS wrap ถูกต้อง (มีสองแบบปนกัน)
9. **AES_KEY_FIX32 เป็น symmetric key เดียวทั้งระบบจาก env** (config/environment.go:20) — ใช้ encrypt credentials_token ทุก customer ด้วย key เดียว; ถ้ารั่ว = ถอด TMN credential ทั้งหมด
10. **encrypted_random_key flow ไม่ทำงานจริง** — middleware DecryptMid bypass (ข้อ 2) ทำให้ EncryptedRandomKey header (utils/base-mid.go:14,:37) และ Register's encrypted_random_key (customer.go:34,:51) เป็นการ encrypt response อย่างเดียว ไม่ใช่ end-to-end
11. **HTTP client ไม่มี timeout** — `t.Client = &http.Client{}` (tmn-one/main.go:135; client.go:59) ไม่ตั้ง Timeout → hang ได้; helminstall ก็ `&http.Client{}` ไม่มี timeout (install-helm.go:27)
12. **error swallowed จำนวนมาก** — `tokenEncrypted, _ := encryption.EncryptAesGcm(...)` ทิ้ง error (tmn-one/transaction.go:105,:141); helm install error แค่ Println ไม่ rollback connect (gateway.go:359-362)
13. **Port :8000 hardcoded** ไม่ config ได้ (app.go:65) — ต่างจาก convention env PORT
14. **FundTransfer (p2p) มี logic กลับด้าน** — `if strings.Contains(resp.Status, "success") { return ..., fmt.Errorf("error confirm wallet") }` (tmn-one/main.go:172-174) คืน error เมื่อ success ขณะที่ FundTransferAccount ใช้ `!strings.Contains(...success)` (main.go:201) — น่าจะเป็นบั๊ก
15. **fmt.Errorf(wlResp.Message)** ใช้ message เป็น format string (tmn-one/client.go:202) — go vet warning / format-string risk
16. **OTel/tracing ถูกปิดทั้งหมด** (app.go:43-48,:54) — ไม่มี observability ใน service จัดการเงิน
17. **มีการ retry login เมื่อ token 401 แบบ recursive walletCall** (tmn-one/client.go:173-190) — `isRetry` flag กัน loop ชั้นเดียว แต่ reset token+shield ระหว่าง retry อาจ race ถ้า concurrent
18. **gateway-hub รองรับเฉพาะ TMN** — TMN_ONE/TMN_TWO ชี้ implementation เดียว, default = error "unknown payment gateway" (factory.go:45) — ออกแบบเป็น plugin แต่มี provider เดียว
19. **เก็บ credentials_token (TMN login_token, pin, device_id ฯลฯ ทั้ง struct TMNOne) encrypted ลง gateway_connect** (gateway.go:297; tmn-one/main.go:257-259) — pin ของ TrueMoney ถูกเก็บ (แม้ encrypted) ใน DB
