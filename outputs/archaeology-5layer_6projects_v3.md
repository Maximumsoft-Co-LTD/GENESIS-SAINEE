# System Archaeology v3 — Layer 6: Feature Deep-Dive
**6 Repos: 3rd-payment, GOTOPOPOFFICE, office-abatech, que_payment, telesale-service-mootui, topupserie_mootui**
**วิเคราะห์เมื่อ: 2026-06-10**

> Layer 0–5 ดูได้ที่ [archaeology-5layer_6projects_v2.md](./archaeology-5layer_6projects_v2.md)
> Layer 6 นี้เพิ่มรายละเอียด Feature Deep-Dive + Cross-repo flows

---

## Notation Guide
- `↓ HTTP POST /path` = sync call
- `⟿ queue:NAME` = async publish
- `↩ callback POST /path` = webhook กลับมา
- `🔒 lock` = distributed lock / application-level lock
- `💾 MongoDB: collection` = DB operation
- `📦 Redis cache` = cache read/write
- `⏱ cron:Xs` = triggered by ticker
- `📞 External API` = 3rd-party call

---

## 📦 1. 3rd-payment

### 🔹 Feature 1.1: ฝากเงิน QR (Deposit via QR)

#### ทำอะไรได้บ้าง
สร้าง QR Code / บัญชีธนาคาร สำหรับรับฝากเงิน รองรับ 40+ payment providers, กำหนด min/max, countdown timer, แสดง QR image ที่ gen ด้วย Go + Thai font

#### Full Flow
```
topupserie_mootui (user กด "ฝากเงิน")
  ↓ HTTP POST /api/v2/payin
    {service, payment_code, username, amount, bank_code, office_api}

[3rd-payment] Deposit Controller
  📦 Redis: GET config_{payment_code} (cache 300s)
  💾 MongoDB: service_payment → validate status=1, check whitelist, min/max
  🔒 ThorLock: ObtainLock("PAY_CODE:USER:AMOUNT", 35s)  ← ป้องกัน dup request
  💾 MongoDB: qr_payment → find old order (ช่วง 5 นาที) → ถ้ามี return เดิม

  ↓ SwitchPaymentDeposit() — switch 40+ providers
    📞 External API: e.g. POST peer2pay.com/createQr
       {merchant_no, access_key, amount, order_no, callback_url}
       ← Response: {qr_url, countdown_time}
    Retry 2 ครั้ง ถ้า fail

  💾 MongoDB: qr_payment INSERT
     {order_no, status=0(pending), amount, payment_code, qr_url, countdown_time}

  🔒 ThorLock: ReleaseLock

  ← Response 200: {order_no, url_qrcode, bank_number, countdown_time}
```

**เมื่อลูกค้าโอนเงินแล้ว → ดู Feature 1.2 (Callback)**

#### Repos ที่เกี่ยวข้อง
| ลำดับ | Repo | บทบาท | Endpoint |
|------|------|-------|----------|
| 1 | topupserie_mootui | เรียก deposit | POST /api/v2/payin |
| 2 | 3rd-payment | ประมวลผล, เรียก provider | Deposit Controller |
| 3 | External Payment API | สร้าง QR จริง | peer2pay/hengpay/etc |
| 4 | MongoDB | เก็บ qr_payment | service_payment, qr_payment |
| 5 | Redis (ThorLock) | Lock ป้องกัน dup | key: PAY_CODE:USER:AMOUNT |

#### Happy Path vs Edge Cases
| Scenario | ผลลัพธ์ | Handled? |
|----------|--------|---------|
| ปกติ | QR code + countdown กลับไป user | ✅ |
| Duplicate request ใน 35s | ThorLock block, return error | ✅ |
| Order เดิมยัง pending (5 min) | Return order เดิม | ✅ |
| Provider API timeout | Retry 2 ครั้ง → error | ✅ |
| Amount นอก min/max | Return error "โปรดใส่จำนวน X-Y" | ✅ |
| User spam request (no rate limit) | ❌ ไม่มี rate limit | ❌ |
| User bank ยังไม่ verify | ไม่มี check | ❌ |

#### จุดเสี่ยง
- ❌ ไม่มี rate limiting per user/IP
- ⚠️ Retry count hardcoded (2 ครั้ง) — ถ้า provider slow อาจ request ซ้ำ
- ⚠️ QR image gen แบบ sync block request thread

---

### 🔹 Feature 1.2: รับ Callback จาก Payment Provider

#### ทำอะไรได้บ้าง
รับ webhook จาก 40+ providers เมื่อลูกค้าโอนเงินสำเร็จ ตรวจ signature, update qr_payment → status=1, ส่งต่อไป office ผ่าน RabbitMQ

