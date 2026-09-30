# Customer Master + AR Invoice Service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** เพิ่ม customer master และเอกสาร AR invoice / debit note / credit note (post GL, ใบกำกับภาษี, หักกลบ CN) ใน `micro-business` + gateway ของ `carmen-turborepo-backend-v2` บนตาราง AR ที่ merge แล้ว (PR #701)

**Architecture:** module ใหม่แบบ flat `@Module` ตามแบบ AP (`src/master/customers/`, `src/ar/ar-invoice/`) ไม่ extract แกนร่วมจาก AP — ฟังก์ชัน pure ของ AP (`round2`, `computeDueDate`, `planUnpaidReduction`, `planSequentialReductions`, `deductPlan`, `plannedBase`, `mergePlans`, `UnpaidPlan`, `IUnpaidLine`) import ใช้ตรงจาก `@/ap/ap-invoice/ap-invoice.logic` ได้ แต่**ห้ามแก้ไฟล์ AP** · AR post ผ่าน `GlSubledgerPostingService.postFromSource()` ซึ่งแก้ให้เลือก prefix ตาม `source` · workflow ใช้ `WorkflowOrchestratorService` type ใหม่ `ar_invoice`

**Tech Stack:** NestJS 11, Bun, Turborepo, Prisma 7 (`packages/prisma-shared-schema-tenant`), Zod v4 + `nestjs-zod`, `@repo/rpc-contract` (generated), `@repo/error-catalog`, PostgreSQL

**Spec:** `docs/superpowers/specs/2026-10-01-accounting-ar-invoice-service-design.md` (repo `carmen-accounting-concept`) — plan อ้าง §ของ spec ตลอด อ่านคู่กัน

**Repo ที่แก้:** `/Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2` — path ด้านล่างสัมพัทธ์กับ root นี้ เว้นแต่ระบุ

## Global Constraints

- **ไม่มี test step** (preference ผู้ใช้ `~/.claude/CLAUDE.md`): ทุก task จบด้วย `bun run build:package` (ถ้าแตะ `packages/*`) → `bun run check-types` → `npx eslint --no-fix <ไฟล์ที่แก้>` → commit; ห้ามสร้าง `*.spec.ts`; suite เดิมต้องผ่านก่อน merge (Task 11)
- Branch `feature/accounting-ar-invoice-service` จาก `main` (repo นี้ integrate บน `main`; ห้ามแตะ `prod`)
- **ห้ามแก้ไฟล์ใต้ `apps/micro-business/src/ap/`** — import pure function ได้ แก้ไม่ได้
- RPC contract: handler ใช้ literal `@MessagePattern({ cmd, service: 'micro-business' })` ชั่วคราว → `bun run gen:rpc-contract` → แทนด้วย `<Service>.<action>.pattern`; ห้ามแก้ `packages/rpc-contract/src/contracts/*.ts` ด้วยมือ
- enum มาจาก Prisma เท่านั้น: Zod `z.nativeEnum(enum_x)` ห้าม `z.enum([...])`
- naming: class PascalCase, ไฟล์ kebab-case, snake_case เฉพาะ DB/wire, ฟังก์ชัน ≤ 20 statements, boolean `is/has/can`, JSDoc สองภาษา (EN บรรทัดแรก TH บรรทัดถัดไป) ทุก export — ตามไฟล์ AP ที่เป็นแม่แบบ
- เงินเป็น `Prisma.Decimal` เสมอ ปัดด้วย `round2` (HALF_UP 2 ตำแหน่ง)
- ทุก RPC handler ห่อ `this.ctx.run(payload, () => ...)`
- gateway: submit/approve/review/reject ไม่มี `@Permission`; void ใช้ `@Permission({ 'accounting.ar': ['void'] })` (resource `accounting.ar` จองไว้แล้วใน `packages/prisma-shared-schema-platform/prisma/*` — ไม่แก้ seed เหมือนที่ AP ไม่ได้แก้ใน `d7fb723d2`)
- ห้าม `bun run lint` ระดับ root; ห้าม `prisma migrate dev`; ทุกคำสั่ง prisma ที่แตะ DB ต้องนำหน้าด้วย `DATABASE_URL=<scratch>` (package `.env` ชี้ CARMEN_TENANT จริง); psql ไม่รับ `?schema=` ใน URI
- ห้าม merge / push / เปิด PR / deploy migration ลง BU จริงโดยไม่ถามผู้ใช้

## Review Focus

ไม่มี test อัตโนมัติ จึงระบุ task ที่ต้องกันในโค้ดและตรวจด้วยมือ:

1. **ARCN ถูกอ้างโดย ARIV draft สองใบแล้ว approve ติดกัน** ต้องไม่ทำให้ CN ติดลบ → Task 8 lock CN `FOR UPDATE` แล้วเรียก `arReferenceViolation` กับแถวที่อ่านใหม่หลัง lock (`AR_REFERENCE_OVER_APPLIED`); ตรวจมือ Task 11
2. **ใบที่ถูก send back หลังเคย submit** ต้องไม่ได้เลขใบกำกับ; post เท่านั้นที่ออก และออกครั้งเดียว → Task 8 `issueTaxInvoice` เรียกใน `postInTx` เท่านั้น และข้ามถ้ามีแถว `tb_ar_tax_invoice` แล้ว
3. **base ไม่สมดุลเพราะเศษปัด** (rate 35.12345, หลายบรรทัด, มี tax2) → Task 5 `computeArLine` คิด `base_total = base_net + base_vat + base_tax2` และ JV Dr AR ใช้ `base_total` ต่อบรรทัด; ตรวจมือ Task 11
4. **เปลี่ยน customer/currency/rate ตอน update draft ที่มี reference** → Task 7 validate reference ใหม่ทุกครั้งที่ update และตอน submit
5. **AP regression จากการแก้ facade / GlSetting** (AP JV ต้องยังใช้ `ap_jv_prefix_id` และ error key เดิม) → Task 1 รัน test เดิมของ facade + AP และ Task 11 ตรวจมือ + suite เต็ม

---

### Task 0: Branch, baseline, MODULE id ที่ว่าง

**Files:** ไม่แก้ไฟล์

- [ ] **Step 1: แตก branch และตรวจ baseline**

```bash
cd /Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2
git checkout main && git pull
git checkout -b feature/accounting-ar-invoice-service
bun install && bun run build:package && bun run check-types
```
Expected: ผ่านทั้งหมด ถ้าไม่ผ่านหยุดและรายงาน

- [ ] **Step 2: ยืนยัน MODULE id ที่จะใช้ยังว่าง**

```bash
grep -n "517\|622" packages/error-catalog/src/module.ts
```
Expected: ไม่มีผล (จะใช้ `CUSTOMER: 517`, `AR_INVOICE: 622`) ถ้ามี ให้ใช้เลขว่างถัดไปในหมวดเดียวกันแทนทุกที่ใน Task 2

- [ ] **Step 3: ยืนยันว่ายังไม่มีโค้ด AR**

```bash
ls apps/micro-business/src/ar apps/micro-business/src/master/customers 2>&1 | head -2
grep -rln "tb_ar_invoice\|tb_customer" apps/micro-business/src | grep -v enrichment | head
```
Expected: `No such file or directory` และไม่มีไฟล์ (ยกเว้น enrichment map) ถ้ามี หยุดและรายงาน

---

### Task 1: Foundation — migration enum, gl_setting keys, running code, facade

**Files:**
- Create: `packages/prisma-shared-schema-tenant/prisma/migrations/<ts>_accounting_ar_service_enums/migration.sql`
- Modify: `packages/prisma-shared-schema-tenant/prisma/schema.prisma` (enum `enum_workflow_type`, `enum_ar_tax_invoice_status`)
- Modify: `apps/micro-business/src/app-config/app-config.service.ts:86-101` (`GlSettingSchema`)
- Modify: `apps/micro-business/src/gl/gl-posting/gl-posting.service.ts:85-103` (`GlSetting`)
- Modify: `apps/micro-business/src/gl/gl-subledger-posting/interface/gl-subledger-posting.interface.ts:8`
- Modify: `apps/micro-business/src/gl/gl-subledger-posting/gl-subledger-posting.service.ts:118-124`
- Modify: `apps/micro-business/src/master/running-code/const/running-code.const.ts:126`, `apps/micro-business/src/procurement/const/procurement.const.ts:78` (บรรทัด `'AP-IV'` เป็นจุดอ้างอิง)

**Interfaces:**
- Produces: `enum_workflow_type.ar_invoice`, `enum_ar_tax_invoice_status.void`; `GlSetting.ar_control_account_id | output_vat_account_id | ar_jv_prefix_id`; `SubledgerSourceRefType` มี `'ar_invoice'`; running-code type `AR-IV`, `AR-DN`, `AR-CN`, `TX-IV`, `TX-DN`, `TX-CN`

- [ ] **Step 1: เพิ่มค่า enum ใน schema.prisma**

```prisma
enum enum_workflow_type {
  purchase_request
  store_requisition
  purchase_order
  gl_jv
  ap_invoice
  ap_payment
  ar_invoice
}
```
```prisma
enum enum_ar_tax_invoice_status {
  pending
  confirmed
  submitted
  void
}
```
(คง comment เดิมเหนือ enum ไว้ ถ้ามี)

- [ ] **Step 2: สร้าง migration บน scratch DB (ตาม AR schema spec §5)**

```bash
cd packages/prisma-shared-schema-tenant
PGADMIN="postgresql://<user>:<pass>@localhost:5432/postgres"      # พิมพ์เองทั้งเส้น ห้ามคัดจาก .env
SCRATCH="postgresql://<user>:<pass>@localhost:5432/ar_svc_scratch"
psql "$PGADMIN" -c "CREATE DATABASE ar_svc_scratch"
psql "$SCRATCH" -c "SELECT current_database(), current_schema()"   # ต้องได้ ar_svc_scratch | public
DATABASE_URL="$SCRATCH" bunx prisma migrate deploy
TS=$(date -u +%Y%m%d%H%M%S); mkdir -p prisma/migrations/${TS}_accounting_ar_service_enums
DATABASE_URL="$SCRATCH" bunx prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma --script > prisma/migrations/${TS}_accounting_ar_service_enums/migration.sql
cat prisma/migrations/${TS}_accounting_ar_service_enums/migration.sql
```
Expected: มีแค่ 2 statement (อาจมี comment):
```sql
ALTER TYPE "enum_workflow_type" ADD VALUE 'ar_invoice';
ALTER TYPE "enum_ar_tax_invoice_status" ADD VALUE 'void';
```
ถ้ามี `DROP` / `ALTER TABLE` อื่น → scratch หรือ schema เพี้ยน หยุดและรายงาน (drift ที่รู้อยู่แล้ว: `DROP INDEX "PO_delivery_point_id_idx"` — ถ้าโผล่ ให้ลบบรรทัดนั้นออกจาก migration นี้และแจ้งผู้ใช้ ไม่แก้ในงานนี้)

- [ ] **Step 3: ยืนยัน migration ลงแล้วไม่มี diff แล้วลบ scratch**

```bash
DATABASE_URL="$SCRATCH" bunx prisma migrate deploy
DATABASE_URL="$SCRATCH" bunx prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma --exit-code; echo "exit=$?"
psql "$PGADMIN" -c "DROP DATABASE ar_svc_scratch"
bun run db:generate && cd ../..
```
Expected: `exit=0` (ถ้าไม่เป็น 0 เพราะ drift `PO_delivery_point_id_idx` ที่รู้อยู่ ให้บันทึกไว้และไปต่อ)

- [ ] **Step 4: เพิ่ม gl_setting keys**

`app-config.service.ts` ใน `GlSettingSchema` ต่อท้าย `ap_jv_prefix_id`:
```ts
  // AR posting accounts (AR invoice service spec §4)
  ar_control_account_id: z.string().uuid().optional(),
  output_vat_account_id: z.string().uuid().optional(),
  ar_jv_prefix_id: z.string().uuid().optional(),
```
`gl-posting.service.ts` ใน `interface GlSetting` ต่อท้าย `ap_jv_prefix_id?: string;`:
```ts
  ar_control_account_id?: string;
  output_vat_account_id?: string;
  ar_jv_prefix_id?: string;
```
แก้ JSDoc เหนือ interface: EN "The `ap_*` / `ar_*` / VAT / WHT / FX keys are the control accounts and prefixes the AP and AR modules post with." TH "คีย์ `ap_*` / `ar_*` / VAT / WHT / FX คือบัญชีคุมและ prefix ที่โมดูล AP และ AR ใช้ลงบัญชี"

- [ ] **Step 5: แก้ facade ให้รองรับ AR**

`gl-subledger-posting.interface.ts:8`:
```ts
export type SubledgerSourceRefType = 'ap_invoice' | 'ap_payment' | 'ar_invoice';
```
`gl-subledger-posting.service.ts` เพิ่ม constant ระดับไฟล์ก่อน `class GlSubledgerPostingService` (import `enum_gl_jv_source` จาก `@repo/prisma-shared-schema-tenant` ถ้ายังไม่มี):
```ts
/**
 * Default JV prefix key per subledger source; AP keeps `ap_jv_prefix_id` as before
 * key ของ prefix JV เริ่มต้นต่อ source ของ subledger; AP ยังใช้ `ap_jv_prefix_id` เหมือนเดิม
 */
const SOURCE_PREFIX_KEY: Partial<Record<enum_gl_jv_source, 'ap_jv_prefix_id' | 'ar_jv_prefix_id'>> = {
  [enum_gl_jv_source.ap]: 'ap_jv_prefix_id',
  [enum_gl_jv_source.ar]: 'ar_jv_prefix_id',
};
```
แทนบรรทัด 118–124 (`const setting = ...` ถึงปิดบล็อก `if (!prefixId)`):
```ts
    const setting = await this.postingService.getSetting();
    const prefixKey = SOURCE_PREFIX_KEY[input.source] ?? 'ap_jv_prefix_id';
    const prefixId = input.prefix_id ?? setting?.[prefixKey] ?? setting?.auto_jv_prefix_id;
    if (!prefixId) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_SETTING_ACCOUNT_MISSING, {
        key: prefixKey,
      });
    }
```
แก้ JSDoc ของ `postFromSource` ส่วนที่บอกว่า default prefix คือ `ap_jv_prefix_id` เป็น "per-source key (`ap_jv_prefix_id` / `ar_jv_prefix_id`) then `auto_jv_prefix_id`" ทั้ง EN/TH

