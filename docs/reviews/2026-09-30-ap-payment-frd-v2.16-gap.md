# Gap: AP Payment FRD v2.16 เทียบกับ implementation ใน micro-business

**เอกสารที่ review:** `Accounting-docs/AP/Payment/functional_requirement_document_carmen_cloud_erp_ap_payment_module_v2_16.md` (หัวเอกสารเขียน FRD v1.0, module v2.16, Status "Approved for Implementation")
**วันที่ review:** 2026-09-30
**ผู้รับ:** ทีม dev (กลุ่ม A) และ BA (กลุ่ม B, C)
**baseline ที่ใช้เทียบ:**

- spec ที่ตกลงแล้ว: [../superpowers/specs/2026-09-23-accounting-foundation-ap-design.md](../superpowers/specs/2026-09-23-accounting-foundation-ap-design.md) §4.3, §7, §8, §14
- โค้ด `carmen-turborepo-backend-v2` main (`b80024588`): `apps/micro-business/src/ap/ap-payment/*`, schema `packages/prisma-shared-schema-tenant/prisma/schema.prisma`, gateway `apps/backend-gateway/src/application/ap-payment/ap-payment.controller.ts`

**หลักการตัดสิน:** เมื่อ FRD กับโค้ดขัดกัน ยึด spec/โค้ดที่ตกลงแล้วเป็น baseline (spec §14) เว้นแต่ BA ตัดสินให้เปลี่ยน

แบ่ง 3 กลุ่ม:

- **A. FRD กำหนดแต่โค้ดยังไม่มี** → เข้า backlog (ระบุ sub-project และขนาด)
- **B. โค้ดเบี่ยงจาก FRD โดยตั้งใจ** → BA ปรับ FRD ให้ตรง
- **C. FRD ผิดหรือขัดกันเอง** → BA ตัดสิน

| กลุ่ม | จำนวน | สำคัญสูง |
|---|---:|---|
| A | 10 | A1, A2, A8 |
| B | 12 | B4, B6, B8 |
| C | 10 | C1, C2, C3 |

---

## 0. สิ่งที่ FRD กำหนดและโค้ดมีแล้ว (ไม่ต้องทำอะไร)

