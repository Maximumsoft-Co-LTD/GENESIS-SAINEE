## Card: go_sms

identity: Go 1.24.0 | Dual-mode (_cmd/main.go) HTTP port 8000 OR RabbitMQ subscriber | port 8000 (HTTP mode) | Dockerfile present (Alpine multi-stage)

provides:
  http:
    POST /global/createSMS | no auth | sms/route.go:21
    GET  /global/sms/:nameserver | no auth | sms/route.go:22
    POST /global/smsNewTopup/:nameserver | no auth | sms/route.go:24
    POST /global/smsNewTopupJSON/:nameserver | no auth | sms/route.go:25
    POST /global/log-check-sms/:service | no auth | sms/route.go:26
    POST /global/sendLineStatement/:server | AuthHeader() | sms/route.go:27
    GET  /global/healthcheck/:service | no auth | sms/route.go:20
    GET  /metrics | Prometheus ginmetrics | controller/api.go:27
  consume:
    RabbitMQ fanout exchange (GROUP env: TOPUP|NEWTOPUP|PUSHBULLET|LINEBOT_NEWTOPUP) | controller/que-sub.go:24-29
  cron: none observed

consumes:
  http-out: https://monitor.thezeus.online (hardcoded Firebase monitoring URL) | sms/firebase.go:33
  publish:  fanout exchange "SMS" | PublishSMS() | sms/que-pub.go:77-97
  publish:  fanout exchange "SMS_NEWTOPUP_{service}" | PublishSMSNewTopup() | sms/que-pub.go:100-127
  publish:  fanout exchange "STATEMENT" | PublishBankStatement() | sms/que-pub.go:55-74
  publish:  direct queue "OTP_{bankcode}_{phone}" | PublishOTP() | sms/que-pub.go:128-148
  publish:  direct queue for AddCredit | PublishAddCredit() | sms/que-pub.go:150-162
  data:     MongoDB "abaoffice_db" multi-tenant via ConnectionService | db/db.go:33,67
  external: LINE Bot API (direct HTTP calls) | sms/linebot.go:25-74

observations:
  - CRITICAL: Hardcoded RabbitMQ credential "amqp://abaofficerabbitmq:Zxcvasdf789@52.77.27.18:5672/" | sms/publish.go:240
  - CRITICAL: Hardcoded CloudAMQP credential "amqps://xhnvvjnt:ItKB96u5lnLRw1RgKqgfY4z5ezCJQSSH@exotic-blond-turkey.rmq6.cloudamqp.com/xhnvvjnt" | sms/publish.go:300
  - CRITICAL: Same hardcoded CloudAMQP credential duplicated in LINEBOT_NEWTOPUP path | sms/publish.go:356
  - HIGH: Dual code paths — legacy sms/publish.go (hardcoded URIs, streadway/amqp) and new sms/que-pub.go (env-based, amqp091-go); legacy path still reachable
  - HIGH: Swallowed errors — handlers and publish functions return nil on error without logging | sms/publish.go:18-20,50
  - MEDIUM: No context propagation in RabbitMQ reconnect goroutine | rabbitmq/subscribe.go:99-105
  - MEDIUM: Retry without jitter in reconnect loop | rabbitmq/connection.go:74-87
  - MEDIUM: Layer violation — db/db.go imports sms/service/que (DB layer depends on service layer)
  - MEDIUM: No test files — zero coverage on SMS ingestion critical path
  - LOW: jaeger/init.go declared as package sms instead of package jaeger | jaeger/init.go:1