- [ ] **Step 6: เพิ่ม running code 6 type**

ทั้งสองไฟล์ ต่อจากบล็อก `'AP-*'` (รูปแบบตรงกับ `'AP-IV'`):
```ts
  'AR-IV': { config: { A: 'ARIV', B: `date('yyMM')`, C: `running(4, '0')`, format: '{A}{B}{C}' } },
  'AR-DN': { config: { A: 'ARDN', B: `date('yyMM')`, C: `running(4, '0')`, format: '{A}{B}{C}' } },
  'AR-CN': { config: { A: 'ARCN', B: `date('yyMM')`, C: `running(4, '0')`, format: '{A}{B}{C}' } },
  'TX-IV': { config: { A: 'TXIV', B: `date('yyMM')`, C: `running(4, '0')`, format: '{A}{B}{C}' } },
  'TX-DN': { config: { A: 'TXDN', B: `date('yyMM')`, C: `running(4, '0')`, format: '{A}{B}{C}' } },
  'TX-CN': { config: { A: 'TXCN', B: `date('yyMM')`, C: `running(4, '0')`, format: '{A}{B}{C}' } },
```
แล้ว `grep -rn "'AP-IV'" apps/micro-business/src` — ถ้ามีไฟล์อื่นที่ list type ไว้ (union, array) ให้เพิ่ม 6 ค่าเดียวกันที่นั่นด้วย

- [ ] **Step 7: static check + test เดิมของ facade/AP**

```bash
bun run build:package && bun run check-types
npx eslint --no-fix apps/micro-business/src/gl/gl-subledger-posting/gl-subledger-posting.service.ts apps/micro-business/src/gl/gl-subledger-posting/interface/gl-subledger-posting.interface.ts apps/micro-business/src/app-config/app-config.service.ts apps/micro-business/src/gl/gl-posting/gl-posting.service.ts apps/micro-business/src/master/running-code/const/running-code.const.ts apps/micro-business/src/procurement/const/procurement.const.ts
cd apps/micro-business && bun test src/gl/gl-subledger-posting src/ap src/app-config 2>&1 | tail -5; cd ../..
```
Expected: ผ่าน; test เดิมผ่านทั้งหมด (รันของเดิม ไม่เขียนใหม่) ถ้า test เดิมของ facade พังเพราะ mock `getSetting` ให้แก้โค้ดให้ AP path คืนผลเดิม ไม่แก้ test

- [ ] **Step 8: Commit**

```bash
git add packages/prisma-shared-schema-tenant/prisma apps/micro-business/src/app-config apps/micro-business/src/gl apps/micro-business/src/master/running-code apps/micro-business/src/procurement/const
git commit -m "feat(ar): foundation for AR invoice — workflow/tax enums, gl_setting keys, running codes, per-source JV prefix"
```

---

### Task 2: Error catalog

**Files:**
- Modify: `packages/error-catalog/src/module.ts` (`CUSTOMER: 517` ต่อ `SHELF: 516`; `AR_INVOICE: 622` ต่อ `AP_PAYMENT: 621`)
- Modify: `packages/error-catalog/src/catalog.ts` (ต่อท้ายกลุ่ม `AP_PAYMENT_*`)

**Interfaces:**
- Produces: `ERROR_CATALOG.<CODE>` ทุกตัวด้านล่าง

- [ ] **Step 1: เพิ่ม MODULE**

```ts
  SHELF: 516,
  CUSTOMER: 517,
```
```ts
  AP_PAYMENT: 621,
  AR_INVOICE: 622,
```

- [ ] **Step 2: เพิ่ม entry (รูปเดียวกับ `AP_INVOICE_NOT_FOUND`)**

id = `makeId(MODULE.<module>, <n>)`; ข้อความที่มี `{param}` ใช้รูป placeholder เดียวกับ entry เดิมที่มี params (ดู `GL_SETTING_ACCOUNT_MISSING`):

| code | module, n | http | message_en | message_th |
|---|---|---|---|---|
| CUSTOMER_NOT_FOUND | CUSTOMER, 1 | 404 | Customer not found | ไม่พบลูกค้า |
| CUSTOMER_CODE_DUPLICATE | CUSTOMER, 2 | 409 | Customer code {code} already exists | รหัสลูกค้า {code} มีอยู่แล้ว |
| CUSTOMER_TAX_NO_INVALID | CUSTOMER, 3 | 400 | Tax ID must be 13 digits and branch no 5 digits | เลขผู้เสียภาษีต้องเป็นตัวเลข 13 หลัก และเลขสาขา 5 หลัก |
| CUSTOMER_IN_USE | CUSTOMER, 4 | 409 | Customer is used by AR documents; deactivate it instead | ลูกค้ามีเอกสาร AR อ้างอยู่ ให้ปิดการใช้งานแทนการลบ |
| CUSTOMER_INACTIVE | CUSTOMER, 5 | 400 | Customer {code} is inactive | ลูกค้า {code} ถูกปิดการใช้งาน |
| AR_INVOICE_NOT_FOUND | AR_INVOICE, 1 | 404 | AR document not found | ไม่พบเอกสาร AR |
| AR_INVOICE_IMMUTABLE | AR_INVOICE, 2 | 409 | AR document cannot be changed in its current status | เอกสาร AR แก้ไขไม่ได้ในสถานะปัจจุบัน |
| AR_SOURCE_NOT_SUPPORTED | AR_INVOICE, 3 | 400 | Only manual AR invoices, debit notes and credit notes are supported | รองรับเฉพาะ invoice / debit note / credit note ที่สร้างเอง (manual) |
| AR_DISCOUNT_INVALID | AR_INVOICE, 4 | 400 | Line {sequence_no}: {reason} | บรรทัด {sequence_no}: {reason} |
| AR_TAX2_WHT_NOT_ALLOWED | AR_INVOICE, 5 | 400 | Line {sequence_no}: tax 2 cannot be a withholding-tax profile | บรรทัด {sequence_no}: ภาษีที่ 2 ใช้ profile ภาษีหัก ณ ที่จ่ายไม่ได้ |
| AR_ACCOUNT_NOT_POSTABLE | AR_INVOICE, 6 | 400 | Line {sequence_no}: account {account} is not postable | บรรทัด {sequence_no}: บัญชี {account} ลงรายการไม่ได้ |
| AR_COST_CENTER_REQUIRED | AR_INVOICE, 7 | 400 | Line {sequence_no}: account {account} requires a cost center | บรรทัด {sequence_no}: บัญชี {account} ต้องระบุ cost center |
| AR_CUSTOMER_CURRENCY_MISMATCH | AR_INVOICE, 8 | 400 | {doc_no} belongs to another customer or currency | {doc_no} เป็นของลูกค้าหรือสกุลเงินอื่น |
| AR_REFERENCE_TYPE_NOT_ALLOWED | AR_INVOICE, 9 | 400 | {doc_no} cannot be referenced; only credit notes can be applied to an invoice | อ้าง {doc_no} ไม่ได้ ใช้หักกลบกับ invoice ได้เฉพาะใบลดหนี้ |
| AR_REFERENCE_NOT_POSTED | AR_INVOICE, 10 | 400 | {doc_no} is not posted | {doc_no} ยังไม่ได้ post |
| AR_REFERENCE_RATE_MISMATCH | AR_INVOICE, 11 | 400 | {doc_no} rate differs from the invoice rate | อัตราแลกเปลี่ยนของ {doc_no} ไม่เท่ากับของ invoice |
| AR_REFERENCE_OVER_APPLIED | AR_INVOICE, 12 | 400 | Applied amount exceeds what {doc_no} or this invoice allows | ยอดที่หักเกินกว่าที่ {doc_no} หรือ invoice นี้รับได้ |
| AR_TAX_INVOICE_NO_CONFLICT | AR_INVOICE, 13 | 409 | Tax invoice number {tax_invoice_no} is already used | เลขใบกำกับภาษี {tax_invoice_no} ถูกใช้แล้ว |
| AR_TAX_INVOICE_MIXED_VAT_RATE | AR_INVOICE, 14 | 400 | A tax invoice cannot mix VAT rates | ใบกำกับภาษีหนึ่งใบมี VAT หลายอัตราไม่ได้ |
| AR_INVOICE_IS_REFERENCED | AR_INVOICE, 15 | 409 | Document is applied to another invoice; void that invoice first | เอกสารถูกนำไปหักกลบกับ invoice อื่น ให้ยกเลิก invoice นั้นก่อน |
| AR_INVOICE_HAS_RECEIPT | AR_INVOICE, 16 | 409 | Document has receipts and cannot be voided | เอกสารมีใบเสร็จแล้ว ยกเลิกไม่ได้ |
| AR_INVOICE_LINE_INVALID | AR_INVOICE, 17 | 400 | Line {sequence_no}: {reason} | บรรทัด {sequence_no}: {reason} |

`AR_INVOICE_LINE_INVALID` ใช้กับบรรทัดผิดทั่วไป (qty ≤ 0, ไม่มีบรรทัด, reason ว่าง ฯลฯ) แบบที่ AP ใช้ `AP_INVOICE_LINE_INVALID`

- [ ] **Step 3: static check + test เดิมของ catalog**

```bash
bun run build:package && bun run check-types
cd packages/error-catalog && bun test 2>&1 | tail -3; cd ../..
npx eslint --no-fix packages/error-catalog/src/module.ts packages/error-catalog/src/catalog.ts
```
Expected: ผ่าน (`catalog.spec.ts` เดิมตรวจ id ไม่ซ้ำ/module ถูก)

- [ ] **Step 4: Commit**

```bash
git add packages/error-catalog/src
git commit -m "feat(error-catalog): add CUSTOMER and AR_INVOICE error codes"
```

---

### Task 3: Customer master — micro-business

**Files:**
- Create: `apps/micro-business/src/master/customers/{customers.controller.ts,customers.service.ts,customers.module.ts,interface/customers.interface.ts,dto/customer.serializer.ts}`
- Modify: `apps/micro-business/src/app.module.ts` (import + `CustomersModule` ถัดจาก `VendorsModule` บรรทัด ~98 และ ~402)
- Modify: `apps/micro-business/src/common/activity/activity-registry.ts` (ถัดจากบล็อก `tb_vendor` บรรทัด ~497), `entity-snapshot.ts`
- Generated: `packages/rpc-contract/src/contracts/customers.ts`, `index.ts`

**Interfaces:**
- Consumes: `ERROR_CATALOG.CUSTOMER_*`, `GL_ACCOUNT_NOT_POSTABLE`, `CURRENCY_NOT_FOUND`, `CREDIT_TERM_NOT_FOUND`
- Produces: RPC `customers.find-all|find-one|find-all-by-id|create|update|delete`; `ICreateCustomer`, `IUpdateCustomer`, `ICustomerAddressInput`, `ICustomerDefaults`

แม่แบบ: `apps/micro-business/src/master/vendors/*` — copy โครง ห้าม copy ส่วน business type / certificate / contact

- [ ] **Step 1: interface**

อ่าน `apps/micro-business/src/master/vendors/interface/vendors.interface.ts` ก่อน — ถ้า vendor ส่งที่อยู่ด้วยรูปอื่น (เช่น `vendor_address: { add, update, delete }`) ให้ใช้รูปเดียวกันและตั้งชื่อ `customer_address` แทน `addresses` ด้านล่าง เพื่อให้ FE ใช้ร่วมกันได้

`interface/customers.interface.ts`:
```ts
import { enum_customer_address_type } from '@repo/prisma-shared-schema-tenant';

/** One address row in a customer payload / แถวที่อยู่หนึ่งแถวใน payload ลูกค้า */
export interface ICustomerAddressInput {
  id?: string;
  address_type: enum_customer_address_type;
  address_line1?: string | null;
  address_line2?: string | null;
  sub_district?: string | null;
  district?: string | null;
  city?: string | null;
  province?: string | null;
  postal_code?: string | null;
  country?: string | null;
  is_active?: boolean;
}

/** Create payload / payload การสร้าง */
export interface ICreateCustomer {
  code: string;
  name: string;
  description?: string | null;
  note?: string | null;
  customer_group?: { id: string; name: string }[] | null;
  registered_name?: string | null;
  tax_no?: string | null;
  branch_no?: string | null;
  tax_profile_id?: string | null;
  default_currency_id?: string | null;
  credit_term_id?: string | null;
  credit_limit?: number | string | null;
  ar_chart_of_accounts_id?: string | null;
  pms_account_code?: string | null;
  is_active?: boolean;
  addresses?: {
    add?: ICustomerAddressInput[];
    update?: (ICustomerAddressInput & { id: string })[];
    remove?: { id: string }[];
  };
}

/** Update payload / payload การแก้ไข */
export interface IUpdateCustomer extends Partial<ICreateCustomer> {
  id: string;
}

/** Defaults resolved from master data / ค่า default ที่ resolve จาก master */
export interface ICustomerDefaults {
  default_currency_id: string | null;
  default_currency_code: string | null;
  credit_term_id: string | null;
  credit_term_name: string | null;
  credit_term_days: number | null;
  ar_chart_of_accounts_id: string | null;
}
```

- [ ] **Step 2: service**

`customers.service.ts` — `class CustomersService extends TenantScopedService` แบบ `VendorsService`:

