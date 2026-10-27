> **สถานะ:** ร่างช่วยกรอกโดย AI ตามคำขอ ไม่ใช่หลักฐานว่านิสิตตรวจเองหรือประชุมแล้ว ช่อง [รอข้อมูลจริง] ต้องเติมก่อนส่ง

**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| checkoutservice → productcatalogservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml (PRODUCT_CATALOG_SERVICE_ADDR); system/online-boutique/kubernetes-manifests/productcatalogservice.yaml (Service.ports.name: grpc)` | ✅ โดย AI; รอนิสิตตรวจเอง | มี endpoint ใน env และ Service port ชื่อ grpc; ไม่ยืนยันว่าถูกเรียกทุก request |
| checkoutservice → shippingservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml (SHIPPING_SERVICE_ADDR); system/online-boutique/kubernetes-manifests/shippingservice.yaml (Service.ports.name: grpc)` | ✅ โดย AI; รอนิสิตตรวจเอง | มี endpoint ใน env และ Service port ชื่อ grpc; ไม่ยืนยันว่าถูกเรียกทุก request |
| checkoutservice → paymentservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml (PAYMENT_SERVICE_ADDR); system/online-boutique/kubernetes-manifests/paymentservice.yaml (Service.ports.name: grpc)` | ✅ โดย AI; รอนิสิตตรวจเอง | มี endpoint ใน env และ Service port ชื่อ grpc; ไม่ยืนยันว่าถูกเรียกทุก request |
| checkoutservice → emailservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml (EMAIL_SERVICE_ADDR); system/online-boutique/kubernetes-manifests/emailservice.yaml (Service.ports.name: grpc)` | ✅ โดย AI; รอนิสิตตรวจเอง | มี endpoint ใน env และ Service port ชื่อ grpc; ไม่ยืนยันว่าถูกเรียกทุก request |
| checkoutservice → currencyservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml (CURRENCY_SERVICE_ADDR); system/online-boutique/kubernetes-manifests/currencyservice.yaml (Service.ports.name: grpc)` | ✅ โดย AI; รอนิสิตตรวจเอง | มี endpoint ใน env และ Service port ชื่อ grpc; ไม่ยืนยันว่าถูกเรียกทุก request |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): shoppingassistantservice มีเพียง env แต่ไม่พบ deployment และ enable flag ยังเป็น comment; loadgenerator มีไฟล์แต่ถูก comment ออกจาก base kustomization; ไม่มีหลักฐาน HTTPS

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. **ร่าง AI ให้นิสิตพิจารณา:** `.proto` บอกสัญญา RPC และ message แต่ไม่บอกผู้เรียกจริง ภาษาที่ implement หรือที่เก็บข้อมูล ส่วน manifest เพิ่มเส้นทางที่ตั้งค่าและ Redis ได้ แต่ยังไม่พิสูจน์ runtime behavior
2. **ร่าง AI ให้นิสิตพิจารณา:** การวาด shoppingassistantservice ว่าใช้งานจริงจาก env เพียงตัวเดียวจะทำให้สมาชิกใหม่คิดว่ามี deployment และ feature เปิดอยู่ ทั้งที่หลักฐานชุดนี้ไม่แสดงเช่นนั้น
