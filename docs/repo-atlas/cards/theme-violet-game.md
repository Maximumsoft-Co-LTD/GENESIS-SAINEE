# Card: theme-violet-game
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- stack: **Vite 4 + Vue 3.3 + TypeScript** (ไม่ใช่ Nuxt — vite.config.ts:11, package.json:79) + Pinia + vue-router 4 + Vuetify 3 + Tailwind + vue-auth3 | package name: `ngernn` (package.json:2)
- mode: **SPA** (CSR ล้วน, nginx `try_files ... /index.html` — .nginx/nginx.conf:18-20) + SEO injection หลัง build ด้วย shell script (public/post-build-script.sh)
- port: dev 6900 (vite.config.ts:118-120) | nginx listen 6900, EXPOSE 6900 (.nginx/nginx.conf:2, Dockerfile:30)
- Dockerfile: 2-stage — node:19.9 build → nginx:1.23-alpine + ติดตั้ง jq/curl/bash/imagemagick + **cron ทุกชั่วโมง** รัน post-build-script.sh (Dockerfile:3,15,19-24) | CMD `./main-script.sh` = รัน post-build → crond → nginx (public/main-script.sh:4-6) | ไม่พบ .github/workflows
- มี build variant `gamelotto` (base `/gamelotto/` — package.json:8-12)
- ตัวตนระบบ: frontend ลูกค้าเว็บหวย/เกม "Ngernn Lottery" (console.log 'Lotto version 1.2.6' — src/main.ts:94)

## provides
### http — frontend pages
- SPA views 41 ไฟล์ .vue ใต้ `src/views/` (นับจริง), router มี ~63 path entries (src/router/index.ts) — หน้า login, register, home, lotto bet/result/preset, deposit (auto/qrpay/truewallet/peer2pay/decimal/7eleven/slipverify — src/config.json:160-249), withdraw, affiliate, ranking, minigame, history, profile
- sitemap.xml + robots.txt generate ตอน build (`scripts/generate-sitemap.ts` — package.json:7) แล้วถูก sed แทน `{{hostname}}` ตอน start container (post-build-script.sh:66-72)

### consume / cron
- **cron ใน container**: `0 * * * *` รัน `/usr/share/nginx/html/post-build-script.sh` ทุกชั่วโมง (Dockerfile:24) → curl `$DOMAIN/api/prefix` แล้ว sed อัด SEO meta/GTM ลง index.html (post-build-script.sh:35-47,242)

## consumes

### http-out

**Base URL (src/url_config.ts:1-33, ใช้โดย axios instance เดียวใน src/APIService.ts:6-11):**
| ค่า | เงื่อนไข | สถานะ |
|---|---|---|
| `location.origin + '/api'` | production | ใช้จริง (url_config.ts:27,31) — ผูกโดเมน deploy, ต้องมี reverse proxy `/api` (ไม่อยู่ใน nginx.conf ของ repo — .nginx/nginx.conf ไม่มี location /api) |
| `https://violetgame-homey-dev.thesonicblue.xyz/` | development | ใช้จริงตอน dev (url_config.ts:9) |
| `https://punpro-dev.uppicture.online`, `https://tangtem168.com/`, `https://game.no1huay.com`, `https://violet-dev-dev.uppicture.online/`, `https://ufa-dev.uppicture.online/`, `https://9c41-18-141-205-156.ngrok-free.app`, `https://lucky.ngernn.co/`, `https://scb-lotto.thesonicblue.xyz`, `https://lotto-seamless-api.hammerstone.xyz/`, `https://lotto-seamless-api-staging.hammerstone.xyz/` | — | 🟡 comment ไว้ (url_config.ts:5-15,32) |
| `BASE_URL = ''` (เคยเป็น `https://lotto-seamless-api.hammerstone.xyz/api`) | seamless mode | 🟡 ค่าเดิม comment, ปัจจุบัน prefix ว่าง = base เดียวกัน (src/services/LottoService.ts:67-68, src/services/UserService.ts:3-4) |

ทุก request ถูกต่อท้าย `?resp=original` อัตโนมัติ (src/APIService.ts:83-89) และแนบ header `Authorization: <token>` (src/APIService.ts:69-72)

**Auth (vue-auth3 — src/plugins/vue-auth3.ts):**
- `POST {baseURL}/login` หรือ `{baseURL}/lotto/login` (ตาม `login_mode` ใน config.json — vue-auth3.ts:32-34)
- `POST {baseURL}/is_login` / `{baseURL}/lotto/is_login` (fetch user + refresh — vue-auth3.ts:36-53; ปิดเมื่อ `lotto.mode === 'seamless'` — vue-auth3.ts:43)
- `POST {baseURL}/logout` (vue-auth3.ts:55-59)

