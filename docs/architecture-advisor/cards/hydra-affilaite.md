## Card: hydra-affilaite

identity: Go 1.16 / Gin | _cmd/main.go -> hydra.StartServer() | port 5053 (server.go:32) | Dockerfile present, docker-compose.yml present

provides:
  http:
    GET  /v1                                        | no auth | route.go:18
    GET  /v1/get_demo                               | no auth | route.go:19
    POST /v1/upload_demo                            | no auth | route.go:20
    DELETE /v1/delete_distribute                    | no auth | route.go:23
    POST /v1/get_distribute                         | no auth | route.go:24
    PUT  /v1/disable_distribute                     | no auth | route.go:25
    POST /v1/dashboard/:campaign_id/:ads_id         | no auth | route.go:27
    POST /v1/raw_data                               | no auth | route.go:28
    POST /v1/raw_data_ads                           | no auth | route.go:29
    POST /v1/turnover_data                          | no auth | route.go:30
    POST /v1/campaign                               | no auth | route.go:31
    GET  /v1/get_withdraw                           | no auth | route.go:33
    POST /v1/re_withdraw                            | no auth | route.go:34
    POST /v1/reset_statement                        | no auth | route.go:37
    POST /v1/reset_statement_campaign               | no auth | route.go:38
    GET  /landing/prefix/:service                   | no auth | route.go:43
    POST /landing/summary_report/:key               | no auth | route.go:44
    POST /emp/login                                 | no auth | route.go:51
    POST /emp/login_username                        | no auth | route.go:52
    POST /emp/register                              | no auth | route.go:53
    POST /emp/register_line                         | AuthJWT(1) | route.go:54
    POST /emp/password                              | AuthJWT(1) | route.go:54 (separate)
    GET  /emp/member                                | AuthJWT(1) | route.go:55
    POST /emp/logout                                | no auth | route.go:56
    POST /emp/member_affiliate                      | no auth | route.go:59
    POST /emp/member_statement_affiliate            | no auth | route.go:60
    POST /emp/dashboard                             | AuthJWT(3) | route.go:63
    PUT  /emp/dashboard                             | AuthJWT(3) | route.go:64
    POST /emp/dashboard/:campaign_id                | AuthJWT(3) | route.go:65
    POST /emp/dashboard/:campaign_id/:ads_id        | AuthJWT(3) | route.go:67
    POST /emp/dashboard_report/:campaign_id/:ads_id | AuthJWT(3) | route.go:68
    POST /emp/finish_campaign                       | AuthJWT(3) | route.go:70
    POST /emp/history_cost_ads/:campaign_id/:ads_id | AuthJWT(3) | route.go:71
    POST /emp/create_cost_ads                       | AuthJWT(3) | route.go:72
    POST /emp/create_campaign                       | AuthJWT(3) | route.go:74
    POST /emp/create_campaign_many                  | AuthJWT(3) | route.go:75
    POST /emp/edit_campaign                         | AuthJWT(3) | route.go:76
    POST /emp/edit_campaign_status                  | AuthJWT(3) | route.go:77
    POST /emp/edit_ads                              | AuthJWT(3) | route.go:78
    POST /emp/create_ads                            | AuthJWT(3) | route.go:79
    POST /emp/create_ads_many                       | AuthJWT(3) | route.go:80
    POST /emp/create_ads_batch                      | AuthJWT(3) | route.go:83
    GET  /emp/ads_collection                        | AuthJWT(3) | route.go:84
    GET  /emp/ads_collection/export_excel           | AuthJWT(3) | route.go:85
    GET  /emp/ads_collection/summary/:campaign_id   | AuthJWT(3) | route.go:86
    POST /emp/ads_collection/create_index           | AuthJWT(25) | route.go:87
    POST /emp/re_hydra_statement                    | AuthJWT(3) | route.go:89
    POST /emp/re_hydra_campaign_statement           | AuthJWT(3) | route.go:90
    POST /emp/re_hydra_campaign_statement/reset_then_rehydra | AuthJWT(3) | route.go:91
    GET  /emp/re_hydra_campaign_statement/status/:campaign_id | AuthJWT(3) | route.go:92
    POST /emp/highest_deposit                       | AuthJWT(3) | route.go:93
    GET  /emp/check_demo                            | no auth | route.go:94
    GET  /emp/web_service/:action                   | AuthJWT(3) | route.go:97
    GET  /emp/department                            | AuthJWT(3) | route.go:98
    GET  /emp/list_menu                             | AuthJWT(3) | route.go:99
    GET  /emp/employee                              | AuthJWT(3) | route.go:100
    GET  /emp/employee/:id                          | AuthJWT(3) | route.go:101
    GET  /emp/sub_employee                          | AuthJWT(3) | route.go:102
    PUT  /emp/employee                              | AuthJWT(3) | route.go:103
    POST /emp/employee                              | AuthJWT(3) | route.go:104
    PUT  /emp/employee_active                       | AuthJWT(3) | route.go:105
    DELETE /emp/employee                            | AuthJWT(3) | route.go:106
    GET  /emp/setting_role                          | AuthJWT(3) | route.go:109
    POST /emp/domains                               | AuthJWT(3) | route.go:133
    GET  /emp/domains/service/:service              | AuthJWT(3) | route.go:134
    PUT  /emp/domains/:id                           | AuthJWT(3) | route.go:135
    DELETE /emp/domains/:service/:id                | AuthJWT(3) | route.go:136
    PATCH /emp/campaigns/:service/:campaign_id/domains/:domain_id/use | AuthJWT(3) | route.go:139
    GET  /emp/campaigns/:service/:campaign_id/domains/use | AuthJWT(3) | route.go:140
    POST /emp/list_banner                           | AuthJWT(3) | route.go:146
    POST /emp/banner                                | AuthJWT(3) | route.go:147
    DELETE /emp/banner                              | AuthJWT(3) | route.go:148
    GET  /emp/article/:groupName/:serviceName       | AuthJWT(3) | route.go:153
    POST /emp/article/:groupName/:serviceName       | AuthJWT(3) | route.go:155
    GET  /emp/agent_type                            | no auth | route.go:164
    DELETE /redis/:redisKey                         | no auth | route.go:168
    POST /external/create_cloudfront               | no auth | route.go:175

