# สิ่งที่ต้องส่ง

ส่ง URL นี้ใน MCV หลังตรวจข้อมูลครบและ push แล้ว:

https://github.com/D12el/AI_For_SwArch_Assignment

## ไฟล์ที่ต้องมีใน repository
- ใบงานภาษาไทย `exercise/00-prompt-log.md` ถึง `exercise/05-reflection.md`
- ผล AI ห้าไฟล์ `1A-architecture.md`, `1B-architecture.md`, `2a-adr.md`, `2b-adr.md`, `3-review.md` ใน `exercise/outputs/`
- ADR `docs/adr/0002-use-rabbitmq-for-post-order-actions.md` ที่นิสิตตรวจและรับรองแล้ว

## สิ่งที่ยังต้องทำจริงก่อนส่ง
1. ชื่อ/รหัสนิสิตเติมแล้ว; ระบุแพ็กเกจและคะแนนประโยชน์ใน prompt log
2. หากอาจารย์ต้องการทดลองตามขั้นตอนเคร่งครัด ให้ทำแต่ละ run ใหม่คนละแชต ผลชุดนี้ทำใน session รวมและ 2a ไม่ใช่ zero-shot จริง
3. ตรวจ dependency อย่างน้อย 5 เส้นด้วยตนเอง; แถวปัจจุบันยืนยันโดย AI เท่านั้น
4. อ่าน ADR ตัดสินใจและแก้ไขเอง แล้วระบุการเปลี่ยนแปลงในใบงาน 02
5. เติมความประทับใจจริงในใบงาน 03
6. ใช้บันทึกประชุมร่วมจาก repository ของ Thukdanai แล้ว รวมมติและบทบาท SRE ของ Thanaphat
7. ตรวจ reflection ที่ AI เรียบเรียงจากบันทึกกลุ่มในมุม SRE และปรับให้ตรงกับความเห็นตนเอง
8. ตรวจ Settings ของ GitHub ว่า repository เป็น public; ยังไม่ได้ยืนยันสถานะ visibility จาก remote config
9. commit/push การแก้ไขหลังเติมข้อมูล และส่ง URL ใน MCV

ยังไม่ใช่งานที่เสร็จครบตามแบบฝึกหัดจนกว่าส่วนของนิสิตและกลุ่มข้างต้นจะครบ โบนัสไม่บังคับจึงไม่ได้ทำ

## เอกสารประชุมร่วม

- docs/adr/0002-use-rabbitmq-for-post-order-events.md เป็นสำเนา ADR ของ Thukdanai ที่กลุ่มรีวิว
- exercise/outputs/group-3-review.md เป็นสำเนาผลรีวิวที่ใช้ประชุม
- ADR post-order-actions และ outputs/3-review.md เดิมเป็นผลงาน Codex ใน session ของ Thanaphat แยกจากเอกสารประชุมร่วม