- `findOne(id)`: `tb_customer.findFirst({ where: { id, deleted_at: null }, include: { tb_customer_address: { where: { deleted_at: null } } } })` → ไม่เจอ `Result.errorFromCatalog(ERROR_CATALOG.CUSTOMER_NOT_FOUND)`; serialize ด้วย `dto/customer.serializer.ts` (Decimal `credit_limit` → string แบบ `vendor.serializer.ts`)
- `findAll(paginate)`: โครง `VendorsService.findAll` (paginate, sort, search); search fields `code`, `name`, `tax_no`; filter `is_active` ผ่าน filter ของ paginate เดิม; `deleted_at: null`
- `findAllById(ids)`: แบบ vendor
- `create(data)`:
  1. `validateTaxIds(data)`
  2. `this.resolveDefaults(data)`
  3. code ซ้ำ: `tb_customer.findFirst({ where: { code: data.code, deleted_at: null } })` → `CUSTOMER_CODE_DUPLICATE` `{ code }`
  4. `tax_profile_id` (ถ้ามี) ต้องมีอยู่, `is_active !== false`, `deleted_at: null` → ไม่เช่นนั้น `Result.error('Tax profile not found', ErrorCode.NOT_FOUND)` (รูปเดียวกับที่ vendor ตอบ reference ที่ไม่พบ)
  5. `address_type` ซ้ำใน `addresses.add` → `Result.error('Duplicate address_type', ErrorCode.INVALID_ARGUMENT)`
  6. `$transaction`: create header (`created_by_id: this.userId`, defaults จากข้อ 2) + address `add`
  7. Prisma `P2002` บน `customer_code_u` → `CUSTOMER_CODE_DUPLICATE`
  8. คืน `{ id }`
- `update(data)`: โหลดแถว (ไม่เจอ → `CUSTOMER_NOT_FOUND`) → ตรวจแบบ create เฉพาะ field ที่ส่ง (code ซ้ำยกเว้นตัวเอง) → `$transaction`: update header (`updated_by_id`, `doc_version: { increment: 1 }`) + address add/update/remove (remove = `deleted_at`, `deleted_by_id`)
- `delete(id)`: โหลดแถว → 
```ts
    const inUse = await this.prismaService.tb_ar_invoice.count({
      where: { customer_id: id, deleted_at: null, doc_status: { not: enum_ar_invoice_status.void } },
    });
    if (inUse > 0) return Result.errorFromCatalog(ERROR_CATALOG.CUSTOMER_IN_USE);
```
  แล้ว soft delete แบบ `VendorsService.delete` (`deleted_at`, `deleted_by_id`, `updated_by_id`, `is_active: false`)

helper ระดับไฟล์:
```ts
const TAX_NO_RE = /^\d{13}$/;
const BRANCH_NO_RE = /^\d{5}$/;

/**
 * Reject a tax id / branch that is present but malformed
 * ปฏิเสธเลขผู้เสียภาษี / สาขาที่ส่งมาแต่รูปแบบผิด
 * @param data - Customer payload / payload ลูกค้า
 * @returns ok or `CUSTOMER_TAX_NO_INVALID` / ok หรือ `CUSTOMER_TAX_NO_INVALID`
 */
function validateTaxIds(data: Pick<ICreateCustomer, 'tax_no' | 'branch_no'>): Result<true> {
  const isBadTax = !!data.tax_no && !TAX_NO_RE.test(data.tax_no);
  const isBadBranch = !!data.branch_no && !BRANCH_NO_RE.test(data.branch_no);
  if (isBadTax || isBadBranch) return Result.errorFromCatalog(ERROR_CATALOG.CUSTOMER_TAX_NO_INVALID);
  return Result.ok(true);
}
```
`private async resolveDefaults(data): Promise<Result<Partial<ICustomerDefaults>>>` — copy `VendorsService.resolveApDefaults` (`vendors.service.ts:48-96`) ทุกบรรทัด เปลี่ยน `ap_chart_of_accounts_id` → `ar_chart_of_accounts_id` และชนิดเป็น `ICustomerDefaults`

- [ ] **Step 3: controller + module + app.module**

`customers.controller.ts` copy `vendors.controller.ts` เปลี่ยน service; handler 6 ตัวใช้ literal ชั่วคราวบน method ที่ตรงชื่อ:
```ts
  @MessagePattern({ cmd: 'customers.find-one', service: 'micro-business' })
  @MessagePattern({ cmd: 'customers.find-all', service: 'micro-business' })
  @MessagePattern({ cmd: 'customers.find-all-by-id', service: 'micro-business' })
  @MessagePattern({ cmd: 'customers.create', service: 'micro-business' })
  @MessagePattern({ cmd: 'customers.update', service: 'micro-business' })
  @MessagePattern({ cmd: 'customers.delete', service: 'micro-business' })
```
ทุก method ห่อ `this.ctx.run(payload, ...)` เหมือน vendor; `customers.module.ts` copy `vendors.module.ts`; เพิ่ม `CustomersModule` ใน `app.module.ts` ถัดจาก `VendorsModule` ทั้ง import และ `imports: [...]`

- [ ] **Step 4: gen contract แล้วแทน literal**

```bash
bun run gen:rpc-contract
git diff --stat packages/rpc-contract
```
Expected: ไฟล์ใหม่ `packages/rpc-contract/src/contracts/customers.ts` (`export const Customers = defineService('customers', {...})`) + export ใน `index.ts`
แทน literal ทั้ง 6 ด้วย `Customers.findOne.pattern` ฯลฯ และ import `Customers` แบบที่ `vendors.controller.ts` import `Vendors`

- [ ] **Step 5: activity registry + snapshot**

`activity-registry.ts` ถัดจากบล็อก `entityName: 'tb_vendor'`:
```ts
  {
    entityName: 'tb_customer',
    mutations: [
      ['customers.create', 'create', CREATED_ID],
      ['customers.update', 'update', EDITED_ID],
      ['customers.delete', 'delete', DELETED_ID],
    ],
  },
```
`entity-snapshot.ts`: ถ้า `tb_vendor` อยู่ใน map นี้ ใส่ `tb_customer: { tb_customer_address: true },` ถัดจากมัน; ถ้าไม่ ใส่ใต้ comment ใหม่ `// accounts receivable` ถัดจากกลุ่ม `// accounts payable`

- [ ] **Step 6: static check**

```bash
bun run build:package && bun run check-types
npx eslint --no-fix apps/micro-business/src/master/customers apps/micro-business/src/app.module.ts apps/micro-business/src/common/activity/activity-registry.ts apps/micro-business/src/common/activity/entity-snapshot.ts
bun run audit:tcp-drift && bun run audit:message-pattern-literal
```
Expected: ผ่าน

- [ ] **Step 7: Commit**

```bash
git add apps/micro-business/src/master/customers apps/micro-business/src/app.module.ts apps/micro-business/src/common/activity packages/rpc-contract
git commit -m "feat(customer): customer master CRUD over RPC with activity logging"
```

---

### Task 4: Customer master — gateway

**Files:**
- Create: `apps/backend-gateway/src/config/config_customers/{config_customers.controller.ts,config_customers.service.ts,config_customers.module.ts,swagger/request.ts,swagger/response.ts}`
- Modify: `apps/backend-gateway/src/config/route-config.ts` (ถัดจาก `ConfigVendorsModule`)
- Regenerate: `apps/backend-gateway/src/platform/applications/app-api-catalog.generated.ts`

**Interfaces:**
- Consumes: `Customers.*` (Task 3)
- Produces: REST `api/config/:bu_code/customers` — `GET /`, `GET /:customer_id`, `POST /`, `PUT /:customer_id`, `DELETE /:customer_id`

แม่แบบ: `apps/backend-gateway/src/config/config_vendors/*`

- [ ] **Step 1: copy และแก้**

copy `config_vendors` → `config_customers` (ไม่เอา `*.spec.ts`) แล้วแก้:
- `ConfigVendors*` → `ConfigCustomers*`; route → `api/config/:bu_code/customers`; param `vendor_id` → `customer_id`; RPC `Vendors.*` → `Customers.*`; `ApiTags('Config: Customers')`; `AppIdGuard('vendors.<x>')` → `AppIdGuard('customers.<x>')`; `operationId` `vendors_*` → `customers_*`
- permission: ใช้ decorator แบบ config_vendors ทุกจุด — resource ของลูกค้าต้องมีอยู่แล้วใน `packages/prisma-shared-schema-platform/prisma/seed.permission.data.ts` (`grep -n "customer" <ไฟล์>`) ถ้าไม่มี resource ที่เหมาะ **หยุดและถามผู้ใช้** ห้ามแต่ง resource ใหม่
- `swagger/request.ts`: Zod ตาม `ICreateCustomer` / `IUpdateCustomer`; `address_type: z.nativeEnum(enum_customer_address_type)`; `tax_no: z.string().regex(/^\d{13}$/).nullish()`; `branch_no: z.string().regex(/^\d{5}$/).nullish()`; `credit_limit: z.union([z.number(), z.string()]).nullish()`; `createZodDto` แบบไฟล์ vendor
- `swagger/response.ts`: field ของ `tb_customer` + `tb_customer_address[]`

- [ ] **Step 2: register + regenerate catalog**

เพิ่ม `ConfigCustomersModule` ใน `route-config.ts` ถัดจาก `ConfigVendorsModule`
```bash
bun run scripts/generate-app-api-catalog/run.ts
git diff --stat apps/backend-gateway/src/platform/applications/app-api-catalog.generated.ts
```
Expected: มี entry `customers.*`

- [ ] **Step 3: static check + audits**

```bash
bun run check-types
npx eslint --no-fix apps/backend-gateway/src/config/config_customers apps/backend-gateway/src/config/route-config.ts
bun run audit:api-system-permission && bun run audit:guard-providers && bun run audit:bu-scope-guard && bun run audit:app-api-catalog-drift && bun run audit:zod-dto-openapi
```
Expected: ผ่าน

- [ ] **Step 4: Commit**

```bash
git add apps/backend-gateway/src/config apps/backend-gateway/src/platform/applications/app-api-catalog.generated.ts
git commit -m "feat(gateway): customer master config endpoints"
```

---

### Task 5: AR invoice — interface, logic, numbering, workflow mapper, serializer

**Files:**
- Create: `apps/micro-business/src/ar/ar-invoice/interface/ar-invoice.interface.ts`
- Create: `apps/micro-business/src/ar/ar-invoice/ar-invoice.logic.ts`
- Create: `apps/micro-business/src/ar/ar-invoice/ar-invoice.running-code.ts`
- Create: `apps/micro-business/src/ar/ar-invoice/workflow/ar-invoice-workflow.mapper.ts`
- Create: `apps/micro-business/src/ar/ar-invoice/dto/ar-invoice.serializer.ts`

**Interfaces:**
- Consumes: `round2` จาก `@/ap/ap-invoice/ap-invoice.logic`; `ISubledgerPostingLine`; `IDimensionRef`
- Produces: `IArInvoiceLineInput`, `IArInvoiceReferenceInput`, `ICreateArInvoice`, `IUpdateArInvoice`, `IArInvoiceAction`; `computeArLine(input, ctx): IArComputedLine`, `sumArLines`, `emptyArLine`; `IArJvSourceDetail`, `IArJvBuildContext`, `buildArInvoiceJvLines(ctx, details)`; `IArCreditNoteRounding`, `buildArCreditNoteRoundingJvLines(ctx, refs, fx, startSeq)`; `ArDocType`, `AR_DOC_TYPE_RUNNING_CODE`, `generateArDocNo(params)`, `generateArTaxInvoiceNo(params)`; `arInvoiceToWorkflowDocument(inv)`; `serializeArInvoice(doc)`

- [ ] **Step 1: interface**

```ts
import { enum_ar_invoice_doc_type, enum_ar_invoice_source } from '@repo/prisma-shared-schema-tenant';
import { IDimensionRef } from '@/gl/gl-dimension/interface/gl-dimension.interface';

/** One AR line as sent by the client / บรรทัด AR ตามที่ client ส่งมา */
export interface IArInvoiceLineInput {
  sequence_no: number;
  group_no?: number | null;
  description?: string | null;
  reference_info?: string | null;
  date_from?: Date | null;
  date_to?: Date | null;
  unit_id?: string | null;
  unit_name?: string | null;
  quantity: number | string;
  unit_price: number | string;
  discount_pct?: number | string | null;
  discount_amount?: number | string | null;
  discount_is_override?: boolean;
  cr_chart_of_accounts_id: string;
  cr_cost_center_id: string;
  vat_tax_profile_id?: string | null;
  vat_amount?: number | string | null;
  vat_is_override?: boolean;
  vat_chart_of_accounts_id?: string | null;
  vat_cost_center_id?: string | null;
  tax2_tax_profile_id?: string | null;
  tax2_amount?: number | string | null;
  tax2_is_override?: boolean;
  tax2_chart_of_accounts_id?: string | null;
  tax2_cost_center_id?: string | null;
  dr_chart_of_accounts_id?: string | null;
  dr_cost_center_id?: string | null;
  is_pms_folio?: boolean;
  dimensions?: IDimensionRef[] | null;
}

/** A credit note applied to an invoice / ใบลดหนี้ที่นำมาหักกลบกับ invoice */
export interface IArInvoiceReferenceInput {
  ref_ar_invoice_id: string;
  applied_amount: number | string;
  remarks?: string | null;
}

/** Create payload / payload การสร้าง */
export interface ICreateArInvoice {
  doc_type: enum_ar_invoice_doc_type;
  doc_source?: enum_ar_invoice_source;
  doc_date: Date;
  customer_id: string;
  credit_term_days?: number | null;
  currency_id?: string | null;
  exchange_rate?: number | string | null;
  description?: string | null;
  source_doc_ref?: string | null;
  is_pms_folio?: boolean;
  is_tax_invoice?: boolean;
  is_wht_recorded?: boolean;
  wht_amount?: number | string | null;
  original_tax_invoice_no?: string | null;
  attachments?: unknown[] | null;
  details: { add: IArInvoiceLineInput[] };
  references?: IArInvoiceReferenceInput[] | null;
}

/** Update payload — details/references replace the whole set / payload การแก้ไข — details/references แทนทั้งชุด */
export interface IUpdateArInvoice {
  doc_version: number;
  doc_date?: Date;
  customer_id?: string;
  credit_term_days?: number | null;
  currency_id?: string | null;
  exchange_rate?: number | string | null;
  description?: string | null;
  source_doc_ref?: string | null;
  is_tax_invoice?: boolean;
  is_wht_recorded?: boolean;
  wht_amount?: number | string | null;
  original_tax_invoice_no?: string | null;
  attachments?: unknown[] | null;
  details?: { add: IArInvoiceLineInput[] };
  references?: IArInvoiceReferenceInput[] | null;
}

/** Workflow / void action payload / payload ของ action workflow / void */
export interface IArInvoiceAction {
  doc_version?: number;
  reason?: string | null;
  stage?: string | null;
}
```