#### Full Flow
```
External Payment Provider
  ↩ callback POST /api/v2/callback-payin/:payment_code
    {transaction_ref, amount, bank_code, signature/hash}

[3rd-payment] IPWhitelistMiddleware
  💾 MongoDB: service_payment → check whitelist_ip
  ← ถ้าไม่อยู่ whitelist → 403 Forbidden

[3rd-payment] CallbackDeposit Controller (~16K บรรทัด)
  📦 Redis: GET config (payment_code)
  switch payment_name → provider-specific handler (69 cases)
    → Verify signature: MD5/SHA256/RSA (ต่างกันแต่ละ provider)
    ← ถ้า signature ผิด → return error

  💾 MongoDB: qr_payment → findOne({order_no})
    ← status=1 already → return 200 (idempotent)
    ← status=99 (manual) → return 200, skip

  🔒 ThorLock: ObtainLock(order_no, 180s)  ← ป้องกัน duplicate callback
    ← fail → "already processing"

  💾 MongoDB: qr_payment UPDATE
     {status=1, transfer_datetime, payer_account, amount_actual}

  ⟿ queue: QUE_PAYMENT_DEPOSIT → office system
     {type:"DEPOSIT", order_no, service, username, amount, status:1}

  🔒 ThorLock: ReleaseLock
  ↩ Response ตาม format ของ provider
```

#### จุดเสี่ยง
- 🔴 `callback.go` 16,182 บรรทัด = monolith 69 cases ใน switch
- ⚠️ RabbitMQ publish ล้มเหลว → qr_payment update แล้ว แต่ office ไม่รู้ (no retry)
- ⚠️ Nack(false,false) ใน queue consumer = drop message ถาวร

---

### 🔹 Feature 1.3: ถอนเงิน (Withdraw)

#### Full Flow
```
topupserie_mootui
  ↓ HTTP POST /api/v2/create-withdraw
    {payment_code, service, username, amount, bank_code, bank_number}

[3rd-payment] CreateWithdraw Controller
  💾 MongoDB: service_payment → validate, check min/max
  💾 MongoDB: bank_summary → check current_balance >= amount+fee
    ← ไม่พอ → return "ยอดเงินไม่เพียงพอ"

  cost = (amount/100) * fee_withdraw
  total = amount + cost

  💾 MongoDB: withdraw_statement INSERT {status=0, order_no, ...}
  💾 MongoDB: bank_summary UPDATE {current_balance -= total}

  ↓ SwitchPaymentWithdraw() → 40+ provider switch
    📞 External API: e.g. POST speedpay.com/api/defray/V2
       {merchant_no, order_no, amount, bank_code, bank_number, MD5_sign}
       ← {transaction_ref, status}

  💾 MongoDB: withdraw_statement UPDATE {transaction_ref, status=2(waiting)}
  ⟿ queue: QUE_PAYMENT_WITHDRAW → office

  ← Response: {order_no, amount, cost, status:"pending", transaction_ref}

—— เมื่อ provider โอนเสร็จ ——
  ↩ callback POST /api/v2/callback-payout/:payment_code
  💾 MongoDB: withdraw_statement UPDATE {status=1, is_success=1}
  ⟿ queue: QUE_PAYMENT_PAYOUT → office
```

#### จุดเสี่ยง
- 🔴 ไม่มี ThorLock บน withdraw → duplicate request ได้
- 🔴 Balance deduct + provider call ไม่ atomic → crash กลางทาง = inconsistent
- ⚠️ Provider fail → ต้อง manual rollback balance

---

## 📦 2. GOTOPOPOFFICE

### 🔹 Feature 2.1: Admin อนุมัติฝากเงิน (DepositAddCredit)

#### ทำอะไรได้บ้าง
Admin ตรวจสลิป อนุมัติ → เรียก Agent API เพิ่มเครดิต → update deposit_statement → publish RabbitMQ

#### Full Flow
```
office-abatech (Admin กด "อนุมัติ")
  ↓ HTTP POST /deposit/:service/DepositAddCredit (JWT required)
    {id, status, ...}

[GOTOPOPOFFICE] AuthRequiredDB Middleware → verify JWT + role

DepositAddCredit Controller
  💾 MongoDB: deposit_statement findOne {_id: id}
  💾 MongoDB: member_account findOne {username}
  💾 MongoDB: database_config findOne {service} → get Agent API URL

  ↓ HTTP GET {AGENT_API}/deposit?key=...&point=...&username=...
    ← Response: {CurrentBalance, PreviousDownlineGiven, Ref}
    ← error → return "API AGENT เกิดข้อผิดพลาด" (status ไม่ update)

  💾 MongoDB: deposit_statement UPDATE
     {status=1, after_credit, is_check=1}

  ⟿ RabbitMQ: STATEMENT_{service} (fanout exchange)
     → listener update member_credit downstream

  💾 MongoDB: working_log INSERT (audit)

  ← Response: {code:0, msg:"ทำรายการสำเร็จ"}
```

#### Repos ที่เกี่ยวข้อง
| ลำดับ | Repo | บทบาท | Endpoint |
|------|------|-------|----------|
| 1 | office-abatech | Admin UI ส่ง request | POST /deposit/:service/DepositAddCredit |
| 2 | GOTOPOPOFFICE | ประมวลผล approval | DepositAddCredit controller |
| 3 | topupserie_mootui (Agent API) | เพิ่มเครดิต player | GET /deposit?key=... |
| 4 | MongoDB | deposit_statement, member_account | update status + credit |
| 5 | RabbitMQ | Event fanout | STATEMENT_{service} |

