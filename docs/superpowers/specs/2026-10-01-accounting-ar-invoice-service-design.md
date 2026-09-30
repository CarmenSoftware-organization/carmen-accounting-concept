# Design — Customer Master + AR Invoice Service (ARIV / ARDN / ARCN)

**วันที่:** 2026-10-01
**สถานะ:** รอ review
**repo ที่แก้:** `carmen-turborepo-backend-v2` (`apps/micro-business`, `apps/backend-gateway`, `packages/*`) branch `feature/accounting-ar-invoice-service` จาก `main`
**ต่อจาก:** [2026-09-30-accounting-ar-schema-design.md](./2026-09-30-accounting-ar-schema-design.md) (schema merged, backend-v2 PR #701 `32c0fbd29`) และ [2026-09-23-accounting-foundation-ap-design.md](./2026-09-23-accounting-foundation-ap-design.md) (pattern AP)
**แหล่งอ้างอิง:** `docs/PRD-module-ar.md` v1.1 (FRD v1.07), review `docs/reviews/2026-09-28-ar-frd-v1.07-review.md`

---

## 1. ที่มาและการตัดสินใจหลัก

review FRD v1.07 ระบุว่า OI-1..3 ต้องได้คำตอบจาก BA ก่อน implement AR แต่ OI-1/OI-2 (FX และ VAT ของมัดจำ) มีผลแค่กับ ARDP และการตัดมัดจำ ส่วน OI-3 schema ตัดสินไปแล้วตามแบบ AP spec นี้จึงทำเฉพาะส่วนที่ไม่ติด BA

| หัวข้อ | ตัดสิน (ผู้ใช้ยืนยัน 2026-10-01) | เหตุผล |
|---|---|---|
| ขอบเขต | customer master + ARIV/ARDN/ARCN แบบ manual | ไม่ติด OI-1/2; ARDP, Receipt, PMS folio แยกเป็น spec ถัดไป |
| หักกลบ CN | ARIV อ้าง ARCN ผ่าน Doc Reference แบบ AP ที่ implement จริง (rate ต้องเท่ากัน, ลงเฉพาะคู่บรรทัดเศษปัดสตางค์) | ยังไม่มี Receipt จึงต้องมีทางให้ CN ลดยอดค้างของ invoice ที่เจาะจง |
| เลขใบกำกับภาษี | ออกตอน post (อนุมัติขั้นสุดท้าย) | send back / reject ก่อน post จะไม่เผาเลข เลขใบกำกับต่อเนื่อง |
| แนวทางโค้ด | module ใหม่ตามแบบ AP (แนวทาง A) ไม่ extract แกนร่วม | AR ต่างจาก AP มาก (tax2, discount %, Dr/Cr กลับข้าง, เลขใบกำกับของเราเอง) และ AP ยังรอ verify บน BU จริง; พิจารณา extract เมื่อมีสองตัวอย่างที่รันจริงแล้ว |

จุดที่ตัดสินแทน BA อยู่ใน §13 เพื่อส่งให้ BA ดู

---

## 2. ขอบเขต

### ในขอบเขต

- Customer master: CRUD `tb_customer` + `tb_customer_address` (micro-business, RPC, gateway config, permission)
- AR invoice 3 ประเภทใน module เดียว: Invoice (ARIV), Debit Note (ARDN), Credit Note (ARCN) — create/update/delete/submit/approve/reject/review/void, find-all/find-one, reference-candidates
- Doc Reference: ARIV หักกลบ ARCN ที่ posted
- ใบกำกับภาษี TXIV/TXDN/TXCN ออกตอน post
- แก้ facade `GlSubledgerPostingService` ให้รองรับ `ar_invoice` (AP behavior ไม่เปลี่ยน)
- migration ALTER enum 2 ค่า, key `gl_setting` 3 ตัว, running code 6 type, error catalog, activity registry, Bruno

### นอกขอบเขต

ARDP และการตัดมัดจำ (รอ OI-1/2) · Receipt / ARRC / TXRC · PMS folio / Add Folio · source `copy`, `ai` · VAT แบบ inclusive · aging report query · credit limit check · Print · frontend · การแก้ AP นอกเหนือจาก facade

ผลที่ยอมรับ: จนกว่าจะมี Receipt `unpaid_amount` ลดได้ทางเดียวคือผ่าน CN reference invoice จึงยังปิดยอดไม่ได้ แต่ aging ต่อใบถูกต้อง

---

## 3. ตำแหน่งในโค้ด

```text
apps/micro-business/src/
├── gl/gl-subledger-posting/     ← แก้: SubledgerSourceRefType + prefix ตาม source
├── master/customers/            ← ใหม่: controller, service, module, interface, dto (แบบ master/vendors)
└── ar/ar-invoice/               ← ใหม่: controller, service, logic, validation, writer, posting,
                                    running-code, workflow/ar-invoice-workflow.mapper, dto/serializer, interface
apps/backend-gateway/src/
├── application/ar-invoice/      ← ใหม่ (แบบ application/ap-invoice)
└── config/config_customers/     ← ใหม่ (แบบ config/config_vendors)
packages/
├── rpc-contract/src/contracts/  ← customer, ar-invoice
├── error-catalog/src/catalog.ts ← CUSTOMER_*, AR_*
└── prisma-shared-schema-tenant/prisma/migrations/<ts>_accounting_ar_service_enums/
```

ทุก feature เป็น flat `@Module` import ตรงใน `apps/micro-business/src/app.module.ts` เหมือน `ApInvoiceModule`

---

## 4. Facade และ foundation ที่แก้

- `gl-subledger-posting.interface.ts`: `SubledgerSourceRefType = 'ap_invoice' | 'ap_payment' | 'ar_invoice'`
- `postFromSource`: prefix เริ่มต้นเลือกตาม `input.source` — `ap` → `ap_jv_prefix_id`, `ar` → `ar_jv_prefix_id` แล้ว fallback `auto_jv_prefix_id`; error `GL_SETTING_ACCOUNT_MISSING` ส่ง `key` ตาม source ที่ใช้ ลำดับอื่นทั้งหมดไม่เปลี่ยน
- ใช้ key เดิม `realized_fx_gain_account_id` / `realized_fx_loss_account_id` สำหรับเศษปัด CN (§9)
- `app-config.service.ts` `GlSettingSchema` เพิ่ม `ar_control_account_id`, `output_vat_account_id`, `ar_jv_prefix_id` (uuid optional)
- migration `accounting_ar_service_enums` (สร้างแบบ AR schema spec §5 บน scratch DB, ห้าม `migrate dev`):
  - `ALTER TYPE "enum_workflow_type" ADD VALUE 'ar_invoice';`
  - `ALTER TYPE "enum_ar_tax_invoice_status" ADD VALUE 'void';`
- running code seed: `AR-IV`, `AR-DN`, `AR-CN` (เลขเอกสาร `ARIV{YY}{MM}{0000}` ฯลฯ) และ `TX-IV`, `TX-DN`, `TX-CN` (เลขใบกำกับ `TXIV{YY}{MM}{0000}` ฯลฯ)
- ไม่ seed `gl_setting` (ระบบแจ้ง `GL_SETTING_ACCOUNT_MISSING` ตอน post)

---

## 5. Customer master

**ตำแหน่ง:** `master/customers/` · gateway `config_customers` route `api/config/:bu_code/customers` (`GET /`, `GET /:customer_id`, `POST /`, `PUT /:customer_id`, `DELETE /:customer_id`) · RPC `customer`: find-all, find-one, find-all-by-id, create, update, delete

**กฎ**

- `code`, `name` บังคับ; `code` ห้ามซ้ำในแถวที่ไม่ถูกลบ → `CUSTOMER_CODE_DUPLICATE` (unique `customer_code_u` เป็น guard สุดท้าย)
- default ตรวจแบบ `VendorsService.resolveApDefaults`: `default_currency_id` → snapshot `default_currency_code` (`CURRENCY_NOT_FOUND`); `credit_term_id` → snapshot ชื่อ + วัน (`CREDIT_TERM_NOT_FOUND`); `ar_chart_of_accounts_id` ต้อง postable (`GL_ACCOUNT_NOT_POSTABLE`); `tax_profile_id` ต้องมีและ active
- `tax_no` ถ้ามีต้องเป็นตัวเลข 13 หลัก, `branch_no` ถ้ามีต้องเป็นตัวเลข 5 หลัก → `CUSTOMER_TAX_NO_INVALID`; ว่างได้ (ลูกค้าต่างประเทศ)
- address เป็น array ใน payload เดียวกับ customer (add / update / remove แบบ vendor); `address_type` ละ 1 แถว (unique `customeraddress_type_u`)
- `credit_limit` เก็บอย่างเดียว ไม่บังคับ
- delete = soft delete + `is_active = false`; ถ้ามี `tb_ar_invoice` ที่ `doc_status <> void` และไม่ถูกลบอ้างถึง → `CUSTOMER_IN_USE` (เลิกใช้ให้ปิด `is_active`)
- find-all: paginate / search (code, name, tax_no) / filter `is_active` แบบ vendor
- permission: pattern เดียวกับ `config_vendors` resource `config.customer`
- activity registry `entityName: 'customer'` (create/update/delete)

---

## 6. AR invoice — สถานะและ action

`doc_status`: `draft → in_review → posted → void` (`draft → void` ได้ตรง) ผ่าน `WorkflowOrchestratorService` type `ar_invoice`

| action | จาก → ไป | ทำอะไร |
|---|---|---|
| create / update / delete | draft | validate §7–§9; `doc_no` ออกตอน create; delete = soft delete เฉพาะ draft |
| submit | draft → in_review | validate + period ของ `doc_date` เปิด (ERR_AR_001) + ตรวจ mixed VAT rate (§10.2); ไม่มี workflow `ar_invoice` active ของ BU → post ทันที (branch เดียวกับ AP) |
| approve | in_review → in_review / posted | post เมื่อ `isFinalApproval` |
| review (send back) | in_review → draft | `buildReviewWorkflow` ไป stage ผู้สร้าง |
| reject | in_review → draft | `buildRejectWorkflow`, `last_action = rejected` |
| void | draft → void / posted → void | §11 |

**post (`postInvoice(tx)`, tx เดียว):** ตรวจ period อีกครั้ง → lock ARIV และ ARCN ที่อ้าง (§9) → สร้างบรรทัด JV (§8) → `postFromSource` → ออกเลขใบกำกับ + `tb_ar_tax_invoice` ถ้า `is_tax_invoice` (§10) → apply reference (§9) → `doc_status = posted`, `gl_jv_id`, `gl_jv_no`, `posted_at`, `posted_by_id`, `unpaid_amount = total_amount` ต่อบรรทัด (หักส่วนที่ reference หักไป), `outstanding_amount = Σ unpaid`

---

## 7. กฎ header และบรรทัด

### 7.1 Header

- customer ต้องมีและ `is_active` → `CUSTOMER_INACTIVE`; เลือกแล้ว default `currency` (customer.default_currency → base), `credit_term_*`, `dr_chart_of_accounts_id` ของบรรทัดใหม่ (customer.ar_chart_of_accounts → `gl_setting.ar_control_account_id`); snapshot `customer_code`, `customer_name`
- `exchange_rate` default `ExchangeRateService.findByDateAndCurrency(doc_date, currency_code)` แก้ได้ ต้อง > 0; base currency = `tb_business_unit.default_currency_id`
- `due_date = doc_date + credit_term_days` คำนวณฝั่ง server ทุกครั้งที่บันทึก
- `is_wht_recorded` / `wht_amount` เก็บเป็นข้อมูลอ้างอิง ไม่มีผล GL (OI-6); `wht_amount ≥ 0`
- `doc_source` ต้องเป็น `manual` และ `is_pms_folio = false` ทั้ง header และบรรทัด → `AR_SOURCE_NOT_SUPPORTED`
- header totals = Σ บรรทัด

### 7.2 การคำนวณบรรทัด

Decimal ทั้งหมด (`Prisma.Decimal`), `round2` HALF_UP แบบ `ap-invoice.logic.ts`

```text
sub_total_amount = round(quantity × unit_price)
discount_amount  = discount_is_override ? round(ค่าที่ส่งมา) : round(sub_total × discount_pct / 100)
net_amount       = sub_total − discount
vat_amount       = vat_is_override  ? round(ค่าที่ส่งมา) : round(net × vat_rate / 100)
tax2_amount      = tax2_is_override ? round(ค่าที่ส่งมา) : round(net × tax2_rate / 100)
total_amount     = net + vat + tax2
base_x           = round(x × exchange_rate)  สำหรับ sub_total, discount, net, vat, tax2
base_total       = base_net + base_vat + base_tax2
```

- `quantity > 0`, `unit_price ≥ 0`; `0 ≤ discount_pct ≤ 100` และ `0 ≤ discount ≤ sub_total` → `AR_DISCOUNT_INVALID` (ERR_AR_005 — ปฏิเสธ ไม่ clamp ที่ server)
- `vat_rate` / `tax2_rate` มาจาก profile ของบรรทัด; ไม่มี profile → rate 0
- VAT เป็น exclusive เท่านั้น
- tax2 profile ต้อง `tax_type = vat` ถ้าเป็น `wht` → `AR_TAX2_WHT_NOT_ALLOWED` (OI-6); vat profile ต้อง `tax_type = vat` ด้วย
- บัญชี VAT: `vat_chart_of_accounts_id` → profile.`chart_of_accounts_id` → `gl_setting.output_vat_account_id`; tax2: `tax2_chart_of_accounts_id` → profile.`chart_of_accounts_id` ไม่มี fallback; ขาดเมื่อยอด ≠ 0 → `GL_SETTING_ACCOUNT_MISSING` (key ที่เกี่ยวข้อง)
- `cr_chart_of_accounts_id`, `cr_cost_center_id`, `dr_chart_of_accounts_id` บังคับ; บัญชีทุกตัวที่จะลง JV ต้อง postable → `AR_ACCOUNT_NOT_POSTABLE` และถ้าบัญชี `is_require_cost_center` ต้องมี cost center → `AR_COST_CENTER_REQUIRED`
- dimension ตรวจด้วย `GlDimensionValidator.validateLines` (เขียน `tb_ar_invoice_detail_dimension`)
- ต้องมีอย่างน้อย 1 บรรทัดและ `total_amount > 0` ตอน submit

---

## 8. รูป JV

`source = ar`, `source_ref_type = ar_invoice`, `source_ref_id = id`, prefix `gl_setting.ar_jv_prefix_id`, `jv_date = doc_date`, `description = "<doc_no> <customer_name>"`

| เอกสาร | บรรทัด | txn | base |
|---|---|---|---|
| ARIV / ARDN | Dr `dr_account` (AR, `dr_cost_center`) | total | base_total |
| | Cr `cr_account` (revenue, `cr_cost_center`, dims) | net | base_net |
| | Cr VAT account (`vat_cost_center`) | vat | base_vat |
| | Cr tax2 account (`tax2_cost_center`) | tax2 | base_tax2 |
| ARCN | กลับข้าง ARIV ทุกบรรทัด | | |
| void | reversal ของ JV ต้นทางลงวันที่ void | | |

- บรรทัดที่ยอด 0 ไม่ส่ง; dimension ของบรรทัด AR copy ไปบรรทัด revenue
- base สมดุลโดยโครงสร้าง (base_total = ผลรวมส่วนที่ปัดแล้ว) facade ยังตรวจ `GL_JV_NOT_BALANCED`
- reference ถึง ARCN ไม่มีบรรทัดตัดยอด มีเฉพาะคู่บรรทัดเศษปัดสกุลฐาน (§9)

---

## 9. Doc Reference — ARIV หักกลบ ARCN

ทำตาม `applyReferences` / `planUnpaidReduction` / `buildCreditNoteRoundingJvLines` / void restore ใน `ap-invoice.posting.ts` + `ap-invoice.logic.ts` (พฤติกรรม AP ที่ implement จริง ซึ่งละเอียดกว่า AP spec §6.6)

- ใช้ได้เฉพาะ `doc_type = invoice`; ใบที่อ้างต้องเป็น `credit_note` → อื่นๆ `AR_REFERENCE_TYPE_NOT_ALLOWED`; ต้อง `posted` และไม่ถูกลบ → `AR_REFERENCE_NOT_POSTED`; ลูกค้าและสกุลเงินเดียวกัน → `AR_CUSTOMER_CURRENCY_MISMATCH` (ERR_AR_004)
- `exchange_rate` ของ CN ต้องเท่ากับของ ARIV พอดี → `AR_REFERENCE_RATE_MISMATCH` (CN อัตราต่างให้ไปหักใน Receipt ภายหลัง)
- ยอดที่อ้างได้ = `min(total_amount − reference_applied_amount, outstanding_amount)` ของ CN; `0 < applied ≤ ยอดที่อ้างได้` และ `Σ applied ≤ total_amount` ของ ARIV → ไม่เช่นนั้น `AR_REFERENCE_OVER_APPLIED`
- `applied_net_amount` / `applied_vat_amount` แบ่งตามสัดส่วน `(net + tax2) : vat` ของ CN เพื่อแสดงผล; `applied_amount = applied_net + applied_vat`
- ตรวจตอน create/update/submit และตรวจซ้ำหลัง lock ตอน post
- ตอน post: `SELECT ... FOR UPDATE` CN → `reference_applied_amount += applied`, ลด `unpaid` บรรทัด CN ตาม plan, คำนวณ `outstanding_amount` CN ใหม่; ARIV ลด unpaid ตามลำดับบรรทัด; เก็บ `base_applied_at_invoice_rate` / `base_applied_at_ref_rate`; `realized_fx_amount = 0`
- GL: ไม่มีบรรทัดตัดยอด แต่เพราะแต่ละฝั่งหักยอดฐานที่ปัดทีละส่วน ยอดฐานที่ลด `I` (ARIV) กับ `C` (CN) อาจต่างกันระดับสตางค์ AR เป็นบัญชีด้านเดบิตและ CN มียอดด้านเครดิต ยอดเดบิตสุทธิของ AR ใน subledger จึงขยับ `C − I` ให้ลงคู่บรรทัดสกุลฐาน (rate 1) ต่อ reference: `diff = C − I > 0` → Dr AR (บัญชี/cc ของบรรทัด CN) / Cr `realized_fx_gain_account_id`; `diff < 0` → Cr AR / Dr `realized_fx_loss_account_id`; `diff = 0` → ไม่ลง (เครื่องหมายกลับกับ AP) บัญชี FX ต้องมีเฉพาะด้านที่ใช้ ไม่งั้น `GL_SETTING_ACCOUNT_MISSING`
- `reference-candidates(customer_id, currency_id, exclude_invoice_id?)` คืน ARCN ที่ยอดที่อ้างได้ > 0

---

## 10. ใบกำกับภาษี (`tb_ar_tax_invoice`)

### 10.1 การออก

- ออกเฉพาะ `is_tax_invoice = true` ใน tx เดียวกับ post (ไม่ออกตอน save / submit)
- `tax_prefix` = `TXIV` / `TXDN` / `TXCN` ตาม doc_type; `tax_invoice_no` จาก running code `TX-*` อิง YYMM ของ `doc_date` (free function แบบ `ap-invoice.running-code.ts`)
- ชน unique `artaxinvoice_no_u` → retry 1 ครั้ง ยังชน → `AR_TAX_INVOICE_NO_CONFLICT` (ERR_AR_006) และ rollback ทั้ง tx
- `tax_invoice_date = doc_date`; `filing_month` / `filing_year` จาก `doc_date`; `tax_status = pending`
- snapshot: `customer_registered_name` = `registered_name` → `name`; `customer_tax_no`, `customer_branch_no`; ที่อยู่ `register_address` → `billing_address` (`address_line1`, `address_line2`, `province`, `postal_code`)
- ยอด THB: `base_amount = Σ base_net`, `vat_amount = Σ base_vat`, `total_amount = base + vat` (ไม่รวม tax2); ARCN บันทึกเป็นลบ
- ARCN: ถ้าผู้ใช้ส่ง `original_tax_invoice_no` จะเก็บใน `tb_ar_invoice.info.original_tax_invoice_no` (ยังไม่บังคับ)

### 10.2 กฎ

- `is_tax_invoice = true` และบรรทัดที่มี VAT มีมากกว่า 1 rate → `AR_TAX_INVOICE_MIXED_VAT_RATE` (ตรวจตอน submit); `vat_rate` = rate นั้น หรือ 0 ถ้าไม่มี VAT
- `findOne` คืน `tax_invoice` ของเอกสาร

---

## 11. Void

- draft → void: เปลี่ยนสถานะ + `void_*`
- posted → void:
  1. `reference_applied_amount = 0` → ไม่เช่นนั้น `AR_INVOICE_IS_REFERENCED`
  2. ไม่มี `tb_ar_receipt_detail` ในใบเสร็จที่ไม่ void และไม่ถูกลบ → `AR_INVOICE_HAS_RECEIPT` (ERR_AR_003)
  3. `reverseBySource` ลงวันที่ void (period ต้องเปิด → `GL_PERIOD_NOT_OPEN`) → `void_gl_jv_id`
  4. `tb_ar_tax_invoice.tax_status = void` (ไม่ลบ เลขคงอยู่)
  5. คืนยอด reference ที่ใบนี้อ้าง: CN `reference_applied_amount −= applied`, คืน unpaid บรรทัด CN, คำนวณ outstanding ใหม่ (แบบ void ของ AP)
- `void_reason` บังคับ; ทุกอย่างใน tx เดียว

---

## 12. Contract, gateway, permission, error, concurrency

### 12.1 RPC contract

| contract | actions |
|---|---|
| `customer` | find-all, find-one, find-all-by-id, create, update, delete |
| `ar-invoice` | find-all, find-one, create, update, delete, submit, approve, reject, review, void, reference-candidates |

`'<service>.<kebab-action>'` → `bun run gen:rpc-contract`; `bun run audit:tcp-drift` ต้องผ่าน

### 12.2 REST

- `api/:bu_code/ar-invoice`: `GET /`, `GET /:id`, `POST /`, `PUT /:id`, `DELETE /:id`, `POST /:id/submit|approve|reject|review|void`, `GET /reference-candidates`
- payload มี `user_id`, `bu_code`, `version`; action ที่เปลี่ยนสถานะส่ง `doc_version`
- Swagger DTO ด้วย Zod + `createZodDto`, enum `z.nativeEnum` + `@ApiProperty({ enum, enumName })`

### 12.3 Permission

- `AR_RESOURCE = 'accounting.ar'`; CRUD และ void แบบ AP (`@Permission({ 'accounting.ar': ['void'] })` สำหรับ void)
- submit / approve / reject / review ไม่มี `@Permission` ใช้ `KeycloakGuard` + `AppIdGuard('ar-invoice.<action>')`
- `audit:api-system-permission`, `audit:guard-providers`, `audit:bu-scope-guard` ต้องผ่าน

### 12.4 Error catalog (`packages/error-catalog/src/catalog.ts`, bilingual)

| code | เมื่อ |
|---|---|
| CUSTOMER_NOT_FOUND / CUSTOMER_INACTIVE | §5, §7.1 |
| CUSTOMER_CODE_DUPLICATE / CUSTOMER_TAX_NO_INVALID / CUSTOMER_IN_USE | §5 |
| AR_INVOICE_NOT_FOUND | |
| AR_INVOICE_IMMUTABLE | สถานะไม่อนุญาต หรือ `doc_version` ไม่ตรง |
| AR_SOURCE_NOT_SUPPORTED / AR_DISCOUNT_INVALID / AR_TAX2_WHT_NOT_ALLOWED | §7 |
| AR_ACCOUNT_NOT_POSTABLE / AR_COST_CENTER_REQUIRED | §7.2 |
| AR_CUSTOMER_CURRENCY_MISMATCH / AR_REFERENCE_TYPE_NOT_ALLOWED / AR_REFERENCE_NOT_POSTED / AR_REFERENCE_RATE_MISMATCH / AR_REFERENCE_OVER_APPLIED | §9 |
| AR_TAX_INVOICE_NO_CONFLICT / AR_TAX_INVOICE_MIXED_VAT_RATE | §10 |
| AR_INVOICE_IS_REFERENCED / AR_INVOICE_HAS_RECEIPT | §11 |

ใช้ของเดิม: `GL_PERIOD_NOT_OPEN`, `GL_JV_NOT_BALANCED`, `GL_SETTING_ACCOUNT_MISSING`, `GL_DIMENSION_*`, `GL_ACCOUNT_NOT_POSTABLE`, `CURRENCY_NOT_FOUND`, `CREDIT_TERM_NOT_FOUND`

### 12.5 Concurrency, audit, transaction

- `doc_version` optimistic ทุก update/action (`where: { id, doc_version }` ไม่เจอ → `AR_INVOICE_IMMUTABLE`)
- post อยู่ใน `$transaction` เดียว (§6); lock ARIV + CN ก่อน validate reference
- running number (เอกสารและใบกำกับ): lookup last no + insert ใน tx เดียว; unique index เป็น guard; retry 1 ครั้ง
- activity registry `entityName: 'ar_invoice'` (create/update/delete/submit/approve/reject/review/void) — ตาราง AR อยู่ใน `excludeModels` จึงต้องลงทะเบียน ไม่เช่นนั้นไม่มี trail
- ทุก handler ผ่าน `this.ctx.run(payload, ...)` (`audit:tenant-context` ต้องผ่าน)
- `apps/micro-business/CLAUDE.md`: แก้ bullet `tb_customer*` / `tb_ar_*` (ไม่ใช่ schema-only แล้ว) + รายการ `gl_setting` key ที่ AR ต้องใช้

### 12.6 Deploy

migration ADD VALUE ปลอดภัยต่อโค้ดเดิม; หลัง merge และ Deploy Dev ยิง `POST /api-system/tenant/migrations/:bu_id/deploy` ครบทุก BU (ครอบ migration `accounting_ar_tables` ที่ยังค้างด้วย) แล้วตรวจ `prisma migrate status`

---

## 13. จุดที่เบี่ยงจาก FRD/PRD (ส่ง BA)

| เรื่อง | FRD v1.07 | spec นี้ |
|---|---|---|
| จังหวะ post (OI-3) | ขัดกันเอง | post เมื่ออนุมัติขั้นสุดท้าย |
| LOA Reject | → Void | → draft, `last_action = rejected` |
| เลขใบกำกับภาษี | ออกตอน Submit | ออกตอน post |
| `ref_doc_type` (OI-5.2) | ARDP / ARCN / ARDN | ARCN เท่านั้น (ARDP รอ OI-1/2, ARDN ไม่ใช้หักกลบ) |
| สถานะใบที่อ้างได้ (OI-5.5) | Submitted หรือ Posted | Posted เท่านั้น |
| อัตราของ CN ที่อ้าง | ไม่ระบุ | ต้องเท่ากับ invoice |
| ARCN ยอดติดลบ (OI-5.6) | ป้อนติดลบ | เก็บบวก กลับข้างบัญชีใน JV |
| Tax 2 (OI-6) | ภาษีเสริมหรือ WHT | ภาษีเสริมเท่านั้น (profile `vat`) |
| VAT07_INC | มี | ยังไม่รองรับ (tax profile ไม่มี flag inclusive) |
| Discount เกิน (ERR_AR_005) | clamp + highlight | server ปฏิเสธ (clamp เป็นหน้าที่ UI) |
| เลขใบกำกับเดิมของใบลดหนี้ | ไม่ระบุ | เก็บใน `info.original_tax_invoice_no` ไม่บังคับ — ขอ BA ยืนยันว่าต้องเป็น column บังคับหรือไม่ |
| Source | Manual / PMS Folio / Copy / AI | Manual เท่านั้นในรอบนี้ |

---

## 14. การตรวจสอบ

ไม่เขียน test อัตโนมัติใหม่ (preference ของผู้ใช้) ใช้ static check + suite เดิม + ตรวจด้วยมือ

- `bun run build:package` → `bun run check-types` → eslint scoped (`npx eslint --no-fix <files>`) → `bun run gates`
- suite เดิมของ micro-business และ backend-gateway ต้องผ่าน (facade ถูกแก้)
- migration: diff บน scratch DB มีเฉพาะ `ALTER TYPE ... ADD VALUE` 2 บรรทัด; หลัง deploy `migrate diff --exit-code` = 0
- Bruno (`carmen-turborepo-backend-bruno`): folder `customer`, `ar-invoice`
- ตรวจด้วยมือบน BU ทดสอบ:
  1. ตั้ง `gl_setting` (`ar_control_account_id`, `output_vat_account_id`, `ar_jv_prefix_id`), tax profile VAT 7% และ tax2 (vat type), customer พร้อม register address
  2. post AP invoice เดิม 1 ใบ → JV ใช้ prefix `ap_jv_prefix_id` เหมือนเดิม
  3. ARIV มี discount %, VAT, tax2, `is_tax_invoice` → submit → send back → submit → approve จนสุด → ตรวจ JV (`source = ar`, source_ref, dims), `tb_ar_tax_invoice` (TXIV เลขแรกของเดือน, ยอด THB), unpaid/outstanding; ยืนยันว่า send back ไม่เผาเลข
  4. approve ซ้ำ → ไม่มี JV ใบที่สอง
  5. ARCN → post (TXCN ยอดลบ) → ARIV ใหม่อ้าง CN บางส่วน → post → ตรวจ unpaid/outstanding ทั้งสองใบ และ JV ของ ARIV ไม่มีบรรทัดตัดยอด (มีแค่คู่เศษปัดถ้า diff ≠ 0); CN อัตราต่าง → `AR_REFERENCE_RATE_MISMATCH`
  6. void ARCN ที่ถูกอ้าง → `AR_INVOICE_IS_REFERENCED`; void ARIV ข้อ 5 → reversal JV, tax_status void, CN ได้ยอดคืน; จากนั้น void ARCN ได้
  7. ปิด period แล้ว approve ใบค้าง → `GL_PERIOD_NOT_OPEN`; tax2 profile ประเภท wht → `AR_TAX2_WHT_NOT_ALLOWED`; VAT 2 rate + tax invoice → `AR_TAX_INVOICE_MIXED_VAT_RATE`; delete customer ที่มีใบค้าง → `CUSTOMER_IN_USE`

---

## 15. งานต่อเนื่อง

1. concept repo หลัง merge: README แถว AR → "Customer + Invoice/DN/CN", PRD AR อ้าง spec นี้
2. ส่ง §13 ให้ BA พร้อม OI-1/2/4
3. spec ถัดไป: AR Receipt (ARRC + TXRC + WHT รับ) → ARDP + ตัดมัดจำ (หลัง OI-1/2) → PMS folio (หลังมี interface) → aging report
4. พิจารณา extract แกนร่วม AP/AR หลังทั้งสองรันบน BU จริง