- [ ] **Step 2: logic — การคำนวณบรรทัด (spec §7.2)**

`ar-invoice.logic.ts` (base_net = base_sub_total − base_discount แบบ AP เพื่อให้ base ตรงกับ AP ทุกประการ):
```ts
import { enum_ar_invoice_doc_type, Prisma } from '@repo/prisma-shared-schema-tenant';
import { round2 } from '@/ap/ap-invoice/ap-invoice.logic';
import { ISubledgerPostingLine } from '@/gl/gl-subledger-posting/interface/gl-subledger-posting.interface';
import { IDimensionRef } from '@/gl/gl-dimension/interface/gl-dimension.interface';

const ZERO = new Prisma.Decimal(0);
const HUNDRED = new Prisma.Decimal(100);

/** Rates and overrides one line is computed with / อัตราและค่า override ที่ใช้คำนวณบรรทัด */
export interface IArComputeLineContext {
  exchange_rate: Prisma.Decimal;
  vat_rate: Prisma.Decimal;
  tax2_rate: Prisma.Decimal;
  discount_override: Prisma.Decimal | null;
  vat_override: Prisma.Decimal | null;
  tax2_override: Prisma.Decimal | null;
}

/** Every derived amount of an AR line / ยอดที่คำนวณได้ทั้งหมดของบรรทัด AR */
export interface IArComputedLine {
  sub_total_amount: Prisma.Decimal;
  discount_amount: Prisma.Decimal;
  net_amount: Prisma.Decimal;
  vat_amount: Prisma.Decimal;
  tax2_amount: Prisma.Decimal;
  total_amount: Prisma.Decimal;
  base_sub_total_amount: Prisma.Decimal;
  base_discount_amount: Prisma.Decimal;
  base_net_amount: Prisma.Decimal;
  base_vat_amount: Prisma.Decimal;
  base_tax2_amount: Prisma.Decimal;
  base_total_amount: Prisma.Decimal;
}

/**
 * Compute one AR line; base total is the sum of rounded base parts so Dr AR equals the Cr legs
 * คำนวณหนึ่งบรรทัด AR; base total = ผลรวมส่วน base ที่ปัดแล้ว เพื่อให้ Dr AR เท่ากับฝั่ง Cr พอดี
 * @param input - Quantity, price, discount percent / จำนวน ราคา เปอร์เซ็นต์ส่วนลด
 * @param input.quantity - Quantity / จำนวน
 * @param input.unit_price - Unit price / ราคาต่อหน่วย
 * @param input.discount_pct - Discount percent / เปอร์เซ็นต์ส่วนลด
 * @param ctx - Rates and overrides / อัตราและค่า override
 * @returns Computed amounts / ยอดที่คำนวณ
 */
export function computeArLine(
  input: { quantity: Prisma.Decimal; unit_price: Prisma.Decimal; discount_pct: Prisma.Decimal },
  ctx: IArComputeLineContext,
): IArComputedLine {
  const subTotal = round2(input.quantity.mul(input.unit_price));
  const discount = round2(ctx.discount_override ?? subTotal.mul(input.discount_pct).div(HUNDRED));
  const net = subTotal.sub(discount);
  const vat = round2(ctx.vat_override ?? net.mul(ctx.vat_rate).div(HUNDRED));
  const tax2 = round2(ctx.tax2_override ?? net.mul(ctx.tax2_rate).div(HUNDRED));
  const baseSubTotal = round2(subTotal.mul(ctx.exchange_rate));
  const baseDiscount = round2(discount.mul(ctx.exchange_rate));
  const baseNet = baseSubTotal.sub(baseDiscount);
  const baseVat = round2(vat.mul(ctx.exchange_rate));
  const baseTax2 = round2(tax2.mul(ctx.exchange_rate));
  return {
    sub_total_amount: subTotal,
    discount_amount: discount,
    net_amount: net,
    vat_amount: vat,
    tax2_amount: tax2,
    total_amount: net.add(vat).add(tax2),
    base_sub_total_amount: baseSubTotal,
    base_discount_amount: baseDiscount,
    base_net_amount: baseNet,
    base_vat_amount: baseVat,
    base_tax2_amount: baseTax2,
    base_total_amount: baseNet.add(baseVat).add(baseTax2),
  };
}

/**
 * All-zero line, the seed of `sumArLines`
 * บรรทัดศูนย์ทั้งหมด ใช้เป็นค่าตั้งต้นของ `sumArLines`
 * @returns Zero line / บรรทัดศูนย์
 */
export function emptyArLine(): IArComputedLine {
  return {
    sub_total_amount: ZERO,
    discount_amount: ZERO,
    net_amount: ZERO,
    vat_amount: ZERO,
    tax2_amount: ZERO,
    total_amount: ZERO,
    base_sub_total_amount: ZERO,
    base_discount_amount: ZERO,
    base_net_amount: ZERO,
    base_vat_amount: ZERO,
    base_tax2_amount: ZERO,
    base_total_amount: ZERO,
  };
}

/**
 * Sum computed lines into header totals
 * รวมบรรทัดเป็นยอด header
 * @param lines - Computed lines / บรรทัดที่คำนวณแล้ว
 * @returns Totals / ยอดรวม
 */
export function sumArLines(lines: IArComputedLine[]): IArComputedLine {
  const keys = Object.keys(emptyArLine()) as (keyof IArComputedLine)[];
  const totals = emptyArLine();
  for (const line of lines) for (const key of keys) totals[key] = totals[key].add(line[key]);
  return totals;
}
```

- [ ] **Step 3: logic — JV builder (spec §8) และคู่เศษปัด CN (spec §9)**

ต่อท้าย `ar-invoice.logic.ts`:
```ts
/** Detail columns the AR JV builder reads (tax accounts already resolved) / คอลัมน์บรรทัดที่ตัวสร้าง JV อ่าน (resolve บัญชีภาษีแล้ว) */
export interface IArJvSourceDetail {
  dr_chart_of_accounts_id: string;
  dr_cost_center_id: string | null;
  cr_chart_of_accounts_id: string;
  cr_cost_center_id: string;
  vat_chart_of_accounts_id: string;
  vat_cost_center_id: string | null;
  tax2_chart_of_accounts_id: string;
  tax2_cost_center_id: string | null;
  net_amount: Prisma.Decimal;
  vat_amount: Prisma.Decimal;
  tax2_amount: Prisma.Decimal;
  total_amount: Prisma.Decimal;
  base_net_amount: Prisma.Decimal;
  base_vat_amount: Prisma.Decimal;
  base_tax2_amount: Prisma.Decimal;
  base_total_amount: Prisma.Decimal;
  description: string | null;
  dimensions: IDimensionRef[];
}

/** Header context of the AR JV builder / บริบท header ของตัวสร้าง JV ของ AR */
export interface IArJvBuildContext {
  doc_type: enum_ar_invoice_doc_type;
  currency_id: string;
  exchange_rate: Prisma.Decimal;
  base_currency_id: string;
}

/** One leg before the credit-note flip / หนึ่งขาก่อนสลับด้านของใบลดหนี้ */
interface IArLeg {
  account: string;
  cc: string | null;
  amount: Prisma.Decimal;
  base: Prisma.Decimal;
  isDebit: boolean;
  description: string | null;
  dims: IDimensionRef[];
}

/**
 * Push one leg unless zero; a credit note flips every leg
 * เพิ่มหนึ่งขาเว้นถ้าเป็นศูนย์; ใบลดหนี้สลับทุกขา
 * @param out - Accumulator / ตัวสะสม
 * @param seq - Next sequence / ลำดับถัดไป
 * @param ctx - Header context / บริบท header
 * @param leg - Leg to push / ขาที่จะเพิ่ม
 * @returns Next sequence / ลำดับถัดไป
 */
function pushArLeg(
  out: ISubledgerPostingLine[],
  seq: number,
  ctx: IArJvBuildContext,
  leg: IArLeg,
): number {
  if (leg.amount.isZero()) return seq;
  const isCredit = ctx.doc_type === enum_ar_invoice_doc_type.credit_note;
  const debit = leg.isDebit !== isCredit;
  out.push({
    sequence_no: seq,
    chart_of_accounts_id: leg.account,
    cost_center_id: leg.cc,
    debit: debit ? leg.amount : null,
    credit: debit ? null : leg.amount,
    base_debit: debit ? leg.base : ZERO,
    base_credit: debit ? ZERO : leg.base,
    currency_id: ctx.currency_id,
    exchange_rate: ctx.exchange_rate,
    description: leg.description,
    dimensions: leg.dims,
  });
  return seq + 1;
}

/**
 * Ledger lines of one ARIV/ARDN/ARCN: Dr AR total / Cr revenue net / Cr VAT / Cr tax 2, per line
 * บรรทัด ledger ของ ARIV/ARDN/ARCN: Dr AR total / Cr รายได้ net / Cr VAT / Cr ภาษีที่ 2 ต่อบรรทัด
 * @param ctx - Header context / บริบท header
 * @param details - Lines / บรรทัด
 * @returns Facade lines, balanced in base / บรรทัดของ facade ที่สมดุลในสกุลฐาน
 */
export function buildArInvoiceJvLines(
  ctx: IArJvBuildContext,
  details: IArJvSourceDetail[],
): ISubledgerPostingLine[] {
  const out: ISubledgerPostingLine[] = [];
  let seq = 1;
  for (const d of details) {
    const description = d.description;
    seq = pushArLeg(out, seq, ctx, {
      account: d.dr_chart_of_accounts_id,
      cc: d.dr_cost_center_id,
      amount: d.total_amount,
      base: d.base_total_amount,
      isDebit: true,
      description,
      dims: [],
    });
    seq = pushArLeg(out, seq, ctx, {
      account: d.cr_chart_of_accounts_id,
      cc: d.cr_cost_center_id,
      amount: d.net_amount,
      base: d.base_net_amount,
      isDebit: false,
      description,
      dims: d.dimensions,
    });
    seq = pushArLeg(out, seq, ctx, {
      account: d.vat_chart_of_accounts_id,
      cc: d.vat_cost_center_id,
      amount: d.vat_amount,
      base: d.base_vat_amount,
      isDebit: false,
      description,
      dims: [],
    });
    seq = pushArLeg(out, seq, ctx, {
      account: d.tax2_chart_of_accounts_id,
      cc: d.tax2_cost_center_id,
      amount: d.tax2_amount,
      base: d.base_tax2_amount,
      isDebit: false,
      description,
      dims: [],
    });
  }
  return out;
}

/** Base rounding of one CN reference / เศษปัดสกุลฐานของการอ้างใบลดหนี้หนึ่งรายการ */
export interface IArCreditNoteRounding {
  /** CN-side base moved − invoice-side base moved / ยอดฐานฝั่ง CN ที่ขยับ − ฝั่ง invoice */
  diff: Prisma.Decimal;
  /** AR (dr) account of the CN line that absorbed the take / บัญชี AR (dr) ของบรรทัด CN ที่ถูกหัก */
  account: string;
  /** That CN line's `dr_cost_center_id` / `dr_cost_center_id` ของบรรทัด CN นั้น */
  cost_center_id: string | null;
}

/**
 * Base-only AR ↔ realized-FX pair per CN reference (sign mirrored from AP: AR is debit-normal)
 * คู่บรรทัด AR ↔ realized FX สกุลฐานต่อการอ้าง CN (เครื่องหมายกลับจาก AP เพราะ AR เป็นบัญชีด้านเดบิต)
 *
 * Invoice base unpaid drops by I and CN base unpaid (a credit balance) drops by C, so the
 * subledger's net AR debit moves by C − I; GL follows with diff = C − I: > 0 → Dr AR / Cr FX gain,
 * < 0 → Cr AR / Dr FX loss, 0 → nothing (spec §9).
 * ยอดค้างฐานของ invoice ลด I และของ CN (ยอดด้านเครดิต) ลด C ยอดเดบิตสุทธิของ AR ใน subledger จึงขยับ
 * C − I; GL ตามด้วย diff = C − I: > 0 → Dr AR / Cr กำไร FX, < 0 → Cr AR / Dr ขาดทุน FX, 0 → ไม่ลง (spec §9)
 * @param ctx - Header context / บริบท header
 * @param refs - Rounding per reference / เศษปัดต่อการอ้าง
 * @param fx - FX accounts (only the side used must be set) / บัญชี FX (ต้องมีเฉพาะด้านที่ใช้)
 * @param fx.gain - Gain account / บัญชีกำไร
 * @param fx.loss - Loss account / บัญชีขาดทุน
 * @param startSeq - First sequence / ลำดับแรก
 * @returns Facade lines (empty when every diff is zero) / บรรทัดของ facade (ว่างถ้า diff เป็นศูนย์ทั้งหมด)
 */
export function buildArCreditNoteRoundingJvLines(
  ctx: IArJvBuildContext,
  refs: IArCreditNoteRounding[],
  fx: { gain: string; loss: string },
  startSeq: number,
): ISubledgerPostingLine[] {
  const out: ISubledgerPostingLine[] = [];
  const currency = { currency_id: ctx.base_currency_id, exchange_rate: new Prisma.Decimal(1) };
  const description = 'Credit-note netting base rounding';
  let seq = startSeq;
  for (const r of refs) {
    if (r.diff.isZero()) continue;
    const isArDebit = r.diff.gt(0);
    const abs = r.diff.abs();
    out.push(
      baseLine(seq++, r.account, r.cost_center_id, isArDebit, abs, currency, description),
      baseLine(seq++, isArDebit ? fx.gain : fx.loss, null, !isArDebit, abs, currency, description),
    );
  }
  return out;
}

/**
 * One base-currency line at rate 1
 * บรรทัดสกุลฐานหนึ่งบรรทัดที่อัตรา 1
 * @param seq - Sequence / ลำดับ
 * @param account - Account id / รหัสบัญชี
 * @param cc - Cost center / cost center
 * @param debit - Debit side / ด้านเดบิต
 * @param amount - Amount (txn = base) / ยอด (txn = base)
 * @param currency - Base currency at rate 1 / สกุลฐานอัตรา 1
 * @param description - Description / คำอธิบาย
 * @returns Facade line / บรรทัดของ facade
 */
function baseLine(
  seq: number,
  account: string,
  cc: string | null,
  debit: boolean,
  amount: Prisma.Decimal,
  currency: { currency_id: string; exchange_rate: Prisma.Decimal },
  description: string,
): ISubledgerPostingLine {
  return {
    sequence_no: seq,
    chart_of_accounts_id: account,
    cost_center_id: cc,
    debit: debit ? amount : null,
    credit: debit ? null : amount,
    base_debit: debit ? amount : ZERO,
    base_credit: debit ? ZERO : amount,
    ...currency,
    description,
  };
}
```

