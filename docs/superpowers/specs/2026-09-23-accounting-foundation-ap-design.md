# Design — Accounting Foundation Gap + Accounts Payable (AP)

วันที่ 2026-09-23
สถานะ: รอ review
เป้าหมายโค้ด: `carmen-turborepo-backend-v2` (`apps/micro-business`, `apps/backend-gateway`, `packages/*`)

---

## 1. ที่มาและการตัดสินใจหลัก

repo นี้มี PRD 9 ใบ + `prisma/schema.prisma` ที่ออกแบบระบบบัญชีเป็น microservice แยกชื่อ `micro-accounting`
แต่โค้ดจริงใน `carmen-turborepo-backend-v2` มี General Ledger ครบชุดอยู่แล้วใน
`apps/micro-business/src/gl/` (jv, posting, period, prefix, template, balance, budget, account-group)
บนตาราง `tb_gl_*` ของ tenant Prisma schema และมี workflow orchestrator, running code, tenant context
เป็นโค้ดภายใน `micro-business` ทั้งหมด ระบบไม่มี event bus (สื่อสารด้วย HTTP-as-RPC เท่านั้น)

การตัดสินใจที่ตกลงกันวันนี้ (ผู้ใช้ยืนยันทุกข้อ)

| หัวข้อ | ตัดสินใจ | เหตุผล |
|---|---|---|
| Service boundary | ทุก accounting module อยู่ใน `apps/micro-business` ต่อจาก `src/gl` | GL, workflow, numbering, tenant runner อยู่ที่นั่นแล้ว; AP ต้อง join vendor/PO/GRN ใน DB เดียวกัน; ไม่มี event bus ให้ decouple |
| ขอบเขต spec แรก | Foundation gap ที่ AP ต้องใช้ + AP ครบวงจร (invoice → payment → post) | ได้ของใช้จริงเร็วสุด และวางรากให้ AR/FA |
| GRN linkage | ผูก GRN แบบ structured หลายใบต่อ invoice, ยังไม่มี GRNI | GRN ปัจจุบันไม่ post GL จึงไม่มี double count เมื่อ invoice เป็นผู้ post |
| Payment | รวม Payment Voucher + `tb_bank_account` ขั้นต่ำ | AP ที่จ่ายเงินไม่ได้ใช้จริงไม่ได้ |
| WHT | หักตอนจ่าย เก็บข้อมูลครบสำหรับ 50 ทวิ แต่ยังไม่ออกหนังสือ | ตรงกฎหมายไทย; หนังสือรับรอง/ภ.ง.ด. ไปอยู่ Tax module |
| Dimension | สร้าง foundation ตาม `accounting-foundation.md` §6 (data-driven) | `tb_dimension` เดิมเป็น custom-field ของ inventory ไม่ใช่ accounting dimension |
| AP → GL | Subledger posting facade บาง ๆ ใน `src/gl` | ใช้ `createInternal` + `post()` เดิม, กัน post ซ้ำ, AR/FA ใช้ซ้ำได้ โดยไม่สร้าง rule engine |

ผลข้างเคียง: PRD/README/`prisma/schema.prisma` ใน repo นี้ที่กล่าวถึง `micro-accounting` และ TCP transport
ถือว่าล้าสมัย ต้องปรับตามหลัง (ดู §15)

แหล่งอ้างอิง

- AP FRD v4.5.06: `../Accounting-docs/AP/Invoice/carmen_cloud_erp_ap_module_functional_requirement_document_frd.md`
- `../carmen-accounting-task/docs/accounting-foundation.md` §4 (reuse table), §6 (dimension), §9 (posting), §13 (open decisions)
- `../carmen-accounting-task/docs/backlog/accounting-backlog.yaml` epic `accounts-payable`, `posting-engine`, `accounting-dimensions`, `tax-wht`
- โค้ด GL: `apps/micro-business/src/gl/gl-jv/gl-jv.service.ts` (`createInternal`), `gl-posting/gl-posting.service.ts` (`post`, `reverse`, `voidJv`)

---

## 2. ขอบเขต

### ในขอบเขต

- Foundation: accounting dimension (นิยาม/ค่า/กฎต่อบัญชี/junction บน JV), subledger posting facade, refactor `post()`/`reverse()` ให้รับ tx ภายนอก, extend `tb_tax_profile`, extend `tb_vendor`, `tb_bank_account` master, key ใหม่ใน `gl_setting`
- AP Invoice 4 ประเภทใน form เดียว: Invoice (APIV), Debit Note (APDN), Credit Note (APCN), Deposit (APDP)
- สร้าง invoice จาก GRN ที่ committed (หลาย GRN ต่อใบ, ติดตาม matched qty)
- Doc Reference หักกลบ DN/CN/DP กับ invoice พร้อม realized FX สำหรับ deposit
- Tax invoice record สำหรับ ภ.พ.30 (ข้อมูลอย่างเดียว)
- Payment Voucher: allocation ระดับบรรทัด, WHT ตอนจ่าย, other expense, realized FX, void
- Workflow ผ่าน `WorkflowOrchestratorService` เดิม (type ใหม่ `ap_invoice`, `ap_payment`)
- RPC contract, gateway controller/service/swagger, permission, error catalog, activity registry, seed, migration
- Backend เท่านั้น

### นอกขอบเขต (ทำใน spec ถัดไป)

Frontend ทั้งหมด · GRNI/inventory posting · WHT certificate และ ภ.ง.ด. export · รายงาน ภ.พ.30 ·
bank reconciliation/statement · vendor bank account · จ่ายข้ามสกุลเงิน · budget check (ระบุ hook point เท่านั้น) ·
AI/Excel import · cheque register · aging dashboard query · การแก้ PRD/README/concept Prisma ใน repo นี้

---

## 3. ตำแหน่งในโค้ดและโครง module

```text
apps/micro-business/src/
├── gl/
│   ├── gl-dimension/             ← ใหม่: tb_gl_dimension, tb_gl_dimension_value, tb_gl_account_dimension_rule
│   ├── gl-subledger-posting/     ← ใหม่: GlSubledgerPostingService (postFromSource / reverseBySource)
│   ├── gl-jv/                    ← แก้: createInternal เขียน tb_gl_jv_detail_dimension
│   └── gl-posting/               ← แก้: แยก postInTx / reverseInTx (behavior-preserving)
├── master/
│   ├── bank-account/             ← ใหม่
│   ├── tax-profile/              ← แก้: ฟิลด์ tax_type / wht
│   └── vendor/                   ← แก้: ฟิลด์ default currency / credit term / AP account
└── ap/
    ├── ap-invoice/               ← ใหม่: controller, service, logic, validation, running-code, workflow mapper, dto, interface
    └── ap-payment/               ← ใหม่: โครงเดียวกัน
```

