# Jira AP tasks — extract from epic CA-4 (2026-10-01)

Source: Jira project CA (carmensoftware.atlassian.net), `parent = CA-4`, pulled 2026-10-01 via twg REST.
Purpose: reconcile the AP backlog (written 2026-09-21 from FRD mockups) against what backend-v2 PR #671 (merged 2026-09-23) actually implemented in `micro-business/src/ap`.

Totals: 29 issues (13 BE, 14 FE), 106 story points. All still **To Do**; epic CA-4 moved to In Progress 2026-10-01.

> **Link direction note:** at extract time Jira had the links reversed — CA-38 (design doc) *blocked by* CA-39 (backlog breakdown) and every FE story *blocking* its BE pair (CA-181 blocks CA-182, …), plus CA-180 blocking spike CA-172. **Fixed 2026-10-01**: the 12 links were deleted and recreated as design → backlog, spike → FE, BE → FE. The "Blocked by / Blocks" columns below still reproduce the pre-fix Jira state; read them reversed. CA-39 was also Resolved the same day (backlog exists; this file is the key map it asked for).

## Reconciliation vs backend-v2 `81802ec97` (PR #671)

Verified 2026-10-01 against `apps/micro-business/src/ap/{ap-invoice,ap-payment}` and gateway `apps/backend-gateway/src/application/{ap-invoice,ap-payment}`. Abbreviations: `inv/` = ap-invoice, `pay/` = ap-payment, `gw/` = gateway application, `schema` = packages/prisma-shared-schema-tenant/prisma/schema.prisma, `gl-sub` = gl/gl-subledger-posting/gl-subledger-posting.service.ts.

| Key | Verdict | Evidence | Gaps vs acceptance |
|---|---|---|---|
| CA-182 header + doc no | **Implemented** | schema:7104-7188; running code inv/ap-invoice.running-code.ts:100-130; rate snapshot inv/ap-invoice.service.ts:1197-1219; void draft service.ts:775-791 | snapshot lacks rate_date/rate_type/rate_source (work item only, not acceptance) |
| CA-184 line calc | **Partial** | inv/ap-invoice.logic.ts:59-108 (`computeLine`, HALF_UP 2dp); lines only for posting contract logic.ts:393-436 | amounts are JSON numbers not decimal strings (serializer.ts:10-11); rounding fixed 2dp not `currency.decimal_places`; no explicit Rounding Adjustment line (avoided by construction) |
| CA-186 reference offset | **Partial** | `tb_ap_invoice_reference` schema:7292-7311; candidates service.ts:552-592; offset at post inv/ap-invoice.posting.ts:736-757; FX posting.ts:778-797 | PO is not a reference type (only posted CN/deposit); GRN linked per line via `tb_ap_invoice_detail_source`; no net/tax split per reference; numbers not decimal strings |
| CA-187 post invoice via contract | **Partial** | posting.ts:278-312 → `GlSubledgerPostingService.postFromSource` gl-sub:109-150; balance check gl-jv.validation.ts:47; idempotent by (source, ref type, ref id) gl-sub:308-324 + unique index | no Journal Entry preview endpoint; contract has no `source_version`/`posting_rule_code`/`idempotency_key` (idempotency = source ref id); balance check inside facade, not in AP |
| CA-188 settlement select | **Partial** | `GET ap-payment/outstanding-documents?vendor_id` gw controller:89-143 → pay/ap-payment.service.ts:312-356; `tb_ap_payment_detail` schema:7422-7444; partial settlement reserve service.ts:812-828, reduce at post pay/ap-payment.posting.ts:462-483 | no doc-type filter or search params; no net/VAT/total snapshot per settlement line; numbers not decimal strings |
| CA-190 multi-bank allocation | **Not started** | header single `bank_account_id` + enum `payment_method` schema:7357-7360, enum 339-345 | no payment-method line table, no allocation, payment type is enum not master FK |
| CA-192 other expense | **Partial** | `tb_ap_payment_expense` schema:7466-7476; writer pay/ap-payment.writer.ts:126-136; JV Dr lines pay/ap-payment.logic.ts:361-430 | description optional (story requires reject); no dimension1/2; whole-set replace via PUT, no per-line endpoint; no preview |
| CA-193 post payment via contract | **Partial** | pay/ap-payment.posting.ts:215-249, lines 350-370; totals re-verified 358-363; same balance/idempotency as CA-187 | no Journal GL preview (only `POST wht-preview` gw:144); no idempotency_key/source_version/posting_rule_code |
| CA-194 void before post only | **Partial (contradicts story)** | void draft pay/ap-payment.service.ts:728-757 | posted vouchers ARE voidable via reversal JV (service.ts:555-571 → posting.ts:271-309 `reverseBySource`); `in_review` cannot be voided. Story's "reject void when Posted" is deliberately not how it was built |
| CA-195 AP outstanding aggregate | **Not started** | generic list only inv/ap-invoice.service.ts:148-185 | no open-documents/aging/due-date aggregate endpoint |
| CA-197 pending approvals query | **Not started** | `workflow_current_stage` persisted service.ts:852-866 | no endpoint; closest is list `doc_status=in_review` |
| CA-199 tax reconciliation/VAT | **Not started** | data exists `tb_ap_invoice_tax.tax_status`, filing_month/year schema:7312-7338 | no group-by / filing-period endpoint |
| CA-201 cross-doc activity log | **Not started** | shared activity-log module exists (src/log/activity-log) | AP never writes to it (0 hits in src/ap); only per-doc `workflow_history` JSON |

Gateway AP routes that exist today: ap-invoice `GET /`, `GET grn-candidates`, `GET :id`, `GET :id/reference-candidates`, `POST /`, `POST from-grn`, `POST :id/{submit,approve,review,reject,void}`; ap-payment `GET /`, `GET outstanding-documents`, `GET :id`, `POST /`, `POST wht-preview`, `POST :id/{submit,approve,review,reject,void}`.

**Where the implementation deliberately differs from the backlog**
- Single table `tb_ap_invoice` with `doc_type` for invoice / debit note / credit note / deposit; "offset" = referencing a posted CN or deposit of the same vendor+currency. PO is not an AP reference; GRN is matched per line.
- Posting contract is `ISubledgerPostingInput {source, source_ref_type, source_ref_id, jv_date, description, currency_id, exchange_rate, lines[]}`; idempotency by DB unique index on source ref, not a caller-supplied key.
- Every amount crosses the API as a JSON number (`z.coerce.number()`); internally `Prisma.Decimal` HALF_UP 2dp. Contradicts the decimal-string bullets in CA-184/186/188/190/195 — needs a BA/tech decision, not a per-story fix.
- Statuses are draft / in_review / posted / void; approve == post. No "In Progress"/"Rejected" states from the mockup.
- No preview or dashboard read models at all.

**Frontend:** no AP UI exists in carmen-platform or carmen-inventory-frontend-react (CA-176..181, 183, 185, 189, 191, 196, 198, 200, 202 all genuinely To Do).

**Summary:** 1 implemented, 7 partial, 5 not started out of 13 BE stories; 0 of 14 FE stories started.

## Index

| Key | Side | Status | Pts | Blocked by | Blocks | Summary |
|---|---|---|---|---|---|---|
| CA-38 | - | Resolved | 3 | CA-39 |  | เขียน design doc |
| CA-39 | - | To Do | 2 |  | CA-38 | แตก backlog หลัง design อนุมัติ |
| CA-176 | FE | To Do | 2 |  |  | Navigation Shell, BU Selector และ EN/TH Toggle |
| CA-177 | FE | To Do | 3 |  |  | Approved POs List และปุ่มเริ่มรายการ |
| CA-178 | FE | To Do | 3 |  |  | List รวมหลาย Doc Type และคอลัมน์ Status |
| CA-179 | FE | To Do | 3 |  |  | โครงเอกสารระดับ Skeleton |
| CA-180 | FE | To Do | 2 |  | CA-172 | เลือกและแสดง Tax Profile Snapshot |
| CA-181 | FE | To Do | 3 |  | CA-182 | ฟอร์มหัวเอกสารและ action bar |
| CA-182 | BE | To Do | 5 | CA-181 |  | บันทึกหัวเอกสารและออกเลขที่เอกสาร |
| CA-183 | FE | To Do | 5 |  | CA-184 | ตารางรายการ item และ dialog Edit Invoice Item Detail |
| CA-184 | BE | To Do | 5 | CA-183 |  | คำนวณและบันทึก Sub total/Discount/Total ต่อบรรทัด |
| CA-185 | FE | To Do | 3 |  | CA-186 | ตาราง Referenced Source Documents & Offset Liabilities |
| CA-186 | BE | To Do | 5 | CA-185 |  | จับคู่และหักออฟเซ็ต PO/Deposit กับยอด Invoice |
| CA-187 | BE | To Do | 8 |  |  | Post Invoice ต้องยิงผ่าน GL posting contract ไม่เขียน ledger เอง |
| CA-188 | BE | To Do | 5 | CA-189 |  | เลือก Invoice/Credit Note ค้างชำระมาตัดจ่ายเข้า Settlement ของ Payment Voucher |
| CA-189 | FE | To Do | 3 |  | CA-188 | modal เลือก Invoice/Credit Note ค้างชำระมา Apply to Batch |
| CA-190 | BE | To Do | 5 | CA-191 |  | บันทึกวิธีจ่ายเงินหลายรายการ (Multi-Bank Allocation) ต่อหนึ่ง Payment Voucher |
| CA-191 | FE | To Do | 3 |  | CA-190 | หน้าจอ Add Payment Method และตาราง Multi-Bank Allocation |
| CA-192 | BE | To Do | 2 |  |  | เพิ่มรายการ Other Expense/Miscellaneous Surcharge เข้า Payment Voucher |
| CA-193 | BE | To Do | 8 |  |  | Post Payment ต้องยิงผ่าน GL posting contract ไม่เขียน ledger เอง |
| CA-194 | BE | To Do | 2 |  |  | Void Payment Voucher ทำได้เฉพาะก่อน Post เท่านั้น |
| CA-195 | BE | To Do | 5 | CA-196 |  | query/aggregation endpoint ของเอกสาร AP ค้างชำระ (feed ให้ AP Aging Analysis และ Due Date Tracker) |
| CA-196 | FE | To Do | 3 |  | CA-195 | แสดง stat tile และ panel AP Aging Analysis กับ Due Date Tracker |
| CA-197 | BE | To Do | 3 | CA-198 |  | query endpoint รายการ AP document ที่รออนุมัติ (feed Pending Approvals Queue) |
| CA-198 | FE | To Do | 2 |  | CA-197 | แสดง panel Pending Approvals Queue |
| CA-199 | BE | To Do | 5 | CA-200 |  | query/aggregation endpoint ของ Tax Reconciliation & VAT ตาม tax status |
| CA-200 | FE | To Do | 3 |  | CA-199 | แสดง panel Tax Reconciliation & VAT |
| CA-201 | BE | To Do | 3 | CA-202 |  | query endpoint activity log ข้ามเอกสาร AP (feed Review & Status Audit Tracker) |
| CA-202 | FE | To Do | 2 |  | CA-201 | แสดง panel Review & Status Audit Tracker |

## Detail

### CA-38 — [DOC] Accounts Payable — เขียน design doc

- Status: **Resolved** · Points: 3 · Labels: accounting, ap, doc
- Blocked by: CA-39 · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-38

**บริบท**
README.md ระบุ AP เป็นขั้นส่งมอบที่ 5 แต่ยังไม่มีเอกสารออกแบบ
จึงยังแตก story ที่มี acceptance criteria ไม่ได้

**สิ่งที่ต้องทำ**
- กำหนดขอบเขต AP ที่ไม่ทับกับ Accounting Foundation
- ระบุ posting rule ที่ AP จะเรียกผ่าน posting contract ของ accounting-foundation.md §9
- ระบุ master data ที่ reuse และที่ AP เป็นเจ้าของ
- ระบุ idempotency key และ source version ที่ AP จะส่งเข้า Journal Staging
- ระบุ open decisions ที่ต้องตัดสินก่อน implementation

