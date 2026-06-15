# 🎯 Smart Promotion Targeting Engine

<p align="left">
  <img src="https://img.shields.io/badge/Python-3.11-3776AB?style=flat&logo=python&logoColor=white"/>
  <img src="https://img.shields.io/badge/Pandas-2.0-150458?style=flat&logo=pandas&logoColor=white"/>
  <img src="https://img.shields.io/badge/Scikit--Learn-1.3-F7931E?style=flat&logo=scikit-learn&logoColor=white"/>
  <img src="https://img.shields.io/badge/XGBoost-2.0-006E9B?style=flat&logo=xgboost&logoColor=white"/>
  <img src="https://img.shields.io/badge/Jupyter-Notebook-F37626?style=flat&logo=jupyter&logoColor=white"/>
  <img src="https://img.shields.io/badge/Status-In%20Progress-FAEEDA?style=flat&labelColor=BA7517&color=FAEEDA"/>
</p>

> **ระบบแนะนำโปรโมชันอัจฉริยะสำหรับร้านค้า SME Retail**  
> ใช้ Customer Segmentation + Revenue Forecast + Promotion Classifier  
> เพื่อลดการสูญเสีย margin และเพิ่ม Promotion ROI อย่างมีนัยสำคัญ

---

## 📌 What — โปรเจกต์นี้ทำอะไร?

ระบบ 3-Layer Data Science Pipeline ที่ตอบคำถามว่า  
**"ควรยิงโปรโมชันไหน ให้ลูกค้าคนไหน และเมื่อไหร่"**

| Layer | Model | Input | Output |
|-------|-------|-------|--------|
| **1 — Segmentation** | K-Means Clustering | RFM Features | Customer Segment |
| **2 — Forecast** | Ridge Regression | RFM + Segment | Predicted Spend 30d |
| **3 — Classifier** | XGBoost | RFM + Segment + Forecast | Redeem Probability |

**Final Output:** Next-Best-Offer Table

```
customer_id │ segment   │ predicted_spend_30d │ recommended_promo │ redeem_prob │ action
────────────┼───────────┼─────────────────────┼───────────────────┼─────────────┼────────
C0042       │ At-Risk   │ ฿ 180               │ PR06 (Flash 25%)  │ 0.84        │ ✅ ยิงโปร
C0115       │ Champion  │ ฿ 2,340             │ —                 │ —           │ ⏸ งด
C0287       │ New       │ ฿  65               │ PR08 (New 30%)    │ 0.61        │ ✅ ยิงโปร
```

---

## ❓ Why — ทำไมถึงทำโปรเจกต์นี้?

### Business Problem

ร้านค้า SME Retail มักออกโปรโมชันแบบ **one-size-fits-all**  
เช่น ลด 15–20% ให้ลูกค้าทุกคนโดยไม่แบ่งกลุ่ม ส่งผลให้เกิดปัญหา 2 ด้านพร้อมกัน

```
❌ Margin รั่วไหล        →  ลูกค้าที่จะซื้ออยู่แล้วได้ส่วนลดฟรีโดยไม่จำเป็น
❌ โอกาสถูกมองข้าม      →  ลูกค้า At-Risk ที่กำลังจะหายไปไม่ได้รับโปรที่ตรงกลุ่ม
```

### KPI ที่ต้องการปรับปรุง

| KPI | นิยาม | เป้าหมาย |
|-----|--------|---------|
| **Incremental Revenue per Promo** | Revenue จริงที่เพิ่มขึ้นหลังหักส่วนลด | เพิ่มจาก baseline |
| **Promotion ROI** | (Revenue เพิ่ม − ต้นทุนส่วนลด) / ต้นทุน | > 0% (ทุกบาทที่ลดมีผลตอบแทน) |
| **Offer Acceptance Rate** | % ลูกค้าที่ซื้อหลังได้รับโปรที่ recommend | สูงกว่า random baseline |

---

## 💡 How — ทำอย่างไร?

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        INPUT DATA                           │
│        Sales Transaction  ·  Customer Master                │
│              Promotion Master  ·  Product Master            │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│           LAYER 1 — RFM + K-Means Clustering                │
│                                                             │
│  Recency · Frequency · Monetary  →  Normalize  →  K-Means   │
│  Elbow + Silhouette Score เพื่อหา optimal k                   │
│                                                             │
│  Output: {Champion | Loyal | At-Risk | New}                 │
└──────────────────────────────┬──────────────────────────────┘
                               │ segment → feature ของ Layer 2
                               ▼
┌─────────────────────────────────────────────────────────────┐
│           LAYER 2 — Revenue Regression (Ridge)              │
│                                                             │
│  Input: RFM + segment                                       │
│  → ทำนาย predicted_spend_next_30d                           │
│  → แยก "ซื้ออยู่แล้ว" vs "ต้องกระตุ้น"                              │
│  → วัดด้วย R² และ Residual Plot                               │
│                                                             │
│  ⚡ เหตุผลที่ใช้ Ridge: RFM มี multicollinearity สูง               │
│     L2 regularization ช่วย stabilize coefficient             │
│                                                             │
│  Output: predicted_spend_30d  ←  key feature ของ Layer 3    │
└──────────────────────────────┬──────────────────────────────┘
                               │ business context ฝังใน model
                               ▼