ทุก feature เป็น flat `@Module` (`imports: [TenantModule, CommonModule, ...]`) และ import ตรงใน
`apps/micro-business/src/app.module.ts` เหมือน `GlJvModule`, `GlPostingModule` ไม่มี umbrella module

ฝั่ง gateway: `apps/backend-gateway/src/application/ap-invoice/`, `application/ap-payment/`
ตามโครง `application/gl-jv/` และ master 4 ตัวใต้ `config/config_*` ตาม pattern เดิม

---

## 4. Schema (tenant Prisma schema: `packages/prisma-shared-schema-tenant/prisma/schema.prisma`)

ธรรมเนียมที่ใช้กับทุกตารางใหม่: prefix `tb_`, `id uuid gen_random_uuid()`, `doc_version Int @default(0)`,
`created_at/created_by_id/updated_at/updated_by_id/deleted_at/deleted_by_id`, `info Json?`, `dimension Json?`
(custom-field เดิม), unique แบบ `(field, deleted_at)`, จำนวนเงิน `Decimal(20,5)`, อัตราแลกเปลี่ยน `Decimal(15,5)`,
enum ชื่อ `enum_<table>_<field>` และเป็นแหล่งความจริงเดียวของ Zod/TS

### 4.1 Foundation ตารางใหม่

**`tb_gl_dimension`** — นิยาม dimension

| ฟิลด์ | ชนิด | หมายเหตุ |
|---|---|---|
| code | VarChar(20) | unique (code, deleted_at) |
| name, name_local | VarChar | |
| description | VarChar? | |
| sequence | Int | ลำดับแสดงผล |
| is_active | Boolean | |

seed 7 แถว: `market`, `sales`, `project`, `event`, `location`, `channel`, `guest_type`

**`tb_gl_dimension_value`**

| ฟิลด์ | ชนิด | หมายเหตุ |
|---|---|---|
| gl_dimension_id | Uuid FK | |
| code | VarChar(30) | unique (gl_dimension_id, code, deleted_at) |
| name, name_local | VarChar | |
| effective_from, effective_to | Date? | ค่าใช้ได้เมื่อ `jv_date` อยู่ในช่วง |
| is_active | Boolean | |

**`tb_gl_account_dimension_rule`**

| ฟิลด์ | ชนิด | หมายเหตุ |
|---|---|---|
| chart_of_accounts_id | Uuid FK | |
| gl_dimension_id | Uuid FK | unique (chart_of_accounts_id, gl_dimension_id, deleted_at) |
| requirement | `enum_gl_account_dimension_rule_requirement` = mandatory / optional / prohibited | บัญชีที่ไม่มีแถว = optional |

**`tb_gl_jv_detail_dimension`** — junction บน JV line (ไม่มี soft delete, ลบตามบรรทัด)

| ฟิลด์ | ชนิด | หมายเหตุ |
|---|---|---|
| gl_jv_detail_id | Uuid FK | |
| gl_dimension_id | Uuid FK | unique (gl_jv_detail_id, gl_dimension_id) |
| gl_dimension_value_id | Uuid FK | |
| dimension_code, value_code | VarChar | snapshot สำหรับ report |

**`tb_bank_account`**

| ฟิลด์ | ชนิด | หมายเหตุ |
|---|---|---|
| code | VarChar(20) | unique (code, deleted_at) |
| name | VarChar | |
| bank_name, bank_branch | VarChar? | |
| account_no | VarChar(50) | |
| currency_id, currency_code | Uuid, VarChar(3) | |
| chart_of_accounts_id | Uuid FK | บัญชีเงินฝากใน COA |
| is_active | Boolean | |
| description, note | VarChar? | |

### 4.2 Foundation ตาราง/enum ที่ extend

| ที่ | เพิ่ม |
|---|---|
| `tb_tax_profile` | `tax_type enum_tax_profile_tax_type (vat / wht) @default(vat)`, `chart_of_accounts_id Uuid?` (บัญชีภาษี default), `wht_pnd_form enum_tax_profile_wht_pnd_form? (pnd_1, pnd_2, pnd_3, pnd_53, pnd_54)`, `wht_income_type VarChar?` (ประเภทเงินได้ตามใบแนบ) |
| `tb_vendor` | `default_currency_id Uuid?`, `default_currency_code VarChar(3)?`, `credit_term_id Uuid?`, `credit_term_name VarChar?`, `credit_term_days Int?`, `ap_chart_of_accounts_id Uuid?` |
| `enum_workflow_type` | `ap_invoice`, `ap_payment` |
| `tb_gl_jv_header` | partial unique index `gljvheader_source_ref_u (source, source_ref_type, source_ref_id) WHERE deleted_at IS NULL AND jv_status <> 'void'` (migration SQL) |
| `tb_config_running_code` | แถว type `AP-IV`, `AP-DN`, `AP-CN`, `AP-DP`, `AP-PV` pattern `{prefix}{date('yyMM')}{running(4,'0')}` prefix `APIV/APDN/APCN/APDP/APPV` |
| `gl_setting` ใน `tb_application_config.value` | `ap_control_account_id`, `input_vat_account_id`, `wht_payable_account_id`, `advance_deposit_account_id`, `realized_fx_gain_account_id`, `realized_fx_loss_account_id`, `ap_jv_prefix_id` (fallback `auto_jv_prefix_id`) |

`enum_gl_jv_source` มีค่า `ap` อยู่แล้ว ไม่ต้องแก้

### 4.3 AP ตารางใหม่

**`tb_ap_invoice`** (header ทั้ง 4 doc type)

