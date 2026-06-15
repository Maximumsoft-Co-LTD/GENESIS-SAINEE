# Target Architecture Patterns — catalog + วิธีเลือก (Phase 5)

จุดประสงค์: ไม่ใช่ให้ลอก pattern แต่ให้ "รู้จักทางเลือกที่มีจริง" แล้วเลือกแบบที่ trade-off ตรงกับ **Architecture Drivers** จาก Phase 4 ที่สุด ทุกข้อเสนอต้องโยงกลับว่าแก้ pain point ข้อไหน

## วิธีให้คะแนน (ใช้ปิดท้าย Phase 5)

ทำตารางคะแนน: แถว = ทางเลือก, คอลัมน์ = drivers จาก Phase 4 (ให้น้ำหนักตามที่ user สนใจ) ให้คะแนนแต่ละช่อง + เหตุผลสั้น แล้วเลือกตัวที่คะแนนถ่วงน้ำหนักสูงสุด **โดยดู constraints เป็นตัวตัด** (ทีมเล็ก/timeline สั้น = ตัดแบบที่ต้อง rewrite ใหญ่ออกแม้คะแนน "สวย" จะสูง)

driver ที่พบบ่อย: ลดเวลา dev ฟีเจอร์ใหม่ · เงินต้องไม่หาย/ปลอดภัย · รองรับ tenant/โหลดเพิ่ม · ลดต้นทุน ops · ทีมเล็ก/skill จำกัด · ห้ามดาวน์ไทม์ตอนย้าย

---

## Patterns

### A. Modular Monolith (รวมเข้า)
รวมหลาย service เล็กที่คุยกันถี่เป็น process เดียว แบ่ง module ภายในตาม domain (ขอบเขตชัดด้วย package/interface ไม่ใช่ network)
- **เหมาะเมื่อ:** ทีมเล็ก, service ส่วนใหญ่ deploy/scale พร้อมกันอยู่แล้ว, latency/ความเปราะจาก sync chain ยาวเป็นปัญหาหลัก, micro-overhead (network, dup code, ops) มากกว่าประโยชน์ที่ได้
- **แก้:** SPOF จาก chain ยาว, โค้ดซ้ำ (เรียก function แทน HTTP), ops หลาย service
- **เสี่ยง/เสีย:** scale แยกส่วนไม่ได้, deploy ใหญ่ขึ้น, ถ้าไม่คุมขอบเขต module จะกลายเป็น big ball of mud

### B. Service Grouping ตาม Domain (ยุบให้เหลือน้อยลง)
ไม่รวมหมดแต่ยุบ N service จิ๋วให้เหลือไม่กี่ service ตาม bounded context (เช่น wallet+deposit+withdraw → "money service")
- **เหมาะเมื่อ:** service ถูกแบ่งถี่เกินไปจน domain เดียวกระจายหลายที่, แต่ยังต้องการ scale/deploy บางส่วนแยก
- **แก้:** ขอบเขตไม่ตรง domain, shared DB (ให้ service เจ้าของ domain เป็นเจ้าของข้อมูล), call ข้าม service ที่จริงๆ ควรเป็น in-process
- **เสี่ยง/เสีย:** ต้องตกลง bounded context ให้ชัดก่อน ไม่งั้นย้ายผิดเส้น

### C. Shared Platform / Internal Library (แก้ของซ้ำ โดยไม่ขยับ topology)
คงจำนวน service เดิม แต่ดึงของที่ลอกกัน (auth/JWT, response helper, observability, http client มี timeout, payment callback) ออกเป็น lib/sidecar/gateway กลาง
- **เหมาะเมื่อ:** topology โอเคแล้ว ปัญหาหลักคือความไม่สม่ำเสมอ + โค้ดซ้ำ + แก้ที่เดียวอีกที่ค้าง
- **แก้:** maintainability, security ที่ทำคนละแบบ, observability ไม่ทั่ว — เป็น quick-win เชิงโครงสร้าง effort ต่ำสุด
- **เสี่ยง/เสีย:** lib กลางกลายเป็น coupling แบบใหม่ (เปลี่ยน lib = ทุก service ต้อง bump), versioning ต้องคุม

### D. Event-Driven Backbone (ลด coupling sync)
แทน sync chain ยาวด้วย event/queue เป็นแกนกลาง — producer ปล่อย event, consumer ทำงานต่อแบบ async + idempotent
- **เหมาะเมื่อ:** sync chain ยาวเปราะ, ต้องการ decouple, มี flow ที่ไม่ต้องตอบ real-time (notify, report, settlement)
- **แก้:** SPOF จาก chain, latency สะสม, reliability (retry/DLQ ที่ broker)
- **เสี่ยง/เสีย:** eventual consistency, ต้องลงทุน idempotency + observability หนัก ไม่งั้น debug ยากกว่าเดิม, อย่าใช้กับ flow ที่ต้อง strong consistency (ยอดเงิน real-time)

### E. API Gateway / BFF (จัดระเบียบทางเข้า)
รวมทางเข้าเป็น gateway เดียว (auth, rate limit, routing, CORS รวมศูนย์) — มัก combine กับ A–D
- **เหมาะเมื่อ:** frontend หลายตัวเรียก service ตรงๆ กระจาย, auth/CORS ทำคนละแบบทุก service
- **แก้:** security ไม่สม่ำเสมอ, endpoint เปิดตรงโดยไม่ผ่าน policy กลาง

---

## วิธีย้าย (ใส่ใน roadmap เสมอ — ลดความเสี่ยงตอนเปลี่ยน)

- **Strangler-fig** — ห่อระบบเดิม ค่อยๆ ดึงทีละ flow มาที่ของใหม่ จนของเดิมไม่เหลืออะไร (default สำหรับ migration ที่ห้ามดาวน์ไทม์)
- **Dual-write / shadow** — เขียนทั้งของเก่า+ใหม่ขนานกัน เทียบผลก่อนตัดสวิตช์ (เหมาะตอนย้าย data store/เจ้าของข้อมูล)
- **Feature flag / canary** — ปล่อยทีละ % ของทราฟฟิก ถอยได้เร็ว
- **Branch by abstraction** — แทรก interface คั่นก่อน แล้วสลับ implementation ข้างหลัง

**กฎเรียงลำดับเฟส:** quick win + risk 🔴 ที่แก้ถูกแก้ง่ายมาก่อน → ทุบ SPOF/ความเปราะ → ค่อยจัดระเบียบ/รวม/แยก → ของที่แค่สวยขึ้นไว้ท้าย ทุกเฟสต้องปล่อยจริงได้เองโดยไม่รอเฟสถัดไป
