Agent: Codex; ไม่ทราบชื่อโมเดลที่แน่นอน

# รีวิว ADR-0002 แบบ ATAM

หลักฐาน: `docs/adr/0002-use-rabbitmq-for-post-order-actions.md` และ `docs/scenario.md` รีวิวโดย AI ตัวเดียวกับผู้ร่างใน session เดียว จึงไม่ใช่ independent review

## Quality-attribute scenarios
| แหล่งกระตุ้น | สิ่งกระตุ้น | สภาพแวดล้อม | การตอบสนอง | ตัวชี้วัด |
|---|---|---|---|---|
| ลูกค้า | ส่งคำสั่งซื้อ | peak 200 orders/s และ MailFast ช้า/ล่ม | core order สำเร็จและบันทึกงานที่รอได้ | PlaceOrder p95 ≤ 500 ms; ไม่ fail เพราะสาม downstream (`scenario.md`, Quality goals) |
| broker/worker | ส่ง event ซ้ำหรือ crash หลัง commit ก่อน ack | loyalty ยังอยู่ใน outage envelope ที่ต้องตกลง | ประมวลผลซ้ำโดยไม่เพิ่มแต้มซ้ำ | ผลแต้มหนึ่งครั้งต่อ order และเสร็จภายใน 60 s (`scenario.md`, Loyalty points) |
| MailFast | ตอบช้า ล้มเหลว หรือ rate-limit | 200 orders/s; provider 50 emails/s; p99 8 s | email backlog ไม่กีดขวาง loyalty; จำกัดการส่งรวมทุก replica | ≤ 50 sends/s รวม retry; delivery deadline ยังต้องตกลง (`scenario.md`, MailFast) |

## Findings table
| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | ADR แยก queue ของ email/loyalty/analytics และกำหนด global limiter | รองรับ quota 50/s โดยไม่ให้ email แย่งงานแต้ม (`ADR`, Decision) แต่ต้องทดสอบ capacity จริง |
| Non-risk | ledger unique order_id และอัปเดตแต้มใน transaction เดียวก่อน ack | แก้ผลซ้ำใน business state ไม่อ้างว่า broker ส่งครั้งเดียว (`ADR`, Decision) |
| Risk | order store และจุด accepted ยังไม่ระบุ แม้เป็นเงื่อนไขของ outbox | ยังพิสูจน์ไม่ได้ว่า order ที่สำเร็จทุกอันมี durable event (`ADR`, Context/Open questions) |
| Risk | deadline แต้ม 60 s กับ loyalty outage ไม่มีขอบเขต | ไม่อาจรับประกันทั้งรับ order เสมอและให้แต้มทันเมื่อ outage ยาวกว่า deadline (`scenario.md`, Availability/Loyalty) |
| Sensitivity point | ระยะพีก T และ effective send rate หลัง retry | backlog อย่างน้อย 150T; storage และเวลา drain ไวต่อ T และ traffic หลังพีก (`ADR`, Decision) |
| Trade-off point | เพิ่ม durable store ในเส้นทางรับ order | ได้ durability แต่เพิ่ม latency และ failure dependency; ต้องทดสอบ p95 500 ms (`ADR`, Consequences) |
| Trade-off point | RabbitMQ แทน database polling | เพิ่ม fan-out/modifiability แต่เพิ่มภาระดูแลต่อทีม 4 คน (`ADR`, options; `scenario.md`, Team) |

## Missing or weak
- การเปรียบเทียบ polling ยังไม่มี capacity/ops estimate จึงยังสรุป cost superiority ของ RabbitMQ ไม่ได้
- ยังไม่เลือก replication/durability policy ของ broker/store หรือ retention และ ownership ของ dead-letter replay (`ADR`, Decision/Open questions)
- stable order identity ข้าม client retry และ recovery เมื่อ payment สำเร็จแต่ shipping ล้มยังต้องตัดสินใจ; ระวัง event สำหรับ order ที่ยังไม่ accepted (`ADR`, Consequences)
- consumer dedup ไม่รับประกัน email ไม่ซ้ำเมื่อ MailFast ส่งแล้ว timeout; provider idempotency ยังไม่ทราบ (`ADR`, Decision)

## Verdict
**accept with changes** ในระดับแนวทาง ยังไม่พร้อมนำขึ้น production
1. กำหนด order-accepted boundary, store/transaction และ recovery พร้อม stable client request key
2. ตกลง outage envelope ของแต้มและ email deadline/peak duration; หากธุรกิจไม่ยอมผ่อนต้องเพิ่ม capacity/quota และ availability ที่พิสูจน์ได้
3. กำหนด durability/replication/retention, เจ้าของ alert/replay และเกณฑ์ failure/load test ก่อนอนุมัติใช้งาน
