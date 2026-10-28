Agent: Codex; Model: GPT-6

# ADR-0002: เผยแพร่ OrderPlaced ผ่าน RabbitMQ เพื่อแยกงานหลังสั่งซื้อ

- **Status:** Proposed
- **Date:** 2026-10-28
- **Deciders:** ยังไม่ระบุ (ดู Open questions)

## Context and problem

R-12 กำหนดให้ส่งอีเมลยืนยันผ่าน MailFast แทน logic จำลองของ emailservice, ให้ loyaltyservice ใหม่เพิ่มคะแนน และบันทึกเหตุการณ์ “order placed” สำหรับฝ่ายการตลาด; ปีหน้าคาดว่าจะมี SMS, fraud scoring และการปรับ recommendations (docs/scenario.md, “The new requirement (R-12)”). ปัจจุบัน PlaceOrder เรียก sendOrderConfirmation หลังชำระเงินและจัดส่ง แล้วบันทึกคำเตือนหากอีเมลล้มเหลว แต่ยังคืนผลสำเร็จ (system/online-boutique/src/checkoutservice/main.go, PlaceOrder, sendOrderConfirmation).

p95 ของ PlaceOrder ต้อง ≤ 500 ms จากปัจจุบันราว 300 ms; ช่วงสูงสุดมี 200 orders/s. MailFast มี p95 1.5 s, p99 8 s, ล้มเหลวราว 0.5% และจำกัด 50 emails/s. คะแนนต้องเข้าภายใน 1 นาทีและเพิ่มครั้งเดียว; อีเมล คะแนน หรือ analytics ล่มต้องไม่ทำให้คำสั่งซื้อล้มเหลว. ทีมมี 4 คน ถนัด Go/gRPC และ 2 คนเคยใช้ RabbitMQ; ระบบอยู่บน Kubernetes และต้องการ open-source แบบ self-hosted (docs/scenario.md, “Quality goals and constraints”).

## Decision drivers

1. **Availability:** งานหลังสั่งซื้อขัดข้องต้องไม่ทำให้คำสั่งซื้อล้มเหลว (docs/scenario.md, “Quality goals and constraints”).
2. **Performance:** p95 ของ PlaceOrder ≤ 500 ms แม้ MailFast ช้า (docs/scenario.md, “Quality goals and constraints”).
3. **Correctness:** คะแนนภายใน 1 นาทีและเพิ่มครั้งเดียวต่อคำสั่งซื้อ (docs/scenario.md, “Quality goals and constraints”).
4. **Scalability และ modifiability:** รับ 200 orders/s และเพิ่มงานปีหน้าได้ (docs/scenario.md, “The new requirement (R-12)”, “Quality goals and constraints”).
5. **Cost และ operability:** ใช้ทักษะทีมและส่วนประกอบที่รันเองบน Kubernetes (docs/scenario.md, “Quality goals and constraints”).

## Considered options

1. ให้ checkoutservice เรียก MailFast, loyaltyservice และ analytics โดยตรงแบบ synchronous
2. บันทึก OrderPlaced ใน transactional outbox แล้วเผยแพร่ผ่าน RabbitMQ ไปยัง consumer แยกกัน
3. ให้ checkoutservice เริ่มงาน asynchronous ในกระบวนการโดยไม่บันทึกเหตุการณ์ถาวร

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. เรียกตรง | โครงสร้างน้อยและใช้รูปแบบ gRPC ที่ทีมถนัด | MailFast p95 1.5 s เกินงบ checkout 500 ms หากรอผล; หากละเลยข้อผิดพลาด งานอาจสูญหาย; เพิ่มงานต้องแก้ checkout (docs/scenario.md, “Quality goals and constraints”; system/online-boutique/src/checkoutservice/main.go, PlaceOrder) |
| 2. Outbox + RabbitMQ | [Inferred] แยก latency และความล้มเหลวของ consumer; คิวแยกช่วยไม่ให้อีเมลที่ติด rate limit ขวางคะแนน; เพิ่ม subscriber ได้ | ต้องดูแล broker, ที่เก็บ outbox, relay, retry และ dead-letter; ต้องออกแบบขอบเขต commit ใหม่ เพราะ PlaceOrder ไม่แสดง order store หรือ outbox (system/online-boutique/src/checkoutservice/main.go, PlaceOrder) |
| 3. งานในกระบวนการ | checkout ตอบเร็วและไม่ต้องดูแล broker | งานอาจสูญหายเมื่อกระบวนการหยุด; ควบคุม rate limit, retry และการเพิ่ม consumer ได้ยาก; รับประกันคะแนนได้ยาก [Inferred] |

## Decision

