Agent: Codex; ไม่ทราบชื่อโมเดลที่แน่นอน

# ADR-0002: แยกงานหลังสั่งซื้อด้วย event

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** รอนิสิตระบุ

## Context and problem
R-12 กำหนดให้ส่งอีเมลผ่าน MailFast เพิ่ม loyaltyservice และบันทึก analytics พร้อมรองรับงานเพิ่มปีหน้า (requirement ที่คัดจาก `docs/scenario.md`, R-12)

> ร่างอ้างอิงรูปแบบไม่มีบริบทเท่านั้น ไม่ใช่ผล zero-shot จริง เพราะ session นี้อ่าน scenario และใบงานแล้ว ห้ามใช้เป็นหลักฐานเปรียบเทียบเชิงทดลอง

## Decision drivers
- [Inferred] modifiability: เพิ่มผู้รับงานโดยลดการแก้ checkout
- [Inferred] availability: ลดการผูกความสำเร็จของคำสั่งซื้อกับ downstream
- ตัวเลข latency, throughput และ SLA ไม่ใช้ในร่างนี้

## Considered options
1. เรียกทั้งสามปลายทางแบบ synchronous
2. ส่ง event ผ่าน message broker ให้ consumer แยกกัน
3. บันทึกงานในฐานข้อมูลแล้วให้ worker polling

### Pros and cons of the options
| Option | Pros | Cons |
|---|---|---|
| synchronous | โครงสร้างการเรียกตรง ติดตาม flow ง่าย | latency รวมเพิ่ม และแก้ checkout เมื่อเพิ่มงาน |
| message broker | แยก consumer และการ retry ได้ | เพิ่มภาระปฏิบัติการและต้องจัดการ duplicate |
| database polling | ใช้ transaction ผูกงานกับข้อมูลได้ | ต้องจัดการ polling, contention และการแบ่งงาน |

## Decision
[Inferred: ข้อเสนอ] เลือก durable event พร้อม consumer แยกอีเมล แต้ม และ analytics ใช้ outbox เมื่อมี transactional order store และทำ idempotency ต่อ order_id; ยังไม่เลือกผลิตภัณฑ์ broker เพราะ requirement อย่างเดียวไม่ให้ข้อมูลทีมและการปฏิบัติการ

## Consequences
- **Positive:** เพิ่มงานใหม่โดยเพิ่ม consumer (modifiability)
- **Negative:** ผลหลังสั่งซื้อไม่เกิดทันที (consistency) และต้องดูแล broker (cost)
- **Risks and mitigations:** event สูญหายระหว่างยืนยัน order กับส่ง event; ออกแบบ durable intent และตรวจ reconciliation (reliability)

## Validation
ทดสอบ consumer ล่ม การส่งซ้ำ และ crash ก่อน/หลัง publish; เกณฑ์เชิงตัวเลขรอยืนยัน

## Open questions
- latency/peak/SLA แต่ละงานเท่าไร และ MailFast มี rate limit หรือไม่?
- จุด commit ของ order และ durable order store อยู่ที่ใด?
- provider รองรับ idempotency หรือไม่ และทีมดูแล broker ใดได้?
