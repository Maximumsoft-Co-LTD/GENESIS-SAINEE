## Card: hydra-hyperx-web

identity: Nuxt 2 / Vue 2 / Vuetify 2 (SSR=false, target=static) | nuxt.config.js | dev port 8900 (nuxt.config.js:293) | Dockerfile present, docker-compose.yml present

provides:
  http:    (frontend SPA — no backend routes served by this process)

consumes:
  http-out:
    production: location.origin + "/api" (same-origin proxy, base path /api) | plugins/axios.js:4
    development: http://localhost:5053 (hardcoded) | plugins/axios.js:7
    fallback reference: https://aff-system.warmlight.online/ (commented out in axios.js, present in env.txt) | env.txt:1, plugins/axios.js:2

    API paths called (all relative to base URL):
      GET  /emp/department                    | plugins/controller/emp.js:25
      GET  /emp/employee                      | plugins/controller/emp.js:44
      PUT  /emp/employee                      | plugins/controller/emp.js:57
      POST /emp/employee                      | plugins/controller/emp.js:70
      GET  /emp/member (auth user fetch)      | nuxt.config.js:133
      POST /emp/login_username (auth login)   | nuxt.config.js:126
      POST /emp/logout (auth logout)          | nuxt.config.js:131
      GET  /v1/member                         | plugins/controller/v1.js:7
      POST /v1/tree_diagram                   | plugins/controller/v1.js:26
      GET  /v1/config_web                     | plugins/controller/v1.js:45
      POST /emp/* campaign/ads management routes | plugins/controller/emp.js (full campaign CRUD)
      POST /emp/dashboard, /emp/create_campaign, /emp/create_ads, etc. | plugins/controller/emp.js

observations:
  - No explicit BASE_URL env variable consumed — production base URL is derived from location.origin+"/api", making the frontend tightly coupled to an nginx/reverse-proxy that must expose the hydra-affilaite backend at /api | plugins/axios.js:4
  - Development baseURL is hardcoded as http://localhost:5053 — developers must run hydra-affilaite locally; no env override supported | plugins/axios.js:7
  - env.txt contains a live production URL "https://aff-system.warmlight.online/" — this file is likely checked into the repo and exposes the production hostname; should be .gitignored | env.txt:1
  - Auth token type is "Token" (custom scheme) not "Bearer" per nuxt.config.js:137 — backend middleware.AuthJWT must match this; misconfiguration would silently break auth
  - @nuxtjs/auth-next strategy stores JWT in localStorage (default for local scheme) — XSS risk; no httpOnly cookie fallback configured | nuxt.config.js:101-197
  - Nuxt 2 is EOL (June 2024) — security patches no longer provided; migration to Nuxt 3 needed
  - No i18n module configured — Thai-language UI text is embedded directly in components
  - No API timeout configured in axios plugin — all requests use browser default (no timeout) | plugins/axios.js (entire file)
  - Package.json includes "child_process" as a runtime dependency — this is a Node.js built-in and installing it as a package is a no-op but indicates potential misunderstanding | package.json:34
  - nuxt.config.js:100 sets axios: {} (empty config) — no baseURL, no timeout, no retry configured at module level; all config delegated to plugin
  - 🟡 env.txt is not a standard .env file and does not appear to be loaded by any Nuxt config — its purpose is unclear; may be a documentation artifact or leftover deploy artifact