| กลุ่ม | ฟิลด์ |
|---|---|
| เอกสาร | `doc_no VarChar(30)` unique (doc_no, deleted_at) · `doc_type enum_ap_invoice_doc_type (invoice, debit_note, credit_note, deposit)` · `doc_status enum_ap_invoice_status (draft, in_review, posted, void)` · `doc_source enum_ap_invoice_source (manual, grn, copy, excel, ai, interface)` · `doc_date Date` (วันที่ลง GL) · `description VarChar?` |
| vendor | `vendor_id Uuid` · `vendor_name VarChar` · `vendor_invoice_no VarChar(100)` · `invoice_date Date` · `credit_term_id Uuid?` · `credit_term_name VarChar?` · `credit_term_days Int` · `due_date Date` |
| เงิน | `currency_id Uuid` · `currency_code VarChar(3)` · `exchange_rate Decimal(15,5)` · `base_currency_id Uuid` · `base_currency_code VarChar(3)` · `sub_total_amount, discount_amount, net_amount, vat_amount, total_amount Decimal(20,5)` · `base_sub_total_amount, base_discount_amount, base_net_amount, base_vat_amount, base_total_amount Decimal(20,5)` · `wht_estimate_amount Decimal(20,5)` (แสดงเท่านั้น) · `outstanding_amount Decimal(20,5)` = Σ `unpaid_amount` ของบรรทัด · `reference_applied_amount Decimal(20,5)` (ยอดที่ใบนี้ถูกใบอื่นอ้างไปหัก ใช้กับ DN/CN/DP) |
| GL | `gl_jv_id Uuid?` · `gl_jv_no VarChar?` · `posted_at DateTime?` · `posted_by_id Uuid?` |
| void | `void_at DateTime?` · `void_by_id Uuid?` · `void_reason VarChar?` · `void_gl_jv_id Uuid?` |
| workflow | ชุดเดียวกับ `tb_gl_jv_header`: `workflow_id, workflow_name, workflow_history Json @default("[]"), workflow_current_stage, workflow_previous_stage, workflow_next_stage, user_action Json?, last_action enum_last_action?, last_action_at_date, last_action_by_id, last_action_by_name` |
| อื่น | `attachments Json @default("[]")` รูป `[{originalName, fileToken, contentType}]` · `info`, `dimension`, `doc_version`, audit cols |

unique เพิ่ม: partial `(vendor_id, vendor_invoice_no) WHERE deleted_at IS NULL AND doc_status <> 'void'`

**`tb_ap_invoice_detail`**

| กลุ่ม | ฟิลด์ |
|---|---|
| บรรทัด | `ap_invoice_id Uuid FK` · `sequence_no Int` · `description VarChar?` · `product_id Uuid?` · `product_name VarChar?` · `unit_id Uuid?` · `unit_name VarChar?` · `quantity Decimal(20,5)` · `unit_price Decimal(20,5)` · `sub_total_amount, discount_amount, net_amount Decimal(20,5)` |
| Dr | `dr_chart_of_accounts_id Uuid FK` · `dr_account_code VarChar?` · `dr_cost_center_id Uuid?` · `dr_cost_center_code VarChar?` |
| VAT | `vat_tax_profile_id Uuid?` · `vat_rate Decimal(15,5)` · `vat_amount Decimal(20,5)` · `vat_is_override Boolean` · `vat_chart_of_accounts_id Uuid?` · `vat_cost_center_id Uuid?` |
| WHT | `wht_tax_profile_id Uuid?` · `wht_rate Decimal(15,5)?` · `wht_estimate_amount Decimal(20,5)` (ไม่ post) |
| Cr | `cr_chart_of_accounts_id Uuid FK` (AP control) · `cr_account_code VarChar?` · `cr_cost_center_id Uuid?` |
| รวม | `total_amount Decimal(20,5)` = net + vat · `unpaid_amount Decimal(20,5)` · `base_sub_total_amount, base_discount_amount, base_net_amount, base_vat_amount, base_total_amount, base_unpaid_amount Decimal(20,5)` |
| อื่น | `info`, `dimension`, `doc_version`, audit cols |

**`tb_ap_invoice_detail_dimension`** — โครงเดียวกับ `tb_gl_jv_detail_dimension` โดยใช้ `ap_invoice_detail_id`

**`tb_ap_invoice_detail_source`** — ผูก GRN (ไม่มี soft delete)

| ฟิลด์ | ชนิด |
|---|---|
| ap_invoice_detail_id | Uuid FK |
| good_received_note_id | Uuid |
| grn_no | VarChar |
| good_received_note_detail_item_id | Uuid |
| matched_qty | Decimal(20,5) |
| grn_unit_price | Decimal(20,5) snapshot |
| matched_amount | Decimal(20,5) = net ของบรรทัด invoice |
| variance_amount | Decimal(20,5) = matched_amount − matched_qty × grn_unit_price (ข้อมูล ไม่ block) |

**`tb_ap_invoice_reference`** — Doc Reference หักกลบ (ไม่มี soft delete)

| ฟิลด์ | ชนิด | หมายเหตุ |
|---|---|---|
| ap_invoice_id | Uuid FK | ใบ APIV ที่หัก |
| ref_ap_invoice_id | Uuid FK | ใบ DN/CN/DP ที่ถูกอ้าง |
| ref_doc_type | enum snapshot | |
| applied_amount | Decimal(20,5) | txn currency |
| base_applied_at_invoice_rate | Decimal(20,5) | |
| base_applied_at_ref_rate | Decimal(20,5) | |
| realized_fx_amount | Decimal(20,5) | บวก = loss, ลบ = gain (เฉพาะ DP) |

**`tb_ap_invoice_tax`** — tax invoice record สำหรับ ภ.พ.30

| ฟิลด์ | ชนิด | หมายเหตุ |
|---|---|---|
| ap_invoice_id | Uuid FK | 1:1 |
| tax_invoice_no | VarChar(100) | = vendor_invoice_no |
| tax_invoice_date | Date | = invoice_date |
| vendor_tax_no, vendor_branch_no, vendor_name | VarChar | snapshot |
| base_amount, vat_amount | Decimal(20,5) | สกุล base; APCN เป็นลบ |
| vat_rate | Decimal(15,5) | |
| tax_status | `enum_ap_invoice_tax_status (pending, on_review, unclaim, confirmed, submitted, void)` | เริ่ม pending |
| expiry_claim_date | Date | invoice_date + 6 เดือน |
| filing_month, filing_year | Int? | Tax module ตั้งค่าเมื่อยื่น |

**`tb_ap_payment`**

