# 📖 Data Dictionary: SME Retail Targeted Promotion

เอกสารฉบับนี้จัดทำขึ้นเพื่ออธิบายโครงสร้างข้อมูล และความหมายของคอลัมน์ต่างๆ ในโปรเจกต์ เพื่อให้ผู้ตรวจสอบเข้าใจตรรกะของระบบได้ทันที

---

## 1. customer_master.csv
* **ที่อยู่ไฟล์:** `data/raw/customer_master.csv`
* **จำนวนแถว:** 500 แถว (ข้อมูลลูกค้า 500 คน)
* **Primary Key:** `customer_id`

| Column Name | Data Type | Example Value | Description / Business Logic |
| :--- | :--- | :--- | :--- |
| `customer_id` | object (String) | `C0001` | รหัสประจำตัวลูกค้าแต่ละราย (เติม 0 ข้างหน้าให้ครบ 4 หลัก) |
| `gender` | object (String) | `F` | เพศของลูกค้า (สุ่มจำลองในอัตราส่วน ชาย 45% / หญิง 55%) |
| `true_segment` | object (String) | `Champion` | กลุ่มลูกค้าเดิมที่ร้านค้าแบ่งไว้ (Champion, Loyal, At-Risk, New) |

---

## 2. promotion_master.csv
* **ที่อยู่ไฟล์:** `data/raw/promotion_master.csv`
* **จำนวนแถว:** 4 แถว (แคมเปญโปรโมชั่น 4 รูปแบบ)
* **Primary Key:** `promotion_id`

| Column Name | Data Type | Example Value | Description / Business Logic |
| :--- | :--- | :--- | :--- |
| `promotion_id` | object (String) | `P_NEWYEAR` | รหัสของแคมเปญโปรโมชั่น |
| `promo_type` | object (String) | `percent_discount` | ประเภทส่วนลด ได้แก่ แบบเปอร์เซ็นต์ หรือ แบบหักเงินคงที่ (fixed_discount) |
| `discount_value` | int64 | `10` | มูลค่าส่วนลด (หากเป็นเปอร์เซ็นต์คือ 10%, หากเป็นเงินสดคือ ลด 200 บาท) |
| `start_date` | object (Date) | `2023-12-15` | วันที่เริ่มต้นเปิดใช้งานโปรโมชั่น |
| `end_date` | object (Date) | `2024-01-15` | วันที่สิ้นสุดโปรโมชั่น |
| `target_segment` | object (String) | `Champion` | กลุ่มลูกค้าเป้าหมายที่ได้รับสิทธิ์ (`All` หมายถึงได้ทุกคน) |

---

## 3. sales_transaction.csv
* **ที่อยู่ไฟล์:** `data/raw/sales_transaction.csv`
* **จำนวนแถว:** ~3,000 แถว (ประวัติการซื้อขายในรอบ 6 เดือน)
* **Foreign Keys:** `customer_id` ➔ `customer_master.csv`, `promo_used` ➔ `promotion_master.csv`

| Column Name | Data Type | Example Value | Description / Business Logic |
| :--- | :--- | :--- | :--- |
| `transaction_id` | object (String) | `T000001` | รหัสการซื้อขาย |
| `customer_id` | object (String) | `C0001` | รหัสลูกค้าที่เข้ามาซื้อสินค้า |
| `transaction_date` | object (Date) | `2024-03-09` | วันที่เกิดการซื้อขาย (สุ่มจำลองช่วงเวลา 6 เดือนของปี 2024) |
| `subtotal` | float64 | `148.13` | ยอดรวมก่อนหักส่วนลด (สุ่มจากช่วงราคาตามพฤติกรรมของแต่ละ Segment) |
| `promo_id` | object (String) | `P_NEWYEAR` | รหัสโปรโมชั่นที่ลูกค้าเลือกใช้ในบิลนั้น (`None` คือไม่ใช้โปร) |
| `discount` | float64 | `14.81` | จำนวนเงินส่วนลดที่ได้รับจริงในบิลนั้น (คำนวณตามเงื่อนไขของโปร) |
| `final_amount` | float64 | `133.32` | ยอดเงินสุทธิที่ลูกค้าจ่ายจริง (`subtotal` - `discount`) |

---

## ⚠️ Known Data Quality Issues

| Issue | ตาราง | ผลกระทบ | วิธีรับมือ |
|-------|-------|---------|-----------|
| promotion_id = NONE | sales_transaction | join กับ promo ไม่ได้ | filter ออกก่อน join |
| ราคาไม่ตรง product_master | product vs transaction | revenue ผิด | ใช้ subtotal จาก transaction เสมอ |
