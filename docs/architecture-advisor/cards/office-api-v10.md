## Card: office-api-v10

identity: Go 1.18, Gin | _cmd/main.go โ’ server.go โ’ StartServer() | :7777 | Dockerfile present, docker-compose-jfrog.yml present

provides:
  http:
    POST   /api/login                                    | no auth                    | route/employee.go:22
    POST   /api/login-bypass                             | no auth                    | route/employee.go:23
    POST   /api/login-check                              | no auth                    | route/employee.go:24
    POST   /api/login-verify                             | no auth                    | route/employee.go:36
    POST   /api/login-line-supercom                      | no auth                    | route/employee.go:25
    POST   /api/validate-bank-user                       | no auth                    | route/employee.go:27
    POST   /api/validate-phone-user                      | no auth                    | route/employee.go:28
    GET    /api/login-platform                           | no auth                    | route/callbackLogin.go:14
    GET    /api/make-link-login                          | no auth                    | route/callbackLogin.go:15
    GET    /api/register-platform                        | no auth                    | route/callbackLogin.go:16
    POST   /api/login-platform-verify                    | no auth                    | route/callbackLogin.go:17
    POST   /api/slip-verify-deposit-statement-by-member/:service | no auth          | route/deposit.go:39
    POST   /api/WithdrawCallback                         | no auth                    | route/withdraw.go:53
    POST   /api/auto/createQueueWithdraw/:service        | no auth                    | route/withdraw.go:59
    POST   /api/WithdrawStatementBotMode/:service        | no auth                    | route/withdraw.go:64
    POST   /api/checkWithdrawStatementBotMode/:service   | no auth                    | route/withdraw.go:65
    POST   /api/CallbackWithdrawStatementBotMode/:service| no auth                    | route/withdraw.go:66
    POST   /api/Payment/PaymentCreateStatement/:service  | no auth                    | route/payment.go:17
    POST   /api/Payment/CallBackWithdrawAmount/:service  | no auth                    | route/payment.go:18
    POST   /api/Xpay/CreateStatement/:service            | no auth                    | route/payment.go:19
    POST   /api/callback-update-status/withdraw          | no auth                    | route/payment.go:20
    GET    /api/GetDataDefaultTemplate/:service          | no auth                    | route/config.go:46
    GET    /api/DF_OFFICE/:service                       | no auth                    | route/config.go:47
    POST   /api/init-service                             | no auth                    | route/config.go:48
    POST   /api/init-service-callback                    | no auth                    | route/config.go:49
    POST   /api/update-configsystem/:service             | no auth                    | route/migrate.go:15
    POST   /api/clear-script-status-game-report/:service | no auth                    | route/migrate.go:16
    POST   /api/firebase-noti-bank_config/:service       | no auth                    | route/bank.go:136
    PATCH  /api/bank-config-set-balance/:service         | no auth (mock/test)        | route/bank.go:140
    POST   /api/bank-config-set-access-token/:service    | no auth (mock/test)        | route/bank.go:141
    POST   /api/withdraw_proxy_bank                      | no auth (callback)         | route/bank.go:101
    POST   /api/change_account_proxy_bank                | no auth (callback)         | route/bank.go:108
    POST   /api/bank-account-install-withdraw            | no auth (callback)         | route/bank.go:110
    POST   /webhook/verify-bank/:username                | no auth (webhook)          | route/member.go:103
    POST   /api/SaveAccessTokenLine/:service             | no auth                    | route/configsystem.go:174
    POST   /api/SaveConfigRe2FA                          | no auth (DEV only comment) | route/configsystem.go:177
    GET    /api/GetMemberList/:service                   | AuthRequiredDB             | route/member.go:16
    GET    /api/employees-list                           | AuthRequiredDB             | route/employee.go:41
    POST   /api/employees-create                         | AuthRequiredDB             | route/employee.go:44
    PUT    /api/employees-update-permission              | AuthRequiredDB             | route/employee.go:47
    POST   /api/role-save                                | AuthRequiredDB             | route/employee.go:79
    POST   /api/role-update-permission                   | AuthRequiredDB             | route/employee.go:81
    POST   /api/ConfirmMoveStatement/:service            | AuthRequiredDB             | route/deposit.go:35
    POST   /api/DepositAddCredit/:service                | AuthRequiredDB             | route/deposit.go:36
    POST   /api/withdraw-confirm/:service                | AuthRequiredDB("WITHDRAW_AUTO") | route/withdraw.go:35
    POST   /api/withdraw-confirmauto/:service            | AuthRequiredDB("WITHDRAW_AUTO") | route/withdraw.go:36
    POST   /api/withdraw-again/:service                  | AuthRequiredDB("WITHDRAW_AUTO") | route/withdraw.go:38
    POST   /api/ManageWithdrawStatementRemain-auto-cornjobs/:service | CheckkeyCronJobNextDay | route/withdraw.go:22
    POST   /api/bankconfig-save/:service                 | AuthRequiredDB             | route/bank.go:18
    POST   /api/credit-return/:service                   | AuthRequiredDB("TRANSECTION_MANAGECREDIT_RETURN") | route/credit.go:20
    POST   /api/websetting-create/:service               | AuthRequiredDB             | route/configsystem.go:15
    GET    /api/employees-list (and all other /api/* routes) | AuthRequiredDB        | route/employee.go:17

  consume: none (no RabbitMQ consumer/subscribe โ€” publisher only)

  cron: none registered (no cron scheduler in codebase)

consumes:
  http-out:
    SUPERCOM_API env + /office/{project}/{memberId} โ€” GET supercom member lookup during auth | middlewares/auth.go:181
    SUPERCOM_API env + /office/ip-whitelist/ip/{office} โ€” GET IP whitelist | middlewares/auth.go:347 (header: SUPERCOM_API_KEY env)
    dbConfig.APIURL + /deposit?key=...&point=...&bonus_amount=...&canplay=...&ref_code=... โ€” GET credit add on deposit confirm | controller/deposit.go:1025
    HASH_CENTRAL_API env (config.Env.HASH_CENTRAL_API) โ€” POST hash-central channel ops | config/environment.go:17
    URL_SLIP_VERIFY env (config.Env.URL_SLIP_VERIFY) โ€” slip verification external service | config/environment.go:15
    URI_DEVICE_GENERATE env (config.Env.URI_DEVICE_GENERATE) โ€” device generation endpoint | config/environment.go:18
    https://oauth.pinkmonkey.online โ€” ๐ก referenced indirectly via login platform flow | controller/callbackLogin.go (AUTHSERVICE analog)
    AWS S3 ap-southeast-1 bucket "rocketwinoffice" โ€” file upload/delete | helper/helper.go:664-700

  publish:
    exchange STATEMENT_{service} (fanout, type="BANK")     โ€” bank statement event | helper/queue.go:162
    exchange STATEMENT_{service} (fanout, type="WITHDRAW") โ€” withdraw statement event | helper/queue.go:212
    exchange STATEMENT_{service} (fanout, type="DEPOSIT")  โ€” deposit statement event | helper/queue.go:260
    queue {service} (direct)                               โ€” add credit queue trigger | helper/queue.go:25
    queue {nameq} (direct, JSON body with service+id)      โ€” withdraw queue trigger | helper/queue.go:115
    queue OUTSTANDING_{service} (direct, type="OUTSTANDINGUPDATE") โ€” outstanding update | helper/queue.go:359
    exchange STATEMENT_{service} (fanout, type="MIGRATECOUPONV2") โ€” coupon migration | helper/queue.go:326
    RABBITMQ_URL comes from dbConfig.DBHostName per-service (read from database_config collection) ๐ก not from env directly

  data:
    OFFICE DB (env: MONGODB_DB_NAME โ’ e.g. "*_office"):
      employees         | R/W | middlewares/auth.go:167, route/employee.go
      demo_account      | R   | middlewares/auth.go:286
      database_config   | R   | db/db.go:128 โ€” registry of all per-service DB connections
      work_logs         | W   | helper/helper.go:1287, repo/log.go:35-37
      bank_config       | R   | controller/statement.go:3106
    Per-service DB (keyed by service param, connected dynamically from database_config):
      member_account    | R/W | controller/member.go:3292
      ranking_config    | R   | controller/member.go:3298
      self_stat_ranking | R/W | controller/member.go:3319-3331
      event_config      | R   | controller/wheel.go:102
      bank_statement    | R/W | controller/runbankFunc.go:94,520
      bank_statement_3d | R/W | controller/runbankFunc.go:174,521
      bank_statement_withdraw_whitelist | W | controller/runbankFunc.go:546
      generate_statement| R   | controller/runbankFunc.go:376
      bank_list         | W   | controller/runbankFunc.go:610
      config_system     | R   | controller/runbankFunc.go:553
      member_log        | W   | repo/log.go:99
      (many additional collections via generic repo.go โ€” collection name passed as param)

  external:
    AWS S3 (bucket: rocketwinoffice, region: ap-southeast-1) | helper/helper.go:664-700
    LINE Notify API (https://notify-api.line.me/api/notify) | report/publish.go:71
    Telegram Bot API | go.mod: github.com/go-telegram-bot-api/telegram-bot-api | helper/helper.go
    Sentry (no โ€” backend has no Sentry import)

observations:
  - HARDCODED AWS credentials in source: AWS_ACCESS_KEY_ID="[REDACTED_AWS_KEY_ID]" and AWS_SECRET_ACCESS_KEY="[REDACTED_AWS_SECRET]" appear in TWO files | helper/helper.go:666-667 and controller/config.go:977-978
  - HARDCODED MongoDB test credentials: "mongodb+srv://root:Zxcvasdf789@test02.9oh7e.mongodb.net/test" used as fallback when MODE != PRODUCTION, meaning non-production deploys always connect to shared test cluster | db/db.go:88
  - HARDCODED cron-job secret key "2ac714e8d1451244432013c2d7aa2fd7" and expected decoded value "147ce3cba244318c00c298680ca1da08" in source โ€” if leaked anyone can trigger /ManageWithdrawStatementRemain-auto-cornjobs | middlewares/auth.go:617-620
  - LARGE NUMBER of unauthenticated routes on sensitive operations: deposit callbacks, withdraw callbacks, payment create-statement, init-service, bank-config-set-balance/access-token, firebase-noti, proxy bank callbacks, webhook/verify-bank โ€” none require AuthRequiredDB | route/withdraw.go, route/payment.go, route/bank.go, route/config.go, route/migrate.go
  - CORS wildcard: Access-Control-Allow-Origin "*", Access-Control-Allow-Methods "*", Access-Control-Allow-Headers "*" โ€” accepts any origin for all routes | server.go:109-113
  - Multiple http.Client{} with NO Timeout field โ€” will hang indefinitely on slow external responses | controller/bank.go:512, controller/bank.go:694, controller/accountOption.go:79
  - RABBITMQ_URL per-service is read from database_config collection (not from env) โ€” if collection is corrupted or missing, AMQP connections silently fail with bare return (no error log, no DLQ) | helper/queue.go:12-17 (all publish funcs return on dial error without alerting)
  - All AMQP publish functions swallow errors: if amqp.Dial or ch.Publish fails, function returns silently โ€” deposit confirmations can be lost with no trace | helper/queue.go:12-55
  - /api/DepositAddCredit and /api/ConfirmMoveStatement (money credit ops) have no idempotency key โ€” duplicate POST will double-credit | route/deposit.go:35-36
  - AUTHLOGIN=="DEMO" bypass: entire session-key check and permission model is skipped, any ObjectID in demo_account collection grants access | middlewares/auth.go:166,286
  - STATUS_REPORT=="TRUE" env switch replaces all routes with export-only routes โ€” dangerous if misconfigured in production | server.go:49-51
  - go.mod uses github.com/dgrijalva/jwt-go v3.2.0 (CVE-2020-26160, known vulnerable JWT library) alongside github.com/golang-jwt/jwt โ€” dual JWT libraries in same codebase | go.mod:7,11
  - helper.FixIsLoginTelegramLine called in goroutine on startup โ€” side-effect mutation on startup with no error propagation | server.go:103
