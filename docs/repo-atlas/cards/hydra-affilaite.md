# Card: hydra-affilaite
> เธงเธดเน€เธเธฃเธฒเธฐเธซเน: 2026-06-12 | commit: 40367af

## identity
- **stack**: Go 1.16 (go.mod:3) / module `hydra` (go.mod:1) โ€” Gin v1.7.4 (server.go:8, go.mod:8), MongoDB driver raw, go-redis/v8, aws-sdk-go (S3 + CloudFront), nleeper/goment, dgrijalva/jwt-go โ€” backend เธฃเธฐเธเธ affiliate/เนเธเธฉเธ“เธฒ (campaign, ads, dashboard, เธชเธฃเนเธฒเธ landing page เธเธ S3+CloudFront) multi-tenant
- **module**: `hydra` (go.mod:1) ; entry `_cmd/main.go:8` เน€เธฃเธตเธขเธ `hydra.StartServer()`
- **entrypoint**: `StartServer()` server.go:13 โ€” เนเธซเธฅเธ” .env (server.go:14), gin+CORS (17-18), เน€เธเธทเนเธญเธก Mongo (20), Route (30)
- **port**: `:5053` เธฎเธฒเธฃเนเธ”เนเธเนเธ” (server.go:32) โ€” **เนเธกเนเธ•เธฃเธเธเธฑเธ docker-compose** เธ—เธตเน map `5052:5052` (docker-compose.yml:11) เนเธฅเธฐ image tag `aff-system:uat` ; image เธเธทเนเธญ `aff-system` (CI env NAME, build.yaml)
- **Dockerfile/CI**: Dockerfile FROM golang:1.18-alpine3.16 builder โ’ alpine:3.13, copy `./static` (Dockerfile:1-12, เนเธกเน EXPOSE). CI `.github/workflows/build.yaml`: self-hosted โ’ Kaniko โ’ JFrog (`JFROG_PATH/aff-system`) โ’ curl `API_DEPLOY_DEV` (UAT/dev) ; tag = uat เธชเธณเธซเธฃเธฑเธ main