**Acceptance**
- มีไฟล์ design doc ของ AP อยู่ใน repository
- เอกสารระบุ posting contract ที่ใช้และไม่มีการเขียน ledger โดยตรง
- เอกสารผ่าน review และ merge เข้า main แล้ว

**กฎที่ห้ามละเมิด**
- README.md — AP, AR และ Fixed Assets ต้องส่งรายการเข้า GL ผ่าน posting contract เดียวกัน ห้ามเขียน journal tables โดยตรง
- accounting-foundation.md §5 — Inventory, AP, AR, Fixed Assets และ external API ต้องส่ง deterministic idempotency key
- ห้ามแต่ง acceptance criteria ของฟีเจอร์ AP ก่อนเอกสารนี้ถูกอนุมัติ

> **Comment 2026-10-01 carmensoftware Developer:** Design doc ของ AP รวมอยู่ใน spec foundation + AP ซึ่ง merge เข้า main ของ carmen-accounting-concept แล้ว (2026-09-23)
- 2026-09-23-accounting-foundation-ap-design.md — scope AP, posting ผ่าน subledger posting facade, master data ที่ reuse/own, open decisions
- 2026-09-23-accounting-foundation-ap.md (plan) — implementation plan 17 tasks
Acceptance ครบ (ไฟล์อยู่ใน repo, ระบุ posting contract, merge แล้ว) จึงปิดเป็น Resolved

### CA-39 — [DOC] Accounts Payable — แตก backlog หลัง design อนุมัติ

- Status: **To Do** · Points: 2 · Labels: accounting, ap, doc
- Blocked by: - · Blocks: CA-38
- Jira: https://carmensoftware.atlassian.net/browse/CA-39

**บริบท**
ทำหลัง design doc ของ AP ถูกอนุมัติแล้วเท่านั้น

**สิ่งที่ต้องทำ**
- แตก epic นี้เป็น story ตามกติกาในสเปค docs/superpowers/specs/2026-08-31-accounting-jira-backlog-design.md
- แยก FE กับ BE เป็นคนละ story และผูก blocks ให้ครบ
- ใส่ context, work, acceptance และ rules ครบทุกใบ

**Acceptance**
- validate ผ่านโดยไม่มี error
- story ทุกใบอ้างหัวข้อใน design doc ที่เพิ่งอนุมัติ
- issue ถูกสร้างใน Jira ครบและบันทึกใน .jira-map.json

**กฎที่ห้ามละเมิด**
- ห้ามแต่ง acceptance criteria ที่ design doc ไม่ได้ระบุ
- Story point ต้องอยู่ในชุด 1, 2, 3, 5, 8

### CA-176 — [FE] AP Module — Navigation Shell, BU Selector และ EN/TH Toggle

- Status: **To Do** · Points: 2 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-176

**บริบท**
Mockup element extraction ของกลุ่ม AP แสดง top bar และเมนูหลักของทั้งโมดูล
(BU selector, ปุ่มสลับภาษา EN/TH, เมนู 12 รายการ) เนื้อหาในแต่ละหน้าเมนู
(Invoice entry, Payment/Payment Approval, Dashboard/Aging) เป็นขอบเขตของ
agent อื่น story นี้ทำเฉพาะโครง nav shell ที่ครอบทุกหน้า

**สิ่งที่ต้องทำ**
- สร้าง BU selector (dropdown) พร้อมตัวเลือกตามรูปแบบที่เห็น เช่น BU-01 (Bangkok Grand Hotel & Spa)
- สร้างปุ่มสลับภาษา EN/TH
- สร้างเมนูหลักตามลำดับที่ mockup แสดง — Dashboard, PO & Deposit Requests, Deposit, Invoice, Debit Note, Credit Note, Payment, Vendor, Procedure, Setting, Configuration, General Ledger
- ทำ routing ไปหน้าที่เกี่ยวข้องต่อเมนู (เนื้อหาในหน้า Invoice/Payment/ Dashboard เป็น placeholder ที่รอ agent เจ้าของ implement)
- แสดง badge ตัวเลขที่เมนู "PO & Deposit Requests" ตามค่าที่ backend ส่งมา

**Acceptance**
- ผู้ใช้เห็นเมนูครบ 12 รายการตามลำดับใน mockup พร้อม BU selector และปุ่ม EN/TH
- คลิกแต่ละเมนูแล้ว navigate ไปหน้าที่ถูกต้อง (หน้าเนื้อหาอาจเป็น placeholder ถ้ายังไม่ implement โดย agent เจ้าของ)
- เมนู "PO & Deposit Requests" แสดง badge ตัวเลขได้เมื่อมีค่าจาก backend

**กฎที่ห้ามละเมิด**
- อ้าง AP/hotel_ap_erp_mockup_v4_4_3.html — nav list (Dashboard, PO & Deposit Requests badge '2', Deposit, Invoice, Debit Note, Credit Note, Payment, Vendor, Procedure, Setting, Configuration, General Ledger), BU selector (BU-01..BU-03), ปุ่ม EN/TH — ยังไม่มี FRD ยืนยัน
- เมนู Setting และ Configuration ปรากฏใน mockup เป็นชื่อปุ่มเท่านั้น ไม่มี field หรือหน้าจอย่อยที่ยืนยันได้จากหน้าจอ — เปิดเป็น placeholder page เท่านั้น ห้ามแต่งเนื้อหาของหน้า Setting/Configuration เอง
- เมนู General Ledger ชี้ไปยังโมดูล GL ที่แยกอยู่แล้ว (general-ledger-journal-voucher-design.md) — story นี้ทำแค่ nav entry ไม่ทำเนื้อหาหน้า General Ledger Explorer

### CA-177 — [FE] PO & Deposit Requests — Approved POs List และปุ่มเริ่มรายการ

- Status: **To Do** · Points: 3 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-177

**บริบท**
Mockup แสดงหน้า "PO & Deposit Requests" ที่มี Approved POs List พร้อมปุ่ม
"Request Deposit" และ "+ Receive Goods (STD)" และ hint text
"Click Request Deposit" PO เองเป็นข้อมูลของ Procurement/Inventory ไม่ใช่
ของ Accounting — story นี้ทำเฉพาะโครงหน้าจอฝั่ง AP ที่ดึง PO ที่อนุมัติแล้ว
มาแสดงเพื่อเริ่ม flow

**สิ่งที่ต้องทำ**
- แสดงตาราง Approved POs List (field ต่อแถวยังไม่ยืนยันจาก mockup — ใส่อย่างน้อยเท่าที่จำเป็นต่อการเลือกทำ action)
- ปุ่ม "Request Deposit" ต่อรายการ/ต่อ selection เพื่อเริ่ม flow ขอเงินมัดจำ
- ปุ่ม "+ Receive Goods (STD)" สำหรับเริ่ม flow รับของตาม PO
- แสดง hint/empty-state "Click Request Deposit" เมื่อยังไม่มีรายการที่ เลือก/ยังไม่ทำ action

**Acceptance**
- หน้า PO & Deposit Requests แสดง Approved POs List พร้อมปุ่ม Request Deposit และ + Receive Goods (STD) ตามที่เห็นใน mockup
- แสดง hint text "Click Request Deposit" ในสถานะที่ยังไม่ได้เลือก/ทำ รายการ ตรงกับ mockup
- ไม่มี field หรือ validation เพิ่มเติมที่ mockup ไม่ได้แสดง

**กฎที่ห้ามละเมิด**
- อ้าง AP/hotel_ap_erp_mockup_v4_4_3.html — h2 'PO & Deposit Requests', h3 'Approved POs List', ปุ่ม 'Request Deposit', ปุ่ม '+ Receive Goods (STD)', h4 'Click Request Deposit' — ยังไม่มี FRD ยืนยัน
- mockup ไม่ได้ระบุ field ต่อแถวของ Approved POs List หรือปลายทางหลังกด Request Deposit — ห้ามเดาผลลัพธ์ ต้องยืนยันกับ Procurement/Inventory PO API ก่อน implement

### CA-178 — [FE] AP Documents — List รวมหลาย Doc Type และคอลัมน์ Status

- Status: **To Do** · Points: 3 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-178

**บริบท**
Mockup มี h2 "Accounts Payable Documents" พร้อมตารางคอลัมน์ Doc Type,
Doc No, Doc Date, Vendor Invoice No, Vendor/Supplier, Due Date,
Net Amount, Status, Action ซึ่งดูเป็น list กลางที่ครอบคลุมเอกสาร AP
หลายประเภท ปุ่ม "New Invoice Entry" เหนือตารางเป็นขอบเขตของ agent
ap.invoice.* — story นี้แตกเฉพาะโครง list และคอลัมน์ status ไม่แตะ
behavior การสร้าง/แก้ invoice

**สิ่งที่ต้องทำ**
- สร้างตาราง AP Documents ด้วยคอลัมน์ตามที่ mockup แสดง — Doc Type, Doc No, Doc Date, Vendor Invoice No, Vendor/Supplier, Due Date, Net Amount, Status, Action
- รองรับแสดงหลาย Doc Type ในตารางเดียว (ค่าที่เป็นไปได้อย่างน้อยตาม เมนูที่มี — enum จริงต้องยืนยันจาก backend)
- คอลัมน์ Status แสดงสถานะเอกสารต่อแถว (label/ค่าที่เป็นไปได้ mockup ไม่ได้ระบุ — รอ backend ยืนยัน)
- ปุ่ม Action ต่อแถวเป็น placeholder ตาม permission (mockup ไม่ได้ระบุว่า เป็นปุ่มอะไรบ้าง)

**Acceptance**
- ตาราง AP Documents แสดงคอลัมน์ครบ 9 คอลัมน์ตามลำดับใน mockup
- ตารางแสดงเอกสารได้มากกว่าหนึ่ง Doc Type ในหน้าเดียว
- ไม่ทำปุ่ม "New Invoice Entry" หรือ behavior การเปิดฟอร์ม invoice entry (เป็นของ agent ap.invoice.*)

**กฎที่ห้ามละเมิด**
- อ้าง AP/hotel_ap_erp_mockup_v4_4_3.html — h2 'Accounts Payable Documents', th: Doc Type, Doc No (doc_no), Doc Date, Vendor Invoice No, Vendor/Supplier, Due Date, Net Amount, Status, Action — ยังไม่มี FRD ยืนยัน
- ปุ่ม 'New Invoice Entry' อยู่ในขอบเขตของ agent ap.invoice.* ห้ามแตะ — story นี้ทำเฉพาะโครง list/status
- mockup ไม่ได้ระบุค่า status ที่เป็นไปได้ (enum) หรือความหมายของปุ่มใน คอลัมน์ Action — ห้ามแต่ง ต้องยืนยันก่อน implement

### CA-179 — [FE] Deposit, Debit Note, Credit Note — โครงเอกสารระดับ Skeleton

- Status: **To Do** · Points: 3 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-179

**บริบท**
เมนู Deposit, Debit Note, Credit Note ปรากฏใน nav (ชื่อปุ่มเท่านั้น)
แต่ mockup element extraction ที่ได้มาไม่ได้เปิดหน้าจอ detail ของสาม
เอกสารนี้เลย — field/ตารางทั้งหมดที่เห็น (Doc No, Vendor, Invoice No,
Tax Invoice, Item lines ฯลฯ) อยู่ในหน้า "New Invoice Entry" ซึ่งเป็น
ขอบเขตของ agent ap.invoice.* สิ่งที่พอยืนยันได้คือ toolbar ปุ่มทั่วไปของ
เอกสาร (Back to List, Add, Draft, Save, Void, Copy, Print, Log) ที่ดู
เป็น generic action ไม่ได้ผูกกับ field เฉพาะ Invoice (ต่างจาก AI(OCR),
Tax Doc., Approve/Reject/Send Back, Item Details, Tax Invoice,
Doc Reference, Journal(GL), Delete ที่ผูกกับ Invoice โดยตรง)