| กลุ่ม | ฟิลด์ |
|---|---|
| เอกสาร | `doc_no VarChar(30)` unique · `doc_status enum_ap_payment_status (draft, in_review, posted, void)` · `payment_date Date` (วันที่ลง GL) · `paid_date Date?` (วันเงินออกจริง) · `description VarChar?` · `reference_no VarChar(100)?` |
| vendor / ผู้รับเงิน | `vendor_id, vendor_name` · `payee_name VarChar` · `payee_tax_no VarChar(20)?` · `payee_branch_no VarChar(5)?` · `payee_address VarChar?` (snapshot จาก `tb_vendor` + `tb_vendor_address` ประเภท register_address) |
| การจ่าย | `bank_account_id Uuid FK` · `bank_account_code, bank_account_name VarChar` · `payment_method enum_ap_payment_method (bank_transfer, cheque, cash, credit_card, promptpay)` · `cheque_no VarChar(50)?` · `cheque_date Date?` |
| เงิน | `currency_id, currency_code` · `exchange_rate Decimal(15,5)` · `base_currency_id, base_currency_code` · `total_applied_amount, total_wht_amount, total_expense_amount, net_paid_amount Decimal(20,5)` (net_paid = applied − wht + expense) · `base_*` ของทั้ง 4 · `realized_fx_amount Decimal(20,5)` |
| GL / void / workflow / อื่น | เหมือน `tb_ap_invoice` |

**`tb_ap_payment_detail`** — allocation ระดับบรรทัด (ไม่มี soft delete)

| ฟิลด์ | ชนิด | หมายเหตุ |
|---|---|---|
| ap_payment_id | Uuid FK | |
| ap_invoice_id | Uuid FK | |
| ap_invoice_detail_id | Uuid FK | unique (ap_payment_id, ap_invoice_detail_id) |
| doc_type | enum snapshot | |
| applied_amount | Decimal(20,5) | บวกสำหรับ invoice/debit_note/deposit, **ลบ**สำหรับ credit_note |
| invoice_exchange_rate | Decimal(15,5) | snapshot |
| base_applied_at_invoice_rate | Decimal(20,5) | |
| base_applied_at_payment_rate | Decimal(20,5) | |
| realized_fx_amount | Decimal(20,5) | |

**`tb_ap_payment_wht`** (ไม่มี soft delete)

| ฟิลด์ | ชนิด |
|---|---|
| ap_payment_id | Uuid FK |
| wht_tax_profile_id | Uuid FK |
| wht_tax_profile_name | VarChar |
| pnd_form | enum snapshot |
| income_type | VarChar |
| base_amount | Decimal(20,5) |
| wht_rate | Decimal(15,5) |
| wht_amount | Decimal(20,5) |
| is_override | Boolean |
| chart_of_accounts_id | Uuid (บัญชี WHT payable ที่ใช้จริง) |

**`tb_ap_payment_expense`** (ไม่มี soft delete)

| ฟิลด์ | ชนิด |
|---|---|
| ap_payment_id | Uuid FK |
| chart_of_accounts_id | Uuid FK |
| cost_center_id | Uuid? |
| description | VarChar? |
| amount, base_amount | Decimal(20,5) |

---

## 5. Foundation behaviors

### 5.1 Dimension

- CRUD ผ่าน `gl-dimension`, `gl-dimension-value`, `gl-account-dimension-rule` (RPC contract แยก 3 ตัว)
- deactivate/delete dimension หรือ value ถูก guard ถ้ามีการใช้ใน `tb_gl_jv_detail_dimension` หรือ
  `tb_ap_invoice_detail_dimension` ของเอกสารที่ไม่ void → error `GL_DIMENSION_IN_USE`
- `GlDimensionValidator.validateLines(tx, lines, jvDate)` ใช้ร่วมกันโดย facade และ AP validation:
  1. โหลด rule ของทุกบัญชีในบรรทัด
  2. mandatory ต้องมี value → `GL_DIMENSION_REQUIRED` (บอก account code + dimension code)
  3. prohibited ต้องไม่มี → `GL_DIMENSION_PROHIBITED`
  4. value ต้อง active, เป็นของ dimension นั้น, และ `jv_date` อยู่ในช่วง effective → `GL_DIMENSION_VALUE_INVALID`
- `GlJvService.createInternal` รับ `dimensions?[]` ต่อบรรทัดและเขียน `tb_gl_jv_detail_dimension` พร้อม snapshot code
- JV ที่สร้างด้วยมือผ่าน `gl-jv.create/update` รับ dimension ได้ด้วยและถูก validate ด้วยกฎเดียวกันตอน submit/post (เพิ่มใน `validateJvLines`)

### 5.2 Subledger posting facade (`GlSubledgerPostingService`)

```ts
interface ISubledgerPostingLine {
  sequence_no: number;
  chart_of_accounts_id: string;
  cost_center_id?: string | null;
  debit?: Decimal; credit?: Decimal;            // txn currency, ใส่ได้ด้านเดียว
  base_debit?: Decimal; base_credit?: Decimal;  // base currency
  currency_id?: string; exchange_rate?: Decimal; // default จาก header
  description?: string | null;
  dimensions?: { gl_dimension_id: string; gl_dimension_value_id: string }[];
}
interface ISubledgerPostingInput {
  source: enum_gl_jv_source;                    // 'ap'
  source_ref_type: 'ap_invoice' | 'ap_payment'; // ขยายเป็น union ในอนาคต
  source_ref_id: string;
  jv_date: Date;
  description: string;
  prefix_id?: string | null;                    // default gl_setting.ap_jv_prefix_id → auto_jv_prefix_id
  currency_id: string; exchange_rate: Decimal;
  lines: ISubledgerPostingLine[];
}
postFromSource(input, tx): Promise<Result<{ jv_id: string; jv_no: string; already_posted: boolean }>>
reverseBySource(ref: { source_ref_type; source_ref_id }, reversalDate: Date, reason: string, tx)
  : Promise<Result<{ jv_id: string; jv_no: string }>>
```

ลำดับใน `postFromSource`

1. หา JV เดิมที่ `(source, source_ref_type, source_ref_id)` และ `jv_status <> void` ถ้ามี → คืน `{ already_posted: true }` ไม่สร้างซ้ำ (partial unique index เป็น guard ชั้น DB; unique violation แปลงเป็นผลลัพธ์เดียวกัน)
2. resolve period จาก `jv_date` (ตรรกะเดียวกับ `resolvePeriod` + `allow_post_to_closed_period`) ไม่เปิด → `GL_PERIOD_NOT_OPEN`
3. `GlDimensionValidator.validateLines`
4. ตรวจสมดุล Σdebit = Σcredit ทั้ง txn และ base ไม่สมดุล → `GL_JV_NOT_BALANCED` ผู้เรียกต้องส่งบรรทัดที่สมดุลมาเอง facade ไม่เติมบรรทัด rounding
5. `GlJvService.createInternal(data, tx, { source, source_ref_type, source_ref_id })` → โหลด header ที่เพิ่งสร้าง + `gl_setting` → `GlPostingService.postInTx(tx, header, setting, null)`
6. คืน `jv_id`, `jv_no`

