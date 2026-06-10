# System Archaeology — 3rd-payment, GOTOPOPOFFICE, telesale-service-mootui, topupserie_mootui, TOPUP_OFFICE
**Maximumsoft Co., LTD | วิเคราะห์เมื่อ: 2026-06-09**

---

## 🔴 Critical Findings — ปัญหาวิกฤต

| โปรเจค | หมวด | ปัญหา | คำอธิบาย |
|--------|------|-------|----------|
| GOTOPOPOFFICE | Auth | JWT key อ่อนแอมาก | ใช้ `SHA1(Host + ISO week number)` เป็น signing key — คาดเดาได้, เปลี่ยนทุกวันจันทร์, ทำให้ admin ทุกคน logout พร้อมกันโดยไม่มี grace period |
| GOTOPOPOFFICE | Redis | `DelGame()` ล้าง DB ทั้งหมด | เรียก **`FlushDB()`** แทนที่จะลบ key เฉพาะ — ลบข้อมูลทั้ง Redis DB2 ทันที |
| telesale-service-mootui | Auth | ไม่มี Auth เลยสักตัว | **ทุก route** ใน `/api/v1/*` ไม่มี middleware ป้องกัน — ใครก็เรียกได้จาก internal network |
| topupserie_mootui | Auth | `SECRET_KEY` ว่าง = เข้าได้ทุกคน | ถ้า env `SECRET_KEY` ไม่ได้ตั้งค่า → ผ่าน auth ได้ทุก request — API เปิดโล่ง |
| 3rd-payment | Auth | Webhook ไม่ verify signature | รับ payment callback จาก 30+ provider **โดยไม่ตรวจ HMAC signature** — ปลอม transaction ได้ |
| GOTOPOPOFFICE + topupserie_mootui | Dependency | JWT library มีช่องโหว่ | ใช้ `dgrijalva/jwt-go` v3.2.0 — **CVE-2020-26160**, library ถูก archive แล้ว, ตรวจ claims ไม่ถูกต้อง |

---

## 🟡 High Findings — ปัญหาสำคัญ

| โปรเจค | หมวด | ปัญหา | คำอธิบาย |
|--------|------|-------|----------|
| GOTOPOPOFFICE | Auth | DEMO department bypass | `DeptCode == "DEMO"` ข้ามการตรวจสอบ user ใน DB ไปเลย |
| GOTOPOPOFFICE | Auth | Hardcode LINE user ID ใน auth | ฝัง LINE UID `U875d254f9...` ไว้ใน code เพื่อยกเว้น auth — ไม่ควรอยู่ใน production |
| GOTOPOPOFFICE | Resilience | SUPERCOM_API timeout ไม่ abort request | เมื่อ call ล้มเหลว function return เงียบๆ โดยไม่เรียก `c.Abort()` — request ดำเนินต่อโดยไม่มี user context |
| telesale-service-mootui | Config | Redis DB index ผิดเสมอ | `strconv.Atoi("REDIS_DB")` — แปลง literal string ไม่ใช่ env var → DB index เป็น 0 เสมอ ไม่ว่าตั้งค่าอะไรไว้ |
| telesale-service-mootui | CORS | AllowOrigins เปิดหมด | `AllowedOrigins = *` ทุก environment แม้แต่ production |
| topupserie_mootui | Logging | Hardcode username ใน production log | ชื่อ `"TestRateLimit"` ถูกบันทึกลง `agent_log` ทุกครั้ง — ของที่ทดสอบหลงเข้า production |
| topupserie_mootui | RateLimit | Rate-limit ban คืน HTTP 200 | client แยกไม่ออกระหว่าง "ถูก ban" กับ "สำเร็จ" — monitoring พลาดได้ |
| TOPUP_OFFICE | EOL | Vue 2 หมดอายุแล้ว | Vue 2 หยุด security patch ตั้งแต่ **ธันวาคม 2566** |
| 3rd-payment | Build | go version ผิด | `go 1.26` ใน go.mod — Go 1.26 ไม่มีอยู่จริง, อาจทำให้ CI พัง |

---

## Boundary Map — แผนที่ขอบเขตระบบ

### 3rd-payment (port 8081)
| Protocol | ผู้เรียกเข้ามา (Ingress) | เรียกออกไปหา (Egress) | การยืนยันตัวตน |
|----------|--------------------------|----------------------|----------------|
| HTTP/HTTPS | Internal services + webhook จาก 30+ payment provider | KissPay, BigPay, TrustPay, XPay APIs; RabbitMQ; MongoDB; Telegram Bot API; OTel | Service whitelist บางส่วน; **ไม่มี HMAC บน webhook** |

### GOTOPOPOFFICE (port 7777)
| Protocol | ผู้เรียกเข้ามา (Ingress) | เรียกออกไปหา (Egress) | การยืนยันตัวตน |
|----------|--------------------------|----------------------|----------------|
| HTTP | Admin browser (TOPUP_OFFICE); telesale proxy | MongoDB multi-tenant; Redis DB0+DB2; RabbitMQ; LINE API; SMS kub.com; AWS S3; SUPERCOM_API | JWT HS256 SHA1(Host+week); ตรวจ permission code |

