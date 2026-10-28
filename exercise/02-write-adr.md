**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้ เพราะ Run a เห็นเพียง R-12; ตัวเลขอยู่ใน Open questions | ใช้ p95 checkout 500 ms, peak 200 orders/s, MailFast p95 1.5 s และ 50 emails/s จาก `docs/scenario.md` |
| ทางเลือกที่พิจารณา (ระบุ) | เรียกงานหลังสั่งซื้อตรง; ส่ง `OrderPlaced` ผ่านช่องทางทนทาน | เรียกตรง; transactional outbox + RabbitMQ; งาน asynchronous ใน process |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | เสนอ dedup ด้วย `eventId`/business key แต่ไม่รู้ข้อกำหนดครั้งเดียวหรือ rate limit จึงยังไม่กำหนดกลไกจริง | เสนอ `orderId` unique key กับการบันทึกแต้มแบบ atomic; คุม MailFast รวม ≤ 50 emails/s และแยกคิว |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่พบตัวเลขที่แต่ง; `OrderPlaced` และช่องทางทนทานเป็นข้อเสนอออกแบบ ไม่ใช่ข้อเท็จจริงของระบบ | ไม่พบตัวเลขที่แต่ง; RabbitMQ มาจากทักษะทีมใน scenario ส่วน outbox/order store เป็นข้อเสนอใหม่ ไม่ใช่ของเดิม |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | ขยายเงื่อนไขก่อนใช้จริง: ต้องเพิ่ม durable order record/outbox และระบุว่าร่างยังรอผู้เรียนตัดสินใจ | Run b เสนอ outbox แต่ `PlaceOrder` ไม่แสดง order store หรือ commit ที่ใช้ร่วมกัน (`main.go`, `PlaceOrder`) |
| 2 | ย้ายปัญหาคิว MailFast โต 150 รายการ/s จากข้อเสี่ยงทั่วไปเป็นจุดที่ต้องยืนยัน peak/quota ก่อนใช้จริง | 200 orders/s เกินขีดจำกัด 50 emails/s หากเกิดต่อเนื่อง (`docs/scenario.md`, Quality goals and constraints) |
| 3 | เพิ่มการทดสอบกรณีจ่ายเงิน/จัดส่งสำเร็จแต่บันทึก order/outbox ล้มเหลว | Run b กล่าวถึงความเสี่ยงแล้ว แต่ยังไม่วัดการกู้คืน; `PlaceOrder` ทำ side effect ก่อนสร้างผล order (`main.go`, `PlaceOrder`) |

**สถานะ:** ADR ที่ `docs/adr/0002-publish-order-placed-with-rabbitmq.md` เป็นร่างที่ AI ช่วยทำและยังรอผู้เรียนตรวจ แก้ และตัดสินใจเอง ไม่ใช่ฉบับที่ผู้เรียนลงชื่อรับรองแล้ว
