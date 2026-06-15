# Card: office-v10x
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- **stack:** ไม่ใช่ Nuxt — เป็น **Vite 4 + Vue 3 SPA** (TypeScript, Pinia, vue-router, Tailwind, naive-ui) — `office-v10x/package.json:67` (`"vue": "^3.2.36"`), `office-v10x/package.json:107` (`"vite": "^4.3.5"`), `office-v10x/vite.config.ts:14` (`plugins: [vue()]`)
- **package name:** `office-v10x` v0.0.0 — `office-v10x/package.json:2-3`; เวอร์ชันจริงประกาศใน console: `"office-v10x v2.5.15"` — `office-v10x/src/plugins/trackversion.ts:2`
- **SSR/SPA:** SPA ล้วน (client-side routing `createWebHistory` — `office-v10x/src/router/index.ts:14`; production เสิร์ฟ static ด้วย `serve -s` — `office-v10x/Dockerfile:23`)
- **port:** dev = 8088 (`office-v10x/vite.config.ts:13`), Docker/prod = 8080 (`office-v10x/Dockerfile:17,23`)
- **Dockerfile/CI:** multi-stage node:19.9.0-alpine3.17 → pnpm build → `serve -s ./ -l 8080` (`office-v10x/Dockerfile:1-23`); CI ผ่าน reusable workflow ของ Maximumsoft สร้าง image `frontend-officev10x` push เข้า JFrog (`office-v10x/.github/workflows/pipeline.yml:20-23`); compose ระบุ image `thezeustech.jfrog.io/docker-local/frontend-officev10x:uat` (`office-v10x/docker-compose-jfog.yml:6`)

## provides
### http — admin pages (back-office ทั้งระบบ)
- **98 routes** (นับ `component:` ใน `office-v10x/src/router/index.ts`), view component 110 ไฟล์ใต้ `office-v10x/src/views/`
- กลุ่ม page หลัก (ตามโฟลเดอร์ views + route): login/register (`src/router/index.ts:16-33`), dashboard (`src/views/DashboardView.vue`), **transection** (deposit approve / withdraw approve / manage credit / payment gateway — `src/views/transection/`), member (`src/views/member/`), statement (`src/views/statement/`), report (`src/views/report/`), setting + settingWebsite (`src/views/setting/`), employee + role + 2FA (`src/views/employee/`, `src/views/role/`), agent (`src/views/agent/`), extension (coupon/wheel/ranking/mission/sms — `src/views/extension/`), system log (`src/views/system/`), bill, blocklistuser, checkbank, linenotify, whitelist
- ไม่มี server-side API ของตัวเอง — เป็น static SPA (`office-v10x/Dockerfile:23`)

## consumes
### http-out
| source | base URL | หมายเหตุ |
|---|---|---|
| axios instance กลาง `api` | `HOSTNAME` = **dev:** `http://localhost:7777` + `/api` (`src/services/base-url.ts:26,60`) / **prod:** `location.origin + "/api"` 🟡 (same-origin — ตัว backend จริงถูก inject ตอน deploy ผ่าน reverse proxy/ingress ที่ map `/api` ไป backend; ไม่มี env/proxy config ในโค้ด) | `src/services/api-config.ts:10-11`; แนบ `Authorization: Bearer <token>` ทุก request (`src/services/api-config.ts:21-25`) |
| vue-auth3 login driver | `${HOSTNAME}/login-verify` | `src/plugins/vue-auth3.ts:33-35` |
| `AUTHSERVICE` = `https://oauth.pinkmonkey.online` (hardcoded) | 🟡 import แล้วใช้เฉพาะในโค้ดที่ comment ไว้ — ปัจจุบัน OAuth URL จริงได้จาก backend ผ่าน `/make-link-login` | `src/services/base-url.ts:62-63`, `src/stores/auth.ts:8,118`, `src/services/api-service/employee/index.ts:563` |