**สิ่งที่ต้องทำ**
- ให้ Deposit, Debit Note, Credit Note ใช้ list เดียวกับ ap.main.document-list (กรองด้วย Doc Type ของตัวเอง)
- เปิดหน้า detail ที่มี toolbar ทั่วไปตามที่ยืนยันได้ — Back to List, Add, Draft, Save, Void, Copy, Print, Log
- ไม่แตะ field เฉพาะเอกสาร (header field, item line, tax section) ในรอบนี้ เพราะ mockup ไม่ได้เปิดหน้าจอเหล่านี้ให้เห็น

**Acceptance**
- เมนู Deposit, Debit Note, Credit Note เปิดหน้าจอที่มี toolbar ทั่วไป ตามที่ระบุ (Back to List, Add, Draft, Save, Void, Copy, Print, Log)
- ไม่มีการเพิ่ม field เฉพาะเอกสาร (เช่น invoice no, tax invoice) ให้กับ สามเอกสารนี้ในรอบนี้ เพราะไม่มีหลักฐานจาก mockup

**กฎที่ห้ามละเมิด**
- อ้าง AP/hotel_ap_erp_mockup_v4_4_3.html — ปุ่มเมนู Deposit, Debit Note, Credit Note (nav) และปุ่ม toolbar ทั่วไป Back to List/Add/Draft/Save/ Void/Copy/Print/Log — ยังไม่มี FRD ยืนยัน
- mockup element extraction ไม่ได้จับภาพหน้าจอ detail ของ Deposit/ Debit Note/Credit Note โดยตรง (เห็นเฉพาะหน้า Invoice entry) — ห้ามเดา field เฉพาะเอกสารจากที่เห็นในหน้า Invoice story นี้จงใจเขียน หยาบเพื่อรอหลักฐานเพิ่ม

### CA-180 — [FE] Vendor Lookup ในเอกสาร AP — เลือกและแสดง Tax Profile Snapshot

- Status: **To Do** · Points: 2 · Labels: accounting, ap, fe, master-data
- Blocked by: - · Blocks: CA-172
- Jira: https://carmensoftware.atlassian.net/browse/CA-180

**บริบท**
Mockup แสดง dropdown เลือก vendor ในฟิลด์ "Vendor *" ด้วยตัวเลือกรูปแบบ
code : name (C003, CP001, LOT02, SINO04) และในส่วน "Vendor Tax Profile"
ของ Tax Invoice section มี field Registered Name (TH/EN), Tax
Registration ID (13 หลัก), Branch No. (5 หลัก), Registered Billing
Address ซึ่งเป็นข้อมูลระดับ master ที่เอกสาร AP ต้องดึงมาแสดง/snapshot
ส่วนใครเป็นเจ้าของ vendor master เต็มรูปแบบเป็นคำถามที่แยกไว้ที่
spike.ap-vendor-master-ownership — story นี้ทำเฉพาะฝั่ง "เลือก/แสดง"
vendor ที่มีอยู่แล้วในเอกสาร AP เท่านั้น

**สิ่งที่ต้องทำ**
- เพิ่ม vendor selector แบบ code คู่กับ name ในฟอร์มเอกสาร AP ที่ต้อง อ้างอิง vendor (Invoice เป็นตัวอย่างที่เห็นใน mockup)
- แสดง read-only vendor tax profile snapshot (Registered Name TH/EN, Tax ID 13 หลัก, Branch No. 5 หลัก, Registered Billing Address) เมื่อ เลือก vendor แล้ว โดยดึงจาก master data ที่มีอยู่เดิม ไม่สร้างใหม่

**Acceptance**
- ผู้ใช้เลือก vendor จาก dropdown ที่แสดงรูปแบบ code คู่กับ name ได้ตรงกับ mockup
- เมื่อเลือก vendor แล้ว เห็น vendor tax profile fields (Registered Name TH/EN, Tax ID, Branch No., Billing Address) เป็น read-only
- story นี้ไม่สร้างหน้า CRUD จัดการ vendor master ใหม่ (รอผลตัดสินจาก spike.ap-vendor-master-ownership)

**กฎที่ห้ามละเมิด**
- อ้าง AP/hotel_ap_erp_mockup_v4_4_3.html — option list ของ field 'Vendor *' (C003, CP001, LOT02, SINO04) และ h4 'Vendor Tax Profile' (Vendor Registered Name (TH/EN), Tax Registration ID (13 Digits), Branch No. (5 Digits), Registered Billing Address) — ยังไม่มี FRD ยืนยัน
- accounting-foundation.md §3 Shared Carmen data ระบุหลักการไม่สร้าง master data ซ้ำ แต่ Vendor ไม่อยู่ในรายการ shared data หรือ Accounting-owned data ที่ Foundation ระบุไว้เลยทั้งสองฝั่ง (§3, §4) — การเป็นเจ้าของ vendor master เต็มรูปแบบเป็น open question แยกไว้ที่ spike.ap-vendor-master-ownership story นี้ทำเฉพาะการเลือก/แสดง vendor ที่มีอยู่แล้วเท่านั้น

### CA-181 — [FE] AP Invoice — ฟอร์มหัวเอกสารและ action bar

- Status: **To Do** · Points: 3 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: CA-182
- Jira: https://carmensoftware.atlassian.net/browse/CA-181

**บริบท**
Mockup "New Invoice Entry" (AP/Invoice/hotel_ap_erp_Invoice_mockup_v4_4_8.html)
แสดงฟอร์มหัวเอกสาร Invoice พร้อม action bar ที่ยังไม่มี story ใดใน epic
accounts-payable ครอบคลุม (มีเพียง accounts-payable.design-doc และ
accounts-payable.backlog ซึ่งเป็น placeholder เอกสาร)
ปุ่ม "AI (OCR)" และ "Allocate Tax" ที่ปรากฏบนหน้าเดียวกันไม่รวมอยู่ใน
acceptance ของ story นี้ เพราะ element list มีแค่ชื่อปุ่มโดยไม่มี field
หรือพฤติกรรมยืนยันเลย (เขียน acceptance เกินนี้จะเป็นการแต่งเอง) — บันทึกไว้
เป็นช่องว่างใน ap-invoice.md แทน

**สิ่งที่ต้องทำ**
- สร้างฟอร์มหัวเอกสาร Invoice ตาม field ที่ mockup ระบุ — Doc No*, Input Date*, Vendor* (dropdown จาก vendor master ผ่าน FK), Currency, Exch Rate, Invoice No*, Invoice Date*, Credit, Due Date, Description
- สร้าง action bar ของเอกสาร — Draft, Save, Void, Copy, Print, Delete
- แสดงปุ่ม Attachment และ Log เป็นจุดเข้าถึงของ epic attachments/activity log ที่มีอยู่แล้ว (ไม่ implement ตรรกะแนบไฟล์/ประวัติเองในนี้)
- แสดงปุ่ม Approve/Reject/Send Back แยกจากปุ่ม Save/Void/Delete ของ document โดยไม่เพิ่มสถานะ approved/rejected เข้า document lifecycle ของ AP เอง
- แสดงปุ่ม AI (OCR) และ Allocate Tax เป็นปุ่มที่มีอยู่บน action bar เท่านั้น (ไม่ implement พฤติกรรมเบื้องหลัง เพราะ mockup ไม่ได้ยืนยัน field ใดๆ)

**Acceptance**
- ฟอร์มมี field ครบตามที่ mockup ระบุ และ field ที่มี * (Doc No, Input Date, Vendor, Invoice No, Invoice Date) ต้องกรอกก่อนบันทึกสำเร็จ
- Vendor เป็น dropdown เลือกจาก vendor master ที่มีอยู่แล้ว (ตัวอย่าง C003, CP001, LOT02, SINO04 ตาม mockup) หน้านี้ไม่มีปุ่มสร้าง vendor ใหม่
- ปุ่ม Draft/Save/Void/Copy/Print/Delete ปรากฏบน action bar ตามที่ mockup แสดง
- ปุ่ม Approve/Reject/Send Back ปรากฏเป็นกลุ่มแยกจากปุ่มจัดการเอกสาร และไม่มี field ใดในหน้าจอที่ตั้งชื่อสถานะเอกสารเป็น approved/rejected ตรงๆ
- ปุ่ม AI (OCR) และ Allocate Tax ปรากฏบนหน้าจอ โดยไม่ผูก logic เพิ่มเติมในนี้

**กฎที่ห้ามละเมิด**
- อ้าง AP/Invoice/hotel_ap_erp_Invoice_mockup_v4_4_8.html — ยังไม่มี FRD ยืนยัน
- accounting-foundation.md §8 Status model — ห้ามเพิ่ม approved/rejected เป็น document/JV lifecycle status; Approval/return/reject ต้องเป็น workflow action/history ที่แยกแกนจาก lifecycle status ของเอกสาร
- accounting-foundation.md §3 Shared Carmen data — Business Unit, Currency และ Vendor-related lookup เป็น master data กลาง ห้ามสร้างสำเนาใหม่ในหน้านี้
- ไม่แต่งพฤติกรรมของปุ่ม AI (OCR) และ Allocate Tax เพราะ mockup มีแค่ชื่อปุ่ม ไม่มี field หรือ flow ยืนยัน

### CA-182 — [BE] AP Invoice — บันทึกหัวเอกสารและออกเลขที่เอกสาร

- Status: **To Do** · Points: 5 · Labels: accounting, ap, be
- Blocked by: CA-181 · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-182

**บริบท**
ฟอร์มหัวเอกสาร (ap.invoice.header-fe) ต้องมี backend รองรับการบันทึก, การออก
เลขที่เอกสารและ snapshot currency/exchange rate ตาม field ที่ mockup ยืนยัน

**สิ่งที่ต้องทำ**
- persist หัวเอกสาร Invoice — doc_no, input_date, vendor_id (FK), currency_id, invoice_no, invoice_date, credit terms, due_date, description
- ออกเลขที่เอกสาร (Doc No) ผ่าน running-code engine เดิม ตาม accounting-foundation.md §3/§4 ไม่สร้าง numbering engine ใหม่
- เก็บ exchange-rate snapshot ตามชุด field ที่ §4 Exchange-rate guardrails กำหนด (transaction_currency, exchange_rate, rate_date, rate_type, rate_source, functional_currency) แทนการอ่าน current rate ย้อนหลัง
- รองรับสถานะ Draft และ Void ตามปุ่มที่ FE เรียก โดย Void ก่อน post ไม่สร้าง ledger ใดๆ

**Acceptance**
- บันทึกหัวเอกสารสำเร็จเมื่อ field required (*) ครบตามที่ FE ส่งมา
- Doc No ออกโดย running-code engine ไม่ซ้ำและ atomic ต่อ BU
- Currency และ Exch Rate ถูก snapshot ที่ตัวเอกสาร ไม่ผูกกับ current rate ของ master ที่เปลี่ยนภายหลัง
- Void เปลี่ยนสถานะเป็น cancelled/voided โดยไม่ลบข้อมูลและไม่สร้าง ledger

**กฎที่ห้ามละเมิด**
- อ้าง AP/Invoice/hotel_ap_erp_Invoice_mockup_v4_4_8.html — ยังไม่มี FRD ยืนยัน
- accounting-foundation.md §4 Exchange-rate guardrails — ต้องเก็บ snapshot currency/exchange rate ครบตามชุด field ที่กำหนด ห้ามคำนวณย้อนหลังด้วย current rate
- accounting-foundation.md §8 — void ก่อน post เปลี่ยนเอกสารเป็น cancelled/voided โดยไม่สร้าง ledger; หลัง post ห้ามลบหรือ void แบบ destructive ต้องสร้าง reversal journal
- accounting-foundation.md §3/§4 Running-code — reuse running-code engine เดิม ห้ามสร้าง numbering engine ใหม่ของ AP เอง

