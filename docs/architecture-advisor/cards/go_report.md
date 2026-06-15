## Card: go_report

identity: Go 1.15, Gin (none — no HTTP server), entrypoint `_cmd/main.go` | no HTTP port | Dockerfile + cloudbuild.yaml present

provides:
  http:    (none — no HTTP server; pure queue consumer)
  consume: exchange=$EXCHANGE_NAME (fanout, durable) / queue=$QUEUE_NAME (exclusive, auto-ack=true) | handlers: DEPOSIT→ReportActiveUser, WITHDRAW→ReportWithdrawActive+ReportWithdraw, BANK→ReportStatement, MEMBER→ReportMember, LOGIN→ReportMemberLogin, MIGRATECOUPONV2→MigrateCouponV2 | rabbitmq/subscribe.go:60-158
  consume: queue=OUTSTANDING_$SERVICE (no exchange declared, direct queue) | handler: OUTSTANDINGUPDATE→UpdateTodayOutstanding | rabbitmq/subscribe.go:216-291
  cron:    every 1 hour → StartRecovery (reprocess deposit_statement+withdraw_statement last 24h) | reportOld/cronjob.go:35
  cron:    every day at 02:00 → ManageDuplicateReportActiveUser | reportOld/cronjob.go:36
  cron:    every day at 02:20 → UpdateReportNotEqualStatement | reportOld/cronjob.go:37
  cron:    every 5 minutes → outStandingUpdateReport2 (drain Redis list `outstanding_deposit_statement`) | reportOld/cronjob.go:38

consumes:
  http-out: hardcoded http://mongo-sms.api-node.com/v1/logSMSByPhoneNumber/$SERVICE POST (no timeout) | report/deposit.go:352, report/withdraw.go:183
  http-out: hardcoded http://event-coupon.api-node.com/api/couponConfig/$service GET | report/script.go:1367
  http-out: hardcoded http://event-coupon.api-node.com/api/couponByID/$id GET | report/script.go:1386
  http-out: $Coupon.CouponService/coupon?service=$SERVICE GET, $CouponService/coupon/ref_bonus?service=$SERVICE POST (URL from queue payload) | report/migrateReport.go:33-36, 230-231
  http-out: hardcoded https://coupon-service-escobar.thesonicblue.xyz/api/coupon GET, PATCH (PROJECT==ESCOBAR only) | report/script.go:1451-1541
  data:     abaoffice_db.database_config | R | db/db.go:107 (bootstrap: looks up tenant config by SERVICE)
  data:     $configDatabase.DbName.deposit_statement | R/W (UpdateOne status_report, UpdateOne ref_bonus bulk) | report/deposit.go:233-234, report/script.go:596-620
  data:     $configDatabase.DbName.withdraw_statement | R/W (UpdateOne status_report, UpdateOne ref_bonus bulk) | report/withdraw.go:69-70, report/script.go:608-619
  data:     $configDatabase.DbName.bank_statement | W (UpdateOne status_report) | report/statement.go:24
  data:     $configDatabase.DbName.member_account | R (GetOne by username) | report/repository.go:349-362, report/member.go:69
  data:     $configDatabase.DbName.report_day | R/W (upsert) | report/deposit.go:56-80
  data:     $configDatabase.DbName.report_hour | R/W (upsert) | report/deposit.go:86-113
  data:     $configDatabase.DbName.report_active_user_month | W (upsert) | report/deposit.go:245
  data:     $configDatabase.DbName.report_active_user_day | W (upsert) | report/deposit.go:251, report/member.go:30
  data:     $configDatabase.DbName.report_active_user_hour | W (upsert) | report/deposit.go:253
  data:     $configDatabase.DbName.report_bonus_month | W (upsert) | report/deposit.go:136
  data:     $configDatabase.DbName.report_bonus_week | W (upsert) | report/deposit.go:139
  data:     $configDatabase.DbName.report_bonus_day | W (upsert) | report/deposit.go:142, report/deposit.go:868
  data:     $configDatabase.DbName.report_bonus_hour | W (upsert) | report/deposit.go:145
  data:     $configDatabase.DbName.report_sms | R/W | report/repository.go:300-346
  data:     $configDatabase.DbName.config_history | R/W (GetOne, CreateMany, DeleteMany COUPON_V2) | report/deposit.go:641, report/migrateReport.go:208-219
  data:     $configDatabase.DbName.config_system | R/W (GetOne, UpdateOne queue_is_close, is_script_report) | report/script.go:238-256, 717
  data:     $configDatabase.DbName.ticket_config | R/W (GetMany, BulkWrite ref_bonus) | report/script.go:747, 831
  data:     $configDatabase.DbName.bonus_config | R/W (GetMany, BulkWrite ref_bonus) | report/script.go:861, 915
  data:     $configDatabase.DbName.ranking_config | R/W (GetMany, BulkWrite ref_bonus) | report/script.go:1146, 1293
  data:     $configDatabase.DbName.event_config | R/W (GetMany COUPON+MINIEVENT, BulkWrite ref_bonus) | report/script.go:1313, 1423
  data:     $configDatabase.DbName.shop_exchange | R/W | report/script.go:1601, 1638
  data:     $configDatabase.DbName.shop_item | R/W | report/script.go:1645, 1678
  data:     Redis DB=5 key=`config_history_<refBonus>` (GET/SET TTL=15m) | report/deposit.go:627-654
  data:     Redis DB=5 list key=`outstanding_deposit_statement` (LPUSH, LRANGE, LTRIM) | reportOld/cronjob.go:92-205