| FRD | โค้ด |
|---|---|
| ตัดจ่ายหลายใบ + partial (§1.2 #3, TC-03) | allocation ระดับบรรทัด `applied_amount ≤ available` — `ap-payment.validation.ts:114-121` |
| ซ่อน invoice ที่ balance ≤ 0 จาก modal (§6.1) | `outstandingDocuments` กรอง `outstanding_amount ≠ 0`, บรรทัด `unpaid_amount > 0` — `ap-payment.service.ts:312-356` |
| Base Amt = Pay Amt × invoice rate (§3.2) | `baseAtInvoiceRate` — `ap-payment.logic.ts:59-65` |
| Base Disbursed = Pay Amt × header rate (§3.3) | บรรทัด Bank ใช้ `base_net_paid_amount` = Σ applied × payment rate — `ap-payment.logic.ts:185-188, 375-385` |
| Realized FX Dr/Cr อัตโนมัติ (§6.2 #5) | `realized_fx_amount = base@payment − base@invoice`; > 0 = Dr loss, < 0 = Cr gain — `ap-payment.logic.ts:206, 414-428` |
| WHT คำนวณจาก tax base ไม่รวม VAT, แก้ยอดได้ (§3.4) | `computeWhtRows` base = applied × net/total; `is_override` — `ap-payment.logic.ts:104-125`, endpoint `POST wht-preview` |
| Other expense พร้อมบัญชี + cost center (§3.5) | `tb_ap_payment_expense` — `schema.prisma:7409-7424`; Dr ต่อบรรทัด — `ap-payment.logic.ts:400-413` |
| JV auto-populate Dr AP / Cr Bank / Cr WHT / Dr Expense / FX (§6.2) | `buildPaymentJvLines` — `ap-payment.logic.ts:361-430` |
| Double-entry balance (§8 ERR_GL_OUT_OF_BALANCE) | บังคับใน `GlSubledgerPostingService.postFromSource` (`GL_JV_NOT_BALANCED`) |
| LOA 3 ขั้น approve / send back / reject (§1.2 #2) | `WorkflowOrchestratorService` type `ap_payment`; endpoints submit/approve/review/reject — gateway controller `:400-593` |
| Period ปิดห้าม post (§3.1) | `isPeriodOpen(payment_date)` ตอน submit และ post — `ap-payment.service.ts:400-401`, `ap-payment.posting.ts:220-221` |
| Void คืนยอด aging (§4 Void #3, §6.1) | `adjustUnpaid('restore')` + recompute `outstanding_amount` ใน tx เดียวกับ reversal — `ap-payment.posting.ts:288-294, 462-483` |
| Exchange rate default ตามวันที่ (§3.1) | `resolveRate` → `findByDateAndCurrency(payment_date)` — `ap-payment.service.ts:1001-1030` |
| Attachments, Activity log (§1.2 #8, §9 #3) | `tb_ap_payment.attachments`; activity registry `ap-payment.*` — `activity-registry.ts:540-550` |
| เลขที่ `APPV{YY}{MM}{5 หลัก}` reset ทุกเดือน (§3.1) | running code type `AP-PV` ตาม pattern config + advisory lock — `ap-payment.running-code.ts:15, 59-79` (ตั้งค่า pattern ให้ตรงได้ ดู B9) |
| Optimistic lock / audit (§9 #3) | `doc_version` guard — `ap-payment.service.ts:581-623` |

---

## A. FRD กำหนดแต่โค้ดยังไม่มี (backlog)

### A1. Multi-Bank Allocation — Payment Methods หลายแถวต่อใบ (§1.2 #4, §2 3.2, §3.3, §5 `ap_payment_method`) — **สำคัญสูง**

**โค้ดตอนนี้:** 1 ใบ = 1 บัญชีธนาคาร: `bank_account_id`, `payment_method` enum (`bank_transfer | cheque | cash | credit_card | promptpay`), `cheque_no`, `cheque_date` บน header — `schema.prisma:7300-7305`; บรรทัด Cr Bank มีบรรทัดเดียว — `ap-payment.logic.ts:375-385`

**FRD ต้องการ:** ตาราง `ap_payment_method` หลายแถว แต่ละแถวมี payment type (จาก Payment Type master), payee, ref/cheque no., pay amount, base amount, account/CC/dimension

**ขนาด:** M — ตารางลูกใหม่, JV บรรทัด Cr Bank ต่อแถว, guardrail Σ method = net_paid (ดู C2), ย้าย `payee_name`/`cheque_no` ลงแถว

**เสนอ:** ทำใน sub-project **Cash & Bank** (spec §15 #3) เพราะต้องออกแบบร่วมกับ cheque register และ bank reconciliation; ระหว่างนี้ FRD ควรระบุว่า phase แรกรองรับ 1 บัญชีต่อใบ

### A2. WHT Certificate (50 ทวิ) — เลขที่, form, ที่อยู่สรรพากร, condition, จำกัด 3 บริการ (§1.2 #5, §2 3.3, §3.4, §5 `ap_payment_wht` / `ap_payment_wht_detail`, TC-06, TC-10) — **สำคัญสูง**

**โค้ดตอนนี้:** `tb_ap_payment_wht` 1 แถวต่อ tax profile: `pnd_form` snapshot จาก profile, `income_type`, `base_amount`, `wht_rate`, `wht_amount`, `is_override`, `chart_of_accounts_id` — `schema.prisma:7388-7407`; ผู้รับเงิน snapshot บน header 4 ช่อง (`payee_name`, `payee_tax_no`, `payee_branch_no`, `payee_address` เป็น string เดียว) — `schema.prisma:7295-7298`, `ap-payment.writer.ts:253-256`

**ยังไม่มี:** `wht_no` running `WHT{YY}{MM}{5}`, การเลือก form ต่อใบรับรอง, ที่อยู่แยก 8 ช่อง, `wht_condition` (1/2/3), การจำกัด 3 แถวต่อใบรับรอง, การพิมพ์ 50 ทวิ

**ขนาด:** L

**เสนอ:** อยู่ใน sub-project **Tax** ตาม spec §15 #3 (WHT certificate, ภ.ง.ด.) ซึ่งเป็นลำดับถัดไปอยู่แล้ว ข้อสังเกตสำหรับ BA: การจำกัด 3 บริการเป็นข้อจำกัดของ**แบบฟอร์ม** 50 ทวิ ไม่ควรจำกัดที่ PV (PV หนึ่งใบอาจต้องออกใบรับรองมากกว่า 1 ฉบับ) — ควรจำกัดตอนสร้างใบรับรอง

### A3. Header `source` (Manual / Auto / Posting / AI Upload) และ `source_doc_no` (§1.2 #1, §3.1)

**โค้ดตอนนี้:** มีเฉพาะ `reference_no VarChar(100)` — `schema.prisma:7291`; ไม่มี enum source

**ขนาด:** S แต่ต้องการนิยามก่อน (ดู C9: `Auto` และ `Posting` หมายถึงอะไร)

### A4. Dimension บนบรรทัด Other Expense และ Payment Method (§3.3, §3.5 `dim1`, `dim2`)

**โค้ดตอนนี้:** `tb_ap_payment_expense` มี `cost_center_id` แต่ไม่มี dimension — `schema.prisma:7409-7424`; header มี `dimension Json` — `schema.prisma:7346`; dimension ของบรรทัด AP มาจาก invoice line (spec §8)

**ขนาด:** S — เพิ่ม dimension json บนบรรทัด expense และ copy ลง `tb_gl_jv_detail_dimension`

### A5. กรอง invoice ใน Select Document ด้วย `invoice input_date ≤ payment input_date` (§3.2, §4 Select Document, §8 ERR_INV_DATE_AHEAD, TC-02)

**โค้ดตอนนี้:** `outstandingDocuments(vendorId)` ไม่กรองวันที่ — `ap-payment.service.ts:312-328`; ตอน submit/post ก็ไม่ตรวจ

**ขนาด:** XS — เพิ่ม query param `as_of_date` และตรวจซ้ำตอน post ด้วย `tb_ap_invoice.doc_date` (ดู C1 ว่าจะเทียบกับ `payment_date` หรือ `paid_date`)

### A6. Guard `Paid Date ≥ Input Date` (§3.1, §4 Submit #1, §8 ERR_DATE_INVALID, TC-01)

**โค้ดตอนนี้:** `paid_date` optional ไม่มีการตรวจเทียบ `payment_date` — `ap-payment.service.ts:849-850`, `interface/ap-payment.interface.ts:41-42`

**ขนาด:** XS — แต่รอ C1 ก่อน (นิยามวันที่ต้องนิ่ง)

### A7. เหตุผล void ≥ 5 ตัวอักษร (§4 Void, §7.2)

**โค้ดตอนนี้:** ว่างได้ default เป็น `'void'` — `ap-payment.service.ts:567`

**ขนาด:** XS — บังคับ min length ที่ DTO gateway

### A8. Input Tax Reconciliation bridge — ปลดล็อกภาษีซื้อรอใบกำกับเมื่อ PV posted (§1.2 Out-of-scope #3, §6.3) — **สำคัญสูง**

**โค้ดตอนนี้:** `tb_ap_invoice_tax` บันทึกตอน invoice (spec §6.7 "ข้อมูลอย่างเดียว"); PV `findOne` คืน `tax_invoices[]` ของ invoice ที่จ่าย; ไม่มี reclass Undue VAT → Input VAT และไม่มี flag "พร้อม reconcile"

**ขนาด:** M

**เสนอ:** sub-project **Tax** (ภ.พ.30) — ต้องตัดสินก่อนว่า trigger คือ "PV posted" (FRD) หรือ "ได้รับใบกำกับภาษี" (หลักภาษีไทย: ภาษีซื้อใช้ได้เมื่อมีใบกำกับ ไม่ใช่เมื่อจ่าย) ดู C8

### A9. AP Aging report / dashboard และ "Real-time AP Aging Rollback" (§2 5, §6.1)

**โค้ดตอนนี้:** ข้อมูลพร้อม (`unpaid_amount` คืนตอน void ใน tx เดียว) แต่ query รายงาน aging อยู่นอกขอบเขต spec §2

**ขนาด:** M → sub-project **Reporting** หรือ AP dashboard mockup v4.4.4

### A10. Print JV form, Payment Voucher print, Summary/Detail view toggle, Sticky footer (§2 4, §2 5, TC-09)

**โค้ดตอนนี้:** frontend/report ทั้งหมดอยู่นอกขอบเขต (spec §2); backend ส่งข้อมูลระดับบรรทัดแล้ว Summary view เป็นเรื่อง UI รวมยอด (ดู B2)

---

## B. โค้ดเบี่ยงจาก FRD โดยตั้งใจ (BA ปรับ FRD)

### B1. ชุดสถานะและผลของ Send Back / Reject (§2 2, §3.1 Status)

| FRD | โค้ด |
|---|---|
| `Draft`, `In Progress`, `Posted`, `Voided`, `Returned` | `draft`, `in_review`, `posted`, `void` — `schema.prisma:331-336` |
| Send Back → `Returned` | กลับ `draft` + `last_action = review/rejected` (spec §14) |
| Reject → Void/Rejected | กลับ `draft`, `last_action = rejected` |

**เสนอ:** FRD ตัด `Returned` ออก ใช้ `Draft` + activity log แสดงว่าถูกส่งกลับ (ทั้ง §2 badge list ก็ไม่มี `Returned` อยู่แล้ว ดู C6)

### B2. Allocation ระดับบรรทัด invoice ไม่ใช่ระดับใบ (§3.2, §5 `ap_payment_settle`)

**FRD:** `ap_payment_settle` 1 แถวต่อ invoice (`invoice_no`, `settle_pay_amt`) และ snapshot ยอด invoice ทั้งใบ
**โค้ด:** `tb_ap_payment_detail` 1 แถวต่อ `ap_invoice_detail_id` (unique ต่อ PV) — `schema.prisma:7365-7386`; เหตุผล: WHT profile และบัญชี AP อยู่ที่บรรทัด (spec §7.2, §8)

**ผลต่อ UI:** Summary View ต้องกระจาย `Pay Amt.` ระดับใบลงบรรทัด (เช่น ตามสัดส่วน `unpaid_amount` หรือเรียงบรรทัด) — FRD ควรระบุกติกาการกระจาย

### B3. ยอดคงเหลือเก็บจริงและมีการจองจาก PV ที่ยังไม่ post (§6.1 "No Static Balance Field")

**FRD:** ไม่เก็บ balance, คำนวณสด `total − Σ settle ที่ Posted`
**โค้ด:** เก็บ `unpaid_amount` (บรรทัด) และ `outstanding_amount` (ใบ) ปรับใน tx เดียวกับการ post — `schema.prisma:7084, 7170`, `ap-payment.posting.ts:462-483`; **และ** PV ที่เป็น `draft`/`in_review` จองยอดไว้ (`available = unpaid − held`) กันจ่ายซ้ำ — `ap-payment.validation.ts:113-121, 215-226`, `ap-payment.service.ts:329-331, 351`

**ผลที่ผู้ใช้เห็นเหมือน FRD** (ซ่อนเมื่อ 0, คืนเมื่อ void) แต่ต่างตรงที่ใบที่ถูก PV draft จองไว้จะเห็น `available_amount` ลดลง — FRD §6.1 ควรเขียนกติกาการจองนี้ (แนะนำคงไว้ เพราะ FRD เองก็ไม่มีวิธีกันสอง PV draft จ่ายใบเดียวกันเกิน)

### B4. PV ยอดสุทธิ 0 หรือติดลบ (§4 Save Draft, §7.1 "Zero Balance Allowed", TC-04) — **สำคัญสูง**

**FRD:** Save Draft ได้แม้ total = 0; guardrail ตอน submit ยอมรับ total = 0 (invoice 214 + CN −214)
**โค้ด:** ปฏิเสธตั้งแต่ create/update: `Σ applied ≤ 0` → `AP_PAYMENT_EMPTY` — `ap-payment.validation.ts:276-277`; `net_paid ≤ 0` (ทั้ง txn และ base) → `AP_INVOICE_LINE_INVALID` — `ap-payment.validation.ts:307-315`; เหตุผล: PV ที่ไม่มีเงินออกไม่มีบรรทัด Bank ให้ลง

**ทางที่ระบบรองรับอยู่แล้ว:** หักกลบ CN/DN/DP กับ invoice ทำผ่าน **Doc Reference บน AP Invoice** (spec §6.6) ไม่ใช่ PV

**เสนอ:** FRD แก้ TC-04 และ §7.1 เป็น "PV ต้องมี net > 0; การหักกลบ CN เต็มจำนวนทำที่ AP Invoice → Reference" หรือถ้า BA ยืนยันว่าต้องทำผ่าน PV ต้องเปิดประเด็นออกแบบใหม่ (JV ไม่มีบรรทัด Bank; ไม่ใช่ "payment")

### B5. บัญชี Realized FX กำหนดที่ระบบ ไม่ใช่ต่อใบ (§3.2 `fx_account_code`, §5 `fx_account_code`/`fx_cost_center`, §6.2 #5, TC-08)

**โค้ด:** `gl_setting.realized_fx_gain_account_id` / `realized_fx_loss_account_id`; เลือกอัตโนมัติตามเครื่องหมาย — `ap-payment.posting.ts:395-404`, `gl-posting.service.ts:98-102`; ไม่มีช่องให้ผู้ใช้เลือก (`ICreateApPayment` — `interface/ap-payment.interface.ts:39-54`)

**เสนอ:** FRD ตัด "User Custom Acc" และ TC-08 ออก ให้ตั้งค่าครั้งเดียวที่ GL Setting (สอดคล้อง AR PRD) — ถ้า BA ต้องการ override ต่อใบจริง ควรเป็น cost center ไม่ใช่บัญชี

### B6. Void = reversal JV เสมอ ไม่มี "cancel JV" และต้องอยู่ใน period ที่เปิด (§4 Void, §7.2, §8 ERR_VOID_CLOSED_PERIOD, TC-07, §5 `ap_payment_void_log`) — **สำคัญสูง**

| FRD | โค้ด |
|---|---|
| Period เปิด → status Voided + "Cancel linked GL JV" | reversal JV ลงวันที่ void เสมอ (`reverseBySource(now)`) ไม่มีการลบ/ยกเลิก JV ที่ post แล้ว — `ap-payment.posting.ts:281-286` |
| Period ปิด → auto-reverse วันปัจจุบัน + void_log | เหมือนกัน แต่ **ต้องการให้ period ของวันที่ void เปิดอยู่** ไม่งั้น `GL_PERIOD_NOT_OPEN` — `ap-payment.posting.ts:277-279` |
| ตาราง `ap_payment_void_log` | เก็บบน header: `void_at`, `void_by_id`, `void_reason`, `void_gl_jv_id` — `schema.prisma:7327-7330` |
| ตรวจ period ของ `paid_date` | ตรวจ period ของวันที่ void (ไม่ใช่วันที่ PV) |

**เหตุผล:** GL immutable (spec §5.3) การ "cancel" JV ที่ post แล้วทำให้ trial balance ย้อนหลังเปลี่ยน

**เสนอ:** FRD รวม 2 กรณีเป็นกรณีเดียว: "Void สร้าง reversal JV ลงวันที่ void; วันที่ void ต้องอยู่ใน period ที่เปิด" และตัด `ap_payment_void_log` (ใช้ `void_*` บน header + activity log)

### B7. สกุลเงินของ PV มาจาก invoice ที่จ่าย ไม่ใช่เลือกที่ header (§3.1 Currency)

**โค้ด:** currency = currency ของ invoice ใบแรก และทุกใบต้องสกุลเดียวกัน — `ap-payment.service.ts:880-890`, `ap-payment.validation.ts:272-275`; ธนาคารต้องเป็นสกุล PV หรือ base — `ap-payment.validation.ts:403-407`; จ่ายข้ามสกุลอยู่นอกขอบเขต (spec §2)

**เสนอ:** FRD §3.1 Currency เปลี่ยนเป็น readonly (derive จากเอกสารที่เลือก) และตัด default `USD` (ดู C4)

### B8. Tab 2 Journal เป็น preview อ่านอย่างเดียว ไม่ใช่ Fast-Entry ที่แก้ได้ (§1.2 #7, §2 4 "Interactive Fast-Entry Grid") — **สำคัญสูง**

**โค้ด:** JV สร้างจากข้อมูล PV ตอน post แบบ deterministic (`buildPaymentJvLines`) และตรวจว่ายอด header ไม่ drift — `ap-payment.posting.ts:350-370`; ไม่มี input ให้แก้บรรทัด JV; dimension บรรทัด AP copy จาก invoice line

**เหตุผล:** subledger ต้องเท่ากับ GL เสมอ (spec §5.2) ถ้าผู้ใช้แก้บัญชี/ยอดใน JV ได้ AP aging กับ GL จะไม่ตรงกัน

**เสนอ:** FRD เปลี่ยน Tab 2 เป็น "JV Preview (read-only)"; สิ่งที่แก้ได้คือข้อมูลต้นทางใน Tab 1 (บัญชี expense, WHT amount, cost center) แล้ว preview คำนวณใหม่ — ถ้า BA ต้องการ "บรรทัดปรับปรุงเพิ่ม" ควรเป็น manual JV แยกใน GL

### B9. เลขที่เอกสารและ JV prefix มาจาก config ไม่ใช่ hard-code (§3.1 `APPV{YY}{MM}{5}`, §7.1 "Prefix PV")

**โค้ด:** running code type `AP-PV` ตาม pattern ที่ตั้งใน BU — `ap-payment.running-code.ts:15, 59-68`; JV prefix จาก `gl_setting.ap_jv_prefix_id` — `gl-subledger-posting.service.ts:119-122`

**เสนอ:** FRD เขียนเป็น "default `APPV{YY}{MM}{00000}` ตั้งค่าได้ที่ Running Code" และ JV prefix อ้าง Master Data / JV Prefix FRD

### B10. Guardrail ยอดจ่าย (§2 5, §4 Submit #2, §8 ERR_PAYMENT_DISCREPANCY, TC-05)

**โค้ด:** มีธนาคารเดียว บรรทัด Cr Bank = `net_paid = applied − wht + expense` โดยนิยาม — `ap-payment.logic.ts:204-205, 375-385` จึงไม่มี guardrail นี้; จะจำเป็นเมื่อทำ A1

**เสนอ:** ผูกกับ A1 และแก้สูตรตาม C2

### B11. WHT คำนวณในสกุลเอกสาร แปลงเป็นบาทด้วย rate ของ PV (§3.4 `tax_base` "เงินบาท")

**โค้ด:** `base_amount`/`wht_amount` สกุลเอกสาร; base currency = × `exchange_rate` ของ PV — `ap-payment.logic.ts:109, 121, 190, 394`; default มาจาก tax profile ของบรรทัด invoice ผ่าน `wht-preview`

**เสนอ:** FRD ระบุว่าใบรับรอง 50 ทวิ ใช้ยอดบาทที่แปลงแล้ว (rate วันจ่าย) — ให้ BA/ที่ปรึกษาภาษียืนยันว่าใช้ rate วันจ่ายได้

### B12. ผู้รับเงิน (payee) snapshot จาก vendor master ครั้งเดียวต่อ PV แก้ไม่ได้ (§3.4 Name / Tax ID / Branch / Address ต่อใบรับรอง)

**โค้ด:** `payee_name` = `vendor.name`, `payee_tax_no`, `payee_branch_no` จาก `tb_vendor`; `payee_address` จาก `tb_vendor_address` ประเภท register_address — `ap-payment.writer.ts:253-256`, `ap-payment.service.ts:934`; `ICreateApPayment` ไม่รับ payee override

**เสนอ:** ถ้าต้องแก้ payee ต่อใบ (เช่น จ่ายเช็คในนามอื่น) ให้เพิ่มใน A1 (`pay_to_name` ต่อแถว payment method) ส่วนข้อมูลภาษีของผู้ถูกหักให้แก้ที่ vendor master

---

## C. FRD ผิดหรือขัดกันเอง (BA ตัดสิน)

### C1. วันที่ไหนเป็นวันที่ลง GL และตรวจ period — **สำคัญสูง**

| จุดใน FRD | ระบุ |
|---|---|
| §3.1 Input Date | "ไม่สามารถเลือกย้อนหลังไปในงวดที่ปิดแล้ว" |
| §3.1 Paid Date | "ต้องอยู่ใน Open Period" และ Mandatory |
| §4 Submit #5 | "เปลี่ยนเป็น Posted, **แสตมป์ Paid Date**" (ระบบใส่เอง?) |
| §7.2 Void | "Check Accounting Period of Payment.**paid_date**" |
| §6.1 / TC-02 | กรอง invoice ด้วย **input_date** |

**โค้ด:** `payment_date` = วันที่ลง GL และตรวจ period (spec §4.3), `paid_date` = วันเงินออกจริง optional ไม่มีผลกับ GL — `schema.prisma:7288-7289`, `ap-payment.posting.ts:220, 231`

**ต้องตัดสิน:** วันที่เดียวที่ขับ JV/period คือ Input Date หรือ Paid Date — แนะนำ **Paid Date** เป็นวันที่ลง GL (บรรทัด Bank ต้องตรง statement สำหรับ bank reconciliation) และ Input Date เป็นแค่วันที่สร้างเอกสาร; ถ้าตกลงแบบนี้ ความหมาย `payment_date` ในโค้ด = Paid Date ของ FRD และ A5/A6 ต้องเทียบกับวันนี้

### C2. สูตร guardrail นับ WHT ซ้ำ (§2 5, §4 Submit #2, §8) — **สำคัญสูง**

FRD: `Σ Payment Method − WHT + Other Exp = Net Payment`
แต่ §5 นิยาม `net_payment_amount` = Settle − WHT + Other Exp อยู่แล้ว แทนค่าจะได้ `Σ Payment Method = Settle` ซึ่งผิด (เงินออกจากธนาคารต้องไม่รวม WHT ที่หักไว้)

**สูตรที่ถูก:** `Σ Payment Method = Net Payment` (= Settle − WHT + Other Exp) — ตรงกับบรรทัด Cr Bank ในโค้ด (`ap-payment.logic.ts:204`); TC-05 ควรใส่ตัวเลข WHT/Exp ให้ครบเพื่อทดสอบได้

### C3. PV ยอด 0 (§4 Save Draft, §7.1, TC-04) — **สำคัญสูง**

FRD ยอมให้ total = 0 ทั้ง draft และ submit แต่ไม่ได้บอกว่า JV จะมีหน้าตาอย่างไร (ไม่มีบรรทัด Bank) และไม่มีเลขที่จ่ายเงินจริง — ผูกกับ B4; แนะนำตอบด้วยการชี้ไป AP Invoice Reference

### C4. Default currency ขัดกัน (§3.1 vs §5)

§3.1 "Default: `USD` หรือตาม Vendor Profile" แต่ §5 `currency_code DEFAULT 'THB'` และ base currency คือ THB — ตาม B7 ควรตัด default ทิ้ง

### C5. Schema ใน §5 ไม่ตรงกับหลักของระบบ

- `ap_payment_settle.invoice_no VARCHAR` ไม่ใช่ FK; โค้ดใช้ `ap_invoice_id`/`ap_invoice_detail_id` uuid FK
- `status`, `source`, `wht_form`, `wht_condition` เป็น VARCHAR; โค้ดใช้ enum
- `ap_payment_header` ไม่มี `doc_version`, `workflow_*`, `gl_jv_id`, `base_*` ของแต่ละยอด; snapshot ยอด invoice ซ้ำใน settle ทำให้ต้อง sync
- ไม่มี `deleted_at` (soft delete) ทั้งที่ระบบใช้ทั่วทั้ง tenant

**เสนอ:** ให้ §5 อ้าง `schema.prisma:7282-7424` เป็นแหล่งจริงแทนการเขียน DDL ใหม่

### C6. รายการสถานะไม่ตรงกันในเอกสารเดียว

§2 badge: Draft / In Progress / Posted / Voided; §3.1: เพิ่ม `Returned` — ดู B1

### C7. รายการรหัสบริการ WHT hard-code และไม่ครบ (§3.4 Service Category)

ระบุ 01 (5%), 02 (2%), 03 (1%), 05 (3%), 06 (3%) — ข้าม 04 และอัตราไม่ครบทุกกรณี (เช่น ภ.ง.ด.2 ดอกเบี้ย/ปันผลมีอัตราอื่น) — ควรอ้าง **Master Data / WHT Service Type FRD v1.0** และ tax profile แทน (โค้ด: `tb_tax_profile` type `wht` — `ap-payment.validation.ts:409-418`)

### C8. Input Tax bridge ปลดล็อกตอน "PV Posted" (§6.3)

หลักภาษีซื้อไทยใช้สิทธิ์เมื่อ**มีใบกำกับภาษี**ในเดือนภาษีนั้น ไม่ผูกกับการจ่ายเงิน (ยกเว้น undue VAT ของค่าบริการที่เกิดเมื่อจ่าย) — FRD ควรแยก 2 กรณี: สินค้า (ใบกำกับตอนตั้งหนี้) กับบริการ (undue → due เมื่อจ่าย) ก่อน dev ทำ A8

### C9. นิยาม `source` = `Auto`, `Posting` (§3.1)

ไม่มีคำอธิบายว่า `Auto` (จาก payment run?) และ `Posting` (จาก GL?) ต่างกันอย่างไร และ `AI Upload` ต่างจาก AP Invoice `AI` อย่างไร — ต้องนิยามก่อนทำ A3

### C10. เงื่อนไข Void "Document is Active" (§4 Void)

ไม่ชัดว่า `In Progress` void ได้ไหม — โค้ดอนุญาตเฉพาะ `draft` และ `posted`; ใบที่อยู่ระหว่างอนุมัติต้อง reject กลับ draft ก่อน — `ap-payment.service.ts:562`; BA ยืนยัน

---

## D. สรุป backlog สำหรับ dev

| # | เรื่อง | sub-project | ขนาด | รอ BA |
|---|---|---|---|---|
| A1 | Multi-bank payment methods | Cash & Bank | M | C2 |
| A2 | WHT certificate 50 ทวิ | Tax | L | — |
| A8 | Input tax bridge (undue → due) | Tax | M | C8 |
| A3 | `source` enum + source doc | AP follow-up | S | C9 |
| A4 | Dimension บน expense line | AP follow-up | S | — |
| A5 | กรอง invoice ตามวันที่ | AP follow-up | XS | C1 |
| A6 | `paid_date ≥ payment_date` | AP follow-up | XS | C1 |
| A7 | void reason ≥ 5 chars | AP follow-up | XS | — |
| A9 | AP aging query | Reporting | M | — |
| A10 | Print / UI | Frontend | — | — |

ข้อที่ไม่รอ BA (A4, A7) ทำได้ทันทีเป็น PR เล็กใน micro-business; A5/A6 รอ C1 เพราะเปลี่ยนความหมายวันที่แล้วต้องแก้ทั้งคู่