consumes:
  data:
    MONGODB_DB_NAME env (OFFICE DB) | database_config (R on startup, drives multi-tenant connections) | _config/db.go:64
    MONGODB_DB_NAME env (OFFICE DB) | account_office (R/W) | controller/member.go via loginService
    MONGODB_DB_NAME env (OFFICE DB) | campaign (R/W) | controller/dashboard.go
    MONGODB_DB_NAME env (OFFICE DB) | ads_statement (R/W) | controller/dashboard.go
    MONGODB_DB_NAME env (OFFICE DB) | ads_collection (R/W) | controller/dashboard.go
    per-service tenant DBs (loaded from database_config at startup) | campaign, ads_statement, deposit_statement, withdraw_statement, member_affiliated, self_stat_realtime, game_group | controller/dashboard.go

  external:
    AWS S3 bucket "hydra-landing" (ap-southeast-1) | image upload/download | service/awsService.go:31,63
    LINE Login OAuth | CHANNEL_SECRET env, CHANNEL_ID env | service/loginService.go:141,146,209,214

observations:
  - CRITICAL: AWS_ACCESS_KEY_ID "[REDACTED_AWS_KEY_ID]" and AWS_SECRET_ACCESS_KEY "[REDACTED_AWS_SECRET]" are HARDCODED in source code โ€” these are live IAM credentials and must be rotated immediately | service/mainService.go:478-479
  - Redis password hardcoded as "redispw" in source โ€” should be moved to env var | service/redisService.go:73
  - JWT fallback secret is literal string "secret" when SECRET env is empty โ€” trivially forgeable tokens | service/authJWT.go:43
  - POST /v1/reset_statement and data-restat endpoints have no auth โ€” any unauthenticated caller can reset affiliate/hydra statement data | route.go:37-38
  - DELETE /redis/:redisKey has no auth โ€” any caller can delete any Redis key | route.go:168
  - POST /emp/member_affiliate and /emp/member_statement_affiliate have no auth despite returning affiliate member data | route.go:59-60
  - http.Client{} used without Timeout in both POST path of HttpHandle and CALLHttpExternal โ€” risk of goroutine leaks | service/mainService.go:102,135
  - Multi-tenant ResourceService loaded from database_config collection at startup with no refresh mechanism โ€” adding new tenants requires restart | _config/db.go:61-104
  - go.mod uses go 1.16 (very old); github.com/dgrijalva/jwt-go is unmaintained and has known CVEs โ€” should migrate to golang-jwt/jwt | go.mod:1,6
  - Password stored and compared in plaintext in MongoDB (account_office.password) โ€” no bcrypt or similar hashing | controller/member.go:80,226
  - ๐ก affiliate_standalone calls http://localhost:5053/cron/calculate_data and /cron/trigger_done โ€” these routes do not appear in route.go; the endpoints seem missing or never implemented