**API paths (src/services/*.ts ผ่าน apiService):**
- Config/Prefix: `GET /prefix` (ConfigService.ts:6), `GET /article` (ConfigService.ts:26), `POST /bank_code_list` (ConfigService.ts:16)
- Register/Login: `POST /register_v3` (RegisterService.ts:6), `POST /lotto/register` (RegisterService.ts:16), `POST /get/otp` (RegisterService.ts:48,62), `POST /check-username` (RegisterService.ts:93), `POST /member-token` (RegisterService.ts:104), `POST /login_line` (LoginService.ts:7), `POST /forget_password` (LoginService.ts:17), `POST /forget-password` (LoginService.ts:27), `POST /forget-password-verify-otp` (LoginService.ts:40), `POST /forget-password-bank` (LoginService.ts:53)
- User/Wallet: `POST /amount` (UserService.ts:13-17), `POST /member` (UserService.ts:27), `POST /reset_password` (UserService.ts:55), `POST /get/wallet` (WalletService.ts:13), `POST /get/turnover` (WithdrawService.ts:6), `POST /withdraw` (WithdrawService.ts:22), `POST /get/upLevel` (RankingService.ts:16), `POST /rankingInfo` (RankingService.ts:6)
- Deposit: `POST /bank/auto` (DepositService.ts:10), `POST /bank/truewallet` (DepositService.ts:19), `POST /decimal` (DepositService.ts:28), `POST /history_qrpay_v2` (DepositService.ts:127), `POST /peer2pay_pay` (DepositService.ts:155)
- Bonus/Coupon/Affiliate: `POST /bonus_free` + `/get/bonus_free` (BonusFreeService.ts:12,29), `POST /get/coupon` (CouponService.ts:10), `POST /affiliate` (AffiliateService.ts:20), `POST /creditfree_affiliate` + `/get/creditfree_affiliate` (AffiliateService.ts:82,96)
- Game: `GET /game/category_game` (GameService.ts:7), `GET /game/favorite_game` (GameService.ts:17), `GET game/list_gametag` (GameService.ts:28 — ไม่มี `/` นำหน้า), `GET /game/filter_game?...` (GameService.ts:40), `GET /game/accessGame/${game}/${gameid}?...` (GameService.ts:54), `POST /game/favorite_update` (GameService.ts:65), `GET /game/favorite_brand` (GameService.ts:76), `GET /game/history_game` (HistoryService.ts:41), `POST /history` (HistoryService.ts:6)
- Lotto: `GET /game/group_lotto` (LottoService.ts:73,82), `POST /game/history_lotto` (LottoService.ts:132), `GET /game/reward_lotto` (LottoService.ts:140,418), `GET /game/yiki_number?round=` (LottoService.ts:297), `POST /game/prebet_lotto` (LottoService.ts:362), `POST /game/bet_lotto` (LottoService.ts:394,398), `GET /game/list_check_lotto` (LottoService.ts:408), `GET/POST/DELETE /game/list_number` (PresetService.ts:6,16,26)
- Seamless/MiniGame: `GET /login?key=&username=` (SeamlessService.ts:6), `POST /login_minigame` (MiniGameService.ts:6)
- จาก container (ไม่ใช่ browser): `GET ${DOMAIN}/api/prefix` ทุกชั่วโมง (post-build-script.sh:45-47, Dockerfile:24) — `DOMAIN` เป็น env runtime 🟡 inject ตอน deploy

### data — localStorage/cookie
- **token เก็บใน cookie อายุ 1 ปี** โดย vue-auth3 (`cookie: { expires: 365 วัน }` — vue-auth3.ts:30)
- **token ใน localStorage**: `localStorage.getItem('token')` ใช้ตั้ง Authorization (APIService.ts:46), seamless mode เซฟ `setItem('token', value)` (src/stores/seamless.ts:43)
- `regis-token` (token สมัครสมาชิกจาก query string) ใน localStorage (src/App.vue:247,251; src/views/RegisterView.vue:119,126)
- อื่น ๆ: `hash` (md5 ของ URLDev — url_config.ts:17-21), `hid` (hydra affiliate id จาก query — RegisterView.vue:135, LoginView.vue:335), `username`, `lotto.mode`, `lotto.route_name`, `auth_remember`, `line_user_login`, `refresh`, `dataFavorite`, `groupLotto`, `gameType` (grep ทั่ว src)

### external
- **Sentry**: DSN hardcode `https://c567f9bba923118cced834cdbd051fbc@o4506229598846976.ingest.sentry.io/4506241643446272` + BrowserTracing + Session Replay (replaysSessionSampleRate 0.1, onError 1.0) (src/main.ts:55-71)
- **Firebase Realtime Database**: config hardcode โปรเจกต์ `ngernn-e1901` (apiKey `AIzaSyDtMOPoO4B24fVkl3_Fh4XDbkA21yHyctU` — src/stores/firebase.ts:19-27); databaseURL เลือก runtime: dev/localhost → `https://officex-homey-dev.asia-southeast1.firebasedatabase.app/`, prod → `https://${firebasePrefix}.asia-southeast1.firebasedatabase.app/` (prefix มาจาก API — firebase.ts:37-44)
- **LiveChat widget** (`@livechat/widget-vue`): license_id มาจาก `/prefix` API runtime (src/App.vue:182-184,650-651)
- **Google Tag Manager**: index.html มี GTM snippet กับ placeholder `{{googletag}}` 🟡 ถูก sed แทนค่าจริงจาก `/api/prefix` ตอน container start/cron (index.html:48-60, post-build-script.sh:196-220)
- **Image CDN**: `https://self-imagex.image-etc.co` hardcode เป็น global image base (src/main.ts:73-74); ภาพ deposit จาก `https://d3tyje694b3tw3.cloudfront.net/...` (src/views/deposit/TruewalletGiftView.vue:156,162)
- **LINE OAuth** ผ่าน `POST /login_line` (LoginService.ts:7) + `line_user_login` ใน localStorage

## observations
1. token เก็บทั้ง cookie 1 ปี (vue-auth3.ts:30) และ localStorage (APIService.ts:46, stores/seamless.ts:43) — localStorage อ่านได้จาก XSS; อายุ cookie 1 ปียาวผิดปกติสำหรับเว็บการเงิน/พนัน
2. Sentry DSN + Session Replay hardcode ในโค้ด client (src/main.ts:55-71) — replay บันทึก session ผู้ใช้เว็บพนัน ส่งขึ้น sentry.io org กลาง; `tracePropagationTargets` ยังเป็นค่า template `yourserver.io` (main.ts:61)
3. Firebase apiKey/projectId hardcode (stores/firebase.ts:19-27) และเลือก databaseURL จากการเช็ค `location.origin.includes('violet-dev-dev'|'violet-uat-dev'|'localhost')` (firebase.ts:38-44) — สภาพแวดล้อมถูกฝังในโค้ด ไม่ใช่ env
4. SEO/GTM injection ทำด้วย `sed` แก้ index.html ใน container ทุกชั่วโมงผ่าน cron (Dockerfile:24, post-build-script.sh) — ถ้า `/api/prefix` ล่มตอน start, script `exit 1` ทำให้ container ตาย (post-build-script.sh:45-58 + main-script.sh:4 รันก่อน nginx)
5. base URL dev จำนวนมาก comment สลับมือใน url_config.ts:5-15 (มี ngrok URL ชั่วคราว, โดเมนลูกค้าหลายเจ้า: tangtem168.com, no1huay.com, ngernn.co, hammerstone.xyz) — ร่องรอย multi-tenant theme เดียว deploy หลายเว็บ
6. `hashLocal = Md5.hashStr(URLDev)` ถูกเขียนลง localStorage ตั้งแต่ import module (side effect ระดับ top-level — url_config.ts:17-21) แม้ใน production ที่ไม่ใช้ URLDev
7. seamless mode รับ token/username ผ่าน query string (`GET /login?key=&username=` — SeamlessService.ts:6) และ `regis-token` รับจาก `route.query.token` (App.vue:251) — credential ใน URL ติด log/history ได้
8. behavior ฝัง hostname เฉพาะลูกค้าในโค้ด view: เช็ค `violet-dev-dev.uppicture.online` / `violet-uat-dev.uppicture.online` ซ้ำหลายจุดใน src/views/LottoListV3View.vue:386-424, LottoResultView.vue:109
9. nginx.conf ใน repo ไม่มี `location /api` (.nginx/nginx.conf) แต่โค้ด client เรียก `location.origin + '/api'` (url_config.ts:31) — reverse proxy `/api` ต้องอยู่ที่ infra ชั้นนอก (ingress) ไม่เห็นใน repo
10. GameService.ts:28 path `'game/list_gametag'` ไม่มี `/` นำหน้า — เป็น relative ต่อ URL ปัจจุบันของ axios baseURL (อาจกลายเป็น `/api/game/list_gametag` ถูกต้องโดยบังเอิญเพราะ baseURL ลงท้าย `/api`)
11. ไม่พบ .env / .github workflows; image registry: `thezeustech.jfrog.io/docker-local/theme-violet-game:dev` (docker-compose.yml:8) และ harbor proxy comment ไว้ใน Dockerfile:2,14 🟡
12. `console.log('Lotto version 1.2.6')` + sitemap log ใน production bundle (main.ts:94-99)