- [ ] **Step 4: numbering**

`ar-invoice.running-code.ts` — copy `ap-invoice.running-code.ts` ทั้งไฟล์ (รวม `buildLiteralPrefix` และ JSDoc เรื่อง advisory lock) แล้วแก้:
```ts
/** AR doc types this service numbers (deposit is out of scope) / ประเภทเอกสาร AR ที่ออกเลข (ไม่รวมมัดจำ) */
export type ArDocType = Exclude<enum_ar_invoice_doc_type, 'deposit'>;

/** Running-code types and literals per AR doc type / type running code และ literal ต่อประเภทเอกสาร AR */
export const AR_DOC_TYPE_RUNNING_CODE: Record<
  ArDocType,
  { type: string; prefix: string; taxType: string; taxPrefix: string }
> = {
  invoice: { type: 'AR-IV', prefix: 'ARIV', taxType: 'TX-IV', taxPrefix: 'TXIV' },
  debit_note: { type: 'AR-DN', prefix: 'ARDN', taxType: 'TX-DN', taxPrefix: 'TXDN' },
  credit_note: { type: 'AR-CN', prefix: 'ARCN', taxType: 'TX-CN', taxPrefix: 'TXCN' },
};
```
- `generateArDocNo(params)` = `generateApDocNo` โดย `docType: ArDocType`, ใช้ `AR_DOC_TYPE_RUNNING_CODE[docType].type`, lock key `` `ar-doc-no:${buCode}:${type}:${datePart}` ``, query `prisma.tb_ar_invoice.findFirst({ where: { doc_no: { startsWith: `${literalPrefix}${datePart}` }, deleted_at: null }, orderBy: { doc_no: 'desc' }, select: { doc_no: true } })`
- `generateArTaxInvoiceNo(params)` เหมือนกันแต่ใช้ `.taxType`, lock key `` `ar-tax-no:${buCode}:${type}:${datePart}` ``, query `prisma.tb_ar_tax_invoice.findFirst({ where: { tax_invoice_no: { startsWith: `${literalPrefix}${datePart}` } }, orderBy: { tax_invoice_no: 'desc' }, select: { tax_invoice_no: true } })` — **ไม่กรอง `deleted_at`** เพื่อไม่ออกเลขซ้ำกับใบที่ void/ลบ; `lastNo = Number(latest.tax_invoice_no.slice(-width))`

- [ ] **Step 5: workflow mapper + serializer**

`workflow/ar-invoice-workflow.mapper.ts` copy `ap-invoice-workflow.mapper.ts` เปลี่ยนชื่อฟังก์ชันเป็น `arInvoiceToWorkflowDocument` และคำ `ap`/`AP` ในชื่อ/JSDoc เป็น `ar`/`AR` (field ที่อ่านเหมือนเดิม)
`dto/ar-invoice.serializer.ts` copy `ap-invoice.serializer.ts`: แปลง Decimal ทุกคอลัมน์เงิน/rate ของ `tb_ar_invoice`, `tb_ar_invoice_detail` (รวม `discount_pct`, `tax2_*`, `folio_amount`), `tb_ar_invoice_reference`, `tb_ar_tax_invoice`; คืน `tax_invoice` เป็น object เดียวหรือ `null`; คืน `original_tax_invoice_no` จาก `info`; export `serializeArInvoice`

- [ ] **Step 6: static check + commit**

```bash
bun run check-types
npx eslint --no-fix apps/micro-business/src/ar
git add apps/micro-business/src/ar
git commit -m "feat(ar): AR invoice line math, JV builder, numbering, workflow mapper"
```

---

### Task 6: AR invoice — validation และ writer

**Files:**
- Create: `apps/micro-business/src/ar/ar-invoice/ar-invoice.validation.ts`
- Create: `apps/micro-business/src/ar/ar-invoice/ar-invoice.writer.ts`

**Interfaces:**
- Consumes: Task 5; `GlDimensionValidator` (แบบที่ `ApInvoiceWriter` ใช้); `computeDueDate` จาก AP logic; `ERROR_CATALOG.AR_*`, `CUSTOMER_*`
- Produces: `validateArCustomer(db, customerId): Promise<Result<tb_customer>>`; `validateArLines(db, lines): Promise<Result<IResolvedArLineRefs>>` (`IResolvedArLineRefs = { profiles: Map<string, tb_tax_profile> }`); `checkArHeader(data: ICreateArInvoice): Result<true>`; `IArWriteHeaderContext`; `class ArInvoiceWriter { writeDocument(tx, ctx): Promise<Result<{ id: string }>> }`

แม่แบบ: `ap-invoice.validation.ts`, `ap-invoice.writer.ts`

- [ ] **Step 1: validation**

copy `ap-invoice.validation.ts` แล้ว:
- `ApValidationDb` → `ArValidationDb = Pick<PrismaClient, 'tb_chart_of_accounts' | 'tb_cost_center' | 'tb_tax_profile' | 'tb_customer'>`
- `validateApVendor` → `validateArCustomer`: `tb_customer.findFirst({ where: { id, deleted_at: null } })` → ไม่เจอ `CUSTOMER_NOT_FOUND`; `is_active === false` → `CUSTOMER_INACTIVE` `{ code }`
- `validateApLines(db, lines, isDeposit)` → `validateArLines(db, lines)`: โหลดบัญชี (`cr`, `dr`, `vat`, `tax2` ของทุกบรรทัด), cost center และ profile (`vat_tax_profile_id`, `tax2_tax_profile_id`) แบบเดียวกัน แล้วเรียก `checkArAmounts` + `checkArRefs` ต่อบรรทัด
- `checkAmounts` → `checkArAmounts`:
```ts
function checkArAmounts(l: IArInvoiceLineInput): Result<true> {
  const line = (reason: string) =>
    Result.errorFromCatalog(ERROR_CATALOG.AR_INVOICE_LINE_INVALID, { sequence_no: l.sequence_no, reason }) as Result<never>;
  const disc = (reason: string) =>
    Result.errorFromCatalog(ERROR_CATALOG.AR_DISCOUNT_INVALID, { sequence_no: l.sequence_no, reason }) as Result<never>;
  const qty = new Prisma.Decimal(l.quantity);
  const price = new Prisma.Decimal(l.unit_price);
  const pct = new Prisma.Decimal(l.discount_pct ?? 0);
  if (qty.lte(0) || price.lt(0)) return line('quantity must be > 0 and unit price ≥ 0');
  if (l.is_pms_folio) return Result.errorFromCatalog(ERROR_CATALOG.AR_SOURCE_NOT_SUPPORTED) as Result<never>;
  if (pct.lt(0) || pct.gt(100)) return disc('discount percent must be 0–100');
  if (!l.discount_is_override) return Result.ok(true);
  const amount = new Prisma.Decimal(l.discount_amount ?? 0);
  if (amount.lt(0) || amount.gt(round2(qty.mul(price)))) return disc('discount exceeds subtotal');
  return Result.ok(true);
}
```
- `checkRefs` → `checkArRefs(l, accounts, costCenters, profiles)`: ทุกบัญชีที่ส่งมา (`cr` บังคับ, `dr`/`vat`/`tax2` ถ้าส่ง) ใช้เงื่อนไข postable เดียวกับ `checkRefs` ของ AP → `AR_ACCOUNT_NOT_POSTABLE` `{ sequence_no, account: <code> }`; บัญชีที่ `is_require_cost_center` ต้องมี cc คู่ (`cr_cost_center_id` / `dr_cost_center_id` / `vat_cost_center_id` / `tax2_cost_center_id`) → `AR_COST_CENTER_REQUIRED`; `cr_cost_center_id` ต้องมีอยู่จริง → ไม่งั้น `AR_INVOICE_LINE_INVALID`; profile ต้อง active; vat profile ต้อง `tax_type === enum_tax_profile_tax_type.vat` → ไม่งั้น `AR_INVOICE_LINE_INVALID` reason `'VAT profile must be a VAT type'`; tax2 profile `tax_type === enum_tax_profile_tax_type.wht` → `AR_TAX2_WHT_NOT_ALLOWED` `{ sequence_no }`
- เพิ่ม:
```ts
/**
 * Header rules that do not need the database (spec §7.1)
 * กฎ header ที่ไม่ต้องใช้ฐานข้อมูล (spec §7.1)
 * @param data - Payload / payload
 * @returns ok or a catalog error / ok หรือ error จาก catalog
 */
export function checkArHeader(data: ICreateArInvoice): Result<true> {
  const isManual = !data.doc_source || data.doc_source === enum_ar_invoice_source.manual;
  if (data.doc_type === enum_ar_invoice_doc_type.deposit || !isManual || data.is_pms_folio) {
    return Result.errorFromCatalog(ERROR_CATALOG.AR_SOURCE_NOT_SUPPORTED);
  }
  if (new Prisma.Decimal(data.wht_amount ?? 0).lt(0)) {
    return Result.errorFromCatalog(ERROR_CATALOG.AR_INVOICE_LINE_INVALID, { sequence_no: 0, reason: 'wht_amount must be ≥ 0' });
  }
  if (data.details.add.length === 0) {
    return Result.errorFromCatalog(ERROR_CATALOG.AR_INVOICE_LINE_INVALID, { sequence_no: 0, reason: 'at least one line is required' });
  }
  return Result.ok(true);
}
```

- [ ] **Step 2: writer**

copy `ap-invoice.writer.ts` → `ar-invoice.writer.ts` ตัด GRN ทั้งหมด (`SourceItem`, `checkSourceItems`, `writeSources`, `GRN_SOURCE_SELECT`) แล้ว:
- `IWriteHeaderContext` → `IArWriteHeaderContext`:
```ts
export interface IArWriteHeaderContext {
  existing_id: string | null;
  doc_version: number | null;
  data: ICreateArInvoice;
  customer: tb_customer;
  currency: { id: string; code: string };
  base_currency: { id: string; code: string };
  exchange_rate: Prisma.Decimal;
  credit_term_days: number;
  default_dr_account_id: string | null;
  profiles: Map<string, tb_tax_profile>;
  user_id: string;
  bu_code: string;
}
```
  (ถ้า `IWriteHeaderContext` ของ AP มี field อื่นที่ writer ใช้ เช่น `commonLogic` หรือ dimension resolver ให้คงไว้ชื่อเดิม)