#### จุดเสี่ยง
- 🔴 ไม่มี MongoDB transaction → Agent API success แต่ DB update fail = inconsistent
- 🔴 Race condition บน credit update (concurrent requests)
- ⚠️ RabbitMQ publish ไม่มี retry → message loss

---

### 🔹 Feature 2.2: สมัครสมาชิก (RegisterMember)

#### Full Flow
```
office-abatech (Admin กด "สมัคร")
  ↓ HTTP POST /member/:service/RegisterMember
    {username, password, phone_number, bank_code, bank_number}

[GOTOPOPOFFICE]
  💾 MongoDB: database_config → get APIFRONTENDURL, apikey

  ↓ HTTP POST {office-abatech_API}/auto/register
     {username, password, phone_number, bank_code, bank_number}
     Header: x-api-key
     ← Response: {code:0, username}

  💾 MongoDB: working_log INSERT

  ← Response relay กลับ

—— async: bank verify ——
  ↩ POST /webhook/verify-bank/{username}
  💾 MongoDB: member_account UPDATE bank_list status
```

#### จุดเสี่ยง
- ⚠️ ไม่มี idempotency key → double-click = 2 accounts
- ⚠️ Bank verify webhook อาจไม่ trigger → member stuck "pending bank"
- ⚠️ Password ส่งผ่าน HTTP → ต้องการ TLS + server-side hash

---

### 🔹 Feature 2.3: Report ฝาก-ถอน (Statement Reports)

#### Full Flow
```
office-abatech (Admin เลือก date range)
  ↓ GET /deposit/:service/DepositStatementListByFilter?start=...&end=...

[GOTOPOPOFFICE]
  📦 Redis: GET DepositSummaryByAccount_{service}_{date} (TTL=2min)
    HIT → return cached
    MISS →
      💾 MongoDB: deposit_statement aggregate
         $match: {datetime: {$gte, $lt}, status}
         $sort, $limit, $skip (pagination)
      📦 Redis SET (TTL=2min)

  ← Response: {payload: [...], count: N}
```

#### จุดเสี่ยง
- ⚠️ Cache ไม่ invalidate เมื่อ deposit เปลี่ยนสถานะ → stale 2 นาที
- ⚠️ ไม่มี compound index บน (datetime, status) → slow query บน dataset ใหญ่
- ⚠️ Date parse เป็น time.Local → timezone confusion

---

## 📦 3. office-abatech (Frontend)

### 🔹 Feature 3.1: Login (LINE OAuth + 2FA + Password Reset)

#### Full Flow
```
——— Path A: LINE OAuth ———
User กด "Login with LINE"
  → Redirect: https://access.line.me/oauth2/v2.1/authorize
  ← LINE redirect: /login?code=LINE_AUTH_CODE

login.vue mounted()
  ↓ HTTP GET /login?code=XXX&detail=IP
  ← Response: {code:0, token, header, group}

  jwt.decode(token) → payload.result (user object)
  📦 Vuex commit: user, token, headertoken, serviceGroup
  💾 localStorage: token, headertoken, serviceGroup, group, webname, logo

  level >= 9 → redirect /Dashboard
  level < 9 → redirect /News

——— Path B: Username/Password + 2FA ———
User กรอก username + password
  ↓ HTTP POST /LoginUserPass {username, password}
  ← {code:0, payload: {id, is_first_login, is_encode_2fa}}

  is_first_login == false → STEP=2 (Reset Password)
  ↓ HTTP POST /ResetPassword {id, username, password, new_password}
  ← {secret, qrcode_secret: "otpauth://..."}
  STEP=3 (Scan QR → Google Authenticator)

  User กรอก OTP 6 หลัก
  ↓ HTTP POST /LoginVerify {id, username, password, pin}
  ← {code:0, token, header, ...}
  [same as Path A: store + redirect]
```

#### จุดเสี่ยง
- 🔴 CLAUDE_API_KEY ใน .env → expose ใน nuxt.config.js env block → ส่งไปทุก user browser
- 🔴 Firebase credentials hardcoded ใน TopNavbar.vue (email + password plaintext)
- 🔴 JWT signing secret `"@ABAOFFICE"` visible ใน frontend code → forge token ได้
- ⚠️ Token ใน localStorage → XSS steal ได้ (ควรใช้ HttpOnly cookie)
- ⚠️ ไม่มี token refresh → 7h หมดอายุ → ค้างหน้าจอ

---

### 🔹 Feature 3.2: อนุมัติฝากเงิน Manual (Withdraw Manual Approval)

#### Full Flow
```
Admin navigates to /WithdrawManual

components/WithdrawManual/WithdrawManualList.vue mounted()
  ↓ HTTP GET /GetWithdrawManualServiceList ← bank dropdown
  ↓ HTTP POST /GetWithdrawManualList {web, bank_number, date}
  ← [{id, username, Withdraw, Deposit, Status, Datetime, operator}]

Admin กรอก username field → กด "ยืนยัน"
  row.operator = store.user.name
  ↓ HTTP POST /UpdateStatusWithdrawManual {id, status:3, username}
  ← {code:0}
  row.Status = 3 (visual update)

Admin กด "ยกเลิก"
  ↓ HTTP POST /UpdateStatusWithdrawManual {id, status:2, username}
  ← {code:0}  ← refund queued ใน backend
```