## provides
### http (route.go:12-176)
| METHOD | path | auth | evidence |
|--------|------|------|----------|
| GET | /v1 | เนเธกเนเธกเธต | route.go:18 Demo |
| GET | /v1/get_demo | เนเธกเนเธกเธต | route.go:19 |
| POST | /v1/upload_demo | เนเธกเนเธกเธต (เนเธเน HeaderToStuctMockData) | route.go:20 ; demo.go:48 |
| DELETE | /v1/delete_distribute | เนเธกเนเธกเธต | route.go:23 (CloudFront delete) |
| POST | /v1/get_distribute | เนเธกเนเธกเธต | route.go:24 |
| PUT | /v1/disable_distribute | เนเธกเนเธกเธต | route.go:25 |
| POST | /v1/dashboard/:campaign_id/:ads_id | เนเธกเนเธกเธต (header "Key" เธ–เนเธฒ SERVICE_ADS set) | route.go:27 ; dashboard.go:2258,2266 |
| POST | /v1/raw_data, /v1/raw_data_ads, /v1/turnover_data, /v1/campaign | เนเธกเนเธกเธต (header "Key" เธ–เนเธฒ SERVICE_ADS set) | route.go:28-31 ; dashboard.go:2304,2372,2445,2517 |
| GET | /v1/get_withdraw | เนเธกเนเธกเธต | route.go:33 ; dashboard.go:7889 |
| POST | /v1/re_withdraw | เนเธกเนเธกเธต | route.go:34 ; dashboard.go:7933 |
| POST | /v1/reset_statement, /v1/reset_statement_campaign | เนเธกเนเธกเธต | route.go:37-38 ; restat.go:16,49 |
| GET | /landing/prefix/:service | เนเธกเนเธกเธต (query hid) | route.go:43 ; prefix.go:403 |
| POST | /landing/summary_report/:key | เนเธกเนเธกเธต (decode base64 key) | route.go:44 ; dashboard.go:2579 |
| POST | /emp/login, /emp/login_username, /emp/register | เนเธกเนเธกเธต (login) | route.go:50-52 ; member.go:19,133,271 |
| POST | /emp/register_line | AuthJWT(1) | route.go:53 |
| POST | /emp/password | AuthJWT(1) | route.go:54 ; member.go:180 |
| GET | /emp/member | AuthJWT(1) | route.go:55 |
| POST | /emp/logout | เนเธกเนเธกเธต | route.go:56 |
| POST | /emp/member_affiliate, /emp/member_statement_affiliate | **เนเธกเนเธกเธต auth** | route.go:59-60 ; dashboard.go:4115,4156 |
| POST/PUT | /emp/dashboard (+ /:campaign_id, /:campaign_id/:ads_id, /dashboard_report/...) | AuthJWT(3) | route.go:63-68 |
| POST | /emp/finish_campaign, /history_cost_ads, /create_cost_ads | AuthJWT(3) | route.go:70-72 |
| POST | /emp/create_campaign(_many), /edit_campaign(_status), /edit_ads, /create_ads(_many) | AuthJWT(3) | route.go:74-80 |
| POST | /emp/create_ads_batch | AuthJWT(3) | route.go:83 (batch 20k/req) |
| GET | /emp/ads_collection (+ /export_excel, /summary/:campaign_id) | AuthJWT(3) | route.go:84-86 |
| POST | /emp/ads_collection/create_index | AuthJWT(25) | route.go:87 |
| POST | /emp/re_hydra_statement, /re_hydra_campaign_statement (+ reset_then_rehydra) | AuthJWT(3) | route.go:89-91 ; dashboard.go:6480 |
| GET | /emp/re_hydra_campaign_statement/status/:campaign_id | AuthJWT(3) | route.go:92 |
| POST | /emp/highest_deposit | AuthJWT(3) | route.go:93 |
| GET | /emp/check_demo | เนเธกเนเธกเธต | route.go:94 ; dashboard.go:7507 |
| GET | /emp/web_service/:action, /department, /list_menu, /employee(/:id), /sub_employee | AuthJWT(3) | route.go:97-102 |
| PUT/POST/DELETE | /emp/employee(_active) | AuthJWT(3) | route.go:103-106 |
| GET/PUT/POST | /emp/setting_role (+/:code, /find/:id) | AuthJWT(3) | route.go:109-113 |
| GET/PUT/POST/DELETE | /emp/platform(/all) | AuthJWT(3) | route.go:116-120 |
| POST/PUT/GET | /emp/template(/:id), /template_link, /template_link_campaign | AuthJWT(3) | route.go:123-130 |
| POST/GET/PUT/DELETE/PATCH | /emp/domains(...), /campaigns/:service/:campaign_id/domains... | AuthJWT(3) | route.go:133-140 |
| GET | /emp/member_log | AuthJWT(3) | route.go:143 |
| POST/DELETE | /emp/list_banner, /banner, /set_banner | AuthJWT(3) | route.go:146-149 |
| GET/POST/PUT/PATCH/DELETE | /emp/article/:groupName/:serviceName(...) | AuthJWT(3) | route.go:152-161 |
| GET | /emp/agent_type | **เนเธกเนเธกเธต auth** | route.go:164 |
| DELETE | /redis/:redisKey | **เนเธกเนเธกเธต auth** | route.go:167-168 ; redis.go:780 |
| GET | /emp/login_line | เนเธกเนเธกเธต | route.go:171 ; member.go:119 |
| POST | /external/create_cloudfront | **เนเธกเนเธกเธต auth** | route.go:174-175 ; template.go:630 |

### consume
| exchange/queue | handler | evidence |
|---|---|---|
| เนเธกเนเธเธ (เนเธกเนเธกเธต RabbitMQ/AMQP โ€” go.mod เนเธกเนเธกเธต amqp) | - | go.mod:5-24 |

### cron
| schedule | what | evidence |
|---|---|---|
| เนเธกเนเธเธ scheduler เธเธฃเธฐเธเธณ (เนเธกเนเธกเธต gocron/cron import) | เธเธฒเธเธซเธเธฑเธเธ—เธณเนเธเธ `go ...()` async เธ•เธญเธเน€เธฃเธตเธขเธ API: `go hydraReStat` (restat.go:45), `go hydraReStatByCampaign` (restat.go:79) | restat.go:45,79 |

## consumes
### http-out
| env key | base URL | paths | evidence |
|---|---|---|---|
| `CHANNEL_ID`/`CHANNEL_SECRET` | https://api.line.me | /oauth2/v2.1/token (loginService.go:144,212), /v2/profile (loginService.go:155,223) | LINE Login (OAuth) เนเธฅเธ codeโ’tokenโ’profile |
| `obj.SystemURL` (เธเธฒเธ database_config field `system_url`) ๐ก runtime | <SystemURL> | /api/prefix_v2 (prefix.go:443), /api/game/category_game (prefix.go:596), /api/bank_code_list (prefix.go:625) | เธ”เธถเธ config/เน€เธเธก/เธเธเธฒเธเธฒเธฃเธเธญเธเน€เธงเนเธ tenant |
| (เธฎเธฒเธฃเนเธ”เนเธเนเธ”, runtime AWS) | cloudfront.amazonaws.com / S3 ap-southeast-1 | เธชเธฃเนเธฒเธ/เธฅเธ/เธ”เธน CloudFront distribution, PutObject/ListObjects S3 bucket `hydra-landing` | awsService.go:20-175,327,487 |

