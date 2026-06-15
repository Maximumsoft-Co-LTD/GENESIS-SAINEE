## Card: queue-dynamic-go

identity: Go 1.19 / no HTTP server | _cmd/main.go -> queuework.StartQueue() | no HTTP port (pure queue worker) | Dockerfile present, docker-compose.yml present

provides:
  http:    (none — no HTTP listener)

  consume:
    queue QUEUE_NAME_<SERVICE> env (e.g. "DYNAMICQUEUE_<SERVICE>") | no exchange, default routing | pub/amqp.go:33-35
    message type dispatch:
      type="COVIDAFFILIATE"       -> covid.AffiliateSystem()      | pub/amqp.go:81-83
      type="AFFILIATE_LOTTO"      -> covid.AffStatementLotto()    | pub/amqp.go:84-85
      type="COVID_CREDIT"         -> covid.CovidCreditHandle()    | pub/amqp.go:86-87
      type="COVID_ACCOUNT"        -> covid.CovidAccountHandle()   | pub/amqp.go:88-89
      type="COVID_OUTSTANDING"    -> covid.CovidOutstandingHandle()| pub/amqp.go:90-91
      type="COVID_RESET"          -> covid.CovidResetCredit()     | pub/amqp.go:92-93
      type="HYDRA_UFA_AFFILIATE"  -> covid.AffiliateSystemHydraUFA()| pub/amqp.go:94-95
      DynamicQueue=true           -> queue.ReceiveQueueLog()       | pub/amqp.go:96-98
      (default)                   -> helper.Response(1, "NOT_FOUND_PARAMETER") | pub/amqp.go:99-100

  publish:
    (reply-to pattern only) | reply to d.ReplyTo queue if set | pub/amqp.go:103-119

consumes:
  data:
    MONGODB_DB_NAME env (MongoDB) | campaign (R/W) | covid/affiliated.go:49,188,300,621,754
    MONGODB_DB_NAME env | ads_collection (R/W) | covid/affiliated.go:407,613,739
    MONGODB_DB_NAME env | ads_statement (R/W) | covid/affiliated.go:209,260,339,643,693,794,1156,1216
    MONGODB_DB_NAME env | deposit_statement (R/W) | covid/affiliated.go:267,700,902
    MONGODB_DB_NAME env | withdraw_statement (R/W) | covid/affiliated.go:346,801,1244
    MONGODB_DB_NAME env | member_affiliated (R) | covid/affiliated.go:1407,1417
    MONGODB_DB_NAME env | self_stat_realtime (R/W) | covid/affiliated.go (TicketServiceRealtime)
    MONGODB_DB_NAME env | ticket (R/W) | covid/affiliated.go (TicketServiceRealtime)

  external:
    Telegram Bot API | TELEGRAM_BOT_TOKEN + TELEGRAM_CHAT_ID env | sends connect/disconnect notifications | pub/connection.go:50-55,63-68
    OpenTelemetry collector | OTEL_URL env (grpc) | traces | startQueue.go:31-38, otel/

observations:
  - Consumer uses autoAck=true (auto-ack on message receipt before processing) — any processing failure silently drops the message with no retry and no DLQ | pub/amqp.go:60
  - Three separate versions of the core hydra stat handler exist (handleHydraStat, handleHydraStatV2, handleHydraStatV3) — v2 is called by AffiliateSystem; v3 appears fully implemented but NOT called; code divergence risk | covid/affiliated.go:44,363,818
  - Reconnector only retries once (recon=false) — if RabbitMQ disconnects a second time after the first reconnect, StartAMQP is NOT restarted and the worker permanently stops consuming | pub/connection.go:83-97
  - QUEUE_NAME env must match publisher value ("DYNAMICQUEUE") exactly — mismatch would silently leave messages unprocessed with no alerting | pub/amqp.go:33
  - No idempotency check on AffiliateSystem / handleHydraStatV2 — duplicate messages (e.g. from publisher retry on connection loss) would double-increment campaign stats | covid/affiliated.go:23-42
  - MongoDB connection timeout set to 5 seconds at context creation — _config/db.go:56; very short for batch insert operations
  - COVID_USE env gates handleCovidStat — empty string disables it; this env dependency is undocumented | covid/affiliated.go:31
  - Service name "queuework" in go.mod differs from deployment naming convention — could confuse Kubernetes service discovery | go.mod:1