#### จุดเสี่ยง
- 🔴 Race condition: 2 admins อนุมัติ request เดียวกันพร้อมกัน → double payment
- 🔴 ไม่มี rollback ถ้า payment API fail หลัง status=3
- ❌ ไม่มี error handling เมื่อ API return code != 0 (silent fail)
- ⚠️ ไม่มี real-time update → admin ต้อง refresh manual

---

### 🔹 Feature 3.3: Dashboard + Real-time Withdraw Monitor

#### Full Flow
```
Admin navigates to /Dashboard

DashboardAll.vue mounted()
  ↓ HTTP POST /GetDashboardSummaryALL
  ← [{datestring, deposit, withdraw, profit, count_member_online, ...}] (14 วัน)
  Render: Bar chart (Chart.js)

TopNavbar.vue (always loaded)
  🔔 Firebase RT: ref("/TESTWITHDRAW/{service}")
    .on("child_added") → popup แจ้ง admin ทันที

Admin กด Refresh
  ↓ HTTP POST /GetDashboardSummaryALLQuery (clear cache)
  → push router /Dashboard?1={random} (force reload)
```

#### จุดเสี่ยง
- ⚠️ Firebase apiKey + email + password hardcoded ใน component
- ⚠️ Dashboard ไม่ auto-refresh (manual only) นอกจาก Firebase notification

---

## 📦 4. que_payment

### 🔹 Feature 4.1: ประมวลผลถอนเงิน (Withdrawal CronJob)

#### ทำอะไรได้บ้าง
Worker ดึงรายการถอนจาก MongoDB queue ทุก 20 วินาที → lock bank_summary → ส่งไป provider ผ่าน RabbitMQ → callback office

#### Full Flow
```
⏱ cron:20s [StartServV2]

  💾 MongoDB: que_payment
     $match: {status=0, payment_name=PAYMENT_NAME}
     $sort: create_at:1 (FIFO)
     $group: first per service (dedup)
  ← []QuePayment items

  For each item (max 10 concurrent, semaphore):
    [QueRunningV2]
    💾 MongoDB: que_payment UPDATE {status=2} (processing)
    💾 MongoDB: withdraw_statement findOne (retry 3x, delay 5s)
      ← is_success==1 → skip (already done)

    🔒 is_active lock: bank_summary UPDATE {is_active=1}
      ← is_active already 1 → return error (bank busy)

    ⟿ RabbitMQ: CallRabbitWithdraw(payment_code, order_no)
       → queue: QUE_PAYMENT_{NAME}
       ← Response: {code, message}

    🔒 Unlock: bank_summary UPDATE {is_active=0}
    💾 MongoDB: que_payment UPDATE {status=1} (done)

    ↩ callback POST office/WithdrawCallback
       {status:1, orderNo, comment:"สำเร็จ"}
```

#### Repos ที่เกี่ยวข้อง
| ลำดับ | Repo | บทบาท | Queue/Endpoint |
|------|------|-------|----------------|
| 1 | que_payment | CronJob worker | ⏱ cron:20s |
| 2 | 3rd-payment / payment providers | ส่ง payout | ⟿ QUE_PAYMENT_{NAME} |
| 3 | GOTOPOPOFFICE | รับ callback | ↩ POST /WithdrawCallback |
| 4 | MongoDB | bank_summary lock + que_payment state | is_active flag |

#### จุดเสี่ยง
- 🔴 `is_active` flag = application-level lock (ไม่ใช่ DB lock จริง) → crash = lock ค้าง
- ⚠️ ถ้า RabbitMQ fail → ยัง mark status=1 (ผิดพลาด)
- ⚠️ Callback fire-and-forget → office ไม่รู้ ถ้า fail

---

### 🔹 Feature 4.2: Deposit Consumer (RabbitMQ Mode)

#### Full Flow
```
⟿ queue: QUE_PAYMENT_{PAYMENT_NAME} รับ message

[StartAMQPV2WithContext] OTel span: "QueTypeSwitchV2"
  Unmarshal ReqQue {ID, Service, Type="DEPOSIT", PaymentCode}
    ← JSON error → Nack(false,false) = DROP MESSAGE ❌

  QueTypeSwitchV2 → DepositAgentV2
    💾 MongoDB: qr_payment findOne {order_no}
    💾 MongoDB: service_payment config
      ← is_success==1 → skip

    💾 MongoDB: bank_summary findOne {payment_code, service}
      ← ไม่พบ → CREATE new + sleep(5s) ← ⚠️ race condition

    amountInc = AmountCreate - (CostDeposit + CostWithdraw + Profit)
    💾 MongoDB: bank_summary UPDATE {current_balance += amountInc}
    💾 MongoDB: qr_payment UPDATE {is_success=1}

    ↩ (goroutine) callback POST office/PaymentCreateStatement/{service}
       {datetime, Channel, Deposit, BankNo, balance, before/after_credit}

  Ack message ✓
```