`reverseBySource`: หา JV ต้นทางที่ posted → `GlPostingService.reverseInTx(tx, jvId, reversalDate, true, { source_ref_type, source_ref_id })`
ใบ reversal มี `source = reversal` และ `source_ref_*` เดิม จึงไม่ชน idempotency index

### 5.3 Refactor GL posting (behavior-preserving)

- `post(jvId, postAt)` → โหลด header + setting นอก tx เหมือนเดิม แล้วเรียก `postInTx(tx, header, setting, postAt)` ภายใน `$transaction` ของตัวเอง
- `postInTx` เก็บลำดับเดิมทุกขั้น: `resolvePeriod` → `lockFiscalYear` เป็น statement แรก → `validateJvLines` → เขียน status/period/balance
- `reverse(jvId, jvDate, canPost)` → `reverseInTx(tx, ..., meta?)` เพิ่ม param `meta` เพื่อส่ง `source_ref_type/id` ลงใบ reversal (เดิมไม่มี)
- `voidJv`, `closeYear`, `runDueJobs` ไม่แตะ
- ต้องเทียบ behavior ด้วยการรัน flow gl-jv เดิม (create → submit → approve → reverse → void) ผ่าน Bruno หลัง refactor

### 5.4 Tax profile และ vendor

- `tax_type` default `vat` ทำให้ profile เดิมทั้งหมดยังเป็น VAT; AP เลือก VAT profile จาก `tax_type = vat` และ WHT profile จาก `tax_type = wht` เท่านั้น
- `chart_of_accounts_id` บน profile เป็น default ของบรรทัด VAT/WHT (แก้ได้ที่บรรทัด) ถ้าว่างใช้ `gl_setting.input_vat_account_id` / `wht_payable_account_id`
- vendor: field ใหม่ทั้งหมด nullable; AP ใช้เป็น default ตอนเลือก vendor, ถ้าว่างใช้ base currency, credit term 0 วัน, `gl_setting.ap_control_account_id`

### 5.5 Bank account

CRUD ปกติ; `is_active = false` หรือ delete ถูก guard ถ้ามี `tb_ap_payment` ที่ไม่ void อ้างอยู่ → `BANK_ACCOUNT_IN_USE`

---

## 6. AP Invoice

### 6.1 doc type

| doc_type | prefix | ผลต่อหนี้ | JV (ต่อบรรทัด) |
|---|---|---|---|
| invoice | APIV | +หนี้ | Dr `dr_account` net · Dr `vat_account` vat · Cr `cr_account` total |
| debit_note | APDN | +หนี้ | เหมือน invoice |
| credit_note | APCN | −หนี้ | Dr `cr_account` total · Cr `dr_account` net · Cr `vat_account` vat |
| deposit | APDP | ตั้งเงินมัดจำ | Dr `advance_deposit_account` (จาก `gl_setting`) net · Cr `cr_account` net (ไม่มี VAT บรรทัด) |

### 6.2 Status และ action

`doc_status`: `draft → in_review → posted → void` (`draft → void` ได้ตรง)

| action | จาก → ไป | ทำอะไร |
|---|---|---|
| create / update / delete | draft | validate ครบ; `doc_no` ออกตอน create; delete = soft delete เฉพาะ draft |
| submit | draft → in_review | validate + ตรวจ period เปิดสำหรับ `doc_date` + สร้าง `tb_ap_invoice_tax` (ถ้ามี VAT) + hook `budgetCheck()` (no-op ใน phase นี้); ถ้าไม่มี workflow `ap_invoice` active ของ BU → post ทันที (branch เดียวกับ gl-jv) |
| approve | in_review → in_review / posted | ผ่าน orchestrator; post เมื่อ `isFinalApproval` |
| review (send back) | in_review → draft | `buildReviewWorkflow` ไป stage ผู้สร้าง |
| reject | in_review → draft | `buildRejectWorkflow`, `last_action = rejected` (ต่างจาก FRD ที่ไป Void; ผู้ใช้ตกลงแล้ว) |
| void | draft → void / posted → void | posted: ต้องไม่มี `tb_ap_payment_detail` ในใบที่ไม่ void และ `reference_applied_amount = 0`; `reverseBySource` ลงวันที่ void; tax record → void; คืน matched qty (ลบ `tb_ap_invoice_detail_source`); ลบ `tb_ap_invoice_reference` ที่ใบนี้อ้างออกและคืน `reference_applied_amount` ให้ใบที่ถูกอ้าง |

การ post (ใน `postInvoice(tx)`) = สร้างบรรทัด JV ตาม §6.1 + บรรทัด reference (§6.6) → `postFromSource` →
ตั้ง `doc_status = posted`, `gl_jv_id`, `posted_at`, `unpaid_amount = total_amount` ต่อบรรทัด (credit_note ก็บวก),
`outstanding_amount = Σ unpaid` ทั้งหมดใน tx เดียว

### 6.3 กฎ header

- vendor ต้อง active; เลือกแล้ว default: `currency` (vendor.default_currency → base), `credit_term_days`, `cr_chart_of_accounts_id` ของบรรทัดใหม่ (vendor.ap_chart_of_accounts → `gl_setting.ap_control_account_id`)
- `exchange_rate` default จาก `ExchangeRateService.findByDateAndCurrency(doc_date, currency_code)` แก้ได้ ต้อง > 0; base currency = `tb_business_unit.default_currency_id`
- `due_date = invoice_date + credit_term_days` คำนวณฝั่ง server ทุกครั้งที่บันทึก
- `vendor_invoice_no` ห้ามซ้ำใน vendor เดียวกันในใบที่ไม่ void → `AP_VENDOR_INVOICE_NO_DUPLICATE`
- `doc_date` ต้องอยู่ใน GL period เปิด ตรวจตอน submit และตอน post (period อาจปิดระหว่างรออนุมัติ)
- Dr/Cr account ต้อง postable (`is_active` และ type ไม่ใช่ header/summary) และถ้า `is_require_cost_center` ต้องมี cost center; `use_in` ของ COA ที่มี `ap` ใช้เป็น filter ฝั่ง lookup เท่านั้น ไม่ block

