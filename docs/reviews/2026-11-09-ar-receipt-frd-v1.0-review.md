# Review: AR Receipt & Tax Invoice Module FRD v1.0 (New Concept)

**เอกสารที่ review:** [Drive › AR › Receipt](https://drive.google.com/drive/folders/1C03zIg5uKwk2vJoNKSZB9hRUHK-EbAfb?usp=drive_link) — `Functional Requirement Document: AR Receipt & Tax Invoice Module.md` (FRD v1.0, sign-off 27 ก.ย. 2026) + mockup `carmen_ar_receipt_mockup_v1_9.html` (ไม่ได้ไล่ละเอียด อ้างอิงจาก FRD เป็นหลัก)
**วันที่ review:** 2026-11-09
**Backend commit:** `carmen-turborepo-backend` main @ `1abb6a9c0` (ARIV/ARDN/ARCN + customer implement แล้ว PR #704; receipt เป็น schema-only)
**ผู้รับ:** BA / ทีม dev / ที่ปรึกษาบัญชี
**เทียบกับ:** [PRD-module-ar.md](../PRD-module-ar.md) §2.6 · [AR schema spec](../superpowers/specs/2026-09-30-accounting-ar-schema-design.md) · [ARIV service spec](../superpowers/specs/2026-10-01-accounting-ar-invoice-service-design.md) · [AP payment gap review](2026-09-30-ap-payment-frd-v2.16-gap.md)

**กรอบการอ่าน:** FRD นี้เป็น concept ไม่ใช่ spec ของเรา และไม่ใช่หน้าที่เราแก้ ทีมหยิบไปปรับใช้ ตารางด้านล่างจึงจัดเป็น **Adopt** (เอาตามนั้น) / **Adapt** (เอาแต่ปรับ ระบุว่าอะไรและทำไม) / **Reject-Defer** ส่วนความไม่สอดคล้องภายใน FRD บันทึกเป็น "ความเสี่ยงสำหรับเรา" เท่านั้น

---

## 1. สรุป

**Verdict: Adapt เป็นส่วนใหญ่ — ใช้เป็น input ของ spec "AR Receipt" ถัดไปได้ แต่ห้ามทำตาม FRD ตรงๆ ใน 3 เรื่อง (tax invoice, ตาราง/การ track ยอดค้าง, Auto-JV)**

ฝั่ง UX และ business rule (Rule 1–6, Multi-bank, WHT รับ, Other Charges ติดลบ, Tab 2 defaulting จาก customer) ใช้ได้เกือบทั้งหมด และสอดคล้องกับที่ AP Payment ทำไว้แล้ว ส่วนที่ต้องระวัง:

1. **TXRC กับ TXIV ซ้อนกัน (สำคัญสูง)** — FRD ออกใบกำกับภาษีตอน post Receipt "หากเปิด E-Tax หรือมี VAT" (§4 Post Receipt #4) และ partial payment ถอด VAT 7/107 ตามสัดส่วน (§1.2 #3) แต่ ARIV ของเราตั้ง Output VAT และออก `TXIV` ตั้งแต่ post invoice แล้ว ถ้าทำตาม FRD ภาษีขายจะถูกนับสองรอบ และ JV ของ FRD ไม่มีบรรทัด VAT เลย (§6.2)
2. **Schema receipt ที่เรามี (merged, schema-only) ครอบ FRD ได้มากกว่า FRD ครอบเรา** — `tb_ar_receipt/_detail/_wht` + `tb_ar_tax_invoice` (FK ฝั่ง receipt) มีแล้ว ส่วนที่ขาดจริงมี 2 ตาราง: `tb_ar_receipt_method` (multi-bank) และ `tb_ar_receipt_charge` (Other Charges)
3. **FRD ไม่เก็บยอดค้างและอ้าง invoice ด้วย `invoice_no` string** (§5 `ar_receipt_settle`, §6.1) — Reject; เรามี allocation ระดับบรรทัด + `unpaid_amount` ที่ lock ได้ และ AP ผ่าน review มาแล้ว
4. **Auto-JV: โครงตรงกับ `buildPaymentJvLines`** แต่ FRD hardcode เลขบัญชี (`115400X`, `504100X`, `7010000/5050000`) และสูตร FX มีเครื่องหมายตรงข้ามกับ convention ของเรา (§3 ข้อ B3, B4)
5. **`gl_setting` ยังไม่มี `wht_receivable_account_id`** (schema comment อ้างถึง แต่ `GlSetting` ใน `gl-posting.service.ts:91-106` ไม่มี) และไม่มีบัญชีค่าธรรมเนียมธนาคาร ต้องเพิ่มก่อน post ได้

---

## 2. Adopt / Adapt / Reject — Data model และ Settlement

| # | FRD | ที่เรามี (evidence) | Verdict | เหตุผล / สิ่งที่ปรับ |
|---|-----|---------------------|---------|---------------------|
| A1 | `ar_receipt_header` แยกจาก invoice (§5) | `tb_ar_receipt` — `schema.prisma:7979`; ตัดสินใจไว้แล้วใน PRD OI-4 | **Adopt** | ตรงกับที่เราเสนอไว้ใน review v1.07 ข้อ 4 |
| A2 | Header: customer, doc_date, currency, rate, description, source, source_doc_no (§3.1) | `receipt_date`, `reference_no`, `currency_*`, `exchange_rate` — `:7979-8018` | **Adapt** | `source` ของ FRD (`Manual/Auto/Posting/AI Upload`) ไม่มีใน `tb_ar_receipt` — ถ้าจะมี ใช้ enum เดียวกับ AP (`A3` ใน AP gap) แทนการเพิ่ม string อิสระ `source_doc_no` ≈ `reference_no` (ใช้ที่มีอยู่) |
| A3 | `status` = DRAFT/IN_PROGRESS/POSTED/VOIDED/RETURNED (§3.1) | `enum_ar_receipt_status` = `draft/in_review/posted/void` — `schema.prisma:382`; `workflow_*` columns | **Adapt** | RETURNED = send back เป็น workflow stage ไม่ใช่ status (ตาม ARIV/AP B1) |
| A4 | LOA 3 ขั้น, GL ลงตอน final approve (§4 Post Receipt #4) | ARIV: GL ลงครั้งเดียวที่ `posted`; AP payment ใช้ `WorkflowOrchestratorService` | **Adopt** | จังหวะ post ตรงกับที่เราเลือกสำหรับ ARIV (OI-3) — FRD นี้ไม่มีปัญหาแบบ v1.07 |
| A5 | `ar_receipt_settle` อ้าง invoice ด้วย `invoice_no`, ไม่มี FK, copy ตัวเลขมาเก็บ (§5 #2) | `tb_ar_receipt_detail` อ้าง `ar_invoice_detail_id` (FK) + `doc_type` + `invoice_exchange_rate` — `:8064-8084` | **Reject** | ระดับบรรทัด + FK จำเป็นเพื่อให้ ARIV void guard (`ar-invoice.posting.ts:698`) และ unpaid ต่อบรรทัดทำงาน |
| A6 | Balance Due คำนวณสดจาก Σ settle ของ receipt POSTED, "ไม่เก็บค่าคงที่" (§6.1) | `unpaid_amount` / `base_unpaid_amount` ต่อบรรทัด + `outstanding_amount` ที่ header; adjust ใน tx เดียวกับ post/void — `ar-invoice.posting.ts:798, 826-848` | **Reject** | เรา lock แถวและลด/คืนยอดใน transaction (pattern เดียวกับ AP) การคำนวณสดทุกครั้งทำให้ aging หนักและไม่ atomic กับ concurrent receipt |
| A7 | Settle ได้ทั้ง ARIV และ ARCN ใน batch เดียว, Net Pay = 0 ได้, ติดลบห้าม (Rule 2, TC-ARRC-02/03) | `tb_ar_receipt_detail` — CN line เป็นยอดลบ; `net_received_amount` | **Adopt** | CHECK `net_pay >= 0` ใส่เป็น validation ใน service (`AR_RECEIPT_NET_NEGATIVE`) ไม่ใช่ DB constraint เพื่อให้ save draft ได้ |
| A8 | Base Amt = Pay Amt × invoice rate (§3.2) | `base_applied_at_invoice_rate` (`:8070`); AP `baseAtInvoiceRate` — `ap-payment.logic.ts:59` | **Adopt** | ตรงกัน |
| A9 | Select AR Invoices กรอง `invoice_date <= doc_date` (Rule 1) | AP gap A5 ยังเป็น backlog เช่นกัน | **Adopt** | ทำพร้อม AP A5 ด้วย helper เดียวกัน; ใช้ `doc_date` ของ invoice (ไม่ใช่ input date) |
| A10 | Partial settlement ถอด VAT 7/107 ตามสัดส่วนทุกครั้ง (§1.2 #3) | ARIV ตั้ง VAT เต็มตอน post | **Reject** | เหตุผลดูข้อ T1 — VAT ไม่ต้องแยกตอนรับเงิน ถ้า invoice มี TXIV แล้ว |
| A11 | Multi-bank allocation หลายแถว/ใบ + `receipt_date` ต่อแถว (`ar_receipt_method`, §3.3) | `tb_ar_receipt` มี `bank_account_id` เดียว + `receipt_method` enum + `received_date` ที่ header; AP ก็ยังขาด (AP gap **A1**) | **Adapt** | ต้องเพิ่ม child table — ตัดสินใจเรื่อง multi-bank พร้อม AP A1 ให้ได้ shape เดียวกัน; FRD ใช้ `master_payment_type` ที่เรายังไม่มี → ใช้ `tb_bank_account` (มี `chart_of_accounts_id` แล้ว `:7226`) + `receipt_method` enum แทน; cash/credit card settlement ต้องมี bank account ประเภทนั้นรองรับ |
| A12 | Other Charges: account, cost center, dim1/dim2, ติดลบได้ (Rule 3, §3.5) | header มี `total_charge_amount` แต่ไม่มี child; AP มี `tb_ap_payment_expense` — `:7624` | **Adapt** | เพิ่ม `tb_ar_receipt_charge` mirror `tb_ap_payment_expense`; dimension ใช้ pattern เดียวกับ AP A4 แทน `dim1/dim2` hardcode; ติดลบ = Cr (income offset) |
| A13 | WHT รับ: service 01/02/03/05/06 พร้อมอัตราตายตัว, ฐาน, ยอด override ได้ (§3.4) | `tb_ar_receipt_wht` — `wht_tax_profile_id`, `pnd_form`, `income_type`, `wht_rate`, `is_override`, certificate no/date — `:8088-8105` | **Adapt** | ใช้ `tb_tax_profile` (WHT) เป็นแหล่งอัตรา ไม่ hardcode; **FRD ไม่มีเลข/วันที่ 50 ทวิ** แต่เรามี — เก็บไว้ (ใช้ตอนยื่นเครดิตภาษี); รหัส 01–06 ของ FRD ไม่ตรงกับ `enum_tax_profile_wht_pnd_form` (pnd_1/2/3/53/54) ต้อง map ผ่าน BA |
| A14 | Void: เหตุผล + `void_date` บังคับ, เก็บ `ar_receipt_void_log` (Rule 6, §5 #7) | `void_at/void_by_id/void_reason/void_gl_jv_id` บน header — `:8030-8034` | **Adapt** | ไม่ต้องมี log table แยก (AP ไม่มี); เพิ่มแค่ `void_date` (ปัจจุบันมี `void_at` timestamp) ถ้า BA ต้องการวันที่ที่ผู้ใช้ระบุแยกจากเวลาที่กด |
| A15 | Lock read-only หลัง POSTED (Rule 4) | ARIV/AP: เอกสาร posted แก้ไม่ได้ | **Adopt** | — |
| A16 | Send E-Mail เฉพาะ POSTED, Dynamic print title, Jump-to, Summary/Detail toggle | ไม่มีใน backend (frontend/print) | **Defer** | งาน frontend/print; ไม่กระทบ data model ยกเว้น log การส่งเมล (ใช้ activity log) |

---

## 3. Adopt / Adapt / Reject — Auto-JV, Tax Invoice, e-Tax, WHT

### 3.1 Auto-JV matrix (§6.2)

| # | FRD | ที่เรามี | Verdict | เหตุผล / สิ่งที่ปรับ |
|---|-----|----------|---------|---------------------|
| B1 | Dr Bank ต่อบัญชี (101100X) / Dr WHT (115400X) / Dr-Cr Charges (504100X) / Cr AR / FX | `buildPaymentJvLines` (`ap-payment.logic.ts:361`) + `computePaymentTotals` (`:177`) กลับข้างได้ | **Adopt (โครง)** | สร้าง `buildReceiptJvLines` mirror AP; Dr bank = `base_net_received_amount` (`:8014`) |
| B2 | เลขบัญชี hardcode (`1130000`, `115400X`, `504100X`, `7010000/5050000`) | AP อ่านจาก `gl_setting` — `ap-payment.posting.ts:396-411`; ARIV ใช้ `ar_control_account_id` / `output_vat_account_id` — `gl-posting.service.ts:101-104` | **Reject** | ใช้ `gl_setting` + บัญชีของ customer (`ar_chart_of_accounts_id`, `schema.prisma:7662`) และ `tb_bank_account.chart_of_accounts_id` |
| B3 | สูตร FX = Σ Invoice Base − (Pay × Receipt rate) ไม่ระบุว่าบวก = gain หรือ loss (§6.2 #6) | receipt: `base@receipt − base@invoice`, **> 0 = gain (Cr)** — `schema.prisma:8018`; AP: `base@payment − base@invoice`, > 0 = loss (Dr) — `ap-payment.logic.ts:206` | **Adapt** | สูตรของ FRD เป็นเครื่องหมายกลับกับ convention ของเรา ถ้า dev ลอกสูตรมาตรงๆ gain/loss จะสลับ — ใช้ของเรา; ระวังด้วยว่า comment ที่ `tb_ar_invoice_reference` (`:7924`) ใช้ "> 0 = loss" คนละทิศกับ receipt ใน module เดียวกัน (เป็นคนละบริบทแต่ควรใส่ comment กันสับสน) |
| B4 | ไม่มีบรรทัด VAT ใน JV แม้ออก TXRC (§6.2, §4 Post #4) | ARIV ลง Cr Output VAT ตอน post (`buildArInvoiceJvLines` — `ar-invoice.logic.ts:186`) | **Reject / ต้องตัดสินใจ** | ดู T1 — ถ้า invoice มี TXIV แล้ว receipt ไม่ต้องมีบรรทัด VAT; ถ้าจะรองรับ VAT-ที่-receipt ต้องมี Output VAT Pending → Output VAT reclass ซึ่งยังไม่มีใน `GlSetting` |
| B5 | Dr AR สำหรับ CN ที่หักกลบ (#5) | CN line ใน detail เป็นยอดลบ; AP `buildApLegs` กลับ Dr/Cr ตาม doc_type | **Adopt** | — |
| B6 | JV prefix `RV`, ปรับ JV ได้เองใน Tab 3 "Fast-Entry Grid" (editable account/Dr/Cr) | `gl_setting.ar_jv_prefix_id` มี; JV สร้างผ่าน `GlSubledgerPostingService.postFromSource` สมดุลเอง | **Adapt** | แสดง preview ได้ แต่ **ห้ามแก้ Dr/Cr ได้อิสระ** — JV ต้องมาจาก source เท่านั้น (ไม่งั้น sub-ledger กับ GL ไม่ตรง); ปรับได้เฉพาะ account/cost center/dimension ของ charge |
| B7 | Post ต้อง Variance = 0 (ERR_GL_UNBALANCED) | `GL_JV_NOT_BALANCED` ใน facade | **Adopt** | — |

### 3.2 Tax Invoice / e-Tax

| # | FRD | ที่เรามี | Verdict | เหตุผล / สิ่งที่ปรับ |
|---|-----|----------|---------|---------------------|
| T1 | ออก Sales Tax Invoice (TX…) ตอน post Receipt ถ้า E-Tax เปิด **หรือมี VAT** (§4 Post #4, §1.2 #7) | ARIV: `issueTaxInvoice` ออก `TXIV` ตอน post เมื่อ `is_tax_invoice` — `ar-invoice.posting.ts:533`; Receipt: `is_tax_invoice` = "TXRC issued at receipt (services)" — `schema.prisma:7999` | **Adapt (สำคัญสูง)** | TXRC ใช้ได้เฉพาะกรณี tax point = วันรับเงิน (บริการ, invoice ที่ไม่ได้ออก TXIV) FRD ใช้ trigger "มี VAT" ซึ่งกว้างเกินไปและชนกับ TXIV; แนะนำ: allow TXRC เฉพาะ allocate กับ invoice ที่ `is_tax_invoice = false` และห้ามผสมกับ invoice ที่มี TXIV ใน receipt เดียว (error ใหม่) — **ต้องให้ที่ปรึกษาบัญชียืนยัน** |
| T2 | `ar_receipt_tax`: tax_name/tax_id/branch/address/postal + `total_tax_base/amount` + `is_claimed` + `claimed_period` (§5 #6) | `tb_ar_tax_invoice` มี FK `ar_receipt_id` (unique) อยู่แล้ว + `tax_status` (`pending/confirmed/submitted/void`) + `filing_month/year` — `schema.prisma:7938-7976` | **Adapt** | ไม่สร้าง `ar_receipt_tax` ซ้ำ; `is_claimed` ≈ `tax_status = submitted`, `claimed_period` ≈ `filing_year/month` |
| T3 | Rule 5: Void ไม่ได้ถ้าภาษีถูกยื่น ภ.พ.30 แล้ว → ต้องออก ARCN | ARIV void → ตั้ง `tax_status = void` — `ar-invoice.posting.ts:249` (ยังไม่มี guard `submitted`) | **Adopt** | เพิ่ม guard `AR_TAX_INVOICE_SUBMITTED` ทั้ง ARIV และ receipt พร้อมกัน (ปัจจุบัน ARIV ยังไม่ block) |
| T4 | Tax invoice no. แยก running จาก receipt, สร้างตอน post ไม่ใช่ตอน save draft (§3.6) | `generateArTaxInvoiceNo` + conflict error `AR_TAX_INVOICE_NO_CONFLICT` — `ar-invoice.posting.ts:540-553`; prefix `TXRC` รองรับแล้วเป็น VarChar | **Adopt** | reuse ฟังก์ชันเดิม เพิ่ม docType ฝั่ง receipt |
| T5 | `Ref. Tax Invoice` (ใบแทน/แก้ไข) ค้นหาจาก AR หรือพิมพ์เอง | `tb_ar_tax_invoice` ไม่มี ref column | **Adapt** | เพิ่ม `ref_tax_invoice_no` (string) + optional `ref_tax_invoice_id`; ใบแทนต้อง mark ใบเดิมเป็น `void` — ไม่ได้ระบุใน FRD |
| T6 | E-Tax toggle `is_etax` → ส่งเข้ากระบวนการ e-Tax Invoice & e-Receipt (§3.6) | ไม่มี; PRD §6 ระบุ integration ภายนอกยังไม่นิยาม | **Defer** | FRD ไม่ระบุ provider/format (XML, ลายเซ็นดิจิทัล, ส่ง RD) — เก็บเป็น flag + `etax_status` ได้เมื่อมี spec ผู้ให้บริการ; ยังไม่ทำใน phase แรก |
| T7 | ปุ่ม E-Mail ส่ง PDF ถึงลูกหนี้ (AES-256 ใน NFR) | `micro-notification` มีอยู่ (นอก AR) | **Defer** | ทำหลังมี print/PDF |

### 3.3 WHT และ AR Profile tax defaulting

| # | FRD | ที่เรามี | Verdict | เหตุผล / สิ่งที่ปรับ |
|---|-----|----------|---------|---------------------|
| W1 | WHT รับรู้ที่ Receipt (Dr WHT receivable) | PRD §4.4 + v1.07 OI-6; `AR_TAX2_WHT_NOT_ALLOWED` — `ar-invoice.validation.ts:276` | **Adopt** | ยืนยันแนวที่เราตัดสินแล้ว (WHT ไม่ลงที่ invoice) |
| W2 | ฐาน WHT = Tax Base ที่ผู้ใช้กรอก (§3.4) ไม่ระบุว่าตัด VAT | AP: `computeWhtRows` ฐาน = applied × net/total (ไม่รวม VAT) — `ap-payment.logic.ts:104-125` | **Adapt** | ใช้ default จาก allocation แบบ AP; FRD ปล่อยกรอกเองอิสระ เสี่ยงฐานรวม VAT |
| P1 | Tab 2 default จาก `ar_profile`: company_name / tax_id / branch_no / `tax_invoice_address` (1,2,3) / postal (§3.6, §6.4) | `tb_customer.registered_name / tax_no / branch_no` + `tb_customer_address` (`address_type`, split address) — `schema.prisma:7641, 7687`; ARIV ใช้ `arTaxInvoiceData(doc, customer, …)` snapshot ตอนออก — `ar-invoice.posting.ts:970` | **Adapt** | ชื่อ field ต่างแต่เนื้อเดียวกัน — reuse resolver ของ ARIV; "Address 1/2/3 selector" ของ FRD ไม่มีใน `tb_customer` (เรามี `address_type` ตาม `enum_customer_address_type`) → ถ้า BA ต้องการเลือกที่อยู่ใบกำกับภาษีได้ ต้องเพิ่ม flag `is_tax_invoice_address` บน `tb_customer_address` (อยู่ในขอบเขต AR Profile FRD ไม่ใช่ receipt) |
| P2 | ผู้ใช้ override ที่อยู่/ชื่อเฉพาะใบนี้ได้ ไม่กระทบ master | `tb_ar_tax_invoice` เก็บ snapshot `customer_registered_name`, `customer_tax_no`, ที่อยู่ — `:7950-7957` | **Adopt** | snapshot-on-issue อยู่แล้ว; ต้องเปิดให้แก้ได้ก่อน post (ARIV ทำแบบเดียวกัน) |
| P3 | ตรวจ checksum Tax ID 13 หลัก (§3.6) | `tax_no VarChar` ไม่มี validator | **Adopt** | เพิ่มที่ customer + tax invoice พร้อมกัน (ไม่ใช่เฉพาะ receipt) |
| P4 | `ERR_AR_TAX_DEF_MISSING` แจ้งเตือนเมื่อเปิด E-Tax แต่ไม่มี Tax ID | — | **Adopt** | warning ที่ submit; ลูกค้าต่างชาติไม่มี tax_no ได้ (`tax_no` nullable) |

---

## 4. ความเสี่ยงสำหรับเรา (ความไม่สอดคล้องภายใน FRD)

| # | จุดที่ขัดกัน | ผลต่อเรา |
|---|-------------|----------|
| R1 | §6.2 JV ไม่มี VAT แต่ §4 บอกออก TX และ §1.2 #3 ถอด VAT 7/107 | ถ้า implement ตาม §1.2 แต่ JV ตาม §6.2 → sub-ledger VAT register ไม่ตรง GL; ต้องตัดสิน T1 ก่อนเขียน posting |
| R2 | Header ว่า `source` = `Manual/Auto/Posting/AI Upload` (§3.1) แต่ ARIV ใช้ `manual/pms_folio/copy/ai` | map ให้เป็น enum เดียว อย่าสร้างชุดใหม่ |
| R3 | §3.3 `Amount In (THB)` เป็น base แต่ `ar_receipt_method.pay_amount` + `base_amount` ทั้งคู่ใน schema (§5); เอกสารไม่บอกว่ารับเป็นสกุลใด | กรณี invoice USD ลูกค้าจ่าย THB: ต้องกำหนดว่า `receipt.currency` = สกุลที่รับจริง หรือสกุล invoice (อัตรา FX คำนวณต่างกัน) |
| R4 | Rule 5 อิง `gl_output_tax.tax_status='Claimed'` (§4 Void) แต่ §7.2/§5 อิง `ar_receipt_tax.is_claimed` | สองแหล่งความจริง — เรามี `tb_ar_tax_invoice.tax_status` แหล่งเดียว ใช้อันนั้น |
| R5 | `Save Draft` อนุญาต Variance ≠ 0 (§4) แต่ CHECK `net_pay_amount >= 0` อยู่ที่ DB (§5 #1) | draft ที่ net ติดลบจะ save ไม่ได้ → ย้ายเป็น validation ที่ submit (A7) |
| R6 | Out-of-scope ตัด Bad Debt write-off และ short-payment (§1.2) แต่ Other Charges ติดลบ + `total_charge_amount` ใน schema เราระบุ "short payment written off" | ผู้ใช้จะใช้ Other Charges ตัดส่วนต่างเป็นค่าใช้จ่าย — ควรมี limit/approval หรือ account type guard |
| R7 | FRD อ้าง `master_payment_type`, `sys_currency`, `sys_activity_log`, `gl_output_tax` ซึ่งไม่ใช่ชื่อตารางจริงของเรา | อย่าสร้างตารางตามชื่อใน FRD — map ไปที่ `tb_bank_account`, currency เดิม, activity registry, `tb_ar_tax_invoice` |
| R8 | เลขที่ `ARRC{YY}{MM}{5 หลัก}` (§3.1) vs schema comment เรา `ARRC{YY}{MM}{0000}` (4 หลัก) | ตั้งค่าผ่าน running-code pattern (AP B9) ไม่ hardcode |

---

## 5. Schema delta (สิ่งที่ต้องเพิ่ม/แก้บน `schema.prisma` ที่ merged แล้ว)

| ตาราง | การเปลี่ยนแปลง | ที่มา |
|-------|----------------|-------|
| `tb_ar_receipt_method` **(ใหม่)** | `ar_receipt_id`, `bank_account_id`, `receipt_method` (enum เดิม), `payer_name`, `received_date`, `reference_no/cheque_no/cheque_date`, `amount`, `base_amount`, dimension | A11 — mirror ผลของ AP A1; ถ้าตัดสินใจ "1 receipt = 1 bank" ให้ Reject A11 แล้วไม่ต้องมีตารางนี้ |
| `tb_ar_receipt` | ถ้ามี method table: `bank_account_id`/`receipt_method`/`cheque_*` ย้ายไป child (หรือคงเป็น default) · เพิ่ม `source`/`source_doc_no` ถ้า BA ต้องการ · เพิ่ม `void_date DATE` (optional) | A2, A11, A14 |
| `tb_ar_receipt_charge` **(ใหม่)** | `ar_receipt_id`, `chart_of_accounts_id`, `cost_center_id`, `description`, `amount` (signed), `base_amount`, dimension | A12 — mirror `tb_ap_payment_expense` (`:7624`) |
| `tb_ar_receipt_detail` | ไม่เปลี่ยน (ครบกว่า FRD `ar_receipt_settle`) | A5 |
| `tb_ar_receipt_wht` | ไม่เปลี่ยน; ต้อง map รหัส 01–06 → `pnd_form` + `income_type` | A13 |
| `tb_ar_tax_invoice` | เพิ่ม `ref_tax_invoice_no` (+ optional `ref_tax_invoice_id`); เพิ่ม `is_etax` / `etax_status` เมื่อ T6 พร้อม | T5, T6 |
| `tb_customer_address` | (นอกขอบเขต receipt) flag เลือกที่อยู่ใบกำกับภาษี | P1 |
| `GlSetting` (`gl-posting.service.ts:91`) + `tb_gl_setting` | เพิ่ม `wht_receivable_account_id`, `bank_charge_account_id` (default ของ charge), `output_vat_pending_account_id` (เฉพาะถ้า T1 เลือกรองรับ VAT ที่ receipt) | B2, T1 |
| ตารางที่ **ไม่ต้องสร้าง** | `ar_receipt_settle`, `ar_receipt_tax`, `ar_receipt_void_log`, `master_payment_type` | A5, T2, A14, R7 |

Error codes ใหม่ที่คาด: `AR_RECEIPT_NET_NEGATIVE`, `AR_RECEIPT_TAX_MIXED` (TXRC กับ invoice ที่มี TXIV), `AR_TAX_INVOICE_SUBMITTED`, `AR_RECEIPT_DATE_BEFORE_INVOICE`

---

## 6. คำถามถึง BA / ทีม (9 ข้อ)

1. **TXRC trigger (T1):** ออก TXRC เฉพาะ invoice ที่ไม่ได้ออก TXIV (service tax point = วันรับเงิน) ใช่หรือไม่? กรณี invoice ที่ออก TXIV แล้ว receipt ต้องออกเอกสารภาษีอะไรอีกหรือไม่ (ปกติไม่ต้อง) — ขอที่ปรึกษาบัญชียืนยัน
2. **VAT ที่ receipt:** ถ้ามี TXRC ต้องการ JV แบบ Output VAT Pending → Output VAT หรือไม่? ปัจจุบัน ARIV ลง Output VAT ตรงๆ ตอน post ไม่มี "pending"
3. **Multi-bank (A11):** 1 receipt ต่อหลาย bank/method จริงหรือไม่ หรือแยกเป็นหลายใบ? ตัดสินพร้อม AP A1
4. **สกุลเงินรับ (R3):** invoice USD รับเป็น THB ต้องรองรับหรือไม่ และ FX คำนวณจากอัตราไหน
5. **รหัส WHT 01–06 (A13):** map ไป `pnd_form` / `income_type` อย่างไร และอัตรา 5/2/1/3/3 ใน FRD มาจากไหน (ไม่ตรงกับตารางอัตราที่ใช้ทั่วไป)
6. **Other Charges ติดลบ (R6):** ใช้เป็นส่วนต่างรับเกิน/ปัดเศษได้เท่าไร ต้องอนุมัติเพิ่มเมื่อเกินเกณฑ์หรือไม่ และยอมให้ตัด short payment ผ่านช่องทางนี้หรือไม่
7. **ที่อยู่ใบกำกับภาษี (P1):** ต้องการ selector ที่อยู่ 1/2/3 ใน AR Profile หรือใช้ `address_type` ที่มี — ประเด็นนี้ควรไปรวมกับ AR Profile FRD
8. **e-Tax (T6):** ผู้ให้บริการ/รูปแบบ (XML + ลายเซ็น, ส่ง RD) คืออะไร และอยู่ phase ไหน
9. **Void หลังยื่น ภ.พ.30 (T3):** ยืนยัน block ทั้ง ARIV และ receipt พร้อมกัน — ARIV ปัจจุบันยังไม่มี guard นี้

---

## 7. ขั้นตอนถัดไปที่แนะนำ

1. ให้ BA/ที่ปรึกษาบัญชีตอบข้อ 1–2 และ 4 ก่อน (กำหนดรูปแบบ JV และภาษี) — ข้อ 3, 5–9 ตอบทีหลังได้โดยไม่บล็อก
2. เขียน spec "AR Receipt service design" (ต่อจาก `2026-10-01-accounting-ar-invoice-service-design.md` §ถัดไป ข้อ 3) โดยยึด `tb_ar_receipt*` ที่มีอยู่เป็นฐาน และ mirror `ap-payment.{logic,posting,validation,writer}.ts`; ใส่ตาราง delta ในข้อ 5 เป็น migration เล็ก (`tb_ar_receipt_method`, `tb_ar_receipt_charge`, `GlSetting` keys)
3. แยก ticket: (a) guard `AR_TAX_INVOICE_SUBMITTED` บน ARIV void, (b) validator Tax ID checksum, (c) `GlSetting` keys — ทำได้โดยไม่ต้องรอ receipt
4. อัปเดต PRD §2.6 + OI list ด้วยข้อสรุปจาก review นี้หลัง BA ตอบ