#### จุดเสี่ยง
- 🔴 Nack(false,false) = message หายถาวร ถ้า JSON corrupt (ไม่มี DLQ)
- 🔴 Balance update + qr_payment update ไม่ atomic → crash กลางทาง = inconsistent
- ⚠️ bank_summary create race condition (2 goroutines)

---

### 🔹 Feature 4.3: Retry Failed Withdrawals

#### Full Flow
```
⏱ cron:5m [ServRetryOrder]

  💾 MongoDB: withdraw_statement
     filter: {
       status=16,           ← pending payout
       is_success!=1,
       transaction_ref="",  ← ยังไม่มี ref จาก provider
       datetime: {$gte: -3days, $lt: -5hours}
     }

  For each stale record:
    ← callback_status==1 → skip (already notified)

    💾 MongoDB: withdraw_statement UPDATE {status=4, comment:"รายการถอนหมดอายุ"}
    ↩ callback POST office/WithdrawCallback
       {status:4, orderNo, comment:"รายการถอนหมดอายุ"}
```

#### จุดเสี่ยง
- ⚠️ ไม่มี lock → concurrent retry อาจส่ง callback ซ้ำ
- ⚠️ callback ไม่มี exponential backoff → ถ้า office down ก็หยุดทันที
- ⚠️ CallbackStatus ไม่ update หลัง retry → infinite loop potential

---

## 📦 5. telesale-service-mootui

### 🔹 Feature 5.1: Reserve Member ลงใน Campaign (Reserve Booking)

#### ทำอะไรได้บ้าง
Agent จองสิทธิ์โทรหาสมาชิกใน campaign group ระบบใช้ MongoDB transaction ป้องกัน double-booking และอัปเดต counter ของ group + operator แบบ atomic

#### Full Flow
```
Call Center Agent
  ↓ HTTP POST /api/v1/reserve
    {service, group_id, username, phone_number, operator_id}

[telesale] ReserveService.Reserve()
  💾 MongoDB: reserve findOne
     {group_id, phone_number, username, status:1(active)}
     ← พบ → ErrReserved ❌

  💾 MongoDB Transaction START ───────────────────────────────┐
    💾 MongoDB: group findOne {_id: group_id}                 │
    💾 MongoDB: group_operator find/create {group_id, operator_id} │
       group_operator.ReserveMemberCount += 1                 │
    💾 MongoDB: reserve INSERT {status:1, ...}                │
    💾 MongoDB: group UPDATE {ReserveMemberCount += 1}        │
  MongoDB Transaction COMMIT ──────────────────────────────────┘

  📦 Redis: InvalidateByPrefix("group_reserve_{group_id}")

  ← Response: {id, status:200, data: reserve}
```

#### จุดเสี่ยง
- 🔴 ไม่มี auth middleware → ทุกคน reserve ได้โดยไม่ต้อง login
- ⚠️ Check `existReserve` อยู่นอก transaction → race condition ได้
- ⚠️ ไม่มี phone format validation

---

### 🔹 Feature 5.2: โทรหาลูกค้า + รับ Recording (MakeCallV2 + Callback)

#### Full Flow
```
——— MAKE CALL ———
Agent กด "โทร"
  ↓ HTTP POST /api/v2/call
    {reserve_id, operator_id, member_info{phone_number, username, ...}}

[telesale] CallService.MakeCallV2()
  💾 MongoDB: reserve findOne → validate owner + not canceled
  💾 MongoDB: extension_operator findOne → validate active
  💾 MongoDB: provider_extension findOne → get extension, phone, appKey
  💾 MongoDB: provider findOne → get ApiDomain, ApiKey, ApiID

  💾 MongoDB Transaction START ──────────────────────────────────┐
    💾 MongoDB: call INSERT                                       │
       {_id: newID, status:Pending, reserve_id, phone_number,    │
        operator_id, caller_number, member_info, datetime}        │
    📞 YaleCom API: GET /robocall/send                           │
       ?company_id=...&key=...                                    │
       &caller_id_number={extension.phone}                       │
       &destination_number={member.phone_number}                 │
       &transfer_to={extension}                                  │
       &ref_1={call.ID}  ← สำคัญ: ใช้ link callback กลับมา     │
    ← {status: true/false, message}                             │
    ← fail → ErrProviderFailed → rollback                       │
  MongoDB Transaction COMMIT ────────────────────────────────────┘

  (goroutine) counter increments: extension, provider, reserve, group

  ← Response: {id, status:Pending, ...}

——— RECEIVE RECORDING CALLBACK ———
YaleCom (หลังโทรเสร็จ)
  ↩ HTTP POST /api/v1/provider/callback/yalecom
    {data: "{xml_cdr_uuid, call_result, call_duration, start_stamp,
             end_stamp, ref_1: call_id, ...}" }

[telesale] ProviderService.ReceiveCallDetail()
  Unmarshal nested JSON → CallBackDetail
    ref_1 → call.ID

  💾 MongoDB: call findOne {_id: ref_1}
  💾 MongoDB Transaction:
    call UPDATE {callback_detail, callback_status:1}
    reserve UPDATE {callback_detail}
  COMMIT
```