### 6.4 การคำนวณบรรทัด

Decimal ทั้งหมด (`Prisma.Decimal`), ปัด HALF_UP 2 ตำแหน่งที่ระดับบรรทัด, header = Σ บรรทัด

```text
sub_total_amount    = quantity × unit_price
net_amount          = sub_total_amount − discount_amount            (discount ≤ sub_total)
vat_amount          = round(net_amount × vat_rate / 100)           ยกเว้น vat_is_override → ใช้ค่าที่ส่งมา
wht_estimate_amount = round(net_amount × wht_rate / 100)           แสดงเท่านั้น
total_amount        = net_amount + vat_amount
base_x              = round(x × exchange_rate)                     ทุก x ข้างต้น
```

`quantity > 0`, `unit_price ≥ 0`; deposit ต้องมีบรรทัดอย่างน้อย 1 และห้ามมี VAT profile

### 6.5 สร้างจาก GRN (`doc_source = grn`)

- `grn-candidates(vendor_id, currency_id)`: GRN `doc_status = committed`, `post_type = ap`, vendor + currency ตรง, และมี item ที่ `received_qty − Σ matched_qty (ในใบที่ไม่ void) > 0`
- `create-from-grn({ grn_item_ids[] , header overrides })`: สร้าง draft พร้อมบรรทัดต่อ item: `quantity = remaining qty`, `unit_price = received_price`, VAT จาก `tax_profile_id/tax_rate` ของ item, `dr_chart_of_accounts_id` จาก `tb_product_account_code_mapping` ของสินค้า (ไล่ product → item group → sub category → category) ถ้าไม่พบให้ว่างไว้ ผู้ใช้เลือกก่อน submit; `tb_ap_invoice_detail_source` 1 แถวต่อบรรทัด
- ตอน update/submit ตรวจ `matched_qty ≤ remaining` ใหม่ทุกครั้ง → `AP_GRN_OVER_MATCHED`; GRN ต้องยัง committed → `AP_GRN_NOT_COMMITTED`
- `variance_amount` เก็บเป็นข้อมูล ไม่มี tolerance block
- `tb_good_received_note` ไม่ถูกแก้ไขใด ๆ

### 6.6 Doc Reference (หักกลบ)

- ใช้กับ `doc_type = invoice` เท่านั้น อ้าง DN/CN/DP ของ vendor + currency เดียวกันที่ **posted** (เข้มกว่า FRD; ผู้ใช้ตกลงแล้ว) และ `total_amount − reference_applied_amount > 0`
- `applied_amount ≤ remaining` ของใบอ้าง → `AP_REFERENCE_OVER_APPLIED`; `Σ applied ≤ total_amount` ของ invoice
- เมื่อ invoice post: เพิ่ม `reference_applied_amount` ของใบอ้าง; DN/CN ที่ถูกอ้างจนหมดไม่ต้องจ่ายผ่าน PV อีก (`unpaid_amount` ของบรรทัดใบอ้างลดตามสัดส่วน); DP ที่ถูกอ้าง `unpaid_amount` ลดเช่นกัน
- GL เพิ่มเฉพาะ reference ถึง **deposit**:
  - Dr `cr_account` ของ invoice = applied (base ที่ invoice rate)
  - Cr `advance_deposit_account` = applied (base ที่ deposit rate)
  - ผลต่าง base → Dr `realized_fx_loss_account` หรือ Cr `realized_fx_gain_account` เป็นบรรทัดสกุล base (rate 1)
- reference ถึง DN/CN **ไม่มี GL** เพราะ DN/CN ปรับ AP ไปแล้วตอน posted (ต่างจาก FRD §6.1 ที่เขียน Part 2 รวม ๆ; ผู้ใช้ตกลงแล้ว)

### 6.7 Tax invoice record

สร้างตอน submit ถ้ามีบรรทัด `vat_amount ≠ 0`: `tax_invoice_no = vendor_invoice_no`, `tax_invoice_date = invoice_date`,
`expiry_claim_date = invoice_date + 6 เดือน`, `base_amount = Σ base_net`, `vat_amount = Σ base_vat`, snapshot vendor tax no/branch,
`tax_status = pending`; credit_note บันทึกยอดลบ; แก้ invoice หลัง send-back → คำนวณ record ใหม่; void → `tax_status = void`

### 6.8 Numbering และ attachments

- `generateApDocNo({ doc_type, doc_date })` เป็น free function แบบ `gl-jv.running-code.ts`: type `AP-IV/DN/CN/DP`, last no จาก `tb_ap_invoice` ที่ `doc_no startsWith prefix+yyMM` ใน tx เดียวกับ create; unique index บน `doc_no` เป็น guard
- attachments: JSON array ตาม pattern เดิม gateway แปลง fileToken → url ผ่าน `micro-file`; ขนาดไฟล์บังคับที่ `micro-file`

---

## 7. AP Payment

### 7.1 กฎ

- ทุกบรรทัด allocation ต้อง vendor และ currency เดียวกับ PV → `AP_VENDOR_CURRENCY_MISMATCH`
- invoice ที่ allocate ต้อง `posted`; `applied_amount ≤ unpaid_amount − Σ applied ใน PV อื่นที่ไม่ void` ต่อบรรทัด → `AP_PAYMENT_OVER_ALLOCATED` (จองยอดตั้งแต่ draft)
- credit_note allocate เป็นค่าลบ; `total_applied_amount = Σ applied` ต้อง > 0 → `AP_PAYMENT_EMPTY`
- bank account active และ `currency_code` = PV currency หรือ = base → `AP_PAYMENT_BANK_CURRENCY_MISMATCH`
- `payment_date` อยู่ใน period เปิด ตรวจตอน submit และ post; `exchange_rate` default จาก `findByDateAndCurrency(payment_date)`
- `net_paid_amount = total_applied − total_wht + total_expense` และ base เช่นกัน

### 7.2 WHT

