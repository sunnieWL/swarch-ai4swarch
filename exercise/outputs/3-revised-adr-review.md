# Additional review: ADR-0002 (Codex)

**ผู้ตรวจ:** Codex (GPT-6; ไม่ทราบรุ่นย่อย)
**ขอบเขต:** ตรวจร่างที่แก้แล้วเพิ่มเติมจาก run 3; ไม่ใช่การอนุมัติของผู้เรียน

## ผลตรวจ

ร่างคงหัวข้อของ docs/adr/template.md ครบ ระบุ durable order/outbox เป็นสิ่งที่ต้องเพิ่ม และไม่อ้างว่า payment, shipping หรือ MailFast รองรับ idempotency อยู่แล้ว (docs/adr/0002-publish-order-placed-with-rabbitmq.md บรรทัด 42–45; system/online-boutique/protos/demo.proto, PlaceOrderRequest, ChargeRequest, ShipOrderRequest) การใช้ durable queues, persistent messages, publisher confirm, mandatory routing, mark SENT หลัง confirm และ commit ก่อน ACK มีเงื่อนไขสำคัญครบ (ADR บรรทัด 44–45) ร่างยอมรับตรงไปตรงมาว่า loyalty ล่มเกิน 1 นาทีจะขัดกับ R-12 (ADR บรรทัด 46; docs/scenario.md บรรทัด 27–28)

1. **ต้องกำหนด retry ให้ตรวจสอบได้:** การใช้ requestId แบบ unique ยังไม่กำหนดวิธีปฏิเสธ key เดิมที่ส่ง payload ต่างกัน หรือแยก PENDING_PAYMENT ที่ยังไม่เรียก payment ออกจากกรณีส่งคำขอแล้วแต่ผลไม่ชัด การสั่งว่า retry ห้ามเริ่ม charge/ship ซ้ำจึงอาจปล่อย order ค้างโดยไม่มีทางดำเนินต่อ ควรบันทึก snapshot/hash ของคำขอ, กำหนด transition แบบ atomic และขั้นตอน reconcile พร้อมเกณฑ์เวลา (ADR บรรทัด 42–43; system/online-boutique/src/checkoutservice/main.go บรรทัด 238–260) การรับประกันไม่ charge/ship ซ้ำยังขึ้นกับความสามารถ provider ที่ร่างระบุว่าต้องยืนยัน (demo.proto, ChargeRequest, ShipOrderRequest)
2. **แยกกฎ ACK ตามชนิด side effect:** ข้อความว่า consumer ทุกตัว commit ผลและ dedup ก่อน ACK ใช้ได้เมื่อผลกับ ledger อยู่ในธุรกรรมเดียว แต่การส่ง MailFast ไม่มีธุรกรรมร่วมกับ ledger และ timeout อาจส่งอีเมลสำเร็จแล้ว ควรระบุรูปแบบ ACK/retry ของ email เป็นกรณีเฉพาะ และระบุว่าจะตรวจ duplicate ของ analytics อย่างไร (ADR บรรทัด 45, 58; docs/scenario.md, R-12)
3. **กำหนดเกณฑ์ capacity ก่อนรับรอง:** ที่ 200 orders/s และ quota 50 emails/s backlog โตอย่างน้อย 150 รายการ/s ระหว่าง peak; retry ทำให้โตเร็วขึ้น ร่างคำนวณทิศทางถูกแล้ว แต่ยังไม่มีระยะ peak, ขนาดข้อความ, พื้นที่เก็บขั้นต่ำ หรือเวลา drain ที่ยอมรับได้ ควรกำหนดค่าเหล่านี้และเกณฑ์ผ่านของ load test ก่อนเปิดใช้ (ADR บรรทัด 53, 58, 64; docs/scenario.md บรรทัด 24–26)
4. **ยืนยันจุดเริ่มนับเวลาและตัดข้อความที่ไม่มีหลักฐาน:** ADR เริ่มนับ 1 นาทีเมื่อ ACCEPTED แต่ R-12 ใช้คำว่า “of the order”; ควรให้ธุรกิจยืนยันนิยามนี้ (ADR บรรทัด 46; docs/scenario.md บรรทัด 27) ข้อความเรื่อง “5 run” และไฟล์ review เดิมใน ADR บรรทัด 7 พิสูจน์ไม่ได้จากไฟล์ที่อนุญาตในงานนี้ ควรลบหรือให้ผู้เรียนตรวจแยก

**ไฟล์ที่อ่าน:** docs/adr/0002-publish-order-placed-with-rabbitmq.md; docs/scenario.md; docs/adr/template.md; system/online-boutique/src/checkoutservice/main.go; system/online-boutique/protos/demo.proto

## หลังแก้ตามรีวิว

1. **แก้ในร่าง:** ผูก `requestId` กับข้อมูลคำขอ, แยกสถานะก่อนส่งกับผลที่ไม่ชัด, ให้ `READY_TO_CHARGE` และ `PAYMENT_RECORDED` รับงานต่อด้วย atomic claim และกำหนด durable record/ทางดำเนินงานของผู้ดูแลสำหรับ reconcile แล้ว (ADR บรรทัด 42–43) ความสามารถตรวจผล payment/shipping ยังต้องยืนยันก่อนใช้จริง
2. **แก้ในร่าง:** แยกธุรกรรมแต้มและ analytics จาก MailFast; กรณีส่งอีเมลสำเร็จแต่บันทึกผลล้มเหลวถูกจัดเป็นผลไม่ชัดและเข้า reconcile แล้ว (บรรทัด 45) ความสามารถตรวจสถานะหรือ idempotency ของ MailFast ยังไม่ปรากฏในหลักฐาน
3. **แก้เป็นเงื่อนไขก่อนรับรอง:** กำหนดให้ทราบระยะ peak, ขนาดข้อความ, พื้นที่เก็บ และเวลา drain แล้วใช้เป็นเกณฑ์ load test (บรรทัด 53, 58) ค่าจริงยังรอธุรกิจและการวัด
4. **แก้ในร่าง:** ขอคำยืนยันจากธุรกิจว่า `ACCEPTED` คือจุดเริ่มนับ 1 นาที และลบข้ออ้างเรื่อง run/review เดิมที่ตรวจไม่ได้แล้ว (บรรทัด 7, 46, 63) คำยืนยันเรื่องเวลายังรอธุรกิจ

**การตรวจอิสระเพิ่มเติม:** พบว่าถ้า storage ล้มเหลวจะบันทึก `NEEDS_RECONCILIATION` ในจังหวะนั้นไม่ได้ จึงแก้ ADR ให้คง durable `ATTEMPTED` ที่เขียนไว้ก่อนเรียกบริการเป็นหลักฐานกู้คืน; บันทึกผลตรวจสอบเมื่อ storage กลับมา. เงื่อนไขด้านผู้ให้บริการและธุรกิจยังค้างตามข้างต้น

**ไฟล์ที่อ่านในการตรวจรอบสุดท้าย:** `docs/adr/0002-publish-order-placed-with-rabbitmq.md`; `exercise/outputs/3-revised-adr-review.md`