┌─────────────────────────────────────────────────────────────┐
│           LAYER 3 — Promotion Classifier (XGBoost)          │
│                                                             │
│  Input: RFM + segment + predicted_spend + promo_type        │
│  → Train/Test 80:20                                         │
│  → Confusion Matrix + Feature Importance + Lift Curve       │
│                                                             │
│  Output: redeem_probability per customer per promotion      │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                    NEXT-BEST-OFFER TABLE                    │
│    customer_id · segment · predicted_spend                  │
│    recommended_promo · redeem_probability · action          │
└─────────────────────────────────────────────────────────────┘
```

### Why 3 Layers? (ไม่ใช่แค่ XGBoost ตรงๆ)

> Layer 2 (Regression) คือจุดต่างหลักของ solution นี้  
> มันตอบคำถามที่ rule-based system ตอบไม่ได้ว่า  
> *"ถ้าไม่มีโปรเลย ลูกค้าคนนี้จะซื้อเองเท่าไหร่?"*  
> ทำให้ XGBoost มี business context ฝังอยู่ข้างใน  
> และช่วย save margin จากลูกค้าที่ซื้ออยู่แล้วได้จริง

---

## 📁 Repository Structure

```
smart-promo-targeting/
│
├── 📄 README.md                        ← คุณอยู่ที่นี่
│
├── 📁 data/
│   ├── raw/                            ← Mock CSV 5 ไฟล์ (ห้ามแก้ไข)
│   │   ├── sales_transaction.csv       (~3,000 transactions · 6 เดือน)
│   │   ├── customer_master.csv         (500 customers)
│   │   ├── promotion_master.csv        (10 promotions)
│   │   ├── product_master.csv          (50 products)
│   │   └── store_master.csv            (5 สาขา)
│   └── processed/                      ← Cleaned data หลัง EDA
│       ├── rfm_features.csv            ← RFM ที่คำนวณแล้ว
│       └── transactions_with_segment.csv
│
├── 📁 notebooks/
│   ├── 01_eda.ipynb                    ← Data Quality + RFM Engineering
│   ├── 02_clustering.ipynb             ← K-Means Segmentation
│   ├── 03_regression.ipynb             ← Revenue Forecast (Ridge)
│   └── 04_xgboost.ipynb               ← Promotion Classifier + Output
│
├── 📁 diagrams/
│   ├── solution_workflow.png           ← Pipeline overview
│   ├── 01_revenue_distribution.png
│   ├── 01_rfm_distributions.png
│   ├── 01_rfm_by_segment.png
│   ├── 01_rfm_correlation.png
│   └── 01_promo_usage_by_segment.png
│
├── 📁 docs/
│   └── data_dictionary.md             ← อธิบาย column ทุกตัวใน 5 ตาราง
│
└── 🐍 mock_data.py                    ← Script สร้าง mock data
```

---

## 🚀 Getting Started

### 1. Clone Repository

```bash
git clone https://github.com/[your-username]/smart-promo-targeting.git
cd smart-promo-targeting
```

### 2. Install Dependencies

```bash
pip install pandas numpy scikit-learn xgboost matplotlib seaborn jupyter
```

### 3. Generate Mock Data

```bash
python mock_data.py
# Output: data/raw/*.csv (5 ไฟล์)
```

### 4. Run Notebooks (ตามลำดับ)

```bash
jupyter notebook notebooks/01_eda.ipynb
jupyter notebook notebooks/02_clustering.ipynb
jupyter notebook notebooks/03_regression.ipynb
jupyter notebook notebooks/04_xgboost.ipynb
```

> 💡 **Google Colab:** Upload notebook + CSV ไฟล์ แล้วรัน Runtime → Run all

---

## 📊 Expected Results

| Deliverable | รายละเอียด |
|-------------|-----------|
| Customer Segment Profiles | 4 กลุ่ม พร้อม RFM mean per cluster |
| Revenue Regression Report | R² score + Residual plot + Feature coefficients |
| XGBoost Performance | Confusion Matrix + Feature Importance + Lift Curve |
| Next-Best-Offer Table | customer_id → recommended_promo → action |

### Validation Strategy

```
1. Back-test บน holdout set (20% ของ transaction data)
2. Lift Curve — เปรียบเทียบ targeted vs random promotion
3. Precision@K — model recommend top-K customers ได้ตรงแค่ไหน
4. Baseline comparison — Promo ROI ก่อน vs หลังใช้ model
```

---

## 📅 Project Roadmap

```
Phase 1 — Segment & Predict  (สัปดาห์ 1–2)
  ✅ EDA + Data Quality Check
  ✅ RFM Feature Engineering
  🔄 K-Means Clustering
  🔄 Ridge Regression Forecast

Phase 2 — Deploy Targeting  (สัปดาห์ 3–4)
  ⬜ XGBoost Classifier
  ⬜ Next-Best-Offer Table
  ⬜ Back-test Validation

Phase 3 — Extend  (Optional)
  ⬜ Time-Series Forecast component
  ⬜ A/B Test framework concept
  ⬜ Mock Dashboard (Streamlit / Power BI)
```

---

## 🤖 AI Tools Used

| Tool | ใช้ทำอะไร | Verified โดย |
|------|-----------|-------------|
| Claude (Anthropic) | Problem framing · Statistical logic review | Domain knowledge จากวิชา ML + Regression |
| GitHub Copilot | Code autocomplete ใน notebooks | รัน output และตรวจทุก cell ด้วยตัวเอง |

---

## 👤 Author

**[ชื่อ-นามสกุล]**  
สาขาวิชาวิทยาศาสตร์ข้อมูลและการวิเคราะห์เชิงสถิติ  
คณะวิทยาศาสตร์ประยุกต์ · [ชื่อมหาวิทยาลัย]

Cooperative Education @ **Botnoi Group**  
2 มิถุนายน 2026 – 30 ตุลาคม 2026

---

<p align="center">
  <sub>Built with ❤️ for Co-operative Education · Statistics & Data Science</sub>
</p>
