# ADR-0002: เผยแพร่ OrderPlaced ผ่าน RabbitMQ

- **Status:** Proposed
- **Date:** 2026-10-28
- **Deciders:** [รอยืนยัน]

> AI ช่วยแก้ร่างวันที่ 2026-10-29; ยังรอผู้เรียนตรวจและตัดสินใจ

## Context and problem

R-12 ต้องส่งอีเมลผ่าน MailFast, ให้ `loyaltyservice` เพิ่มแต้มครั้งเดียวภายใน 1 นาที และบันทึก analytics โดยงานเหล่านี้ล่มแล้ว order ต้องไม่ล้มเหลว (`docs/scenario.md`, R-12 และ Quality goals and constraints). `PlaceOrder` ปัจจุบันสร้าง UUID ใหม่ทุกครั้ง เรียก payment แล้ว shipping ก่อนสร้าง `OrderResult` และส่งอีเมลแบบ gRPC; ไม่เห็นการบันทึก order หรือ outbox (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`). `PlaceOrderRequest` ไม่มี request key และ `ChargeRequest` ไม่มี idempotency key (`system/online-boutique/protos/demo.proto`, `PlaceOrderRequest`, `ChargeRequest`). จึงยังรับประกันไม่ได้ว่าการ retry จะไม่จ่ายเงินหรือจัดส่งซ้ำ

เป้าหมาย p95 ของ `PlaceOrder` ≤ 500 ms (ปัจจุบันราว 300 ms), peak 200 orders/s; MailFast p95 1.5 s, ล้มเหลวราว 0.5% และจำกัดรวม 50 emails/s. ทีมมี 4 คน สองคนเคยใช้ RabbitMQ; ใช้ Kubernetes และต้องการ open-source แบบ self-hosted (`docs/scenario.md`, Quality goals and constraints)

## Decision drivers

1. เก็บการยอมรับ order และงานหลังสั่งซื้อไม่ให้สูญหายโดยไม่ให้ consumer ล่มทำ order ล้มเหลว (availability, correctness)
2. รักษา p95 checkout ≤ 500 ms และให้แต้มภายใน 1 นาทีโดยไม่ซ้ำ (performance, correctness)
3. เพิ่มงานหลังสั่งซื้อในอนาคตได้ แต่จำกัดภาระปฏิบัติการของทีม (modifiability, operability)
4. ไม่ส่ง MailFast เกิน 50 emails/s และติดตาม backlog ได้ (scalability, operability)

## Considered options

1. ให้ `PlaceOrder` เรียกงานหลังสั่งซื้อตรงแบบ synchronous
2. บันทึก order กับ transactional outbox แล้ว relay `OrderPlaced` ผ่าน RabbitMQ (ตัวเลือกที่เสนอ)
3. publish เข้า durable RabbitMQ โดยตรงพร้อม publisher confirm หลัง checkout
4. เริ่มงาน asynchronous ใน process โดยไม่มีงานค้างถาวร

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. เรียกตรง | ใช้รูปแบบ gRPC ที่ทีมถนัด | MailFast p95 1.5 s เกินงบ checkout 500 ms ถ้ารอผล; consumer ล่มอาจกระทบ order (`docs/scenario.md`, Quality goals and constraints) |
| 2. Outbox + RabbitMQ | [Inferred] commit order กับเหตุการณ์ได้พร้อมกัน; แยก retry และเพิ่ม consumer ได้ | ต้องเพิ่ม durable order store, relay, broker และการดูแล; ยังต้องแก้ช่องว่างของ payment/shipping ภายนอกธุรกรรม |
| 3. Publish ตรงพร้อม confirm | มี broker แต่ไม่ต้องทำ relay | [Inferred] ผล payment/shipping, order record และ broker ไม่ commit พร้อมกัน; crash ระหว่างขั้นทำให้เหตุการณ์หายหรือซ้ำได้ |
| 4. งานใน process | ส่วนประกอบใหม่เริ่มต้นน้อย | [Inferred] crash ทำให้งานค้างหาย; คุม retry และแต้มภายใน 1 นาทียาก |

## Decision

**เสนอเลือกตัวเลือก 2 โดยมีเงื่อนไขก่อนใช้จริงดังนี้**

1. **จุดยอมรับ order:** เพิ่ม durable order record กับ client `requestId` ใหม่ที่คงเดิมเมื่อ retry; unique key ต้องผูกกับข้อมูลคำขอที่ไม่เปลี่ยน เช่น user, รายการสินค้า, ยอด และที่อยู่ โดยไม่เก็บเลขบัตร. Key เดิมแต่ข้อมูลต่างกันให้ปฏิเสธ. Commit `READY_TO_CHARGE` ก่อน side effect; เปลี่ยนสถานะเป็น `CHARGE_ATTEMPTED` แบบ atomic ก่อนส่งคำขอ แล้วบันทึก `PAYMENT_RECORDED`; ทำเช่นเดียวกันกับ `SHIP_ATTEMPTED` และ `SHIPPING_RECORDED`. สถานะ `ATTEMPTED` ที่ crash/timeout อาจมีผลภายนอกแล้ว จึงต้อง reconcile ก่อนดำเนินต่อ. เมื่อยืนยันทั้งสองผล จึง commit `ACCEPTED` กับ outbox `OrderPlaced` ในธุรกรรมเดียวก่อนตอบสำเร็จ. Retry key เดิมที่ `READY_TO_CHARGE` หรือ `PAYMENT_RECORDED` ก่อน `SHIP_ATTEMPTED` เริ่มขั้นที่ยังไม่เคยส่งได้ด้วย atomic claim; ที่ `ATTEMPTED` ต้องตรวจผลก่อน ไม่เรียกซ้ำ. API สำหรับสถานะ pending เป็นข้อเสนอใหม่ [Inferred; `system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`; `system/online-boutique/protos/demo.proto`, `PlaceOrderRequest`]
2. **ช่องว่างหลัง side effect:** ก่อนเรียก charge/ship ต้องมี durable record ที่เก็บ `orderId`, `requestId`, ขั้นที่กำลังทำ และเวลา attempt ในสถานะ `CHARGE_ATTEMPTED` หรือ `SHIP_ATTEMPTED`. ถ้า side effect สำเร็จแต่บันทึกผลหรือ outbox ล้มเหลว ให้คงสถานะ `ATTEMPTED` ที่บันทึกไว้เป็นหลักฐานกู้คืน หยุด retry side effect อัตโนมัติ และแจ้งเตือน; เมื่อ storage กลับมา จึงบันทึกรหัสธุรกรรม/ติดตามที่ทราบกับ `NEEDS_RECONCILIATION` ได้. ถ้าไม่ทราบผลก็ใช้ทางตรวจสอบเดียวกัน. ผู้ดูแลต้องเทียบกับข้อมูลของผู้ให้บริการแล้วบันทึกผลที่ยืนยันได้เพื่อเดินขั้นต่อ; หากยืนยันไม่ได้ให้คงสถานะค้างและส่งต่อผู้รับผิดชอบ ไม่ตอบว่า order สำเร็จ. ต้องเพิ่มหรือยืนยันความสามารถใช้ `orderId` เป็น idempotency key และตรวจผลย้อนหลังกับ payment/shipping; API ปัจจุบันไม่แสดงสิ่งนี้. หากยังทำไม่ได้ ต้องมีขั้นตอนตรวจหลักฐานจากผู้ให้บริการและแก้ด้วยคนก่อนเปิดใช้; refund/cancel เป็นข้อเสนอใหม่ที่ต้องตกลง ไม่ใช่ความสามารถที่มีแล้ว (`main.go`, `chargeCard`, `shipOrder`; `demo.proto`, `ChargeRequest`, `ShipOrderRequest`)
3. **การส่ง broker:** ตั้ง exchange, คิว email/loyalty/analytics และ retry/dead-letter queues แบบ durable; ตรวจ routing keys, bindings และสิทธิ์แต่ละคิวก่อนใช้. สร้าง delivery record แยกต่อปลายทางจาก `OrderPlaced` เดียว แล้ว relay ส่ง persistent message ด้วย publisher confirms และ mandatory routing. Mark delivery นั้นเป็น `SENT` หลัง broker confirm ACK และไม่มีการคืนข้อความว่า route ไม่ได้เท่านั้น; NACK, timeout, binding ขาดหรือ unroutable ให้คงงานไว้ retry/แจ้งเตือน. Crash หลัง confirm แต่ก่อน mark ส่งซ้ำได้ จึงให้ consumer รองรับ duplicate [Inferred]
4. **consumer:** `loyaltyservice` ที่ต้องสร้างใหม่ใช้ `orderId` unique และ commit การเพิ่มแต้มกับ dedup record ในธุรกรรมเดียวก่อน ACK; analytics บันทึก `eventId` unique กับ event ก่อน ACK. MailFast เป็น side effect ภายนอกที่ commit พร้อม ledger ไม่ได้: เก็บสถานะ attempt และ ACK เมื่อบันทึกผลสำเร็จแล้ว; หาก timeout หรือส่งสำเร็จแต่บันทึกผลไม่สำเร็จ ให้ถือว่าผลไม่ชัด; ห้ามอ้างว่า retry จะไม่ส่งซ้ำ. ตรวจสถานะ/idempotency ของ MailFast หากมี หรือส่งเข้าคิว reconcile แทนการ retry โดยไม่ตรวจ. ความสามารถนั้นยังไม่ปรากฏในหลักฐาน [Inferred; `docs/scenario.md`, R-12]
5. **เวลาและอัตรา:** ADR นิยาม `ACCEPTED` ที่บันทึกก่อนตอบสำเร็จเป็นจุด “order placed” และเริ่มนับ 1 นาทีตรงนี้; ต้องให้ธุรกิจยืนยันนิยามนี้ ไม่เลื่อนจุดเริ่มไปหลังระบบกลับมา. วัดเวลารวม outbox, relay, queue และ loyalty. หาก loyalty ล่มเกิน 1 นาที จะรักษาทั้งเงื่อนไขแต้มภายใน 1 นาทีและ checkout ไม่ล้มเหลวพร้อมกันไม่ได้; บันทึก breach แจ้งเตือน และตกลงข้อยกเว้นหรือเปลี่ยน requirement กับธุรกิจ. จำกัด MailFast รวมทุก consumer/replica และ retry ≤ 50 emails/s (`docs/scenario.md`, Quality goals and constraints)

## Consequences

- **Positive:** MailFast และ consumer อื่นออกจากเส้นทางตอบ `PlaceOrder`; การล่มของ consumer ไม่ทำ order ที่บันทึกแล้วล้มเหลว (performance, availability) [Inferred]
- **Positive:** outbox และ dedup รองรับการส่งซ้ำโดยไม่เพิ่มแต้มซ้ำ; เพิ่ม consumer ใหม่ได้ (correctness, modifiability) [Inferred]
- **Negative:** การเขียน order ก่อนตอบและการดูแล broker/relay/ledger เพิ่ม latency, ค่าใช้จ่าย และงาน on-call (performance, cost, operability) [Inferred]
- **Risks and mitigations:** ที่ peak 200 orders/s อีเมลเข้าเร็วกว่าขีดจำกัด 50/s อย่างน้อย 150 รายการ/s; retry ใช้ quota เดียวกันและทำให้ระบายช้าลง. หลัง peak คิวจะลดได้ก็ต่อเมื่ออัตรา order ใหม่รวม retry ต่ำกว่า 50/s. ก่อนรับรองต้องทราบระยะ peak, ขนาดข้อความ, พื้นที่เก็บที่มี และเวลาระบายคิวที่ธุรกิจยอมรับ; ใช้ค่าเหล่านี้กำหนดเกณฑ์ผ่าน load test/พื้นที่สำรอง. เฝ้าดู queue depth, oldest age, retry/dead-letter และ disk พร้อมแจ้งเตือนก่อนเต็ม โดยไม่สมมติว่าเพิ่ม quota ได้ (scalability, timeliness; `docs/scenario.md`, Quality goals and constraints)
- **Risks and mitigations:** ความสามารถ retry/ตรวจสถานะ payment และ shipping ยังไม่ปรากฏใน API ที่ให้มา; ห้ามเปิดใช้การ retry อัตโนมัติจนแก้หรือมีขั้นตอน reconcile ที่พิสูจน์ได้ (correctness, availability; `demo.proto`, `ChargeRequest`, `ShipOrderRequest`)

## Validation

แผนทดสอบก่อนรับรอง ไม่ใช่ผลทดสอบ: ยิง 200 orders/s ตามระยะ peak ที่ธุรกิจยืนยัน แล้ววัด p95 checkout ≤ 500 ms, พื้นที่เก็บและเวลา drain ตามเกณฑ์ที่ตกลง. ทดสอบ key เดิม payload เดิม/ต่างกัน, crash/timeout ก่อนและหลัง `CHARGE_ATTEMPTED`, `SHIP_ATTEMPTED`, commit `ACCEPTED`+outbox, broker confirm และ mark `SENT`; ผลภายนอกไม่ชัดต้องเข้า reconcile โดยไม่ยิงซ้ำ. ส่ง event ซ้ำ ตรวจแต้ม/analytics ไม่ซ้ำ และตรวจ email timeout แยกว่ามีความเสี่ยงส่งซ้ำ. ปิด broker, relay, loyalty และ MailFast ทีละส่วน ตรวจ durability, routing/return, retry/dead-letter, ACK หลัง commit, อายุคิว และอัตรา MailFast รวม ≤ 50/s. วัดเวลาแต้มจาก `ACCEPTED` ≤ 1 นาที; ทดสอบ loyalty ล่มเกิน 1 นาทีและรายงานว่า requirement ถูกละเมิด (`docs/scenario.md`, Quality goals and constraints)

## Open questions

- [รอยืนยัน] ใครเป็นผู้ตัดสินใจและยอมรับภาระ durable order store, broker และ on-call?
- ธุรกิจยืนยันหรือไม่ว่า `ACCEPTED` คือจุด “order placed”; เมื่อ loyalty ล่มเกิน 1 นาทีจะจัดการความขัดแย้งอย่างไร และหน้า confirmation ต้องแสดงแต้มที่บันทึกแล้วหรือไม่? Not evidenced in the provided files
- ช่วง peak ยาวเท่าไร อัตราหลัง peak และเวลาส่งอีเมลสูงสุดที่ธุรกิจยอมรับได้เท่าไร? Not evidenced in the provided files
- ผู้ให้บริการ payment/shipping มี idempotency key หรือ API ตรวจผลย้อนหลังหรือไม่; ถ้าไม่มีจะอนุมัติขั้นตอน reconcile/compensation ใด? Not evidenced in the provided files
- สูตรแต้ม, schema analytics และข้อจำกัดข้อมูลส่วนบุคคลคืออะไร? Not evidenced in the provided files