- `computeForLine(l, ctx)`:
```ts
  private computeForLine(l: IArInvoiceLineInput, ctx: IArWriteHeaderContext): IArComputedLine {
    const vat = l.vat_tax_profile_id ? ctx.profiles.get(l.vat_tax_profile_id) : undefined;
    const tax2 = l.tax2_tax_profile_id ? ctx.profiles.get(l.tax2_tax_profile_id) : undefined;
    const override = (on: boolean | undefined, v: number | string | null | undefined) =>
      on ? new Prisma.Decimal(v ?? 0) : null;
    return computeArLine(
      {
        quantity: new Prisma.Decimal(l.quantity),
        unit_price: new Prisma.Decimal(l.unit_price),
        discount_pct: new Prisma.Decimal(l.discount_pct ?? 0),
      },
      {
        exchange_rate: ctx.exchange_rate,
        vat_rate: vat?.tax_rate ?? new Prisma.Decimal(0),
        tax2_rate: tax2?.tax_rate ?? new Prisma.Decimal(0),
        discount_override: override(l.discount_is_override, l.discount_amount),
        vat_override: override(l.vat_is_override, l.vat_amount),
        tax2_override: override(l.tax2_is_override, l.tax2_amount),
      },
    );
  }
```
- `headerData(ctx, totals)`: `doc_type`, `doc_date`, `doc_source: enum_ar_invoice_source.manual`, `is_pms_folio: false`, `customer_id/code/name` จาก customer, `credit_term_id/name` จาก customer, `credit_term_days`, `due_date: computeDueDate(doc_date, credit_term_days)`, `currency_id/code`, `base_currency_id/code`, `exchange_rate`, ยอดรวมทุกช่องจาก `sumArLines` (รวม `tax2_amount`, `base_tax2_amount`), `is_tax_invoice`, `is_wht_recorded`, `wht_amount`, `source_doc_ref`, `description`, `attachments`, `info: doc_type === credit_note && original_tax_invoice_no ? { original_tax_invoice_no } : {}`, `updated_by_id`
- `detailData(invoiceId, l, c, ctx)`: map ทุก field ของ `IArInvoiceLineInput` + `IArComputedLine`; `vat_rate` / `tax2_rate` จาก profile (หรือ 0); `dr_chart_of_accounts_id: l.dr_chart_of_accounts_id ?? ctx.default_dr_account_id` (service รับประกันว่าไม่ null — Task 7); snapshot `cr_account_code` / `dr_account_code` / `cr_cost_center_code` ถ้า AP snapshot code ในจุดเดียวกัน; `is_pms_folio: false`; `unpaid_amount: 0`, `base_unpaid_amount: 0`
- `writeLines`: dimension ผ่าน `GlDimensionValidator` + เขียน `tb_ar_invoice_detail_dimension` แบบ AP
- `referenceRows(tx, invoiceId, ctx)`: ต่อ reference โหลด CN (`tb_ar_invoice.findUniqueOrThrow`) แล้ว
```ts
  const applied = new Prisma.Decimal(r.applied_amount);
  const netShare = cn.total_amount.isZero()
    ? new Prisma.Decimal(0)
    : round2(applied.mul(cn.net_amount.add(cn.tax2_amount)).div(cn.total_amount));
  // row: { ar_invoice_id: invoiceId, ref_ar_invoice_id: cn.id, ref_doc_type: enum_ar_invoice_doc_type.credit_note,
  //        applied_amount: applied, applied_net_amount: netShare, applied_vat_amount: applied.sub(netShare),
  //        remarks: r.remarks ?? null, created_by_id: ctx.user_id }
```
- `createHeader`: `doc_no` จาก `generateArDocNo({ commonLogic, prisma: tx, userId, buCode, docType: data.doc_type as ArDocType, docDate: data.doc_date })`
- `replaceHeader`: `updateMany({ where: { id, doc_version } ...})` + `doc_version: { increment: 1 }` แล้วลบบรรทัด/dimension/reference เดิมก่อนเขียนใหม่ แบบ AP (count = 0 → `AR_INVOICE_IMMUTABLE`)

- [ ] **Step 3: static check + commit**

```bash
bun run check-types
npx eslint --no-fix apps/micro-business/src/ar
git add apps/micro-business/src/ar
git commit -m "feat(ar): AR invoice validation and writer"
```

---

### Task 7: AR invoice service — CRUD และ reference candidates

**Files:**
- Create: `apps/micro-business/src/ar/ar-invoice/ar-invoice.service.ts`

**Interfaces:**
- Consumes: Task 5, 6; `ExchangeRateService.findByDateAndCurrency`; `GlPostingService.getSetting`
- Produces: `class ArInvoiceService` — `findOne(id)`, `findAll(paginate)`, `create(data)`, `update(id, data)`, `delete(id)`, `referenceCandidates(id)`, `loadFull(db, id)`; `AR_INVOICE_FULL_INCLUDE`, `ArInvoiceFull`; `arReferenceAvailable(row)`, `arReferenceViolation(row, applied, doc, selfId)`

แม่แบบ: `ap-invoice.service.ts` — ตัด GRN (`grnCandidates`, `matchedQtyByItem`, `createFromGrn`, `grnLines`, `resolveProductAccount`, `checkGrnItems`, `GrnItemWithHeader`) และ `isVendorInvoiceDuplicate`

- [ ] **Step 1: include + CRUD**

```ts
export const AR_INVOICE_FULL_INCLUDE = {
  tb_ar_invoice_detail: {
    where: { deleted_at: null },
    orderBy: { sequence_no: 'asc' },
    include: { tb_ar_invoice_detail_dimension: true },
  },
  tb_ar_invoice_reference: true,
  tb_ar_tax_invoice: true,
} satisfies Prisma.tb_ar_invoiceInclude;

export type ArInvoiceFull = Prisma.tb_ar_invoiceGetPayload<{ include: typeof AR_INVOICE_FULL_INCLUDE }>;
```
ตรวจชื่อ relation จริงก่อน: `awk '/^model tb_ar_invoice /,/^}/' packages/prisma-shared-schema-tenant/prisma/schema.prisma | grep "tb_ar_"` แล้วใช้ชื่อฝั่ง `"ArInvoiceReferences"` (ใบนี้อ้างใบอื่น) ให้ตรง

copy ส่วนที่เหลือจาก AP แล้วเปลี่ยน:
- `findAll`: search `doc_no`, `customer_code`, `customer_name`; filter `doc_type`, `doc_status` แบบ AP
- `create` / `update` → `buildContext(data, existing)`:
  1. `checkArHeader(data)`
  2. `validateArCustomer(this.prismaService, data.customer_id)`
  3. `resolveHeader`: currency = `data.currency_id ?? customer.default_currency_id ?? base`; `credit_term_days = data.credit_term_days ?? customer.credit_term_days ?? 0`; rate จาก `resolveRate` เดิม
  4. `default_dr_account_id = customer.ar_chart_of_accounts_id ?? setting?.ar_control_account_id ?? null`; ถ้าเป็น null และมีบรรทัดที่ไม่ส่ง `dr_chart_of_accounts_id` → `Result.errorFromCatalog(ERROR_CATALOG.GL_SETTING_ACCOUNT_MISSING, { key: 'ar_control_account_id' })`
  5. `validateArLines(this.prismaService, data.details.add)` (โหลดบัญชี default dr ด้วยเพื่อให้ตรวจ postable)
  6. คำนวณ `sumArLines` ของบรรทัดใหม่ แล้ว `validateReferences(data.references ?? [], {...header, total_amount}, existing?.id ?? null)` (Step 2)
  7. `this.prismaService.$transaction((tx) => this.writer.writeDocument(tx, ctx))`
- update: สถานะต้อง `draft`, `doc_version` ตรง (`AR_INVOICE_IMMUTABLE`); `mergeUpdate` แบบ AP — ไม่ส่ง `details` ใช้บรรทัดเดิม (`toArLineInput` map field AR ครบ), ไม่ส่ง `references` ใช้ของเดิม → validate reference ใหม่ทุกครั้ง (Review Focus 4)
- delete: เฉพาะ draft, soft delete แบบ AP

- [ ] **Step 2: validateReferences + referenceCandidates**

ฟังก์ชันระดับไฟล์ (export — Task 8 ใช้ซ้ำ):
```ts
/**
 * Amount of a credit note still available to apply
 * ยอดของใบลดหนี้ที่ยังนำไปหักกลบได้
 * @param row - Credit note / ใบลดหนี้
 * @returns min(total − applied, outstanding) / min(total − ที่หักไปแล้ว, ยอดค้าง)
 */
export function arReferenceAvailable(
  row: Pick<tb_ar_invoice, 'total_amount' | 'reference_applied_amount' | 'outstanding_amount'>,
): Prisma.Decimal {
  return Prisma.Decimal.min(row.total_amount.sub(row.reference_applied_amount), row.outstanding_amount);
}

/**
 * Why a credit note cannot be applied to this invoice, or null when it can (spec §9)
 * เหตุที่ใบลดหนี้นำมาหักกลบกับ invoice นี้ไม่ได้ หรือ null ถ้าได้ (spec §9)
 * @param row - Referenced document (re-read after lock when posting) / เอกสารที่ถูกอ้าง (อ่านใหม่หลัง lock ตอน post)
 * @param applied - Amount to apply / ยอดที่จะหัก
 * @param doc - Referencing invoice / invoice ที่อ้าง
 * @param selfId - Referencing invoice id (null on create) / id ของ invoice ที่อ้าง (null ตอนสร้าง)
 * @returns Catalog error or null / error จาก catalog หรือ null
 */
export function arReferenceViolation(
  row: tb_ar_invoice | undefined,
  applied: Prisma.Decimal,
  doc: { customer_id: string; currency_id: string; exchange_rate: Prisma.Decimal },
  selfId: string | null,
): Result<never> | null {
  const fail = (code: 'AR_INVOICE_NOT_FOUND' | 'AR_REFERENCE_TYPE_NOT_ALLOWED' | 'AR_REFERENCE_NOT_POSTED' | 'AR_CUSTOMER_CURRENCY_MISMATCH' | 'AR_REFERENCE_RATE_MISMATCH' | 'AR_REFERENCE_OVER_APPLIED', docNo: string) =>
    Result.errorFromCatalog(ERROR_CATALOG[code], { doc_no: docNo }) as Result<never>;
  if (!row || row.id === selfId) return fail('AR_INVOICE_NOT_FOUND', '-');
  if (row.doc_type !== enum_ar_invoice_doc_type.credit_note) return fail('AR_REFERENCE_TYPE_NOT_ALLOWED', row.doc_no);
  if (row.doc_status !== enum_ar_invoice_status.posted) return fail('AR_REFERENCE_NOT_POSTED', row.doc_no);
  if (row.customer_id !== doc.customer_id || row.currency_id !== doc.currency_id) return fail('AR_CUSTOMER_CURRENCY_MISMATCH', row.doc_no);
  if (!row.exchange_rate.equals(doc.exchange_rate)) return fail('AR_REFERENCE_RATE_MISMATCH', row.doc_no);
  if (applied.lte(0) || applied.gt(arReferenceAvailable(row))) return fail('AR_REFERENCE_OVER_APPLIED', row.doc_no);
  return null;
}
```
method:
```ts
  private async validateReferences(
    refs: IArInvoiceReferenceInput[],
    doc: {
      doc_type: enum_ar_invoice_doc_type;
      customer_id: string;
      currency_id: string;
      exchange_rate: Prisma.Decimal;
      total_amount: Prisma.Decimal;
    },
    selfId: string | null,
  ): Promise<Result<true>> {
    if (refs.length === 0) return Result.ok(true);
    if (doc.doc_type !== enum_ar_invoice_doc_type.invoice) {
      return Result.errorFromCatalog(ERROR_CATALOG.AR_REFERENCE_TYPE_NOT_ALLOWED, { doc_no: '-' });
    }
    const rows = await this.prismaService.tb_ar_invoice.findMany({
      where: { id: { in: refs.map((r) => r.ref_ar_invoice_id) }, deleted_at: null },
    });
    let sum = new Prisma.Decimal(0);
    for (const r of refs) {
      const applied = new Prisma.Decimal(r.applied_amount);
      const bad = arReferenceViolation(rows.find((x) => x.id === r.ref_ar_invoice_id), applied, doc, selfId);
      if (bad) return bad;
      sum = sum.add(applied);
    }
    if (sum.gt(doc.total_amount)) {
      return Result.errorFromCatalog(ERROR_CATALOG.AR_REFERENCE_OVER_APPLIED, { doc_no: '-' });
    }
    return Result.ok(true);
  }
```
`referenceCandidates(id)`: โหลด invoice (ไม่เจอ `AR_INVOICE_NOT_FOUND`; ไม่ใช่ `invoice` → `AR_REFERENCE_TYPE_NOT_ALLOWED`) → `tb_ar_invoice.findMany({ where: { doc_type: credit_note, doc_status: posted, deleted_at: null, customer_id, currency_id, exchange_rate: inv.exchange_rate } })` → กรอง `arReferenceAvailable(row).gt(0)` → คืน `{ id, doc_no, doc_date, total_amount, available_amount }` (Decimal → string)

- [ ] **Step 3: static check + commit**

```bash
bun run check-types
npx eslint --no-fix apps/micro-business/src/ar
git add apps/micro-business/src/ar
git commit -m "feat(ar): AR invoice CRUD and credit-note reference candidates"
```

---

### Task 8: AR invoice posting, ใบกำกับภาษี, void, workflow actions

**Files:**
- Create: `apps/micro-business/src/ar/ar-invoice/ar-invoice.posting.ts`
- Modify: `apps/micro-business/src/ar/ar-invoice/ar-invoice.service.ts`

**Interfaces:**
- Consumes: Task 5–7; `GlSubledgerPostingService.postFromSource / reverseBySource / requireSettingAccount`; `GlPostingService.getSetting`; `WorkflowOrchestratorService`; `CommonLogic`; `abortWith` (`@/gl/gl-subledger-posting/tx-abort`); จาก AP logic: `planUnpaidReduction`, `planSequentialReductions`, `deductPlan`, `plannedBase`, `mergePlans`, `UnpaidPlan`, `IUnpaidLine`
- Produces: `class ArInvoicePosting` — `isPeriodOpen(db, date)`, `postInTx(tx, doc, userId)`, `voidPostedInTx(tx, doc, reason, userId)`, `recomputeOutstanding(tx, id)`; service `submit`, `approve`, `review`, `reject`, `void`

แม่แบบ: `ap-invoice.posting.ts` — copy ทั้งไฟล์ แล้ว**ลบ** `checkGrnSources`, `grnSourceViolation`, `depositReferenceLines`, `advanceCostCenters`, การใช้ `buildDepositReferenceJvLines`, `checkNoLivePayment`, `upsertTaxRecord`, `taxRecordData`, `addMonthsClamped`, `referenceViolation`, `referenceTypeViolation`, `referenceAvailable` (ใช้ของ Task 7 แทน) และเปลี่ยนทุก `tb_ap_*` → `tb_ar_*`, `ap_invoice_id` → `ar_invoice_id`, `ref_ap_invoice_id` → `ref_ar_invoice_id`, `vendor_name` → `customer_name`, `enum_ap_*` → `enum_ar_*`, `SOURCE_REF_TYPE = 'ar_invoice'`, `enum_gl_jv_source.ar`; CN-side account ใช้ `dr_chart_of_accounts_id` / `dr_cost_center_id` ของบรรทัด CN (บัญชี AR) แทน `cr_*` ของ AP

- [ ] **Step 1: post (spec §6, §8, §9)**