**ต้องตัดสินใน story นี้**
Vendor ที่เลือกในฟอร์มนี้อ้างจาก vendor master ชุดใด (ของ Accounting เอง
หรือ reuse จาก Procurement/Inventory) ขึ้นกับผลของ
spike.ap-vendor-master-ownership (เขียนโดย agent ap.main.*) ห้าม hardcode
แหล่งข้อมูล vendor ก่อน spike ตอบ

### CA-183 — [FE] AP Invoice — ตารางรายการ item และ dialog Edit Invoice Item Detail

- Status: **To Do** · Points: 5 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: CA-184
- Jira: https://carmensoftware.atlassian.net/browse/CA-183

**บริบท**
Mockup มี tab "Item Details" แสดงตารางรายการและ dialog "Edit Invoice Item
Detail" ที่ยังไม่มี story ใดครอบคลุม story นี้ตัด field ภาษี (Tax Profile 1
VAT, Tax Profile 2 WHT, Tax Acc Code/Cost Center, Tax Amount) และปุ่ม
Dimension ออก เพราะซ้ำกับขอบเขตของ story/spike อื่นที่มีอยู่แล้ว (ดู
ap-invoice.md หัวข้อ "ไม่แตกเพราะซ้ำ")

**สิ่งที่ต้องทำ**
- แสดงตาราง item ต่อบรรทัด — คอลัมน์ #, Description/Comment, Unit, Qty, Price/Unit, Sub total, Discount, Total/Unpaid, ACT. (คอลัมน์ Tax ใน ตารางเดียวกันแสดงผลจาก tax.fe.summary ของ epic tax-wht ไม่ใช่ของ story นี้)
- เปิด dialog 'Edit Invoice Item Detail' พร้อม field — Comment/Item Description*, Unit* (PCS/TRIP/SET/KG/BOX), Qty*, Price/Unit*, Dr Acc Code (Expense/Asset), Cost Center (Dept), Sub total Amt., Discount Profile, Disc Acc Code, Disc Cost Center, Disc Amt., Override, Cr Acc Code (Accounts Payable), Cr Cost Center, Total Amt.
- ปุ่ม Add/Cancel/Save Item Line ต่อบรรทัด

**Acceptance**
- ตารางและ dialog มี field ครบตามที่ mockup ระบุ ไม่รวม Tax Profile 1/2 และ ปุ่ม Dimension ในหน้าเดียวกัน
- Dr Acc Code/Cost Center และ Cr Acc Code/Cost Center เลือกจาก Chart of Accounts/Department master ผ่าน FK ไม่พิมพ์อิสระ
- field Override ปรากฏสำหรับแก้ยอดที่คำนวณอัตโนมัติตามที่ mockup มี

**กฎที่ห้ามละเมิด**
- อ้าง AP/Invoice/hotel_ap_erp_Invoice_mockup_v4_4_8.html — ยังไม่มี FRD ยืนยัน
- accounting-foundation.md §3 Shared Carmen data — Department เป็น master data กลาง ใช้ FK ไม่สร้างสำเนาใหม่
- ไม่รวม Tax Profile 1 (VAT) / Tax Profile 2 (WHT) ในบรรทัดนี้ — ซ้ำกับ tax.fe.dialog ของ epic tax-wht (ดูรายละเอียดใน ap-invoice.md)
- ไม่รวมปุ่ม Dimension / ALL DIMENSIONS ในบรรทัดนี้ — ซ้ำกับหลักฐานที่ spike.dimension-source เดิมใช้อยู่แล้ว (ดูรายละเอียดใน ap-invoice.md)

### CA-184 — [BE] AP Invoice — คำนวณและบันทึก Sub total/Discount/Total ต่อบรรทัด

- Status: **To Do** · Points: 5 · Labels: accounting, ap, be
- Blocked by: CA-183 · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-184

**บริบท**
Item lines ของ ap.invoice.line-items-fe ต้องมี backend คำนวณและเก็บยอดต่อ
บรรทัดเป็น decimal string ตาม decimal contract ของ Foundation story นี้ไม่
รวมยอดภาษี (tax amount) ซึ่งเป็นของ tax.be.posting ของ epic tax-wht

**สิ่งที่ต้องทำ**
- persist item line — description, unit, qty, price/unit, dr_account_id + cost_center, cr_account_id (Accounts Payable) + cost_center
- คำนวณ Sub total และ Discount ตาม Discount Profile/Disc Amt ที่ผู้ใช้กำหนด และปัดเป็น decimal string ตาม rounding mode HALF_UP ระดับบรรทัด
- รวม Total/Unpaid ต่อบรรทัดและยอดสุทธิ (Net Amount) ของเอกสาร ไม่รวม tax amount ซึ่งมาจาก tax.be.posting
- เตรียมข้อมูล account/cost center/amount ต่อบรรทัดให้พร้อมส่งเป็น lines[] เข้า posting engine ตาม contract (รายละเอียดการเรียกจริงอยู่ใน ap.invoice.post-via-posting-contract) ไม่เขียน ledger entry เอง

**Acceptance**
- Sub total, Discount และ Total/Unpaid เก็บเป็น decimal string ตาม currency.decimal_places ของเอกสาร ไม่ใช่ JSON number หรือ binary float
- ผลต่างจากการปัด (ถ้ามี) บันทึกเป็น explicit Rounding Adjustment แยก ไม่ปรับ ยอด line อื่นหรือซ่อน variance
- Dr Acc Code/Cost Center และ Cr Acc Code/Cost Center ต่อบรรทัดถูกเก็บพร้อม ส่งต่อเป็น lines[] แต่ยังไม่มีการเขียน ledger entry ใดๆ ในขั้นตอนนี้

**กฎที่ห้ามละเมิด**
- อ้าง AP/Invoice/hotel_ap_erp_Invoice_mockup_v4_4_8.html — ยังไม่มี FRD ยืนยัน
- accounting-foundation.md §4 Decimal and rounding contract — amount/rate เป็น decimal string ห้าม JSON number ห้าม binary float, ปัด HALF_UP ระดับ line, variance ต้องเป็น explicit Rounding Adjustment line
- accounting-foundation.md §9 Posting engine — ห้าม AP เขียน ledger entry โดยตรง ต้องเตรียมข้อมูลเป็น lines[] ให้ posting contract เท่านั้น

### CA-185 — [FE] AP Invoice — ตาราง Referenced Source Documents & Offset Liabilities

- Status: **To Do** · Points: 3 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: CA-186
- Jira: https://carmensoftware.atlassian.net/browse/CA-185

**บริบท**
Mockup มี tab "Doc Reference" แสดง section "Referenced Source Documents &
Offset Liabilities" สำหรับผูก Invoice กับเอกสารต้นทาง (PO/Deposit) — ตรงกับ
ขอบเขต "matching กับ PO/deposit" ที่ได้รับมอบหมาย ไม่รวมหน้าจอ "PO & Deposit
Requests" เอง (เป็นของ ap.main.po-deposit-requests)

**สิ่งที่ต้องทำ**
- แสดง section "Referenced Source Documents & Offset Liabilities" พร้อมปุ่ม "Add Document Ref"
- แสดงตารางคอลัมน์ตาม mockup — Ref Type, Ref Doc No., Ref Date, Net Amt (Trans/Base), Tax Amt (Trans/Base), Total Offset (Trans/Base), Remarks/ Adjust Reason
- เปิดให้เลือกเอกสารอ้างอิงจากรายการที่มีอยู่แล้ว (ไม่สร้างหน้าจอเลือก เอกสารเอง)

**Acceptance**
- ปุ่ม Add Document Ref เปิดให้เลือกเอกสารอ้างอิงได้
- ตารางแสดงคอลัมน์ครบตามที่ mockup ระบุ แยกค่า transaction currency กับ base currency ให้เห็นชัดเจน
- แก้ไข Remarks/Adjust Reason ได้ตาม field ที่ mockup มี

**กฎที่ห้ามละเมิด**
- อ้าง AP/Invoice/hotel_ap_erp_Invoice_mockup_v4_4_8.html — ยังไม่มี FRD ยืนยัน
- accounting-foundation.md §4 Decimal and rounding contract — Net/Tax/Total Offset ที่แสดงต้องเป็น decimal string ห้าม JSON number
- ไม่รวมหน้าจอ 'PO & Deposit Requests' เอง — อยู่นอกขอบเขตของ agent นี้ (เป็นของ ap.main.po-deposit-requests)

### CA-186 — [BE] AP Invoice — จับคู่และหักออฟเซ็ต PO/Deposit กับยอด Invoice

- Status: **To Do** · Points: 5 · Labels: accounting, ap, be
- Blocked by: CA-185 · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-186

**บริบท**
ตาราง Referenced Source Documents (ap.invoice.doc-ref-fe) ต้องมี backend
ผูกเอกสารอ้างอิงและคำนวณยอดออฟเซ็ตต่อรายการ

**สิ่งที่ต้องทำ**
- ผูก Invoice กับเอกสารอ้างอิง (PO, Deposit) ผ่าน FK พร้อมเก็บ Ref Type, Ref Doc No., Ref Date
- คำนวณ Net Amt, Tax Amt และ Total Offset แยกเป็น transaction currency และ base currency ต่อรายการอ้างอิง
- ปรับยอด unpaid/net ของ invoice ตาม total offset ที่ผูกไว้

**Acceptance**
- เอกสารอ้างอิงที่ผูกแล้วลด unpaid amount ของ invoice ตามยอด offset จริง
- Net Amt, Tax Amt และ Total Offset เก็บเป็น decimal string ทั้งสองสกุล (transaction และ base)
- ลบการอ้างอิงที่มีผลต่อยอดของ invoice ที่ post แล้วไม่ได้ ต้องแก้ผ่าน reversal/adjusting journal เท่านั้น

**กฎที่ห้ามละเมิด**
- อ้าง AP/Invoice/hotel_ap_erp_Invoice_mockup_v4_4_8.html — ยังไม่มี FRD ยืนยัน
- accounting-foundation.md §4 Decimal and rounding contract — amount/rate เป็น decimal string ห้าม JSON number ห้าม binary float
- accounting-foundation.md §7 ข้อ 8 — Posted journal และ ledger entry เป็น immutable แก้ผ่าน reversal/adjusting journal เท่านั้น

### CA-187 — [BE] AP Invoice — Post Invoice ต้องยิงผ่าน GL posting contract ไม่เขียน ledger เอง

- Status: **To Do** · Points: 8 · Labels: accounting, ap, be
- Blocked by: - · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-187

**บริบท**
Mockup มี section "JOURNAL ENTRY DETAILS (AUTOMATED POSTING PREVIEW)"
ตารางคอลัมน์ #, COST CENTER, ACCOUNT CODE, COMMENT, CUR., RATE, DEBIT, CREDIT
พร้อมปุ่ม "Add Row (Alt+A)" ชื่อ panel เองบอกว่าเป็น preview ของสิ่งที่จะ
post ไม่ใช่หน้าให้พิมพ์ ledger เอง
README.md และ accounting-foundation.md §9 กำหนดชัดว่า subledger อย่าง AP
ต้องเรียก posting engine ผ่าน contract (source_type, source_id,
source_version, posting_rule_code, posting_date, lines[], idempotency_key)
ห้ามเขียน ledger entry ตรง — เอกสารนี้เป็น canonical เหนือ mockup ในเรื่อง
posting path
ส่วนปุ่ม "Add Row (Alt+A)" ที่ทำให้ preview นี้ดูแก้ไขได้อิสระเป็นคำถามเปิด
แยกไว้ที่ spike.ap-invoice-jv-preview-editability ไม่ปนกับ story นี้ —
story นี้ครอบคลุมเฉพาะเส้นทาง posting ปกติที่ generate จาก invoice lines

