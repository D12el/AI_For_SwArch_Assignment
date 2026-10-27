Agent: Codex; ไม่ทราบชื่อโมเดลที่แน่นอน

# ADR-0002: ใช้ RabbitMQ แยกงานหลังสั่งซื้อ

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** รอนิสิตระบุและรับรอง

## Context and problem
R-12 ต้องส่ง MailFast email เพิ่ม loyaltyservice และ analytics และรองรับ SMS/fraud scoring/recommendations ปีหน้า (`docs/scenario.md`, R-12)

เป้าหมาย PlaceOrder p95 ≤ 500 ms จากประมาณ 300 ms; peak 200 orders/s; MailFast p95 1.5 s, p99 8 s, failure ประมาณ 0.5%, limit 50 emails/s; แต้มภายใน 1 นาทีและห้ามซ้ำ; order ต้องไม่ล้มเพราะสาม downstream ล่ม ทีม 4 คนถนัด Go/gRPC และ 2 คนเคยใช้ RabbitMQ; ใช้ Kubernetes และชอบ open-source self-hosted (`docs/scenario.md`, Quality goals and constraints)

โค้ดปัจจุบันเรียก chargeCard, shipOrder แล้ว sendOrderConfirmation ใน PlaceOrder; email error ถูก log แล้วคืนผลสำเร็จ แต่ยังรอคำเรียก email จบ (`system/online-boutique/src/checkoutservice/main.go`, PlaceOrder) ไฟล์นี้ไม่แสดง durable order transaction/outbox จึงไม่อ้างว่ามีแล้ว รูปแบบ ADR อ้าง `docs/adr/0001-use-redis-for-cart-storage.md` แต่ความทนทานของ cart ไม่ใช่เกณฑ์ของ financial order

## Decision drivers
1. availability: downstream ทั้งสามล่มต้องไม่ทำให้ order ล้ม
2. correctness: แต้มต่อ order ครั้งเดียว และ timeliness ภายใน 60 วินาที
3. performance: checkout p95 ≤ 500 ms ที่ 200 orders/s
4. modifiability: เพิ่มงานใหม่โดยไม่เพิ่ม synchronous call ใน checkout
5. operability/cost: เหมาะกับทีมและ self-hosted Kubernetes

## Considered options
1. synchronous gRPC fan-out จาก checkout ไปทุกปลายทาง
2. RabbitMQ พร้อม durable outbox และ queue แยกตามงาน
3. transactional database work queue ให้ worker polling แยกตามประเภท

### Pros and cons of the options
| Option | Pros | Cons |
|---|---|---|
| synchronous | ใช้ Go/gRPC ที่ทีมถนัด; tracing ตรง | รอ MailFast ที่ช้ากว่าเป้าหมาย checkout; ต้องออกแบบ durable retry เพิ่มเมื่อปลายทางล่ม |
| RabbitMQ + outbox | แยก backlog และ retry; เพิ่ม subscriber ได้; ทีมบางคนเคยใช้ | เพิ่ม broker, outbox store และ recovery; ไม่ทำให้ exactly-once เกิดเอง |
| database queue | ผูก order และงานใน local transaction ได้; ลด broker หากมี DB อยู่แล้ว | หลักฐานยังไม่แสดง order DB; polling และ contention ต้องทดสอบ; fan-out ต้องเขียนเอง |

## Decision
[ข้อเสนอ ยังไม่ใช่ implementation] เลือก RabbitMQ โดยมีเงื่อนไขว่าต้องสร้าง durable order/dispatch intent และ outbox ใน local transaction เดียวกันหลังนิยาม order-accepted ให้ชัด ห้ามใช้แค่ goroutine หรือ publish หลัง response โดยไม่มี durable intent

Relay ส่ง persistent event พร้อม publisher confirm ไป exchange และ durable queue แยก email, loyalty, analytics; ใช้ consumer acknowledgement หลังผลถูกบันทึก หาก crash หลัง publish ก่อน mark outbox อาจส่งซ้ำได้ จึงใช้ event_id และ stable order_id สำหรับ deduplication ข้อกำหนดเหล่านี้เป็นการออกแบบที่เสนอ ไม่ใช่สิ่งที่พบใน source

