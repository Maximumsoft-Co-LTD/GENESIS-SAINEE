# Card: hydra-hyperx-web
> วิเคราะห์: 2026-06-12 | commit: 40367af

## identity
- stack: Nuxt 2 (`nuxt@^2.15.8` — package.json:39) + Vue 2.6 + Vuetify 2 + @nuxtjs/axios + @nuxtjs/auth-next | package name: `hydra-system` (package.json:2)
- mode: **SPA** (`ssr: false`, `target: 'static'` — nuxt.config.js:6-7) — build ด้วย `nuxt generate` แล้วเสิร์ฟไฟล์ static
- port: dev server 8900 (nuxt.config.js:292-295) | Docker เสิร์ฟด้วย `http-server` port 6913 (Dockerfile:4,21-23) | docker-compose map `6913:8900` (docker-compose.yml:13-14)
- Dockerfile: node:16.19-alpine, `npm install --legacy-peer-deps`, `npm run generate`, CMD `http-server --cors -p6913 dist` (Dockerfile:2,9,18,23) | ไม่พบ `.github/workflows` ใน repo นี้
- ตัวตนระบบ: หน้า back-office/affiliate ของระบบ "Hydra" (titleTemplate `%s | Hydra` — nuxt.config.js:10) สำหรับจัดการ campaign/ads/affiliate

