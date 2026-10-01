# Jira AR tasks — extract from epic CA-5 (2026-10-01)

Source: Jira project CA (carmensoftware.atlassian.net), JQL `parent = CA-5 OR key = CA-5 OR labels = ar` plus text search for AR/receivable/customer, pulled 2026-10-01 via twg REST.
Purpose: same as the AP extract — see what the backlog says about AR and compare with what backend-v2 PR #701 (schema) and PR #704 (customer + ARIV/ARDN/ARCN service, merged 2026-10-01) implemented.

**Finding at extract time: the AR epic had only two DOC stories and zero implementation stories.** CA-40 (design doc) was Resolved; CA-41 (break down backlog) was the only open AR work item while the backend already had a working AR invoice module. Resolved the same day by creating CA-244..CA-263 (see next section). Text search returned 28 hits but all others are foundation/AP/master-data issues that merely mention "AR" (listed under cross-references).

## Index

| Key | Type | Status | Pts | Summary |
|---|---|---|---|---|
| CA-5 | Epic | In Progress | - | Accounts Receivable |
| CA-40 | Story | Resolved | 3 | [DOC] AR — เขียน design doc |
| CA-41 | Story | To Do | 2 | [DOC] AR — แตก backlog หลัง design อนุมัติ |

Cross-references that mention AR (not AR-owned): CA-169 (Payment Type delete-safeguard must check AP/AR tables), CA-11/CA-71 (posting contract every subledger must call), CA-27 (micro-business decision, Resolved).

> Link direction note: at extract time CA-40 was recorded as *blocked by* CA-41, same reversal as the AP pair (CA-38 / CA-39). Fixed 2026-10-01 (design doc now blocks backlog breakdown). The CA-244..263 links created the same day were already in the correct direction.

## What exists in code (backend-v2 `81802ec97`)

- **Customer master** — `apps/micro-business/src/master/customers/` (module, controller, service, serializer); tables `tb_customer` (schema L7483), `tb_customer_address` (L7529).
- **AR invoice module** — `apps/micro-business/src/ar/ar-invoice/` (controller, logic, posting, reference, running-code, service, validation, writer, serializer, interface, workflow mapper). One module for ARIV / ARDN / ARCN via `doc_type`.
- **Tables** — `tb_ar_invoice` (L7559), `tb_ar_invoice_detail` (L7653), `tb_ar_invoice_detail_dimension` (L7738), `tb_ar_invoice_reference` (L7756), `tb_ar_tax_invoice` (L7780); schema-only so far: `tb_ar_receipt` (L7821), `tb_ar_receipt_detail` (L7906), `tb_ar_receipt_wht` (L7930).
- **Gateway routes** (`apps/backend-gateway/src/application/ar-invoice/ar-invoice.controller.ts`): `GET /`, `GET :id`, `GET :id/reference-candidates`, `POST /`, `PUT :id`, `DELETE :id`, `POST :id/{submit,approve,review,reject,void}`.
- **Posting** — through `GlSubledgerPostingService` with new source `ar_invoice`; tax invoice TXIV/TXDN/TXCN issued at post; ARIV nets posted ARCN at the same rate.
- **Frontend** — none.

Spec: `docs/superpowers/specs/2026-10-01-accounting-ar-invoice-service-design.md` (§2 scope, §13 FRD deviations for BA, §15 follow-ups). Schema spec: `docs/superpowers/specs/2026-09-30-accounting-ar-schema-design.md`.

## Created in Jira 2026-10-01 (CA-41 done)