- loyaltyservice ต้องทำ unique order_id ใน ledger และเพิ่มยอดแต้มภายใน transaction เดียวกัน แล้วจึง ack เพื่อให้ผลทางธุรกิจเกิดครั้งเดียวแม้รับ event ซ้ำ แยก queue/worker จาก email
- email worker เปลี่ยน mock logic เป็น MailFast integration ใช้ global limiter ร่วมทุก replica ไม่เกิน 50 emails/s รวม retry; backoff/jitter และจัดการ rate-limit response ใช้ provider idempotency หากมี หากไม่มี timeout หลังส่งอาจทำให้ email ซ้ำได้ ต้องตกลงกับธุรกิจ
- analytics ใช้ event_id deduplicate และเก็บเวลาเกิดเหตุการณ์แยกเวลารับ
- retry แบบจำกัด แยก poison message ไป dead-letter queue มี alert และคู่มือ replay พร้อม metric oldest-message age โดยเฉพาะ loyalty deadline

[Inferred: คำนวณจาก scenario] ถ้าหนึ่ง order ใช้หนึ่ง email ช่วงพีก backlog เพิ่มอย่างน้อย 200−50 = 150 emails/s ก่อนคิด retry; ระยะพีก T วินาทีสร้าง backlog อย่างน้อย 150T รายการ การเพิ่ม worker ไม่เพิ่ม quota ของ MailFast จึงต้องตกลง email delivery deadline และระยะพีกก่อนกำหนด storage

## Consequences
- **Positive:** email backlog ไม่ขวาง loyalty และ checkout (availability, performance); เพิ่ม consumer ได้ (modifiability)
- **Positive:** transaction ใน loyalty ledger ป้องกัน double points จาก redelivery (correctness)
- **Negative:** ต้องดูแล broker และ durable store เพิ่ม (cost, operability); UI อาจเห็นแต้มภายหลัง (consistency)
- **Risks and mitigations:** outbox store ล่มอาจขวางการรับ order แม้ downstream แยกแล้ว ต้องตกลง failure policy และวัดความพร้อมใช้งานของ store (availability, durability)
- **Risks and mitigations:** loyaltyservice ล่มนานกว่า 60 วินาทีทำให้รับประกัน deadline ไม่ได้ ต้องนิยาม outage envelope และแนวทางชดเชยก่อนอนุมัติ (timeliness)
- **Risks and mitigations:** payment/shipping เป็น side effect ภายนอก local transaction ไม่ครอบคลุมทั้งหมด ต้องนิยาม recovery และ stable request key; uuid ใหม่ทุก PlaceOrder ไม่ป้องกัน client retry ซ้ำข้าม request (`main.go`, PlaceOrder/uuid.NewUUID) (correctness)

## Validation
เป็นแผนทดสอบ ยังไม่ได้รัน
1. โหลด 200 orders/s วัด PlaceOrder p95 ≤ 500 ms; ห้ามบวกค่า p95 ของแต่ละ service เพื่ออ้างว่า end-to-end ผ่าน
2. จำลอง email/analytics/loyalty unavailable: order ที่ส่วน core สำเร็จยังคืนสำเร็จ งานค้างใน durable intent/queue; ทดสอบ broker outage ผ่าน outbox
3. ส่ง order event เดิมซ้ำพร้อมกันและ crash ก่อน/หลัง ledger commit: ledger มีผลแต้มหนึ่งครั้งต่อ order; วัด order accepted → points committed ≤ 60 s ภายใต้ outage envelope ที่ตกลง
4. ทดสอบ MailFast limit ข้าม replica รวม retries, timeout 8 s และ failure injection ประมาณ 0.5%; วัด backlog/oldest age และไม่ส่งเกิน 50/s
5. kill relay หลัง confirm ก่อน mark sent, restart consumer ก่อน ack และ replay dead-letter: ไม่สูญ durable event และไม่เพิ่มแต้มซ้ำ

## Open questions
- ระยะพีก 11.11 และ email/analytics deadline, retention และงบ storage เท่าไร?
- durable order/dispatch store ใช้เทคโนโลยีใด และจุด order accepted คือเมื่อไร?
- หาก store ใช้ไม่ได้จะรับ order แบบใดโดยไม่ทิ้งงาน?
- outage สูงสุดที่ SLA 1 นาทีครอบคลุมและแนวทาง breach/compensation คืออะไร?
- MailFast มี idempotency key, quota แบบใด และ API protocol ที่แน่นอนหรือไม่? Not evidenced in the provided files
- กติกาคำนวณแต้ม order cancellation/refund และ retention ของ dedup key คืออะไร?