**สิ่งที่ต้องทำ**
- ประกอบ lines[] ของ posting contract จาก invoice item lines (Dr Acc Code/Cost Center ต่อบรรทัด) และ Cr Acc Code (Accounts Payable)/Cr Cost Center ของหัวเอกสาร ให้ตรงกับที่แสดงใน Journal Entry preview
- เรียก posting engine ผ่าน contract เดียวกับที่ §9 ระบุ พร้อม idempotency key ต่อการ submit/post invoice หนึ่งครั้ง ห้าม AP เขียนตาราง ledger เอง
- ปฏิเสธการ post ถ้า preview ไม่สมดุล (debit รวม ≠ credit รวม) ก่อนส่งเข้า posting engine
- retry ด้วย idempotency key เดิมต้องไม่สร้าง ledger ซ้ำ

**Acceptance**
- Given invoice มี item lines ครบและผ่าน validation, when เปิด panel Journal Entry Details, then preview แสดงบรรทัดตามคอลัมน์ #, Cost Center, Account Code, Comment, Cur., Rate, Debit, Credit ที่สมดุลกัน
- Given preview สมดุล, when submit/post invoice, then ระบบเรียก posting engine ผ่าน contract (source_type/source_id/source_version/ posting_rule_code/posting_date/lines/idempotency_key) ไม่เขียนตาราง ledger ของ AP เอง
- Given preview ไม่สมดุล, when พยายาม post, then ระบบปฏิเสธก่อนเรียก posting engine
- Given post ซ้ำด้วย idempotency key เดิม (retry หลัง timeout), then ไม่ เกิด ledger entry ซ้ำ

**กฎที่ห้ามละเมิด**
- อ้าง AP/Invoice/hotel_ap_erp_Invoice_mockup_v4_4_8.html — h4 JOURNAL ENTRY DETAILS (AUTOMATED POSTING PREVIEW), th COST CENTER/ACCOUNT CODE/COMMENT/CUR./RATE/DEBIT/CREDIT — ยังไม่มี FRD ยืนยัน
- README.md — AP ต้องส่งรายการเข้า GL ผ่าน posting contract เดียวกัน ห้ามเขียน journal tables โดยตรง
- accounting-foundation.md §9 Posting engine และ §7 ข้อ 9-10 — database transaction เดียว, idempotency key ทุกคำสั่ง post

**ต้องตัดสินใน story นี้**
ปุ่ม "Add Row (Alt+A)" ใน preview นี้อนุญาตให้ผู้ใช้เพิ่ม/แก้บรรทัด debit-
credit เองก่อน submit หรือเป็น preview ที่ generate จาก invoice lines
เท่านั้นโดยปุ่มนี้ไม่มีผลจริง ขึ้นกับผลของ
spike.ap-invoice-jv-preview-editability ห้ามเปิดใช้ปุ่มนี้จนกว่า spike
จะตอบ

### CA-188 — [BE] AP Payment — เลือก Invoice/Credit Note ค้างชำระมาตัดจ่ายเข้า Settlement ของ Payment Voucher

- Status: **To Do** · Points: 5 · Labels: accounting, ap, be
- Blocked by: CA-189 · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-188

**บริบท**
Mockup "AP/Payment Approval" (carmen_cloud_erp_ap_payment_module_v2_16.html) มีปุ่ม
"Select Documents" ใต้หัวข้อ "SETTLED INVOICES & CREDIT NOTES" ที่เปิด modal
"Select Outstanding Invoices & Credit Notes" (ตัวกรอง All Types (INV + CN) /
Invoices Only (INV) / Credit Notes Only (CN), ค้นหาโดย Invoice No / Tax Invoice
No / Description, สลับมุมมอง Summary View / Item-Level Breakdown, ตาราง Invoice
Date, Due Date, Inv. Total, Balance Due, ปุ่ม Apply to Batch) แล้วรายการที่เลือก
จะไปลงในตาราง "SETTLED INVOICES & CREDIT NOTES" ของ Payment Voucher เอง
(คอลัมน์ Doc No. / Tax Inv, Date, Orig. FX, Net Amount, VAT, Total Inv.,
Line / Item Ref, Department & Account, Total Amount)
นี่คือ UI mockup ไม่ใช่ FRD — ไม่มีข้อความยืนยันสูตรคำนวณ Balance Due หรือกฎการ
ล็อก invoice ระหว่างตัดจ่าย จึงเขียนเฉพาะสิ่งที่โครงหน้าจอยืนยันได้จริง คือ
"มีเอกสารค้างชำระให้เลือก และเอกสารที่เลือกจะถูกนำไปสร้างเป็นบรรทัด settlement
ของ payment นี้"

**สิ่งที่ต้องทำ**
- เปิด endpoint คืนรายการ invoice/credit note ค้างชำระ (Balance Due > 0) ของ vendor เดียวกับ payment voucher พร้อม field ตามคอลัมน์ modal (Invoice Date, Due Date, Inv. Total, Balance Due) และรองรับ filter ประเภทเอกสาร (All/Invoices Only/Credit Notes Only) กับค้นหาแบบ Invoice No / Tax Invoice No / Description
- เมื่อ "Apply to Batch" ให้สร้าง settlement line ผูกกับ payment voucher ต่อ เอกสารที่เลือก พร้อม snapshot Net Amount, VAT, Total Inv., Orig. FX ณ เวลา ที่เลือก (ไม่ query invoice ซ้ำตอน post)
- ลด Balance Due ที่เหลือของ invoice/credit note ต้นทางตามยอดที่ถูกตัดจ่ายใน payment voucher นี้ (partial settlement ต้องยังเลือกไปตัดจ่ายใบถัดไปได้)
- เก็บ amount/rate ทุกฟิลด์เป็น decimal string ตาม decimal contract ห้าม JSON number หรือ binary float

**Acceptance**
- Given invoice/credit note มี Balance Due > 0 ของ vendor เดียวกับ payment voucher, when เรียก endpoint ค้นหาเอกสารค้างชำระ, then ได้ผลลัพธ์เอกสารนั้นพร้อม Invoice Date, Due Date, Inv. Total, Balance Due
- Given เลือกเอกสารแล้ว Apply to Batch, when settlement line ถูกสร้าง, then ยอด Net Amount/VAT/Total Inv./Orig. FX ที่แสดงในตาราง SETTLED INVOICES & CREDIT NOTES ตรงกับค่า ณ เวลาที่เลือก
- Given payment voucher ตัดจ่าย invoice บางส่วน, when บันทึก settlement line, then Balance Due ของ invoice ต้นทางลดลงตามยอดที่ตัดจ่ายและยังเลือกไปตัดจ่าย payment voucher ใบอื่นได้อีกถ้ายังเหลือ Balance Due
- amount/rate ทุกฟิลด์ที่ตอบกลับเป็น decimal string ไม่ใช่ JSON number

**กฎที่ห้ามละเมิด**
- อ้าง carmen_cloud_erp_ap_payment_module_v2_16.html — ปุ่ม Select Documents, modal Select Outstanding Invoices & Credit Notes (All Types/Invoices Only/Credit Notes Only, Invoice Date, Due Date, Inv. Total, Balance Due, Apply to Batch) และตาราง SETTLED INVOICES & CREDIT NOTES — ยังไม่มี FRD ยืนยัน
- accounting-foundation.md §4 Decimal and rounding contract — amount/rate เป็น decimal string ห้าม JSON number ห้าม binary float
- ห้ามแต่งสูตรคำนวณ Balance Due หรือกฎ lock เอกสารระหว่างตัดจ่ายที่ mockup ไม่ได้ยืนยัน

### CA-189 — [FE] AP Payment — modal เลือก Invoice/Credit Note ค้างชำระมา Apply to Batch

- Status: **To Do** · Points: 3 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: CA-188
- Jira: https://carmensoftware.atlassian.net/browse/CA-189

**บริบท**
ต่อยอดจาก ap.payment.apply-settlement-documents — mockup มี modal "Select
Outstanding Invoices & Credit Notes" ที่มีช่องค้นหา (ตัวเลือก Invoice No /
Tax Invoice No / Description), ตัวกรองประเภทเอกสาร (All Types (INV + CN) /
Invoices Only (INV) / Credit Notes Only (CN)), ปุ่มสลับมุมมอง "Summary View" /
"Item-Level Breakdown", ตารางคอลัมน์ Invoice Date, Due Date, Inv. Total,
Balance Due และปุ่ม Cancel / Apply to Batch

**สิ่งที่ต้องทำ**
- สร้าง modal พร้อมช่องค้นหาที่สลับ field ค้นหาได้ตามตัวเลือก (Invoice No / Tax Invoice No / Description) และตัวกรองประเภทเอกสาร 3 ค่า
- เพิ่ม toggle "Summary View" / "Item-Level Breakdown" สลับรูปแบบตาราง (ยังไม่ ทราบรายละเอียดความต่างของสองมุมมองจาก mockup — ทำ toggle state ไว้ก่อน รอ backend/FRD ยืนยัน field ที่ต่างกัน)
- เลือกได้หลายแถว แล้วกด Apply to Batch เรียก endpoint ของ ap.payment.apply-settlement-documents; Cancel ปิด modal โดยไม่มีผล

**Acceptance**
- Given เปิด modal, when พิมพ์คำค้นในช่องที่เลือกประเภท Invoice No / Tax Invoice No / Description, then ตารางกรองเหลือเฉพาะเอกสารที่ตรงเงื่อนไข
- Given เลือกตัวกรอง Invoices Only (INV) หรือ Credit Notes Only (CN), then ตารางแสดงเฉพาะประเภทที่เลือก
- Given เลือกแถวอย่างน้อยหนึ่งแถวแล้วกด Apply to Batch, then modal ปิดและ payment voucher มี settlement line ใหม่ตามที่เลือก
- Given กด Cancel, then modal ปิดโดยไม่มี settlement line ใหม่เกิดขึ้น

**กฎที่ห้ามละเมิด**
- อ้าง carmen_cloud_erp_ap_payment_module_v2_16.html — h3 Select Outstanding Invoices & Credit Notes, ตัวเลือกค้นหา Invoice No/Tax Invoice No/Description, filter All Types/Invoices Only/Credit Notes Only, ปุ่ม Summary View/Item-Level Breakdown, th Invoice Date/Due Date/Inv. Total/Balance Due, ปุ่ม Cancel/Apply to Batch — ยังไม่มี FRD ยืนยัน
- ไม่แต่งความต่างของ Summary View กับ Item-Level Breakdown เพราะ mockup ให้แค่ชื่อปุ่ม ไม่มีรายละเอียด field ที่ต่างกัน

### CA-190 — [BE] AP Payment — บันทึกวิธีจ่ายเงินหลายรายการ (Multi-Bank Allocation) ต่อหนึ่ง Payment Voucher

- Status: **To Do** · Points: 5 · Labels: accounting, ap, be
- Blocked by: CA-191 · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-190

**บริบท**
Mockup มีหัวข้อ "Payment Methods & Multi-Bank Allocation" กับปุ่ม "Add
Payment" ตารางคอลัมน์ Payment Method & Bank, Pay to (Payee), Ref. No. /
Cheque No., Account Code, Cost Center, Dimensions และ modal "Add Payment
Method" ที่มีฟิลด์ Payment Type (ระบุว่าเป็น "Master Table" พร้อมตัวอย่าง
BBL : Bank Transfer Direct API, SCB : Corporate Cheque, KBANK : Kasikorn
Direct), Pay to (Payee Name), Ref. No. / Cheque No., Pay Amt. และปุ่ม
"Add to Method List"
mockup ระบุ Payment Type ว่าเป็น "(Master Table)" เอง จึงถือเป็น shared
master data ที่ AP อ้างผ่าน FK ไม่ใช่ AP ไปสร้างตาราง lookup ของตัวเองหรือ
hardcode ตัวเลือกไว้ในโค้ด (BBL/SCB/KBANK เป็นตัวอย่างข้อมูลในตาราง master
ไม่ใช่ enum ตายตัว)