```ts
  async postInTx(
    tx: Prisma.TransactionClient,
    doc: ArInvoiceFull,
    userId: string,
  ): Promise<Result<IArInvoicePostResult>> {
    const postable = await this.checkPostable(tx, doc);
    if (postable.isError()) abortWith(postable);
    const netting = await this.planNetting(tx, doc);
    const lines = await this.buildLines(doc, netting);
    if (lines.isError()) abortWith(lines);
    const posted = await this.subledger.postFromSource(
      {
        source: enum_gl_jv_source.ar,
        source_ref_type: SOURCE_REF_TYPE,
        source_ref_id: doc.id,
        jv_date: doc.doc_date,
        description: `${doc.doc_no} ${doc.customer_name}`,
        currency_id: doc.currency_id,
        exchange_rate: doc.exchange_rate,
        lines: lines.value,
      },
      tx,
    );
    if (posted.isError()) abortWith(posted);
    if (posted.value.already_posted) abortWith(Result.errorFromCatalog(ERROR_CATALOG.GL_JV_IMMUTABLE));
    await this.issueTaxInvoice(tx, doc, userId);
    await this.applyReferences(tx, doc, netting, userId);
    await this.markPosted(tx, doc, netting, posted.value, userId);
    return Result.ok({ id: doc.id, doc_status: enum_ar_invoice_status.posted, gl_jv_no: posted.value.jv_no });
  }
```
- `planNetting(tx, doc)`: เหมือน AP แต่มีเฉพาะ CN — `lockReferencedRows` (`FOR UPDATE` แบบ AP) → ต่อ reference เรียก `arReferenceViolation(lockedRow, r.applied_amount, doc, doc.id)` ถ้าไม่ null → `abortWith(violation)` (Review Focus 1) → plan ฝั่ง CN `planUnpaidReduction(cnLines, applied, cn.exchange_rate)` และฝั่ง invoice `planSequentialReductions(invoiceLines, takes, doc.exchange_rate)` แบบ AP (unpaid ตั้งต้นฝั่ง invoice = total ต่อบรรทัด เหมือน AP)
- `buildLines(doc, netting)`:
```ts
  private async buildLines(doc: ArInvoiceFull, netting: IArNettingPlan): Promise<Result<ISubledgerPostingLine[]>> {
    const setting = await this.glPosting.getSetting();
    const details = await this.jvDetails(doc, setting);
    if (details.isError()) return Result.error(details.error);
    const ctx: IArJvBuildContext = {
      doc_type: doc.doc_type,
      currency_id: doc.currency_id,
      exchange_rate: doc.exchange_rate,
      base_currency_id: doc.base_currency_id,
    };
    const lines = buildArInvoiceJvLines(ctx, details.value);
    const rounding = arCreditNoteRoundings(doc, netting);
    const fx = this.roundingFxAccounts(setting, rounding);
    if (fx.isError()) return Result.error(fx.error);
    return Result.ok([...lines, ...buildArCreditNoteRoundingJvLines(ctx, rounding, fx.value, lines.length + 1)]);
  }
```
- `jvDetails(doc, setting)`: โหลด profile ของบรรทัด (`tb_tax_profile.findMany({ where: { id: { in: [...vat ids, ...tax2 ids] } } })`) แล้ว map เป็น `IArJvSourceDetail`:
  - `vat_chart_of_accounts_id = d.vat_chart_of_accounts_id ?? profile(vat).chart_of_accounts_id ?? <requireSettingAccount(setting, 'output_vat_account_id')>` — เรียก `requireSettingAccount` เฉพาะเมื่อ `vat_amount ≠ 0` และสองตัวแรกว่าง; ถ้า `vat_amount = 0` ใช้ `''` (บรรทัดศูนย์ถูกข้าม)
  - `tax2_chart_of_accounts_id = d.tax2_chart_of_accounts_id ?? profile(tax2).chart_of_accounts_id` — ถ้า `tax2_amount ≠ 0` และว่าง → `Result.errorFromCatalog(ERROR_CATALOG.GL_SETTING_ACCOUNT_MISSING, { key: 'tax2_chart_of_accounts_id' })`; ถ้าศูนย์ใช้ `''`
  - `dimensions` จาก `tb_ar_invoice_detail_dimension`
- `arCreditNoteRoundings(doc, netting)`: เหมือน `creditNoteRoundings` ของ AP → `{ diff: bases.atRef.sub(bases.atInvoice), account: cn.account, cost_center_id: cn.cost_center_id }`
- `roundingFxAccounts(setting, rounding)`: **กลับเงื่อนไขจาก AP** — `realized_fx_gain_account_id` เมื่อมี `diff > 0`, `realized_fx_loss_account_id` เมื่อมี `diff < 0`
- `markPosted` / `applyReferences` / `recordReferenceBases` / `applyPlan` / `recomputeOutstanding`: แบบ AP (เฉพาะ CN)

- [ ] **Step 2: ใบกำกับภาษี (spec §10)**

```ts
  /**
   * Issue the tax invoice once, at post, inside the posting transaction (spec §10)
   * ออกใบกำกับภาษีครั้งเดียวตอน post ภายใน transaction ของการ post (spec §10)
   * @param tx - Transaction / transaction
   * @param doc - Document being posted / เอกสารที่กำลัง post
   * @param userId - Actor / ผู้ทำ
   */
  private async issueTaxInvoice(tx: Prisma.TransactionClient, doc: ArInvoiceFull, userId: string): Promise<void> {
    if (!doc.is_tax_invoice || doc.tb_ar_tax_invoice) return;
    const customer = await tx.tb_customer.findUniqueOrThrow({
      where: { id: doc.customer_id },
      include: { tb_customer_address: { where: { deleted_at: null, is_active: true } } },
    });
    const taxNo = await generateArTaxInvoiceNo({
      commonLogic: this.commonLogic,
      prisma: tx,
      userId,
      buCode: this.bu_code,
      docType: doc.doc_type as ArDocType,
      docDate: doc.doc_date,
    });
    try {
      await tx.tb_ar_tax_invoice.create({ data: arTaxInvoiceData(doc, customer, taxNo, userId) });
    } catch (e) {
      const isDuplicate = e instanceof Prisma.PrismaClientKnownRequestError && e.code === 'P2002';
      if (isDuplicate) {
        abortWith(Result.errorFromCatalog(ERROR_CATALOG.AR_TAX_INVOICE_NO_CONFLICT, { tax_invoice_no: taxNo }));
      }
      throw e;
    }
  }
```
advisory lock ใน `generateArTaxInvoiceNo` ทำให้สองผู้เรียกไม่อ่านเลขเดียวกัน — unique violation หลัง lock แปลว่ามีคนแก้ running code config ระหว่างทาง จึงหยุดพร้อม error แทน "retry 1 ครั้ง" ใน spec §10.1 · inject `CommonLogic` ใน constructor (ดูว่า `ApInvoiceService` ได้ `CommonLogic` มาอย่างไรแล้วทำเหมือนกัน) · `this.bu_code` ใช้ชื่อ property เดียวกับที่ `TenantScopedService` ให้ (ดู `VendorsService` ที่ log `tenant_id: this.bu_code`) · ชนิด `Prisma.PrismaClientKnownRequestError` import ตามที่ไฟล์ AP import (ถ้า AP ใช้ helper อื่นตรวจ P2002 ให้ใช้ helper นั้น)

ฟังก์ชันระดับไฟล์:
```ts
/**
 * Output-tax register row for a posted AR document (THB, credit note negative — spec §10.1)
 * แถวทะเบียนภาษีขายของเอกสาร AR ที่ post แล้ว (THB, ใบลดหนี้ติดลบ — spec §10.1)
 * @returns Create input / ข้อมูลสำหรับ create
 */
function arTaxInvoiceData(
  doc: ArInvoiceFull,
  customer: tb_customer & { tb_customer_address: tb_customer_address[] },
  taxNo: string,
  userId: string,
): Prisma.tb_ar_tax_invoiceUncheckedCreateInput {
  const sign = doc.doc_type === enum_ar_invoice_doc_type.credit_note ? -1 : 1;
  const byType = (t: enum_customer_address_type) =>
    customer.tb_customer_address.find((a) => a.address_type === t);
  const addr = byType(enum_customer_address_type.register_address) ?? byType(enum_customer_address_type.billing_address);
  const vatLine = doc.tb_ar_invoice_detail.find((d) => !d.vat_amount.isZero());
  const base = doc.base_net_amount.mul(sign);
  const vat = doc.base_vat_amount.mul(sign);
  return {
    ar_invoice_id: doc.id,
    tax_prefix: AR_DOC_TYPE_RUNNING_CODE[doc.doc_type as ArDocType].taxPrefix,
    tax_invoice_no: taxNo,
    tax_invoice_date: doc.doc_date,
    filing_month: doc.doc_date.getUTCMonth() + 1,
    filing_year: doc.doc_date.getUTCFullYear(),
    tax_status: enum_ar_tax_invoice_status.pending,
    customer_registered_name: customer.registered_name ?? customer.name,
    customer_tax_no: customer.tax_no,
    customer_branch_no: customer.branch_no,
    address_line1: addr?.address_line1 ?? null,
    address_line2: addr?.address_line2 ?? null,
    province: addr?.province ?? null,
    postal_code: addr?.postal_code ?? null,
    base_amount: base,
    vat_rate: vatLine?.vat_rate ?? new Prisma.Decimal(0),
    vat_amount: vat,
    total_amount: base.add(vat),
    created_by_id: userId,
  };
}
```
`doc.base_net_amount` = Σ base_net (ไม่รวม tax2) ตรงกับ spec §10.1

- [ ] **Step 3: void (spec §11)**

`voidPostedInTx` แบบ AP โดย:
- แทน `checkNoLivePayment` ด้วย `checkNoLiveReceipt` — ตรวจ schema ก่อน (`awk '/^model tb_ar_receipt_detail /,/^}/' packages/prisma-shared-schema-tenant/prisma/schema.prisma`) แล้วนับแถว `tb_ar_receipt_detail` ที่ผูกกับ invoice นี้ (ผ่าน `ar_invoice_id` หรือ `tb_ar_invoice_detail: { ar_invoice_id }` ตามชื่อ FK จริง) และ `tb_ar_receipt: { deleted_at: null, doc_status: { not: enum_ar_receipt_status.void } }` → > 0 = `AR_INVOICE_HAS_RECEIPT`
- `checkVoidable`: `reference_applied_amount` ≠ 0 → `AR_INVOICE_IS_REFERENCED`; period ของวันที่ void ต้องเปิด → `GL_PERIOD_NOT_OPEN` (แบบ AP)
- หลัง `reverseBySource` → `releaseReferences` (แบบ AP, CN เท่านั้น) → `tx.tb_ar_tax_invoice.updateMany({ where: { ar_invoice_id: doc.id }, data: { tax_status: enum_ar_tax_invoice_status.void, updated_at: now, updated_by_id: userId } })` → update header แบบ AP (`void_*`, `outstanding_amount: 0`, `doc_version: { increment: 1 }`)
- ไม่มีขั้น `detail_source.deleteMany`

- [ ] **Step 4: workflow actions ใน service**

copy `submit`, `approve`, `review`, `reject`, `void`, `loadForAction`, `guardedUpdate`, `workflowColumns`, `postDocument`, `sendToReview`, `voidDraft`, `voidPosted`, `resolveWorkflowId`, `withReason` จาก `ap-invoice.service.ts` แล้วแก้:
- `resolveWorkflowId`: `workflow_type: enum_workflow_type.ar_invoice`
- ใช้ `arInvoiceToWorkflowDocument` และ `ArInvoicePosting`
- `submit` ก่อนเปลี่ยนสถานะ (ใน `loadForAction` แล้ว):
  1. `this.posting.isPeriodOpen(this.prismaService, doc.doc_date)` → false = `GL_PERIOD_NOT_OPEN`
  2. `doc.total_amount.lte(0)` → `AR_INVOICE_LINE_INVALID` `{ sequence_no: 0, reason: 'total must be > 0' }`
  3. `mixedVatRate(doc)`
  4. `validateReferences(doc.tb_ar_invoice_reference.map(...), doc, doc.id)`
```ts
/**
 * A tax invoice carries one VAT rate (spec §10.2)
 * ใบกำกับภาษีหนึ่งใบมี VAT อัตราเดียว (spec §10.2)
 * @param doc - Document / เอกสาร
 * @returns ok or `AR_TAX_INVOICE_MIXED_VAT_RATE` / ok หรือ `AR_TAX_INVOICE_MIXED_VAT_RATE`
 */
function mixedVatRate(doc: ArInvoiceFull): Result<true> {
  if (!doc.is_tax_invoice) return Result.ok(true);
  const rates = new Set(
    doc.tb_ar_invoice_detail.filter((d) => !d.vat_amount.isZero()).map((d) => d.vat_rate.toString()),
  );
  return rates.size > 1
    ? Result.errorFromCatalog(ERROR_CATALOG.AR_TAX_INVOICE_MIXED_VAT_RATE)
    : Result.ok(true);
}
```
- `void`: `reason` ว่าง → `AR_INVOICE_LINE_INVALID` `{ sequence_no: 0, reason: 'void reason is required' }` (ถ้า AP ใช้ error อื่นสำหรับกรณีนี้ ให้ใช้ตัวที่คู่กันของ AR ใน catalog)
- ไม่ copy budget hook ของ AP (ถ้ามี)

- [ ] **Step 5: static check + commit**

```bash
bun run check-types
npx eslint --no-fix apps/micro-business/src/ar
git add apps/micro-business/src/ar
git commit -m "feat(ar): post AR documents to GL with tax invoice, CN netting and void"
```

---

### Task 9: AR invoice — controller, module, contract, activity registry