### telesale-service-mootui (port: HTTP_PORT env)
| Protocol | ผู้เรียกเข้ามา (Ingress) | เรียกออกไปหา (Egress) | การยืนยันตัวตน |
|----------|--------------------------|----------------------|----------------|
| HTTP | GOTOPOPOFFICE proxy; ผู้เรียกโดยตรง | MongoDB; Redis; YaleCom VOIP API | **ไม่มีเลย** |

### topupserie_mootui (port: PORT env)
| Protocol | ผู้เรียกเข้ามา (Ingress) | เรียกออกไปหา (Egress) | การยืนยันตัวตน |
|----------|--------------------------|----------------------|----------------|
| HTTP | Customer frontends (web-theme-nextgen, platform_Tangtem) | MongoDB x2; RabbitMQ; Redis; external respap APIs | JWT Bearer หรือ API key (สลับด้วย TYPEAUTH env) |

### TOPUP_OFFICE (port 3000 dev)
| Protocol | ผู้เรียกเข้ามา (Ingress) | เรียกออกไปหา (Egress) | การยืนยันตัวตน |
|----------|--------------------------|----------------------|----------------|
| HTTP | Admin browser | GOTOPOPOFFICE; topupserie_mootui; Firebase | JWT เก็บฝั่ง client ส่งเป็น Authorization header |

---

## Trust Boundaries — ขอบเขตความน่าเชื่อถือ

| ขอบเขต | ระดับความน่าเชื่อถือ | มาตรการที่มี | ช่องโหว่ที่เหลือ |
|--------|---------------------|-------------|-----------------|
| Internet → 3rd-payment | ไม่น่าเชื่อถือ | Service whitelist; OTel; Prometheus | ไม่มี HMAC บน webhook; ไม่มี rate limit; 30+ แหล่งที่ไม่ยืนยัน |
| Internet → topupserie_mootui | น่าเชื่อถือบางส่วน | JWT หรือ API key; Redis rate limit ด้วย CF-IP | `SECRET_KEY` ว่าง = เข้าได้; ban คืน HTTP 200 |
| Admin Browser → GOTOPOPOFFICE | น่าเชื่อถือบางส่วน | JWT + permission code; VPN middleware; dept filter | SHA1 key คาดเดาได้; DEMO bypass; hardcode LINE UID |
| GOTOPOPOFFICE → SUPERCOM_API | น่าเชื่อถือ (internal) | HTTP call; VPN gate | ล้มเหลวโดยไม่ abort; ไม่มี circuit breaker |
| GOTOPOPOFFICE → telesale-service-mootui | น่าเชื่อถือ (internal) | สมมติว่า internal network ปลอดภัย | **telesale ไม่มี auth เลย** — ใครก็เรียกได้ |
| telesale-service-mootui → YaleCom VOIP | น่าเชื่อถือ (3rd party) | API key + Company ID | credentials เก็บ plaintext ใน MongoDB |
| Cloudflare → topupserie_mootui | น่าเชื่อถือ (CDN) | Rate limit ด้วย CF-Connecting-IP | ปลอม CF-Connecting-IP ได้ถ้า origin IP หลุด |
| MongoDB database_config collection | เป้าหมายมูลค่าสูง | TLS ใน production | ถ้าถูกเจาะ → ได้ MongoDB URI และ Redis credentials ทุก tenant |

---

## Top Failure Modes — สถานการณ์ที่ระบบอาจล่ม

| โปรเจค | สถานการณ์ | ผลกระทบ | โอกาสเกิด | ช่องโหว่ |
|--------|-----------|---------|-----------|---------|
| GOTOPOPOFFICE | JWT หมุน key ทุกวันจันทร์เที่ยงคืน | สูง | **แน่นอน** | admin ทุกคน logout พร้อมกัน ไม่มี grace period |
| GOTOPOPOFFICE | เรียก `DelGame()` ผิด environment | วิกฤต | ต่ำ | `FlushDB()` ลบ Redis DB2 ทั้งหมด |
| topupserie_mootui | deploy โดยไม่ตั้ง `SECRET_KEY` | วิกฤต | ต่ำ | API เปิดโล่งทั้งหมด |
| telesale-service-mootui | Redis DB index ผิดตลอด | ปานกลาง | **แน่นอน** | bug ใน config — ใช้ DB0 เสมอ |
| GOTOPOPOFFICE | SUPERCOM_API timeout | สูง | ปานกลาง | ไม่ abort request — ดำเนินต่อโดยไม่มี user |
| 3rd-payment | webhook flood | สูง | ปานกลาง | ไม่มี rate limit; MQ connection ล้มเลิก event หาย |
| topupserie_mootui | RabbitMQ disconnect ระหว่าง publish | สูง | ปานกลาง | event หายช่วง reconnect; ไม่มี publisher confirm |
| telesale-service-mootui | YaleCom API ล่ม | สูง | ปานกลาง | ไม่มี retry, circuit breaker, หรือ fallback |
| 3rd-payment | MongoDB ล่มตอน startup | วิกฤต | ต่ำ | ไม่มี retry; process exit ทันที |

