## Card: affiliate_standalone

identity: Go 1.23 / Gin | _cmd/main.go -> affiliate.StartServer() | port 4042 (server.go:64) | Dockerfile present, docker-compose.yml present

provides:
  http:
    GET  /v1/demo                           | no auth | route.go:72
    POST /v1/reset_cron                     | no auth | route.go:73
    POST /v1/aff_manual                     | no auth | route.go:74
    GET  /v1/pay_manual                     | no auth | route.go:75
    GET  /v1/pay_manual_restat              | no auth | route.go:76
    POST /v1/pay_manual_realtime            | no auth | route.go:77
    POST /v1/pay_manual_from_self_stat_realtime | no auth | route.go:78
    GET  /v1/pay_wallet_from_aff_statement  | no auth | route.go:79
    POST /v1/resetcashback                  | no auth | route.go:80
    POST /v1/aff_by_self_stat_id            | no auth | route.go:81
    POST /v1/callback_sport                 | no auth | route.go:88
    POST /v1/callback_muay                  | no auth | route.go:89
    POST /v1/callback_affiliate             | no auth | route.go:93
    POST /v1/cashback_manual                | no auth | route.go:95
    POST /v1/hydra_re_stat                  | no auth | route.go:97
    POST /v1/hydra_re_register              | no auth | route.go:98
    POST /v1/HydraReportActiveDay           | no auth | route.go:99
    POST /v1/clear_wallet_aff               | no auth | route.go:101
    POST /v1/HydraReportUserActiveDay       | no auth | route.go:102
    POST /v1/reset_selfstat                 | no auth | route.go:104
    POST /v1/paid_cashback                  | no auth | route.go:105
    POST /v1/pay_cashback_realtime          | no auth | route.go:106
    GET  /v1/calculate_affilateufabet       | no auth | route.go:108
    GET  /v1/pay_affilateufabet             | no auth | route.go:109
    POST /v1/cal_affilatebyID               | no auth | route.go:111
    POST /v1/clear_voidbet_allinone         | no auth | route.go:113
    POST /v1/ticket_reset_date              | no auth | route.go:115
    POST /v1/manual/covid_report_day        | no auth | route.go:117
    POST /v1/manual-tournament              | no auth | route.go:119
    GET  /v1/report_incorrect_lotto         | no auth | route.go:121
    POST /v1/manual-calculate-self-stat     | no auth | route.go:123
    POST /v1/callback_lotto (LOTTO_TYPE=default) | no auth | route.go:149
    POST /v1/callback_lotto (LOTTO_TYPE=AGENT)   | no auth | route.go:154
    POST /v1/callback_mini_game (LOTTO_TYPE=AGENT) | no auth | route.go:155
    PATCH /v1/re_cal_mini_game (LOTTO_TYPE=AGENT) | no auth | route.go:156
    GET  /v1/sum_report_daymonth (LOTTO_TYPE=AGENT) | no auth | route.go:158
    GET  /v1/cron_calculate (LOTTO_TYPE=AGENT)      | no auth | route.go:160
    GET  /v1/cron_aff_realtime (LOTTO_TYPE=AGENT)   | no auth | route.go:161
    POST /test/create_member                | no auth | route.go:127
    POST /test/create_deposit               | no auth | route.go:128
    GET  /test/callback                     | no auth | route.go:131
    GET  /test/commission                   | no auth | route.go:132
    POST /test/affiliate_cal/manual         | no auth | route.go:138
    POST /test/affiliate_cal_all/manual     | no auth | route.go:139

  publish:
    queue DYNAMICQUEUE_<SERVICE> | type=COVIDAFFILIATE, HYDRA_UFA_AFFILIATE, AFFILIATE_LOTTO deposit/withdraw events | service/affiliateRealtimeV2.go:571,664,758,793,880
    queue WALLETWITHDRAW_<SERVICE> | type=EVENTACTION withdraw credit events | service/mainService.go:253 via service/statementService.go:359

  cron:
    Every 4min | AffiliateRealtimeV2 (default mode) | service/cronService.go:240
    Cron "20 * * * *" | RealtimeWithdraw | service/cronService.go:241
    Cron "0 8 * * *" | RealtimeDeposit | service/cronService.go:242
    Every 10min | CronCallbackSport | service/cronService.go:245
    Every 5min | CreatetransectionSport | service/cronService.go:248
    Every 1day at 23:59 | CalculateV2 (affiliate commission calc) | service/cronService.go:251
    Every 1day at 00:30 | HandlePayToWallet (pay affiliate reward) | service/cronService.go:252
    Every 1day at 01:15 | PayToWalletV2 | service/cronService.go:253
    Every 1day at 02:00 | CronUpdateDuration | service/cronService.go:254
    Every 3h | GamesHitsDay | service/cronService.go:257
    Every 3h | UpdateGameRocketWin | service/cronService.go:258
    Every 1day at 01:00 | UpdateBrandGameRocketWin | service/cronService.go:259
    Every 1day at 00:10 | HandleResetCashback | service/cronService.go:270
    Every 1day at 01:10 | HandleResetCommission | service/cronService.go:275
    Every 1day at 00:30 | TicketService | service/cronService.go:278
    Every 1day at 00:30 | TournamentServiceV2 | service/cronService.go:281
    Every Monday at 00:30 | ResetCreditCovid | service/cronService.go:286
    Every 1day at 00:45 | DeleteAgentLog | service/cronService.go:290
    Every 5min | BankNotify | service/cronService.go:293
    Every 1day at 23:30 (once) | SetDefaultBonusID | service/cronService.go:296
    Every 1day at 00:00:01 | autoCarryNextDay | service/cronService.go:298
    Every 30min (random start) | CheckBankOutstanding OLD | service/cronService.go:302
    Every 10min (random start) | CheckBankOutstanding NEW | service/cronService.go:303
    Cron "45 * * * *" (LOTTO_TYPE=AGENT) | SummaryReportHour | service/cronService.go:266
    UFA mode - Every 30min | RealtimeDepositWithdrawUFA | service/cronService.go:208
    UFA mode - Every 15min | LoadTransactionAgentUFATransfer | service/cronService.go:210
    UFA mode - Every 1day at 01:00 | CronCalculateTransactionAgentUFATransfer | service/cronService.go:211
    UFA mode - Every 1day at 02:00 | CalculateV2UFATransfer | service/cronService.go:212
    UFA mode - Every 1day at 02:30 | HandlePayToWalletUFATransfer | service/cronService.go:213
    UFABET/BETFLIX mode - Every 1day at 00:30 | AffiliateRealtimeUfabet | service/cronService.go:224
    UFABET/BETFLIX mode - Every 1day at 01:30 | PayToWalletV2 | service/cronService.go:225