- dev URL port 7777 ตรงกับ backend **office-api-v10** (และมี dev URL สำรอง ~25 รายการ comment ไว้ — `src/services/base-url.ts:5-57`)
- รูปแบบ path เกือบทั้งหมด: `/<endpoint>/<webService()>` โดย `webService()` = tenant key จาก Pinia/localStorage `web-service` (`src/services/web-service.ts:3-9`) → **multi-tenant ผ่าน path segment**
- API ทั้งหมด **626 unique paths** ใน 26 module ใต้ `src/services/api-service/` (716 จุดเรียก) — กลุ่มสำคัญ:

**เงิน — deposit (approve/เติมเครดิต):**
- `POST /DepositAddCredit/{service}` — `src/services/api-service/deposit/index.ts:350` (ใช้จากหน้า `src/views/transection/deposit/DepositView.vue`, `DepositTransection.vue`)
- `POST /deposit-add-credit-real-wallet/` — `deposit/index.ts:386`
- `POST /slip-verify-deposit-statement/` — `deposit/index.ts:405`; `POST /callback-deposit-statement/` — `deposit/index.ts:421`
- `POST /deposit-remain-move/` — `deposit/index.ts:314`; `/deposit-remain-hidden/` — `deposit/index.ts:332`
- list/summary: `/GetDepositStatement/`, `/GetLastDepositStatementV2/`, `/GetDepositStatementv2/`, `/GetDepositStatementP2P/`, `/GetDepositSummary(v2)/`, `/GetDepositStatementRemain(Today)/`, `/GetDepositStatementRealWallet(v2)/`, `/GetDepositStatementFloat(v2)/` — `deposit/index.ts:11-251`
- `POST /confirm-bank-deposit`, `/refund-deposit-to-confirm`, `/statement-deposit-not-confirm` — `src/services/api-service/transection/payment.ts:432-470`

**เงิน — withdraw (approve/cancel):**
- `POST /withdraw-confirm/` — `src/services/api-service/withdraw/index.ts:392`; `/withdraw-confirmauto/` — :411; `/withdraw-cancel/` — :429; `/withdraw-confirmmanual/` — :447; `/withdraw-confirmedit/` — :465 (เรียกจาก `src/components/transection/CardWithdrawDetail.vue:470-579`)
- `POST /return-withdraw-statement-credit/` — `withdraw/index.ts:501`; `/ManageWithdrawStatementRemain/` — :299; `/withdraw-hidden/` — :483
- `POST /confirm-withdraw-edit-status/` — :621; `/confirm-withdraw-edit-change-account-status/` — :639; `/withdraw_proxy_bank_job/` — :384; `/test-withdraw/` — :521; `/test-withdraw-truewallet/` — :538
- list: `/GetWithdrawStatement(v2/V3)/`, `/GetWithdrawSummary(v2)/`, `/GetWithdrawStatementRemain/`, `/GetWithdrawStatementLog/`, `/get_withdraw_statement_kpi/` — `withdraw/index.ts:11-209`

**เงิน — credit adjust (เพิ่ม/ตัด/คืนเครดิต):**
- `/credit-check/` — `src/services/api-service/credit/index.ts:9`; `/credit-return/` — :27; `/credit-return-wallet/` — :45; `/credit-delete/` — :63; `/credit-delete-wallet/` — :81; `/credit-freebonus/` — :99; `/credit-editturn/` — :117; `/credit-point/` — :135; `/free-ticket/` — :153; `/free-wallet/` — :171 (UI: `src/views/transection/ManageCreditView.vue`, `src/components/credit/AddCredit.vue:137`, `AddBonusFree.vue:178`, `AddEditTurn.vue:89`)
- `/clear-wallet-statement/` — `src/services/api-service/members/index.ts:1136`; `/clear-wallet-user/` — :1154; `/set-point/` — :401