- `wht-preview(allocations[])`: สำหรับบรรทัด invoice ที่มี `wht_tax_profile_id` คำนวณ `base = applied_amount × (net_amount / total_amount)` (ตัด VAT ออก), `wht = round(base × rate/100)`, group ตาม profile → รายการ `tb_ap_payment_wht` default; credit_note ให้ค่าลบ
- ผู้ใช้แก้ `wht_amount` ได้ (`is_override = true`); ผลรวมต้อง ≥ 0
- บัญชี: `tax_profile.chart_of_accounts_id` → `gl_setting.wht_payable_account_id`
- snapshot ผู้รับเงินบน header ตอน submit จาก `tb_vendor` (tax_no, branch_no) และ `tb_vendor_address` ประเภท `register_address`

### 7.3 Lifecycle

เหมือน invoice (`draft → in_review → posted → void`) ผ่าน workflow type `ap_payment`; void หลัง post → `reverseBySource` +
คืน `unpaid_amount` ทุกบรรทัดที่ allocate + ล้าง `outstanding` ที่เกี่ยวข้อง ใน tx เดียว

### 7.4 ผลหลัง post

`tb_ap_invoice_detail.unpaid_amount −= applied` (และ `base_unpaid`), `tb_ap_invoice.outstanding_amount = Σ unpaid`;
`findOne` ของ PV คืน `tax_invoices[]` จาก `tb_ap_invoice_tax` ของทุก invoice ที่ allocate (panel "Tax Invoices Received in this Payment")

---

## 8. รูป JV ทั้งหมด

| เอกสาร | บรรทัด | txn | base |
|---|---|---|---|
| APIV / APDN | Dr `dr_account` (+cc, dims) | net | net × rate |
| | Dr `vat_account` | vat | vat × rate |
| | Cr `cr_account` (AP control) | total | total × rate |
| APCN | กลับด้าน APIV | | |
| APDP | Dr `advance_deposit_account` | net | net × rate |
| | Cr `cr_account` | net | net × rate |
| APIV อ้าง APDP | Dr `cr_account` | applied | applied × invoice_rate |
| | Cr `advance_deposit_account` | applied | applied × deposit_rate |
| | Dr/Cr realized FX (สกุล base) | diff | diff |
| APPV | Dr `cr_account` ของแต่ละ invoice (รวมต่อบัญชี) | applied | applied × invoice_rate |
| | Cr `bank_account.chart_of_accounts_id` | net_paid | net_paid × payment_rate |
| | Cr WHT payable (ต่อ profile) | wht | wht × payment_rate |
| | Dr expense (ต่อบรรทัด) | amount | amount × payment_rate |
| | Dr/Cr realized FX (สกุล base) | diff ที่เหลือ | ทำ base สมดุล |
| void ใด ๆ | reversal ของ JV ต้นทาง ลงวันที่ void | | |

ทุก JV: `source = ap`, `source_ref_type = ap_invoice | ap_payment`, `source_ref_id = id`, prefix `gl_setting.ap_jv_prefix_id`,
`description = "<doc_no> <vendor_name>"`, dimension ของบรรทัด AP copy ลง `tb_gl_jv_detail_dimension`

---

## 9. Contract, gateway, permission

### 9.1 RPC contract (`packages/rpc-contract/src/contracts/`)

| contract | actions |
|---|---|
| `ap-invoice` | find-all, find-one, create, update, delete, submit, approve, reject, review, void, grn-candidates, create-from-grn, reference-candidates |
| `ap-payment` | find-all, find-one, create, update, delete, submit, approve, reject, review, void, outstanding-documents, wht-preview |
| `gl-dimension` / `gl-dimension-value` / `gl-account-dimension-rule` | find-all, find-one, create, update, delete |
| `bank-account` | find-all, find-one, create, update, delete |

pattern `'<service>.<kebab-action>'` สร้างผ่าน temp literal → `bun run gen:rpc-contract` → แทนที่; `bun run audit:tcp-drift` ต้องผ่าน

### 9.2 REST (gateway)

- `api/:bu_code/ap-invoice` และ `api/:bu_code/ap-payment`: `GET /`, `GET /:id`, `POST /`, `PUT /:id`, `DELETE /:id`, `POST /:id/submit|approve|reject|review|void`, `GET /grn-candidates`, `POST /from-grn`, `GET /:id/reference-candidates`, `GET /outstanding-documents`, `POST /wht-preview`
- master: config module `config_gl-dimensions`, `config_gl-dimension-values`, `config_gl-account-dimension-rules`, `config_bank-accounts` ตาม pattern `config_*` เดิม
- payload ทุกตัวมี `user_id`, `bu_code`, `version`; action ที่แก้สถานะส่ง `doc_version`
- Swagger DTO ด้วย Zod + `createZodDto`, enum ด้วย `z.nativeEnum(enum_x)` + `@ApiProperty({ enum, enumName })`

### 9.3 Permission

- submit/approve/reject/review: ไม่มี `@Permission` (workflow engine ตัดสิน) ใช้ `KeycloakGuard` + `AppIdGuard('ap-invoice.<action>')`
- void: `@Permission({ 'accounting.ap': ['void'] })`
- master CRUD: ตาม config module เดิม
- `bun run audit:api-system-permission`, `audit:guard-providers`, `audit:bu-scope-guard` ต้องผ่าน

---

## 10. Error catalog (`packages/error-catalog/src/catalog.ts`, bilingual, module-scoped id)

| code | เมื่อ |
|---|---|
| AP_INVOICE_NOT_FOUND / AP_PAYMENT_NOT_FOUND | |
| AP_INVOICE_IMMUTABLE / AP_PAYMENT_IMMUTABLE | แก้/ลบ/ทำ action ที่สถานะไม่อนุญาต หรือ doc_version ไม่ตรง |
| AP_VENDOR_INVOICE_NO_DUPLICATE | §6.3 |
| AP_VENDOR_CURRENCY_MISMATCH | reference หรือ allocation ข้าม vendor/currency |
| AP_GRN_NOT_COMMITTED / AP_GRN_OVER_MATCHED | §6.5 |
| AP_REFERENCE_NOT_POSTED / AP_REFERENCE_OVER_APPLIED | §6.6 |
| AP_INVOICE_HAS_PAYMENT / AP_INVOICE_IS_REFERENCED | กัน void |
| AP_PAYMENT_OVER_ALLOCATED / AP_PAYMENT_EMPTY / AP_PAYMENT_BANK_CURRENCY_MISMATCH | §7.1 |
| AP_ACCOUNT_NOT_POSTABLE / AP_COST_CENTER_REQUIRED | §6.3 |
| GL_DIMENSION_REQUIRED / GL_DIMENSION_PROHIBITED / GL_DIMENSION_VALUE_INVALID / GL_DIMENSION_IN_USE | §5.1 |
| GL_SETTING_ACCOUNT_MISSING | key ใน `gl_setting` ที่ต้องใช้ว่าง (params: key) |
| BANK_ACCOUNT_IN_USE | §5.5 |