### publish
| exchange/queue | payload | evidence |
|---|---|---|
| เนเธกเนเธเธ | - | เนเธกเนเธกเธต publisher |

### data
| DB | collection | R/W | evidence |
|---|---|---|---|
| Mongo (OFFICE=MONGODB_DB_NAME) | database_config | R (`status:1`) เธชเธฃเนเธฒเธ connection เธ•เนเธญ tenant | _config/db.go:64 ; demo.go:79,254 ; prefix.go:429 |
| Mongo (per-tenant `resource[service]`) | campaign | R/W ($inc traffic_click adv_list) | prefix.go:455,483 ; dashboard.go (54 เธเธธเธ”) ; restat BulkUpdate |
| | ads_statement | R/W/upsert ($inc traffic_click/deposit/turnover/winloss) | prefix.go:510,518 ; restat.go:419,671,942,1671 ; dashboard.go |
| | ads_statement_delete | W (archive เธเนเธญเธ re-stat) | restat.go:254,517,739 |
| | ads_collection | R/W (batch ads, เธชเธฃเนเธฒเธ index) | prefix.go:468,473 ; dashboard.go ; route.go:83-87 |
| | self_stat_realtime / self_stat | R (เธเนเธญเธกเธนเธฅเธ”เธดเธเธเธณเธเธงเธ“ stat) | restat.go workerPool.go:743 ; restat.go process() |
| | campaign / deposit_statement / withdraw_statement | R/W (เธเธณเธเธงเธ“ affiliate, re_withdraw mark is_recal) | dashboard.go:7933-7988 ; route.go:33-34 |
| | member_account | R (affiliate username/hydra_id) | dashboard.go:4138 ; restat.go member loop |
| | account_office (OFFICE) | R/W (เธเธเธฑเธเธเธฒเธ, login, set username/password, index unique key_register) | member.go:75,107,209,249 ; demo.go:143,167,198 ; loginService.go:107,166,238 |
| | setting_role / setting_menu / setting_dep / setting_platform | R/W (เธชเธฃเนเธฒเธ default เธ•เธญเธ boot CreateDefault) | demo.go:132-360 ; setting.go ; route.go:109-120 |
| | config_system | R (banner/theme) | prefix.go:538 |
| | article_hydra | R/W | prefix.go:645 ; article.go:402 |
| | static_aws | R/W (template + cloudfront status) | demo.go:65,110 ; template.go:680 |
| | hydra_log | W (audit log) | member.go:103 ; rateLimitService.go:98 ; mainService HandleHydraLog |
| | domain / history_cost_ads / report_active_user_day(_month) / link_affiliate / game_group | R/W | route.go:133-140 ; collection tally |

### external
| LINE API / third-party | purpose | auth |
|---|---|---|
| LINE Login (api.line.me OAuth) | เธเธเธฑเธเธเธฒเธ login/register เธ”เนเธงเธข LINE | client_id/secret เธเธฒเธ env `CHANNEL_ID`/`CHANNEL_SECRET` (loginService.go:140-141,208-209); เธชเนเธ code+redirect_uri เนเธฅเธ token (CALLHttpExternal mainService.go:130) |
| AWS S3 (`hydra-landing`, ap-southeast-1) | เธญเธฑเธเนเธซเธฅเธ” landing page/template/เธฃเธนเธ | **AccessKey/Secret เธฎเธฒเธฃเนเธ”เนเธเนเธ”เนเธเธเธญเธฃเนเธช** เธเนเธฒเธ `GetStaticAWS()` (mainService.go:474-483) |
| AWS CloudFront | เธชเธฃเนเธฒเธ/เนเธเน/เธฅเธ distribution เธ•เนเธญ landing domain | static credentials เน€เธ”เธตเธขเธงเธเธฑเธ (awsService.go); `create_cloudfront` เธ เธฒเธขเธเธญเธเนเธเน AccessKey/Secret เธเธฒเธ body (awsService.go:327-330) |
| เน€เธงเนเธ tenant (SystemURL) | prefix/เน€เธเธก/เธเธเธฒเธเธฒเธฃ | เนเธกเนเธกเธต auth header (HttpHandle mainService.go:78) |