เลือก **ตัวเลือก 2**. ออกแบบให้การยอมรับคำสั่งซื้อบันทึกข้อมูลคำสั่งซื้อและ OrderPlaced ในธุรกรรม durable เดียวกันก่อนตอบสำเร็จ จากนั้น relay ส่งไป RabbitMQ ด้วย persistent message, publisher confirmation และส่งซ้ำเมื่อยังไม่ได้รับการยืนยัน. แยก consumer และคิวของ MailFast, loyaltyservice และ analytics. การล่มของแต่ละ consumer จึงไม่ขัดขวางการยอมรับคำสั่งซื้อ [Inferred]. ต้องเพิ่มขอบเขตการบันทึกคำสั่งซื้อ เพราะ PlaceOrder ที่ตรวจได้สร้าง OrderResult แล้วส่งกลับ แต่ไม่แสดงการบันทึก order หรือ outbox (system/online-boutique/src/checkoutservice/main.go, PlaceOrder).

ใช้ OrderId เป็นกุญแจ idempotency: loyaltyservice บันทึกคะแนนและกุญแจนี้แบบ atomic โดยมี unique constraint; acknowledge หลัง commit. ผลคือเพิ่มคะแนนครั้งเดียวแม้ broker ส่งซ้ำ [Inferred]. consumer ของ MailFast จำกัดอัตรารวมไม่เกิน 50 emails/s, retry แบบหน่วงเมื่อผิดพลาด และส่งรายการที่แก้เองไม่ได้ไป dead-letter พร้อมแจ้งเตือน. consumer ของ analytics ใช้คิวของตนเอง. แทนการเรียก emailservice เดิมใน PlaceOrder ด้วย consumer ที่ส่งผ่าน MailFast (system/online-boutique/src/checkoutservice/main.go, sendOrderConfirmation; docs/scenario.md, “The new requirement (R-12)”, “Quality goals and constraints”).

## Consequences

- **Positive:** งาน MailFast ที่ p95 1.5 s ไม่อยู่ในเส้นทางตอบกลับของ PlaceOrder (performance); consumer ล่มไม่ทำให้ checkout ล่ม (availability) [Inferred].
- **Positive:** คิวแยกและ idempotency ทำให้คะแนนไม่ถูกบล็อกโดยอีเมลที่ติด rate limit และรองรับการส่งซ้ำ (correctness, scalability) [Inferred].
- **Positive:** เพิ่มงานปีหน้าด้วย consumer ใหม่ของ OrderPlaced ได้ (modifiability) [Inferred].
- **Negative:** broker และ outbox เพิ่มภาระดูแลของทีม 4 คน (cost, operability); อีเมลจะส่งหลังคำสั่งซื้อสำเร็จ (timeliness) [Inferred].
- **Risks and mitigations:** ที่ 200 orders/s แต่ MailFast ส่งได้ 50 emails/s คิวอีเมลสะสม 150 รายการ/s ตลอดช่วง peak; จำกัดอัตรา วัดอายุคิว แจ้งเตือน และตกลงความสามารถรองรับระยะยาวกับ MailFast (scalability, timeliness) [Inferred; docs/scenario.md, “Quality goals and constraints”]. relay หยุดต้องคงเหตุการณ์ไว้เพื่อส่งซ้ำ; consumer ล่มใช้ retry/dead-letter แยกกัน (availability, recoverability) [Inferred].
- **Risks and mitigations:** โค้ดปัจจุบันชำระเงินและจัดส่งก่อนสร้าง OrderResult; ต้องออกแบบกรณี side effect เกิดแล้วแต่บันทึก order/outbox ไม่สำเร็จ (correctness) [Inferred; system/online-boutique/src/checkoutservice/main.go, PlaceOrder].

## Validation

ทดสอบโหลด 200 orders/s และตรวจ p95 ของ PlaceOrder ≤ 500 ms. ปิด MailFast, loyaltyservice และ analytics ทีละส่วน แล้วตรวจว่าคำสั่งซื้อที่ยอมรับยังสำเร็จและเหตุการณ์รอกู้คืนได้. ส่ง OrderPlaced ซ้ำและตรวจว่ามีคะแนนหนึ่งรายการต่อ OrderId; วัดเวลาจากการยอมรับจนบันทึกคะแนนว่า ≤ 1 นาที รวมกรณี retry. ตรวจ MailFast ≤ 50 emails/s, อายุคิว, dead-letter และการกู้คืน relay/broker (docs/scenario.md, “Quality goals and constraints”).

## Open questions

- ใครเป็นผู้ตัดสินใจและวันที่อนุมัติ? Not evidenced in the provided files.
- จะเพิ่ม order store/transactional outbox ที่ใด และกำหนดจุด commit กับการชำระเงินและจัดส่งอย่างไร? Not evidenced in the provided files (system/online-boutique/src/checkoutservice/main.go, PlaceOrder).
- สูตรคะแนน, ข้อมูลที่ loyaltyservice ต้องใช้, ปลายทาง/สัญญาข้อมูล analytics และข้อกำหนดการเก็บรักษาคืออะไร? Not evidenced in the provided files.
- MailFast รองรับ idempotency key, การตรวจสถานะการส่ง, การเพิ่ม rate limit หรือการชดเชย backlog หรือไม่? Not evidenced in the provided files.
- ระยะเวลา peak และอัตราคำสั่งซื้อเฉลี่ยหลัง peak เท่าไร เพื่อพิสูจน์ว่าคิวอีเมลระบายได้? Not evidenced in the provided files.