**Files:**
- Create: `apps/micro-business/src/ar/ar-invoice/ar-invoice.controller.ts`, `ar-invoice.module.ts`
- Modify: `apps/micro-business/src/app.module.ts` (ถัดจาก `ApInvoiceModule` บรรทัด ~114 และ ~418)
- Modify: `apps/micro-business/src/common/activity/activity-registry.ts` (ถัดจากบล็อก `tb_ap_payment`), `entity-snapshot.ts` (ถัดจาก `tb_ap_payment`)
- Generated: `packages/rpc-contract/src/contracts/ar-invoice.ts`, `index.ts`

**Interfaces:**
- Consumes: `ArInvoiceService` (Task 7–8)
- Produces: RPC `ar-invoice.find-all|find-one|create|update|delete|submit|approve|review|reject|void|reference-candidates`

- [ ] **Step 1: controller + module**

`ar-invoice.controller.ts` copy `ap-invoice.controller.ts` ตัด `grnCandidates`, `createFromGrn`; `ApInvoiceService` → `ArInvoiceService`; literal ชั่วคราว `@MessagePattern({ cmd: 'ar-invoice.<kebab-action>', service: 'micro-business' })` 11 ตัว; payload key เหมือน AP
`ar-invoice.module.ts` = module ของ AP (imports `TenantModule`, `CommonModule`, `ExchangeRateModule`, `GlPostingModule`, `GlSubledgerPostingModule`; controllers `ArInvoiceController`; providers `ArInvoiceService`, `ArInvoiceWriter`, `ArInvoicePosting`, `WorkflowOrchestratorService`; exports `ArInvoiceService`, `ArInvoicePosting`) — JSDoc ของ module เขียนใหม่ (ไม่ copy ข้อความเรื่อง AP payment)
เพิ่ม `ArInvoiceModule` ใน `app.module.ts`

- [ ] **Step 2: gen contract**

```bash
bun run gen:rpc-contract
```
แทน literal ด้วย `ArInvoice.<action>.pattern`; ตรวจว่า `packages/rpc-contract/src/contracts/ar-invoice.ts` มี 11 action

- [ ] **Step 3: activity registry + snapshot**

```ts
  {
    entityName: 'tb_ar_invoice',
    mutations: [
      ['ar-invoice.create', 'create', CREATED_ID],
      ['ar-invoice.update', 'update', EDITED_ID],
      ['ar-invoice.submit', 'submit', EDITED_ID],
      ['ar-invoice.approve', 'approve', EDITED_ID],
      ['ar-invoice.review', 'review', EDITED_ID],
      ['ar-invoice.reject', 'reject', EDITED_ID],
      // Same `void` AuditAction as ap-invoice, not folded into `update`.
      ['ar-invoice.void', 'void', EDITED_ID],
      ['ar-invoice.delete', 'delete', DELETED_ID],
    ],
  },
```
`entity-snapshot.ts` (รวมกับ `tb_customer` ใต้ `// accounts receivable` ถ้า Task 3 สร้างกลุ่มนี้แล้ว):
```ts
  tb_ar_invoice: {
    tb_ar_invoice_detail: { include: { tb_ar_invoice_detail_dimension: true } },
    tb_ar_invoice_reference: true,
    tb_ar_tax_invoice: true,
  },
```

- [ ] **Step 4: static check + audits + commit**

```bash
bun run build:package && bun run check-types
npx eslint --no-fix apps/micro-business/src/ar apps/micro-business/src/app.module.ts apps/micro-business/src/common/activity
bun run audit:tcp-drift && bun run audit:message-pattern-literal
git add apps/micro-business/src packages/rpc-contract
git commit -m "feat(ar): expose AR invoice over RPC with activity logging"
```

---

### Task 10: AR invoice — gateway

**Files:**
- Create: `apps/backend-gateway/src/application/ar-invoice/{ar-invoice.controller.ts,ar-invoice.service.ts,ar-invoice.module.ts,swagger/request.ts,swagger/response.ts}`
- Modify: `apps/backend-gateway/src/application/route-application.ts` (ถัดจาก `ApInvoiceModule`)
- Regenerate: `apps/backend-gateway/src/platform/applications/app-api-catalog.generated.ts`

**Interfaces:**
- Consumes: `ArInvoice.*` (Task 9)
- Produces: REST `api/:bu_code/ar-invoice` ตาม spec §12.2

แม่แบบ: `apps/backend-gateway/src/application/ap-invoice/*`

- [ ] **Step 1: copy และแก้**

copy `ap-invoice` → `ar-invoice` แล้ว:
- ตัด route `grn-candidates`, `from-grn`
- `const AR_RESOURCE = 'accounting.ar';` และ void `@Permission({ [AR_RESOURCE]: ['void'] })` (รูปเดียวกับ AP)
- `@Controller('api/:bu_code/ar-invoice')`, `@ApiTags('Accounting: AR Invoice')`, `AppIdGuard('ar-invoice.<camelAction>')`, `operationId: 'arInvoice_<action>'`, RPC `ArInvoice.*`, class `Ar*`; JSDoc ของ class เขียนเป็น AR (เอกสาร invoice / debit note / credit note)
- route `GET /:id/reference-candidates`
- `swagger/request.ts`: Zod ตาม `ICreateArInvoice` / `IUpdateArInvoice` / `IArInvoiceAction`: `doc_type: z.nativeEnum(enum_ar_invoice_doc_type)`, `doc_source: z.nativeEnum(enum_ar_invoice_source).optional()`, `doc_date: z.coerce.date()`, เงิน `z.union([z.number(), z.string()])`, `discount_pct` 0–100, `details: z.object({ add: z.array(ArLineSchema).min(1) })`, `references: z.array(ArReferenceSchema).nullish()`
- `swagger/response.ts`: field ของ `tb_ar_invoice` + details + references + `tax_invoice`
- เพิ่ม `ArInvoiceModule` ใน `route-application.ts`

- [ ] **Step 2: regenerate catalog + audits**

```bash
bun run scripts/generate-app-api-catalog/run.ts
bun run check-types
npx eslint --no-fix apps/backend-gateway/src/application/ar-invoice apps/backend-gateway/src/application/route-application.ts
bun run audit:api-system-permission && bun run audit:guard-providers && bun run audit:bu-scope-guard && bun run audit:app-api-catalog-drift && bun run audit:zod-dto-openapi && bun run audit:rest-contract
```
Expected: ผ่าน ถ้า audit เรื่อง permission บ่นว่า `accounting.ar` อยู่ใน `PLANNED_RESOURCES` ขณะมี endpoint แล้ว ให้ดูว่า AP ผ่าน audit เดียวกันได้อย่างไร (`accounting.ap` ก็ยังอยู่ในเซ็ตนั้น) แล้วทำเหมือนกัน ถ้าไม่ชัด หยุดและรายงาน

- [ ] **Step 3: Commit**

```bash
git add apps/backend-gateway/src/application apps/backend-gateway/src/platform/applications/app-api-catalog.generated.ts
git commit -m "feat(gateway): AR invoice endpoints"
```

---

### Task 11: Docs, gates, suite เดิม, Bruno, ตรวจด้วยมือ

**Files:**
- Modify: `apps/micro-business/CLAUDE.md` (bullet `tb_customer*` / `tb_ar_*` บรรทัด ~31 และหลัง bullet `gl_setting` ของ AP บรรทัด ~21)
- Create (repo `carmen-turborepo-backend-bruno`): folder `customer/`, `ar-invoice/`

- [ ] **Step 1: แก้ CLAUDE.md**

แทน bullet "`tb_customer*` and `tb_ar_*` ... are schema-only ..." ด้วย:
```md
- **AR has a service for customer master + ARIV/ARDN/ARCN** (`src/master/customers`, `src/ar/ar-invoice`; spec `carmen-accounting-concept/docs/superpowers/specs/2026-10-01-accounting-ar-invoice-service-design.md`). Deposits (ARDP), receipts (`tb_ar_receipt*`) and PMS folio are still schema-only — `checkArHeader` rejects `deposit` and non-manual sources. Tax-invoice numbers (`TX-IV/DN/CN`) are issued at post, never at submit. A credit note can be applied to an invoice only at the same exchange rate; the only GL it adds is a base-rounding AR ↔ realized-FX pair. `tb_ar_tax_invoice` has nullable `ar_invoice_id` / `ar_receipt_id`; exactly one must be set, enforced by the service.
```
เพิ่มหลัง bullet `gl_setting` ของ AP:
```md
- **`gl_setting` keys AR posting reads:** `ar_control_account_id` (only when neither the line nor the customer has an AR account), `output_vat_account_id` (only when a VAT line has no account on the line or its profile), `ar_jv_prefix_id` (else `auto_jv_prefix_id`), and `realized_fx_gain_account_id` / `realized_fx_loss_account_id` only when credit-note netting leaves a base rounding difference. Tax 2 has no `gl_setting` fallback — its account comes from the line or the tax profile.
```

- [ ] **Step 2: gates + suite เดิม**

```bash
bun run build:package && bun run check-types && bun run gates
bunx turbo run test --filter=micro-business --filter=backend-gateway --filter=@repo/error-catalog --filter=@repo/prisma-shared-schema-tenant
```
suite ใหญ่ใช้เวลา > 30 นาที ให้รันแบบ background แล้วรอผล — Expected: ผ่านทั้งหมด ถ้า test เดิมพังเพราะ facade/GlSetting/running code ให้แก้โค้ด (ไม่แก้ test) แล้วรายงาน

- [ ] **Step 3: Bruno**

ใน `carmen-turborepo-backend-bruno` copy folder ของ vendor → `customer` และ `ap-invoice` → `ar-invoice` แก้ URL/body ตาม route ใหม่ มี request: customer create/list/get/update/delete; ar-invoice create ARIV (discount_pct, VAT, tax2, is_tax_invoice), create ARCN, update, submit, approve, review, reject, void, reference-candidates, list, get · commit ใน repo bruno (branch ตาม convention ของ repo นั้น) — ถามผู้ใช้ก่อน push

- [ ] **Step 4: ตรวจด้วยมือบน BU ทดสอบ (spec §14)**

ถามผู้ใช้ว่าใช้ BU ไหนเป็น BU ทดสอบ แล้ว deploy migration เฉพาะ BU นั้น (`POST /api-system/tenant/migrations/:bu_id/deploy` ครอบทั้ง `accounting_ar_tables` และ `accounting_ar_service_enums`) จากนั้นทำตาม spec §14 ข้อ 1–7 ทีละข้อ บันทึกผล (request/response ย่อ + query ตรวจ `tb_gl_jv_header`, `tb_gl_jv_detail`, `tb_ar_tax_invoice`, `tb_ar_invoice.outstanding_amount`) เพิ่มกรณี Review Focus:
  - RF1: ARIV 2 ใบ draft อ้าง CN ใบเดียวกันเต็มยอด → approve ใบแรก → approve ใบที่สองต้องได้ `AR_REFERENCE_OVER_APPLIED`
  - RF2: submit → send back → submit → approve: `tb_ar_tax_invoice` มีแถวเดียว เลขต่อจากใบก่อนหน้าไม่ข้าม
  - RF3: ARIV rate 35.12345 สามบรรทัดมี tax2 → post ผ่าน ไม่มี `GL_JV_NOT_BALANCED`
  - RF4: update draft ที่มี reference แล้วเปลี่ยน customer → `AR_CUSTOMER_CURRENCY_MISMATCH`
  - RF5: post AP invoice 1 ใบ → JV ใช้ prefix `ap_jv_prefix_id`

ห้าม deploy ลง BU อื่นโดยไม่ถามผู้ใช้

- [ ] **Step 5: Commit + เตรียม PR (ถามผู้ใช้ก่อน push)**

```bash
git add apps/micro-business/CLAUDE.md
git commit -m "docs(micro-business): AR invoice service notes and gl_setting keys"
git log --oneline main..HEAD
```
สรุปผลให้ผู้ใช้ แล้วถามว่าจะ push + เปิด PR หรือไม่ PR body: สรุป, ลิงก์ spec, ขั้น deploy ราย BU (spec §12.6), ผลตรวจมือ Step 4, ตารางเบี่ยงจาก FRD (spec §13)

---

## Self-review (ทำแล้ว)

- **spec coverage:** §4 → Task 1 · §5 → Task 3–4 · §6 → Task 7–8 · §7 → Task 5–6 · §8 → Task 5, 8 · §9 → Task 7–8 · §10 → Task 5, 8 · §11 → Task 8 · §12.1–12.3 → Task 3–4, 9–10 · §12.4 → Task 2 · §12.5 → Task 3, 8, 9, 11 · §12.6 / §14 → Task 11 · §15 ข้อ 1 ทำหลัง merge (นอก plan)
- **เบี่ยงจาก spec ที่ตั้งใจ:** (1) เลขใบกำกับไม่ retry — advisory lock กันการชนแล้ว ชนหลัง lock → `AR_TAX_INVOICE_NO_CONFLICT` (Task 8 Step 2) · (2) เพิ่ม `AR_INVOICE_LINE_INVALID` ใน catalog สำหรับบรรทัดผิดทั่วไปแบบ AP (Task 2) · (3) `base_net = base_sub_total − base_discount` แบบ AP แทน `round(net × rate)` เพื่อให้ตรงกับ AP (Task 5)
- **type consistency:** `computeArLine` / `IArComputedLine` / `sumArLines` (Task 5) → writer (Task 6), service (Task 7) · `ArDocType` / `AR_DOC_TYPE_RUNNING_CODE` / `generateArDocNo` / `generateArTaxInvoiceNo` (Task 5) → writer (Task 6), posting (Task 8) · `arReferenceViolation` / `arReferenceAvailable` (Task 7) → posting (Task 8) · `IArJvBuildContext` ไม่มี field บัญชี VAT — บัญชี resolve ใน `jvDetails` (Task 8) ก่อนเข้า `buildArInvoiceJvLines` · `SubledgerSourceRefType 'ar_invoice'` (Task 1) → posting (Task 8)