**เงิน — payment gateway (3rd party):**
- `POST {service}/recharge-payment` — `src/services/api-service/transection/payment.ts:360`; `{service}/withdraw-payment` — :378; `/withdraw-payment/status` — :396; `/cancel-payout` — :414; `/payment-statement-submit` — :488; `/payment-slip-statement-submit` — :504; `/payment-neworder` — :520
- bank gateway / 3rd gateway connect: `/get-3rd-gateway/`, `/register-3rd-gateway/`, `/connect-3rd-gateway/`, `/disconnect-3rd-gateway/` — `src/services/api-service/bank/index.ts:1221-1286`; `/connect-bank-gateway/`, `/disconnect-bank-gateway/` — :1128-1147; `/callback-peer2pay` — :570

**member bank approve:** `/bank-member/approve/` — `members/index.ts:1249`; `/bank-member/reject/` — :1266; `/bank-member/approve-all/` — :1304; `/bank-member/reject-all/` — :1321; `/bank-member/check-approve/` — :1219

**auth/employee:** `/login-verify` — `employee/index.ts:360`; `/login-user-pass` — :378; `/two-factor` — :414; `/employees-2fa/` — :173; `/make-link-login` — :563; `/make-link-connect(-supercom)` — :576,592; `/register-platform` — :624; `/employees-update-permission` — :271; role: `/role-save`, `/role-update-permission`, `/employees-array-update-permission` — `system/role.ts:50,86,157`

**อื่น ๆ (ระบุ module):** bank config ~70 paths (`bank/index.ts`), statement (`/bankstatement-deposit/`, `/CreateStatement/`, `/expense-*` — `statement/index.ts`), report ~70 paths (`report/index.ts`), reportcredit (`/get-credit-list/` ฯลฯ), members ~80 paths, setting (bonus/cashback/commission/game/line/telegram/website), extension (coupon/wheel/sms/mission/ranking/ticket/tournament), verifyslip (`/create-verify-slip/` ฯลฯ), agent (`/agent/login-agent`, `/agent/wallet-*` — `agent/index.ts`), telesale, wallets, patch (`/check-patch/`, `/update-patch/`)

### data — token/session storage
- JWT เก็บใน **localStorage** key `auth_token` (JSON `{value, expiration}`) — `src/services/helper-function.ts:367-374` (set), `:391` (get); เซ็ตจาก `src/stores/auth.ts:29`
- axios interceptor อ่าน token จาก Pinia store หรือ localStorage แล้วใส่ `Authorization: Bearer` — `src/services/api-config.ts:17-25`
- tenant ที่เลือกเก็บใน localStorage `web-service` — `src/stores/layout.ts:50`
- Pinia persisted state (`pinia-plugin-persistedstate`) — `src/main.ts:91`
- response code 401/4011 → ลบ `auth_token` + redirect `/login` — `src/services/api-config.ts:44-57`
- ไม่พบการใช้ cookie เก็บ token (vue-auth3 fetchData/refresh ปิดอยู่ — `src/plugins/vue-auth3.ts:36-41`)

### external — third-party
- **Sentry** DSN hardcoded `https://93751f4a78...@o4506229598846976.ingest.sentry.io/4506308818698240` เปิดเฉพาะ non-dev พร้อม Replay — `src/main.ts:60-78`
- **Firebase โปรเจกต์ 1** `aba-group-new` + Realtime DB `https://withdraw-monitor-new.firebaseio.com` (apiKey hardcoded) — `src/plugins/firebase.ts:13-26`; ใช้เป็น withdraw monitor realtime — `src/components/sidebars/Sidebar.vue:322`, `src/layouts/mainLayout.vue:183,482`
- **Firebase โปรเจกต์ 2** `https://topup-group-default-rtdb.asia-southeast1.firebasedatabase.app` (apiKey hardcoded) — `src/plugins/firebase.ts:28-34`; ใช้เป็น remote kill-switch: ถ้า status==0 redirect ออกไป `https://google.com` — `src/layouts/mainLayout.vue:350-356`; และใน `src/views/DisableView.vue:130-137`, `src/components/headers/Navbar2.vue:840-847`
- Firebase URL per-tenant จาก config backend (`selectedWeb.firebase_url`) — `src/views/setting/NotifyListView.vue:252`
- **LiveChat** widget license `16918092` hardcoded — `src/layouts/mainLayout.vue:55-56,148`
- **S3** โลโก้เว็บ `https://rocketwinoffice.s3.ap-southeast-1.amazonaws.com/{service}/logoWeb/{service}` — `src/stores/layout.ts:37,54`, `src/components/headers/Navbar2.vue:332`
- icons8 CDN — `src/components/headers/Navbar2.vue:253,259`; ลิงก์ Google Authenticator (App Store) — `src/views/employee/EmployeeView.vue:467-468`