**สิ่งที่ต้องทำ**
- รองรับเพิ่มได้หลาย payment method line ต่อ payment voucher หนึ่งใบ (multi-bank/multi-method) แต่ละ line เก็บ payment_type (FK ไปยัง Payment Type master table), payee name, ref./cheque no., account code, cost center, dimensions, pay amount
- โหลดตัวเลือก Payment Type จาก master table ที่มีอยู่แล้ว ห้าม hardcode รายการ BBL/SCB/KBANK ไว้ในโค้ด backend ของ AP
- เก็บ pay amount เป็น decimal string ตาม decimal contract

**Acceptance**
- Given payment voucher หนึ่งใบ, when เพิ่ม payment method มากกว่าหนึ่งรายการ, then ทุกรายการถูกบันทึกแยกเป็น line พร้อม field ครบตามคอลัมน์ Payment Method & Bank/Pay to/Ref. No./Account Code/Cost Center/Dimensions
- Given เรียกตัวเลือก Payment Type, when แสดงในฟอร์ม Add Payment Method, then รายการมาจาก Payment Type master table ไม่ใช่ค่าคงที่ในโค้ด AP
- pay amount ทุก line เป็น decimal string

**กฎที่ห้ามละเมิด**
- อ้าง carmen_cloud_erp_ap_payment_module_v2_16.html — h3 Payment Methods & Multi-Bank Allocation, ปุ่ม Add Payment, th Payment Method & Bank/Pay to (Payee)/Ref. No./Cheque No./Account Code/Cost Center/Dimensions, modal Add Payment Method (label Payment Type (Master Table), option BBL/SCB/KBANK, Pay to (Payee Name), Ref. No./Cheque No., Pay Amt.) — ยังไม่มี FRD ยืนยัน
- accounting-foundation.md §3 Shared Carmen data — master data ที่ reuse ผ่าน FK ไม่ใช่ของ Accounting เป็นเจ้าของ; mockup เองระบุ Payment Type ว่าเป็น 'Master Table' จึงยึดหลักเดียวกัน
- accounting-foundation.md §4 Decimal and rounding contract — amount เป็น decimal string
- ไม่ยืนยันกฎว่าผลรวม Pay Amt. ของทุก method ต้องเท่ากับยอดที่ต้องจ่ายพอดี เพราะ mockup ไม่มีคอลัมน์ยอดรวม/คงเหลือให้เห็น — ทิ้งเป็นจุดเปิดใน backlog ถ้ามี FRD ยืนยันภายหลัง

### CA-191 — [FE] AP Payment — หน้าจอ Add Payment Method และตาราง Multi-Bank Allocation

- Status: **To Do** · Points: 3 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: CA-190
- Jira: https://carmensoftware.atlassian.net/browse/CA-191

**บริบท**
ต่อยอดจาก ap.payment.multi-bank-payment-methods — mockup มี modal "Add
Payment Method" (dropdown Payment Type (Master Table) พร้อมตัวอย่าง
BBL/SCB/KBANK, ช่อง Pay to (Payee Name), Ref. No. / Cheque No., Pay Amt.,
ปุ่ม Add to Method List) และตาราง "Payment Methods & Multi-Bank Allocation"
ที่แสดงรายการที่เพิ่มแล้ว

**สิ่งที่ต้องทำ**
- สร้าง modal Add Payment Method ตามฟิลด์ที่ mockup ระบุ โดยดึงตัวเลือก Payment Type จาก master table ผ่าน endpoint ของ ap.payment.multi-bank-payment-methods ไม่ hardcode รายการในหน้าจอ
- กด "Add to Method List" แล้วแถวใหม่ปรากฏในตาราง Payment Methods & Multi-Bank Allocation ตามคอลัมน์ Payment Method & Bank/Pay to/Ref. No./ Account Code/Cost Center/Dimensions

**Acceptance**
- Given เปิด modal Add Payment Method, when โหลดตัวเลือก Payment Type, then รายการที่แสดงมาจาก master table ไม่ใช่ตัวเลือกตายตัวในโค้ด FE
- Given กรอกฟอร์มครบแล้วกด Add to Method List, then ตาราง Multi-Bank Allocation มีแถวใหม่ตามข้อมูลที่กรอก

**กฎที่ห้ามละเมิด**
- อ้าง carmen_cloud_erp_ap_payment_module_v2_16.html — h3 Add Payment Method, label Payment Type (Master Table), option BBL/SCB/KBANK, label Pay to (Payee Name)/Ref. No./Cheque No./Pay Amt., ปุ่ม Add to Method List, ตาราง Payment Methods & Multi-Bank Allocation — ยังไม่มี FRD ยืนยัน

### CA-192 — [BE] AP Payment — เพิ่มรายการ Other Expense/Miscellaneous Surcharge เข้า Payment Voucher

- Status: **To Do** · Points: 2 · Labels: accounting, ap, be
- Blocked by: - · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-192

**บริบท**
Mockup มีหัวข้อ "Other Expense & Miscellaneous Surcharges" ปุ่ม "Add
Expense" และ modal "Add Other Expense" ที่มีฟิลด์ Description / Memo *
(บังคับกรอก — มี * ใน mockup), Account Code (ตัวอย่าง option 5041005
Freight & Handling, 5041002 Bank Charges Expense, 5041003 Swift Transfer
Fee, 5041009 Miscellaneous AP Expense), Cost Center (ตัวอย่าง 801/800/201),
Dimension 1 (Project/Event), Dimension 2 (Brand/Segment), Base Amt. (THB) *
(บังคับกรอก) และปุ่ม Save Expense

**สิ่งที่ต้องทำ**
- เปิด endpoint เพิ่ม/แก้ไข/ลบรายการ other expense ต่อ payment voucher หนึ่ง ใบ เก็บ description, account_code (FK ไป Chart of Accounts), cost_center, dimension1, dimension2, base_amount
- บังคับกรอก description และ base_amount ตามที่ mockup ทำเครื่องหมาย * ไว้
- นำ line เหล่านี้ไปรวมในชุดข้อมูลที่ใช้สร้าง Journal Entry preview (เป็น บรรทัด debit เพิ่มเติม)
- เก็บ base_amount เป็น decimal string

**Acceptance**
- Given กรอก Description และ Base Amt. ครบ, when Save Expense, then บันทึก other expense line สำเร็จพร้อม account code/cost center/dimension ที่เลือก
- Given ไม่กรอก Description หรือ Base Amt., when พยายาม Save Expense, then ระบบปฏิเสธเพราะเป็นฟิลด์บังคับ
- other expense line ที่บันทึกแล้วปรากฏในชุดข้อมูล Journal Entry preview ของ payment voucher เดียวกัน

**กฎที่ห้ามละเมิด**
- อ้าง carmen_cloud_erp_ap_payment_module_v2_16.html — h3 Other Expense & Miscellaneous Surcharges, ปุ่ม Add Expense, modal Add Other Expense (label Description / Memo *, Account Code พร้อม option 5041005/5041002/5041003/5041009, Cost Center option 801/800/201, Dimension 1/2, Base Amt. (THB) *, ปุ่ม Save Expense) — ยังไม่มี FRD ยืนยัน
- accounting-foundation.md §4 Decimal and rounding contract — amount เป็น decimal string
- ฟิลด์บังคับยึดตามเครื่องหมาย * ที่ mockup แสดงจริงเท่านั้น (Description, Base Amt.) ไม่ได้แต่งเพิ่ม

### CA-193 — [BE] AP Payment — Post Payment ต้องยิงผ่าน GL posting contract ไม่เขียน ledger เอง

- Status: **To Do** · Points: 8 · Labels: accounting, ap, be
- Blocked by: - · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-193

**บริบท**
Mockup มี tab "3. Journal GL" ที่แสดง "JOURNAL ENTRY DETAILS (AUTOMATED
POSTING PREVIEW)" ตารางคอลัมน์ #, COST CENTER, ACCOUNT CODE, COMMENT,
CUR., RATE, DEBIT, CREDIT พร้อมปุ่ม "Print JV" และปุ่ม "Post Payment" อยู่
ท้ายหน้า (ชื่อ preview เองบอกว่าเป็น "AUTOMATED POSTING PREVIEW" คือคำนวณ
preview ให้ก่อน ไม่ใช่ให้ผู้ใช้พิมพ์บรรทัดบัญชีเอง)
README.md และ accounting-foundation.md §9 กำหนดชัดว่า subledger อย่าง AP
ต้องเรียก posting engine ผ่าน contract (source_type, source_id,
source_version, posting_rule_code, posting_date, lines[], idempotency_key)
ห้ามเขียน ledger entry ตรง — เอกสารนี้เป็น canonical เหนือ mockup ในเรื่อง
posting path เพราะ mockup ไม่ได้แสดงรายละเอียด backend contract แต่ก็ไม่ได้
ขัดกับหลักการนี้เช่นกัน (mockup แสดงแค่ผลลัพธ์ preview ของสิ่งที่จะ post)

**สิ่งที่ต้องทำ**
- ประกอบ lines[] ของ posting contract จาก settlement lines (invoice/credit note ที่ตัดจ่าย), payment method lines, และ other expense lines ของ payment voucher เดียวกัน ให้ตรงกับที่แสดงใน Journal Entry preview (Cost Center, Account Code, Comment, Currency, Rate, Debit, Credit)
- เรียก posting engine ผ่าน contract เดียวกับที่ accounting-foundation.md §9 ระบุ พร้อม idempotency key ต่อการกด Post Payment หนึ่งครั้ง ห้าม AP เขียนตาราง ledger เอง
- ปฏิเสธการ post ถ้า preview ไม่สมดุล (debit รวม ≠ credit รวม) ก่อนส่งเข้า posting engine
- retry ด้วย idempotency key เดิมต้องไม่สร้าง ledger ซ้ำ

**Acceptance**
- Given payment voucher มี settlement/payment method/expense lines ครบ, when เปิด tab Journal GL, then preview แสดงบรรทัดตามคอลัมน์ #, Cost Center, Account Code, Comment, Cur., Rate, Debit, Credit ที่สมดุลกัน (debit รวม = credit รวม)
- Given preview สมดุล, when กด Post Payment, then ระบบเรียก posting engine ผ่าน contract (source_type/source_id/source_version/posting_rule_code/posting_date/lines/idempotency_key) ไม่เขียนตาราง ledger ของ AP เอง
- Given preview ไม่สมดุล, when กด Post Payment, then ระบบปฏิเสธก่อนเรียก posting engine
- Given กด Post Payment ซ้ำด้วย idempotency key เดิม (เช่น retry หลัง timeout), then ไม่เกิด ledger entry ซ้ำ

**กฎที่ห้ามละเมิด**
- อ้าง carmen_cloud_erp_ap_payment_module_v2_16.html — button 3. Journal GL, h3 JOURNAL ENTRY DETAILS (AUTOMATED POSTING PREVIEW), th #/COST CENTER/ACCOUNT CODE/COMMENT/CUR./RATE/DEBIT/CREDIT, button Print JV, button Post Payment — ยังไม่มี FRD ยืนยัน
- README.md — AP ต้องส่งรายการเข้า GL ผ่าน posting contract เดียวกัน ห้ามเขียน journal tables โดยตรง
- accounting-foundation.md §9 Posting engine — ลำดับตรวจ/lock/สร้าง ledger และ §7 ข้อ 9-10 (database transaction เดียว, idempotency key ทุกคำสั่ง post)

### CA-194 — [BE] AP Payment — Void Payment Voucher ทำได้เฉพาะก่อน Post เท่านั้น

- Status: **To Do** · Points: 2 · Labels: accounting, ap, be
- Blocked by: - · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-194