#### Repos ที่เกี่ยวข้อง
| ลำดับ | Repo | บทบาท | Endpoint |
|------|------|-------|----------|
| 1 | telesale-service-mootui | Call orchestrator | POST /api/v2/call |
| 2 | YaleCom VoIP | ส่งสาย + recording | GET /robocall/send |
| 3 | YaleCom (webhook) | ส่ง callback | POST /api/v1/provider/callback/yalecom |
| 4 | MongoDB | call, reserve, provider records | transaction |

#### จุดเสี่ยง
- 🔴 ไม่มี webhook signature verification → anyone POST fake callback ได้
- 🔴 member_info จาก request param (ไม่ใช่ reserve record) → data override
- ⚠️ Counter increments แบบ goroutine async → crash = counter ผิด
- ⚠️ Callback ก่อน transaction commit = call ไม่พบ

---

### 🔹 Feature 5.3: รายงาน Operator (Operator Report)

#### Full Flow
```
Manager
  ↓ HTTP GET /api/v1/report/operator
    ?service=...&start_date=...&end_date=...&operator_id=...&page=1&limit=10

[telesale] ReportService.GetOperatorReport()
  💾 MongoDB Aggregation #1: count distinct operators
     $match → $group {_id: operator_id} → $count

  💾 MongoDB Aggregation #2: paginated operator stats
     $match {datetime:range, operator_id, service}
     $group {_id: operator_id,
             total_call: $sum:1,
             total_success: $sum($cond status==1),
             total_failed: $sum($cond status==2),
             avg_duration: $avg}
     $sort {total_call:-1}
     $skip, $limit

  ← Response: {meta:{page,limit,total}, records:[...]}
```

#### จุดเสี่ยง
- ⚠️ ไม่มี auth middleware → public endpoint
- ⚠️ ไม่มี compound index บน (datetime, operator_id, service) → slow aggregation
- ⚠️ ไม่มี cache → ทุก request hit DB

---

## 📦 6. topupserie_mootui

### 🔹 Feature 6.1: ฝากเงิน QR Auto (QRpay Payment Auto)

#### ทำอะไรได้บ้าง
ระบบเลือก payment gateway อัตโนมัติด้วย **scoring algorithm** (fee 60% + range 40%) จาก 5 channels → retry fallback ถ้า channel ล้มเหลว

#### Full Flow
```
User กด "ฝากเงิน QR"
  ↓ HTTP POST /qrpay_payment_auto (JWT required)
    {username, amount, bank_code, bank_number}

[topupserie] middleware.AuthHeader → verify JWT

Controller:
  📦 Redis: GET config_system (ConfigSystem cache)
  💾 MongoDB: member_account findOne → validate bank + fullname
  ↓ HTTP GET API_QR_ALL/v2/get-bankconfig → list payment gateways

  calculatePaymentScoreSimple():
    For each payment:
      FeeScore  = (1 - fee/maxFee) * 100 * 0.6
      RangeScore = (amount อยู่ช่วง 40-60% ของ min-max) * 0.4
      TotalScore = FeeScore + RangeScore
    Sort by score DESC, shuffle, limit 5

  RETRY LOOP (max 5 channels):
    For each payment_code:
      💾 MongoDB: generate_statement INSERT
         {status:0, TypeGenerate:QRPAYVER2, EndDate:+10min}

      ↓ HTTP POST API_QR_ALL/v2/payin
         {payment_code, username, amount, firstName, lastName, office_api}
         ← {Code, Data.UrlQRcode, CountdownTime, OrderNo}

      success → break loop
      fail    → sleep 50ms → next channel

  ⟿ RabbitMQ: DEPOSIT_QRPAY_AUTO event (goroutine)
  💾 MongoDB: user_log INSERT

  ← Response: {UrlQRcode, DepositPaymentType, CountdownTime, OrderNo, BankName}
```

#### Cross-repo: Full Deposit Journey
```
User → topupserie [POST /qrpay_payment_auto]
     → 3rd-payment [POST /v2/payin] via API_QR_ALL
     → External Provider (Peer2Pay/Hengpay/etc) → QR Code
     ← QR URL กลับมา user

[User โอนเงิน]

     ↩ Provider callback → 3rd-payment [POST /callback-payin/:code]
       🔒 ThorLock(180s)
       💾 qr_payment UPDATE status=1
       ⟿ RabbitMQ: QUE_PAYMENT_DEPOSIT

     ⟿ Queue consumer → que_payment DepositAgentV2
       💾 bank_summary UPDATE balance
       ↩ callback → GOTOPOPOFFICE/topupserie

     GOTOPOPOFFICE DepositAddCredit
       ↓ Agent API → เพิ่มเครดิต
       💾 deposit_statement UPDATE
       ⟿ RabbitMQ: STATEMENT_{service}
```

#### จุดเสี่ยง
- ⚠️ Random shuffle → user ได้ channel ต่างกันแต่ละครั้ง (inconsistent UX)
- ⚠️ ไม่มี dedup → user spam = หลาย generate_statement + หลาย QR
- ⚠️ config_system timer sync → high write contention บน single document

