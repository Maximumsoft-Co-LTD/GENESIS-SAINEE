# Risk Checklist — สิ่งที่ต้องไล่ดูจริงในโค้ด (Phase 3)

ใช้ร่วมกับ `observations` ดิบจาก interface card ของแต่ละ repo — `observations` คือสิ่งที่บังเอิญเห็นตอนอ่าน, checklist นี้คือสิ่งที่ต้อง**ตั้งใจไปหา** ทุกหมวด เพื่อให้ "list risk ทั้งหมด" ครบจริงไม่ใช่แค่ที่สะดุดตา

แต่ละข้อที่เจอ ให้บันทึกพร้อม `file:line`, ระดับ (🔴/🟠/🟡), conf (🟢/🟡/⚪), ผลกระทบ, ข้อเสนอ — ถ้าไล่ดูแล้ว**ไม่เจอ**ปัญหา ให้ลง section "✅ ตรวจแล้วผ่าน" พร้อมหลักฐานว่าเขาทำถูกที่ไหน (สำคัญพอกัน เพราะกันการเสนอแก้ของที่ไม่พัง)

## 🔐 Security (มักเป็น 🔴)

- **Secret/credential ติด repo** — API key, DB password, JWT secret, private key hardcode ในโค้ดหรือ `.env` ที่ commit เข้า git (เช็ค `git log` ของไฟล์ env ด้วย)
- **JWT/session secret ใช้ซ้ำข้าม service** — ถ้า `ACCESS_SECRET` ตัวเดียวกันหลาย service แปลว่า service หนึ่งรั่ว = ปลอม token ได้ทั้งระบบ
- **Crypto อ่อน** — MD5/SHA1 สำหรับ password, AES-ECB, IV คงที่, `math/rand` สำหรับ token
- **Endpoint ธุรกรรม/admin ที่ไม่มี auth middleware** — ไล่ดูทุก route ว่ามี `AuthRequired` จริง โดยเฉพาะจุดที่ขยับเงิน/แก้สิทธิ์
- **Callback/webhook ที่ไม่ verify signature** — payment provider callback ที่เชื่อ payload ดิบ = ปลอมยอดได้
- **CORS เปิดกว้าง / `*`** บน endpoint ที่มี credential
- **SQL/NoSQL injection** — query ที่ต่อ string จาก input ตรงๆ
- **IDOR** — เข้าถึง resource ด้วย id จาก input โดยไม่เช็คว่าเป็นของ user นั้น

## ⚙️ Reliability (มักเป็น 🟠)

- **HTTP client ไม่มี timeout** — `http.Client{}` เปล่าๆ / axios ไม่มี timeout → ปลายทางค้าง = เราค้างตาม → ลามทั้ง chain
- **ไม่มี retry / circuit breaker** บน call ข้าม service ที่สำคัญ
- **Consumer ไม่มี ack/nack/DLQ ที่ถูก** — RabbitMQ/Kafka consumer ที่ ack ก่อนทำงานเสร็จ = ข้อความหายเมื่อ crash ; ไม่มี DLQ = poison message วน
- **ไม่มี idempotency บน callback/retry** — callback ซ้ำแล้วเครดิตเงินซ้ำ (เช็คว่ามี key กันซ้ำจริงไหม ที่ file:line)
- **Distributed lock หาย/ไม่ release** — update balance พร้อมกันโดยไม่มี lock/transaction
- **จุด crash แล้วลามทั้งกลุ่ม** — sync chain ยาวที่ไม่มี fallback

## 🗄️ Data (🔴–🟠)

- **หลาย service เขียน collection/table เดียวกัน** — ใครเป็นเจ้าของ schema? เปลี่ยน field แล้วใครพัง?
- **ย้ายเงิน/balance นอก transaction** — debit สำเร็จ credit fail = เงินหาย
- **Race condition** ตอน read-modify-write ยอดเงิน/สต็อก พร้อมกัน
- **ไม่มี validation ที่ชั้น data** — required/unique/enum ที่ควรบังคับแต่ไม่บังคับ
- **Cache invalidation ผิด** — TTL ยาวเกินบนข้อมูลที่เปลี่ยนบ่อย / cache ไม่ล้างเมื่อ update → ข้อมูลเพี้ยน

## 🔧 Maintainability (มักเป็น 🟡 แต่สะสมเป็นต้นทุนใหญ่)

- **โค้ด/logic ซ้ำข้าม repo** — JWT verify, response formatter, payment callback handler, crypto helper ที่ลอกกันหลาย repo → แก้บั๊กที่เดียวอีกหลายที่ค้าง (นี่มักกลายเป็น pain point เชิงโครงสร้างใน Phase 4)
- **Dependency เก่า/มีช่องโหว่** — version ที่ EOL หรือมี CVE
- **ไม่มี test เลย** บน logic ที่ขยับเงิน
- **Dead code / endpoint ตาย** ที่หลอกคนอ่านว่ายังใช้
- **God file/function** — ไฟล์พันบรรทัด, function ที่ทำสิบเรื่อง

## 👁️ Observability (🟠 — เพราะทำให้ risk อื่น "เงียบ")

- **Error ถูกกลืน** — `catch {}` ว่าง, `if err != nil { }` ที่ไม่ทำอะไร, return nil ทับ error
- **จุดเงิน/ข้อมูลสำคัญไหลผ่านที่ไม่มี log/trace** — เงินหายแล้ว trace ไม่ได้ว่าหายตรงไหน
- **ไม่มี correlation/trace id ข้าม service** — debug flow ข้าม service ไม่ได้
- **Log ใส่ข้อมูลอ่อนไหว** — log password/token/เลขบัตร

---

**Tip การจัดระดับ:** ถามว่า "ถ้าข้อนี้เกิดจริงตอนตี 3 จะเสียอะไร" — เงินหาย/โดน exploit = 🔴, ระบบล่ม/ข้อมูลเพี้ยนชั่วคราว = 🟠, ยังไม่เจ็บแต่ทำให้ทุกอย่างช้าลงเรื่อยๆ = 🟡 ตัวคูณ: ถ้าจุดนั้น**มีเงินไหลผ่าน** ยกระดับขึ้นหนึ่งขั้น