## observations
- **AWS credentials เธฎเธฒเธฃเนเธ”เนเธเนเธ”เนเธเธเธญเธฃเนเธช** โ€” `GetStaticAWS()` เธเธทเธ `AWS_ACCESS_KEY_ID: "[REDACTED_AWS_KEY_ID]"` เนเธฅเธฐ `AWS_SECRET_ACCESS_KEY: "[REDACTED_AWS_SECRET]"` (mainService.go:478-479) โ€” เธเธฃเธฃเธ—เธฑเธ” `// return os.Getenv(key)` (481) เธ–เธนเธเธเธญเธกเน€เธกเธเธ•เนเธ—เธดเนเธ ๐ก; เธเธตเธขเน S3 เธเธฃเธดเธเธฃเธฑเนเธงเนเธ repo
- **MongoDB/Redis credentials**: Redis hardcode `Username:"default" Password:"redispw"` (redisService.go:72-73); Mongo เธเธฒเธ env เนเธ•เน db.go pattern เน€เธ”เธตเธขเธงเธเธฑเธ repo เธญเธทเนเธ (root creds เนเธ .env เธเธญเธเธเธฅเธธเนเธก)
- **JWT secret fallback เธญเนเธญเธ**: `getSecretKey()` เธ–เนเธฒ env `SECRET` เธงเนเธฒเธเนเธเน `"secret"` (authJWT.go:41-44); token เธเธเธฑเธเธเธฒเธเธญเธฒเธขเธธ 48 เธเธก. (authJWT.go:59)
- **AuthJWT เธกเธตเธเนเธญเธเนเธซเธงเน privilege**: group==5 (superadmin) เธเนเธฒเธเธ—เธธเธเธญเธขเนเธฒเธเธ—เธฑเธเธ—เธต (authJWT.go:27-28); เธเธฒเธฃเน€เธ—เธตเธขเธ `claims["group"].(float64)` เนเธกเนเน€เธเนเธ type assert ok โ’ panic เนเธ”เนเธ–เนเธฒ claim เธซเธฒเธข; เธ•เธฃเธฃเธเธฐ `< group && group != 0` เนเธเธฅเธ โ€” group เธ—เธตเนเธชเธนเธเธเธงเนเธฒเธเนเธฒเธ เนเธ•เน route เธ—เธตเน AuthJWT(0)/เนเธกเนเธกเธต เธเนเนเธกเนเน€เธเนเธเธชเธดเธ—เธเธดเน
- **endpoint เธเธฒเธฃเน€เธเธดเธ/เธฃเธฒเธขเธเธฒเธเนเธกเน auth**: `/v1/get_withdraw`, `/v1/re_withdraw` (route.go:33-34), `/v1/reset_statement(_campaign)` (37-38), `/emp/member_affiliate`, `/emp/member_statement_affiliate` (59-60), `/emp/agent_type` (164), `/redis/:redisKey` DELETE (167), `/external/create_cloudfront` (174) โ€” เธ—เธฑเนเธเธซเธกเธ” **เนเธกเนเธกเธต middleware** ; `re_withdraw`/`reset_statement` เนเธเนเธขเธญเธ” affiliate เนเธ”เนเนเธ”เธขเนเธกเนเธขเธทเธเธขเธฑเธเธ•เธฑเธงเธ•เธ (เธเธฑเธเนเธเน optional header `Key`==SERVICE_ADS เธ–เนเธฒ env set โ€” dashboard.go:2266,2313)
- **re-stat เธฅเธ-เนเธฅเนเธงเธเนเธญเธขเธเธณเธเธงเธ“ เนเธกเน atomic / เน€เธชเธตเนเธขเธเธเนเธญเธกเธนเธฅเธซเธฒเธข**: hydraReStat archive `ads_statement`โ’`ads_statement_delete` เนเธฅเนเธง `DeleteManyStatement` (restat.go:254-261) เธเนเธญเธเธเธณเธเธงเธ“เนเธซเธกเน; เธ–เนเธฒ process เธ•เธฒเธขเธเธฅเธฒเธเธ—เธฒเธ เธเนเธญเธกเธนเธฅ ads_statement เธซเธฒเธขเธเธฑเนเธงเธเธฃเธฒเธง/เธ–เธฒเธงเธฃ เนเธฅเธฐ traffic_click/deposit เธ–เธนเธ reset
- **affiliate stat เธเธณเธเธงเธ“เธ”เนเธงเธข $inc upsert เธเนเธณเนเธ”เน (double-count)**: `process()` upsert `$inc` turnover/winloss/deposit เธฅเธ ads_statement เธ•เนเธญ self_stat เนเธ•เนเธฅเธฐ record (restat.go:1600-1672); เน€เธฃเธตเธขเธ re-stat เธเนเธณ/เธ—เธฑเธ window เน€เธ”เธตเธขเธงเธเธฑเธเนเธ”เธขเนเธกเนเธกเธต idempotency โ’ เธขเธญเธ”เธเธงเธเธเนเธณ; เธฃเธฑเธเนเธเธ async เธซเธฅเธฒเธข goroutine (`go hydraReStat`) เนเธกเนเธกเธต lock เธเธฑเธเธฃเธฑเธเธเนเธญเธ
- **commission/credit เนเธขเธเธเธฒเธเธขเธญเธ”เธเธฒเธเนเธเธ heuristic เน€เธเธฃเธฒเธฐ**: amountCredit/payBonus/fixWithdraw เธ•เธฑเธ”เธชเธดเธเธเธฒเธ `DepositDetail.Total/Bonus` >0/==0 (restat.go:1605-1618) โ€” เธ–เนเธฒ Bonus เน€เธเนเธเธเนเธฒเธฅเธ/0 เธเธฑเธ”เธเธฃเธฐเน€เธ เธ—เธเธดเธ” เธเธฃเธฐเธ—เธเธขเธญเธ”เธเนเธฒเธข affiliate
- **WorkerPool errors เน€เธเนเธเนเธ•เนเนเธกเน fail เธเธฒเธ**: errs เธชเธฐเธชเธกเนเธฅเนเธง return เน€เธเธขเน (workerPool.go:720-771); processJob recover panic เนเธฅเนเธง return error เนเธ•เน job เธญเธทเนเธเน€เธ”เธดเธเธ•เนเธญ โ’ stat เธเธฒเธเธชเนเธงเธเธซเธฒเธข เน€เธเธตเธขเธ
- **HttpHandle POST `panic(err)` เธ—เธณ request เธ•เธฒเธข**: mainService.go:105 panic เน€เธกเธทเนเธญ client.Do error; เนเธกเนเธกเธต timeout เธเธ client (mainService.go:102,135) โ’ outbound เธเนเธฒเธเนเธ”เน
- **CALLHttpExternal/HttpHandle เนเธกเนเธกเธต timeout** (mainService.go:84,135) ; LINE OAuth call เธเนเนเธกเนเธกเธต
- **CORS เน€เธเธดเธ” `*`** เธ—เธธเธ route (server.go:38-41) เธฃเธงเธก endpoint เธเธฒเธฃเน€เธเธดเธเธ—เธตเนเนเธกเน auth
- **default account seed เธเธฑเธ line_user_id/key_register/role superadmin (level 30)** เนเธ static/model/account_office.json (เน€เธเนเธ key_register=เน€เธเธญเธฃเนเนเธ—เธฃ, password เธงเนเธฒเธ เธเธฃเธฃเธ—เธฑเธ” 494) ; CreateDefault เธชเธฃเนเธฒเธเนเธซเนเธ—เธธเธ boot (demo.go:132,275) โ’ เธเธนเนเธฃเธนเน key_register/line_user_id เธฅเธเธ—เธฐเน€เธเธตเธขเธเธชเธงเธกเธชเธดเธ—เธเธดเนเนเธ”เน
- **port mismatch**: code เธเธฑเธเธเธญเธฃเนเธ• 5053 (server.go:32) เนเธ•เน docker-compose expose 5052 (docker-compose.yml:11) โ€” เน€เธชเธตเนเธขเธ deploy เนเธฅเนเธงเน€เธเนเธฒเนเธกเนเธ–เธถเธ
- **เนเธฅเธเธฃเธฒเธฃเธตเน€เธเนเธฒ**: Gin 1.7.4 (2021), jwt-go v3.2.0 deprecated (เธกเธต CVE), Go 1.16 (go.mod:3)
- **demo/upload endpoints เนเธเน HeaderToStuctMockData()** โ€” เธเธฑเธ user เธเธฅเธญเธก id `621efc9a...` group 3 level 30 (mainService.go:225-233) เนเธเนเธเธฃเธดเธเนเธ `/v1/upload_demo` (demo.go:48) โ’ เธญเธฑเธเนเธซเธฅเธ”เนเธเธฅเนเธเธถเนเธ S3 เนเธ”เนเนเธ”เธขเนเธกเน login