consumes:
  http-out:
    SERVICE_API env + /userinfo?key=<username> | GET credit check | service/agentService.go:44
    SERVICE_API env + /balance-history/<username> | GET balance history | service/agentService.go:78
    SERVICE_API env + /checkstatus | GET agent health | service/agentService.go:95
    SERVICE_API env + /seamless/balance | POST balance | service/agentService.go:449
    SERVICE_API env + /load-transaction | POST transaction load | service/agentService.go:473
    http://localhost:5053/cron/calculate_data | POST send affiliate data (dead code path) | service/calculateService.go:1512
    http://localhost:5053/cron/trigger_done | POST trigger done (dead code path) | service/calculateService.go:1530

  data:
    MONGODB_DB_NAME env (MongoDB) | config_system (R/W) | service/calculateService.go:188,1060,1323
    MONGODB_DB_NAME env | member_affiliated (R) | service/calculateService.go:404,427
    MONGODB_DB_NAME env | self_stat_realtime (R/W) | service/calculateService.go:545,1588
    MONGODB_DB_NAME env | statement_affiliate_day (R/W) | service/calculateService.go:903,1187
    MONGODB_DB_NAME env | stat_day / stat_week / stat_month (R/W) | service/calculateService.go:965,1031,1166
    MONGODB_DB_NAME env | wallet_statement (R/W) | service/calculateService.go:971,1025,1160
    MONGODB_DB_NAME env | game_group (R) | service/calculateService.go:429
    MONGODB_DB_NAME env | deposit_statement (R) | service/calculateService.go:795
    MONGODB_DB_NAME env | level_config (R) | service/calculateService.go via GetLevelAffiliateByCalType
    MONGODB_DB_NAME env | affiliate_current (W) | service/calculateService.go:655

observations:
  - ALL /v1 routes and /test routes have NO auth middleware — any caller can trigger commission calculations or cashback resets | route.go:70-144
  - http.Client{} used without Timeout on POST calls in mainService.go:146 — risk of hung goroutines on slow HTTP | service/mainService.go:146
  - MQReconnector only retries once (recon=false after first reconnect attempt) — if second disconnect occurs, RabbitMQ channel is permanently dead | _config/mq.go:43-53
  - SendToAffiliateSystem and TriggerDone call hardcoded localhost:5053 — no env override possible; will silently fail if hydra-affilaite is not co-located | service/calculateService.go:1512,1530
  - PayToWallet writes wallet_statement directly (W) without idempotency check beyond SHA1 hash — double-pay possible if cron retries before is_pay flag is set | service/calculateService.go:1019,1025,1031
  - StatementService publishes EVENTACTION to WALLETWITHDRAW_<SERVICE> with auto-ack on consumer side — failed processing silently drops message with no DLQ | service/statementService.go:359
  - MQ publishes use auto-ack (queue consumers configured with autoAck=true in queue-dynamic-go) — no DLQ protection on any queue | service/mainService.go:247-274
  - Secret for JWT not referenced in this service but env var ACCESS_SECRET is common pattern — this service does not validate JWTs at all
  - BANDGAME env controls cron schedule selection (lines 223-263); misconfig silently falls to default mode with no error | service/cronService.go:223
  - github.com/streadway/amqp (deprecated, unmaintained) used; upgrade path is rabbitmq/amqp091-go | go.mod:13
