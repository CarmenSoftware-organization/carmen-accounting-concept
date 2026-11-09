# Review: AR Dashboard FRD v1.1 (New Concept)

**เอกสารที่ review:** [Drive › AR › AR Dashboard](https://drive.google.com/drive/folders/1OD69XW53W_eXH5K09VYh6XGmNnWwsAta?usp=drive_link) — `AR Dashboard Functional Requirement Document (FRD v1.1).md` + mockup `carmen_cloud_ar_dashboard_mockup_v1_1.html` (ไม่ได้เปิดดู mockup — ใช้ตาราง widget ใน FRD §3 เป็นหลัก)  
**วันที่ review:** 2026-11-09  
**Backend ที่เทียบ:** `carmen-turborepo-backend` main @ `1abb6a9c0` (2026-11-07) — `tb_ar_*` ใน `packages/prisma-shared-schema-tenant/prisma/schema.prisma`, `apps/micro-business/src/ar/ar-invoice/`  
**เทียบกับ:** [PRD-module-ar](../PRD-module-ar.md) §5, [PRD-module-reporting](../PRD-module-reporting.md) §4, [AR schema spec](../superpowers/specs/2026-09-30-accounting-ar-schema-design.md), [AR invoice service spec](../superpowers/specs/2026-10-01-accounting-ar-invoice-service-design.md)  
**หมายเหตุที่มาของไฟล์:** อ่านจากสำเนาในเครื่องของโฟลเดอร์ Drive (ฉบับวันที่ 2026-11-08) — อ่านอย่างเดียว ไม่แก้

**กรอบการ review:** FRD นี้เป็น concept ของคนอื่น ไม่ใช่ spec ของเรา และไม่ใช่หน้าที่เราแก้ ทีมรับ concept มาปรับ เอกสารนี้จึงจัดแต่ละองค์ประกอบเป็น **Adopt / Adapt / Reject-Defer** และบันทึกความไม่สอดคล้องใน FRD เป็น *ความเสี่ยงของเรา* เท่านั้น (ไม่ใช่ fix-list ของผู้เขียน) ต่างจาก review v1.07 ที่ส่งข้อแก้กลับ BA

---

## 1. สรุป verdict

**Adapt อย่างหนัก — รับ "หน้าตา/KPI/drill-down" ปฏิเสธ "DDL §5" ทั้งก้อน**

1. **Dashboard ไม่ควรมีตารางเป็นของตัวเอง** — เป็น read model (query/view บน `tb_ar_invoice`, `tb_ar_invoice_detail`, `tb_ar_receipt*`, `tb_ar_tax_invoice`) FRD §5 ซ้ำกับของที่มีแล้ว: `ar_customer_profile` = `tb_customer`, `ar_invoice_header` = `tb_ar_invoice`, ความต้องการ audit = activity log ที่มีอยู่ ตารางที่ยังไม่มีจริงมี 3 ตัว (folio staging, contract, cheque register) และทั้งสามเป็นของโดเมนอื่น ไม่ใช่ของ dashboard
2. **KPI ที่ทำได้วันนี้ (implemented):** Total AR, Aging 4 bucket + Total, Due status 3 แบบ, Inflow timeline 1-7/8-15/16-30 วัน — ทั้งหมดคำนวณจาก `tb_ar_invoice_detail.unpaid_amount` + `due_date` ที่ PR #704 เขียนไว้แล้ว **แต่ตัวเลข "ลดลงจากการรับเงิน" ยังไม่มี** จนกว่า Receipt จะ implement (ตอนนี้ลดได้ทางเดียวคือ CN reference) → dashboard ขึ้นได้แต่ aging จะ "ค้างทั้งก้อน" จนมี Receipt
3. **ต้องรอ Receipt/Folio/Contract:** Cheque tracker (FT-ARD-10/11), Unapplied Deposits, Unbilled Folios, Pending Contracts, Revenue Forecast (W-02), LOA queue ของ ARRC/ARDP
4. **ECL (TFRS 9): Defer** — FRD เองก็ตัดออกจาก scope (§1.2.2) ตรงกับที่เราควรทำ แนะนำเพียง "เก็บ bucket ให้พอสำหรับ provision matrix ภายหลัง"
5. **FRD ใช้ vocabulary ต่างจากของเรา** (status `SUBMITTED/APPROVED/PAID`, `is_service_tax`, ผังบัญชี `2110000/1131000/2181000/2182000`) — ต้องแปลงทุกจุดก่อนเอาไปใช้ ดู §5 ความเสี่ยง

---

## 2. ตาราง KPI / Widget — สถานะ data source

สถานะ: **implemented** = ข้อมูลอยู่ใน `tb_ar_*` และ service เขียนค่าแล้ว (PR #704) · **schema-only** = มีคอลัมน์/ตารางแต่ไม่มี service เขียน · **missing** = ไม่มีใน schema

| Widget (FRD) | สูตรใน FRD | Source ที่เรามี | สถานะ | หมายเหตุ |
|---|---|---|---|---|
| W-01 Total AR | Σ(Net+VAT−Paid) status `SUBMITTED/POSTED` | `tb_ar_invoice.outstanding_amount` where `doc_status = posted` (`schema.prisma:7760`) | implemented | status ของเราคือ `draft/in_review/posted/void` (`:361`) — AR เกิดตอน `posted` เท่านั้น ไม่นับ in_review |
| W-02 Revenue Forecast (Backlog) | Pending Contracts + Unbilled Folios | — | **missing** (contract) + **schema-only** (folio: `tb_ar_invoice_detail.folio_*`, `tb_customer.pms_account_code`) | ไม่มี `tb_ar_contract`, ไม่มี staging ของ folio; PMS interface ยังไม่มี |
| W-03 Expected Cash Inflow 30D | Σ unpaid where due ≤ today+30 | `tb_ar_invoice_detail.unpaid_amount` + `tb_ar_invoice.due_date` (idx `arinvoice_due_date_idx`) | implemented | รวม overdue ด้วยหรือไม่ FRD ไม่ชัด (ดู OQ-4) |
| W-04 Output VAT – Undue | VAT ของบิลบริการที่ยังไม่มี ARRC | `tb_ar_tax_invoice` (status `pending/confirmed/submitted/void`) | **schema-only** | ไม่มีแนวคิด due/undue; โมเดลเราออกใบกำกับภาษีตอน submit ถ้า `is_tax_invoice` ดูข้อ 5.3 |
| W-05–07 Inflow Timeline | Σ unpaid ตามช่วง due | เหมือน W-03 | implemented | บิล `due_date` อยู่ที่ header; unpaid อยู่ที่ detail → ต้อง join/group |
| W-08 Total AR Header | = W-01 | เหมือน W-01 | implemented | ซ้ำกับ W-01 และ W-13 (3 ค่าเดียวกัน) |
| W-09–12 Aging 1-30/31-60/61-90/>90 | Aging days ช่วง | `due_date` vs report date | implemented | PRD-AR §5.1 กำหนด 6 bucket (Current,1-30,31-60,61-90,91-120,120+) — FRD ใช้ 4 + รวม |
| W-13 Total Aging | Σ W-09..12 | derived | implemented | FRD ไม่รวม "Not yet due" ใน Total Aging → ไม่เท่า W-01 (ดู ข้อ 5.2) |
| W-14–16 Due status | Not yet / This month / Overdue | `due_date`, `unpaid_amount > 0` | implemented | "Due this month" ทับซ้อนกับ Overdue (นิยาม `≤ month end`) |
| M-01 Unbilled PMS Folios | `ar_folio_staging` | — | **missing** | รอ PMS interface (OI-7 ของ review v1.07) |
| M-02 Pending AR Contracts | `ar_contract_header` | — | **missing** | ไม่มีใน PRD-AR/schema spec เลย — เป็น feature ใหม่ทั้งดุ้น |
| M-03 Unapplied Advance Deposits | ARDP ที่ยังไม่ตัด | `tb_ar_invoice` `doc_type = deposit` + `outstanding_amount` | **schema-only** | enum `deposit` มี (`:353`) แต่ service ยังไม่รองรับ ARDP (รอ OI-1/2) |
| M-04 LOA Approval Queue | นับเอกสาร `in_review` แยกประเภท | `tb_ar_invoice.workflow_*`, `doc_type`, `doc_status = in_review` | implemented (ARIV/ARDN/ARCN) · schema-only (ARRC/ARDP) | ใช้ workflow engine เดิม ไม่ต้องสร้างคิวใหม่ |
| M-05 AI Notification | Smart collection alert ฯลฯ | — | **missing** | ไม่มี rule engine/notification ของ AR |
| FT-ARD-10 Cheque tracker | `ar_cheque_register` 4 สถานะ | `tb_ar_receipt.cheque_no/cheque_date/receipt_method=cheque` (`:7979+`) | **schema-only** (แค่ฟิลด์บน receipt; ไม่มี lifecycle on_hand→in_transit→cleared→bounced) | ดู Adapt A6 |
| FT-ARD-11 Bounced → Void Receipt | reopen unpaid | `tb_ar_receipt.doc_status = void` | schema-only | ต้องมี Receipt service + void ที่คืน `unpaid_amount` ก่อน |
| FT-ARD-12 Output VAT pipeline / export ภ.พ.30 | `ar_output_tax_recon` | `tb_ar_tax_invoice` มี `filing_month/year`, `base_amount`, `vat_amount` (`:7938`) | schema-only | ครอบคลุมเกือบหมดแล้ว ไม่ต้องมีตาราง recon แยก |
| FT-ARD-01 Consolidated BU switcher | aggregate ข้าม BU | — | **missing** | tenant schema แยกต่อ BU → ข้าม BU ต้องทำที่ชั้น gateway/report (ดู Adapt A7) |
| (ไม่มีใน FRD) DSO | — | PRD-Reporting §4.2: `AR ÷ (Revenue ÷ 365)` | implemented (ต้อง Revenue จาก GL) | FRD ไม่มี DSO แม้ PRD reporting ของเราระบุไว้ — เพิ่มเอง |

---

## 3. Dashboard ควรมีตารางของตัวเองไหม — **ไม่**

| ตารางใน FRD §5 | ของเราที่ตรงกัน | ตัดสิน |
|---|---|---|
| `ar_customer_profile` | `tb_customer` (`schema.prisma:7641`) มี credit_term, credit_limit, tax_no, branch_no, pms_account_code ครบ ขาดแค่ `customer_type` (ใช้ `customer_group` JSON แทน) | Reject — ซ้ำ |
| `ar_invoice_header` (+ `unpaid_amount_base`, `is_service_tax`, `accounting_period`) | `tb_ar_invoice` — มี `outstanding_amount`, `due_date`, `base_*`; unpaid ราย line | Reject — ซ้ำ; `accounting_period` derive จาก `tb_gl_period` ได้ |
| `ar_folio_staging` | ไม่มี | Defer — เป็นของ PMS interface spec ไม่ใช่ dashboard; ตัดสินใจตอนทำ folio (ดู OQ-6) |
| `ar_contract_header` | ไม่มี | Defer — recurring billing เป็น feature ใหม่ ต้องมี PRD/spec ของตัวเอง |
| `ar_cheque_register` | ฟิลด์ cheque บน `tb_ar_receipt` | Defer/Adapt — ตัดสินตอนทำ Receipt (ดู A6) |
| `ar_output_tax_recon` | `tb_ar_tax_invoice` (`filing_*`, `tax_status`) | Reject — ซ้ำ |
| `carmen_audit_logs` | activity log กลางของ micro-business (`common/activity`) | Reject — ซ้ำ |

**แนวทางที่แนะนำ:** (ก) endpoint query บน `tb_ar_*` ใน `micro-business/src/ar/ar-dashboard/` (ไม่มีตารางใหม่); (ข) ถ้า performance ไม่พอ ค่อยเพิ่ม DB view / materialized view — *ไม่ใช่ตาราง snapshot ที่ต้อง sync* เพราะ `outstanding_amount` เป็น denormalized ที่ service รักษาอยู่แล้ว ข้อเสนอ index เดียวที่น่าเพิ่ม: composite `(doc_status, due_date)` บน `tb_ar_invoice` (ตอนนี้มี `customer_id,doc_status` และ `due_date` แยก — `:7794-7798`)

หมายเหตุ: AP ฝั่งที่ implement แล้ว **ไม่มี** aging/dashboard endpoint ให้ reuse (grep `aging` ใน `apps/micro-business/src/ap` และ `ar` ไม่พบ) และ `micro-report` (Go) เป็นตัว render รายงาน/template (`service/report/template_map.go`) ไม่ใช่ KPI API → aging query ต้องสร้างใหม่ ควรออกแบบให้ AP ใช้ pattern เดียวกันได้ (PRD-Reporting §4.3 ระบุ AP Aging + AR Aging คู่กัน)

---

## 4. การคำนวณหลัก

**Aging / Due status** — คิดจาก `due_date` (ตรงกับ PRD-AR §5.2 และ FRD) โดยใช้ `tb_ar_invoice_detail.unpaid_amount` > 0 และ header `doc_status = posted`; report date เป็น parameter (ไม่ใช่ `now()` ตายตัว) เพื่อให้ย้อนดูวันสิ้นเดือนได้ ซึ่ง FRD ไม่ได้ระบุ แต่ aging ที่ใช้ตอนปิดงวดต้องมี as-of date

**ข้อควรระวัง unpaid ใน base currency:** ใช้ `base_unpaid_amount` (`:7874`) เท่านั้นเมื่อรวมข้ามสกุล และสังเกตว่า FRD W-08 บอก "Base Currency THB" — สกุลฐานของ BU ใน schema เรามี `base_currency_code` ไม่ hardcode THB

**DSO** — FRD ไม่มี; PRD-Reporting §4.2 ใช้สูตรหยาบ `AR ÷ (Revenue ÷ 365)` แนะนำ Adapt เป็นหน้าต่างเลื่อน 90 วัน (Revenue 90 วันจาก GL) เพื่อไม่ให้ตัวเลขแกว่งตามฤดูกาลโรงแรม และต้องนิยามว่า Revenue นับแค่ revenue ที่ขายเชื่อ (AR-originated) หรือทั้งหมด

**Cash forecast** — FRD ใช้ due_date ล้วน (สมมติว่าลูกหนี้จ่ายตรงเวลา) ทำได้ทันที แต่ overstates ในทางปฏิบัติ; ทางเลือก Defer: ปรับด้วย historical days-to-pay ต่อลูกค้า (ต้องมี Receipt ≥ 6-12 เดือนก่อน) → เริ่มด้วยสูตร FRD แล้วติดป้าย "ตามกำหนดชำระ ไม่ใช่คาดการณ์พฤติกรรม"

**TFRS 9 ECL** — FRD ตัดออกจาก scope เอง (§1.2.2 "ECL Calculation Engine… ในส่วน Year-End Audit Tool") แต่ยังอ้าง TFRS 9 ในหัวเอกสาร (หน้า compliance) สิ่งที่ dashboard ควรทำคือแค่ให้ bucket ที่ drill-down ได้ (provision matrix ใช้ bucket + อัตราต่อ bucket) — **Defer** engine, ไม่ต้องมีตาราง/ฟิลด์ ECL ใน dashboard; ต้องใช้บัญชี allowance (contra-AR) และ JV ซึ่งเป็นงานของ GL/period-end ไม่ใช่งาน dashboard

---

## 5. Adopt / Adapt / Reject

### 5.1 Adopt (รับตามที่เป็น)

| # | องค์ประกอบ | เหตุผล | หลักฐาน |
|---|---|---|---|
| D1 | Drill-down ทุกการ์ดไป Invoice List ด้วย filter key (`AGING_*`, `DUE_*`) | ตรงกับ pattern list+filter ที่ frontend มี; ไม่ต้องสร้าง backend ใหม่นอกจาก filter param | FRD §3.1 W-09..W-16; §4 แถวแรก |
| D2 | Inflow timeline 3 ช่วง (1-7/8-15/16-30) | ทำได้จาก `due_date` ล้วน | FRD §3.1 W-05..07; `schema.prisma:7794` |
| D3 | Folio invoice = ไม่ post GL ซ้ำ, ใช้ติดตาม aging | ตรงกับ PRD-AR §3.1 และ `is_pms_folio` / `gl_jv_id` null | FRD §6.2; PRD-AR:126-135; `schema.prisma:7730,7763` |
| D4 | ปฏิเสธการ Approve ตรงจาก dashboard โดยไม่ผ่านคิว (แค่ลิงก์ไปคิว) | ใช้ workflow เดิม | FRD M-04 (เฉพาะส่วน "Review Queue") |
| D5 | Credit-limit เป็น warning ไม่ block draft | ตรงกับ credit_limit ใน `tb_customer` และสเปคที่ตั้งว่า credit limit check เป็นงานถัดไป | FRD ERR_AR_03; `schema.prisma:7661`; invoice spec :39 |
| D6 | ปิดงวดแล้ว bounced cheque → JV ปรับในงวดปัจจุบัน ไม่แก้งวดเก่า | หลักบัญชีปกติ + `tb_gl_period` | FRD ERR_AR_01; `schema.prisma:3274` |

### 5.2 Adapt

| # | องค์ประกอบ | ปรับอะไร | เพราะอะไร |
|---|---|---|---|
| A1 | Status filter ของ W-01 (`SUBMITTED/POSTED`) | ใช้ `doc_status = posted` เท่านั้น | AR ลงบัญชีตอน posted; `in_review` ยังไม่ใช่หนี้ (`schema.prisma:361`; invoice spec :111) |
| A2 | Aging 4 bucket + Total (ไม่รวม Not-yet-due) | ใช้ 6 bucket ตาม PRD-AR §5.1 (Current, 1-30, 31-60, 61-90, 91-120, 120+) และให้ Total = W-01 | ให้ Σ bucket ตรง Total AR และตรงรายงาน AR Aging; 120+ คือ write-off candidate ที่ PRD กำหนด |
| A3 | Inflow timeline "คาดการณ์" | ติดป้าย "ตามกำหนดชำระ"; แยกบรรทัด Overdue เป็นของตัวเอง | ไม่งั้น W-03 (≤ today+30) รวม overdue แบบไม่รู้ตัว |
| A4 | M-04 LOA queue | นับจาก `doc_status = in_review` + `workflow_current_stage` ของผู้ใช้; **ตัด "Approve All Eligible"** ออกจาก v1 | Batch approve ข้าม doc type ข้าม LOA stage เสี่ยง SoD; workflow เดิมอนุมัติทีละใบ |
| A5 | W-04/FT-ARD-12 Output VAT Due/Undue | ใช้ `tb_ar_tax_invoice` กรอง `tax_status` / `filing_*`; "Undue" ต้อง derive จาก invoice ที่ `is_tax_invoice = false` + บริการ ที่ยังไม่มี receipt | ของเราออกใบกำกับภาษีตอน submit ถ้า `is_tax_invoice` (`schema.prisma:7735`); ไม่มี `is_service_tax`; ต้องให้ที่ปรึกษาภาษียืนยันจุดความรับผิด (OQ-2) — จนกว่าจะยืนยัน แสดงเป็น *Output VAT ตาม filing period* ก่อน |
| A6 | Cheque tracker | ย้ายสถานะ cheque ไปเป็นส่วนของ Receipt design (ฟิลด์ `cheque_*` + `cheque_status` + วันที่ clear) แทนตาราง register แยก | `tb_ar_receipt` มี `cheque_no/date` แล้ว (`:7979+`); register แยกซ้ำ receipt และทำให้ Void ต้อง sync 2 ที่; JV "Dr Bank / Cr Cheque on Hand" ต้องตัดสินที่ Receipt spec |
| A7 | Consolidated BU switcher | ทำเป็น fan-out query ที่ชั้น gateway/report ต่อ BU ที่ user มีสิทธิ์ แล้วรวมผล (read-only) | tenant schema แยกต่อ BU — "Aggregation ข้ามตาราง" ตามที่ FRD ว่าไม่ตรงกับสถาปัตยกรรม; ต้องตัดสินเรื่องสกุลเงินรวม (BU ต่างสกุลฐานไหม) |
| A8 | Hero KPI เพิ่ม | เพิ่ม DSO, % Overdue/Total (PRD-Reporting target `AR>90 < 10%`) | PRD-Reporting §4.2 ระบุเป้า KPI ไว้แล้วแต่ FRD ไม่มี |
| A9 | M-01 "Create Invoice from Folio" | เก็บ UX ไว้ แต่เรียก service ARIV ที่มี (`is_pms_folio = true`, `doc_source = pms_folio`) ไม่ insert ตรง | invoice service เป็นตัวรักษา unpaid/outstanding และ numbering (running code `AR-IV`); ต้องรอ PMS interface |

### 5.3 Reject / Defer

| # | องค์ประกอบ | ตัดสิน | เหตุผล |
|---|---|---|---|
| R1 | DDL §5 ทั้ง 7 ตาราง (MySQL `AUTO_INCREMENT`, `VARCHAR`) | **Reject** | เราใช้ PostgreSQL + UUID + Prisma tenant schema; ตารางซ้ำกับของที่ implement แล้ว (ดู §3) |
| R2 | Recurring A/R Contracts + "Apply Contract (All)" (FT-ARD-07, M-02) | **Defer** | ฟีเจอร์ใหม่ที่ไม่มีใน PRD-AR; ต้องมี spec (invoice template, proration, ปิดสัญญา) ก่อน; dashboard เป็นแค่ปุ่ม |
| R3 | Cheque register + Batch Clear Cheques + Bounced reversal (FT-ARD-10/11) | **Defer** | ต้องมี Receipt (ARRC) service ก่อน (`tb_ar_receipt` schema-only); ตัดสินใจ lifecycle ที่ Receipt spec |
| R4 | ECL / TFRS 9 engine | **Defer** | FRD เองก็ตัดออก; งาน period-end ไม่ใช่ dashboard |
| R5 | M-05 AI Accounting Notification (Smart Collection Alert) | **Defer** | ไม่มีข้อมูลนิยาม rule/โมเดล; ไม่มี notification infra ฝั่ง AR; ทำภายหลัง KPI ปกติเสถียร |
| R6 | ผังบัญชี hardcode `2110000/1131000/2181000/2182000` | **Reject** | ของเราอ่านจาก `gl_setting` (`ar_control_account_id`, `output_vat_account_id` — schema spec :45) |
| R7 | RBAC role ชื่อ AR Clerk / Credit Manager / FC | **Reject** (ใช้ permission catalog ของเรา) | สิทธิ์ใน micro-business ผูกกับ role_permission; "สั่ง Void Receipt" ฯลฯ ต้องแมปเป็น permission จริง |
| R8 | NFR "≤1.0s ที่ 50,000 ใบ" | **Defer** | ยังไม่มีข้อมูลขนาดจริงต่อ BU; วัดหลังมี aging query + index แล้ว ค่อยตัดสินว่าต้อง materialize |

---

## 6. Schema delta (ข้อเสนอ — ไม่ใช่ migration)

ไม่มีตารางใหม่สำหรับ dashboard v1 สิ่งที่น่าเพิ่ม:

| รายการ | ที่ | จำเป็นเมื่อ |
|---|---|---|
| Index `(doc_status, due_date)` บน `tb_ar_invoice` | `schema.prisma` ~:7794 | aging query ช้า (วัดก่อน) |
| `cheque_status` + `cleared_at`/`bounced_at` บน `tb_ar_receipt` | Receipt spec | ทำ cheque tracker (A6) |
| `tb_ar_contract` (+detail) | spec ใหม่ | ทำ recurring billing (R2) |
| `ar_folio_staging` หรือเทียบเท่า | PMS interface spec | ทำ Unbilled Folios (M-01, W-02) |
| `customer_type` (หรือยืนยันใช้ `customer_group` JSON) | `tb_customer` | ถ้าต้อง group by OTA/Corporate ใน dashboard |

ไม่ต้องแตะ `prisma/schema.prisma` ของ concept repo จนกว่าจะตัดสินข้อข้างต้น

---

## 7. ลำดับ dependency (KPI ที่ติดอะไร)

| ขั้น | ต้องมีก่อน | ปลด widget |
|---|---|---|
| 0 (ได้เลย) | ARIV/ARDN/ARCN posted (PR #704 ✓) | W-01, W-03, W-05–07, W-08–16, aging ทั้งหมด, LOA queue (ARIV/DN/CN), DSO |
| 1 | **Receipt (ARRC) service** + TXRC | aging "ลดลงจริง", cheque tracker, bounced reversal, Output VAT ที่เกิดตอนรับเงิน (A5), Inflow ที่แม่นขึ้น |
| 2 | OI-1/OI-2 (JV มัดจำ / TFRIC 22) + ARDP service | M-03 Unapplied Deposits, LOA ของ ARDP |
| 3 | PMS interface + folio staging | M-01 Unbilled Folios, W-02 (ส่วน folio) |
| 4 | Contract spec | M-02, W-02 (ส่วน contract) |
| 5 | Consolidated query layer | FT-ARD-01 |

ข้อเสนอ: ส่งมอบ dashboard เป็น 2 ระยะ — ระยะ 0 ขึ้นก่อนพร้อม banner "ยังไม่รวมยอดรับชำระ" หรือรอถึงขั้น 1 เพื่อไม่ให้ผู้ใช้เห็น aging ที่ไม่เคยลด (ตัดสินที่ OQ-1)

---

## 8. ความเสี่ยงที่เห็นใน FRD (สำหรับเรา ไม่ใช่ข้อแก้ถึงผู้เขียน)

| # | ความเสี่ยง | หลักฐาน | ผลต่อเรา |
|---|---|---|---|
| K1 | W-13 "Total Aging" ไม่รวมบิลที่ยังไม่ถึงกำหนด จึง ≠ W-01 แต่ W-08 (Total AR) บอกว่า = W-01 | FRD §3.1 W-08, W-13 | ถ้า copy ตามจะมี 2 ตัวเลข "ยอดรวม" ไม่ตรงกัน → ใช้ A2 |
| K2 | `Due this month` (≤ month end & unpaid) นับรวม overdue | W-15 vs W-16 | ผลรวม W-14+15+16 ซ้ำซ้อน → นิยาม bucket ให้ไม่ทับซ้อนตอนทำ |
| K3 | Void receipt ตั้ง `doc_status='POSTED'` กลับให้ invoice | FRD §7.4 sequence | ของเราไม่มี `PAID` status — สถานะ posted + unpaid>0 พอ; reversal ต้องคืน `unpaid_amount` ราย line ไม่ใช่ header เดียว |
| K4 | บัญชีไม่ตรงกับ review v1.07 และ gl_setting (`1130000` vs `2110000/1131000`) และใช้รหัส `2110000` เป็น AR (ปกติรหัส 2xxx = หนี้สิน) | FRD §6.2 | อย่า copy ผัง; ใช้ gl_setting |
| K5 | Void bounced cheque ใช้ "Cheque on Hand" (Dr Bank / Cr Cheque on Hand) แต่ Receipt design ของเราลง Dr Bank ตรง | FRD §7.4 | ต้องตัดสินที่ Receipt spec ว่าจะมีบัญชีพักเช็คไหม |
| K6 | FRD ระบุ "Approved for Development" ลงนาม 8 พ.ย. 2026 แต่พึ่งตาราง/ฟีเจอร์ที่ไม่มีใน PRD ใด (contract, cheque register) | FRD หัวเอกสาร, §11 | อย่าถือเป็น scope commitment — ยังเป็น concept |
| K7 | FRD ยังเรียก ARRC เป็นเอกสารตัวหนึ่งใน LOA queue ร่วมกับ ARIV | M-04 | review v1.07 ข้อ 4 แนะนำแยก Receipt; ต้องสอดคล้องกัน |

---

## 9. Open questions

| # | คำถาม | ถึงใคร | ติดอะไร |
|---|---|---|---|
| OQ-1 | ปล่อย dashboard ระยะ 0 ก่อน Receipt (aging ไม่ลดจากการรับเงิน) หรือรอ Receipt? | Product/ทีม dev | ลำดับ roadmap |
| OQ-2 | Output VAT Undue/Due — จุดความรับผิด VAT ของรายได้บริการโรงแรมที่ออก Invoice ก่อนรับเงิน; ออกใบกำกับภาษีตอน Invoice (เรา: ตอน submit ถ้า `is_tax_invoice`) ขัดกับ FRD ERR_AR_05 หรือไม่ | ที่ปรึกษาบัญชี/ภาษี | A5, W-04 |
| OQ-3 | Aging bucket: ใช้ 6 bucket ตาม PRD-AR หรือ 4+Total ตาม FRD? | BA/Credit control | A2 |
| OQ-4 | W-03 "Expected Cash Inflow" รวม overdue ไหม? | BA | A3 |
| OQ-5 | Consolidated view: BU ต่างสกุลฐานไหม และ user ที่มีสิทธิ์หลาย BU คือใคร (ระดับ group หรือ per-BU role) | Platform/Backstage | A7 |
| OQ-6 | Folio staging เป็นตารางของเรา หรือดึง PMS สดผ่าน interface? | PMS integration owner | M-01, W-02 |
| OQ-7 | Cheque/PDC ต้อง track lifecycle 4 สถานะจริงหรือไม่ (โรงแรมไทยใช้เช็คลูกหนี้มากน้อยแค่ไหน) | BA/ลูกค้า | A6, R3 |
| OQ-8 | Recurring contract billing อยู่ใน roadmap AR หรือไม่ (ถ้าใช่ ใครเขียน PRD) | Product | R2 |
| OQ-9 | DSO สูตรไหน (หน้าต่าง 90 วัน vs 365; Revenue AR-only vs ทั้งหมด) | Finance | A8 |
| OQ-10 | ECL จะทำในโมดูลไหน/เมื่อไร (Year-End tool ตาม FRD หรือ GL period-end) | Finance + Audit | R4 |

รวม **10** คำถาม

---

## 10. ขั้นตอนถัดไปที่แนะนำ

1. ตัดสิน OQ-1/OQ-3 กับ BA ในประชุมเดียว (กำหนด bucket + ระยะส่งมอบ) แล้วเพิ่มหัวข้อ "AR Dashboard" ใน PRD-AR (หรือ PRD-Reporting §4.3 ถ้าเห็นว่าเป็นงาน reporting) ที่อ้าง FRD นี้เป็น concept
2. เขียน spec สั้น `accounting-ar-dashboard-queries` ครอบ **ระยะ 0 เท่านั้น**: query aging/due/inflow/LOA count, as-of date parameter, ใช้ `base_unpaid_amount`, ไม่มีตารางใหม่ — ทำคู่กับ AP aging ให้ใช้ pattern เดียวกัน
3. ลำดับ spec ถัดไปยังเป็นตามที่ invoice spec :315 วางไว้: **Receipt ก่อน** (รวม cheque lifecycle, A6) → ARDP → PMS folio ตัวเลข dashboard ขั้น 1-3 ผูกกับสามตัวนี้
4. ไม่ต้องรอ FRD ฉบับถัดไป: contract (R2), AI notification (R5), ECL (R4) เก็บเป็น backlog "Defer" พร้อมลิงก์ถึงเอกสารนี้