**บริบท**
Mockup มีปุ่ม "Void" อยู่แถวเดียวกับ "Save Draft" บนหน้า payment voucher
detail และ modal "Void Payment Voucher" กับปุ่ม "Confirm Void" ส่วนตัวกรอง
สถานะของหน้า list มีค่า Draft, In Progress, Posted, Voided, Rejected แยก
จากกัน (Voided เป็นคนละสถานะกับ Posted) mockup ไม่ได้ระบุตรง ๆ ว่า Void ใช้
กับสถานะ Posted ได้หรือไม่ — จุดนี้ยึดตาม accounting-foundation.md ซึ่งเป็น
canonical กว่า UI reference
accounting-foundation.md §7 ข้อ 8 และ §8 ระบุว่า posted journal เป็น
immutable ห้าม destructive void/delete หลัง post ต้องแก้ผ่าน reversal
เท่านั้น ส่วน void ก่อน post เปลี่ยนเป็น cancelled/voided โดยไม่สร้าง ledger
ได้

**สิ่งที่ต้องทำ**
- อนุญาต Void เฉพาะ payment voucher ที่ยังไม่ post (Draft/In Progress) เปลี่ยนสถานะเป็น Voided โดยไม่สร้าง ledger entry ใด ๆ
- ปฏิเสธ Void เมื่อ payment voucher มีสถานะ Posted แล้ว พร้อมข้อความแนะนำให้ ใช้ reversal แทน

**Acceptance**
- Given payment voucher สถานะ Draft หรือ In Progress, when Confirm Void, then สถานะเปลี่ยนเป็น Voided และไม่มี ledger entry ถูกสร้าง
- Given payment voucher สถานะ Posted, when พยายาม Void, then ระบบปฏิเสธ (ต้องแก้ผ่าน reversal เท่านั้น)

**กฎที่ห้ามละเมิด**
- อ้าง carmen_cloud_erp_ap_payment_module_v2_16.html — button Void, h3 Void Payment Voucher, button Confirm Void, option สถานะ Draft/In Progress/Posted/Voided/Rejected — ยังไม่มี FRD ยืนยัน
- accounting-foundation.md §7 ข้อ 8 และ §8 — posted เป็น immutable ห้าม destructive void หลัง post ต้องใช้ reversal; void ก่อน post ไม่สร้าง ledger
- mockup ไม่ได้ยืนยันตรง ๆ ว่าปุ่ม Void ถูก disable ตอน Posted หรือไม่ — กฎ 'ห้าม void หลัง post' มาจาก Foundation ไม่ใช่จาก mockup

### CA-195 — [BE] AP Dashboard — query/aggregation endpoint ของเอกสาร AP ค้างชำระ (feed ให้ AP Aging Analysis และ Due Date Tracker)

- Status: **To Do** · Points: 5 · Labels: accounting, ap, be
- Blocked by: CA-196 · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-195

**บริบท**
hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 19-23 มี h2 "Hotel AP Dashboard"
ตามด้วย h3 สอง stat tile ตัวเลขสกุลบาท (฿405,280.00, ฿0.00) แล้วต่อด้วย h3 "AP Aging
Analysis" และ "Due Date Tracker" ติดกันใน DOM ส่วน hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html
บรรทัด 77-81 มีชุด h3 เดียวกันซ้ำทุกตัวอักษร (แสดงว่าเป็น component เดียวกันที่ reuse ข้ามหน้า)
การ extract เป็น text ไม่ได้เก็บ label กำกับตัวเลขสอง stat tile นั้น — จึง **ไม่ยืนยัน**
ว่าคือ "ยอดรวมค้างชำระ" กับ "ยอดเลยกำหนด" ตามที่มักพบใน AP dashboard ทั่วไป เป็นการอนุมาน
จากตำแหน่งเท่านั้น ผมจึงออกแบบ endpoint ให้คืน raw data (list + sum) แทนการ fix ชื่อ metric
ตาราง "Accounts Payable Documents" ที่มีอยู่แล้วทั้งสอง mockup (บรรทัด 29-37 ของไฟล์ Aging
และ 92-100 ของไฟล์ Other) มีคอลัมน์ Doc Type, Doc No, Doc Date, Vendor Invoice No,
Vendor/Supplier, Due Date, Net Amount, Status อยู่แล้ว จึงเป็นแหล่งข้อมูลจริงของ endpoint นี้
ไม่ใช่การสร้าง entity ใหม่
งานนี้อยู่ในขอบเขต Accounting-owned data ตาม accounting-foundation.md §3 (posting
batch/ledger entries, source-to-journal posting mapping) เพราะเป็นการ query จาก AP
document ที่ผูกกับ ledger ของ Accounting เอง ไม่ใช่การสร้าง master data ใหม่ซ้อนทับ
Business Unit, Currency เป็น shared master data ตาม §3 เช่นกัน — endpoint นี้ใช้ BU
context ที่มีอยู่แล้ว ไม่สร้าง BU selector ใหม่ (selector อยู่ในขอบเขตของ ap-main)

**สิ่งที่ต้องทำ**
- สร้าง read-only endpoint คืนรายการ AP document ที่ยังไม่ closed ของ BU ที่เลือก พร้อม doc_no, doc_date, vendor, due_date, net_amount, status — ใช้ชุดคอลัมน์เดียวกับ ตาราง Accounts Payable Documents ที่มีอยู่แล้ว
- คืนยอดรวม (sum) ของ net_amount ทั้งหมดที่ query ได้ เพื่อใช้แสดงเป็นตัวเลขสรุปด้านบน ของแดชบอร์ด — ไม่ fix ชื่อ/นิยาม metric ของ stat tile ทั้งสองก้อนเพราะ mockup ไม่ยืนยัน
- ห้ามคำนวณหรือ hardcode aging bucket (เช่น 0-30/31-60/61-90/90+) เพราะ mockup ไม่มี ตัวเลข bucket ใด ๆ ยืนยัน — คืนเฉพาะ due_date ดิบให้ผู้บริโภคข้อมูลจัดกลุ่มเองภายหลัง เมื่อกฎธุรกิจถูกยืนยันจาก FRD
- amount ทุกค่าคืนเป็น decimal string ตาม Decimal and rounding contract ห้าม JSON number
- filter ตาม BU ที่ผู้ใช้เลือกอยู่ (BU context มาจากบริบทของหน้า ไม่ใช่ story นี้สร้าง selector เอง)

**Acceptance**
- Given มี AP document ที่ยังไม่ closed ของ BU ที่เลือก, when เรียก endpoint, then ได้ list ของเอกสารพร้อม due_date, net_amount (decimal string) และยอดรวมทั้งหมด
- Given เปลี่ยน BU, when เรียกซ้ำด้วย BU code อื่น, then ได้เฉพาะเอกสารของ BU นั้นเท่านั้น
- Given ไม่มี AP document ค้างอยู่เลยของ BU นั้น, when เรียก endpoint, then ได้ list ว่าง และยอดรวมเป็น "0.00"
- endpoint เป็น read-only ไม่มี side effect ต่อสถานะเอกสารหรือ ledger ใด ๆ

**กฎที่ห้ามละเมิด**
- อ้าง hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 19-23 (h2 Hotel AP Dashboard, สอง stat tile ฿405,280.00/฿0.00, h3 AP Aging Analysis, h3 Due Date Tracker) และ hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 77-81 (ชุด h3 เดียวกันซ้ำ) กับบรรทัด 92-100 (ตาราง Accounts Payable Documents) — ยังไม่มี FRD ยืนยัน label ของ stat tile หรือสูตร aging bucket
- อ้าง accounting-foundation.md §3 Accounting-owned data (posting batch/ledger entries, source-to-journal posting mapping) และ Decimal and rounding contract — amount ต้องเป็น decimal string ห้าม JSON number

**ต้องตัดสินใน story นี้**
นิยามจริงของ stat tile สองก้อน (฿405,280.00 / ฿0.00) และเกณฑ์ aging bucket
(เช่น 0-30/31-60/61-90/90+) ยังไม่มีใน mockup ต้องยืนยันจาก FRD หรือทีม design ก่อน
implement การจัดกลุ่มจริง — endpoint นี้คืนแค่ raw list + sum พอในระหว่างที่ยังไม่ตัดสิน

### CA-196 — [FE] AP Dashboard — แสดง stat tile และ panel AP Aging Analysis กับ Due Date Tracker

- Status: **To Do** · Points: 3 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: CA-195
- Jira: https://carmensoftware.atlassian.net/browse/CA-196

**บริบท**
ตาม hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 19-23 และ
hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 77-81 — สอง stat tile สกุลบาท
และ panel หัวข้อ "AP Aging Analysis" กับ "Due Date Tracker" ปรากฏบน Hotel AP Dashboard
เดียวกัน ใช้ข้อมูลจาก ap.dashboard.outstanding-aging-query ใบเดียวกัน

**สิ่งที่ต้องทำ**
- แสดง stat tile ตัวเลขสกุลบาท (฿ prefix) สองก้อนเหนือ panel อื่นตามตำแหน่งใน mockup โดยดึงค่าจาก endpoint ไม่ hardcode
- แสดง panel "AP Aging Analysis" จากข้อมูลที่ endpoint ส่งมาจริง (list + sum) — ไม่ fix bucket ใด ๆ เพิ่มเติมในชั้น FE เพราะ BE ไม่ได้ส่ง bucket มา
- แสดง panel "Due Date Tracker" เป็น list/table เรียงตาม due_date จากข้อมูลชุดเดียวกัน
- รองรับ EN/TH ตาม pattern เดิมของแอป (Reuse i18n ตาม accounting-foundation.md §4)

**Acceptance**
- Given เปิดหน้า Hotel AP Dashboard, when data โหลดสำเร็จ, then เห็น stat tile สองก้อน แสดงตัวเลขสกุลบาท และ panel หัวข้อ "AP Aging Analysis" กับ "Due Date Tracker" แสดงข้อมูล จาก endpoint
- Given endpoint คืน list ว่าง, when render, then panel แสดงสถานะไม่มีข้อมูลค้างชำระ ไม่ error
- Given endpoint ยังโหลดไม่เสร็จ, when render, then แสดง loading state ไม่ใช่ panel ว่างเปล่าเงียบ ๆ

**กฎที่ห้ามละเมิด**
- อ้าง hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 19-23 และ hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 77-81 — ยังไม่มี FRD ยืนยัน
- อ้าง accounting-foundation.md §4 Reuse Existing Carmen Services (i18n/design/DataGrid — Reuse)

### CA-197 — [BE] AP Dashboard — query endpoint รายการ AP document ที่รออนุมัติ (feed Pending Approvals Queue)

- Status: **To Do** · Points: 3 · Labels: accounting, ap, be
- Blocked by: CA-198 · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-197

**บริบท**
hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 24 และ
hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 82 มี h3 "Pending Approvals Queue"
เป็นหนึ่งใน panel ของ Hotel AP Dashboard เดียวกัน
หลักฐานว่ามี approval state ต่อเอกสารจริง มาจากปุ่ม Approve/Reject/Send Back ที่อยู่ใน
หน้า invoice entry (mockup2 บรรทัด 129-131) — ส่วนนั้นเป็นขอบเขตของ ap-invoice.txt (data
entry ต่อเอกสารเดียว) ผมอ้างอิงเพียงเพื่อยืนยันว่า field สถานะรออนุมัติมีอยู่จริงในระบบ
ไม่ได้แตก story ซ้ำเรื่อง approve/reject เอง — story นี้เป็นแค่ read-only rollup ข้ามหลาย
เอกสารสำหรับ dashboard เท่านั้น
ใช้ shared Workflow engine เดิมตาม accounting-foundation.md §4 (แถว Workflow —
Reuse/Extend) ไม่สร้าง approval state ใหม่ที่ Accounting เป็นเจ้าของเอง และไม่ผูกกับ JV
lifecycle status (§8 ห้ามผสม workflow status เข้า JV lifecycle) เพราะสถานะนี้เป็นของ
เอกสาร AP ก่อน post ไม่ใช่ JV

