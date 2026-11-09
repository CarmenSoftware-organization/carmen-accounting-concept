# Review: AR Profile & Contract Management FRD v1.2

**เอกสารที่ review:** [Drive › AR › AR Profile](https://drive.google.com/drive/folders/12T6wv6QN_j4eYMA1jfCgK6Lbw-ccoglH?usp=drive_link) — `AR Profile & Contract FRD.md` (FRD v1.2, 613 บรรทัด) + mockup v1.2 (ไม่ได้เปิดอ่านละเอียด ใช้ FRD เป็นหลัก)
**วันที่ review:** 2026-11-09
**Backend ที่เทียบ:** `carmen-turborepo-backend` main @ `1abb6a9c0` (customer master + ARIV/ARDN/ARCN merged จาก PR #704)
**ผู้รับ:** ทีม dev / BA
**หมายเหตุที่มาของไฟล์:** อ่านจากสำเนาในเครื่องของโฟลเดอร์ Drive (ฉบับวันที่ 2026-11-08) — อ่านอย่างเดียว เลขบรรทัด FRD ด้านล่างอ้างจากฉบับนี้

**กรอบการอ่าน:** FRD ไม่ใช่ spec ของเรา ทีมหยิบ concept มาปรับ จึงจัดเป็น **Adopt / Adapt / Reject-Defer** ไม่ใช่ fix-list ของผู้เขียน ความไม่สอดคล้องภายใน FRD ระบุเป็นความเสี่ยงของเราเท่านั้น

---

## 1. สรุป verdict

**Adapt เป็นหลัก — ส่วน profile เป็น delta เล็ก ส่วน contract เป็นงานใหม่ทั้งก้อน**

1. **Profile master มีแล้ว ~70%** `tb_customer` + `tb_customer_address` (schema.prisma:7641-7715) ครอบ code/name/tax_no/branch_no/currency/credit term/credit limit/AR account/tax profile/ที่อยู่ 4 ประเภท รวมถึง 5-part Thai address (sub_district/district/province/postal_code) delta จริงคือ tax entity type, contact fields, default payment type, WHT default, billing/mailing contact+email, attachments
2. **FRD ใช้ schema คนละโลก** MySQL, `carmen_ar_*`, PK คอมโพสิต `(business_unit_id, ar_no)`, ฟิลด์ code แทน uuid FK — ห้ามนำ DDL §5 มาใช้ตรง ๆ ต้อง map เข้า Prisma/uuid/tenant-schema ตาม convention ฝั่ง AP
3. **Tax default ยังไม่ถูกใช้จริง** `tb_customer.tax_profile_id` มีแล้วแต่ ARIV ไม่อ่านเลย (`resolveHeader` ดึงแค่ currency / credit_term_days / ar_chart_of_accounts — ar-invoice.service.ts:840-868) บรรทัด ARIV รับ `vat_tax_profile_id` จาก payload เท่านั้น
4. **FRD ขัดกับ ARIV ที่ implement แล้วเรื่อง Tax 2 = WHT** contract line ตั้ง `tax2_profile = 'WHT03'` (FRD:208, :375) แต่ ARIV ปฏิเสธด้วย `AR_TAX2_WHT_NOT_ALLOWED` (ar-invoice.validation.ts:275-279) — Apply Contract จะสร้าง invoice ไม่ผ่าน
5. **Credit limit: FRD ไม่มีกฎบังคับเลย** เป็นแค่ค่าที่เก็บ (FRD:139) backend ก็เก็บอย่างเดียว (spec 2026-10-01 §5 "ไม่บังคับ"; ไม่มี `credit_limit` ใน logic ของ ARIV) — guardrail ของ Folio "Switch A/R" ต้องนิยามกฎใหม่ทั้งหมด
6. **Contract = ตารางใหม่ 3 ตาราง + engine** ยังไม่มี contract/recurring ใน backend (grep ศูนย์) และ FRD ตัด Apply Contract engine ออกจาก scope (FRD:56) — ได้แค่ master ไม่ได้ billing จริง

**Breaking change ต่อ column/API เดิม: ไม่มีถ้าทำแบบ additive** (ทุกฟิลด์ใหม่ nullable/มี default) จุดที่จะ break คือถ้านำ FRD มาใช้ตรง ๆ — ดู §5

---

## 2. Adopt (ใช้ตามที่เขียน)

| # | องค์ประกอบ | FRD | หลักฐานฝั่งเรา | หมายเหตุ |
|---|-----------|-----|---------------|---------|
| A1 | Soft deactivate แทน hard delete เมื่อมี transaction (`ERR_DEL_HAS_TXN`) | :576 | spec ARIV §5: delete = soft + `CUSTOMER_IN_USE` | ตรงกันอยู่แล้ว |
| A2 | Default credit term (30 วัน) → due date | :139 | `credit_term_days` + `due_date = doc_date + days` (ar-invoice.service.ts:864) | มีแล้ว |
| A3 | Thai RD 5-part address ที่ Tax address | :157-162 | `tb_customer_address` `register_address` มี line1/sub_district/district/province/postal_code (schema:7692-7699) | ครบ; postal 5 หลัก ให้ validate ที่ service |
| A4 | แยก Billing กับ Mailing address | :165-177 | enum `billing_address`, `mailing_address` มีแล้ว (schema:397-402) | เพิ่มแค่ contact/email (ดู C2) |
| A5 | Tax ID 13 หลัก + branch 5 หลัก, default `00000` | :148-149 | `CUSTOMER_TAX_NO_INVALID` (customers.service.ts:37-42) | ตรงกัน |
| A6 | Optimistic lock ด้วย `updated_at` | :598 | `doc_version` ใน tb_customer | ใช้ `doc_version` ที่มี |
| A7 | Audit trail / Activity log | :597 | activity-registry มี `tb_customer` (spec §5) | ไม่ต้องสร้าง `carmen_ar_activity_log` — ใช้ระบบกลางที่มี |
| A8 | RBAC แยกสิทธิ์แก้ credit limit ให้ FC | :593 | — | แนวคิดเอาไปใช้ แต่ต้อง field-level (ดู D3) |
| A9 | ข้อกำหนดจำกัดไฟล์ 25 MB | :195, :577 | — | ตรวจกับ storage service กลางก่อนระบุตัวเลข |

## 3. Adapt (หยิบมาแต่ต้องเปลี่ยน)

| # | องค์ประกอบ | FRD | เปลี่ยนเป็น | เหตุผล |
|---|-----------|-----|------------|--------|
| B1 | PK `(business_unit_id, ar_no)` + `carmen_ar_profile` | :297 | ใช้ `tb_customer.id` uuid + `code` (unique `customer_code_u`) ไม่มี `business_unit_id` | tenant-per-BU schema อยู่แล้ว; FK ทุกตารางอ้าง uuid |
| B2 | A/R No. regex `[A-Z0-9_-]`, ห้ามไทย, `VARCHAR(15)` | :116, :298 | validate ที่ service (เพิ่ม rule ใน customers.service) ไม่ใส่ MySQL `REGEXP` CHECK; `code` ปัจจุบัน `VarChar` ไม่จำกัดความยาว | ฝั่งเรายังไม่มีกฎนี้; เป็นการเพิ่ม validation ถ้าตกลง ต้องไม่ย้อนทำลาย code เดิม (ตรวจก่อนว่ามีข้อมูลเดิมที่ผิดกฎหรือไม่) |
| B3 | Customer type `TA/CORP/FIT/GOV` (VARCHAR) | :117 | ใช้ `customer_group Json [{id,name}]` เดิม (schema:7647) | pattern เดียวกับ `tb_vendor.business_type`; เพิ่มค่า seed TA/CORP/FIT/GOV |
| B4 | Tax entity CORPORATE/INDIVIDUAL + `tax_registered_name` NOT NULL + `tax_id` NOT NULL | :267-269, :299 | เพิ่ม `tax_entity_type` enum (nullable, default null/`corporate`); `registered_name`, `tax_no` **คง nullable** | ลูกค้าต่างประเทศ/OTA ไม่มี tax ID (comment ใน schema:7650); NOT NULL จะ block การสร้าง customer ต่างประเทศ และ `tax_no` สำหรับ Individual เก็บ Citizen ID ลงช่องเดียวกัน — ต้อง mask/สิทธิ์เข้าถึง (PDPA) |
| B5 | Contact fields บน header (title/first/last/position/telephone/fax/email) | :93-95, :121-125 | **ไม่เติมคอลัมน์บน tb_customer** — สร้าง `tb_customer_contact` ตาม `tb_vendor_contact` (schema:4155) | vendor ใช้ตารางย่อยอยู่แล้ว; ลูกค้าหนึ่งรายมีผู้ติดต่อหลายคน; ผู้ติดต่อหลัก = `is_primary` |
| B6 | Billing/Mailing: `contact_name`, `email`, `is_*_same_as_tax`, `*_address_text` (ข้อความเดียว 500 ตัว) | :280-289 | เก็บ contact/email ลง `tb_customer_address` (เพิ่ม `contact_name`, `email` nullable) และให้ "Same as Tax" เป็น UI-only (copy ค่า ไม่เก็บ flag) ไม่ใช้ `address_text` | ที่อยู่เราแยก field แล้ว; ข้อความรวม + flag ซ้ำซ้อนและ drift ได้; ARIV snapshot ที่อยู่จาก `register_address` → `billing_address` อยู่แล้ว (spec ARIV §9) |
| B7 | `default_tax_profile` string `VAT07_ADD/INC/NONE` | :136, :261 | ใช้ `tax_profile_id` FK เดิม (ต้อง `tax_type = vat`) และ **ทำให้ ARIV อ่านค่านี้ตอนสร้างบรรทัดใหม่** | ดู §4.2; `VAT07_INC` ARIV ยังไม่รองรับ (spec ARIV out-of-scope: VAT inclusive) |
| B8 | `default_wht_service_type` + `master_wht_service_type` 5 ค่า (rate ฝัง 3/5/2/1/3%) | :137, :468 | เพิ่ม `wht_tax_profile_id` FK → `tb_tax_profile` (`tax_type = wht`, มี `wht_pnd_form`, `wht_income_type` อยู่แล้ว schema:1631-1634) | ไม่สร้าง master ซ้ำ; `tb_ar_receipt_wht.wht_tax_profile_id` (schema:8094) ใช้ FK เดียวกัน; rate เปลี่ยนได้ที่ tax profile |
| B9 | `default_payment_type` FK `master_payment_type` NOT NULL | :135, :260 | **ต้องเช็คก่อนว่ามี payment-type master หรือไม่** (ไม่เจอ `master_payment_type` ใน tenant schema; มี `tb_payment_*`/enum อื่นที่ต้องตรวจ) ค่อยเพิ่มเป็น nullable FK | ARIV ไม่มีฟิลด์ payment type ใน header — ค่านี้ใช้ที่ ARRC (Receipt) มากกว่า; Defer ไปพร้อม Receipt ก็ได้ |
| B10 | Credit limit `credit_limit_thb` | :139, :264 | คง `credit_limit Decimal(20,5)` (มีแล้ว) แต่ **ต้องตัดสินสกุลเงิน** (ดู §4.3) | FRD ตายตัว THB ขณะที่ customer มี `default_currency` |
| B11 | Contract header | :306-333 | สร้าง `tb_ar_contract` (uuid, `customer_id` FK, `contract_no` unique, `start_date`, `end_date`, `charge_every_months`, `currency_id/code`, `is_active`, audit 6 คอลัมน์, `deleted_at`) + เพิ่ม column ที่ FRD ขาด (ดู §4.4) | convention Decimal(20,5), rate (15,5) |
| B12 | Contract detail 4 ส่วน | :338-396 | `tb_ar_contract_detail` โครงเหมือน `tb_ar_invoice_detail` (FK id ของ chart_of_accounts/cost_center/tax_profile แทน code string) + dimension ตาม pattern `tb_ar_invoice_detail_dimension` | FRD เก็บ `cr_revenue_acc_code` VARCHAR และ dim 6 คอลัมน์ตายตัว — ไม่ตรงกับ dimension model ที่ใช้อยู่ (JSON/ตารางย่อย) |
| B13 | เลขสัญญา `YYYY+xxxxx` ด้วย `MAX(contract_no)` | :229, :533 | ใช้ running-code service (เหมือน `AR-IV` ใน ar-invoice.running-code.ts) เพิ่ม type `AR-CT` | `MAX()+1` race ตอนสร้างพร้อมกัน; running-code มี lock อยู่แล้ว |
| B14 | แนบไฟล์ contract / profile เป็นตารางแยก | :403-429 | ใช้ `attachments Json` ตาม tb_ar_invoice (schema ARIV) หรือ storage service กลาง | ไม่มีตาราง attachment แยกใน AR ปัจจุบัน; `storage_path` + encryption AES-256 เป็นเรื่องของ storage layer |
| B15 | Apply Contract → ARIV (batch) | :470, :556-561 | เก็บเป็น **ระยะถัดไป**; ถ้าทำ ต้องเพิ่ม `contract_id` + `contract_period` บน `tb_ar_invoice` (nullable) และ unique กันออกซ้ำ | ดู §4.4 |

## 4. Deep-dive ตามโจทย์

### 4.1 Field-by-field: FRD profile vs `tb_customer`

| FRD field | tb_customer วันนี้ | สถานะ |
|-----------|--------------------|-------|
| ar_no | `code` | มีแล้ว (ชื่อต่าง) |
| customer_type | `customer_group` Json | มีแล้ว (รูปแบบต่าง) |
| register_date | `created_at` | แนะนำใช้ `created_at` หรือ `info.register_date` — ยังไม่ต้องเพิ่ม column ถ้าไม่ต้องย้อนหลัง |
| is_active | `is_active` | มีแล้ว |
| company_name | `name` | มีแล้ว |
| contact title/first/last/position, telephone, fax, email | — | ใหม่ → `tb_customer_contact` (B5) |
| remark | `note` / `description` | มีแล้ว |
| default_payment_type | — | ใหม่/Defer (B9) |
| default_tax_profile | `tax_profile_id` | มี column; **ยังไม่ถูกใช้** (4.2) |
| default_wht_service_type | — | ใหม่ → `wht_tax_profile_id` (B8) |
| credit_term_days | `credit_term_id/name/days` | มีแล้ว (snapshot 3 ตัว) |
| credit_limit_thb | `credit_limit` | มี column; ไม่ถูกบังคับ (4.3) |
| tax_entity_type | — | ใหม่ (B4) |
| tax_registered_name | `registered_name` | มีแล้ว |
| tax_id / citizen id | `tax_no` | มีแล้ว (nullable) |
| branch_no | `branch_no` | มีแล้ว |
| tax address 5 ส่วน | `tb_customer_address` `register_address` | มีแล้ว |
| billing/mailing contact+email+address | `tb_customer_address` billing/mailing | address มี; contact/email ใหม่ (B6) |
| business_unit_id | — | ไม่ต้อง (tenant schema) |
| — (เรามี แต่ FRD ไม่มี) | `default_currency_*`, `ar_chart_of_accounts_id`, `pms_account_code`, `dimension` | คงไว้; `pms_account_code` จำเป็นต่อ Folio |

### 4.2 Tax defaulting: FRD vs ARIV วันนี้

- ARIV วันนี้: VAT/Tax 2 มาจาก **payload ต่อบรรทัด** (`vat_tax_profile_id`, `tax2_tax_profile_id` — ar-invoice.service.ts:1041, :1046) ไม่มีที่ไหนอ่าน `customer.tax_profile_id`
- FRD ต้องการ: profile default → ตั้งต้นใน AR Invoice (FRD:135-136 ระบุ payment type "ส่งต่อเป็นค่าตั้งต้น")
- ข้อเสนอ: ทำที่ service (ไม่ใช่ UI อย่างเดียว) — ถ้าบรรทัดไม่ส่ง `vat_tax_profile_id` ให้ fallback เป็น `customer.tax_profile_id`; ถ้าส่ง `null` โดยชัดเจนหมายถึงไม่มี VAT (ต้องแยก `undefined` กับ `null` ใน DTO) เป็นการเปลี่ยน behavior ของ create ARIV จึงต้องแจ้ง frontend
- `VAT07_INC` (ราคารวม VAT) ยังไม่รองรับใน ARIV → ถ้า customer เลือก INC จะ default ไม่ได้ ต้อง Defer หรือ block ที่ customer form จนกว่า ARIV รองรับ
- **WHT:** FRD ให้ `WHT Service Type` ที่ profile แล้วไปตัดที่ Receipt (FRD:137) — สอดคล้องกับแนวของเรา (WHT รับรู้ที่ ARRC, header `wht_amount` เป็น reference) **ไม่ขัดกับ OI-6** แต่ **ส่วน contract line ที่ตั้ง Tax 2 = WHT 3% (FRD:208) ขัด** → ใน contract ให้ Tax 2 เป็นภาษีขายเสริมเท่านั้น และ WHT ใช้ `wht_tax_profile_id` ที่ customer ไปคำนวณ `wht_amount` reference ตอนสร้าง ARIV จาก contract

### 4.3 Credit limit (รวม guardrail ของ Folio "Switch A/R")

สิ่งที่ FRD กำหนด: ค่าเดียว `credit_limit_thb` NOT NULL default 0 (FRD:139, :264), แก้ได้เฉพาะ FC (FRD:593) **ไม่มีกฎ** ว่าตรวจเมื่อไร นับอะไร บล็อกหรือเตือน ไม่มี error code ที่เกี่ยวข้อง (§8 ไม่มี `ERR_CREDIT_*`) และไม่มีการกล่าวถึง Folio

ปัญหาถ้าเอาไปใช้เป็น guardrail:

- `default 0` กำกวม — 0 = ไม่อนุญาตเครดิตเลย หรือ = ไม่จำกัด? ปัจจุบัน backend ใช้ `0` เป็น default และ "เก็บอย่างเดียว" ถ้าเริ่มบังคับ ลูกค้าเดิมทั้งหมดที่เป็น 0 จะถูกบล็อกพร้อมกัน → **ต้องมีค่าแยกความหมาย** (nullable = ไม่จำกัด หรือ flag `credit_check_mode`)
- ยอดที่ใช้เทียบ: `Σ tb_ar_invoice.outstanding_amount` (เป็นสกุลเอกสาร ต้องแปลง base) ของสถานะ posted, ลบ ARDP/ARCN ที่ยังไม่ตัด ฯลฯ — ยังไม่มี query ยอดค้างระดับ customer (aging out-of-scope ใน spec ARIV §39)
- สกุลเงิน: FRD THB เท่านั้น; `outstanding_amount` เก็บตามสกุลเอกสาร และมี `base_total_amount` — ต้องเลือกเทียบเป็น base currency
- Folio "Switch A/R": ย้ายยอด folio ไปลูกหนี้อีกราย ต้องตรวจ limit ของ **ผู้รับ** ก่อนย้าย แนะนำ service กลาง `CreditCheckService.check(customer_id, add_base_amount)` คืน `{ok, available, exposure}` ให้ทั้ง ARIV submit และ Folio เรียก (ใช้ร่วม ไม่ copy logic)
- Soft หรือ Hard + ใครอนุมัติ override — ต้องตัดสินใจ (Q5-Q7)

### 4.4 Contract: ตารางใหม่และช่องว่างของ FRD

FRD ให้ 3 ตาราง contract (header, detail, attachment) — ต้องสร้างใหม่ทั้งหมด (ฝั่ง backend ไม่มี) ข้อสังเกตด้าน "rates/validity" ที่ถามมา:

- **ไม่มี negotiated room rate / rate card** ใน FRD — "Rate" ในตารางสัญญา (FRD:185) คือ exchange rate (FRD:193, :318) ไม่ใช่ราคาห้อง สัญญา = ใบสั่ง recurring billing เท่านั้น ถ้าธุรกิจคาดหวังสัญญาราคา corporate rate ต้องเป็นงานแยก (Defer)
- **Validity:** มีแค่ `start_date`/`end_date` NOT NULL + CHECK (FRD:314-331) ไม่มี renewal, ไม่มี auto-expire, ไม่มีกฎ overlap ของสัญญาลูกค้าเดียวกัน, ไม่มี `next_bill_date` / `last_billed_period`
- **Idempotency ของ Apply Contract ไม่มี:** ต้องมี `contract_id` + `period_from/to` บน invoice และ unique (`contract_id`, `period_from`, `deleted_at`) ไม่งั้นรันซ้ำแล้วออก invoice ซ้ำ
- **Exchange rate บนสัญญา** (FRD:318) ควรเป็น "สกุลเงิน" เท่านั้น — rate ต้องดึง ณ วันออก invoice (ARIV `resolveRate` ทำอยู่แล้ว ar-invoice.service.ts:851) ไม่ใช่ค่าที่ล็อกไว้ตอนสร้างสัญญา
- `total_amount` บน header (FRD:320) เป็น derived → คำนวณ ไม่เก็บ (หรือเก็บแบบ cache พร้อมกฎ recalc)
- `ERR_CT_UNBALANCED` (FRD:574) "Debit = Credit" ในบรรทัดเดียว — เป็นแค่ Net + Tax = Total ซึ่ง ARIV คำนวณเองอยู่แล้ว (ar-invoice.logic.ts:53-65) ใช้ logic เดียวกัน ไม่เขียนซ้ำ
- Section 4 `dr_ar_acc_code` ต่อบรรทัด: ปัจจุบัน ARIV default จาก `customer.ar_chart_of_accounts_id` → `gl_setting.ar_control_account_id` (service:865-866) — ไม่ควร hardcode/เก็บซ้ำในสัญญา ให้ nullable แล้ว fallback เหมือนเดิม

---

## 5. Schema delta เทียบ `tb_customer` ปัจจุบัน

### เพิ่ม (additive, ไม่ break)

| ตาราง | Column / ตาราง | หมายเหตุ |
|-------|----------------|----------|
| `tb_customer` | `tax_entity_type` enum(`corporate`,`individual`) nullable | B4 |
| `tb_customer` | `wht_tax_profile_id` uuid? FK → `tb_tax_profile` | B8 |
| `tb_customer` | `default_payment_type` (รอผลตรวจ master) | B9 / Defer ได้ |
| `tb_customer` | `credit_check_mode` enum(`none`,`warn`,`block`) default `none` | ถ้าตกลงทำ guardrail (4.3) — default `none` เพื่อไม่ break |
| `tb_customer_address` | `contact_name`, `email` nullable | B6 |
| `tb_customer_contact` (ใหม่) | name, title, position, phone, fax, email, is_primary | B5, mirror `tb_vendor_contact` |
| `tb_ar_contract` (ใหม่) | ตาม B11 + `next_bill_date`, `last_billed_period_to` | B11 |
| `tb_ar_contract_detail` (ใหม่) | mirror `tb_ar_invoice_detail` | B12 |
| `tb_ar_contract_detail_dimension` หรือ Json | ตาม pattern ARIV | B12 |
| `tb_ar_invoice` | `contract_id` uuid?, `contract_period_from/to` date? + unique | B15 (เฉพาะเมื่อทำ Apply Contract) |
| running code | type `AR-CT` | B13 |

### เปลี่ยน
| รายการ | จาก → เป็น |
|--------|-----------|
| `ARIV create` | บรรทัดที่ไม่ส่ง `vat_tax_profile_id` → fallback `customer.tax_profile_id` (behavior change ระดับ service ไม่ใช่ schema) |
| `customers.service` | เพิ่ม validation ของ `code` (charset/ความยาว) ถ้าตกลงรับ B2 |

### Breaking (เฉพาะถ้าเอา FRD มาใช้ตรง ๆ — **ไม่แนะนำ**)
| FRD | ผลกระทบ |
|-----|---------|
| PK `(business_unit_id, ar_no)` แทน uuid | FK ของ `tb_ar_invoice`, `tb_ar_receipt` (customer_id) พังทั้งหมด |
| `tax_id`, `tax_registered_name`, `billing_address_text`, `billing_email` **NOT NULL** | ลูกค้าเดิม/ลูกค้าต่างประเทศที่ nullable จะสร้าง/แก้ไม่ได้; migration ต้อง backfill |
| `default_payment_type`, `default_tax_profile`, `default_wht_service_type` NOT NULL | create customer เดิม (API ปัจจุบันไม่ส่ง) ได้ 400 |
| `ar_no` regex CHECK | code เดิมที่มีไทย/อักขระอื่นจะ violate ตอน migrate |
| rename `credit_limit` → `credit_limit_thb` / ตัดสกุลเงิน | DTO/serializer `ICreateCustomer.credit_limit` (customers.interface.ts:54) เปลี่ยน → client พัง |
| เปลี่ยน credit limit จาก "เก็บอย่างเดียว" เป็น hard block | ลูกค้าที่ `credit_limit = 0` ทั้งหมดออก ARIV ไม่ได้ |

**API เดิมที่ต้องคง:** gateway `api/config/:bu_code/customers` GET/POST/PUT/DELETE, RPC `customers.*`, ฟิลด์ `credit_limit`, `tax_profile_id`, `customer_address` (add/update/remove) — เพิ่มฟิลด์ได้ ห้ามเปลี่ยนชื่อ/ทำ required

---

## 6. Reject / Defer

| # | รายการ | FRD | เหตุผล |
|---|--------|-----|--------|
| R1 | DDL MySQL (`AUTO_INCREMENT`, `ON UPDATE CURRENT_TIMESTAMP`, `REGEXP`, `BIGINT` id) | :241-441 | ใช้ Postgres/Prisma/uuid |
| R2 | ตาราง `carmen_ar_activity_log` (+ `ip_address`) | :431-440 | มี activity registry กลางแล้ว; ถ้าต้องการ IP ให้ขยายระบบกลาง |
| R3 | `master_wht_service_type` ตารางใหม่ | :468 | ใช้ `tb_tax_profile` (B8) |
| R4 | Contract line Tax 2 = WHT | :208, :375 | ขัด `AR_TAX2_WHT_NOT_ALLOWED` |
| R5 | Apply Contract engine + billing scheduler | :56, :556 | FRD ตัดออกเองและไม่กำหนดพฤติกรรม; Defer เป็น spec แยก (ต้องใช้ micro-cronjobs) |
| R6 | `VAT07_INC` เป็น default | :136 | ARIV ยังไม่รองรับ VAT inclusive |
| R7 | UI-only: auto-suggest prefix จากตัวอักษรแรก, 12-col grid, quick switcher, DOC viewer | :68-70, :104-113 | เป็นงาน frontend ไม่กระทบ backend; frontend ตัดสินเอง |
| R8 | Test TC-ARP-001 ผูกข้อมูลตัวอย่าง "11 records" / pre-rendered HTML | :606 | เป็น test ของ mockup ไม่ใช่ของ backend |
| R9 | NFR: client-side filter < 100 ms | :586 | เรา paginate server-side (`findAll` search) |

## 7. ความเสี่ยงจากความไม่สอดคล้องภายใน FRD (สำหรับเรา ไม่ใช่ fix-list)

| # | จุดที่ขัดกัน | ผลกับเรา |
|---|-------------|---------|
| I1 | §3.3 บอก `default_wht_service_type` ส่งไป AR Receipt แต่ §4 Save Profile / §6 ไม่ระบุว่า Receipt อ่านอย่างไร | ต้องออกแบบ contract ระหว่าง customer ↔ ARRC เอง (Receipt ยังไม่ทำ) |
| I2 | `billing_address_text NOT NULL` + `billing_email NOT NULL` (:282-283) แต่ "Same as Tax" ให้ copy ได้ — ถ้า tax address ว่าง (ลูกค้าต่างประเทศ) จะ NOT NULL ไม่ผ่าน | อย่า adopt NOT NULL |
| I3 | Individual ต้องกรอก First/Last name (:119-120) แต่ `tax_registered_name NOT NULL` (:268) — ไม่บอกว่า Individual ใช้ชื่ออะไร | ต้องตัดสิน derive `registered_name` ของ Individual |
| I4 | §3.6 ใช้ "Tax 2 (WHT 3%)" ขณะที่ฝั่ง AR Invoice FRD v1.07 OI-6 ยังไม่ปิด | อ้างอิงข้อ R4; ติดตาม OI-6 |
| I5 | เลขสัญญาใช้ปี ค.ศ. ของ Start Date แต่ตัวอย่าง flow เรียก "Year = 2026" และ query `MAX` ตามปี — ถ้าแก้ start date ข้ามปีหลังสร้าง เลขไม่ตรง | ล็อกเลขเมื่อสร้าง ห้ามคำนวณใหม่ |
| I6 | ปุ่ม "+ ADD CONTRACT" ต้องมี `ar_no` ที่บันทึกแล้ว (:230) และ Save Contract "ทันที" แยกจาก Edit Profile — ไม่ชัดว่า contract ของ profile ที่ inactive ยังสร้าง/ออก invoice ได้ไหม | ต้องกำหนด: inactive customer → `CUSTOMER_INACTIVE` ถูกบล็อกอยู่แล้วที่ ARIV (validation.ts:49) |
| I7 | Status "Approved for Implementation" (:7) แต่ไม่มี sign-off/OI list เหมือน FRD v1.07 | อย่าถือว่าตกลงทางธุรกิจแล้ว — ยืนยันกับ BA |

---

## 8. คำถามเปิดสำหรับ BA / ทีม (11 ข้อ)

1. **Credit limit semantics:** `0` = ไม่อนุญาตเครดิต หรือ ไม่จำกัด? ต้องการ `null`/flag แยกหรือไม่
2. **สกุลเงิน credit limit:** THB อย่างเดียว หรือ base currency ของ BU หรือ ตาม `default_currency`
3. **Exposure นับอะไร:** invoice posted ค้างชำระ + ARDN − ARCN/ARDP ที่ยังไม่ตัด? รวม folio ที่ยังไม่ billed หรือไม่
4. **Soft/Hard:** เตือนหรือบล็อก ใครอนุมัติ override (FC?) และต้องเก็บ audit ของ override ไหม
5. **Folio "Switch A/R":** ตรวจ limit ของลูกหนี้ปลายทางตอนย้ายยอดใช่หรือไม่ และขอบเขต (ยอดที่ย้าย + exposure เดิม)
6. **Tax 2 ในสัญญา:** ยืนยันว่า WHT ไม่เข้าบรรทัด contract (ตาม OI-6 resolution) ใช้ `wht_tax_profile_id` ที่ customer คำนวณ reference
7. **Default tax profile:** ให้ ARIV fallback จาก customer อัตโนมัติ (service-level) หรือให้ frontend ใส่ค่าตั้งต้นเท่านั้น
8. **Payment type:** มี master อยู่ที่ไหน ต้องการที่ customer จริงหรือ Defer ไป Receipt
9. **Contract scope:** ต้องการแค่ master (เก็บข้อมูลสัญญา) หรือรวม Apply Contract batch — ถ้ารวม ใครเป็นเจ้าของ scheduler และกฎ proration/ข้ามเดือน/สัญญาหมดอายุ
10. **Negotiated rates:** ธุรกิจคาดหวัง corporate rate ในสัญญาไหม (FRD ไม่มี) ถ้าใช่ ต้อง FRD แยก
11. **Individual customer / PDPA:** เก็บเลขบัตรประชาชนใน `tax_no` ต้อง mask/จำกัดสิทธิ์/เข้ารหัสหรือไม่ และ `registered_name` ของ Individual มาจากไหน

## 9. ขั้นตอนถัดไปที่แนะนำ

1. **ตัดสิน Q1-Q5 (credit limit)** ก่อนเริ่ม — กระทบทั้ง ARIV submit และ Folio; ยังไม่แตะ schema จนกว่าตกลง `0` vs `null`
2. **Spec เล็ก 1 ฉบับ "Customer master v2"** (additive): `tax_entity_type`, `wht_tax_profile_id`, `tb_customer_contact`, address contact/email, code validation (B2) + ปรับ ARIV ให้ fallback `tax_profile_id` (§4.2) — ไม่มี breaking, merge ได้เป็น PR เดียวต่อจาก #704
3. **แยก spec Contract** (ตาราง 3 ตาราง + running code `AR-CT` + CRUD) ออกจาก Apply Contract; Apply Contract รอ Q9 และ micro-cronjobs
4. แจ้ง frontend: ฟิลด์ใหม่ทั้งหมด optional, `VAT07_INC` ยังไม่รองรับ, Tax 2 ห้ามเป็น WHT
5. อัปเดต Appendix A / PRD §9 (`docs/PRD-module-ar.md`) ด้วยประเด็นข้างต้นเมื่อตัดสินใจเสร็จ — เอกสารนี้ยังไม่แก้ PRD