## observations
- **token ใน localStorage = เสี่ยง XSS** — JWT เก็บ plain ใน localStorage `auth_token` (`src/services/helper-function.ts:372`, `src/services/api-config.ts:22`) ไม่ใช่ httpOnly cookie
- **hardcoded secret ใน source:** `const secret = "B9NVVrFJDE0848efVhfrOBnxsI72hgat"` — `src/stores/auth.ts:25` 🟡 (ประกาศแต่ไม่พบการใช้ — แต่หลุดอยู่ใน bundle)
- **Firebase apiKey 2 ชุด hardcoded** — `src/plugins/firebase.ts:15,31`
- **Sentry DSN hardcoded** ใน source — `src/main.ts:64`; `tracePropagationTargets` ยังเป็น placeholder `yourserver.io` — `src/main.ts:69`
- **สิทธิ์ตรวจฝั่ง client เท่านั้น:** router guard อ่าน `result.level` จาก JWT ที่ decode เองใน browser (`level < 6` ห้ามเข้า `/dashboard`) — `src/router/index.ts:916-930`; ปุ่ม/หน้าใช้ `helper.checkPermission(code)` เทียบกับ `authStore.permission` ใน memory — `src/services/helper-function.ts:464-492`; โหมด dev มี `bypass` คืนสิทธิ์เต็มทุก action — `src/services/helper-function.ts:466-474` → **backend (office-api-v10) ต้อง re-check permission ทุก endpoint เงิน** เพราะ client แค่ซ่อน UI (เช่น เช็ค `checkCreditDelete.is_view` — `src/views/transection/ManageCreditView.vue:183,440`)
- **kill-switch ภายนอกควบคุมทั้งแอป:** ค่าใน Firebase RTDB (topup-group) status==0 จะ redirect ผู้ใช้ทุกคนไป google.com — `src/layouts/mainLayout.vue:350-356` (ใครคุม RTDB นี้ = ปิดเว็บ office ได้)
- **dev URL ภายในหลุดใน source ~25 รายการ** (ngrok, devtunnels, โดเมนลูกค้า troy/cowboy/maan/homey/esco/godfather789 ฯลฯ) comment ไว้ใน `src/services/base-url.ts:5-57` — information leak ของ infra/ลูกค้า
- **redirect login แนบ message จาก backend ลง URL** `/login?error=...&msg=...` — `src/services/api-config.ts:46` (reflected content จาก response)
- **dead code:** โฟลเดอร์ `src/services/api-service/setting/system copy/` ไม่ถูก import ที่ไหน (มี `/withdraw-config-update/` ซ้ำ — `setting/system copy/withdraw/withdraw.ts:26`); `src/views/_Homeold.vue`, `src/views/-TableView.vue`; `AUTHSERVICE` ใช้เฉพาะใน comment — `src/stores/auth.ts:118`
- **dependency เก่า/แปลก:** `xlsx ^0.17.0` (เวอร์ชันมีช่องโหว่ที่รู้จัก) — `package.json:69`; base image `node:19.9.0` (EOL) — `Dockerfile:1`; `eslint` อยู่ใน dependencies (`package.json:40`) ซ้ำกับ devDependencies; `npm`, `install`, `@types/axios` (stub) เป็น runtime deps — `package.json:33,46,52`
- **import ผิด:** `import jwt_encode from "jwt-decode"` — `src/views/agent/LoginView.vue:52` (encode จาก package decode)
- ไม่พบไฟล์ `.env` / การใช้ `import.meta.env.VITE_*` สำหรับ base URL — ทุก endpoint ภายนอกตัดสินจากโค้ด + origin ตอน runtime
