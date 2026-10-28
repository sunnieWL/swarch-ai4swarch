Agent: Codex; model: GPT-6

# Review: ADR-0002

ADR นี้เป็นร่างจาก AI เพื่อให้ผู้เรียนตรวจและตัดสินใจ ยังไม่ผ่านการตรวจหรือรับรองจากผู้เรียน (`docs/adr/0002-publish-order-placed-with-rabbitmq.md`, บรรทัด 3–7) ความเห็นต่อไปนี้เป็นข้อเสนอของผู้ทบทวน ไม่ใช่การอนุมัติ ADR

## Quality-attribute scenarios

| Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|
| ลูกค้า | สั่งซื้อ | ช่วงขาย 11.11 ที่ 200 orders/s และ MailFast มี p95 1.5 s | `PlaceOrder` รับ order และส่งงานอีเมลภายหลัง | p95 ของ `PlaceOrder` ≤ 500 ms; งานอีเมลไม่ทำให้ order ล้มเหลว (`docs/scenario.md`, บรรทัด 24–28) |
| `loyaltyservice` | หยุดทำงานแล้วกลับมารับ `OrderPlaced` ที่ส่งซ้ำ | มีคำสั่งซื้อรอประมวลผล | เพิ่มแต้มหนึ่งครั้งต่อ order และจัดการงานค้าง | แต้มเข้าภายใน 1 นาทีและไม่ซ้ำ; การล่มไม่ทำให้ order ล้มเหลว (`docs/scenario.md`, บรรทัด 27–28) |
| MailFast | รับอีเมลจากคำสั่งซื้อ 200 รายการ/s | จำกัด 50 emails/s และล้มเหลวราว 0.5% | consumer จำกัดอัตรา ส่งซ้ำ และเก็บงานค้าง | ไม่เกิน 50 emails/s; วัดอายุคิวและระยะเวลาระบายคิว (`docs/scenario.md`, บรรทัด 25–26; ADR บรรทัด 40, 47) |

## Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | การจ่ายเงินและจัดส่งเกิดก่อนจุดบันทึก order/outbox ที่ ADR เสนอ (ADR บรรทัด 38–40, 48) | หาก commit ล้มเหลว อาจเกิดผลภายนอกโดยไม่มี `OrderPlaced`; ADR ยังไม่ระบุขั้นตอนกู้คืนหรือชดเชย จึงยังพิสูจน์ความถูกต้องไม่ได้ |
| Non-risk | การย้ายการส่ง MailFast ออกจากเส้นทางตอบ `PlaceOrder` สอดคล้องกับ p95 MailFast 1.5 s และงบ checkout 500 ms (`docs/scenario.md`, บรรทัด 24, 26; ADR บรรทัด 38) | ลดผลของความช้าอีเมลต่อ latency ของ checkout หากการบันทึก outbox ยังอยู่ในงบเวลา |
| Sensitivity point | ระยะ peak และ quota MailFast กำหนดอายุคิว: ที่ 200 orders/s เข้าและส่งได้ 50 emails/s คิวโต 150 รายการ/s (`docs/scenario.md`, บรรทัด 25–26; ADR บรรทัด 47) | peak หนึ่งนาทีเพิ่มงานค้างราว 9,000 รายการ; ยังประเมินเวลาระบายและพื้นที่เก็บไม่ได้จนกว่าจะรู้ระยะ peak |
| Trade-off point | durable order store, outbox และ relay ช่วยเก็บงานหลังสั่งซื้อ แต่เพิ่มการเขียนก่อนตอบและภาระดูแล (ADR บรรทัด 33, 38, 44–46) | ความทนทานและความแก้ไขง่ายเพิ่มขึ้น ขณะที่ latency, cost และ operability อาจแย่ลง; ยังไม่มีผลวัดที่ 200 orders/s |

## Missing or weak

- ยังไม่เปรียบเทียบทางเลือก durable queue ที่ publish จาก checkout พร้อม publisher confirm กับ outbox ว่าเหตุใดภาระ order store ใหม่จึงคุ้มกว่า; ทางเลือกนี้ต้องวิเคราะห์ช่องว่างระหว่างผลจ่ายเงิน/จัดส่งกับการ publish ด้วย (ADR บรรทัด 22–40)
- เงื่อนไขแต้มภายใน 1 นาทีต้องมีงบเวลาของแต่ละขั้นและการทดสอบระหว่าง `loyaltyservice` ล่ม; คำว่า “ล่มแล้ว order ยังสำเร็จ” เพียงอย่างเดียวไม่รับประกันกรอบ 1 นาที (`docs/scenario.md`, บรรทัด 27–28; ADR บรรทัด 40, 52)
- ADR อ้างพฤติกรรมโค้ดจาก `system/online-boutique/src/checkoutservice/main.go` แต่ไฟล์นั้นอยู่นอกหลักฐานที่อนุญาตในงานนี้ จึงตรวจสอบข้ออ้างระดับโค้ดไม่ได้; ลำดับ call และผลเมื่ออีเมลล้มเหลวยืนยันได้เพียงจาก `docs/scenario.md` บรรทัด 16–18

## Verdict

**Accept with changes** ในฐานะคำแนะนำให้แก้ร่างก่อนผู้เรียนตัดสินใจ: (1) กำหนดจุด commit และวิธีกู้คืน/ชดเชยเมื่อจ่ายเงินหรือจัดส่งแล้วแต่บันทึก order/outbox ไม่สำเร็จ; (2) วัด p95 checkout และเวลาส่งแต้มภายใต้ 200 orders/s รวมช่วง consumer ล่ม; (3) ระบุระยะ peak, แผน quota/อายุคิว MailFast และเหตุผลเลือก outbox เทียบกับ durable queue โดยตรง (`docs/scenario.md`, บรรทัด 24–28; ADR บรรทัด 38–40, 47–58)
