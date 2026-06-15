## Card: theme-violet-game

identity: Vue 3 + Vite 4 + TypeScript | src/main.ts | dev port 6900 (vite.config.ts:119) | Dockerfile present, docker-compose.yml present, no CI workflow found

provides:
  http: (SPA — no server-side HTTP routes, all routing is client-side Vue Router)
  consume: (none — frontend only)
  cron: (none)

consumes:
  http-out:
    baseURL = `${location.origin}/api` in production | resolved at runtime from window.location.origin | src/url_config.ts:31
    DEV override: hardcoded URLDev = 'https://violetgame-homey-dev.thesonicblue.xyz/' | src/url_config.ts:9
    All API calls via axios through ApiService class | src/APIService.ts:1-95
    Authorization header set as raw token string: `${token}` (no "Bearer " prefix) | src/APIService.ts:71
    Query param `resp=original` appended to every request via ConvertApexResponse | src/APIService.ts:83-88

  data:
    localStorage key "token" — JWT auth token stored in browser | src/APIService.ts:47
    localStorage key "hash" — MD5 hash of base URL for tenant detection | src/url_config.ts:19-21

  external:
    Sentry (@sentry/vue ^7.73.0) — error tracking | package.json:22
    Firebase (firebase ^10.0.0) — likely push notifications or auth | package.json:26
    LiveChat (@livechat/widget-vue ^1.3.3) — live chat widget | package.json:18

observations:
  - CRITICAL: DEV base URL 'https://violetgame-homey-dev.thesonicblue.xyz/' hardcoded in source — will be compiled into dev builds and visible in browser | src/url_config.ts:9
  - Multiple commented-out URLDev values expose internal hostnames/staging URLs in source code: uppicture.online, thesonicblue.xyz, hammerstone.xyz, ngrok URL, ngernn.co | src/url_config.ts:4-16
  - Authorization header set without "Bearer " prefix (`${token}` not `Bearer ${token}`) — may cause auth failures or inconsistency with backends expecting standard Bearer scheme | src/APIService.ts:71
  - MD5 hash of base URL stored in localStorage as tenant identifier — MD5 is not cryptographically secure and collision-trivial for known URL set | src/url_config.ts:17-21
  - No CSRF protection — SPA with axios, no anti-CSRF token management visible
  - Firebase dependency included but no Firebase config file (.env.firebase) found in repo — secrets likely in env at build time 🟡
  - Using deprecated dgrijalva/jwt-go pattern is upstream (backend); frontend only stores raw token, no client-side JWT validation
  - No .env.example found — API key / Firebase config requirements not documented 🟡
  - package.json name is "ngernn" — possible brand name leak / mismatch with "theme-violet-game" service name
  - Old vite@4 with @vitejs/plugin-legacy — EOL Vite 4 (Vite 5+ released); security patches no longer backported | package.json:79
