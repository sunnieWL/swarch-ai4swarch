# ADR-0002: เผยแพร่ OrderPlaced ผ่าน RabbitMQ

- **Status:** Proposed
- **Date:** 2026-10-28
- **Deciders:** [รอยืนยัน]

> ร่างจาก AI เพื่อให้ผู้เรียนตรวจและตัดสินใจ ยังไม่ใช่ ADR ที่ผู้เรียนรับรอง

## Context and problem

R-12 ต้องส่งอีเมลผ่าน MailFast แทน logic จำลองของ `emailservice`, ให้ `loyaltyservice` เพิ่มแต้ม และบันทึกเหตุการณ์ให้การตลาด; ปีหน้าอาจมีงานเพิ่ม (`docs/scenario.md`, R-12). ปัจจุบัน `checkoutservice.PlaceOrder` เรียกจ่ายเงิน จัดส่ง และส่งอีเมลแบบ gRPC; อีเมลล้มเหลวแล้ว order ยังสำเร็จ (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`, `sendOrderConfirmation`).

`PlaceOrder` ต้องมี p95 ≤ 500 ms (ปัจจุบันราว 300 ms); ช่วง peak มี 200 orders/s. MailFast มี p95 1.5 s, ล้มเหลวราว 0.5% และจำกัด 50 emails/s. แต้มต้องได้ภายใน 1 นาทีและเพียงครั้งเดียว; งานหลังสั่งซื้อล่มต้องไม่ทำให้ order ล้มเหลว (`docs/scenario.md`, Quality goals and constraints).

## Decision drivers

1. ไม่ให้งานหลังสั่งซื้อทำให้ order ล้มเหลว (availability).
2. รักษา p95 ของ `PlaceOrder` ≤ 500 ms (performance).
3. ให้แต้มภายใน 1 นาทีและไม่ซ้ำ (correctness).
4. เพิ่ม consumer ในอนาคตได้ และทีม 4 คนดูแลระบบไหว (modifiability, operability).

## Considered options

1. เรียก MailFast, `loyaltyservice` และ analytics ตรงใน `PlaceOrder`
2. บันทึก `OrderPlaced` ใน transactional outbox แล้วส่งผ่าน RabbitMQ ให้ consumer แยกกัน
3. เริ่มงาน asynchronous ภายใน `checkoutservice` โดยไม่บันทึกงานค้างถาวร

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. เรียกตรง | ใช้รูปแบบการเรียกที่ทีมรู้จัก | MailFast p95 1.5 s เกินงบ 500 ms ถ้ารอผล; ถ้าไม่รอ/ไม่ retry งานอาจหาย; เพิ่มงานต้องแก้ checkout |
| 2. Outbox + RabbitMQ | [Inferred] แยกความล้มเหลวและเพิ่มผู้รับได้; RabbitMQ สอดคล้องกับทักษะทีม 2 คน (`docs/scenario.md`, Team) | ต้องเพิ่มที่เก็บ order/outbox, relay, broker และการดูแล; โค้ดปัจจุบันยังไม่มีขอบเขต commit ให้ใช้ (`main.go`, `PlaceOrder`) |
| 3. งานในกระบวนการ | ตอบ checkout ได้เร็วโดยไม่เพิ่ม broker | [Inferred] งานค้างหายได้เมื่อ process หยุด; คุม retry และเวลาให้แต้มยาก |

## Decision

**เสนอเลือกตัวเลือก 2 โดยมีงานที่ต้องทำก่อนใช้จริง.** เพิ่ม durable order record และ outbox ให้บันทึกการยอมรับ order กับ `OrderPlaced` แบบ atomic ก่อนตอบสำเร็จ จากนั้น relay ส่ง RabbitMQ พร้อม publisher confirm และ retry. แยกคิว/consumer สำหรับ MailFast, `loyaltyservice` และ analytics. เปลี่ยนการเรียก `emailservice` ใน `PlaceOrder` เป็น consumer ที่ส่ง MailFast (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`, `sendOrderConfirmation`; `docs/scenario.md`, R-12).

ให้ `loyaltyservice` ใช้ `orderId` เป็น unique key และบันทึก key กับแต้มในธุรกรรมเดียว ก่อน acknowledge; การส่งซ้ำจึงไม่เพิ่มแต้มซ้ำ [Inferred]. จำกัด MailFast รวมทุก consumer ไม่เกิน 50 emails/s; retry แบบหน่วงและเก็บงานที่แก้เองไม่ได้เพื่อตรวจต่อ. ต้องออกแบบกรณีจ่ายเงิน/จัดส่งสำเร็จแต่บันทึก order ล้มเหลว เพราะโค้ดปัจจุบันทำ side effect ก่อนสร้าง `OrderResult` (`main.go`, `PlaceOrder`).

## Consequences

- **Positive:** MailFast ไม่อยู่ในเส้นทางตอบ checkout จึงลดผลต่อ latency (performance); consumer ล่มแล้ว order ยังรับได้หาก outbox พร้อม (availability) [Inferred].
- **Positive:** เพิ่ม consumer สำหรับ SMS หรือ fraud scoring ได้โดยไม่เพิ่ม call ใน `PlaceOrder` (modifiability) [Inferred].
- **Negative:** ต้องดูแล broker, durable store, relay และคิว (cost, operability); อีเมลถึงภายหลัง (timeliness) [Inferred].
- **Risks and mitigations:** ที่ peak 200 orders/s แต่ MailFast จำกัด 50 emails/s คิวเพิ่มได้ 150 รายการ/s หาก peak ต่อเนื่อง; วัดอายุคิว และยืนยันระยะ peak/การเพิ่ม quota ก่อนใช้จริง (scalability, timeliness; `docs/scenario.md`, Quality goals and constraints).
- **Risks and mitigations:** การจ่ายเงิน/จัดส่งอยู่นอกธุรกรรม order/outbox; ต้องกำหนดการกู้คืนหรือชดเชยเมื่อบันทึก order ไม่สำเร็จ (correctness, availability; `main.go`, `PlaceOrder`).

## Validation

ทดสอบ 200 orders/s และวัด p95 ของ `PlaceOrder` ≤ 500 ms; ปิดแต่ละ consumer แล้วตรวจว่า order สำเร็จและงานไม่หาย. ส่ง `OrderPlaced` ซ้ำแล้วตรวจว่าแต้มเพิ่มครั้งเดียวและเข้าภายใน 1 นาที. วัด MailFast ไม่เกิน 50 emails/s, อายุคิว และการกู้คืน outbox/relay. ทดสอบจุดที่จ่ายเงินหรือจัดส่งแล้ว แต่บันทึก order/outbox ล้มเหลว (`docs/scenario.md`, Quality goals and constraints; `main.go`, `PlaceOrder`).

## Open questions

- [รอยืนยัน] ใครเป็นผู้ตัดสินใจ ADR นี้ และจะยอมรับภาระ order store/outbox หรือไม่?
- จุด commit และวิธีกู้คืนกรณีจ่ายเงิน/จัดส่งสำเร็จก่อนบันทึก order คืออะไร? Not evidenced in the provided files.
- peak 200 orders/s นานเท่าไร และ MailFast เพิ่ม quota ได้หรือไม่? Not evidenced in the provided files.
- สูตรแต้มและสัญญาข้อมูล analytics คืออะไร? Not evidenced in the provided files.