Bulk-created from the candidate table below; 8 delivered items created as Resolved with an evidence comment (PR #701/#704), 11 as To Do; spike CA-263 under CA-1. Links (`Blocks`, correct direction BE → FE, spike → BA-gated work): CA-245→259, CA-246→260/261, CA-248→261, CA-249→261, CA-251→260, CA-252→262/253, CA-263→253/256/258. Story points are initial estimates in the {1,2,3,5,8} set; the team may re-estimate. CA-41 and this map replace the `.jira-map.json` that CA-41 mentions (no such file exists in this repo).

| Key | Status | Pts | Summary |
|---|---|---|---|
| CA-244 | Resolved | 5 | [BE] AR Schema — ตาราง customer, ar_invoice, reference, tax_invoice, receipt และ enum (schema-only) |
| CA-245 | Resolved | 5 | [BE] Customer Master — CRUD tb_customer + address, default snapshot, tax_no validation และ CUSTOMER_IN_USE |
| CA-246 | Resolved | 5 | [BE] AR Invoice — กฎ header/บรรทัด, running code ARIV/ARDN/ARCN และการคำนวณ HALF_UP |
| CA-247 | Resolved | 5 | [BE] AR Invoice — lifecycle draft→in_review→posted→void ผ่าน WorkflowOrchestrator type ar_invoice |
| CA-248 | Resolved | 8 | [BE] AR Invoice — post ผ่าน GlSubledgerPostingService source ar_invoice ตามรูป JV §8 และ void reversal §11 |
| CA-249 | Resolved | 5 | [BE] AR Invoice — Doc Reference ARIV หักกลบ ARCN ที่ posted, same-rate, reference-candidates และคู่บรรทัดเศษปัด |
| CA-250 | Resolved | 3 | [BE] AR Tax Invoice — ออก TXIV/TXDN/TXCN ตอน post พร้อม snapshot ลูกค้า และ mixed VAT rate guard |
| CA-251 | Resolved | 3 | [BE] AR — error catalog, activity registry, RPC contract, gateway ar-invoice, permission accounting.ar และ Bruno |
| CA-252 | To Do | 8 | [BE] AR Receipt — ARRC + TXRC + WHT รับ ตัด unpaid ของ ARIV/ARDN และ ARCN อัตราต่าง |
| CA-253 | To Do | 5 | [BE] ARDP — ใบมัดจำรับล่วงหน้าและการตัดมัดจำเข้า ARIV |
| CA-254 | To Do | 5 | [BE] AR Invoice — source PMS folio (Add Folio) และ is_pms_folio |
| CA-255 | To Do | 2 | [BE] AR Invoice — source copy และ ai |
| CA-256 | To Do | 3 | [BE] AR Invoice — VAT inclusive (VAT07_INC) ผ่าน inclusive flag บน tax profile |
| CA-257 | To Do | 5 | [BE] AR — aging / outstanding aggregate query endpoint (feed AR Aging และ Due Date Tracker) |
| CA-258 | To Do | 2 | [BE] Customer — credit limit check ตอน submit ARIV/ARDN |
| CA-259 | To Do | 5 | [FE] Customer Master — หน้า list พร้อม search/filter และฟอร์ม create/edit พร้อม address และ AR default |
| CA-260 | To Do | 3 | [FE] AR Documents — list รวม ARIV/ARDN/ARCN พร้อมคอลัมน์ status, outstanding และ filter |
| CA-261 | To Do | 8 | [FE] AR Invoice — ฟอร์ม header, line editor, reference ARCN, tax invoice และ action bar ตามสถานะ |
| CA-262 | To Do | 5 | [FE] AR Receipt — ฟอร์มใบเสร็จ เลือกใบค้างชำระมาตัด, WHT รับ และ action bar |
| CA-263 | To Do | - | [SPIKE] AR §13 จุดเบี่ยงจาก FRD ที่ BA ต้องตัดสิน — OI-1/2 ARDP, OI-3 จังหวะ post, OI-5.x reference, OI-6 Tax 2, original_tax_invoice_no |

Jira REST note: in `POST /issueLink` with type Blocks, `inwardIssue` is the blocker ("inward blocks outward"). The first attempt used the opposite and was deleted/recreated.

## Candidate breakdown for CA-41 (from spec §2 / §15 — not yet created in Jira)

Already delivered by PR #701/#704 (would be created as Resolved, for traceability):

| # | Side | Candidate story | Evidence |
|---|---|---|---|
| 1 | BE | AR schema — customer, invoice, reference, tax invoice, receipt tables + enums | PR #701, migration `20260930074505_accounting_ar_tables` |
| 2 | BE | Customer master — CRUD + address, BU scope, permission | `master/customers/*` |
| 3 | BE | AR invoice — header/line rules, running code ARIV/ARDN/ARCN, HALF_UP calc, base_net = base_sub_total − base_discount | `ar-invoice.logic.ts`, `ar-invoice.running-code.ts` |
| 4 | BE | AR invoice — lifecycle submit/approve/review/reject/void with optional workflow | `ar-invoice.service.ts`, `workflow/ar-invoice-workflow.mapper.ts` |
| 5 | BE | AR invoice — post via `GlSubledgerPostingService` (`ar_invoice` source), JV shape §8, void → reversal | `ar-invoice.posting.ts`, facade change |
| 6 | BE | AR invoice — ARIV ↔ posted ARCN reference netting, same-rate rule, reference-candidates | `ar-invoice.reference.ts` |
| 7 | BE | Tax invoice TXIV/TXDN/TXCN issued at post | `tb_ar_tax_invoice`, posting |
| 8 | BE | Error catalog (bilingual), activity registry, Bruno collection | `packages/error-catalog`, Bruno commits |

Not yet built (spec §2 out-of-scope / §15):

| # | Side | Candidate story | Blocked by |
|---|---|---|---|
| 9 | BE | AR Receipt ARRC + TXRC + WHT received (`tb_ar_receipt*` already exist) | — |
| 10 | BE | ARDP deposit + deposit application to invoice | BA OI-1 / OI-2 |
| 11 | BE | PMS folio source (`source = pms_folio`, Add Folio) | PMS interface |
| 12 | BE | Source `copy` / `ai` | — |
| 13 | BE | VAT inclusive (VAT07_INC) — needs tax profile inclusive flag | BA |
| 14 | BE | AR aging / outstanding aggregate query (mirror of AP CA-195) | — |
| 15 | BE | Credit limit check | BA |
| 16 | FE | Customer master list + form | — |
| 17 | FE | AR document list (ARIV/ARDN/ARCN) + status column | — |
| 18 | FE | AR invoice form: header, line editor, reference table, tax invoice display, action bar | — |
| 19 | FE | AR receipt UI | 9 |
| 20 | BA | Decide §13 deviations: OI-1/2 (deposit), OI-3 (post timing), OI-5.x (reference rules), OI-6 (Tax 2), original tax invoice no. column | — |

Story points and Given/When/Then acceptance are intentionally not drafted here — CA-41 rules require them to cite the approved design doc sections, and points must come from the team.

## Detail (AR-owned issues)

### CA-5 — Accounts Receivable

- Type: Epic · Status: **In Progress** · Points: - · Labels: accounting, ar
- Parent: - · Blocked by: - · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-5

**บริบท**
ขั้นส่งมอบที่ 6 ตาม README.md แต่ยังไม่มี design doc
accounting-foundation.md §2 ระบุ AR invoice/receipt เป็น non-goal ของระยะแรก

> **Comment 2026-10-01 carmensoftware Developer:** อัปเดตสถานะ 2026-10-01: AR เริ่ม implement แล้วใน micro-business (backend-v2)
- PR #701 (merged 2026-09-30) — AR และ customer tables, migration 20260930074505_accounting_ar_tables
- PR #704 (merged 2026-10-01) — customer master + AR invoice / debit note / credit note service (CRUD, fin calc, JV, validation, workflow, GL post, tax invoice at post, CN netting)
ยังค้าง: ARDP (deposit), receipt และ PMS folio ยังเป็น schema-only; deploy tenant migration ต่อ BU; fe-license fixture รอ frontend gen; BA ตัดสิน open decisions OI-1..7 ใน design doc
Design doc ปิดที่ CA-40 แล้ว ส่วน CA-41 (แตก backlog) ยังไม่ได้ทำ

### CA-40 — [DOC] Accounts Receivable — เขียน design doc

- Type: Story · Status: **Resolved** · Points: 3 · Labels: accounting, ar, doc
- Parent: CA-5 · Blocked by: CA-41 · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-40

**บริบท**
README.md ระบุ AR เป็นขั้นส่งมอบที่ 6 แต่ยังไม่มีเอกสารออกแบบ

**สิ่งที่ต้องทำ**
- กำหนดขอบเขต AR ที่ไม่ทับกับ Accounting Foundation
- ระบุ posting rule ที่ AR จะเรียกผ่าน posting contract ของ accounting-foundation.md §9
- ระบุ master data ที่ reuse และที่ AR เป็นเจ้าของ
- ระบุ idempotency key และ source version ที่ AR จะส่งเข้า Journal Staging
- ระบุ open decisions ที่ต้องตัดสินก่อน implementation

**Acceptance**
- มีไฟล์ design doc ของ AR อยู่ใน repository
- เอกสารระบุ posting contract ที่ใช้และไม่มีการเขียน ledger โดยตรง
- เอกสารผ่าน review และ merge เข้า main แล้ว

**กฎที่ห้ามละเมิด**
- README.md — AP, AR และ Fixed Assets ต้องส่งรายการเข้า GL ผ่าน posting contract เดียวกัน ห้ามเขียน journal tables โดยตรง
- accounting-foundation.md §5 — Inventory, AP, AR, Fixed Assets และ external API ต้องส่ง deterministic idempotency key
- ห้ามแต่ง acceptance criteria ของฟีเจอร์ AR ก่อนเอกสารนี้ถูกอนุมัติ

> **Comment 2026-10-01 carmensoftware Developer:** Design doc ของ AR เขียนเสร็จ ผ่าน review และ merge เข้า main ของ carmen-accounting-concept แล้ว (2026-10-01)
- 2026-09-30-accounting-ar-schema-design.md — AR/customer schema (10 models, 7 enums) และ mapping กับ FRD v1.07
- 2026-10-01-accounting-ar-invoice-service-design.md — scope, customer master, ARIV/ARDN/ARCN workflow, fin calc, JV, 11 FRD deviations, open decisions OI-1..7
Posting ผ่าน GlSubledgerPostingService (posting contract ตาม foundation §9) ไม่มีการเขียน tb_gl_* โดยตรง — ตรง acceptance ทุกข้อ จึงปิดเป็น Resolved

### CA-41 — [DOC] Accounts Receivable — แตก backlog หลัง design อนุมัติ

- Type: Story · Status: **To Do** · Points: 2 · Labels: accounting, ar, doc
- Parent: CA-5 · Blocked by: - · Blocks: CA-40
- Jira: https://carmensoftware.atlassian.net/browse/CA-41

**บริบท**
ทำหลัง design doc ของ AR ถูกอนุมัติแล้วเท่านั้น

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


## Cross-reference detail

### CA-169 — [SPIKE] Payment Type delete-safeguard ต้องตรวจตาราง AP/AR ที่เป็น Non-goal ของ Phase 1

- Type: Spike · Status: **To Do** · Points: 2 · Labels: accounting, master-data, spike
- Parent: CA-1 · Blocked by: CA-235 · Blocks: -
- Jira: https://carmensoftware.atlassian.net/browse/CA-169

**คำถามที่ต้องตอบ**
FRD Payment Type §4 Delete Icon และ §8 VAL-PT-008 บังคับให้ตรวจ
ap_payment_header และ ar_receipt_header ก่อนอนุญาตลบ payment type
แต่ accounting-foundation.md §2 ระบุว่า AP invoice/payment และ AR
invoice/receipt เป็น Non-goal ของระยะแรก ตารางทั้งสองนี้จึงยังไม่มี
อยู่จริงในระบบตอนนี้ ควรทำอย่างไรกับการตรวจนี้ใน MVP ของ Payment Type

**ทำไมถึง block**
master-data.payment-type.be-dependency-guard implement เฉพาะ
CheckPaymentTypeDependency (vendor/customer profile) ไว้ก่อน ส่วน
CheckPaymentTypeUsage ต่อ ap_payment_header/ar_receipt_header ยังทำ
แบบเต็มรูปแบบไม่ได้จนกว่าจะรู้ว่า MVP นี้ยอมให้ hard delete ได้อย่างมี
เงื่อนไขชั่วคราวหรือไม่ หรือต้องรอ AP/AR ส่งมอบก่อน

**ต้องส่งมอบ**
- คำตอบว่า MVP ของ Payment Type ควร (ก) บล็อก hard delete ทั้งหมดไว้ ก่อนจนกว่า AP/AR ส่งมอบ, (ข) อนุญาต hard delete แบบมีเงื่อนไขชั่วคราว โดยตรวจเฉพาะแหล่งข้อมูลที่มีอยู่ตอนนี้ หรือ (ค) อื่นๆ พร้อมเหตุผล
- ADR สั้นใน docs/decisions/ ระบุว่า epic accounts-payable/ accounts-receivable ต้อง extend CheckPaymentTypeUsage เมื่อ implement ตารางจริง
- PR แก้ FRD หรือหมายเหตุใน backlog ให้ VAL-PT-008 สะท้อนขอบเขตจริง ของ MVP ระยะแรก

**Timebox**
1 วัน