## provides
### http — frontend pages
- SPA pages 80 ไฟล์ .vue ใต้ `pages/` (นับจริง; บางไฟล์ขึ้นต้น `-` = ปิดใช้งานจาก routing เช่น `pages/-emp/-login.vue`) | route หลัก: /login, /register, /campaign/**, /settings/**, /setupDomain/** | catch-all `*` → pages/404.vue (nuxt.config.js:278-284)
- ไม่มี server middleware / API ฝั่ง server (SPA ล้วน — nuxt.config.js:6)

### consume / cron
- ไม่พบ (ไม่มี cron/queue ใน repo)

## consumes

### http-out — backend API เดียว (base URL เดียว ทุก path วิ่งผ่าน @nuxtjs/axios)

**Base URL (plugins/axios.js:1-11):**
| ค่า | เงื่อนไข | สถานะ |
|---|---|---|
| `location.origin + '/api'` | production (NODE_ENV != development) | ใช้จริง — base URL ผูกกับโดเมนที่ deploy (same-origin reverse proxy `/api`) |
| `http://localhost:5053` | development | ใช้จริงตอน dev (plugins/axios.js:6) → ชี้ backend affiliate (พอร์ต 5053) |
| `https://aff-system.warmlight.online` | — | 🟡 comment ไว้ (plugins/axios.js:2) แต่ยังถูก set เป็น env `baseURL` ใน docker-compose.yml:10 และ env.txt:1 (โค้ดจริงไม่ได้อ่าน env นี้ — nuxt-env module ถูก comment ที่ nuxt.config.js:78-96) |
| `https://demo-uat.hydra-affiliate.com/api`, `https://demo-dev.hydra-affiliate.com/api`, `https://demo-dev-hydra-cowboy.thesonicblue.xyz/api`, `http://192.168.1.54:5053` | — | 🟡 comment ไว้ (plugins/axios.js:7-10) |

**Auth endpoints (@nuxtjs/auth-next, nuxt.config.js:112-197):**
- `POST /emp/login_username` (login strategy `local` — nuxt.config.js:124-127)
- `POST /emp/login` (strategy `local_line` — nuxt.config.js:152-154)
- `POST /emp/register` (strategy `register_line` — nuxt.config.js:180-182)
- `POST /emp/logout`, `GET /emp/member` (nuxt.config.js:128-135) — token header `Authorization: Token <jwt>` (nuxt.config.js:136-138)

**`/emp/*` (plugins/controller/emp.js — injected เป็น `$emp_*`):**
`GET/PUT/POST/DELETE /emp/employee` (emp.js:44-103), `/emp/sub_employee` (emp.js:111+), `/emp/department` (emp.js:21-37), `/emp/employee_active`, `/emp/platform`, `/emp/platform/all`, `/emp/agent_type`, `/emp/dashboard` (emp.js:446-470), `/emp/dashboard_report`, `/emp/summary_dashboard_report`, `/emp/dashboard${id}`, `/emp/member_affiliate`, `/emp/member_statement_affiliate`, `/emp/member_log`, `/emp/link_affiliate`, `/emp/template` (emp.js:1464-1525), `/emp/template/${id}`, `/emp/template_link`, `/emp/template_link_campaign${query}`, `/emp/landing_page`, `/emp/banner`, `/emp/list_banner`, `/emp/set_banner`, `/emp/article${params}`, `/emp/article${params}/upload`, `/emp/create_campaign`, `/emp/create_campaign_many`, `/emp/edit_campaign`, `/emp/edit_campaign_status`, `/emp/finish_campaign`, `/emp/create_ads`, `/emp/create_ads_many`, `/emp/create_ads_batch`, `/emp/edit_ads`, `/emp/edit_ads_status`, `/emp/create_cost_ads`, `/emp/history_cost_ads/${cid}/${id}`, `/emp/ads_collection` (emp.js:896), `/emp/ads_collection/summary/${campaignId}`, `/emp/ads_collection/export_excel`, `/emp/ads_collection/create_index`, `/emp/setting_role`, `/emp/setting_role/find/${id}`, `/emp/setting_role/${code}`, `/emp/setting_reward`, `/emp/setting_level_reward`, `/emp/sale`, `/emp/list_menu`, `/emp/web_service/${action}`, `/emp/domains`, `/emp/domains/${id}`, `/emp/domains/service/${service}`, `/emp/domains/${service}/${id}`, `/emp/campaigns/${service}/${campaignID}/domains`, `/emp/campaigns/${service}/${campaignID}/domains/use`, `/emp/campaigns/${service}/${campaignID}/domains/${domainID}/use`, `/emp/login_line` (emp.js:1970 — ดึง LINE client_id), `/emp/register_line` (emp.js:2125), `/emp/password`, `/emp/check_demo`, `/emp/highest_deposit`, `/emp/re_hydra_statement`, `/emp/re_hydra_campaign_statement`, `/emp/re_hydra_campaign_statement/status/${campaignId}`, `/emp/re_hydra_campaign_statement/reset_then_rehydra`

**`/ctm/*` (plugins/controller/ctm.js):**
`/ctm/dashboard` (ctm.js:9-33), `/ctm/summary_income` (ctm.js:54-78), `/ctm/all_member` (ctm.js:99-123), `/ctm/tree_member_list` (ctm.js:144-168), `/ctm/web_service` + `/ctm/web_service/${action}` (ctm.js:189-213), `/ctm/affiliate_detail` + `/ctm/affiliate_detail/${action}` (ctm.js:234-258)

**`/v1/*` (plugins/controller/v1.js):**
`/v1/member` (v1.js:8), `/v1/tree_diagram` (v1.js:29), `/v1/config_web` (v1.js:50), `/v1/reset_statement` (v1.js:70), `/v1/reset_statement_campaign` (v1.js:91)

**`/landing/*` (plugins/controller/summary.js):**
`/landing/summary_report` (summary.js:9-33), `/landing/summary_report/${id}` (summary.js:45)

**เรียก $axios ตรงจาก pages:** `POST/PUT /emp/template` (pages/setupDomain/_id/index.vue:178-196)

**Dead/disabled:** `url: '/user/12345'` ใน pages ที่ขึ้นต้น `-` (ปิดใช้งาน) 🟡 (pages/-income/-TableExpenses.vue:156, -TableIncome.vue:136, -TableMember.vue:164, pages/-reports/-income/-index.vue:162)

### data — token / sensitive ฝั่ง client
- JWT token เก็บผ่าน @nuxtjs/auth-next `$auth.$storage` (universal = cookie + localStorage ค่า default ของ auth-next; config ไม่ได้ปิด localStorage — nuxt.config.js:101-198, plugins/auth.js:22-39)
- `localStorage`: `selectedGroup`, `selectedService` (components/GroupServiceSelect/index.vue:149-150,227-228; components/ServiceControl/index.vue:165-215), `register` flag (pages/login.vue:151)
- ทุก request แนบ header `Authorization: <token>` + JSON (plugins/axios.js:13-26); 401 → redirect `/logout` (plugins/axios.js:28-32)

### external
- **LINE Login OAuth**: redirect ไป `https://access.line.me/oauth2/v2.1/authorize?...client_id=<จาก /emp/login_line>` (pages/login.vue:227-236, pages/register.vue:78, components/AppBar.vue:471)
- ไม่พบ analytics/GTM/CDN script อื่นใน nuxt.config.js head (nuxt.config.js:8-22)

## observations
1. base URL production ผูกกับ `location.origin + '/api'` (plugins/axios.js:4) — ต้องมี reverse proxy `/api` หน้าเว็บเสมอ แต่ไม่พบ nginx config ใน repo (Dockerfile ใช้ http-server เสิร์ฟ static เท่านั้น — Dockerfile:23) → ตัว proxy อยู่นอก repo (infra)
2. env `baseURL=https://aff-system.warmlight.online/` ตั้งใน docker-compose.yml:10 และ env.txt:1 แต่**โค้ดไม่ได้อ่านค่า** (module nuxt-env ถูก comment — nuxt.config.js:78-96) = dead config 🟡
3. behavior ของ UI ถูก hardcode ผูกกับ hostname เฉพาะ: `https://demo-uat.hydra-affiliate.com` / `http://192.168.1.61:8900` ปิด input login (pages/login.vue:141-146), เช็คซ้ำใน pages/settings/emp/new/index.vue:564 และ pages/campaign/_group/_service/_cid/advertise/_id.vue:2520-2521
4. โดเมน demo hardcode ใน option list (มีตัวสะกดผิด `กthesonicblue3.xyz`): pages/campaign/edit/_group/_service/_id.vue:499-503
5. token JWT เก็บใน localStorage/cookie โดย auth-next (เสี่ยง XSS อ่าน token ได้) — nuxt.config.js:113-139; ส่ง ipcon: '0.0.0.0' hardcode ใน login payload (pages/login.vue:152)
6. dependencies เก่ามาก: nuxt 2.15.8 (EOL), webpack 4 (package.json:68), node 16 (Dockerfile:2), `jsonwebtoken@^8.5.1` ติดตั้งฝั่ง client แต่ไม่พบการ import ในโค้ด 🟡 (package.json:36), `child_process` ตีพิมพ์เป็น npm dep ฝั่ง frontend (package.json:30) — น่าสงสัย/ไร้ประโยชน์
7. plugins/auth.js import แปลก ๆ ที่ไม่ใช้งานจริง: `import { tr } from 'date-fns/locale'`, `import { f } from 'dropzone'` (plugins/auth.js:1-2) — โค้ดรกและบ่งชี้ไม่มี lint บังคับ
8. การเช็คสิทธิ์เมนู (`$auth.accessPath`) **ปิดใน development** คืน true เสมอ (plugins/auth.js:50-58) และ middleware accessPath ใช้ `_` (lodash global จาก webpack ProvidePlugin — nuxt.config.js:256-261) ใน middleware/accessPath.js:6 — พึ่ง global implicit
9. schemes/customScheme.js มีอยู่แต่ไม่ถูก reference ใน nuxt.config.js 🟡 (auth ใช้ scheme 'local' ล้วน — nuxt.config.js:113,142,170)
10. ไม่พบ CI workflow (.github/) ใน repo — build ผ่าน docker-compose / docker-compose-jfog.yml (image `thezeusthech/hydra-hyperx-web:uat` — docker-compose.yml:6)