observations:
  - HARDCODED MongoDB credentials in db/db.go:58: `mongodb://abaoffice:zxcvasdf789@mongodb-aba.cluster-c0skt7cppmab.ap-southeast-1.docdb.amazonaws.com:27017` — production secret in source code | db/db.go:58
  - HTTP client in callapi/callapi.go uses monaco-io/request with NO Timeout field set — hung remote blocks consumer goroutine forever | callapi/callapi.go:31-36
  - auto-ack=true on both queue consumers — any panic mid-handler causes silent message loss with no DLQ | rabbitmq/subscribe.go:98, 234
  - RabbitMQ consumer goroutine spawned via `go goreport.SubscribeOutstanding(...)` then blocks on `goreport.Subscribe(...)` in same thread; reconnection logic at rabbitmq/connection.go:79 calls StartAMQP but there is no DLQ wiring — same auto-ack risk post-reconnect
  - MigrateCouponV2 (triggered by MIGRATECOUPONV2 queue message) deletes ALL COUPON_V2 config_history then recreates — no idempotency key, double-delivery destroys configs | report/migrateReport.go:205-222
  - reportSMSDeposit and reportSMSWithdraw are dead code (commented out at call sites) but still call external API with no timeout | report/deposit.go:243, report/withdraw.go:76
  - go 1.15 in go.mod — end-of-life since 2021; many security patches missed | go.mod:3
  - Script functions (MigrateOldStatement*) manipulate config collections (ticket_config, bonus_config, ranking_config, event_config, shop_exchange, shop_item) — cross-domain schema coupling: go_report WRITES to config tables that back-office services own | report/script.go:831, 915, 1293, 1423, 1638
  - `initContextBulk()` sets 60-minute timeout; used in WriteBulkStatement; combined with no-auto-ack=false this means a stalled bulk write holds the consumer for 60 minutes | report/repository.go:573-586
  - Telegram bot token wired in script (tgbotapi import used) — notification token likely comes from env but import present from dead code path | report/script.go:16
  - `OUTSTANDING` queue declared without binding to any exchange (no ExchangeDeclare, no QueueBind) — relies on default exchange; exchange name `OUTSTANDING_$SERVICE` must match publisher exactly | rabbitmq/subscribe.go:185-243
  - reportDashboardByDepositStatement only runs when GROUP==NEWTOPUP; OutStandingUpdateReport2 only runs when Redis list is non-empty — feature-flag branches increase cognitive complexity