---

### 🔹 Feature 6.2: ถอนเงิน + ตรวจสอบ Turnover

#### Full Flow
```
User กด "ถอนเงิน"
  ↓ HTTP POST /withdraw (JWT required)
    {username, amount, bank_code, bank_number}

[topupserie]
  ↓ HTTP GET SERVICE_API/checkstatus → ตรวจ GOTOPOPOFFICE alive
  📦 Redis: GET config_system
    ← WithdrawDisable=true → 400 "withdrawalClosed"

  ↓ HTTP GET {AGENT}/AgentAmount → current_credit

  CHECK TURNOVER (POST /turnover path):
    💾 MongoDB: deposit_statement findOne (is_check=1, latest)
    IF bonus_type == "TURNSLOT":
      credit >= withdrawFix? ← ไม่ผ่าน → error
    IF bonus_type == "TURNCASINO":
      ↓ HTTP GET APIWHEEL/AgentTurnoverRocketWin → bet summary
      sum(betAmount) >= withdrawFix? ← ไม่ผ่าน → error

  CHECK DAILY LIMIT (LTKV):
    💾 MongoDB: report_active_user_day → todayWithdrawn
    amount + todayWithdrawn <= WithdrawMaxDay?

  CHECK GAME STACK:
    ↓ AgentDeleteUserGameStack(username)  ← ลบ game stack ก่อน
    ← error "เงินค้างอยู่ในเกม" → 423

  ⟿ RabbitMQ: ORIGINWITHDRAW
     {username, amount, bank_number, type:"ORIGINWITHDRAW"}

  💾 MongoDB: user_log INSERT

  ← Response: {code:0, data: queue_object}
```

#### Cross-repo: Full Withdraw Journey
```
User → topupserie [POST /withdraw]
     ⟿ RabbitMQ: ORIGINWITHDRAW

     GOTOPOPOFFICE (queue consumer)
       💾 withdraw_statement INSERT
       ↓ 3rd-payment [POST /api/v2/create-withdraw]
         💾 bank_summary deduct
         📞 External Provider payout API
         ← {transaction_ref}

     [Provider โอนเงินเสร็จ]
     ↩ Provider callback → 3rd-payment [POST /callback-payout/:code]
       💾 withdraw_statement UPDATE {status=1}
       ⟿ que_payment OR direct callback

     ↩ GOTOPOPOFFICE/topupserie
       💾 member credit deducted (confirmed)
```

#### จุดเสี่ยง
- 🔴 Race condition: 2 concurrent withdraw requests → duplicate
- ⚠️ Game turnover check sync → APIWHEEL slow = user wait นาน
- ⚠️ ไม่มี PIN verification บน withdraw endpoint

---

### 🔹 Feature 6.3: รับโบนัส (Bonus Selection)

#### Full Flow
```
User กด "รับโบนัส"
  ↓ HTTP POST /bonus (GET list)

[topupserie]
  📦 Redis: GET config_system
  💾 MongoDB: member_promotion findOne {active today}
  💾 MongoDB: deposit_statement findOne {is_check=1, latest} → bonus_id
  💾 MongoDB: bonus_config aggregate (status=1, type=DEPOSIT)

  selectBonusMulti2() — filter bonus by 15+ conditions:
    onetime_user, secondtime_user, firsttime_day, ranking_N,
    continue_N, campaign (hydra_id), unlimit, 104 (no bonus), ...

  ← Response: {bonus: [...eligible bonuses]}

User เลือก bonus_id
  ↓ HTTP POST /bonus (select)
    {bonus_id}  ← หรือ 104 = ไม่รับ

  💾 MongoDB: deposit_statement → check IsWithdrawCondition
  CHECK: member_credit <= minClearBalance (10 บาท)
    ← เกิน → error "ต้องน้อยกว่าเท่ากับ"

  IF bonus_id == 104:
    💾 MongoDB: member_account UPDATE {is_bonus=0}
    💾 MongoDB: member_promotion CANCEL
    ← success

  ELSE:
    ↓ AgentDeleteUserGameStack ← ลบ game stack ก่อน
    💾 MongoDB: member_promotion INSERT {status=0, bonusID, endDate}

    IF DepositType == 2:
      ⟿ RabbitMQ: WALLET-TO-GAME (transfer bonus → game wallet)

  💾 MongoDB: user_log INSERT
  ← Response: {code:0, msg:"success"}
```

#### จุดเสี่ยง
- 🔴 Race: 2 concurrent select requests → 2 member_promotion
- ⚠️ selectBonusMulti2() = function 300+ บรรทัด, nested conditions → ยาก maintain
- ⚠️ Bonus endDate parse เป็น local timezone → timezone bug

---

## 🗺️ Cross-Repo Flow Summary (3 flows สำคัญ)