**สิ่งที่ต้องทำ**
- สร้าง read-only endpoint คืนรายการ AP document ที่มีสถานะรออนุมัติ (pending workflow stage) ของ BU ที่เลือก พร้อม doc_no, vendor, net_amount และ stage ปัจจุบันเท่าที่ shared Workflow module มีให้
- reuse shared Workflow engine เดิม ขยาย document type ให้ครอบคลุมเอกสาร AP แทนการสร้าง approval state ใหม่ซ้อนทับ
- ไม่ทำ action approve/reject ใน endpoint นี้ — เป็น read query เท่านั้น

**Acceptance**
- Given มี AP document ที่สถานะรออนุมัติของ BU ที่เลือก, when เรียก endpoint, then ได้ list พร้อม doc_no, vendor, net_amount และ stage ปัจจุบัน
- Given ไม่มีเอกสารรออนุมัติเลย, when เรียก endpoint, then ได้ list ว่าง
- endpoint เป็น read-only ไม่มี action approve/reject ในตัวมันเอง

**กฎที่ห้ามละเมิด**
- อ้าง hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 24 (h3 Pending Approvals Queue) และ hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 82, 129-131 (ปุ่ม Approve/Reject/Send Back บนหน้า invoice entry ที่บ่งชี้ว่ามี approval state ต่อเอกสาร) — ยังไม่มี FRD ยืนยันรายละเอียด stage
- อ้าง accounting-foundation.md §4 Workflow (Reuse/Extend เดิม ไม่ owned โดย Accounting) และ §8 Optional Workflow (approval เป็น workflow action แยกจาก JV lifecycle status)

### CA-198 — [FE] AP Dashboard — แสดง panel Pending Approvals Queue

- Status: **To Do** · Points: 2 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: CA-197
- Jira: https://carmensoftware.atlassian.net/browse/CA-198

**บริบท**
hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 24 และ
hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 82 — panel "Pending Approvals
Queue" บน Hotel AP Dashboard ใช้ข้อมูลจาก ap.dashboard.pending-approvals-query

**สิ่งที่ต้องทำ**
- แสดง panel "Pending Approvals Queue" เป็น list ของเอกสารรออนุมัติจาก endpoint พร้อม ลิงก์เปิดไปหน้าเอกสารนั้น
- ปุ่ม approve/reject จริงอยู่ในหน้าเอกสาร (ขอบเขต ap-invoice) — dashboard ไม่ทำ action ซ้ำในตัว panel

**Acceptance**
- Given endpoint คืน list ไม่ว่าง, when render, then เห็นรายการเอกสารรออนุมัติพร้อม ลิงก์เปิดไปหน้าเอกสาร
- Given list ว่าง, when render, then panel แสดงสถานะไม่มีรายการรออนุมัติ

**กฎที่ห้ามละเมิด**
- อ้าง hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 24 และ hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 82 — ยังไม่มี FRD ยืนยัน
- อ้าง accounting-foundation.md §4 Reuse Existing Carmen Services (i18n/design/ DataGrid — Reuse)

### CA-199 — [BE] AP Dashboard — query/aggregation endpoint ของ Tax Reconciliation & VAT ตาม tax status

- Status: **To Do** · Points: 5 · Labels: accounting, ap, be
- Blocked by: CA-200 · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-199

**บริบท**
hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 25 และ
hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 83 มี h3 "Tax Reconciliation &
VAT" เป็นหนึ่งใน panel ของ Hotel AP Dashboard เดียวกัน
มิติที่ใช้จัดกลุ่มได้จริงมาจากหน้า invoice entry (mockup2 บรรทัด 148-159): field
"Tax Status (Editable)" มี option Pending (Input VAT Undue), Confirm (Claimed VAT),
OnReview (Audit Incomplete), Submitted (Filed ภ.พ.30), Unclaim (Non-refundable),
None (Exempt) และ field "Tax Filing Period (Editable MM/YYYY)" — ส่วนนั้นเป็นขอบเขตของ
ap-invoice.txt (data entry ต่อเอกสารเดียว) ผมอ้างอิงเพียงเพื่อยืนยันว่า field เหล่านี้
มีอยู่จริงในระบบ ใช้เป็นมิติ group-by ของ dashboard เท่านั้น ไม่ได้แตก story การแก้ไข
field ซ้ำ
field เหล่านี้อยู่ใน Accounting-owned data ตาม accounting-foundation.md §3
("Tax/WHT accounting detail และ account mapping") — เป็นการ report ข้อมูลที่ Accounting
เป็นเจ้าของอยู่แล้ว ไม่ทับกับ epic tax-wht (5 story เดิม) ที่ทำเรื่อง post tax detail
เข้า JV และกันการกลับภาษีอัตโนมัติตอน reverse — เรื่องคนละชั้น (posting เข้า ledger
vs. read-only reconciliation view ข้ามหลายเอกสารสำหรับ VAT filing)

**สิ่งที่ต้องทำ**
- สร้าง read-only endpoint คืนจำนวนเอกสารและยอดรวมของ AP invoice จัดกลุ่มตาม tax_status (Pending/Confirm/OnReview/Submitted/Unclaim/None ตาม enum ที่มีอยู่แล้ว) ของ BU ที่เลือก
- รองรับ filter ด้วย tax_filing_period (MM/YYYY)
- ใช้ field tax_status และ tax_filing_period ที่มีอยู่แล้วต่อเอกสาร ไม่สร้าง state ใหม่ ซ้อนทับ และไม่คำนวณยอดภาษีใหม่หรือกฎ VAT ใด ๆ ที่ไม่ได้อยู่ใน field เดิม

**Acceptance**
- Given มี AP invoice ที่มี tax_status ต่างกันของ BU ที่เลือก, when เรียก endpoint, then ได้จำนวนเอกสารและยอดรวมแยกตามแต่ละ tax_status
- Given filter ด้วย tax_filing_period, when เรียก endpoint, then ได้เฉพาะเอกสารของ period นั้น
- endpoint เป็น read-only ไม่แก้ tax_status ของเอกสารใด ๆ

**กฎที่ห้ามละเมิด**
- อ้าง hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 25 (h3 Tax Reconciliation & VAT) และ hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 83, 152-159 (Tax Status enum, Tax Filing Period MM/YYYY) — ยังไม่มี FRD ยืนยันสูตรหรือ breakdown เพิ่มเติม
- อ้าง accounting-foundation.md §3 Accounting-owned data (Tax/WHT accounting detail และ account mapping) — ไม่ทับกับ epic tax-wht ที่ทำเรื่อง posting tax เข้า JV

### CA-200 — [FE] AP Dashboard — แสดง panel Tax Reconciliation & VAT

- Status: **To Do** · Points: 3 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: CA-199
- Jira: https://carmensoftware.atlassian.net/browse/CA-200

**บริบท**
hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 25 และ
hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 83 — panel "Tax Reconciliation &
VAT" บน Hotel AP Dashboard ใช้ข้อมูลจาก ap.dashboard.tax-reconciliation-query

**สิ่งที่ต้องทำ**
- แสดง panel "Tax Reconciliation & VAT" เป็น breakdown จำนวน/ยอดต่อ tax_status แต่ละ สถานะจาก endpoint
- มี filter ด้วย tax_filing_period (MM/YYYY) ตามที่ endpoint รองรับ

**Acceptance**
- Given endpoint คืนข้อมูล breakdown, when render, then เห็น panel แสดงจำนวน/ยอดต่อ tax_status แต่ละสถานะ
- Given ยังไม่มี invoice ที่ tax_status ใดเลย, when render, then แสดง 0/ว่างสำหรับ สถานะนั้นโดยไม่ error

**กฎที่ห้ามละเมิด**
- อ้าง hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 25 และ hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 83, 152-159 — ยังไม่มี FRD ยืนยัน
- อ้าง accounting-foundation.md §3 Accounting-owned data (Tax/WHT accounting detail)

### CA-201 — [BE] AP Dashboard — query endpoint activity log ข้ามเอกสาร AP (feed Review & Status Audit Tracker)

- Status: **To Do** · Points: 3 · Labels: accounting, ap, be
- Blocked by: CA-202 · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-201

**บริบท**
hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 26 และ
hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 84 มี h3 "Review & Status Audit
Tracker" เป็นหนึ่งใน panel ของ Hotel AP Dashboard เดียวกัน
หลักฐานว่ามี activity log ต่อเอกสารอยู่แล้วมาจาก mockup2 บรรทัด 279-280 (h3 "Invoice
Document Activity Log", ปุ่ม "Close Audit Trail" — เปิดจาก modal ของหน้า invoice entry
เดียว) ส่วนนั้นเป็นขอบเขตของ ap-invoice.txt (ดู log ของเอกสารเดียว) story นี้เป็นการ
rollup ข้ามหลายเอกสารสำหรับ dashboard เท่านั้น ไม่ได้แตก story การดู log รายเอกสารซ้ำ
ตาม accounting-foundation.md §4 แถว "Activity log" ระบุ Reuse/Extend ของ backend
activity registry เดิม (เพิ่ม action submit/approve/reject/post/schedule/reverse/
void/retry สำหรับ JV) — story นี้ขยายแนวทางเดียวกันให้ครอบคลุมเอกสาร AP แทนการสร้าง
ตาราง audit log ใหม่ที่ Accounting เป็นเจ้าของเอง

**สิ่งที่ต้องทำ**
- สร้าง read-only endpoint คืนรายการ activity/status-change ล่าสุดข้ามหลาย AP document ของ BU ที่เลือก (ผู้ทำรายการ, action, เวลา, doc reference) โดยดึงจาก shared Activity Log service เดิม ไม่สร้างตารางใหม่ซ้อนทับ
- ขยาย action ที่ลงทะเบียนให้ครอบคลุม AP document lifecycle ต่อยอดจาก action ชุดที่มีอยู่ แล้วสำหรับ JV

**Acceptance**
- Given มี activity log entries ของ AP document ของ BU ที่เลือก, when เรียก endpoint, then ได้ list เรียงตามเวลาล่าสุดพร้อมผู้ทำรายการและ action
- Given ไม่มี entries, when เรียก endpoint, then ได้ list ว่าง
- endpoint เป็น read-only

**กฎที่ห้ามละเมิด**
- อ้าง hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 26 (h3 Review & Status Audit Tracker) และ hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 84, 279-280 (h3 Invoice Document Activity Log, ปุ่ม Close Audit Trail) — ยังไม่มี FRD ยืนยัน field ครบของ log ระดับ dashboard rollup
- อ้าง accounting-foundation.md §4 Activity log (Reuse/Extend backend activity registry เดิม)

### CA-202 — [FE] AP Dashboard — แสดง panel Review & Status Audit Tracker

- Status: **To Do** · Points: 2 · Labels: accounting, ap, fe
- Blocked by: - · Blocks: CA-201
- Jira: https://carmensoftware.atlassian.net/browse/CA-202

**บริบท**
hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 26 และ
hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 84 — panel "Review & Status
Audit Tracker" บน Hotel AP Dashboard ใช้ข้อมูลจาก ap.dashboard.status-audit-query

**สิ่งที่ต้องทำ**
- แสดง panel "Review & Status Audit Tracker" เป็น list ของ activity/status-change ล่าสุดจาก endpoint

**Acceptance**
- Given endpoint คืน list ไม่ว่าง, when render, then เห็นรายการ activity ล่าสุดพร้อม ผู้ทำรายการ, action และเวลา
- Given list ว่าง, when render, then panel แสดงสถานะไม่มี activity

**กฎที่ห้ามละเมิด**
- อ้าง hotel_ap_erp_Aging_Dashboard_mockup_v4_4_4.html บรรทัด 26 และ hotel_ap_erp_Other_Dashboard_mockup_v4_4_3.html บรรทัด 84, 279-280 — ยังไม่มี FRD ยืนยัน
- อ้าง accounting-foundation.md §4 Reuse Existing Carmen Services (i18n/design/ DataGrid — Reuse)