ใช้ของเดิม: `GL_PERIOD_NOT_OPEN`, `GL_JV_NOT_BALANCED`, `GL_JV_PREFIX_NOT_FOUND`

---

## 11. Concurrency, audit, transaction

- `doc_version` optimistic ทุก update/action (`where: { id, doc_version }` ไม่เจอ → `*_IMMUTABLE`)
- post ทั้งหมดอยู่ใน `$transaction` เดียว: lock บรรทัด AP ที่เกี่ยวข้อง → validate → `postFromSource` (ภายในมี `lockFiscalYear` advisory lock) → update สถานะ AP
- allocation: `SELECT ... FOR UPDATE` บน `tb_ap_invoice_detail` ที่ allocate ก่อนตรวจ `unpaid` ทั้งตอน submit และ post
- running number: lookup last no + insert ใน tx เดียว; unique index บน `doc_no` เป็น guard สุดท้าย (retry 1 ครั้งเมื่อชน)
- audit: Prisma audit extension ครอบตารางใหม่อัตโนมัติ (ห้ามใส่ใน `excludeModels`); ลงทะเบียน command ของ ap-invoice/ap-payment ใน `activity-registry.ts` (create/update/submit/approve/reject/review/void) ด้วย `entityName: 'ap_invoice' | 'ap_payment'`
- ทุก handler ผ่าน `this.ctx.run(payload, ...)` ของ `TenantContextRunner` (`audit:tenant-context` ต้องผ่าน)

---

## 12. Seed, config, migration

- migration ของ tenant schema ผ่าน `prisma migrate dev` ใน `packages/prisma-shared-schema-tenant`; SQL เพิ่มเติมสำหรับ partial unique index (`gljvheader_source_ref_u`, `ap_invoice_vendor_invoice_no_u`, `ap_invoice_doc_no_u`, `ap_payment_doc_no_u`) และ CHECK `debit xor credit` ไม่ต้องเพิ่มเพราะ AP ไม่เขียน `tb_gl_jv_detail` ตรง
- seed: `tb_gl_dimension` 7 แถว; `tb_config_running_code` 5 type; ไม่ seed `gl_setting` (ต้องตั้งเอง; ระบบแจ้ง `GL_SETTING_ACCOUNT_MISSING` ตอน post)
- deploy ต่อ BU ผ่าน `POST /api-system/tenant/migrations/:bu_id/deploy`
- env ใหม่: ไม่มี

---

## 13. การตรวจสอบ

ไม่เขียน test อัตโนมัติในการ execute plan (ตาม preference ของผู้ใช้) ใช้ static check และตรวจด้วยมือ

- `bun run build:package` → `bun run check-types` → eslint แบบ scoped (`npx eslint --no-fix <files>`) → `bun run gates`
- Bruno collection ใน `carmen-turborepo-backend-bruno` เพิ่ม folder `ap-invoice`, `ap-payment`, `gl-dimension`, `bank-account`
- ลำดับตรวจด้วยมือบน BU ทดสอบ
  1. ตั้ง `gl_setting` key ใหม่, สร้าง dimension value, rule mandatory 1 บัญชี, bank account, tax profile WHT 3%
  2. รัน flow gl-jv เดิม (create → submit → approve → reverse → void) ยืนยัน refactor ไม่เปลี่ยน behavior
  3. invoice จาก GRN committed → submit โดยขาด dimension mandatory → ต้องได้ `GL_DIMENSION_REQUIRED`
  4. เติม dimension → approve จนสุด → ตรวจ `tb_gl_jv_header` (source ap, source_ref), `tb_gl_jv_detail_dimension`, `tb_gl_balance`, `tb_ap_invoice_tax`
  5. submit ซ้ำ/approve ซ้ำ → ต้องไม่เกิด JV ใบที่สอง
  6. APDP → post → PV จ่าย DP → APIV อ้าง DP ต่างสกุล → ตรวจบรรทัด realized FX
  7. PV จ่ายบางบรรทัด + WHT + ค่าธรรมเนียม → post → ตรวจ `unpaid_amount`, `outstanding_amount`, JV
  8. void PV → ตรวจ reversal JV และ `unpaid` คืน; void invoice ที่มี PV → ต้องได้ `AP_INVOICE_HAS_PAYMENT`
  9. ปิด period แล้ว approve ใบที่ค้าง → ต้องได้ `GL_PERIOD_NOT_OPEN`

---

## 14. จุดที่เบี่ยงจาก FRD/PRD (ผู้ใช้ตกลงแล้ว)

| เรื่อง | FRD/PRD | spec นี้ |
|---|---|---|
| service | `micro-accounting` แยก, TCP | ใน `micro-business`, HTTP-as-RPC |
| reject | → Void/Rejected | → draft, `last_action = rejected` |
| reference | อ้างใบ submitted/approved ได้ | เฉพาะ posted |
| GL ของ reference DN/CN | Part 2 reversal | ไม่มี GL (netting ระดับ subledger) |
| status จ่ายแล้ว | ไม่ระบุ | derive จาก unpaid ไม่ใช่ status |
| WHT | tax2 บนบรรทัด | ประมาณการบนบรรทัด หักจริงตอนจ่าย |
| GRN/PO | free text | structured link + matched qty |
| dimension บนบรรทัด | 6 ช่องคงที่ | data-driven 7 ตัว (seed) |

---

## 15. งานต่อเนื่องหลัง spec นี้

1. แก้ README, `PRD-accounting-system-overview.md` §2, และ `prisma/schema.prisma` ใน repo นี้ให้ตรงกับการตัดสินใจ (commit แยก)
2. เขียน ADR ปิด `accounting-foundation.md` §13 ข้อ 3 (backend location) และ 6 (tax profile extend)
3. sub-project ถัดไปตามลำดับ: Tax (WHT certificate, ภ.ง.ด., ภ.พ.30) → Cash & Bank → FA → AR → Period End → Budget → IC → Reporting
4. GRNI/inventory posting เมื่อต้องการให้ GRN post GL
