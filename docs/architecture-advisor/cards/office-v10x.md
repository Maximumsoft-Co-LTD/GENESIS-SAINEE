## Card: office-v10x

identity: Vue 3, Vite 4, TypeScript, Pinia, Axios, Tailwind CSS | src/main.ts | dev port 8088 (vite.config.ts:13) | Dockerfile present, docker-compose-jfog.yml present

provides:
  http:    (SPA — no server routes; all calls go to office-api-v10)
  consume: n/a
  cron:    n/a

consumes:
  http-out:
    HOSTNAME = location.origin/api (prod) or "http://localhost:7777/api" (dev, hardcoded) — all API calls via axios instance | src/services/base-url.ts:27,59
    POST {HOSTNAME}/login-verify — authentication | src/plugins/vue-auth3.ts:33
    GET/POST/PUT/PATCH/DELETE {HOSTNAME}/api/* — full back-office operations via api-service/* modules | src/services/api-config.ts:10-12
    AUTHSERVICE = "https://oauth.pinkmonkey.online" (hardcoded) — OAuth/SSO login link generation | src/services/base-url.ts:62
    Firebase Realtime DB "https://withdraw-monitor-new.firebaseio.com" (project: aba-group-new) — withdraw monitoring | src/plugins/firebase.ts:17
    Firebase Realtime DB "https://topup-group-default-rtdb.asia-southeast1.firebasedatabase.app" (project topup) — top-up monitoring | src/plugins/firebase.ts:29-33
    Sentry ingest "https://93751f4a7813d7947b00cf11337e8f96@o4506229598846976.ingest.sentry.io/4506308818698240" — error tracking | src/main.ts:64
    LINE OAuth "https://access.line.me/oauth2/v2.1/authorize" — LINE login flow | src/views/login.vue:700

  data:
    localStorage "auth_token" — JWT token storage (read/write) | src/services/api-config.ts:21-22, src/stores/auth.ts:29
    localStorage "web-service" — selected service key | src/services/web-service.ts:4-7
    Pinia stores (in-memory + pinia-plugin-persistedstate for select stores) | src/main.ts:91

  external:
    Firebase (two projects: aba-group-new, topup-group) | src/plugins/firebase.ts
    Sentry (@sentry/vue 7.83.0) | src/main.ts:62-78
    LINE OAuth login | src/views/login.vue:700
    oauth.pinkmonkey.online (SSO service) | src/services/base-url.ts:62

observations:
  - HARDCODED backend URL "http://localhost:7777" committed as the active dev URL — developer can accidentally point at local instead of UAT/prod instance, and any builds in "development" mode will hit localhost | src/services/base-url.ts:27
  - NO VITE_* environment variables — backend base URL is compile-time hardcoded string, not configurable via .env files; deployment requires code change to switch environments | src/services/base-url.ts:59
  - HARDCODED JWT signing secret in client code: secret = "B9NVVrFJDE0848efVhfrOBnxsI72hgat" stored in Pinia auth store (appears unused but still committed) | src/stores/auth.ts:25
  - HARDCODED Firebase API keys and app IDs committed to source (two Firebase projects): apiKey "AIzaSyA3uo71Yi3KlvfzrR0QoB7qhRRu1GxxVLM" (aba-group-new) and apiKey "AIzaSyDe6izf8KfoNm_dgZemMyVVj5lJ1TVxKVo" (topup-group) | src/plugins/firebase.ts:14-33
  - HARDCODED Sentry DSN committed to source — leaks org/project IDs | src/main.ts:64
  - HARDCODED OAuth service URL "https://oauth.pinkmonkey.online" — not configurable per environment | src/services/base-url.ts:62
  - Token stored in localStorage (not httpOnly cookie) — vulnerable to XSS token theft; 401/4011 response handler redirects with raw error message in URL query param | src/services/api-config.ts:41-49
  - axios interceptor swallows non-401 errors: Promise.reject(err) only after 401 redirect — other network errors may silently fail in some consumers | src/services/api-config.ts:46-53
  - No VITE_SENTRY_DSN, no VITE_FIREBASE_*, no VITE_API_URL — entire external service config is in committed source files, not .env
  - src/stores/auth.ts imports AUTHSERVICE from base-url.ts but it is only used in a commented-out line (GenUrlAuthServiceCallback) — dead import | src/stores/auth.ts:8,118
  - dev server port 8088 (vite.config.ts:13) differs from common port 3000 — may cause CORS issues with backend when testing locally
  - dgrijalva/jwt-go equivalent concern: frontend uses jwt-decode + jwt-encode libraries; jwt-encode v1.0.1 (package.json:50) is a minimal library — validation logic must be confirmed on consuming side
  - Cypress e2e tests present (cypress/ dir) but no CI workflow visible in this repo — test coverage for admin UI unclear | cypress.config.ts
  - No Content-Security-Policy headers configured in Vite — Firebase SDK and external scripts load without CSP guard
