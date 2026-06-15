## Card: 3rd-gateway

identity: Go 1.23 | Gin HTTP server | port 8000 (app.go:65) | Dockerfile present, no docker-compose.yml

provides:
  http:
    POST /api/register                              | no auth middleware (open customer registration)                   | routes/main.go:31
    GET  /api/get-list-gateway                      | AuthClientMiddleware (client_id + api_key DB lookup)             | routes/main.go:33
    POST /api/get-connected-by-account              | AuthClientMiddleware                                              | routes/main.go:34
    POST /api/connect-geteway                       | AuthClientMiddleware                                              | routes/main.go:35
    POST /api/disconnect-default-geteway            | AuthClientMiddleware                                              | routes/main.go:36
    POST /api/fund-transfer                         | AuthClientMiddleware                                              | routes/main.go:38
    POST /api/get-transactions                      | AuthClientMiddleware                                              | routes/main.go:39

  consume: (none — no RabbitMQ consumer)

  cron: (none)

consumes:
  http-out: https://api.tmn.one/api.php (TrueMoneyWallet API, AES-encrypted payload) | gateway-hub/tmn-one/client.go:43
  http-out: https://api.tmn.one/proxy.dev.php/tmn-mobile-gateway/ (wallet endpoint via fasthttp) | gateway-hub/tmn-one/main.go (WalletEndpoint constant)
  http-out: https://tmn.one (TmnoneEndpointcl, via fasthttp) | gateway-hub/tmn-one/main.go
  http-out: https://dp.thezeus.tech/api/initWithdraw (K8S Helm install API, hardcoded URL+API key) | service/helminstall/install-helm.go:15–16
  data:     DB_DBNAME (env) + collection gateway_config    | R   | config/entity.go:15
  data:     DB_DBNAME (env) + collection customer_registry | R/W | config/entity.go:16
  data:     DB_DBNAME (env) + collection gateway_connect   | R/W | config/entity.go:17
  external: TrueMoneyWallet (TMN ONE / TMN TWO) — only gateway implemented in factory | gateway-hub/factory.go:40–47

observations:
  - CRITICAL: Hardcoded API key "mXjtZ7q9D0zhG6NN06832mqLCr0KlM" and hardcoded URL "https://dp.thezeus.tech/api/initWithdraw" in source code — any reader of the repo can call this internal K8S Helm API | service/helminstall/install-helm.go:14–16
  - CRITICAL: /api/register is unauthenticated — any party can self-register as a customer and potentially obtain gateway credentials | routes/main.go:31
  - CRITICAL: TMN gateway HTTP clients (&http.Client{} and &fasthttp.Client{}) have no Timeout set — fund-transfer calls to TrueMoney wallet will hang indefinitely if TrueWallet API is slow/down | gateway-hub/tmn-one/client.go:59, gateway-hub/tmn-one/func.go:19–20, gateway-hub/tmn-one/login.go:44, gateway-hub/tmn-one/main.go:135–136
  - CRITICAL: Credentials token (TMN PIN, LoginToken, TmnID) stored as AES-GCM encrypted blob in MongoDB gateway_connect collection using AES_KEY_FIX32 env — if env key leaks, all stored credentials are compromised | config/environment.go:22, controller/gateway.go:144
  - HIGH: gateway-hub/factory.go only registers TMN_ONE and TMN_TWO — all other gateway_code values return "unknown payment gateway" error at runtime, but gateway_config collection can contain arbitrary codes with no compile-time validation | gateway-hub/factory.go:39–47
  - HIGH: AuthClientMiddleware compares client_id and api_key in plaintext MongoDB filter — no rate limiting, no account lockout; brute-forceable | middleware/customer-client.go:26
  - HIGH: FundTransfer and GetTransactions update credentials_token back to MongoDB after each call (UpdateOne) without optimistic locking — concurrent requests can overwrite each other's refreshed credentials | controller/gateway.go:462,505
  - HIGH: ConnectGateway triggers helminstall.CallK8SHelmByWithdraw unconditionally — a failed helm call is logged but ignored (error swallowed), potentially leaving K8S state out of sync with DB | controller/gateway.go:359–361
  - MEDIUM: No OTel tracing — OpenTelemetry provider initialization is fully commented out in app.go:43–50; only zap logger active
  - MEDIUM: TMN login flow involves storing raw PIN in struct memory during session (t.Pin field) — not cleared after use | gateway-hub/tmn-one/main.go (struct TMNOne field Pin)
  - MEDIUM: No connection pool configuration for MongoDB (minPoolSize=3/maxPoolSize=10 defined in rsconn/mongo-db.go:16–17, applied) — better than default but pool exhaustion not handled
  - MEDIUM: fasthttp.Client used alongside net/http.Client in tmn-one — mixed HTTP stacks, fasthttp does not respect standard net/http proxy/transport settings, inconsistent timeout behavior | gateway-hub/tmn-one/account-detail.go:32, login.go:44
  - INFO: go-mongox library used for type-safe repository pattern (MongoxRepository[T]) — newer than raw bson driver used in 3rd-payment and que_payment
  - INFO: AES_KEY_FIX32 must be exactly 32 bytes — no validation at startup; wrong length causes silent AES-GCM failures at runtime 🟡
  - INFO: No CI/CD workflow file (.github/workflows/) found — unlike other repos in the monorepo 🟡