---

## Dead Code — โค้ดที่ไม่ได้ใช้งาน

| โปรเจค | ตำแหน่ง | ชื่อ | หมายเหตุ |
|--------|---------|------|---------|
| 3rd-payment | main.go:31 | `app.StartCronjob()` | Commented out; controller มีอยู่แต่ไม่ถูกเรียกเลย |
| GOTOPOPOFFICE | db/db.go:56 | `Resource.Close()` | method body ถูก comment ออก; connection leak ถ้าเรียก |
| GOTOPOPOFFICE | middlewares/auth.go | `AuthRequired()` | ไม่เคยถูกใช้; ทุกที่ใช้ `AuthRequiredDB()` แทน |
| GOTOPOPOFFICE | helper/k8shelm.go | `k8shelm` package | Kubernetes Helm helper ที่ไม่มีใครเรียกจาก route หรือ controller |
| topupserie_mootui | middleware/rateLimit.go:101 | `"TestRateLimit"` | TODO comment เป็นภาษาไทย: "กลับมาเปิดใช้ด้วย" — ของทดสอบใน production |
| topupserie_mootui | route_apex_pro.go / route_allinone_transfer.go | Conditional routes | ใช้งานได้เฉพาะ `PROJECT` env ตรงกัน — ส่วนใหญ่ dead |
| telesale-service-mootui | route.go:43 | Swagger route | `// router.GET("/docs/*any")` ถูก comment — ไม่มี API docs เลย |

---

## Tech Debt Highlights — หนี้เทคนิคสำคัญ

| ลำดับความสำคัญ | โปรเจค | รายการ |
|---------------|--------|--------|
| 🔴 วิกฤต | GOTOPOPOFFICE + topupserie_mootui | เปลี่ยน `dgrijalva/jwt-go` → `golang-jwt/jwt/v5` (3rd-payment ใช้อยู่แล้ว) |
| 🟠 สูง | telesale-service-mootui | แก้ 1 บรรทัด: `strconv.Atoi("REDIS_DB")` → `strconv.Atoi(os.Getenv("REDIS_DB"))` |
| 🟠 สูง | topupserie_mootui | ลบ `"TestRateLimit"` hardcode; คืนค่า JWT username extraction |
| 🟠 สูง | TOPUP_OFFICE | migrate Vue 2 / Nuxt 2 → Vue 3 / Nuxt 3 (ดู web-theme-nextgen เป็นแบบ) |
| 🟠 สูง | ทุก Go project | ไม่มี test เลยในระบบ financial transaction — ต้องเพิ่มด่วน |
| 🟡 ปานกลาง | GOTOPOPOFFICE | อัพเดต Gin v1.6.3 → v1.11.x |
| 🟡 ปานกลาง | GOTOPOPOFFICE | เปลี่ยน `uber/jaeger-client-go` → OpenTelemetry SDK (3rd-payment ใช้ otelgo อยู่แล้ว) |
| 🟡 ปานกลาง | GOTOPOPOFFICE | `ioutil.ReadFile` → `os.ReadFile` + จัดการ error ไม่ suppress |
| 🟡 ปานกลาง | 3rd-payment | แก้ `go 1.26` → `go 1.21` หรือ `go 1.22` ใน go.mod |
| 🟡 ปานกลาง | TOPUP_OFFICE | `node-sass` → `sass` (dart-sass); ใช้แทนกันได้ ไม่ต้อง build native |
| 🟢 ต่ำ | 3rd-payment | พิจารณาแยก `chromedp` headless browser ออกเป็น microservice แยก |

---

## Key Dependencies at Risk — dependencies ที่มีความเสี่ยง

| โปรเจค | Package | Version | ความเสี่ยง |
|--------|---------|---------|-----------|
| GOTOPOPOFFICE | `dgrijalva/jwt-go` | v3.2.0 | 🔴 CVE-2020-26160, archive แล้ว |
| topupserie_mootui | `dgrijalva/jwt-go` | v3.2.0 | 🔴 CVE-2020-26160, archive แล้ว |
| GOTOPOPOFFICE | `gin-gonic/gin` | v1.6.3 (ปี 2020) | 🟡 ล้าหลัง 6 major version |
| GOTOPOPOFFICE | `uber/jaeger-client-go` | v2.25.0 | 🟡 Deprecated แล้ว |
| TOPUP_OFFICE | `vue` | ^2.6.11 | 🔴 EOL ธันวาคม 2566 |
| TOPUP_OFFICE | `nuxt` | ^2.14.6 | 🟡 ใกล้ EOL |
| TOPUP_OFFICE | `jsonwebtoken` | ^8.5.1 | 🟡 JWT secret อาจรั่วไปใน client bundle |
| 3rd-payment | `chromedp/chromedp` | v0.15.1 | 🟡 dependency หนักสำหรับ QR verification |