### Flow A: ฝากเงิน QR แบบเต็ม
```
User
  ↓
topupserie [POST /qrpay_payment_auto]
  ↓ scoring → select payment
  ↓ HTTP POST API_QR_ALL/v2/payin
3rd-payment [Deposit Controller]
  ↓ ThorLock → validate → qr_payment INSERT
  ↓ HTTP POST peer2pay.com/createQr
External Provider ← QR URL
  ↓
← QR Code กลับ User (countdown 10 min)

[User โอนเงิน → Provider ตรวจสอบ]

  ↩ Provider → 3rd-payment [POST /callback-payin/:code]
     ThorLock(180s) → verify sig → qr_payment UPDATE status=1
     ⟿ RabbitMQ: QUE_PAYMENT_DEPOSIT

  ⟿ que_payment consumer [DepositAgentV2]
     bank_summary UPDATE balance
     ↩ callback → topupserie/GOTOPOPOFFICE

  GOTOPOPOFFICE [DepositAddCredit]
     ↓ HTTP GET AgentAPI/deposit → เพิ่มเครดิต
     deposit_statement UPDATE status=1
     ⟿ RabbitMQ: STATEMENT_{service}

User เห็นเครดิตเข้า ✓
```

### Flow B: ถอนเงินแบบเต็ม
```
User
  ↓
topupserie [POST /withdraw]
  → check turnover, daily limit, game stack
  ⟿ RabbitMQ: ORIGINWITHDRAW

GOTOPOPOFFICE (consumer)
  💾 withdraw_statement INSERT
  ↓ HTTP POST 3rd-payment/api/v2/create-withdraw

3rd-payment [CreateWithdraw]
  💾 bank_summary deduct
  ↓ HTTP POST speedpay.com/api/defray/V2 (MD5 signed)
  ← {transaction_ref, status:pending}
  💾 withdraw_statement UPDATE {transaction_ref, status=2}
  ⟿ RabbitMQ: QUE_PAYMENT_WITHDRAW

[Provider โอนเงินเสร็จ ~1-5 นาที]

  ↩ Provider → 3rd-payment [POST /callback-payout/:code]
     ThorLock → verify → withdraw_statement UPDATE status=1
     ⟿ RabbitMQ: QUE_PAYMENT_PAYOUT

que_payment OR GOTOPOPOFFICE (consumer)
  member credit deducted (confirmed)

User เห็นยอดถูกหัก ✓
```

### Flow C: Telesale โทรหาลูกค้า
```
Manager
  ↓
telesale [POST /api/v1/group] → create campaign
telesale [POST /api/v1/reserve] → book member (MongoDB transaction)
  reserve.ReserveMemberCount++, group.ReserveMemberCount++

Agent
  ↓
telesale [POST /api/v2/call]
  → MongoDB Transaction:
     call INSERT + YaleCom API call (GET /robocall/send?ref_1=callID)
  COMMIT

  YaleCom → ส่งสายไป customer phone

[Call เสร็จ ~2-5 นาที]

  ↩ YaleCom → telesale [POST /api/v1/provider/callback/yalecom]
     ref_1 → callID
     call UPDATE {callback_detail, recording_url, duration, status}
     reserve UPDATE {callback_detail}

Agent อัปเดต call result (status, bonus_id, description)
  ↓
telesale [PATCH /api/v1/call/:id/detail]
  💾 call UPDATE {status, description, bonus_id, sms_message}

Manager ดูรายงาน
  ↓
telesale [GET /api/v1/report/operator]
  💾 MongoDB aggregate → {total_call, total_success, avg_duration per operator}
```

---

## 🔴 Critical Risks สรุปรวม

| ลำดับ | Issue | Repo | Type | Priority |
|------|-------|------|------|----------|
| 1 | `callback.go` 16K monolith (69 switch cases) | 3rd-payment | Maintainability | P0 |
| 2 | CLAUDE_API_KEY expose ใน nuxt env block | office-abatech | Security | P0 |
| 3 | JWT secret `"@ABAOFFICE"` visible ใน frontend | office-abatech | Security | P0 |
| 4 | Nack(false,false) = drop RabbitMQ message | que_payment | Data Loss | P0 |
| 5 | ไม่มี MongoDB transaction บน DepositAddCredit | GOTOPOPOFFICE | Data Integrity | P0 |
| 6 | is_active flag = application-level lock (ไม่ใช่ DB) | que_payment | Race Condition | P1 |
| 7 | ไม่มี auth middleware (ทุก endpoint) | telesale | Security | P1 |
| 8 | ไม่มี webhook signature verification | telesale | Security | P1 |
| 9 | Double withdraw (no lock บน topupserie) | topupserie | Financial Risk | P1 |
| 10 | Race condition บน Withdraw Manual | office-abatech | Financial Risk | P1 |
| 11 | `dgrijalva/jwt-go` deprecated CVE | GOTOPOPOFFICE, topupserie | Security | P1 |
| 12 | balance deduct + provider call ไม่ atomic | 3rd-payment | Data Integrity | P2 |
| 13 | ไม่มี rate limit บน /qrpay endpoints | topupserie | DDoS | P2 |
| 14 | selectBonusMulti2() = 300 บรรทัด, 15+ conditions | topupserie | Maintainability | P2 |
| 15 | 0% test coverage ทุก repo | ทั้งหมด | Reliability | P2 |

---

*Generated by `/archaeology-5layer-v3` — Layer 6 Feature Deep-Dive | 2026-06-10*
