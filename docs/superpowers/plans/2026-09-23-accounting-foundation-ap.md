# Accounting Foundation Gap + AP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** เพิ่ม accounting dimension foundation, subledger posting facade, และ AP module (invoice/DN/CN/deposit + payment voucher) เข้าไปใน `micro-business` ของ `carmen-turborepo-backend-v2` โดยต่อยอด GL ที่มีอยู่

**Architecture:** ทุกอย่างอยู่ใน `apps/micro-business/src/` เป็น flat `@Module` ต่อ feature เหมือน `gl-*` เดิม AP ไม่เขียน `tb_gl_*` ตรง แต่เรียก `GlSubledgerPostingService.postFromSource()` ซึ่งห่อ `GlJvService.createInternal()` + `GlPostingService.postInTx()` ใน transaction ของ AP เอง workflow ใช้ `WorkflowOrchestratorService` เดิมผ่าน mapper ต่อเอกสาร gateway เป็น controller บาง ๆ ที่เรียก RPC ผ่าน `RpcClient`

**Tech Stack:** NestJS 11, Bun, Turborepo, Prisma (`packages/prisma-shared-schema-tenant`), Zod v4 + `nestjs-zod`, `@repo/rpc-contract` (generated), `@repo/error-catalog`, PostgreSQL

**Spec:** `docs/superpowers/specs/2026-09-23-accounting-foundation-ap-design.md` (repo `carmen-accounting-concept`) — plan นี้อ้าง §ของ spec ตลอด อ่านคู่กัน

**Repo ที่แก้:** `/Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2` ทุก path ด้านล่างสัมพัทธ์กับ root ของ repo นี้ เว้นแต่ระบุ

## Global Constraints

- **ไม่มี test step ใน plan นี้** ตาม preference ของผู้ใช้ (`~/.claude/CLAUDE.md`): ทุก task จบด้วย `bun run check-types` (หรือ `tsc --noEmit` ใน app ที่แก้) + `npx eslint --no-fix <ไฟล์ที่แก้>` + commit; ไม่สร้าง `*.spec.ts`; ตรวจพฤติกรรมด้วยมือผ่าน curl ใน Task 7, 11, 15, 17
- Branch: `feature/accounting-foundation-ap` แตกจาก `main` (repo นี้ integrate บน `main`)
- `bun run build:package` ต้องรันก่อน `check-types`/`dev` ทุกครั้งที่แก้ `packages/*` มิฉะนั้นเจอ `TS2307 cannot find module @repo/*`
- ห้ามแก้ `packages/rpc-contract/src/contracts/*.ts` ด้วยมือ ใช้ 3 ขั้น: handler ใช้ literal `@MessagePattern({ cmd, service })` ชั่วคราว → `bun run gen:rpc-contract` → แทนที่ด้วย `<Service>.<action>.pattern`
- Enum ทุกตัวมาจาก Prisma เท่านั้น: Zod ใช้ `z.nativeEnum(enum_x)` ห้าม `z.enum([...])`
- naming: class PascalCase, ไฟล์ kebab-case, snake_case เฉพาะ DB/wire, ฟังก์ชัน ≤ 20 statements, boolean ขึ้นต้น `is/has/can`, JSDoc สองภาษา (EN บรรทัดแรก TH บรรทัดถัดไป) ทุก export
- จำนวนเงินเป็น `Prisma.Decimal` เสมอ ห้าม `number` ในการคำนวณ; ปัด `toDecimalPlaces(2, Prisma.Decimal.ROUND_HALF_UP)`
- เขียน `Date` ลง Prisma `DateTime` ตรง ๆ ไม่ `.toISOString()`
- ทุก RPC handler ห่อด้วย `this.ctx.run(payload, () => ...)` (`audit:tenant-context`)
- ทุก service ที่ทำ workflow action ไม่มี `@Permission` บน gateway; เฉพาะ `void` ใช้ `@Permission({ 'accounting.ap': ['void'] })`
- ห้าม commit secret; ห้าม `bun run lint` ระดับ root (autofix ข้ามไฟล์) ใช้ eslint แบบ scoped

## Review Focus

spec นิ่งเรื่องเหล่านี้แต่ผู้ใช้จริงจะเจอ; ไม่มี test อัตโนมัติจึงระบุ task ที่ต้องกันไว้ในโค้ดและตรวจด้วยมือ:

1. **ยอดหลังปัดไม่สมดุลใน base currency** (invoice หลายบรรทัด rate 35.12345): Σ base ต่อบรรทัดที่ปัดแล้วอาจต่าง Cr AP ที่ปัดจากยอดรวม → Task 8 `buildInvoiceJvLines` ต้องคำนวณ Cr AP base = Σ base Dr ต่อบรรทัด (ไม่ปัดยอดรวมซ้ำ) และ facade ปฏิเสธถ้าไม่สมดุล (Task 7)
2. **submit ซ้ำสองครั้งพร้อมกัน** (double click) ต้องได้ JV ใบเดียว → Task 7 idempotency + partial unique index Task 2; Task 10/14 ใช้ `doc_version` ใน `where` ของ update
3. **GRN ถูก void หลังสร้าง invoice draft** → Task 9 ตรวจ `doc_status = committed` ซ้ำตอน submit และ post ไม่ใช่แค่ตอน create
4. **PV จ่ายบรรทัดเดิมสองใบ draft พร้อมกัน** → Task 14 ล็อกบรรทัด invoice ด้วย `FOR UPDATE` ก่อนตรวจ unpaid ทั้งตอน submit และ post
5. **void invoice ที่ถูกใบอื่นอ้าง (reference) หรือมี PV** ต้องถูกปฏิเสธด้วย error ที่บอกเหตุ ไม่ใช่ FK error → Task 10 ตรวจ `reference_applied_amount` และ allocation ก่อน reverse

---

### Task 0: Branch, baseline, และ MODULE ids ที่ว่าง

**Files:**
- ไม่แก้ไฟล์ (ตรวจสภาพก่อนเริ่ม)

- [ ] **Step 1: แตก branch และตรวจ baseline**

```bash
cd /Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2
git checkout main && git pull
git checkout -b feature/accounting-foundation-ap
bun install
bun run build:package
bun run check-types
```
Expected: check-types ผ่านทุก app ก่อนแก้อะไร ถ้าไม่ผ่านให้หยุดและรายงาน

- [ ] **Step 2: ยืนยันว่า MODULE id 616–621 ยังว่าง**

```bash
grep -n "61[6-9]\|62[0-1]" packages/error-catalog/src/module.ts
```
Expected: ไม่มีผล ถ้ามี ให้เลื่อนเลขที่ใช้ใน Task 3 ไปช่วงถัดไปที่ว่าง และใช้เลขนั้นแทนทุกที่

- [ ] **Step 3: ยืนยัน DB dev ใช้ได้สำหรับ migrate**

```bash
cd packages/prisma-shared-schema-tenant && cat .env 2>/dev/null | grep -c DATABASE_URL; cd ../..
```
Expected: 1 (มี `DATABASE_URL` ชี้ tenant schema ของ dev) ถ้าไม่มี ให้ขอค่าจากผู้ใช้ก่อนเริ่ม Task 1

---

### Task 1: Prisma schema — foundation (dimension, bank account, extend tax profile/vendor/workflow)

**Files:**
- Modify: `packages/prisma-shared-schema-tenant/prisma/schema.prisma`

**Interfaces:**
- Produces: models `tb_gl_dimension`, `tb_gl_dimension_value`, `tb_gl_account_dimension_rule`, `tb_gl_jv_detail_dimension`, `tb_bank_account`; enums `enum_gl_account_dimension_rule_requirement`, `enum_tax_profile_tax_type`, `enum_tax_profile_wht_pnd_form`; ค่าใหม่ `enum_workflow_type.ap_invoice|ap_payment`; ฟิลด์ใหม่บน `tb_tax_profile`, `tb_vendor`; relation `tb_gl_jv_detail.tb_gl_jv_detail_dimension`

- [ ] **Step 1: เพิ่ม enum ใหม่** แทนที่บล็อก `enum enum_workflow_type` เดิม (schema.prisma ~L276) และเพิ่ม enum อีก 3 ตัวต่อท้าย

```prisma
enum enum_workflow_type {
  purchase_request
  store_requisition
  purchase_order
  gl_jv
  ap_invoice
  ap_payment
}

enum enum_gl_account_dimension_rule_requirement {
  mandatory
  optional
  prohibited
}

enum enum_tax_profile_tax_type {
  vat
  wht
}

enum enum_tax_profile_wht_pnd_form {
  pnd_1
  pnd_2
  pnd_3
  pnd_53
  pnd_54
}
```

- [ ] **Step 2: extend `tb_tax_profile`** เพิ่มหลัง `tax_rate`

```prisma
  tax_type             enum_tax_profile_tax_type      @default(vat)
  chart_of_accounts_id String?                        @db.Uuid
  wht_pnd_form         enum_tax_profile_wht_pnd_form?
  wht_income_type      String?                        @db.VarChar
```
และเพิ่ม relation ในบล็อก relations ของ `tb_tax_profile`:
```prisma
  tb_chart_of_accounts tb_chart_of_accounts? @relation(fields: [chart_of_accounts_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
```
และใน `tb_chart_of_accounts` เพิ่ม `tb_tax_profile tb_tax_profile[]`

- [ ] **Step 3: extend `tb_vendor`** เพิ่มหลัง `branch_no`

```prisma
  default_currency_id     String? @db.Uuid
  default_currency_code   String? @db.VarChar(3)
  credit_term_id          String? @db.Uuid
  credit_term_name        String? @db.VarChar
  credit_term_days        Int?    @db.Integer
  ap_chart_of_accounts_id String? @db.Uuid
```
(ไม่ใส่ relation เพื่อไม่แตะ relation list ของ `tb_currency`/`tb_credit_term`; ตรวจความมีอยู่ใน service)

- [ ] **Step 4: เพิ่ม model foundation 5 ตัว** ต่อท้ายไฟล์

```prisma
model tb_gl_dimension {
  doc_version Int     @default(0) @db.Integer
  id          String  @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  code        String  @db.VarChar(20)
  name        String  @db.VarChar
  name_local  String? @db.VarChar
  description String? @db.VarChar
  sequence    Int     @default(0) @db.Integer
  is_active   Boolean @default(true)

  info      Json? @default("{}") @db.JsonB
  dimension Json? @default("[]") @db.JsonB

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid
  updated_at    DateTime? @default(now()) @db.Timestamptz(6)
  updated_by_id String?   @db.Uuid
  deleted_at    DateTime? @db.Timestamptz(6)
  deleted_by_id String?   @db.Uuid

  tb_gl_dimension_value          tb_gl_dimension_value[]
  tb_gl_account_dimension_rule   tb_gl_account_dimension_rule[]
  tb_gl_jv_detail_dimension      tb_gl_jv_detail_dimension[]
  tb_ap_invoice_detail_dimension tb_ap_invoice_detail_dimension[]

  @@unique([code, deleted_at], map: "gldimension_code_u")
  @@index([code], map: "gldimension_code_idx")
}

model tb_gl_dimension_value {
  doc_version     Int       @default(0) @db.Integer
  id              String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  gl_dimension_id String    @db.Uuid
  code            String    @db.VarChar(30)
  name            String    @db.VarChar
  name_local      String?   @db.VarChar
  effective_from  DateTime? @db.Date
  effective_to    DateTime? @db.Date
  is_active       Boolean   @default(true)

  info      Json? @default("{}") @db.JsonB
  dimension Json? @default("[]") @db.JsonB

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid
  updated_at    DateTime? @default(now()) @db.Timestamptz(6)
  updated_by_id String?   @db.Uuid
  deleted_at    DateTime? @db.Timestamptz(6)
  deleted_by_id String?   @db.Uuid

  tb_gl_dimension                tb_gl_dimension                  @relation(fields: [gl_dimension_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_gl_jv_detail_dimension      tb_gl_jv_detail_dimension[]
  tb_ap_invoice_detail_dimension tb_ap_invoice_detail_dimension[]

  @@unique([gl_dimension_id, code, deleted_at], map: "gldimensionvalue_code_u")
  @@index([gl_dimension_id], map: "gldimensionvalue_dimension_idx")
}

model tb_gl_account_dimension_rule {
  doc_version          Int                                        @default(0) @db.Integer
  id                   String                                     @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  chart_of_accounts_id String                                     @db.Uuid
  gl_dimension_id      String                                     @db.Uuid
  requirement          enum_gl_account_dimension_rule_requirement @default(optional)

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid
  updated_at    DateTime? @default(now()) @db.Timestamptz(6)
  updated_by_id String?   @db.Uuid
  deleted_at    DateTime? @db.Timestamptz(6)
  deleted_by_id String?   @db.Uuid

  tb_chart_of_accounts tb_chart_of_accounts @relation(fields: [chart_of_accounts_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_gl_dimension      tb_gl_dimension      @relation(fields: [gl_dimension_id], references: [id], onDelete: NoAction, onUpdate: NoAction)

  @@unique([chart_of_accounts_id, gl_dimension_id, deleted_at], map: "glaccountdimensionrule_u")
  @@index([chart_of_accounts_id], map: "glaccountdimensionrule_account_idx")
}

model tb_gl_jv_detail_dimension {
  id                    String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  gl_jv_detail_id       String @db.Uuid
  gl_dimension_id       String @db.Uuid
  gl_dimension_value_id String @db.Uuid
  dimension_code        String @db.VarChar(20)
  value_code            String @db.VarChar(30)

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid

  tb_gl_jv_detail       tb_gl_jv_detail       @relation(fields: [gl_jv_detail_id], references: [id], onDelete: Cascade, onUpdate: NoAction)
  tb_gl_dimension       tb_gl_dimension       @relation(fields: [gl_dimension_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_gl_dimension_value tb_gl_dimension_value @relation(fields: [gl_dimension_value_id], references: [id], onDelete: NoAction, onUpdate: NoAction)

  @@unique([gl_jv_detail_id, gl_dimension_id], map: "gljvdetaildimension_u")
  @@index([gl_dimension_value_id], map: "gljvdetaildimension_value_idx")
}

model tb_bank_account {
  doc_version          Int     @default(0) @db.Integer
  id                   String  @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  code                 String  @db.VarChar(20)
  name                 String  @db.VarChar
  bank_name            String? @db.VarChar
  bank_branch          String? @db.VarChar
  account_no           String  @db.VarChar(50)
  currency_id          String  @db.Uuid
  currency_code        String  @db.VarChar(3)
  chart_of_accounts_id String  @db.Uuid
  is_active            Boolean @default(true)
  description          String? @db.VarChar
  note                 String? @db.VarChar

  info      Json? @default("{}") @db.JsonB
  dimension Json? @default("[]") @db.JsonB

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid
  updated_at    DateTime? @default(now()) @db.Timestamptz(6)
  updated_by_id String?   @db.Uuid
  deleted_at    DateTime? @db.Timestamptz(6)
  deleted_by_id String?   @db.Uuid

  tb_chart_of_accounts tb_chart_of_accounts @relation(fields: [chart_of_accounts_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_ap_payment        tb_ap_payment[]

  @@unique([code, deleted_at], map: "bankaccount_code_u")
  @@index([code], map: "bankaccount_code_idx")
}
```

- [ ] **Step 5: เพิ่ม relation ฝั่งตรงข้าม**
  - ใน `tb_gl_jv_detail` เพิ่ม `tb_gl_jv_detail_dimension tb_gl_jv_detail_dimension[]`
  - ใน `tb_chart_of_accounts` เพิ่ม `tb_gl_account_dimension_rule tb_gl_account_dimension_rule[]` และ `tb_bank_account tb_bank_account[]`
  - relation `tb_ap_invoice_detail_dimension` และ `tb_ap_payment` อ้าง model ที่จะเพิ่มใน Task 2 ดังนั้น **ทำ Task 1 และ Task 2 ต่อกันแล้ว format/migrate ครั้งเดียวใน Task 2 Step 3** (ไม่ commit แยกระหว่างกลาง)

---

### Task 2: Prisma schema — AP tables, index, migration, config keys

**Files:**
- Modify: `packages/prisma-shared-schema-tenant/prisma/schema.prisma`
- Modify: `packages/prisma-shared-schema-tenant/src/client.ts` (excludeModels ~L102)
- Modify: `apps/micro-business/src/app-config/app-config.service.ts:87-93` (GlSettingSchema)
- Modify: `apps/micro-business/src/master/running-code/const/running-code.const.ts`
- Create: migration `packages/prisma-shared-schema-tenant/prisma/migrations/<timestamp>_accounting_foundation_ap/migration.sql`

**Interfaces:**
- Produces: models `tb_ap_invoice`, `tb_ap_invoice_detail`, `tb_ap_invoice_detail_dimension`, `tb_ap_invoice_detail_source`, `tb_ap_invoice_reference`, `tb_ap_invoice_tax`, `tb_ap_payment`, `tb_ap_payment_detail`, `tb_ap_payment_wht`, `tb_ap_payment_expense`; enums `enum_ap_invoice_doc_type`, `enum_ap_invoice_status`, `enum_ap_invoice_source`, `enum_ap_invoice_tax_status`, `enum_ap_payment_status`, `enum_ap_payment_method`; `gl_setting` keys ใหม่; running-code type `AP-IV/AP-DN/AP-CN/AP-DP/AP-PV`

- [ ] **Step 1: เพิ่ม enum AP**

```prisma
enum enum_ap_invoice_doc_type {
  invoice
  debit_note
  credit_note
  deposit
}

enum enum_ap_invoice_status {
  draft
  in_review
  posted
  void
}

enum enum_ap_invoice_source {
  manual
  grn
  copy
  excel
  ai
  interface
}

enum enum_ap_invoice_tax_status {
  pending
  on_review
  unclaim
  confirmed
  submitted
  void
}

enum enum_ap_payment_status {
  draft
  in_review
  posted
  void
}

enum enum_ap_payment_method {
  bank_transfer
  cheque
  cash
  credit_card
  promptpay
}
```

- [ ] **Step 2: เพิ่ม model AP 10 ตัว** (spec §4.3)

```prisma
model tb_ap_invoice {
  doc_version Int    @default(0) @db.Integer
  id          String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid

  doc_no      String                   @db.VarChar(30)
  doc_type    enum_ap_invoice_doc_type
  doc_status  enum_ap_invoice_status   @default(draft)
  doc_source  enum_ap_invoice_source   @default(manual)
  doc_date    DateTime                 @db.Date
  description String?                  @db.VarChar

  vendor_id         String   @db.Uuid
  vendor_name       String   @db.VarChar
  vendor_invoice_no String   @db.VarChar(100)
  invoice_date      DateTime @db.Date
  credit_term_id    String?  @db.Uuid
  credit_term_name  String?  @db.VarChar
  credit_term_days  Int      @default(0) @db.Integer
  due_date          DateTime @db.Date

  currency_id        String  @db.Uuid
  currency_code      String  @db.VarChar(3)
  exchange_rate      Decimal @default(1) @db.Decimal(15, 5)
  base_currency_id   String  @db.Uuid
  base_currency_code String  @db.VarChar(3)

  sub_total_amount         Decimal @default(0) @db.Decimal(20, 5)
  discount_amount          Decimal @default(0) @db.Decimal(20, 5)
  net_amount               Decimal @default(0) @db.Decimal(20, 5)
  vat_amount               Decimal @default(0) @db.Decimal(20, 5)
  total_amount             Decimal @default(0) @db.Decimal(20, 5)
  base_sub_total_amount    Decimal @default(0) @db.Decimal(20, 5)
  base_discount_amount     Decimal @default(0) @db.Decimal(20, 5)
  base_net_amount          Decimal @default(0) @db.Decimal(20, 5)
  base_vat_amount          Decimal @default(0) @db.Decimal(20, 5)
  base_total_amount        Decimal @default(0) @db.Decimal(20, 5)
  wht_estimate_amount      Decimal @default(0) @db.Decimal(20, 5)
  outstanding_amount       Decimal @default(0) @db.Decimal(20, 5)
  reference_applied_amount Decimal @default(0) @db.Decimal(20, 5)

  gl_jv_id     String?   @db.Uuid
  gl_jv_no     String?   @db.VarChar
  posted_at    DateTime? @db.Timestamptz(6)
  posted_by_id String?   @db.Uuid

  void_at       DateTime? @db.Timestamptz(6)
  void_by_id    String?   @db.Uuid
  void_reason   String?   @db.VarChar
  void_gl_jv_id String?   @db.Uuid

  workflow_id             String?           @db.Uuid
  workflow_name           String?           @db.VarChar
  workflow_history        Json?             @default("[]") @db.JsonB
  workflow_current_stage  String?           @db.VarChar
  workflow_previous_stage String?           @db.VarChar
  workflow_next_stage     String?           @db.VarChar
  user_action             Json?             @db.JsonB
  last_action             enum_last_action?
  last_action_at_date     DateTime?         @db.Timestamptz(6)
  last_action_by_id       String?           @db.Uuid
  last_action_by_name     String?           @db.VarChar

  attachments Json? @default("[]") @db.JsonB
  info        Json? @default("{}") @db.JsonB
  dimension   Json? @default("[]") @db.JsonB

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid
  updated_at    DateTime? @default(now()) @db.Timestamptz(6)
  updated_by_id String?   @db.Uuid
  deleted_at    DateTime? @db.Timestamptz(6)
  deleted_by_id String?   @db.Uuid

  tb_ap_invoice_detail        tb_ap_invoice_detail[]
  tb_ap_invoice_reference     tb_ap_invoice_reference[] @relation("ApInvoiceReferences")
  tb_ap_invoice_referenced_by tb_ap_invoice_reference[] @relation("ApInvoiceReferencedBy")
  tb_ap_invoice_tax           tb_ap_invoice_tax?
  tb_ap_payment_detail        tb_ap_payment_detail[]

  @@unique([doc_no, deleted_at], map: "apinvoice_doc_no_u")
  @@index([vendor_id, doc_status], map: "apinvoice_vendor_status_idx")
  @@index([doc_date], map: "apinvoice_doc_date_idx")
  @@index([gl_jv_id], map: "apinvoice_gl_jv_idx")
}

model tb_ap_invoice_detail {
  doc_version   Int     @default(0) @db.Integer
  id            String  @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ap_invoice_id String  @db.Uuid
  sequence_no   Int     @db.Integer
  description   String? @db.VarChar
  product_id    String? @db.Uuid
  product_name  String? @db.VarChar
  unit_id       String? @db.Uuid
  unit_name     String? @db.VarChar
  quantity      Decimal @default(1) @db.Decimal(20, 5)
  unit_price    Decimal @default(0) @db.Decimal(20, 5)

  sub_total_amount Decimal @default(0) @db.Decimal(20, 5)
  discount_amount  Decimal @default(0) @db.Decimal(20, 5)
  net_amount       Decimal @default(0) @db.Decimal(20, 5)

  dr_chart_of_accounts_id String  @db.Uuid
  dr_account_code         String? @db.VarChar
  dr_cost_center_id       String? @db.Uuid
  dr_cost_center_code     String? @db.VarChar

  vat_tax_profile_id       String? @db.Uuid
  vat_rate                 Decimal @default(0) @db.Decimal(15, 5)
  vat_amount               Decimal @default(0) @db.Decimal(20, 5)
  vat_is_override          Boolean @default(false)
  vat_chart_of_accounts_id String? @db.Uuid
  vat_cost_center_id       String? @db.Uuid

  wht_tax_profile_id  String?  @db.Uuid
  wht_rate            Decimal? @db.Decimal(15, 5)
  wht_estimate_amount Decimal  @default(0) @db.Decimal(20, 5)

  cr_chart_of_accounts_id String  @db.Uuid
  cr_account_code         String? @db.VarChar
  cr_cost_center_id       String? @db.Uuid

  total_amount          Decimal @default(0) @db.Decimal(20, 5)
  unpaid_amount         Decimal @default(0) @db.Decimal(20, 5)
  base_sub_total_amount Decimal @default(0) @db.Decimal(20, 5)
  base_discount_amount  Decimal @default(0) @db.Decimal(20, 5)
  base_net_amount       Decimal @default(0) @db.Decimal(20, 5)
  base_vat_amount       Decimal @default(0) @db.Decimal(20, 5)
  base_total_amount     Decimal @default(0) @db.Decimal(20, 5)
  base_unpaid_amount    Decimal @default(0) @db.Decimal(20, 5)

  info      Json? @default("{}") @db.JsonB
  dimension Json? @default("[]") @db.JsonB

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid
  updated_at    DateTime? @default(now()) @db.Timestamptz(6)
  updated_by_id String?   @db.Uuid
  deleted_at    DateTime? @db.Timestamptz(6)
  deleted_by_id String?   @db.Uuid

  tb_ap_invoice                  tb_ap_invoice                    @relation(fields: [ap_invoice_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_ap_invoice_detail_dimension tb_ap_invoice_detail_dimension[]
  tb_ap_invoice_detail_source    tb_ap_invoice_detail_source[]
  tb_ap_payment_detail           tb_ap_payment_detail[]

  @@unique([ap_invoice_id, sequence_no, deleted_at], map: "apinvoicedetail_seq_u")
  @@index([ap_invoice_id], map: "apinvoicedetail_invoice_idx")
}

model tb_ap_invoice_detail_dimension {
  id                    String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ap_invoice_detail_id  String @db.Uuid
  gl_dimension_id       String @db.Uuid
  gl_dimension_value_id String @db.Uuid
  dimension_code        String @db.VarChar(20)
  value_code            String @db.VarChar(30)

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid

  tb_ap_invoice_detail  tb_ap_invoice_detail  @relation(fields: [ap_invoice_detail_id], references: [id], onDelete: Cascade, onUpdate: NoAction)
  tb_gl_dimension       tb_gl_dimension       @relation(fields: [gl_dimension_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_gl_dimension_value tb_gl_dimension_value @relation(fields: [gl_dimension_value_id], references: [id], onDelete: NoAction, onUpdate: NoAction)

  @@unique([ap_invoice_detail_id, gl_dimension_id], map: "apinvoicedetaildimension_u")
}

model tb_ap_invoice_detail_source {
  id                                String  @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ap_invoice_detail_id              String  @db.Uuid
  good_received_note_id             String  @db.Uuid
  grn_no                            String  @db.VarChar
  good_received_note_detail_item_id String  @db.Uuid
  matched_qty                       Decimal @default(0) @db.Decimal(20, 5)
  grn_unit_price                    Decimal @default(0) @db.Decimal(20, 5)
  matched_amount                    Decimal @default(0) @db.Decimal(20, 5)
  variance_amount                   Decimal @default(0) @db.Decimal(20, 5)

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid

  tb_ap_invoice_detail tb_ap_invoice_detail @relation(fields: [ap_invoice_detail_id], references: [id], onDelete: Cascade, onUpdate: NoAction)

  @@index([good_received_note_detail_item_id], map: "apinvoicedetailsource_grn_item_idx")
  @@index([good_received_note_id], map: "apinvoicedetailsource_grn_idx")
}

model tb_ap_invoice_reference {
  id                           String                   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ap_invoice_id                String                   @db.Uuid
  ref_ap_invoice_id            String                   @db.Uuid
  ref_doc_type                 enum_ap_invoice_doc_type
  applied_amount               Decimal                  @default(0) @db.Decimal(20, 5)
  base_applied_at_invoice_rate Decimal                  @default(0) @db.Decimal(20, 5)
  base_applied_at_ref_rate     Decimal                  @default(0) @db.Decimal(20, 5)
  realized_fx_amount           Decimal                  @default(0) @db.Decimal(20, 5)

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid

  tb_ap_invoice     tb_ap_invoice @relation("ApInvoiceReferences", fields: [ap_invoice_id], references: [id], onDelete: Cascade, onUpdate: NoAction)
  tb_ref_ap_invoice tb_ap_invoice @relation("ApInvoiceReferencedBy", fields: [ref_ap_invoice_id], references: [id], onDelete: NoAction, onUpdate: NoAction)

  @@unique([ap_invoice_id, ref_ap_invoice_id], map: "apinvoicereference_u")
  @@index([ref_ap_invoice_id], map: "apinvoicereference_ref_idx")
}

model tb_ap_invoice_tax {
  doc_version       Int                        @default(0) @db.Integer
  id                String                     @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ap_invoice_id     String                     @unique(map: "apinvoicetax_invoice_u") @db.Uuid
  tax_invoice_no    String                     @db.VarChar(100)
  tax_invoice_date  DateTime                   @db.Date
  vendor_tax_no     String?                    @db.VarChar
  vendor_branch_no  String?                    @db.VarChar
  vendor_name       String                     @db.VarChar
  base_amount       Decimal                    @default(0) @db.Decimal(20, 5)
  vat_rate          Decimal                    @default(0) @db.Decimal(15, 5)
  vat_amount        Decimal                    @default(0) @db.Decimal(20, 5)
  tax_status        enum_ap_invoice_tax_status @default(pending)
  expiry_claim_date DateTime                   @db.Date
  filing_month      Int?                       @db.Integer
  filing_year       Int?                       @db.Integer

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid
  updated_at    DateTime? @default(now()) @db.Timestamptz(6)
  updated_by_id String?   @db.Uuid

  tb_ap_invoice tb_ap_invoice @relation(fields: [ap_invoice_id], references: [id], onDelete: Cascade, onUpdate: NoAction)

  @@index([tax_invoice_date], map: "apinvoicetax_date_idx")
}

model tb_ap_payment {
  doc_version Int    @default(0) @db.Integer
  id          String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid

  doc_no       String                 @db.VarChar(30)
  doc_status   enum_ap_payment_status @default(draft)
  payment_date DateTime               @db.Date
  paid_date    DateTime?              @db.Date
  description  String?                @db.VarChar
  reference_no String?                @db.VarChar(100)

  vendor_id       String  @db.Uuid
  vendor_name     String  @db.VarChar
  payee_name      String  @db.VarChar
  payee_tax_no    String? @db.VarChar(20)
  payee_branch_no String? @db.VarChar(5)
  payee_address   String? @db.VarChar

  bank_account_id   String                 @db.Uuid
  bank_account_code String                 @db.VarChar(20)
  bank_account_name String                 @db.VarChar
  payment_method    enum_ap_payment_method @default(bank_transfer)
  cheque_no         String?                @db.VarChar(50)
  cheque_date       DateTime?              @db.Date

  currency_id        String  @db.Uuid
  currency_code      String  @db.VarChar(3)
  exchange_rate      Decimal @default(1) @db.Decimal(15, 5)
  base_currency_id   String  @db.Uuid
  base_currency_code String  @db.VarChar(3)

  total_applied_amount      Decimal @default(0) @db.Decimal(20, 5)
  total_wht_amount          Decimal @default(0) @db.Decimal(20, 5)
  total_expense_amount      Decimal @default(0) @db.Decimal(20, 5)
  net_paid_amount           Decimal @default(0) @db.Decimal(20, 5)
  base_total_applied_amount Decimal @default(0) @db.Decimal(20, 5)
  base_total_wht_amount     Decimal @default(0) @db.Decimal(20, 5)
  base_total_expense_amount Decimal @default(0) @db.Decimal(20, 5)
  base_net_paid_amount      Decimal @default(0) @db.Decimal(20, 5)
  realized_fx_amount        Decimal @default(0) @db.Decimal(20, 5)

  gl_jv_id      String?   @db.Uuid
  gl_jv_no      String?   @db.VarChar
  posted_at     DateTime? @db.Timestamptz(6)
  posted_by_id  String?   @db.Uuid
  void_at       DateTime? @db.Timestamptz(6)
  void_by_id    String?   @db.Uuid
  void_reason   String?   @db.VarChar
  void_gl_jv_id String?   @db.Uuid

  workflow_id             String?           @db.Uuid
  workflow_name           String?           @db.VarChar
  workflow_history        Json?             @default("[]") @db.JsonB
  workflow_current_stage  String?           @db.VarChar
  workflow_previous_stage String?           @db.VarChar
  workflow_next_stage     String?           @db.VarChar
  user_action             Json?             @db.JsonB
  last_action             enum_last_action?
  last_action_at_date     DateTime?         @db.Timestamptz(6)
  last_action_by_id       String?           @db.Uuid
  last_action_by_name     String?           @db.VarChar

  attachments Json? @default("[]") @db.JsonB
  info        Json? @default("{}") @db.JsonB
  dimension   Json? @default("[]") @db.JsonB

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid
  updated_at    DateTime? @default(now()) @db.Timestamptz(6)
  updated_by_id String?   @db.Uuid
  deleted_at    DateTime? @db.Timestamptz(6)
  deleted_by_id String?   @db.Uuid

  tb_bank_account       tb_bank_account         @relation(fields: [bank_account_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_ap_payment_detail  tb_ap_payment_detail[]
  tb_ap_payment_wht     tb_ap_payment_wht[]
  tb_ap_payment_expense tb_ap_payment_expense[]

  @@unique([doc_no, deleted_at], map: "appayment_doc_no_u")
  @@index([vendor_id, doc_status], map: "appayment_vendor_status_idx")
  @@index([payment_date], map: "appayment_date_idx")
}

model tb_ap_payment_detail {
  id                           String                   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ap_payment_id                String                   @db.Uuid
  ap_invoice_id                String                   @db.Uuid
  ap_invoice_detail_id         String                   @db.Uuid
  doc_type                     enum_ap_invoice_doc_type
  applied_amount               Decimal                  @default(0) @db.Decimal(20, 5)
  invoice_exchange_rate        Decimal                  @default(1) @db.Decimal(15, 5)
  base_applied_at_invoice_rate Decimal                  @default(0) @db.Decimal(20, 5)
  base_applied_at_payment_rate Decimal                  @default(0) @db.Decimal(20, 5)
  realized_fx_amount           Decimal                  @default(0) @db.Decimal(20, 5)

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid

  tb_ap_payment        tb_ap_payment        @relation(fields: [ap_payment_id], references: [id], onDelete: Cascade, onUpdate: NoAction)
  tb_ap_invoice        tb_ap_invoice        @relation(fields: [ap_invoice_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_ap_invoice_detail tb_ap_invoice_detail @relation(fields: [ap_invoice_detail_id], references: [id], onDelete: NoAction, onUpdate: NoAction)

  @@unique([ap_payment_id, ap_invoice_detail_id], map: "appaymentdetail_u")
  @@index([ap_invoice_detail_id], map: "appaymentdetail_invoice_detail_idx")
}

model tb_ap_payment_wht {
  id                   String                         @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ap_payment_id        String                         @db.Uuid
  wht_tax_profile_id   String                         @db.Uuid
  wht_tax_profile_name String                         @db.VarChar
  pnd_form             enum_tax_profile_wht_pnd_form?
  income_type          String?                        @db.VarChar
  base_amount          Decimal                        @default(0) @db.Decimal(20, 5)
  wht_rate             Decimal                        @default(0) @db.Decimal(15, 5)
  wht_amount           Decimal                        @default(0) @db.Decimal(20, 5)
  is_override          Boolean                        @default(false)
  chart_of_accounts_id String                         @db.Uuid

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid

  tb_ap_payment tb_ap_payment @relation(fields: [ap_payment_id], references: [id], onDelete: Cascade, onUpdate: NoAction)

  @@index([ap_payment_id], map: "appaymentwht_payment_idx")
}

model tb_ap_payment_expense {
  id                   String  @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ap_payment_id        String  @db.Uuid
  chart_of_accounts_id String  @db.Uuid
  cost_center_id       String? @db.Uuid
  description          String? @db.VarChar
  amount               Decimal @default(0) @db.Decimal(20, 5)
  base_amount          Decimal @default(0) @db.Decimal(20, 5)

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid

  tb_ap_payment tb_ap_payment @relation(fields: [ap_payment_id], references: [id], onDelete: Cascade, onUpdate: NoAction)

  @@index([ap_payment_id], map: "appaymentexpense_payment_idx")
}
```

- [ ] **Step 3: สร้าง migration แบบ create-only แล้วเติม SQL ที่ Prisma ทำไม่ได้**

```bash
cd packages/prisma-shared-schema-tenant
bunx prisma format
bunx prisma migrate dev --create-only --name accounting_foundation_ap
```
เปิดไฟล์ `prisma/migrations/<timestamp>_accounting_foundation_ap/migration.sql` แล้วต่อท้ายด้วย:

```sql
-- spec §4.2 — post ซ้ำจาก source เดียวกันไม่ได้ ตราบใดที่ใบเดิมยังไม่ void
CREATE UNIQUE INDEX IF NOT EXISTS "gljvheader_source_ref_u"
  ON "tb_gl_jv_header" ("source", "source_ref_type", "source_ref_id")
  WHERE "deleted_at" IS NULL AND "jv_status" <> 'void' AND "source_ref_id" IS NOT NULL;

-- spec §6.3 — เลข invoice ของ vendor ซ้ำไม่ได้ในใบที่ยังไม่ void
CREATE UNIQUE INDEX IF NOT EXISTS "apinvoice_vendor_invoice_no_u"
  ON "tb_ap_invoice" ("vendor_id", "vendor_invoice_no")
  WHERE "deleted_at" IS NULL AND "doc_status" <> 'void';

-- spec §12 — seed accounting dimension 7 ตัวตาม PRD
INSERT INTO "tb_gl_dimension" ("code", "name", "sequence") VALUES
  ('market', 'Market Segment', 1),
  ('sales', 'Sales Channel', 2),
  ('project', 'Project', 3),
  ('event', 'Event', 4),
  ('location', 'Location', 5),
  ('channel', 'Booking Channel', 6),
  ('guest_type', 'Guest Type', 7)
ON CONFLICT DO NOTHING;
```

```bash
bunx prisma migrate dev
bun run db:generate
cd ../..
```
Expected: migration apply ผ่านบน dev schema, `prisma generate` สร้าง client ที่มี model ใหม่

- [ ] **Step 4: เพิ่ม AP tables ใน `excludeModels`** (`packages/prisma-shared-schema-tenant/src/client.ts` ~L102) เพราะ Task 11/15 จะลงทะเบียน activity ระดับ action ให้แทน (ธรรมเนียมเดียวกับ PR/PO — ถ้าไม่ exclude จะได้ audit ซ้ำ 2 ชั้น) เพิ่มหลังกลุ่ม `// procurement`:

```ts
    // accounts payable — action-level activity registered in activity-registry.ts
    'tb_ap_invoice',
    'tb_ap_invoice_detail',
    'tb_ap_invoice_detail_dimension',
    'tb_ap_invoice_detail_source',
    'tb_ap_invoice_reference',
    'tb_ap_invoice_tax',
    'tb_ap_payment',
    'tb_ap_payment_detail',
    'tb_ap_payment_wht',
    'tb_ap_payment_expense',
```
(ตาราง foundation `tb_gl_dimension*`, `tb_bank_account`, `tb_gl_jv_detail_dimension` **ไม่** exclude ให้ extension เก็บ row-level เหมือน `tb_gl_*`)

- [ ] **Step 5: extend `GlSettingSchema`** (`apps/micro-business/src/app-config/app-config.service.ts:87`)

```ts
const GlSettingSchema = z.object({
  fiscal_year_start_month: z.number().int().min(1).max(12),
  retained_earnings_account_id: z.string().uuid().optional(),
  reversal_prefix_id: z.string().uuid().optional(),
  auto_jv_prefix_id: z.string().uuid().optional(),
  allow_post_to_closed_period: z.boolean().optional(),
  // spec §4.2 — AP posting accounts
  ap_control_account_id: z.string().uuid().optional(),
  input_vat_account_id: z.string().uuid().optional(),
  wht_payable_account_id: z.string().uuid().optional(),
  advance_deposit_account_id: z.string().uuid().optional(),
  realized_fx_gain_account_id: z.string().uuid().optional(),
  realized_fx_loss_account_id: z.string().uuid().optional(),
  ap_jv_prefix_id: z.string().uuid().optional(),
});
```

- [ ] **Step 6: เพิ่ม running-code preset** (`apps/micro-business/src/master/running-code/const/running-code.const.ts` ก่อน `};` ปิด object)

```ts
  'AP-IV': { config: { A: 'APIV', B: `date('yyMM')`, C: `running(4, '0')`, format: '{A}{B}{C}' } },
  'AP-DN': { config: { A: 'APDN', B: `date('yyMM')`, C: `running(4, '0')`, format: '{A}{B}{C}' } },
  'AP-CN': { config: { A: 'APCN', B: `date('yyMM')`, C: `running(4, '0')`, format: '{A}{B}{C}' } },
  'AP-DP': { config: { A: 'APDP', B: `date('yyMM')`, C: `running(4, '0')`, format: '{A}{B}{C}' } },
  'AP-PV': { config: { A: 'APPV', B: `date('yyMM')`, C: `running(4, '0')`, format: '{A}{B}{C}' } },
```
(`CommonLogic.getRunningPattern` สร้างแถว `tb_config_running_code` จาก preset อัตโนมัติเมื่อยังไม่มี จึงไม่ต้อง seed SQL)

- [ ] **Step 7: build + type-check + commit**

```bash
bun run build:package && bun run check-types
npx eslint --no-fix apps/micro-business/src/app-config/app-config.service.ts apps/micro-business/src/master/running-code/const/running-code.const.ts packages/prisma-shared-schema-tenant/src/client.ts
git add packages/prisma-shared-schema-tenant apps/micro-business/src/app-config/app-config.service.ts apps/micro-business/src/master/running-code/const/running-code.const.ts
git commit -m "feat(schema): add accounting dimension, bank account and AP tables

Spec: carmen-accounting-concept docs/superpowers/specs/2026-09-23-accounting-foundation-ap-design.md §4"
```

---

### Task 3: Error catalog

**Files:**
- Modify: `packages/error-catalog/src/module.ts:59-64`
- Modify: `packages/error-catalog/src/catalog.ts` (ต่อท้ายกลุ่ม GL)

**Interfaces:**
- Produces: `ERROR_CATALOG.<code>` ทุกตัวด้านล่าง ใช้โดย Task 4–15

- [ ] **Step 1: เพิ่ม MODULE ids** หลัง `GL_JV_TEMPLATE: 615,`

```ts
  GL_DIMENSION: 616,
  GL_SUBLEDGER: 617,
  BANK_ACCOUNT: 618,
  AP_INVOICE: 620,
  AP_PAYMENT: 621,
```

- [ ] **Step 2: เพิ่ม entries ใน catalog.ts** ต่อจาก entry สุดท้ายของกลุ่ม GL (รูปแบบเดียวกับ `GL_JV_NOT_FOUND` แต่ละ entry มี `code, id, http_status, message_en, message_th`)

```ts
  // ---- GL dimension (spec §5.1) ----
  GL_DIMENSION_NOT_FOUND: { code: 'GL_DIMENSION_NOT_FOUND', id: makeId(MODULE.GL_DIMENSION, 1), http_status: 404, message_en: 'Dimension not found', message_th: 'ไม่พบ dimension' },
  GL_DIMENSION_DUPLICATE_CODE: { code: 'GL_DIMENSION_DUPLICATE_CODE', id: makeId(MODULE.GL_DIMENSION, 2), http_status: 409, message_en: 'Dimension code {code} already exists', message_th: 'รหัส dimension {code} ซ้ำ' },
  GL_DIMENSION_IN_USE: { code: 'GL_DIMENSION_IN_USE', id: makeId(MODULE.GL_DIMENSION, 3), http_status: 409, message_en: 'Dimension {code} is used by documents', message_th: 'dimension {code} ถูกใช้ในเอกสารแล้ว' },
  GL_DIMENSION_REQUIRED: { code: 'GL_DIMENSION_REQUIRED', id: makeId(MODULE.GL_DIMENSION, 4), http_status: 422, message_en: 'Account {account_code} requires dimension {dimension_code}', message_th: 'บัญชี {account_code} ต้องระบุ dimension {dimension_code}' },
  GL_DIMENSION_PROHIBITED: { code: 'GL_DIMENSION_PROHIBITED', id: makeId(MODULE.GL_DIMENSION, 5), http_status: 422, message_en: 'Account {account_code} does not allow dimension {dimension_code}', message_th: 'บัญชี {account_code} ห้ามระบุ dimension {dimension_code}' },
  GL_DIMENSION_VALUE_INVALID: { code: 'GL_DIMENSION_VALUE_INVALID', id: makeId(MODULE.GL_DIMENSION, 6), http_status: 422, message_en: 'Dimension value {value_id} is inactive, expired or belongs to another dimension', message_th: 'ค่า dimension {value_id} ไม่ active หมดอายุ หรือไม่ใช่ของ dimension นี้' },
  // ---- subledger posting (spec §5.2) ----
  GL_SUBLEDGER_SOURCE_NOT_FOUND: { code: 'GL_SUBLEDGER_SOURCE_NOT_FOUND', id: makeId(MODULE.GL_SUBLEDGER, 1), http_status: 404, message_en: 'No posted voucher for {source_ref_type} {source_ref_id}', message_th: 'ไม่พบใบสำคัญที่ post จาก {source_ref_type} {source_ref_id}' },
  GL_SETTING_ACCOUNT_MISSING: { code: 'GL_SETTING_ACCOUNT_MISSING', id: makeId(MODULE.GL_SUBLEDGER, 2), http_status: 422, message_en: 'gl_setting.{key} is not configured', message_th: 'ยังไม่ได้ตั้งค่า gl_setting.{key}' },
  // ---- bank account (spec §5.5) ----
  BANK_ACCOUNT_NOT_FOUND: { code: 'BANK_ACCOUNT_NOT_FOUND', id: makeId(MODULE.BANK_ACCOUNT, 1), http_status: 404, message_en: 'Bank account not found', message_th: 'ไม่พบบัญชีธนาคาร' },
  BANK_ACCOUNT_DUPLICATE_CODE: { code: 'BANK_ACCOUNT_DUPLICATE_CODE', id: makeId(MODULE.BANK_ACCOUNT, 2), http_status: 409, message_en: 'Bank account code {code} already exists', message_th: 'รหัสบัญชีธนาคาร {code} ซ้ำ' },
  BANK_ACCOUNT_IN_USE: { code: 'BANK_ACCOUNT_IN_USE', id: makeId(MODULE.BANK_ACCOUNT, 3), http_status: 409, message_en: 'Bank account {code} is referenced by payments', message_th: 'บัญชีธนาคาร {code} ถูกใช้ในใบจ่ายเงินแล้ว' },
  // ---- AP invoice (spec §6, §10) ----
  AP_INVOICE_NOT_FOUND: { code: 'AP_INVOICE_NOT_FOUND', id: makeId(MODULE.AP_INVOICE, 1), http_status: 404, message_en: 'AP invoice not found', message_th: 'ไม่พบเอกสาร AP' },
  AP_INVOICE_IMMUTABLE: { code: 'AP_INVOICE_IMMUTABLE', id: makeId(MODULE.AP_INVOICE, 2), http_status: 409, message_en: 'AP invoice cannot be changed in its current status', message_th: 'เอกสาร AP แก้ไขไม่ได้ในสถานะปัจจุบัน' },
  AP_VENDOR_INVOICE_NO_DUPLICATE: { code: 'AP_VENDOR_INVOICE_NO_DUPLICATE', id: makeId(MODULE.AP_INVOICE, 3), http_status: 409, message_en: 'Vendor invoice number {vendor_invoice_no} already exists for this vendor', message_th: 'เลขที่ใบแจ้งหนี้ {vendor_invoice_no} ของ vendor นี้ซ้ำ' },
  AP_VENDOR_CURRENCY_MISMATCH: { code: 'AP_VENDOR_CURRENCY_MISMATCH', id: makeId(MODULE.AP_INVOICE, 4), http_status: 422, message_en: 'Referenced document must share vendor and currency', message_th: 'เอกสารที่อ้างต้องเป็น vendor และสกุลเงินเดียวกัน' },
  AP_GRN_NOT_COMMITTED: { code: 'AP_GRN_NOT_COMMITTED', id: makeId(MODULE.AP_INVOICE, 5), http_status: 422, message_en: 'GRN {grn_no} is not committed', message_th: 'GRN {grn_no} ยังไม่ commit' },
  AP_GRN_OVER_MATCHED: { code: 'AP_GRN_OVER_MATCHED', id: makeId(MODULE.AP_INVOICE, 6), http_status: 422, message_en: 'Matched quantity exceeds remaining quantity on GRN {grn_no}', message_th: 'จำนวนที่ match เกินยอดคงเหลือของ GRN {grn_no}' },
  AP_REFERENCE_NOT_POSTED: { code: 'AP_REFERENCE_NOT_POSTED', id: makeId(MODULE.AP_INVOICE, 7), http_status: 422, message_en: 'Referenced document {doc_no} is not posted', message_th: 'เอกสารอ้างอิง {doc_no} ยังไม่ post' },
  AP_REFERENCE_OVER_APPLIED: { code: 'AP_REFERENCE_OVER_APPLIED', id: makeId(MODULE.AP_INVOICE, 8), http_status: 422, message_en: 'Applied amount exceeds remaining balance of {doc_no}', message_th: 'ยอดที่นำมาหักเกินยอดคงเหลือของ {doc_no}' },
  AP_INVOICE_HAS_PAYMENT: { code: 'AP_INVOICE_HAS_PAYMENT', id: makeId(MODULE.AP_INVOICE, 9), http_status: 409, message_en: 'AP invoice has payments and cannot be voided', message_th: 'เอกสาร AP มีการจ่ายเงินแล้ว void ไม่ได้' },
  AP_INVOICE_IS_REFERENCED: { code: 'AP_INVOICE_IS_REFERENCED', id: makeId(MODULE.AP_INVOICE, 10), http_status: 409, message_en: 'AP invoice is referenced by another document and cannot be voided', message_th: 'เอกสาร AP ถูกใบอื่นอ้างอิง void ไม่ได้' },
  AP_ACCOUNT_NOT_POSTABLE: { code: 'AP_ACCOUNT_NOT_POSTABLE', id: makeId(MODULE.AP_INVOICE, 11), http_status: 422, message_en: 'Account {account_id} is inactive or not postable', message_th: 'บัญชี {account_id} ไม่ active หรือลงบัญชีไม่ได้' },
  AP_COST_CENTER_REQUIRED: { code: 'AP_COST_CENTER_REQUIRED', id: makeId(MODULE.AP_INVOICE, 12), http_status: 422, message_en: 'Account {account_code} requires a cost center', message_th: 'บัญชี {account_code} ต้องระบุ cost center' },
  AP_INVOICE_LINE_INVALID: { code: 'AP_INVOICE_LINE_INVALID', id: makeId(MODULE.AP_INVOICE, 13), http_status: 422, message_en: 'Line {sequence_no}: {reason}', message_th: 'บรรทัด {sequence_no}: {reason}' },
  AP_VENDOR_NOT_FOUND: { code: 'AP_VENDOR_NOT_FOUND', id: makeId(MODULE.AP_INVOICE, 14), http_status: 422, message_en: 'Vendor not found or inactive', message_th: 'ไม่พบ vendor หรือ vendor ไม่ active' },
  AP_TAX_PROFILE_INVALID: { code: 'AP_TAX_PROFILE_INVALID', id: makeId(MODULE.AP_INVOICE, 15), http_status: 422, message_en: 'Tax profile {tax_profile_id} is missing or of the wrong type', message_th: 'tax profile {tax_profile_id} ไม่พบหรือประเภทไม่ตรง' },
  // ---- AP payment (spec §7, §10) ----
  AP_PAYMENT_NOT_FOUND: { code: 'AP_PAYMENT_NOT_FOUND', id: makeId(MODULE.AP_PAYMENT, 1), http_status: 404, message_en: 'Payment voucher not found', message_th: 'ไม่พบใบจ่ายเงิน' },
  AP_PAYMENT_IMMUTABLE: { code: 'AP_PAYMENT_IMMUTABLE', id: makeId(MODULE.AP_PAYMENT, 2), http_status: 409, message_en: 'Payment voucher cannot be changed in its current status', message_th: 'ใบจ่ายเงินแก้ไขไม่ได้ในสถานะปัจจุบัน' },
  AP_PAYMENT_OVER_ALLOCATED: { code: 'AP_PAYMENT_OVER_ALLOCATED', id: makeId(MODULE.AP_PAYMENT, 3), http_status: 422, message_en: 'Applied amount exceeds unpaid amount on {doc_no} line {sequence_no}', message_th: 'ยอดจ่ายเกินยอดค้างของ {doc_no} บรรทัด {sequence_no}' },
  AP_PAYMENT_EMPTY: { code: 'AP_PAYMENT_EMPTY', id: makeId(MODULE.AP_PAYMENT, 4), http_status: 422, message_en: 'Payment must apply a positive total amount', message_th: 'ยอดจ่ายรวมต้องมากกว่าศูนย์' },
  AP_PAYMENT_BANK_CURRENCY_MISMATCH: { code: 'AP_PAYMENT_BANK_CURRENCY_MISMATCH', id: makeId(MODULE.AP_PAYMENT, 5), http_status: 422, message_en: 'Bank account currency must match the payment or base currency', message_th: 'สกุลเงินบัญชีธนาคารต้องตรงกับใบจ่ายหรือสกุลฐาน' },
  AP_PAYMENT_INVOICE_NOT_POSTED: { code: 'AP_PAYMENT_INVOICE_NOT_POSTED', id: makeId(MODULE.AP_PAYMENT, 6), http_status: 422, message_en: 'Invoice {doc_no} is not posted', message_th: 'เอกสาร {doc_no} ยังไม่ post' },
```

- [ ] **Step 3: build + commit**

```bash
bun run build:package && npx eslint --no-fix packages/error-catalog/src/catalog.ts packages/error-catalog/src/module.ts
(cd packages/error-catalog && bun run test)
git add packages/error-catalog && git commit -m "feat(error-catalog): add GL dimension, subledger, bank account and AP codes"
```
(`catalog.spec.ts` เป็น test เดิมของ package ต้องยังผ่าน; ถ้า fail เรื่อง id ซ้ำ ให้แก้เลขใน Step 1 ตามที่ Task 0 Step 2 พบ)

---

### Task 4: GL dimension module (นิยาม / ค่า / กฎต่อบัญชี) + validator + gateway config

**Files:**
- Create: `apps/micro-business/src/gl/gl-dimension/interface/gl-dimension.interface.ts`
- Create: `apps/micro-business/src/gl/gl-dimension/dto/gl-dimension.serializer.ts`
- Create: `apps/micro-business/src/gl/gl-dimension/gl-dimension.validator.ts`
- Create: `apps/micro-business/src/gl/gl-dimension/gl-dimension.service.ts`
- Create: `apps/micro-business/src/gl/gl-dimension/gl-dimension-value.service.ts`
- Create: `apps/micro-business/src/gl/gl-dimension/gl-account-dimension-rule.service.ts`
- Create: `apps/micro-business/src/gl/gl-dimension/gl-dimension.controller.ts`
- Create: `apps/micro-business/src/gl/gl-dimension/gl-dimension-value.controller.ts`
- Create: `apps/micro-business/src/gl/gl-dimension/gl-account-dimension-rule.controller.ts`
- Create: `apps/micro-business/src/gl/gl-dimension/gl-dimension.module.ts`
- Modify: `apps/micro-business/src/app.module.ts` (import + imports array หลัง `GlJvPrefixModule`)
- Modify: `apps/micro-business/src/common/dto/index.ts` (export serializer)
- Generated: `packages/rpc-contract/src/contracts/gl-dimension.ts`, `gl-dimension-value.ts`, `gl-account-dimension-rule.ts`
- Create: `apps/backend-gateway/src/config/config_gl-dimensions/{config_gl-dimensions.controller.ts,config_gl-dimensions.service.ts,config_gl-dimensions.module.ts,swagger/request.ts,swagger/response.ts}`
- Create: `apps/backend-gateway/src/config/config_gl-dimension-values/...` (ชุดเดียวกัน)
- Create: `apps/backend-gateway/src/config/config_gl-account-dimension-rules/...` (ชุดเดียวกัน)
- Modify: `apps/backend-gateway/src/config/route-config.ts`

**Interfaces:**
- Consumes: Prisma models จาก Task 1, `ERROR_CATALOG.GL_DIMENSION_*` จาก Task 3
- Produces:
  - `validateLineDimensions(db, lines: IDimensionLine[], onDate: Date): Promise<Result<Map<string, IResolvedDimensionValue>>>` (map key = `gl_dimension_value_id`)
  - `IDimensionRef { gl_dimension_id: string; gl_dimension_value_id: string }`
  - RPC contracts `GlDimension`, `GlDimensionValue`, `GlAccountDimensionRule` แต่ละตัวมี `findAll, findOne, create, update, delete`

- [ ] **Step 1: interface**

`apps/micro-business/src/gl/gl-dimension/interface/gl-dimension.interface.ts`
```ts
import { enum_gl_account_dimension_rule_requirement } from '@repo/prisma-shared-schema-tenant';

/**
 * One dimension value attached to a document line
 * ค่า dimension หนึ่งตัวที่ผูกกับบรรทัดเอกสาร
 */
export interface IDimensionRef {
  gl_dimension_id: string;
  gl_dimension_value_id: string;
}

/**
 * The slice of a line the dimension validator needs
 * ส่วนของบรรทัดที่ตัวตรวจ dimension ต้องใช้
 */
export interface IDimensionLine {
  sequence_no: number;
  chart_of_accounts_id: string;
  dimensions?: IDimensionRef[] | null;
}

/**
 * Snapshot codes the validator resolves for each accepted value
 * รหัส snapshot ที่ตัวตรวจ resolve ให้แต่ละค่าที่ผ่าน
 */
export interface IResolvedDimensionValue {
  gl_dimension_id: string;
  dimension_code: string;
  value_code: string;
}

/** Create payload for a dimension / ข้อมูลสร้าง dimension */
export interface ICreateGlDimension {
  code: string;
  name: string;
  name_local?: string | null;
  description?: string | null;
  sequence?: number;
  is_active?: boolean;
}

/** Update payload for a dimension / ข้อมูลแก้ไข dimension */
export interface IUpdateGlDimension {
  id: string;
  doc_version: number;
  name?: string;
  name_local?: string | null;
  description?: string | null;
  sequence?: number;
  is_active?: boolean;
}

/** Create payload for a dimension value / ข้อมูลสร้างค่า dimension */
export interface ICreateGlDimensionValue {
  gl_dimension_id: string;
  code: string;
  name: string;
  name_local?: string | null;
  effective_from?: Date | null;
  effective_to?: Date | null;
  is_active?: boolean;
}

/** Update payload for a dimension value / ข้อมูลแก้ไขค่า dimension */
export interface IUpdateGlDimensionValue {
  id: string;
  doc_version: number;
  name?: string;
  name_local?: string | null;
  effective_from?: Date | null;
  effective_to?: Date | null;
  is_active?: boolean;
}

/** Create payload for an account dimension rule / ข้อมูลสร้างกฎ dimension ต่อบัญชี */
export interface ICreateGlAccountDimensionRule {
  chart_of_accounts_id: string;
  gl_dimension_id: string;
  requirement: enum_gl_account_dimension_rule_requirement;
}

/** Update payload for an account dimension rule / ข้อมูลแก้ไขกฎ dimension ต่อบัญชี */
export interface IUpdateGlAccountDimensionRule {
  id: string;
  doc_version: number;
  requirement: enum_gl_account_dimension_rule_requirement;
}
```

- [ ] **Step 2: serializer**

`apps/micro-business/src/gl/gl-dimension/dto/gl-dimension.serializer.ts`
```ts
import { z } from 'zod/v4';
import { enum_gl_account_dimension_rule_requirement } from '@repo/prisma-shared-schema-tenant';

const auditFields = {
  doc_version: z.number().nullable().optional(),
  created_at: z.coerce.date().nullable().optional(),
  created_by_id: z.string().nullable().optional(),
  updated_at: z.coerce.date().nullable().optional(),
  updated_by_id: z.string().nullable().optional(),
};

/**
 * Response shape for one dimension definition
 * รูปแบบการตอบกลับของนิยาม dimension หนึ่งตัว
 */
export const GlDimensionResponseSchema = z.object({
  id: z.string(),
  code: z.string(),
  name: z.string(),
  name_local: z.string().nullable().optional(),
  description: z.string().nullable().optional(),
  sequence: z.number(),
  is_active: z.boolean(),
  ...auditFields,
});

/**
 * Response shape for one dimension value
 * รูปแบบการตอบกลับของค่า dimension หนึ่งค่า
 */
export const GlDimensionValueResponseSchema = z.object({
  id: z.string(),
  gl_dimension_id: z.string(),
  code: z.string(),
  name: z.string(),
  name_local: z.string().nullable().optional(),
  effective_from: z.coerce.date().nullable().optional(),
  effective_to: z.coerce.date().nullable().optional(),
  is_active: z.boolean(),
  ...auditFields,
});

/**
 * Response shape for one account dimension rule
 * รูปแบบการตอบกลับของกฎ dimension ต่อบัญชีหนึ่งข้อ
 */
export const GlAccountDimensionRuleResponseSchema = z.object({
  id: z.string(),
  chart_of_accounts_id: z.string(),
  gl_dimension_id: z.string(),
  requirement: z.nativeEnum(enum_gl_account_dimension_rule_requirement),
  ...auditFields,
});
```
เพิ่มใน `apps/micro-business/src/common/dto/index.ts` หลังบรรทัด `export * from '@/gl/gl-jv/dto/gl-jv.serializer';`:
```ts
export * from '@/gl/gl-dimension/dto/gl-dimension.serializer';
```

- [ ] **Step 3: validator** (spec §5.1 ข้อ 1–4)

`apps/micro-business/src/gl/gl-dimension/gl-dimension.validator.ts`
```ts
import {
  enum_gl_account_dimension_rule_requirement,
  type PrismaClient,
} from '@repo/prisma-shared-schema-tenant';
import { Result } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import { IDimensionLine, IResolvedDimensionValue } from './interface/gl-dimension.interface';

/** The read surface the validator needs — client or transaction client / พื้นผิวการอ่านที่ต้องใช้ */
type DimensionValidationDb = Pick<
  PrismaClient,
  'tb_gl_account_dimension_rule' | 'tb_gl_dimension_value' | 'tb_chart_of_accounts'
>;

type RuleRow = { chart_of_accounts_id: string; gl_dimension_id: string; requirement: enum_gl_account_dimension_rule_requirement };

/**
 * Validate every line's dimensions against the rules of its account and the value master
 * ตรวจ dimension ของทุกบรรทัดกับกฎของบัญชีนั้นและ master ของค่า
 *
 * Shared by manual JV, the subledger facade and AP so the three never drift (spec §5.1).
 * ใช้ร่วมกันโดย JV มือ, facade และ AP เพื่อไม่ให้กฎแยกกันเพี้ยน (spec §5.1)
 * @param db - Prisma client or transaction client / client ของ Prisma หรือของ transaction
 * @param lines - Lines carrying account + dimensions / บรรทัดที่มีบัญชีและ dimension
 * @param onDate - Document date used for effective-range checks / วันที่เอกสารสำหรับตรวจช่วงมีผล
 * @returns Snapshot codes keyed by value id, or the first violation / รหัส snapshot คีย์ด้วย value id หรือข้อผิดพลาดแรก
 */
export async function validateLineDimensions(
  db: DimensionValidationDb,
  lines: IDimensionLine[],
  onDate: Date,
): Promise<Result<Map<string, IResolvedDimensionValue>>> {
  const accountIds = [...new Set(lines.map((l) => l.chart_of_accounts_id))];
  const rules = await db.tb_gl_account_dimension_rule.findMany({
    where: { chart_of_accounts_id: { in: accountIds }, deleted_at: null },
    select: { chart_of_accounts_id: true, gl_dimension_id: true, requirement: true },
  });
  const accounts = await db.tb_chart_of_accounts.findMany({
    where: { id: { in: accountIds } },
    select: { id: true, code: true },
  });
  const codeById = new Map(accounts.map((a) => [a.id, a.code]));

  const valueIds = [...new Set(lines.flatMap((l) => (l.dimensions ?? []).map((d) => d.gl_dimension_value_id)))];
  const values = await db.tb_gl_dimension_value.findMany({
    where: { id: { in: valueIds }, deleted_at: null },
    include: { tb_gl_dimension: { select: { code: true, is_active: true } } },
  });
  const valueById = new Map(values.map((v) => [v.id, v]));

  for (const line of lines) {
    const ruleCheck = checkRules(line, rules.filter((r) => r.chart_of_accounts_id === line.chart_of_accounts_id), codeById);
    if (ruleCheck.isError()) return Result.error(ruleCheck.error);
    for (const ref of line.dimensions ?? []) {
      const value = valueById.get(ref.gl_dimension_value_id);
      if (!isUsableValue(value, ref.gl_dimension_id, onDate)) {
        return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_VALUE_INVALID, { value_id: ref.gl_dimension_value_id });
      }
    }
  }

  const resolved = new Map<string, IResolvedDimensionValue>();
  for (const v of values) {
    resolved.set(v.id, { gl_dimension_id: v.gl_dimension_id, dimension_code: v.tb_gl_dimension.code, value_code: v.code });
  }
  return Result.ok(resolved);
}

/**
 * Apply mandatory / prohibited rules of one account to one line
 * ใช้กฎ mandatory / prohibited ของบัญชีหนึ่งกับบรรทัดหนึ่ง
 * @param line - The line / บรรทัด
 * @param rules - Rules of the line's account / กฎของบัญชีในบรรทัด
 * @param codeById - Account code lookup for messages / ตารางหารหัสบัญชีสำหรับข้อความ
 * @returns ok, or the first violated rule / ok หรือกฎแรกที่ละเมิด
 */
function checkRules(line: IDimensionLine, rules: RuleRow[], codeById: Map<string, string>): Result<true> {
  const present = new Set((line.dimensions ?? []).map((d) => d.gl_dimension_id));
  for (const rule of rules) {
    const params = { account_code: codeById.get(line.chart_of_accounts_id) ?? line.chart_of_accounts_id, dimension_code: rule.gl_dimension_id, sequence_no: line.sequence_no };
    if (rule.requirement === enum_gl_account_dimension_rule_requirement.mandatory && !present.has(rule.gl_dimension_id)) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_REQUIRED, params);
    }
    if (rule.requirement === enum_gl_account_dimension_rule_requirement.prohibited && present.has(rule.gl_dimension_id)) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_PROHIBITED, params);
    }
  }
  return Result.ok(true);
}

/**
 * A value is usable when active, of the named dimension, and effective on the date
 * ค่าใช้ได้เมื่อ active เป็นของ dimension ที่ระบุ และมีผล ณ วันนั้น
 * @param value - Value row with its dimension / แถวค่าพร้อม dimension
 * @param dimensionId - Dimension the line claims / dimension ที่บรรทัดอ้าง
 * @param onDate - Document date / วันที่เอกสาร
 * @returns `true` when usable / true เมื่อใช้ได้
 */
function isUsableValue(
  value: { gl_dimension_id: string; is_active: boolean; effective_from: Date | null; effective_to: Date | null; tb_gl_dimension: { is_active: boolean } } | undefined,
  dimensionId: string,
  onDate: Date,
): boolean {
  if (!value || !value.is_active || !value.tb_gl_dimension.is_active) return false;
  if (value.gl_dimension_id !== dimensionId) return false;
  if (value.effective_from && onDate < value.effective_from) return false;
  if (value.effective_to && onDate > value.effective_to) return false;
  return true;
}
```

- [ ] **Step 4: services** (pattern เดียวกับ `gl-jv-prefix.service.ts`)

`apps/micro-business/src/gl/gl-dimension/gl-dimension.service.ts`
```ts
import { Injectable } from '@nestjs/common';
import { TryCatch, Result, GlDimensionResponseSchema } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import { TenantScopedService } from '@/common/tenant-scoped.service';
import { IPaginate } from '@/common/shared-interface/paginate.interface';
import QueryParams from '@/common/libs/paginate.query';
import { withDefaultSort } from '@/common/libs/default-sort';
import getPaginationParams from '@/common/helpers/pagination.params';
import { BackendLogger } from '@/common/helpers/backend.logger';
import { isUniqueConstraintViolation } from '@/common/helpers/is-unique-constraint-violation';
import { ICreateGlDimension, IUpdateGlDimension } from './interface/gl-dimension.interface';

const CODE_PATTERN = /^[a-z0-9_]{2,20}$/;

/**
 * CRUD for accounting dimension definitions (spec §5.1)
 * CRUD ของนิยาม accounting dimension (spec §5.1)
 */
@Injectable()
export class GlDimensionService extends TenantScopedService {
  private readonly logger: BackendLogger = new BackendLogger(GlDimensionService.name);

  /**
   * Find one dimension
   * ค้นหา dimension หนึ่งตัว
   * @param id - Dimension id / รหัส dimension
   * @returns The dimension or NOT_FOUND / dimension หรือ NOT_FOUND
   */
  @TryCatch
  async findOne(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findOne', id }, GlDimensionService.name);
    const row = await this.prismaService.tb_gl_dimension.findFirst({ where: { id, deleted_at: null } });
    if (!row) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_NOT_FOUND);
    return Result.ok(GlDimensionResponseSchema.parse(row));
  }

  /**
   * Paginated list ordered by sequence
   * รายการแบบแบ่งหน้าเรียงตาม sequence
   * @param paginate - Pagination params / พารามิเตอร์แบ่งหน้า
   * @returns Paginated dimensions / dimension แบบแบ่งหน้า
   */
  @TryCatch
  async findAll(paginate: IPaginate): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findAll', paginate }, GlDimensionService.name);
    const q = new QueryParams(
      paginate.page, paginate.perpage, paginate.search, paginate.searchfields,
      ['code', 'name'],
      typeof paginate.filter === 'object' && !Array.isArray(paginate.filter) ? paginate.filter : {},
      withDefaultSort(paginate.sort, ['sequence:asc', 'code:asc']),
      paginate.advance,
    );
    const pagination = getPaginationParams(q.page, q.perpage);
    const rows = await this.prismaService.tb_gl_dimension.findMany({ where: q.where(), orderBy: q.orderBy(), ...pagination });
    const total = await this.prismaService.tb_gl_dimension.count({ where: q.where() });
    return Result.ok({
      paginate: { total, page: q.perpage < 0 ? 1 : q.page, perpage: q.perpage < 0 ? 1 : q.perpage, pages: total === 0 || q.perpage < 0 ? 1 : Math.ceil(total / q.perpage) },
      data: rows.map((r) => GlDimensionResponseSchema.parse(r)),
    });
  }

  /**
   * Create a dimension; code is lower-case snake and immutable
   * สร้าง dimension รหัสเป็น snake ตัวพิมพ์เล็กและแก้ไม่ได้
   * @param data - Creation payload / ข้อมูลสร้าง
   * @returns `{ id, doc_version }` / id ที่สร้าง
   */
  @TryCatch
  async create(data: ICreateGlDimension): Promise<Result<unknown>> {
    this.logger.debug({ function: 'create', data }, GlDimensionService.name);
    const code = data.code.trim().toLowerCase();
    if (!CODE_PATTERN.test(code)) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_DUPLICATE_CODE, { code, reason: 'code must match ^[a-z0-9_]{2,20}$' });
    }
    try {
      const created = await this.prismaService.tb_gl_dimension.create({
        data: {
          code, name: data.name.trim(), name_local: data.name_local ?? null, description: data.description ?? null,
          sequence: data.sequence ?? 0, is_active: data.is_active ?? true, created_by_id: this.userId,
        },
      });
      return Result.ok({ id: created.id, doc_version: created.doc_version });
    } catch (error) {
      if (isUniqueConstraintViolation(error)) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_DUPLICATE_CODE, { code });
      throw error;
    }
  }

  /**
   * Update a dimension (code immutable); deactivating is blocked while in use
   * แก้ไข dimension (code แก้ไม่ได้) ปิดใช้งานไม่ได้ขณะถูกใช้
   * @param data - Update payload / ข้อมูลแก้ไข
   * @returns `{ id, doc_version }` / id และเวอร์ชันใหม่
   */
  @TryCatch
  async update(data: IUpdateGlDimension): Promise<Result<unknown>> {
    this.logger.debug({ function: 'update', data }, GlDimensionService.name);
    const existing = await this.prismaService.tb_gl_dimension.findFirst({ where: { id: data.id, deleted_at: null } });
    if (!existing) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_NOT_FOUND);
    if (data.is_active === false && (await this.countUsage(data.id)) > 0) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_IN_USE, { code: existing.code });
    }
    const updated = await this.prismaService.tb_gl_dimension.update({
      where: { id: data.id, doc_version: data.doc_version },
      data: {
        name: data.name?.trim(), name_local: data.name_local, description: data.description,
        sequence: data.sequence, is_active: data.is_active,
        doc_version: { increment: 1 }, updated_by_id: this.userId, updated_at: new Date(),
      },
    });
    return Result.ok({ id: updated.id, doc_version: updated.doc_version });
  }

  /**
   * Soft-delete a dimension that no document uses
   * ลบ dimension ที่ไม่มีเอกสารใช้แบบ soft-delete
   * @param id - Dimension id / รหัส dimension
   * @returns `{ id }` / id ที่ลบ
   */
  @TryCatch
  async delete(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'delete', id }, GlDimensionService.name);
    const existing = await this.prismaService.tb_gl_dimension.findFirst({ where: { id, deleted_at: null } });
    if (!existing) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_NOT_FOUND);
    if ((await this.countUsage(id)) > 0) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_IN_USE, { code: existing.code });
    await this.prismaService.tb_gl_dimension.update({ where: { id }, data: { deleted_at: new Date(), deleted_by_id: this.userId } });
    return Result.ok({ id });
  }

  /**
   * Count JV / AP lines that carry a value of this dimension on a non-void document
   * นับบรรทัด JV / AP ที่ใช้ค่าของ dimension นี้ในเอกสารที่ไม่ void
   * @param dimensionId - Dimension id / รหัส dimension
   * @returns Usage count / จำนวนที่ใช้
   */
  private async countUsage(dimensionId: string): Promise<number> {
    const jv = await this.prismaService.tb_gl_jv_detail_dimension.count({
      where: { gl_dimension_id: dimensionId, tb_gl_jv_detail: { tb_gl_jv_header: { jv_status: { not: 'void' }, deleted_at: null } } },
    });
    const ap = await this.prismaService.tb_ap_invoice_detail_dimension.count({
      where: { gl_dimension_id: dimensionId, tb_ap_invoice_detail: { tb_ap_invoice: { doc_status: { not: 'void' }, deleted_at: null } } },
    });
    return jv + ap;
  }
}
```

`apps/micro-business/src/gl/gl-dimension/gl-dimension-value.service.ts`
```ts
import { Injectable } from '@nestjs/common';
import { TryCatch, Result, GlDimensionValueResponseSchema } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import { TenantScopedService } from '@/common/tenant-scoped.service';
import { IPaginate } from '@/common/shared-interface/paginate.interface';
import QueryParams from '@/common/libs/paginate.query';
import { withDefaultSort } from '@/common/libs/default-sort';
import getPaginationParams from '@/common/helpers/pagination.params';
import { BackendLogger } from '@/common/helpers/backend.logger';
import { isUniqueConstraintViolation } from '@/common/helpers/is-unique-constraint-violation';
import { ICreateGlDimensionValue, IUpdateGlDimensionValue } from './interface/gl-dimension.interface';

/**
 * CRUD for dimension values (spec §5.1)
 * CRUD ของค่า dimension (spec §5.1)
 */
@Injectable()
export class GlDimensionValueService extends TenantScopedService {
  private readonly logger: BackendLogger = new BackendLogger(GlDimensionValueService.name);

  /**
   * Find one value
   * ค้นหาค่าหนึ่งค่า
   * @param id - Value id / รหัสค่า
   * @returns The value or NOT_FOUND / ค่า หรือ NOT_FOUND
   */
  @TryCatch
  async findOne(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findOne', id }, GlDimensionValueService.name);
    const row = await this.prismaService.tb_gl_dimension_value.findFirst({ where: { id, deleted_at: null } });
    if (!row) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_NOT_FOUND);
    return Result.ok(GlDimensionValueResponseSchema.parse(row));
  }

  /**
   * Paginated list; filter `gl_dimension_id` narrows to one dimension
   * รายการแบบแบ่งหน้า ใช้ filter `gl_dimension_id` เพื่อจำกัดหนึ่ง dimension
   * @param paginate - Pagination params / พารามิเตอร์แบ่งหน้า
   * @returns Paginated values / ค่าแบบแบ่งหน้า
   */
  @TryCatch
  async findAll(paginate: IPaginate): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findAll', paginate }, GlDimensionValueService.name);
    const q = new QueryParams(
      paginate.page, paginate.perpage, paginate.search, paginate.searchfields,
      ['code', 'name'],
      typeof paginate.filter === 'object' && !Array.isArray(paginate.filter) ? paginate.filter : {},
      withDefaultSort(paginate.sort, ['code:asc']),
      paginate.advance,
    );
    const pagination = getPaginationParams(q.page, q.perpage);
    const rows = await this.prismaService.tb_gl_dimension_value.findMany({ where: q.where(), orderBy: q.orderBy(), ...pagination });
    const total = await this.prismaService.tb_gl_dimension_value.count({ where: q.where() });
    return Result.ok({
      paginate: { total, page: q.perpage < 0 ? 1 : q.page, perpage: q.perpage < 0 ? 1 : q.perpage, pages: total === 0 || q.perpage < 0 ? 1 : Math.ceil(total / q.perpage) },
      data: rows.map((r) => GlDimensionValueResponseSchema.parse(r)),
    });
  }

  /**
   * Create a value under an active dimension
   * สร้างค่าใต้ dimension ที่ active
   * @param data - Creation payload / ข้อมูลสร้าง
   * @returns `{ id, doc_version }` / id ที่สร้าง
   */
  @TryCatch
  async create(data: ICreateGlDimensionValue): Promise<Result<unknown>> {
    this.logger.debug({ function: 'create', data }, GlDimensionValueService.name);
    const dimension = await this.prismaService.tb_gl_dimension.findFirst({ where: { id: data.gl_dimension_id, deleted_at: null } });
    if (!dimension) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_NOT_FOUND);
    const code = data.code.trim().toUpperCase();
    try {
      const created = await this.prismaService.tb_gl_dimension_value.create({
        data: {
          gl_dimension_id: data.gl_dimension_id, code, name: data.name.trim(), name_local: data.name_local ?? null,
          effective_from: data.effective_from ?? null, effective_to: data.effective_to ?? null,
          is_active: data.is_active ?? true, created_by_id: this.userId,
        },
      });
      return Result.ok({ id: created.id, doc_version: created.doc_version });
    } catch (error) {
      if (isUniqueConstraintViolation(error)) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_DUPLICATE_CODE, { code });
      throw error;
    }
  }

  /**
   * Update a value (code and dimension immutable)
   * แก้ไขค่า (code และ dimension แก้ไม่ได้)
   * @param data - Update payload / ข้อมูลแก้ไข
   * @returns `{ id, doc_version }` / id และเวอร์ชันใหม่
   */
  @TryCatch
  async update(data: IUpdateGlDimensionValue): Promise<Result<unknown>> {
    this.logger.debug({ function: 'update', data }, GlDimensionValueService.name);
    const existing = await this.prismaService.tb_gl_dimension_value.findFirst({ where: { id: data.id, deleted_at: null } });
    if (!existing) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_NOT_FOUND);
    const updated = await this.prismaService.tb_gl_dimension_value.update({
      where: { id: data.id, doc_version: data.doc_version },
      data: {
        name: data.name?.trim(), name_local: data.name_local, effective_from: data.effective_from, effective_to: data.effective_to,
        is_active: data.is_active, doc_version: { increment: 1 }, updated_by_id: this.userId, updated_at: new Date(),
      },
    });
    return Result.ok({ id: updated.id, doc_version: updated.doc_version });
  }

  /**
   * Soft-delete a value no non-void document uses
   * ลบค่าที่ไม่มีเอกสารที่ไม่ void ใช้แบบ soft-delete
   * @param id - Value id / รหัสค่า
   * @returns `{ id }` / id ที่ลบ
   */
  @TryCatch
  async delete(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'delete', id }, GlDimensionValueService.name);
    const existing = await this.prismaService.tb_gl_dimension_value.findFirst({ where: { id, deleted_at: null } });
    if (!existing) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_NOT_FOUND);
    const jv = await this.prismaService.tb_gl_jv_detail_dimension.count({
      where: { gl_dimension_value_id: id, tb_gl_jv_detail: { tb_gl_jv_header: { jv_status: { not: 'void' }, deleted_at: null } } },
    });
    const ap = await this.prismaService.tb_ap_invoice_detail_dimension.count({
      where: { gl_dimension_value_id: id, tb_ap_invoice_detail: { tb_ap_invoice: { doc_status: { not: 'void' }, deleted_at: null } } },
    });
    if (jv + ap > 0) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_IN_USE, { code: existing.code });
    await this.prismaService.tb_gl_dimension_value.update({ where: { id }, data: { deleted_at: new Date(), deleted_by_id: this.userId } });
    return Result.ok({ id });
  }
}
```

`apps/micro-business/src/gl/gl-dimension/gl-account-dimension-rule.service.ts`
```ts
import { Injectable } from '@nestjs/common';
import { TryCatch, Result, GlAccountDimensionRuleResponseSchema } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import { TenantScopedService } from '@/common/tenant-scoped.service';
import { IPaginate } from '@/common/shared-interface/paginate.interface';
import QueryParams from '@/common/libs/paginate.query';
import { withDefaultSort } from '@/common/libs/default-sort';
import getPaginationParams from '@/common/helpers/pagination.params';
import { BackendLogger } from '@/common/helpers/backend.logger';
import { isUniqueConstraintViolation } from '@/common/helpers/is-unique-constraint-violation';
import { ICreateGlAccountDimensionRule, IUpdateGlAccountDimensionRule } from './interface/gl-dimension.interface';

/**
 * CRUD for account ↔ dimension rules (spec §5.1)
 * CRUD ของกฎ dimension ต่อบัญชี (spec §5.1)
 */
@Injectable()
export class GlAccountDimensionRuleService extends TenantScopedService {
  private readonly logger: BackendLogger = new BackendLogger(GlAccountDimensionRuleService.name);

  /**
   * Find one rule
   * ค้นหากฎหนึ่งข้อ
   * @param id - Rule id / รหัสกฎ
   * @returns The rule or NOT_FOUND / กฎ หรือ NOT_FOUND
   */
  @TryCatch
  async findOne(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findOne', id }, GlAccountDimensionRuleService.name);
    const row = await this.prismaService.tb_gl_account_dimension_rule.findFirst({ where: { id, deleted_at: null } });
    if (!row) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_NOT_FOUND);
    return Result.ok(GlAccountDimensionRuleResponseSchema.parse(row));
  }

  /**
   * Paginated list; filter `chart_of_accounts_id` narrows to one account
   * รายการแบบแบ่งหน้า ใช้ filter `chart_of_accounts_id` เพื่อจำกัดหนึ่งบัญชี
   * @param paginate - Pagination params / พารามิเตอร์แบ่งหน้า
   * @returns Paginated rules / กฎแบบแบ่งหน้า
   */
  @TryCatch
  async findAll(paginate: IPaginate): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findAll', paginate }, GlAccountDimensionRuleService.name);
    const q = new QueryParams(
      paginate.page, paginate.perpage, paginate.search, paginate.searchfields, [],
      typeof paginate.filter === 'object' && !Array.isArray(paginate.filter) ? paginate.filter : {},
      withDefaultSort(paginate.sort, ['created_at:asc']),
      paginate.advance,
    );
    const pagination = getPaginationParams(q.page, q.perpage);
    const rows = await this.prismaService.tb_gl_account_dimension_rule.findMany({ where: q.where(), orderBy: q.orderBy(), ...pagination });
    const total = await this.prismaService.tb_gl_account_dimension_rule.count({ where: q.where() });
    return Result.ok({
      paginate: { total, page: q.perpage < 0 ? 1 : q.page, perpage: q.perpage < 0 ? 1 : q.perpage, pages: total === 0 || q.perpage < 0 ? 1 : Math.ceil(total / q.perpage) },
      data: rows.map((r) => GlAccountDimensionRuleResponseSchema.parse(r)),
    });
  }

  /**
   * Create a rule; one rule per (account, dimension)
   * สร้างกฎ หนึ่งข้อต่อ (บัญชี, dimension)
   * @param data - Creation payload / ข้อมูลสร้าง
   * @returns `{ id, doc_version }` / id ที่สร้าง
   */
  @TryCatch
  async create(data: ICreateGlAccountDimensionRule): Promise<Result<unknown>> {
    this.logger.debug({ function: 'create', data }, GlAccountDimensionRuleService.name);
    const account = await this.prismaService.tb_chart_of_accounts.findFirst({ where: { id: data.chart_of_accounts_id, deleted_at: null } });
    if (!account) return Result.errorFromCatalog(ERROR_CATALOG.GL_ACCOUNT_NOT_POSTABLE);
    const dimension = await this.prismaService.tb_gl_dimension.findFirst({ where: { id: data.gl_dimension_id, deleted_at: null } });
    if (!dimension) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_NOT_FOUND);
    try {
      const created = await this.prismaService.tb_gl_account_dimension_rule.create({
        data: { chart_of_accounts_id: data.chart_of_accounts_id, gl_dimension_id: data.gl_dimension_id, requirement: data.requirement, created_by_id: this.userId },
      });
      return Result.ok({ id: created.id, doc_version: created.doc_version });
    } catch (error) {
      if (isUniqueConstraintViolation(error)) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_DUPLICATE_CODE, { code: `${account.code}/${dimension.code}` });
      throw error;
    }
  }

  /**
   * Change the requirement of a rule
   * เปลี่ยนระดับข้อบังคับของกฎ
   * @param data - Update payload / ข้อมูลแก้ไข
   * @returns `{ id, doc_version }` / id และเวอร์ชันใหม่
   */
  @TryCatch
  async update(data: IUpdateGlAccountDimensionRule): Promise<Result<unknown>> {
    this.logger.debug({ function: 'update', data }, GlAccountDimensionRuleService.name);
    const existing = await this.prismaService.tb_gl_account_dimension_rule.findFirst({ where: { id: data.id, deleted_at: null } });
    if (!existing) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_NOT_FOUND);
    const updated = await this.prismaService.tb_gl_account_dimension_rule.update({
      where: { id: data.id, doc_version: data.doc_version },
      data: { requirement: data.requirement, doc_version: { increment: 1 }, updated_by_id: this.userId, updated_at: new Date() },
    });
    return Result.ok({ id: updated.id, doc_version: updated.doc_version });
  }

  /**
   * Soft-delete a rule (rules carry no document references)
   * ลบกฎแบบ soft-delete (กฎไม่ถูกเอกสารอ้าง)
   * @param id - Rule id / รหัสกฎ
   * @returns `{ id }` / id ที่ลบ
   */
  @TryCatch
  async delete(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'delete', id }, GlAccountDimensionRuleService.name);
    const existing = await this.prismaService.tb_gl_account_dimension_rule.findFirst({ where: { id, deleted_at: null } });
    if (!existing) return Result.errorFromCatalog(ERROR_CATALOG.GL_DIMENSION_NOT_FOUND);
    await this.prismaService.tb_gl_account_dimension_rule.update({ where: { id }, data: { deleted_at: new Date(), deleted_by_id: this.userId } });
    return Result.ok({ id });
  }
}
```

- [ ] **Step 5: controllers** (ใช้ literal pattern ชั่วคราวก่อน generate)

`apps/micro-business/src/gl/gl-dimension/gl-dimension.controller.ts`
```ts
import { Controller, HttpStatus } from '@nestjs/common';
import { MessagePattern, Payload } from '@nestjs/microservices';
import { BaseMicroserviceController, MicroservicePayload, MicroserviceResponse } from '@/common';
import { BackendLogger } from '@/common/helpers/backend.logger';
import { TenantContextRunner } from '@/tenant/tenant-context.runner';
import { GlDimensionService } from './gl-dimension.service';

const SERVICE = 'micro-business';

/**
 * RPC handlers for dimension definitions
 * ตัวรับ RPC ของนิยาม dimension
 */
@Controller()
export class GlDimensionController extends BaseMicroserviceController {
  private readonly logger: BackendLogger = new BackendLogger(GlDimensionController.name);

  constructor(private readonly service: GlDimensionService, private readonly ctx: TenantContextRunner) {
    super();
  }

  /** Find one / ค้นหาหนึ่งรายการ @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'gl-dimension.find-one', service: SERVICE })
  async findOne(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'findOne', payload }, GlDimensionController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.findOne(payload.id)));
  }

  /** List / รายการ @param payload RPC payload @returns Paginated envelope */
  @MessagePattern({ cmd: 'gl-dimension.find-all', service: SERVICE })
  async findAll(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'findAll', payload }, GlDimensionController.name);
    return this.handlePaginatedResult(await this.ctx.run(payload, () => this.service.findAll(payload.paginate)));
  }

  /** Create / สร้าง @param payload RPC payload @returns Envelope 201 */
  @MessagePattern({ cmd: 'gl-dimension.create', service: SERVICE })
  async create(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'create', payload }, GlDimensionController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.create(payload.data)), HttpStatus.CREATED);
  }

  /** Update / แก้ไข @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'gl-dimension.update', service: SERVICE })
  async update(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'update', payload }, GlDimensionController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.update({ ...payload.data, id: payload.id })));
  }

  /** Delete / ลบ @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'gl-dimension.delete', service: SERVICE })
  async delete(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'delete', payload }, GlDimensionController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.delete(payload.id)));
  }
}
```
สร้าง `gl-dimension-value.controller.ts` และ `gl-account-dimension-rule.controller.ts` ด้วยเนื้อหาเดียวกันทุกบรรทัด โดยเปลี่ยน: ชื่อ class เป็น `GlDimensionValueController` / `GlAccountDimensionRuleController`, service เป็น `GlDimensionValueService` / `GlAccountDimensionRuleService`, และ cmd prefix เป็น `gl-dimension-value.` / `gl-account-dimension-rule.` (5 handler: find-one, find-all, create, update, delete)

- [ ] **Step 6: module + register**

`apps/micro-business/src/gl/gl-dimension/gl-dimension.module.ts`
```ts
import { Module } from '@nestjs/common';
import { TenantModule } from '@/tenant/tenant.module';
import { CommonModule } from '@/common/common.module';
import { GlDimensionController } from './gl-dimension.controller';
import { GlDimensionValueController } from './gl-dimension-value.controller';
import { GlAccountDimensionRuleController } from './gl-account-dimension-rule.controller';
import { GlDimensionService } from './gl-dimension.service';
import { GlDimensionValueService } from './gl-dimension-value.service';
import { GlAccountDimensionRuleService } from './gl-account-dimension-rule.service';

/**
 * Accounting dimension module: definitions, values and account rules
 * โมดูล accounting dimension: นิยาม ค่า และกฎต่อบัญชี
 */
@Module({
  imports: [TenantModule, CommonModule],
  controllers: [GlDimensionController, GlDimensionValueController, GlAccountDimensionRuleController],
  providers: [GlDimensionService, GlDimensionValueService, GlAccountDimensionRuleService],
  exports: [GlDimensionService, GlDimensionValueService, GlAccountDimensionRuleService],
})
export class GlDimensionModule {}
```
ใน `apps/micro-business/src/app.module.ts`: เพิ่ม `import { GlDimensionModule } from './gl/gl-dimension/gl-dimension.module';` ใกล้ L107 และ `GlDimensionModule,` ใน imports array หลัง `GlJvPrefixModule,` (L406)

- [ ] **Step 7: generate contract แล้วแทน literal**

```bash
bun run gen:rpc-contract
```
Expected: ได้ไฟล์ `packages/rpc-contract/src/contracts/gl-dimension.ts` (export `GlDimension`), `gl-dimension-value.ts` (`GlDimensionValue`), `gl-account-dimension-rule.ts` (`GlAccountDimensionRule`) และ barrel `index.ts` อัปเดต
แก้ทั้ง 3 controller: `import { GlDimension } from '@repo/rpc-contract';` แล้วแทน `@MessagePattern({ cmd: 'gl-dimension.find-one', service: SERVICE })` → `@MessagePattern(GlDimension.findOne.pattern)` ครบ 5 handler ต่อ controller (findAll, findOne, create, update, delete) ลบ `const SERVICE`
```bash
bun run build:package && bun run audit:tcp-drift
```

- [ ] **Step 8: gateway config module — gl-dimensions**

`apps/backend-gateway/src/config/config_gl-dimensions/swagger/request.ts`
```ts
import { createZodDto } from 'nestjs-zod';
import { z } from 'zod/v4';

/** Create body / เนื้อหาคำขอสร้าง */
export const GlDimensionCreateSchema = z.object({
  code: z.string().trim().toLowerCase().regex(/^[a-z0-9_]{2,20}$/).meta({ example: 'market' }),
  name: z.string().trim().min(1).meta({ example: 'Market Segment' }),
  name_local: z.string().trim().nullable().optional(),
  description: z.string().trim().nullable().optional(),
  sequence: z.number().int().min(0).default(0),
  is_active: z.boolean().default(true),
});
export class GlDimensionCreateDto extends createZodDto(GlDimensionCreateSchema) {}

/** Update body / เนื้อหาคำขอแก้ไข */
export const GlDimensionUpdateSchema = z.object({
  name: z.string().trim().min(1).optional(),
  name_local: z.string().trim().nullable().optional(),
  description: z.string().trim().nullable().optional(),
  sequence: z.number().int().min(0).optional(),
  is_active: z.boolean().optional(),
  doc_version: z.number().int().meta({ example: 1 }),
});
export class GlDimensionUpdateDto extends createZodDto(GlDimensionUpdateSchema) {}
```

`apps/backend-gateway/src/config/config_gl-dimensions/swagger/response.ts`
```ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

/** Dimension response / การตอบกลับ dimension */
export class GlDimensionResponseDto {
  @ApiProperty({ example: '019638a6-2a00-7c4f-8e46-9b7a52c80c4d' }) id: string;
  @ApiProperty({ example: 'market' }) code: string;
  @ApiProperty({ example: 'Market Segment' }) name: string;
  @ApiPropertyOptional({ example: 'กลุ่มตลาด' }) name_local?: string;
  @ApiProperty({ example: 1 }) sequence: number;
  @ApiProperty({ example: true }) is_active: boolean;
  @ApiPropertyOptional({ example: 0 }) doc_version?: number;
}

/** Mutation response / การตอบกลับหลังแก้ไข */
export class GlDimensionMutationResponseDto {
  @ApiProperty({ example: '019638a6-2a00-7c4f-8e46-9b7a52c80c4d' }) id: string;
  @ApiPropertyOptional({ example: 1 }) doc_version?: number;
}
```

`apps/backend-gateway/src/config/config_gl-dimensions/config_gl-dimensions.service.ts`
```ts
import { Injectable } from '@nestjs/common';
import { Result } from '@/common';
import { BackendLogger } from 'src/common/helpers/backend.logger';
import { RpcClient } from '@repo/rpc-client';
import { GlDimension } from '@repo/rpc-contract';

/**
 * Gateway proxy for dimension definitions
 * ตัวส่งต่อของ gateway สำหรับนิยาม dimension
 */
@Injectable()
export class ConfigGlDimensionsService {
  private readonly logger: BackendLogger = new BackendLogger(ConfigGlDimensionsService.name);

  constructor(private readonly rpc: RpcClient) {}

  /** Find one / ค้นหาหนึ่งรายการ */
  async findOne(id: string, user_id: string, bu_code: string, version: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findOne', id, version }, ConfigGlDimensionsService.name);
    return this.rpc.call(GlDimension.findOne, { id, user_id, bu_code, version });
  }

  /** List / รายการ */
  async findAll(paginate: unknown, user_id: string, bu_code: string, version: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findAll', version }, ConfigGlDimensionsService.name);
    return this.rpc.call(GlDimension.findAll, { paginate, user_id, bu_code, version });
  }

  /** Create / สร้าง */
  async create(data: unknown, user_id: string, bu_code: string, version: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'create', version }, ConfigGlDimensionsService.name);
    return this.rpc.call(GlDimension.create, { data, user_id, bu_code, version });
  }

  /** Update / แก้ไข */
  async update(id: string, data: unknown, user_id: string, bu_code: string, version: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'update', id, version }, ConfigGlDimensionsService.name);
    return this.rpc.call(GlDimension.update, { id, data, user_id, bu_code, version });
  }

  /** Delete / ลบ */
  async delete(id: string, user_id: string, bu_code: string, version: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'delete', id, version }, ConfigGlDimensionsService.name);
    return this.rpc.call(GlDimension.delete, { id, user_id, bu_code, version });
  }
}
```

`apps/backend-gateway/src/config/config_gl-dimensions/config_gl-dimensions.controller.ts`
```ts
import { Body, Controller, Delete, Get, HttpCode, HttpStatus, Param, ParseUUIDPipe, Post, Put, Query, Req, Res, UseGuards } from '@nestjs/common';
import { Request, Response } from 'express';
import { ApiBearerAuth, ApiBody, ApiOperation, ApiParam, ApiQuery, ApiTags } from '@nestjs/swagger';
import { KeycloakGuard } from 'src/auth/guards/keycloak.guard';
import { BaseHttpController } from '@/common';
import { IPaginateQuery, PaginateQuery } from 'src/shared-dto/paginate.dto';
import { ApiVersionMinRequest } from 'src/common/decorators/userfilter.decorator';
import { ExtractRequestHeader } from 'src/common/helpers/extract_header';
import { BackendLogger } from 'src/common/helpers/backend.logger';
import { AppIdGuard } from 'src/common/guard/app-id.guard';
import { ApiHeaderRequiredXAppId } from 'src/common/decorators/x-app-id.decorator';
import { ApiStdResponse } from '@/common/swagger/std-response';
import { ConfigGlDimensionsService } from './config_gl-dimensions.service';
import { GlDimensionCreateDto, GlDimensionUpdateDto } from './swagger/request';
import { GlDimensionMutationResponseDto, GlDimensionResponseDto } from './swagger/response';

/**
 * Accounting dimension definition endpoints
 * endpoint นิยาม accounting dimension
 */
@Controller('api/config/:bu_code/gl-dimensions')
@ApiTags('Config: Accounting')
@ApiHeaderRequiredXAppId()
@UseGuards(KeycloakGuard)
@ApiBearerAuth()
export class ConfigGlDimensionsController extends BaseHttpController {
  private readonly logger: BackendLogger = new BackendLogger(ConfigGlDimensionsController.name);

  constructor(private readonly service: ConfigGlDimensionsService) {
    super();
  }

  /** Get one / อ่านหนึ่งรายการ */
  @Get(':gl_dimension_id')
  @UseGuards(new AppIdGuard('gl-dimensions.findOne'))
  @HttpCode(HttpStatus.OK)
  @ApiVersionMinRequest()
  @ApiOperation({ summary: 'Get a dimension by ID', operationId: 'glDimensions_findOne' })
  @ApiParam({ name: 'bu_code', example: 'BU-001' })
  @ApiParam({ name: 'gl_dimension_id', example: '019638a6-2a00-7c4f-8e46-9b7a52c80c4d' })
  @ApiStdResponse(GlDimensionResponseDto)
  async findOne(@Req() req: Request, @Res() res: Response, @Param('gl_dimension_id', new ParseUUIDPipe({ version: '4' })) id: string, @Param('bu_code') bu_code: string, @Query('version') version: string = 'latest'): Promise<void> {
    this.logger.debug({ function: 'findOne', id, version }, ConfigGlDimensionsController.name);
    const { user_id } = ExtractRequestHeader(req);
    this.respond(res, await this.service.findOne(id, user_id, bu_code, version));
  }

  /** List / รายการ */
  @Get()
  @UseGuards(new AppIdGuard('gl-dimensions.findAll'))
  @HttpCode(HttpStatus.OK)
  @ApiVersionMinRequest()
  @ApiOperation({ summary: 'List dimensions', operationId: 'glDimensions_findAll' })
  @ApiParam({ name: 'bu_code', example: 'BU-001' })
  @ApiQuery({ name: 'version', required: false, example: 'latest' })
  @ApiStdResponse(GlDimensionResponseDto, { isArray: true })
  async findAll(@Req() req: Request, @Res() res: Response, @Param('bu_code') bu_code: string, @Query() query: IPaginateQuery, @Query('version') version: string = 'latest'): Promise<void> {
    this.logger.debug({ function: 'findAll', version }, ConfigGlDimensionsController.name);
    const { user_id } = ExtractRequestHeader(req);
    this.respond(res, await this.service.findAll(PaginateQuery(query), user_id, bu_code, version));
  }

  /** Create / สร้าง */
  @Post()
  @UseGuards(new AppIdGuard('gl-dimensions.create'))
  @HttpCode(HttpStatus.CREATED)
  @ApiVersionMinRequest()
  @ApiOperation({ summary: 'Create a dimension', operationId: 'glDimensions_create' })
  @ApiParam({ name: 'bu_code', example: 'BU-001' })
  @ApiBody({ type: GlDimensionCreateDto })
  @ApiStdResponse(GlDimensionMutationResponseDto)
  async create(@Req() req: Request, @Res() res: Response, @Param('bu_code') bu_code: string, @Body() body: GlDimensionCreateDto, @Query('version') version: string = 'latest'): Promise<void> {
    this.logger.debug({ function: 'create', version }, ConfigGlDimensionsController.name);
    const { user_id } = ExtractRequestHeader(req);
    this.respond(res, await this.service.create(body, user_id, bu_code, version));
  }

  /** Update / แก้ไข */
  @Put(':gl_dimension_id')
  @UseGuards(new AppIdGuard('gl-dimensions.update'))
  @HttpCode(HttpStatus.OK)
  @ApiVersionMinRequest()
  @ApiOperation({ summary: 'Update a dimension', operationId: 'glDimensions_update' })
  @ApiParam({ name: 'bu_code', example: 'BU-001' })
  @ApiParam({ name: 'gl_dimension_id', example: '019638a6-2a00-7c4f-8e46-9b7a52c80c4d' })
  @ApiBody({ type: GlDimensionUpdateDto })
  @ApiStdResponse(GlDimensionMutationResponseDto)
  async update(@Req() req: Request, @Res() res: Response, @Param('gl_dimension_id', new ParseUUIDPipe({ version: '4' })) id: string, @Param('bu_code') bu_code: string, @Body() body: GlDimensionUpdateDto, @Query('version') version: string = 'latest'): Promise<void> {
    this.logger.debug({ function: 'update', id, version }, ConfigGlDimensionsController.name);
    const { user_id } = ExtractRequestHeader(req);
    this.respond(res, await this.service.update(id, body, user_id, bu_code, version));
  }

  /** Delete / ลบ */
  @Delete(':gl_dimension_id')
  @UseGuards(new AppIdGuard('gl-dimensions.delete'))
  @HttpCode(HttpStatus.OK)
  @ApiVersionMinRequest()
  @ApiOperation({ summary: 'Delete a dimension', operationId: 'glDimensions_delete' })
  @ApiParam({ name: 'bu_code', example: 'BU-001' })
  @ApiParam({ name: 'gl_dimension_id', example: '019638a6-2a00-7c4f-8e46-9b7a52c80c4d' })
  @ApiStdResponse(GlDimensionMutationResponseDto)
  async delete(@Req() req: Request, @Res() res: Response, @Param('gl_dimension_id', new ParseUUIDPipe({ version: '4' })) id: string, @Param('bu_code') bu_code: string, @Query('version') version: string = 'latest'): Promise<void> {
    this.logger.debug({ function: 'delete', id, version }, ConfigGlDimensionsController.name);
    const { user_id } = ExtractRequestHeader(req);
    this.respond(res, await this.service.delete(id, user_id, bu_code, version));
  }
}
```

`apps/backend-gateway/src/config/config_gl-dimensions/config_gl-dimensions.module.ts`
```ts
import { Module } from '@nestjs/common';
import { ConfigGlDimensionsController } from './config_gl-dimensions.controller';
import { ConfigGlDimensionsService } from './config_gl-dimensions.service';

/** Gateway module for dimension definitions / โมดูล gateway ของนิยาม dimension */
@Module({ controllers: [ConfigGlDimensionsController], providers: [ConfigGlDimensionsService] })
export class ConfigGlDimensionsModule {}
```

- [ ] **Step 9: gateway config module — gl-dimension-values และ gl-account-dimension-rules**

สร้างโฟลเดอร์ `config_gl-dimension-values` และ `config_gl-account-dimension-rules` ด้วยไฟล์ชุดเดียวกับ Step 8 (request.ts, response.ts, service, controller, module) โดยเปลี่ยนตามตาราง:

| สิ่งที่เปลี่ยน | gl-dimension-values | gl-account-dimension-rules |
|---|---|---|
| contract import | `GlDimensionValue` | `GlAccountDimensionRule` |
| class prefix | `ConfigGlDimensionValues*` | `ConfigGlAccountDimensionRules*` |
| route | `api/config/:bu_code/gl-dimension-values` | `api/config/:bu_code/gl-account-dimension-rules` |
| param name | `gl_dimension_value_id` | `gl_account_dimension_rule_id` |
| AppIdGuard prefix | `gl-dimension-values.` | `gl-account-dimension-rules.` |
| operationId prefix | `glDimensionValues_` | `glAccountDimensionRules_` |

request.ts ของ values:
```ts
export const GlDimensionValueCreateSchema = z.object({
  gl_dimension_id: z.string().uuid(),
  code: z.string().trim().toUpperCase().min(1).max(30).meta({ example: 'CORP' }),
  name: z.string().trim().min(1).meta({ example: 'Corporate' }),
  name_local: z.string().trim().nullable().optional(),
  effective_from: z.coerce.date().nullable().optional(),
  effective_to: z.coerce.date().nullable().optional(),
  is_active: z.boolean().default(true),
});
export class GlDimensionValueCreateDto extends createZodDto(GlDimensionValueCreateSchema) {}
export const GlDimensionValueUpdateSchema = z.object({
  name: z.string().trim().min(1).optional(),
  name_local: z.string().trim().nullable().optional(),
  effective_from: z.coerce.date().nullable().optional(),
  effective_to: z.coerce.date().nullable().optional(),
  is_active: z.boolean().optional(),
  doc_version: z.number().int(),
});
export class GlDimensionValueUpdateDto extends createZodDto(GlDimensionValueUpdateSchema) {}
```
request.ts ของ rules:
```ts
import { enum_gl_account_dimension_rule_requirement } from '@repo/prisma-shared-schema-tenant';
export const GlAccountDimensionRuleCreateSchema = z.object({
  chart_of_accounts_id: z.string().uuid(),
  gl_dimension_id: z.string().uuid(),
  requirement: z.nativeEnum(enum_gl_account_dimension_rule_requirement),
});
export class GlAccountDimensionRuleCreateDto extends createZodDto(GlAccountDimensionRuleCreateSchema) {}
export const GlAccountDimensionRuleUpdateSchema = z.object({
  requirement: z.nativeEnum(enum_gl_account_dimension_rule_requirement),
  doc_version: z.number().int(),
});
export class GlAccountDimensionRuleUpdateDto extends createZodDto(GlAccountDimensionRuleUpdateSchema) {}
```
response.ts ของทั้งสอง: class `<Name>ResponseDto` มี `id`, ฟิลด์ตาม serializer ของ Step 2, และ `<Name>MutationResponseDto { id; doc_version? }` (ใส่ `@ApiProperty` ทุกฟิลด์ พร้อม `enumName: 'enum_gl_account_dimension_rule_requirement'` สำหรับ requirement)

ใน `apps/backend-gateway/src/config/route-config.ts`: import 3 module และเพิ่มใน `imports` หลัง `ConfigGlJvPrefixesModule,`:
```ts
    ConfigGlDimensionsModule,
    ConfigGlDimensionValuesModule,
    ConfigGlAccountDimensionRulesModule,
```

- [ ] **Step 10: type-check + gates + commit**

```bash
bun run build:package && bun run check-types
npx eslint --no-fix apps/micro-business/src/gl/gl-dimension apps/backend-gateway/src/config/config_gl-dimensions apps/backend-gateway/src/config/config_gl-dimension-values apps/backend-gateway/src/config/config_gl-account-dimension-rules
bun run audit:tcp-drift && bun run audit:tenant-context && bun run audit:guard-providers
git add -A apps/micro-business/src/gl/gl-dimension apps/micro-business/src/app.module.ts apps/micro-business/src/common/dto/index.ts packages/rpc-contract apps/backend-gateway/src/config
git commit -m "feat(gl): add accounting dimension definitions, values and account rules

Spec §5.1"
```

---

### Task 5: Bank account master

**Files:**
- Create: `apps/micro-business/src/master/bank-account/interface/bank-account.interface.ts`
- Create: `apps/micro-business/src/master/bank-account/dto/bank-account.serializer.ts`
- Create: `apps/micro-business/src/master/bank-account/bank-account.service.ts`
- Create: `apps/micro-business/src/master/bank-account/bank-account.controller.ts`
- Create: `apps/micro-business/src/master/bank-account/bank-account.module.ts`
- Modify: `apps/micro-business/src/app.module.ts`, `apps/micro-business/src/common/dto/index.ts`
- Generated: `packages/rpc-contract/src/contracts/bank-account.ts` (`BankAccount`)
- Create: `apps/backend-gateway/src/config/config_bank-accounts/{controller,service,module,swagger/request.ts,swagger/response.ts}`
- Modify: `apps/backend-gateway/src/config/route-config.ts`

**Interfaces:**
- Produces: `BankAccountResponseSchema`, RPC `BankAccount.{findAll,findOne,create,update,delete}`

- [ ] **Step 1: interface + serializer**

`interface/bank-account.interface.ts`
```ts
/** Create payload / ข้อมูลสร้างบัญชีธนาคาร */
export interface ICreateBankAccount {
  code: string;
  name: string;
  bank_name?: string | null;
  bank_branch?: string | null;
  account_no: string;
  currency_id: string;
  chart_of_accounts_id: string;
  is_active?: boolean;
  description?: string | null;
  note?: string | null;
}

/** Update payload / ข้อมูลแก้ไขบัญชีธนาคาร */
export interface IUpdateBankAccount {
  id: string;
  doc_version: number;
  name?: string;
  bank_name?: string | null;
  bank_branch?: string | null;
  account_no?: string;
  currency_id?: string;
  chart_of_accounts_id?: string;
  is_active?: boolean;
  description?: string | null;
  note?: string | null;
}
```
`dto/bank-account.serializer.ts`
```ts
import { z } from 'zod/v4';

/** Response shape for one bank account / รูปแบบการตอบกลับบัญชีธนาคาร */
export const BankAccountResponseSchema = z.object({
  id: z.string(),
  code: z.string(),
  name: z.string(),
  bank_name: z.string().nullable().optional(),
  bank_branch: z.string().nullable().optional(),
  account_no: z.string(),
  currency_id: z.string(),
  currency_code: z.string(),
  chart_of_accounts_id: z.string(),
  is_active: z.boolean(),
  description: z.string().nullable().optional(),
  note: z.string().nullable().optional(),
  doc_version: z.number().nullable().optional(),
  created_at: z.coerce.date().nullable().optional(),
  updated_at: z.coerce.date().nullable().optional(),
});
```
เพิ่ม `export * from '@/master/bank-account/dto/bank-account.serializer';` ใน `common/dto/index.ts`

- [ ] **Step 2: service**

`bank-account.service.ts`
```ts
import { Injectable } from '@nestjs/common';
import { TryCatch, Result, BankAccountResponseSchema } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import { enum_chart_of_accounts_type } from '@repo/prisma-shared-schema-tenant';
import { TenantScopedService } from '@/common/tenant-scoped.service';
import { IPaginate } from '@/common/shared-interface/paginate.interface';
import QueryParams from '@/common/libs/paginate.query';
import { withDefaultSort } from '@/common/libs/default-sort';
import getPaginationParams from '@/common/helpers/pagination.params';
import { BackendLogger } from '@/common/helpers/backend.logger';
import { isUniqueConstraintViolation } from '@/common/helpers/is-unique-constraint-violation';
import { ICreateBankAccount, IUpdateBankAccount } from './interface/bank-account.interface';

/**
 * Bank account master used by AP payments (spec §5.5)
 * ข้อมูลหลักบัญชีธนาคารที่ใบจ่ายเงิน AP ใช้ (spec §5.5)
 */
@Injectable()
export class BankAccountService extends TenantScopedService {
  private readonly logger: BackendLogger = new BackendLogger(BankAccountService.name);

  /** Find one / ค้นหาหนึ่งรายการ @param id Bank account id @returns Row or NOT_FOUND */
  @TryCatch
  async findOne(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findOne', id }, BankAccountService.name);
    const row = await this.prismaService.tb_bank_account.findFirst({ where: { id, deleted_at: null } });
    if (!row) return Result.errorFromCatalog(ERROR_CATALOG.BANK_ACCOUNT_NOT_FOUND);
    return Result.ok(BankAccountResponseSchema.parse(row));
  }

  /** Paginated list / รายการแบบแบ่งหน้า @param paginate Params @returns Paginated rows */
  @TryCatch
  async findAll(paginate: IPaginate): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findAll', paginate }, BankAccountService.name);
    const q = new QueryParams(
      paginate.page, paginate.perpage, paginate.search, paginate.searchfields,
      ['code', 'name', 'bank_name', 'account_no'],
      typeof paginate.filter === 'object' && !Array.isArray(paginate.filter) ? paginate.filter : {},
      withDefaultSort(paginate.sort, ['code:asc']),
      paginate.advance,
    );
    const pagination = getPaginationParams(q.page, q.perpage);
    const rows = await this.prismaService.tb_bank_account.findMany({ where: q.where(), orderBy: q.orderBy(), ...pagination });
    const total = await this.prismaService.tb_bank_account.count({ where: q.where() });
    return Result.ok({
      paginate: { total, page: q.perpage < 0 ? 1 : q.page, perpage: q.perpage < 0 ? 1 : q.perpage, pages: total === 0 || q.perpage < 0 ? 1 : Math.ceil(total / q.perpage) },
      data: rows.map((r) => BankAccountResponseSchema.parse(r)),
    });
  }

  /**
   * Resolve currency code and confirm the GL account is postable
   * หารหัสสกุลเงินและยืนยันว่าบัญชี GL ลงบัญชีได้
   * @param currencyId - Currency id / รหัสสกุลเงิน
   * @param accountId - COA id / รหัสบัญชี
   * @returns Currency code, or an error / รหัสสกุลเงิน หรือข้อผิดพลาด
   */
  private async resolveRefs(currencyId: string, accountId: string): Promise<Result<string>> {
    const currency = await this.prismaService.tb_currency.findFirst({ where: { id: currencyId, deleted_at: null }, select: { code: true } });
    if (!currency) return Result.errorFromCatalog(ERROR_CATALOG.BANK_ACCOUNT_NOT_FOUND, { reason: 'currency' });
    const account = await this.prismaService.tb_chart_of_accounts.findFirst({ where: { id: accountId, deleted_at: null, is_active: true } });
    if (!account || account.type === enum_chart_of_accounts_type.header || account.type === enum_chart_of_accounts_type.summary) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_ACCOUNT_NOT_POSTABLE);
    }
    return Result.ok(currency.code);
  }

  /** Create / สร้าง @param data Payload @returns `{ id, doc_version }` */
  @TryCatch
  async create(data: ICreateBankAccount): Promise<Result<unknown>> {
    this.logger.debug({ function: 'create', data }, BankAccountService.name);
    const code = data.code.trim().toUpperCase();
    const refs = await this.resolveRefs(data.currency_id, data.chart_of_accounts_id);
    if (refs.isError()) return Result.error(refs.error);
    try {
      const created = await this.prismaService.tb_bank_account.create({
        data: {
          code, name: data.name.trim(), bank_name: data.bank_name ?? null, bank_branch: data.bank_branch ?? null,
          account_no: data.account_no.trim(), currency_id: data.currency_id, currency_code: refs.value,
          chart_of_accounts_id: data.chart_of_accounts_id, is_active: data.is_active ?? true,
          description: data.description ?? null, note: data.note ?? null, created_by_id: this.userId,
        },
      });
      return Result.ok({ id: created.id, doc_version: created.doc_version });
    } catch (error) {
      if (isUniqueConstraintViolation(error)) return Result.errorFromCatalog(ERROR_CATALOG.BANK_ACCOUNT_DUPLICATE_CODE, { code });
      throw error;
    }
  }

  /** Update (code immutable) / แก้ไข (code แก้ไม่ได้) @param data Payload @returns `{ id, doc_version }` */
  @TryCatch
  async update(data: IUpdateBankAccount): Promise<Result<unknown>> {
    this.logger.debug({ function: 'update', data }, BankAccountService.name);
    const existing = await this.prismaService.tb_bank_account.findFirst({ where: { id: data.id, deleted_at: null } });
    if (!existing) return Result.errorFromCatalog(ERROR_CATALOG.BANK_ACCOUNT_NOT_FOUND);
    if (data.is_active === false && (await this.countPayments(data.id)) > 0) {
      return Result.errorFromCatalog(ERROR_CATALOG.BANK_ACCOUNT_IN_USE, { code: existing.code });
    }
    const refs = await this.resolveRefs(data.currency_id ?? existing.currency_id, data.chart_of_accounts_id ?? existing.chart_of_accounts_id);
    if (refs.isError()) return Result.error(refs.error);
    const updated = await this.prismaService.tb_bank_account.update({
      where: { id: data.id, doc_version: data.doc_version },
      data: {
        name: data.name?.trim(), bank_name: data.bank_name, bank_branch: data.bank_branch, account_no: data.account_no?.trim(),
        currency_id: data.currency_id, currency_code: refs.value, chart_of_accounts_id: data.chart_of_accounts_id,
        is_active: data.is_active, description: data.description, note: data.note,
        doc_version: { increment: 1 }, updated_by_id: this.userId, updated_at: new Date(),
      },
    });
    return Result.ok({ id: updated.id, doc_version: updated.doc_version });
  }

  /** Soft-delete when no payment references it / ลบเมื่อไม่มีใบจ่ายเงินอ้าง @param id Id @returns `{ id }` */
  @TryCatch
  async delete(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'delete', id }, BankAccountService.name);
    const existing = await this.prismaService.tb_bank_account.findFirst({ where: { id, deleted_at: null } });
    if (!existing) return Result.errorFromCatalog(ERROR_CATALOG.BANK_ACCOUNT_NOT_FOUND);
    if ((await this.countPayments(id)) > 0) return Result.errorFromCatalog(ERROR_CATALOG.BANK_ACCOUNT_IN_USE, { code: existing.code });
    await this.prismaService.tb_bank_account.update({ where: { id }, data: { deleted_at: new Date(), deleted_by_id: this.userId, is_active: false } });
    return Result.ok({ id });
  }

  /** Count non-void payments on this account / นับใบจ่ายที่ไม่ void @param id Id @returns Count */
  private async countPayments(id: string): Promise<number> {
    return this.prismaService.tb_ap_payment.count({ where: { bank_account_id: id, deleted_at: null, doc_status: { not: 'void' } } });
  }
}
```

- [ ] **Step 3: controller + module + register + contract**

`bank-account.controller.ts`: คัดลอก `gl-dimension.controller.ts` จาก Task 4 Step 5 ทั้งไฟล์ เปลี่ยน class เป็น `BankAccountController`, service เป็น `BankAccountService` (import จาก `./bank-account.service`), cmd prefix เป็น `bank-account.` (find-one, find-all, create, update, delete)

`bank-account.module.ts`
```ts
import { Module } from '@nestjs/common';
import { TenantModule } from '@/tenant/tenant.module';
import { CommonModule } from '@/common/common.module';
import { BankAccountController } from './bank-account.controller';
import { BankAccountService } from './bank-account.service';

/** Bank account master module / โมดูลข้อมูลหลักบัญชีธนาคาร */
@Module({ imports: [TenantModule, CommonModule], controllers: [BankAccountController], providers: [BankAccountService], exports: [BankAccountService] })
export class BankAccountModule {}
```
`app.module.ts`: import + `BankAccountModule,` หลัง `TaxProfileModule,` (L388)

```bash
bun run gen:rpc-contract
```
แทน literal ด้วย `BankAccount.findOne.pattern` ฯลฯ (import `BankAccount` จาก `@repo/rpc-contract`)

- [ ] **Step 4: gateway config_bank-accounts**

คัดลอกโครง `config_gl-dimensions` จาก Task 4 Step 8 ทั้ง 5 ไฟล์ เปลี่ยน: contract `BankAccount`, class prefix `ConfigBankAccounts*`, route `api/config/:bu_code/bank-accounts`, param `bank_account_id`, AppIdGuard prefix `bank-accounts.`, operationId prefix `bankAccounts_`

`swagger/request.ts`
```ts
export const BankAccountCreateSchema = z.object({
  code: z.string().trim().toUpperCase().min(1).max(20).meta({ example: 'KBANK-001' }),
  name: z.string().trim().min(1).meta({ example: 'KBank Current THB' }),
  bank_name: z.string().trim().nullable().optional(),
  bank_branch: z.string().trim().nullable().optional(),
  account_no: z.string().trim().min(1).max(50),
  currency_id: z.string().uuid(),
  chart_of_accounts_id: z.string().uuid(),
  is_active: z.boolean().default(true),
  description: z.string().trim().nullable().optional(),
  note: z.string().trim().nullable().optional(),
});
export class BankAccountCreateDto extends createZodDto(BankAccountCreateSchema) {}
export const BankAccountUpdateSchema = BankAccountCreateSchema.omit({ code: true }).partial().extend({ doc_version: z.number().int() });
export class BankAccountUpdateDto extends createZodDto(BankAccountUpdateSchema) {}
```
`swagger/response.ts`: `BankAccountResponseDto` (id, code, name, bank_name?, account_no, currency_code, chart_of_accounts_id, is_active, doc_version?) และ `BankAccountMutationResponseDto { id; doc_version? }`

`route-config.ts`: เพิ่ม `ConfigBankAccountsModule,`

- [ ] **Step 5: check + commit**

```bash
bun run build:package && bun run check-types
npx eslint --no-fix apps/micro-business/src/master/bank-account apps/backend-gateway/src/config/config_bank-accounts
bun run audit:tcp-drift
git add -A apps/micro-business/src/master/bank-account apps/micro-business/src/app.module.ts apps/micro-business/src/common/dto/index.ts packages/rpc-contract apps/backend-gateway/src/config
git commit -m "feat(master): add bank account master for AP payments

Spec §5.5"
```

---

### Task 6: GL posting refactor (postInTx / reverseInTx) และ dimension บน JV

**Files:**
- Modify: `apps/micro-business/src/gl/gl-jv/interface/gl-jv.interface.ts` (IGlJvLine + base amounts + dimensions)
- Modify: `apps/micro-business/src/gl/gl-jv/gl-jv.logic.ts` (`toBaseAmounts` เคารพ base ที่ส่งมา)
- Modify: `apps/micro-business/src/gl/gl-jv/gl-jv.service.ts:119-210` (`createInternal` เขียน dimension)
- Modify: `apps/micro-business/src/gl/gl-posting/gl-posting.service.ts` (`GlSetting` export + keys, `post`→`postInTx`, `reverse`→`reverseInTx`, `getSetting`)

**Interfaces:**
- Consumes: `validateLineDimensions`, `IDimensionRef` (Task 4)
- Produces:
  - `IGlJvLine.base_debit?/base_credit?/dimensions?`
  - `GlJvHeaderWithLines` type
  - `GlPostingService.getSetting(): Promise<GlSetting | null>`
  - `GlPostingService.postInTx(tx, header: GlJvHeaderWithLines, setting: GlSetting | null, postAt: Date | null): Promise<Result<unknown>>`
  - `GlPostingService.reverseInTx(tx, jvId, jvDate, canPost, meta: { source_ref_type: string; source_ref_id: string } | null): Promise<Result<unknown>>`
  - `GlSetting` มี keys AP ทั้ง 7 ตัว

- [ ] **Step 1: extend `IGlJvLine`** (gl-jv.interface.ts)

```ts
import { IDimensionRef } from '@/gl/gl-dimension/interface/gl-dimension.interface';

export interface IGlJvLine {
  sequence_no: number;
  chart_of_accounts_id: string;
  cost_center_id?: string | null;
  debit?: number | string | null;
  credit?: number | string | null;
  /** Base-currency amounts already rounded by the caller; when absent they are derived from rate / ยอดสกุลฐานที่ผู้เรียกปัดแล้ว ถ้าไม่ส่งจะคำนวณจาก rate */
  base_debit?: number | string | null;
  base_credit?: number | string | null;
  currency_id?: string | null;
  exchange_rate?: number | string | null;
  quantity?: number | string | null;
  description?: string | null;
  note?: string | null;
  dimensions?: IDimensionRef[] | null;
}
```

- [ ] **Step 2: `toBaseAmounts` เคารพ base ที่ส่งมา** (gl-jv.logic.ts แทนที่ฟังก์ชันเดิม)

```ts
export function toBaseAmounts(
  line: IGlJvLine,
  headerRate: Prisma.Decimal,
): { base_debit: Prisma.Decimal; base_credit: Prisma.Decimal } {
  if (line.base_debit !== null && line.base_debit !== undefined && line.base_credit !== null && line.base_credit !== undefined) {
    return { base_debit: new Prisma.Decimal(line.base_debit), base_credit: new Prisma.Decimal(line.base_credit) };
  }
  const rate =
    line.exchange_rate === null || line.exchange_rate === undefined
      ? headerRate
      : new Prisma.Decimal(line.exchange_rate);
  const debit = new Prisma.Decimal(line.debit ?? 0);
  const credit = new Prisma.Decimal(line.credit ?? 0);
  return { base_debit: debit.mul(rate), base_credit: credit.mul(rate) };
}
```

- [ ] **Step 3: `createInternal` ตรวจและเขียน dimension** (gl-jv.service.ts)

เพิ่ม import: `import { validateLineDimensions } from '../gl-dimension/gl-dimension.validator';`
หลังบรรทัด `const accounts = validated.value;` เพิ่ม:
```ts
    const dimensionCheck = await validateLineDimensions(tx, lines, data.jv_date);
    if (dimensionCheck.isError()) {
      return Result.error(dimensionCheck.error);
    }
    const resolvedValues = dimensionCheck.value;
```
หลัง `const created = await tx.tb_gl_jv_header.create({...})` และก่อน `return Result.ok(...)` เพิ่ม:
```ts
    await this.writeLineDimensions(tx, created.id, lines, resolvedValues);
```
เพิ่ม private method ใน class:
```ts
  /**
   * Persist dimension junction rows for the lines just created, keyed by sequence_no
   * บันทึกแถว junction ของ dimension ให้บรรทัดที่เพิ่งสร้าง โดยจับคู่ด้วย sequence_no
   * @param tx - Transaction client / client ของ transaction
   * @param headerId - New header id / id ของ header ใหม่
   * @param lines - Submitted lines / บรรทัดที่ส่งมา
   * @param resolved - Snapshot codes from the validator / รหัส snapshot จากตัวตรวจ
   */
  private async writeLineDimensions(
    tx: Prisma.TransactionClient,
    headerId: string,
    lines: IGlJvLine[],
    resolved: Map<string, IResolvedDimensionValue>,
  ): Promise<void> {
    const details = await tx.tb_gl_jv_detail.findMany({ where: { jv_header_id: headerId }, select: { id: true, sequence_no: true } });
    const detailIdBySeq = new Map(details.map((d) => [d.sequence_no, d.id]));
    const rows = lines.flatMap((line) =>
      (line.dimensions ?? []).map((ref) => {
        const snap = resolved.get(ref.gl_dimension_value_id);
        return {
          gl_jv_detail_id: detailIdBySeq.get(line.sequence_no) as string,
          gl_dimension_id: ref.gl_dimension_id,
          gl_dimension_value_id: ref.gl_dimension_value_id,
          dimension_code: snap?.dimension_code ?? '',
          value_code: snap?.value_code ?? '',
          created_by_id: this.userId,
        };
      }),
    );
    if (rows.length > 0) await tx.tb_gl_jv_detail_dimension.createMany({ data: rows });
  }
```
(import `IResolvedDimensionValue` จาก `../gl-dimension/interface/gl-dimension.interface`)

- [ ] **Step 4: refactor `GlPostingService`** (gl-posting.service.ts)

4a. ทำ `GlSetting` เป็น export และเพิ่ม keys:
```ts
/** The slice of `tb_application_config.gl_setting` the posting paths read / ส่วนของ gl_setting ที่ใช้ลงบัญชี */
export interface GlSetting {
  allow_post_to_closed_period?: boolean;
  reversal_prefix_id?: string;
  retained_earnings_account_id?: string;
  auto_jv_prefix_id?: string;
  ap_control_account_id?: string;
  input_vat_account_id?: string;
  wht_payable_account_id?: string;
  advance_deposit_account_id?: string;
  realized_fx_gain_account_id?: string;
  realized_fx_loss_account_id?: string;
  ap_jv_prefix_id?: string;
}

/** A header with its live lines, the shape `postInTx` consumes / header พร้อมบรรทัดที่ยังไม่ถูกลบ */
export type GlJvHeaderWithLines = Prisma.tb_gl_jv_headerGetPayload<{ include: { tb_gl_jv_detail: true } }>;
```
4b. เพิ่ม public accessor ถัดจาก `glSetting()`:
```ts
  /**
   * Public read of gl_setting for the subledger facade
   * อ่าน gl_setting แบบ public ให้ facade ของ subledger
   * @returns The setting or null / ค่าที่ตั้ง หรือ null
   */
  async getSetting(): Promise<GlSetting | null> {
    return this.glSetting();
  }
```
4c. แทนที่ `post()` ทั้งฟังก์ชัน (L293-392) ด้วย:
```ts
  async post(jvId: string, postAt?: Date | null): Promise<Result<unknown>> {
    this.logger.debug({ function: 'post', jvId, postAt }, GlPostingService.name);
    const header = await this.prismaService.tb_gl_jv_header.findFirst({
      where: { id: jvId, deleted_at: null },
      include: { tb_gl_jv_detail: { where: { deleted_at: null } } },
    });
    if (!header) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_JV_NOT_FOUND);
    }
    const setting = await this.glSetting();
    return this.prismaService.$transaction((tx) => this.postInTx(tx, header, setting, postAt ?? null));
  }

  /**
   * Post a loaded voucher inside the caller's transaction — the only ledger write path
   * ลงบัญชีใบที่โหลดแล้วภายใน transaction ของผู้เรียก — ทางเดียวที่เขียน ledger
   *
   * Extracted from `post()` so AP/AR can commit their document status and the ledger together.
   * The fiscal-year advisory lock is still the first statement that touches balances.
   * แยกออกจาก `post()` เพื่อให้ AP/AR commit สถานะเอกสารกับ ledger พร้อมกัน advisory lock ของปีบัญชี
   * ยังเป็น statement แรกที่แตะยอดคงเหลือเหมือนเดิม
   * @param tx - Active transaction client / client ของ transaction
   * @param header - Header with live lines / header พร้อมบรรทัด
   * @param setting - gl_setting read by the caller / gl_setting ที่ผู้เรียกอ่านมา
   * @param postAt - Future time to schedule, or null to post now / เวลาในอนาคตที่จะลง หรือ null คือลงทันที
   * @returns `{ id, jv_status }` / รหัสและสถานะ
   */
  async postInTx(
    tx: PrismaTx,
    header: GlJvHeaderWithLines,
    setting: GlSetting | null,
    postAt: Date | null,
  ): Promise<Result<unknown>> {
    const jvId = header.id;
    if (
      header.jv_status !== enum_jv_status.draft &&
      header.jv_status !== enum_jv_status.in_review &&
      header.jv_status !== enum_jv_status.scheduled
    ) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_JV_IMMUTABLE);
    }
    const headerRate = new Prisma.Decimal(header.exchange_rate ?? 1);
    const lines: IGlJvLine[] = header.tb_gl_jv_detail.map((d) => ({
      sequence_no: d.sequence_no,
      chart_of_accounts_id: d.chart_of_accounts_id,
      cost_center_id: d.cost_center_id,
      debit: d.debit?.toString() ?? '0',
      credit: d.credit?.toString() ?? '0',
      base_debit: d.base_debit?.toString() ?? '0',
      base_credit: d.base_credit?.toString() ?? '0',
      exchange_rate: d.exchange_rate?.toString() ?? null,
    }));
    const allowClosed = setting?.allow_post_to_closed_period === true;

    const period = await this.resolvePeriod(tx, header.jv_date, header.is_adjustment);
    if (!period) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_PERIOD_NOT_OPEN);
    }
    await this.lockFiscalYear(tx, period.fiscal_year);
    if (period.status !== enum_gl_period_status.open && !allowClosed) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_PERIOD_NOT_OPEN);
    }
    const validated = await validateJvLines(tx, lines, headerRate);
    if (validated.isError()) {
      return validated;
    }
    return this.writePosting(tx, header, period, postAt);
  }
```
และย้ายส่วนที่เหลือของ body เดิม (ตั้งแต่ `const baseLines = ...` ถึง `return Result.ok({ id: jvId, jv_status: enum_jv_status.posted });`) ไปเป็น private method ใหม่ `writePosting(tx, header, period, postAt)` โดยแทน `header.tb_gl_jv_detail` / `jvId = header.id` ตามเดิม (ไม่เปลี่ยนตรรกะแม้แต่บรรทัดเดียว)

4d. แทนที่ `reverse()` (L507-640) ด้วย:
```ts
  async reverse(jvId: string, jvDate: Date, canPost: boolean): Promise<Result<unknown>> {
    this.logger.debug({ function: 'reverse', jvId, jvDate, canPost }, GlPostingService.name);
    return this.prismaService.$transaction((tx) => this.reverseInTx(tx, jvId, jvDate, canPost, null));
  }

  /**
   * Build and (optionally) post the reversing voucher inside the caller's transaction
   * สร้างและ (ถ้าต้องการ) ลงบัญชีใบกลับรายการภายใน transaction ของผู้เรียก
   * @param tx - Active transaction client / client ของ transaction
   * @param jvId - Voucher to reverse / ใบที่จะกลับรายการ
   * @param jvDate - Date of the reversing voucher / วันที่ใบกลับรายการ
   * @param canPost - Post immediately or leave as draft / ลงทันทีหรือทิ้งเป็นร่าง
   * @param meta - Source back-reference copied onto the reversal, or null / การอ้างอิงต้นทางที่ใส่ให้ใบกลับ หรือ null
   * @returns `{ id, jv_status, reversal_of_jv_id }` / รหัส สถานะ และใบต้นทาง
   */
  async reverseInTx(
    tx: PrismaTx,
    jvId: string,
    jvDate: Date,
    canPost: boolean,
    meta: { source_ref_type: string; source_ref_id: string } | null,
  ): Promise<Result<unknown>> {
    const header = await tx.tb_gl_jv_header.findFirst({
      where: { id: jvId, deleted_at: null },
      include: { tb_gl_jv_detail: { where: { deleted_at: null }, orderBy: { sequence_no: 'asc' }, include: { tb_gl_jv_detail_dimension: true } } },
    });
    if (!header) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_JV_NOT_FOUND);
    }
    if (header.jv_status !== enum_jv_status.posted || header.reversed_by_jv_id) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_JV_IMMUTABLE);
    }
    const setting = await this.glSetting();
    if (!setting?.reversal_prefix_id) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_SETTING_MISSING);
    }
    const prefix = await tx.tb_gl_jv_prefix.findFirst({
      where: { id: setting.reversal_prefix_id, is_active: true, deleted_at: null },
      select: { id: true, code: true },
    });
    if (!prefix) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_JV_PREFIX_NOT_FOUND);
    }
    const target = await this.resolvePeriod(tx, jvDate, false);
    if (!target || (target.status !== enum_gl_period_status.open && setting.allow_post_to_closed_period !== true)) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_PERIOD_NOT_OPEN);
    }
    const createdId = await this.createReversalDraft(tx, header, prefix, jvDate, meta);
    if (!canPost) {
      return Result.ok({ id: createdId, jv_status: enum_jv_status.draft, reversal_of_jv_id: jvId });
    }
    const created = await tx.tb_gl_jv_header.findFirst({
      where: { id: createdId },
      include: { tb_gl_jv_detail: { where: { deleted_at: null } } },
    });
    const posted = await this.postInTx(tx, created as GlJvHeaderWithLines, setting, null);
    if (posted.isError()) {
      return posted;
    }
    try {
      await tx.tb_gl_jv_header.update({
        where: { id: jvId },
        data: { reversed_by_jv_id: createdId, doc_version: { increment: 1 }, updated_at: new Date(), updated_by_id: this.userId },
      });
    } catch (error) {
      if (isUniqueConstraintViolation(error)) {
        return Result.errorFromCatalog(ERROR_CATALOG.GL_JV_IMMUTABLE);
      }
      throw error;
    }
    return Result.ok({ id: createdId, jv_status: enum_jv_status.posted, reversal_of_jv_id: jvId });
  }

  /**
   * Write the Dr/Cr-swapped draft header + lines + dimension rows
   * เขียน header/บรรทัด/dimension ของใบร่างที่สลับ Dr/Cr แล้ว
   * @param tx - Active transaction client / client ของ transaction
   * @param header - Source voucher with lines and dimensions / ใบต้นทางพร้อมบรรทัดและ dimension
   * @param prefix - Reversal prefix / prefix ของใบกลับ
   * @param jvDate - Reversal date / วันที่ใบกลับ
   * @param meta - Source back-reference or null / การอ้างอิงต้นทาง หรือ null
   * @returns New header id / id ของ header ใหม่
   */
  private async createReversalDraft(
    tx: PrismaTx,
    header: Prisma.tb_gl_jv_headerGetPayload<{ include: { tb_gl_jv_detail: { include: { tb_gl_jv_detail_dimension: true } } } }>,
    prefix: { id: string; code: string },
    jvDate: Date,
    meta: { source_ref_type: string; source_ref_id: string } | null,
  ): Promise<string> {
    const jvNo = await generateJvNo({ commonLogic: this.commonLogic, prisma: tx, userId: this.userId, buCode: this.bu_code, prefixCode: prefix.code, jvDate });
    const totalDebit = header.tb_gl_jv_detail.reduce((sum, d) => sum.add(new Prisma.Decimal(d.base_credit ?? 0)), new Prisma.Decimal(0));
    const totalCredit = header.tb_gl_jv_detail.reduce((sum, d) => sum.add(new Prisma.Decimal(d.base_debit ?? 0)), new Prisma.Decimal(0));
    const created = await tx.tb_gl_jv_header.create({
      data: {
        prefix_id: prefix.id, prefix_code: prefix.code, jv_no: jvNo, jv_date: jvDate, is_adjustment: false,
        jv_status: enum_jv_status.draft, source: enum_gl_jv_source.reversal, reversal_of_jv_id: header.id,
        source_ref_type: meta?.source_ref_type ?? null, source_ref_id: meta?.source_ref_id ?? null,
        description: header.description, currency_id: header.currency_id, base_currency_id: header.base_currency_id,
        exchange_rate: header.exchange_rate, total_debit: totalDebit, total_credit: totalCredit,
        created_by_id: this.userId, updated_by_id: this.userId,
        tb_gl_jv_detail: {
          create: header.tb_gl_jv_detail.map((d) => ({
            sequence_no: d.sequence_no, chart_of_accounts_id: d.chart_of_accounts_id, account_code: d.account_code, account_name: d.account_name,
            cost_center_id: d.cost_center_id, debit: new Prisma.Decimal(d.credit ?? 0), credit: new Prisma.Decimal(d.debit ?? 0),
            base_debit: new Prisma.Decimal(d.base_credit ?? 0), base_credit: new Prisma.Decimal(d.base_debit ?? 0),
            currency_id: d.currency_id, exchange_rate: d.exchange_rate, quantity: d.quantity, description: d.description,
            created_by_id: this.userId, updated_by_id: this.userId,
            tb_gl_jv_detail_dimension: {
              create: d.tb_gl_jv_detail_dimension.map((dim) => ({
                gl_dimension_id: dim.gl_dimension_id, gl_dimension_value_id: dim.gl_dimension_value_id,
                dimension_code: dim.dimension_code, value_code: dim.value_code, created_by_id: this.userId,
              })),
            },
          })),
        },
      },
      select: { id: true },
    });
    return created.id;
  }
```
`runDueAutoReversals()` เรียก `this.reverse(...)` เดิมอยู่ ไม่ต้องแก้

- [ ] **Step 5: check + commit**

```bash
bun run check-types
npx eslint --no-fix apps/micro-business/src/gl/gl-jv apps/micro-business/src/gl/gl-posting
git add apps/micro-business/src/gl
git commit -m "refactor(gl): expose postInTx/reverseInTx and carry dimensions on JV lines

Behavior-preserving split so subledgers can post inside their own transaction (spec §5.3)"
```

---

### Task 7: Subledger posting facade + ตรวจ regression ของ GL ด้วยมือ

**Files:**
- Create: `apps/micro-business/src/gl/gl-subledger-posting/interface/gl-subledger-posting.interface.ts`
- Create: `apps/micro-business/src/gl/gl-subledger-posting/gl-subledger-posting.service.ts`
- Create: `apps/micro-business/src/gl/gl-subledger-posting/gl-subledger-posting.module.ts`
- Modify: `apps/micro-business/src/app.module.ts`

**Interfaces:**
- Consumes: `GlJvService.createInternal`, `GlPostingService.postInTx/reverseInTx/getSetting`, `GlSetting`, `IGlJvLine`
- Produces:
  - `ISubledgerPostingLine`, `ISubledgerPostingInput`, `ISubledgerPostingResult { jv_id; jv_no; already_posted }`
  - `GlSubledgerPostingService.postFromSource(input, tx): Promise<Result<ISubledgerPostingResult>>`
  - `GlSubledgerPostingService.reverseBySource(ref, reversalDate, reason, tx): Promise<Result<{ jv_id: string; jv_no: string }>>`
  - `GlSubledgerPostingService.requireSettingAccount(setting, key): Result<string>`

- [ ] **Step 1: interface**

```ts
import { enum_gl_jv_source, Prisma } from '@repo/prisma-shared-schema-tenant';
import { IDimensionRef } from '@/gl/gl-dimension/interface/gl-dimension.interface';

/** Source document types that may post through the facade / ชนิดเอกสารต้นทางที่ post ผ่าน facade ได้ */
export type SubledgerSourceRefType = 'ap_invoice' | 'ap_payment';

/**
 * One ledger line handed in by a subledger, already balanced by the caller
 * บรรทัด ledger หนึ่งบรรทัดที่ subledger ส่งมา ผู้เรียกทำให้สมดุลแล้ว
 */
export interface ISubledgerPostingLine {
  sequence_no: number;
  chart_of_accounts_id: string;
  cost_center_id?: string | null;
  debit?: Prisma.Decimal | null;
  credit?: Prisma.Decimal | null;
  base_debit: Prisma.Decimal;
  base_credit: Prisma.Decimal;
  currency_id?: string | null;
  exchange_rate?: Prisma.Decimal | null;
  description?: string | null;
  dimensions?: IDimensionRef[] | null;
}

/** Posting request from a subledger (spec §5.2) / คำขอลงบัญชีจาก subledger (spec §5.2) */
export interface ISubledgerPostingInput {
  source: enum_gl_jv_source;
  source_ref_type: SubledgerSourceRefType;
  source_ref_id: string;
  jv_date: Date;
  description: string;
  prefix_id?: string | null;
  currency_id: string;
  exchange_rate: Prisma.Decimal;
  lines: ISubledgerPostingLine[];
}

/** Posting outcome / ผลการลงบัญชี */
export interface ISubledgerPostingResult {
  jv_id: string;
  jv_no: string;
  already_posted: boolean;
}
```

- [ ] **Step 2: service**

```ts
import { Injectable } from '@nestjs/common';
import { Result } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import { enum_jv_status, Prisma } from '@repo/prisma-shared-schema-tenant';
import { TenantScopedService } from '@/common/tenant-scoped.service';
import { BackendLogger } from '@/common/helpers/backend.logger';
import { isUniqueConstraintViolation } from '@/common/helpers/is-unique-constraint-violation';
import { GlJvService } from '../gl-jv/gl-jv.service';
import { IGlJvLine } from '../gl-jv/interface/gl-jv.interface';
import { GlJvHeaderWithLines, GlPostingService, GlSetting } from '../gl-posting/gl-posting.service';
import { ISubledgerPostingInput, ISubledgerPostingResult, SubledgerSourceRefType } from './interface/gl-subledger-posting.interface';

/**
 * The one door subledgers use to reach the ledger (spec §5.2)
 * ประตูเดียวที่ subledger ใช้เข้าถึง ledger (spec §5.2)
 *
 * Idempotent per (source, source_ref_type, source_ref_id); runs entirely inside the caller's
 * transaction so the source document and its voucher commit or roll back together.
 * ทำซ้ำได้ต่อ (source, source_ref_type, source_ref_id) และทำงานใน transaction ของผู้เรียกทั้งหมด
 * เพื่อให้เอกสารต้นทางกับใบสำคัญ commit หรือ rollback ด้วยกัน
 */
@Injectable()
export class GlSubledgerPostingService extends TenantScopedService {
  private readonly logger: BackendLogger = new BackendLogger(GlSubledgerPostingService.name);

  constructor(
    private readonly glJvService: GlJvService,
    private readonly postingService: GlPostingService,
  ) {
    super();
  }

  /**
   * Read a required account id out of gl_setting or fail with the missing key
   * อ่าน account id ที่ต้องมีจาก gl_setting หรือคืน error พร้อมชื่อ key ที่ขาด
   * @param setting - gl_setting / ค่าที่ตั้ง
   * @param key - Key to read / key ที่จะอ่าน
   * @returns The account id / รหัสบัญชี
   */
  requireSettingAccount(setting: GlSetting | null, key: keyof GlSetting): Result<string> {
    const value = setting?.[key];
    if (typeof value !== 'string' || value.length === 0) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_SETTING_ACCOUNT_MISSING, { key });
    }
    return Result.ok(value);
  }

  /**
   * Create and post a voucher for a source document, or return the one already posted
   * สร้างและลงบัญชีใบสำคัญให้เอกสารต้นทาง หรือคืนใบที่ลงไปแล้ว
   * @param input - Posting request / คำขอลงบัญชี
   * @param tx - Caller's transaction / transaction ของผู้เรียก
   * @returns Voucher id/no and whether it pre-existed / รหัส เลขที่ และเคยมีอยู่แล้วหรือไม่
   */
  async postFromSource(
    input: ISubledgerPostingInput,
    tx: Prisma.TransactionClient,
  ): Promise<Result<ISubledgerPostingResult>> {
    this.logger.debug({ function: 'postFromSource', source_ref_type: input.source_ref_type, source_ref_id: input.source_ref_id }, GlSubledgerPostingService.name);
    const existing = await this.findLiveVoucher(tx, input.source_ref_type, input.source_ref_id);
    if (existing) {
      return Result.ok({ jv_id: existing.id, jv_no: existing.jv_no, already_posted: true });
    }
    const setting = await this.postingService.getSetting();
    const prefixId = input.prefix_id ?? setting?.ap_jv_prefix_id ?? setting?.auto_jv_prefix_id;
    if (!prefixId) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_SETTING_ACCOUNT_MISSING, { key: 'ap_jv_prefix_id' });
    }
    let created: Result<{ id: string; jv_no: string }>;
    try {
      created = await this.glJvService.createInternal(
        {
          prefix_id: prefixId,
          jv_date: input.jv_date,
          description: input.description,
          currency_id: input.currency_id,
          exchange_rate: input.exchange_rate.toString(),
          details: { add: input.lines.map(toJvLine) },
        },
        tx,
        { source: input.source, source_ref_type: input.source_ref_type, source_ref_id: input.source_ref_id },
      );
    } catch (error) {
      if (isUniqueConstraintViolation(error)) {
        const raced = await this.findLiveVoucher(tx, input.source_ref_type, input.source_ref_id);
        if (raced) return Result.ok({ jv_id: raced.id, jv_no: raced.jv_no, already_posted: true });
      }
      throw error;
    }
    if (created.isError()) {
      return Result.error(created.error);
    }
    const header = await tx.tb_gl_jv_header.findFirst({
      where: { id: created.value.id },
      include: { tb_gl_jv_detail: { where: { deleted_at: null } } },
    });
    const posted = await this.postingService.postInTx(tx, header as GlJvHeaderWithLines, setting, null);
    if (posted.isError()) {
      return Result.error(posted.error);
    }
    return Result.ok({ jv_id: created.value.id, jv_no: created.value.jv_no, already_posted: false });
  }

  /**
   * Reverse the posted voucher of a source document (used on void)
   * กลับรายการใบสำคัญที่ post จากเอกสารต้นทาง (ใช้ตอน void)
   * @param ref - Source reference / การอ้างอิงต้นทาง
   * @param ref.source_ref_type - Source document type / ชนิดเอกสารต้นทาง
   * @param ref.source_ref_id - Source document id / รหัสเอกสารต้นทาง
   * @param reversalDate - Date of the reversing voucher / วันที่ใบกลับ
   * @param reason - Reason recorded on the source voucher note / เหตุผลที่บันทึกใน note ของใบต้นทาง
   * @param tx - Caller's transaction / transaction ของผู้เรียก
   * @returns Reversal voucher id/no / รหัสและเลขที่ใบกลับ
   */
  async reverseBySource(
    ref: { source_ref_type: SubledgerSourceRefType; source_ref_id: string },
    reversalDate: Date,
    reason: string,
    tx: Prisma.TransactionClient,
  ): Promise<Result<{ jv_id: string; jv_no: string }>> {
    this.logger.debug({ function: 'reverseBySource', ...ref }, GlSubledgerPostingService.name);
    const voucher = await this.findLiveVoucher(tx, ref.source_ref_type, ref.source_ref_id);
    if (!voucher || voucher.jv_status !== enum_jv_status.posted) {
      return Result.errorFromCatalog(ERROR_CATALOG.GL_SUBLEDGER_SOURCE_NOT_FOUND, ref);
    }
    const reversed = await this.postingService.reverseInTx(tx, voucher.id, reversalDate, true, ref);
    if (reversed.isError()) {
      return Result.error(reversed.error);
    }
    const reversal = reversed.value as { id: string };
    const row = await tx.tb_gl_jv_header.update({
      where: { id: voucher.id },
      data: { note: reason, updated_at: new Date(), updated_by_id: this.userId },
      select: { id: true },
    });
    const created = await tx.tb_gl_jv_header.findFirst({ where: { id: reversal.id }, select: { id: true, jv_no: true } });
    this.logger.debug({ function: 'reverseBySource', reversed: row.id }, GlSubledgerPostingService.name);
    return Result.ok({ jv_id: created?.id ?? reversal.id, jv_no: created?.jv_no ?? '' });
  }

  /**
   * The non-void voucher a source document already produced, if any
   * ใบสำคัญที่ไม่ void ซึ่งเอกสารต้นทางเคยสร้างไว้ ถ้ามี
   * @param tx - Transaction client / client ของ transaction
   * @param sourceRefType - Source type / ชนิดต้นทาง
   * @param sourceRefId - Source id / รหัสต้นทาง
   * @returns Header slice or null / ส่วนของ header หรือ null
   */
  private async findLiveVoucher(tx: Prisma.TransactionClient, sourceRefType: string, sourceRefId: string) {
    return tx.tb_gl_jv_header.findFirst({
      where: { source_ref_type: sourceRefType, source_ref_id: sourceRefId, deleted_at: null, jv_status: { not: enum_jv_status.void }, source: { not: 'reversal' } },
      select: { id: true, jv_no: true, jv_status: true },
    });
  }
}

/**
 * Convert a facade line into the JV line shape createInternal takes
 * แปลงบรรทัดของ facade เป็นรูปบรรทัด JV ที่ createInternal รับ
 * @param line - Facade line / บรรทัดของ facade
 * @returns JV line / บรรทัด JV
 */
function toJvLine(line: ISubledgerPostingInput['lines'][number]): IGlJvLine {
  return {
    sequence_no: line.sequence_no,
    chart_of_accounts_id: line.chart_of_accounts_id,
    cost_center_id: line.cost_center_id ?? null,
    debit: line.debit?.toString() ?? '0',
    credit: line.credit?.toString() ?? '0',
    base_debit: line.base_debit.toString(),
    base_credit: line.base_credit.toString(),
    currency_id: line.currency_id ?? null,
    exchange_rate: line.exchange_rate?.toString() ?? null,
    description: line.description ?? null,
    dimensions: line.dimensions ?? null,
  };
}
```

- [ ] **Step 3: module + register**

```ts
import { Module } from '@nestjs/common';
import { TenantModule } from '@/tenant/tenant.module';
import { CommonModule } from '@/common/common.module';
import { GlJvModule } from '../gl-jv/gl-jv.module';
import { GlPostingModule } from '../gl-posting/gl-posting.module';
import { GlSubledgerPostingService } from './gl-subledger-posting.service';

/** Subledger posting facade module / โมดูล facade การลงบัญชีจาก subledger */
@Module({
  imports: [TenantModule, CommonModule, GlJvModule, GlPostingModule],
  providers: [GlSubledgerPostingService],
  exports: [GlSubledgerPostingService],
})
export class GlSubledgerPostingModule {}
```
`app.module.ts`: import + `GlSubledgerPostingModule,` หลัง `GlPostingModule,`

- [ ] **Step 4: check + commit**

```bash
bun run check-types && npx eslint --no-fix apps/micro-business/src/gl/gl-subledger-posting
git add apps/micro-business/src/gl/gl-subledger-posting apps/micro-business/src/app.module.ts
git commit -m "feat(gl): add subledger posting facade with idempotent postFromSource/reverseBySource

Spec §5.2"
```

- [ ] **Step 5: ตรวจ regression ของ GL ด้วยมือ** (spec §13 ข้อ 2) — รัน dev แล้ว curl ผ่าน gateway; ใส่ `$TOKEN`, `$BU`, `$APP_ID` จาก env ทดสอบ

```bash
bun run dev   # อีก terminal
export API=http://localhost:4000/api/$BU H="-H 'Authorization: Bearer $TOKEN' -H 'x-app-id: $APP_ID' -H 'Content-Type: application/json'"
# 1. สร้าง JV มือ 2 บรรทัด (ใช้ prefix_id/account ที่มีใน BU ทดสอบ)
curl -s -X POST $API/gl-jv $H -d '{"prefix_id":"<PREFIX>","jv_date":"2026-09-23","description":"regression","details":{"add":[{"sequence_no":1,"chart_of_accounts_id":"<ACC_A>","debit":100},{"sequence_no":2,"chart_of_accounts_id":"<ACC_B>","credit":100}]}}'
# 2. submit → (approve ตาม workflow ถ้ามี) → ต้องได้ jv_status posted และ tb_gl_balance ขยับ
curl -s -X POST $API/gl-jv/<JV_ID>/submit $H
# 3. reverse → ต้องได้ใบใหม่ posted และใบเดิมมี reversed_by_jv_id
curl -s -X POST $API/gl-posting/<JV_ID>/reverse $H -d '{"jv_date":"2026-09-23"}'
# 4. void ใบ reversal → ต้องได้ jv_status void
curl -s -X POST $API/gl-posting/<REV_ID>/void $H -d '{"reason":"regression"}'
# 5. JV ที่มี dimension: ตั้ง rule mandatory ให้ ACC_A ก่อน แล้วสร้าง JV โดยไม่ส่ง dimensions → ต้องได้ GL_DIMENSION_REQUIRED
curl -s -X POST http://localhost:4000/api/config/$BU/gl-account-dimension-rules $H -d '{"chart_of_accounts_id":"<ACC_A>","gl_dimension_id":"<DIM_MARKET>","requirement":"mandatory"}'
curl -s -X POST $API/gl-jv $H -d '{"prefix_id":"<PREFIX>","jv_date":"2026-09-23","details":{"add":[{"sequence_no":1,"chart_of_accounts_id":"<ACC_A>","debit":50},{"sequence_no":2,"chart_of_accounts_id":"<ACC_B>","credit":50}]}}'
# 6. ส่งพร้อม dimensions:[{gl_dimension_id, gl_dimension_value_id}] → สร้างได้ และ tb_gl_jv_detail_dimension มีแถว
```
Expected: ขั้น 1–4 ให้ผลเหมือนก่อน refactor (เทียบกับ `main` ถ้าจำเป็น), ขั้น 5 error catalog, ขั้น 6 มี junction row บันทึกผลลงข้อความ commit ถัดไปหรือ PR description

---

### Task 8: AP invoice — interface, serializer, logic, numbering, validation, workflow mapper

**Files:**
- Create: `apps/micro-business/src/ap/ap-invoice/interface/ap-invoice.interface.ts`
- Create: `apps/micro-business/src/ap/ap-invoice/dto/ap-invoice.serializer.ts`
- Create: `apps/micro-business/src/ap/ap-invoice/ap-invoice.logic.ts`
- Create: `apps/micro-business/src/ap/ap-invoice/ap-invoice.running-code.ts`
- Create: `apps/micro-business/src/ap/ap-invoice/ap-invoice.validation.ts`
- Create: `apps/micro-business/src/ap/ap-invoice/workflow/ap-invoice-workflow.mapper.ts`
- Modify: `apps/micro-business/src/common/dto/index.ts`

**Interfaces:**
- Consumes: `IDimensionRef`, `ISubledgerPostingLine`, `GlSetting`, Prisma AP models/enums
- Produces (ใช้โดย Task 9–10):
  - `IApInvoiceLineInput`, `ICreateApInvoice`, `IUpdateApInvoice`, `IApInvoiceReferenceInput`, `ICreateApInvoiceFromGrn`
  - `round2(d)`, `computeLine(input, ctx): IComputedLine`, `sumLines(lines)`, `computeDueDate(invoiceDate, days)`, `allocateAcrossLines(lines, amount)`, `buildInvoiceJvLines(ctx)`, `buildDepositReferenceJvLines(ctx)`
  - `generateApDocNo(params)`, `AP_DOC_TYPE_RUNNING_CODE`
  - `validateApHeaderRefs(db, data)`, `validateApLines(db, lines, docType)`: `Result<IResolvedApLineRefs>`
  - `apInvoiceToWorkflowDocument(inv)`
  - `ApInvoiceResponseSchema`, `ApInvoiceDetailResponseSchema`

- [ ] **Step 1: interface**

```ts
import { enum_ap_invoice_doc_type, enum_ap_invoice_source } from '@repo/prisma-shared-schema-tenant';
import { IDimensionRef } from '@/gl/gl-dimension/interface/gl-dimension.interface';

/** GRN item matched by one invoice line / รายการ GRN ที่บรรทัด invoice ผูก */
export interface IApInvoiceLineSourceInput {
  good_received_note_detail_item_id: string;
  matched_qty: number | string;
}

/** One invoice line as submitted / บรรทัด invoice ตามที่ส่งมา */
export interface IApInvoiceLineInput {
  sequence_no: number;
  description?: string | null;
  product_id?: string | null;
  product_name?: string | null;
  unit_id?: string | null;
  unit_name?: string | null;
  quantity: number | string;
  unit_price: number | string;
  discount_amount?: number | string | null;
  dr_chart_of_accounts_id: string;
  dr_cost_center_id?: string | null;
  vat_tax_profile_id?: string | null;
  vat_amount?: number | string | null;
  vat_is_override?: boolean;
  vat_chart_of_accounts_id?: string | null;
  vat_cost_center_id?: string | null;
  wht_tax_profile_id?: string | null;
  cr_chart_of_accounts_id?: string | null;
  cr_cost_center_id?: string | null;
  dimensions?: IDimensionRef[] | null;
  sources?: IApInvoiceLineSourceInput[] | null;
}

/** Offset against a posted DN/CN/DP / การหักกลบกับ DN/CN/DP ที่ post แล้ว */
export interface IApInvoiceReferenceInput {
  ref_ap_invoice_id: string;
  applied_amount: number | string;
}

/** Create payload / ข้อมูลสร้าง */
export interface ICreateApInvoice {
  doc_type: enum_ap_invoice_doc_type;
  doc_source?: enum_ap_invoice_source;
  doc_date: Date;
  vendor_id: string;
  vendor_invoice_no: string;
  invoice_date: Date;
  credit_term_days?: number | null;
  currency_id?: string | null;
  exchange_rate?: number | string | null;
  description?: string | null;
  attachments?: unknown[] | null;
  details: { add: IApInvoiceLineInput[] };
  references?: IApInvoiceReferenceInput[] | null;
}

/** Update payload; `details` and `references` replace the whole set when present / ข้อมูลแก้ไข ถ้าส่ง details/references จะแทนที่ทั้งชุด */
export interface IUpdateApInvoice {
  doc_version: number;
  doc_date?: Date;
  vendor_invoice_no?: string;
  invoice_date?: Date;
  credit_term_days?: number | null;
  exchange_rate?: number | string | null;
  description?: string | null;
  attachments?: unknown[] | null;
  details?: { add: IApInvoiceLineInput[] };
  references?: IApInvoiceReferenceInput[] | null;
}

/** Create-from-GRN payload / ข้อมูลสร้างจาก GRN */
export interface ICreateApInvoiceFromGrn {
  grn_item_ids: string[];
  doc_date: Date;
  invoice_date: Date;
  vendor_invoice_no: string;
  credit_term_days?: number | null;
  exchange_rate?: number | string | null;
  description?: string | null;
}

/** Action payload shared by approve/reject/review/void / payload ของ action */
export interface IApInvoiceAction {
  doc_version?: number;
  reason?: string | null;
  stage?: string | null;
}
```

- [ ] **Step 2: serializer** (`dto/ap-invoice.serializer.ts`)

```ts
import { z } from 'zod/v4';
import { enum_ap_invoice_doc_type, enum_ap_invoice_source, enum_ap_invoice_status, enum_ap_invoice_tax_status, enum_last_action } from '@repo/prisma-shared-schema-tenant';

const money = z.coerce.number();
const moneyOpt = z.coerce.number().nullable().optional();

/** Line response / การตอบกลับบรรทัด */
export const ApInvoiceDetailResponseSchema = z.object({
  id: z.string(), sequence_no: z.number(), description: z.string().nullable().optional(),
  product_id: z.string().nullable().optional(), product_name: z.string().nullable().optional(),
  unit_id: z.string().nullable().optional(), unit_name: z.string().nullable().optional(),
  quantity: money, unit_price: money, sub_total_amount: money, discount_amount: money, net_amount: money,
  dr_chart_of_accounts_id: z.string(), dr_account_code: z.string().nullable().optional(),
  dr_cost_center_id: z.string().nullable().optional(), dr_cost_center_code: z.string().nullable().optional(),
  vat_tax_profile_id: z.string().nullable().optional(), vat_rate: money, vat_amount: money, vat_is_override: z.boolean(),
  vat_chart_of_accounts_id: z.string().nullable().optional(), vat_cost_center_id: z.string().nullable().optional(),
  wht_tax_profile_id: z.string().nullable().optional(), wht_rate: moneyOpt, wht_estimate_amount: money,
  cr_chart_of_accounts_id: z.string(), cr_account_code: z.string().nullable().optional(), cr_cost_center_id: z.string().nullable().optional(),
  total_amount: money, unpaid_amount: money,
  base_sub_total_amount: money, base_discount_amount: money, base_net_amount: money, base_vat_amount: money, base_total_amount: money, base_unpaid_amount: money,
  tb_ap_invoice_detail_dimension: z.array(z.object({ gl_dimension_id: z.string(), gl_dimension_value_id: z.string(), dimension_code: z.string(), value_code: z.string() })).optional(),
  tb_ap_invoice_detail_source: z.array(z.object({ good_received_note_id: z.string(), grn_no: z.string(), good_received_note_detail_item_id: z.string(), matched_qty: money, grn_unit_price: money, matched_amount: money, variance_amount: money })).optional(),
});

/** Header response / การตอบกลับ header */
export const ApInvoiceResponseSchema = z.object({
  id: z.string(), doc_version: z.number().nullable().optional(),
  doc_no: z.string(), doc_type: z.nativeEnum(enum_ap_invoice_doc_type), doc_status: z.nativeEnum(enum_ap_invoice_status),
  doc_source: z.nativeEnum(enum_ap_invoice_source), doc_date: z.coerce.date(), description: z.string().nullable().optional(),
  vendor_id: z.string(), vendor_name: z.string(), vendor_invoice_no: z.string(), invoice_date: z.coerce.date(),
  credit_term_id: z.string().nullable().optional(), credit_term_name: z.string().nullable().optional(), credit_term_days: z.number(), due_date: z.coerce.date(),
  currency_id: z.string(), currency_code: z.string(), exchange_rate: money, base_currency_id: z.string(), base_currency_code: z.string(),
  sub_total_amount: money, discount_amount: money, net_amount: money, vat_amount: money, total_amount: money,
  base_sub_total_amount: money, base_discount_amount: money, base_net_amount: money, base_vat_amount: money, base_total_amount: money,
  wht_estimate_amount: money, outstanding_amount: money, reference_applied_amount: money,
  gl_jv_id: z.string().nullable().optional(), gl_jv_no: z.string().nullable().optional(), posted_at: z.coerce.date().nullable().optional(),
  void_at: z.coerce.date().nullable().optional(), void_reason: z.string().nullable().optional(), void_gl_jv_id: z.string().nullable().optional(),
  workflow_id: z.string().nullable().optional(), workflow_name: z.string().nullable().optional(), workflow_current_stage: z.string().nullable().optional(),
  workflow_previous_stage: z.string().nullable().optional(), workflow_next_stage: z.string().nullable().optional(),
  workflow_history: z.unknown().optional(), user_action: z.unknown().optional(),
  last_action: z.nativeEnum(enum_last_action).nullable().optional(), last_action_at_date: z.coerce.date().nullable().optional(),
  last_action_by_id: z.string().nullable().optional(), last_action_by_name: z.string().nullable().optional(),
  attachments: z.unknown().optional(),
  created_at: z.coerce.date().nullable().optional(), created_by_id: z.string().nullable().optional(),
  updated_at: z.coerce.date().nullable().optional(), updated_by_id: z.string().nullable().optional(),
  tb_ap_invoice_detail: z.array(ApInvoiceDetailResponseSchema).optional(),
  tb_ap_invoice_reference: z.array(z.object({ ref_ap_invoice_id: z.string(), ref_doc_type: z.nativeEnum(enum_ap_invoice_doc_type), applied_amount: money, realized_fx_amount: money })).optional(),
  tb_ap_invoice_tax: z.object({ tax_invoice_no: z.string(), tax_invoice_date: z.coerce.date(), base_amount: money, vat_amount: money, tax_status: z.nativeEnum(enum_ap_invoice_tax_status), expiry_claim_date: z.coerce.date() }).nullable().optional(),
});
```
เพิ่ม `export * from '@/ap/ap-invoice/dto/ap-invoice.serializer';` ใน `common/dto/index.ts`

- [ ] **Step 3: logic** (`ap-invoice.logic.ts`) — spec §6.4, §6.6, §8; Review Focus #1

```ts
import { enum_ap_invoice_doc_type, Prisma } from '@repo/prisma-shared-schema-tenant';
import { ISubledgerPostingLine } from '@/gl/gl-subledger-posting/interface/gl-subledger-posting.interface';
import { IDimensionRef } from '@/gl/gl-dimension/interface/gl-dimension.interface';

const ZERO = new Prisma.Decimal(0);
const HUNDRED = new Prisma.Decimal(100);

/**
 * Round half-up to two decimals — the one rounding rule for every AP amount (spec §6.4)
 * ปัดครึ่งขึ้นสองตำแหน่ง — กฎปัดเดียวของทุกยอด AP (spec §6.4)
 * @param d - Amount / ยอด
 * @returns Rounded amount / ยอดที่ปัดแล้ว
 */
export function round2(d: Prisma.Decimal): Prisma.Decimal {
  return d.toDecimalPlaces(2, Prisma.Decimal.ROUND_HALF_UP);
}

/** Inputs the line computation needs / ข้อมูลที่การคำนวณบรรทัดต้องใช้ */
export interface IComputeLineContext {
  exchange_rate: Prisma.Decimal;
  vat_rate: Prisma.Decimal;
  wht_rate: Prisma.Decimal | null;
  vat_override: Prisma.Decimal | null;
}

/** Every derived amount of a line / ยอดที่คำนวณได้ทั้งหมดของบรรทัด */
export interface IComputedLine {
  sub_total_amount: Prisma.Decimal;
  discount_amount: Prisma.Decimal;
  net_amount: Prisma.Decimal;
  vat_amount: Prisma.Decimal;
  wht_estimate_amount: Prisma.Decimal;
  total_amount: Prisma.Decimal;
  base_sub_total_amount: Prisma.Decimal;
  base_discount_amount: Prisma.Decimal;
  base_net_amount: Prisma.Decimal;
  base_vat_amount: Prisma.Decimal;
  base_total_amount: Prisma.Decimal;
}

/**
 * Compute one line; base total is the sum of rounded base parts so Dr and Cr agree to the satang
 * คำนวณหนึ่งบรรทัด base total = ผลรวมของส่วน base ที่ปัดแล้ว เพื่อให้ Dr กับ Cr ตรงกันถึงสตางค์
 * @param input - Quantity, price, discount / จำนวน ราคา ส่วนลด
 * @param input.quantity - Quantity / จำนวน
 * @param input.unit_price - Unit price / ราคาต่อหน่วย
 * @param input.discount_amount - Discount / ส่วนลด
 * @param ctx - Rates / อัตรา
 * @returns Computed amounts / ยอดที่คำนวณ
 */
export function computeLine(
  input: { quantity: Prisma.Decimal; unit_price: Prisma.Decimal; discount_amount: Prisma.Decimal },
  ctx: IComputeLineContext,
): IComputedLine {
  const sub_total_amount = round2(input.quantity.mul(input.unit_price));
  const discount_amount = round2(input.discount_amount);
  const net_amount = sub_total_amount.sub(discount_amount);
  const vat_amount = ctx.vat_override !== null ? round2(ctx.vat_override) : round2(net_amount.mul(ctx.vat_rate).div(HUNDRED));
  const wht_estimate_amount = ctx.wht_rate ? round2(net_amount.mul(ctx.wht_rate).div(HUNDRED)) : ZERO;
  const total_amount = net_amount.add(vat_amount);
  const base_sub_total_amount = round2(sub_total_amount.mul(ctx.exchange_rate));
  const base_discount_amount = round2(discount_amount.mul(ctx.exchange_rate));
  const base_net_amount = base_sub_total_amount.sub(base_discount_amount);
  const base_vat_amount = round2(vat_amount.mul(ctx.exchange_rate));
  return {
    sub_total_amount, discount_amount, net_amount, vat_amount, wht_estimate_amount, total_amount,
    base_sub_total_amount, base_discount_amount, base_net_amount, base_vat_amount,
    base_total_amount: base_net_amount.add(base_vat_amount),
  };
}

/** Header totals = Σ lines / ยอดรวม header = Σ บรรทัด */
export type IInvoiceTotals = Omit<IComputedLine, never>;

/**
 * Sum computed lines into header totals
 * รวมบรรทัดเป็นยอด header
 * @param lines - Computed lines / บรรทัดที่คำนวณแล้ว
 * @returns Totals / ยอดรวม
 */
export function sumLines(lines: IComputedLine[]): IInvoiceTotals {
  const keys = Object.keys(lines[0] ?? emptyLine()) as (keyof IComputedLine)[];
  const totals = emptyLine();
  for (const line of lines) {
    for (const key of keys) totals[key] = totals[key].add(line[key]);
  }
  return totals;
}

/** All-zero line / บรรทัดศูนย์ทั้งหมด @returns Zero line */
export function emptyLine(): IComputedLine {
  return {
    sub_total_amount: ZERO, discount_amount: ZERO, net_amount: ZERO, vat_amount: ZERO, wht_estimate_amount: ZERO, total_amount: ZERO,
    base_sub_total_amount: ZERO, base_discount_amount: ZERO, base_net_amount: ZERO, base_vat_amount: ZERO, base_total_amount: ZERO,
  };
}

/**
 * Due date = invoice date + credit term days, computed on the server every save (spec §6.3)
 * วันครบกำหนด = วันที่ใบแจ้งหนี้ + วันเครดิต คำนวณฝั่ง server ทุกครั้ง (spec §6.3)
 * @param invoiceDate - Invoice date / วันที่ใบแจ้งหนี้
 * @param days - Credit term days / จำนวนวันเครดิต
 * @returns Due date / วันครบกำหนด
 */
export function computeDueDate(invoiceDate: Date, days: number): Date {
  const due = new Date(invoiceDate);
  due.setUTCDate(due.getUTCDate() + Math.max(0, days));
  return due;
}

/**
 * Spread an amount over lines in sequence order, first line first, never past a line's unpaid
 * กระจายยอดลงบรรทัดตามลำดับ บรรทัดแรกก่อน ไม่เกินยอดค้างของแต่ละบรรทัด
 * @param lines - Lines with their unpaid amounts / บรรทัดพร้อมยอดค้าง
 * @param amount - Amount to spread / ยอดที่จะกระจาย
 * @returns Reduction per line id / ยอดที่ลดต่อ line id
 */
export function allocateAcrossLines(
  lines: { id: string; unpaid_amount: Prisma.Decimal }[],
  amount: Prisma.Decimal,
): Map<string, Prisma.Decimal> {
  const out = new Map<string, Prisma.Decimal>();
  let remaining = amount;
  for (const line of lines) {
    if (remaining.lte(0)) break;
    const take = Prisma.Decimal.min(remaining, line.unpaid_amount);
    if (take.gt(0)) {
      out.set(line.id, take);
      remaining = remaining.sub(take);
    }
  }
  return out;
}

/** The detail columns the JV builder reads / คอลัมน์บรรทัดที่ตัวสร้าง JV อ่าน */
export interface IJvSourceDetail {
  sequence_no: number;
  dr_chart_of_accounts_id: string;
  dr_cost_center_id: string | null;
  vat_chart_of_accounts_id: string | null;
  vat_cost_center_id: string | null;
  cr_chart_of_accounts_id: string;
  cr_cost_center_id: string | null;
  net_amount: Prisma.Decimal;
  vat_amount: Prisma.Decimal;
  total_amount: Prisma.Decimal;
  base_net_amount: Prisma.Decimal;
  base_vat_amount: Prisma.Decimal;
  base_total_amount: Prisma.Decimal;
  description: string | null;
  dimensions: IDimensionRef[];
}

/** What the JV builder needs from the header and settings / สิ่งที่ตัวสร้าง JV ต้องใช้จาก header และ setting */
export interface IJvBuildContext {
  doc_type: enum_ap_invoice_doc_type;
  currency_id: string;
  exchange_rate: Prisma.Decimal;
  base_currency_id: string;
  input_vat_account_id: string;
  advance_deposit_account_id: string;
}

/**
 * Build the ledger lines of one invoice-type document (spec §8 rows APIV/APDN/APCN/APDP)
 * สร้างบรรทัด ledger ของเอกสารประเภท invoice (spec §8 แถว APIV/APDN/APCN/APDP)
 * @param ctx - Header context / บริบท header
 * @param details - Invoice lines / บรรทัด invoice
 * @returns Facade lines, balanced in base / บรรทัดของ facade ที่สมดุลใน base
 */
export function buildInvoiceJvLines(ctx: IJvBuildContext, details: IJvSourceDetail[]): ISubledgerPostingLine[] {
  const isCredit = ctx.doc_type === enum_ap_invoice_doc_type.credit_note;
  const isDeposit = ctx.doc_type === enum_ap_invoice_doc_type.deposit;
  const out: ISubledgerPostingLine[] = [];
  let seq = 1;
  const push = (account: string, cc: string | null, amount: Prisma.Decimal, base: Prisma.Decimal, isDebit: boolean, description: string | null, dims: IDimensionRef[]) => {
    if (amount.isZero()) return;
    const debit = isDebit !== isCredit;
    out.push({
      sequence_no: seq++, chart_of_accounts_id: account, cost_center_id: cc,
      debit: debit ? amount : null, credit: debit ? null : amount,
      base_debit: debit ? base : new Prisma.Decimal(0), base_credit: debit ? new Prisma.Decimal(0) : base,
      currency_id: ctx.currency_id, exchange_rate: ctx.exchange_rate, description, dimensions: dims,
    });
  };
  for (const d of details) {
    const drAccount = isDeposit ? ctx.advance_deposit_account_id : d.dr_chart_of_accounts_id;
    push(drAccount, d.dr_cost_center_id, d.net_amount, d.base_net_amount, true, d.description, d.dimensions);
    if (!isDeposit) push(d.vat_chart_of_accounts_id ?? ctx.input_vat_account_id, d.vat_cost_center_id, d.vat_amount, d.base_vat_amount, true, d.description, []);
    push(d.cr_chart_of_accounts_id, d.cr_cost_center_id, isDeposit ? d.net_amount : d.total_amount, isDeposit ? d.base_net_amount : d.base_total_amount, false, d.description, []);
  }
  return out;
}

/** One deposit reference to book / การอ้างมัดจำหนึ่งรายการที่จะลงบัญชี */
export interface IDepositReferenceLine {
  applied_amount: Prisma.Decimal;
  base_applied_at_invoice_rate: Prisma.Decimal;
  base_applied_at_ref_rate: Prisma.Decimal;
}

/**
 * Ledger lines for deposits applied to an invoice, plus the realized FX line (spec §6.6, §8)
 * บรรทัด ledger ของมัดจำที่นำมาหักกับ invoice พร้อมบรรทัด realized FX (spec §6.6, §8)
 * @param ctx - Header context / บริบท header
 * @param apAccountId - AP control account of the invoice / บัญชี AP control ของ invoice
 * @param refs - Deposit references / การอ้างมัดจำ
 * @param fx - FX gain/loss accounts / บัญชีกำไรขาดทุนอัตราแลกเปลี่ยน
 * @param fx.gain - Gain account / บัญชีกำไร
 * @param fx.loss - Loss account / บัญชีขาดทุน
 * @param startSeq - First sequence number to use / เลขลำดับแรก
 * @returns Facade lines / บรรทัดของ facade
 */
export function buildDepositReferenceJvLines(
  ctx: IJvBuildContext,
  apAccountId: string,
  refs: IDepositReferenceLine[],
  fx: { gain: string; loss: string },
  startSeq: number,
): ISubledgerPostingLine[] {
  const out: ISubledgerPostingLine[] = [];
  let seq = startSeq;
  let fxDiff = new Prisma.Decimal(0);
  for (const r of refs) {
    out.push({ sequence_no: seq++, chart_of_accounts_id: apAccountId, debit: r.applied_amount, base_debit: r.base_applied_at_invoice_rate, base_credit: new Prisma.Decimal(0), currency_id: ctx.currency_id, exchange_rate: ctx.exchange_rate });
    out.push({ sequence_no: seq++, chart_of_accounts_id: ctx.advance_deposit_account_id, credit: r.applied_amount, base_debit: new Prisma.Decimal(0), base_credit: r.base_applied_at_ref_rate, currency_id: ctx.currency_id, exchange_rate: ctx.exchange_rate });
    fxDiff = fxDiff.add(r.base_applied_at_ref_rate.sub(r.base_applied_at_invoice_rate));
  }
  if (!fxDiff.isZero()) {
    const isLoss = fxDiff.gt(0);
    const abs = fxDiff.abs();
    out.push({
      sequence_no: seq++, chart_of_accounts_id: isLoss ? fx.loss : fx.gain,
      debit: isLoss ? abs : null, credit: isLoss ? null : abs,
      base_debit: isLoss ? abs : new Prisma.Decimal(0), base_credit: isLoss ? new Prisma.Decimal(0) : abs,
      currency_id: ctx.base_currency_id, exchange_rate: new Prisma.Decimal(1), description: 'Realized FX on deposit application',
    });
  }
  return out;
}
```

- [ ] **Step 4: numbering** (`ap-invoice.running-code.ts`) — pattern เดียวกับ `gl-jv.running-code.ts`

```ts
import { format } from 'date-fns';
import { enum_ap_invoice_doc_type, Prisma } from '@repo/prisma-shared-schema-tenant';
import { CommonLogic } from '@/common/common.logic';
import { getPattern } from '@/common/common.helper';

/** Running-code type and literal prefix per doc type (spec §6.8) / ประเภท running code และ prefix ต่อ doc type */
export const AP_DOC_TYPE_RUNNING_CODE: Record<enum_ap_invoice_doc_type, { type: string; prefix: string }> = {
  invoice: { type: 'AP-IV', prefix: 'APIV' },
  debit_note: { type: 'AP-DN', prefix: 'APDN' },
  credit_note: { type: 'AP-CN', prefix: 'APCN' },
  deposit: { type: 'AP-DP', prefix: 'APDP' },
};

/**
 * Next document number for a doc type, resetting per month
 * เลขเอกสารถัดไปของ doc type รีเซ็ตรายเดือน
 * @param params - Numbering inputs / ข้อมูลออกเลข
 * @param params.commonLogic - Running-code helper / ตัวช่วยออกเลข
 * @param params.prisma - Transaction client / client ของ transaction
 * @param params.userId - Caller / ผู้เรียก
 * @param params.buCode - Business unit / หน่วยธุรกิจ
 * @param params.docType - Document type / ประเภทเอกสาร
 * @param params.docDate - Date driving yyMM / วันที่ที่ใช้กับ yyMM
 * @returns Full document number e.g. APIV26090001 / เลขเอกสารเต็ม
 */
export async function generateApDocNo(params: {
  commonLogic: CommonLogic;
  prisma: Prisma.TransactionClient;
  userId: string;
  buCode: string;
  docType: enum_ap_invoice_doc_type;
  docDate: Date;
}): Promise<string> {
  const { commonLogic, prisma, userId, buCode, docType, docDate } = params;
  const { type, prefix } = AP_DOC_TYPE_RUNNING_CODE[docType];
  const pattern = await commonLogic.getRunningPattern(type, userId, buCode);
  const parts = getPattern(pattern);
  const datePattern = parts.find((p) => p.type === 'date');
  const runningPattern = parts.find((p) => p.type === 'running');
  if (!datePattern || !runningPattern) {
    throw new Error(`Missing running code pattern config for ${type}`);
  }
  const datePart = format(docDate, datePattern.pattern);
  const width = Number(runningPattern.pattern);
  const latest = await prisma.tb_ap_invoice.findFirst({
    where: { doc_no: { startsWith: `${prefix}${datePart}` }, deleted_at: null },
    orderBy: { doc_no: 'desc' },
    select: { doc_no: true },
  });
  const lastNo = latest ? Number(latest.doc_no.slice(-width)) : 0;
  return commonLogic.generateRunningCode(type, docDate, lastNo, userId, buCode);
}
```

- [ ] **Step 5: validation** (`ap-invoice.validation.ts`) — spec §6.3, §6.4

```ts
import {
  enum_chart_of_accounts_type, enum_tax_profile_tax_type, Prisma,
  type PrismaClient, type tb_chart_of_accounts, type tb_tax_profile,
} from '@repo/prisma-shared-schema-tenant';
import { Result } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import { IApInvoiceLineInput } from './interface/ap-invoice.interface';

type ApValidationDb = Pick<PrismaClient, 'tb_chart_of_accounts' | 'tb_cost_center' | 'tb_tax_profile' | 'tb_vendor'>;

/** Master rows the line writer needs, keyed by id / แถว master ที่ตัวเขียนบรรทัดต้องใช้ */
export interface IResolvedApLineRefs {
  accounts: Map<string, tb_chart_of_accounts>;
  taxProfiles: Map<string, tb_tax_profile>;
}

const NON_POSTABLE: enum_chart_of_accounts_type[] = [enum_chart_of_accounts_type.header, enum_chart_of_accounts_type.summary];

/**
 * Vendor must exist and be active (spec §6.3)
 * vendor ต้องมีและ active (spec §6.3)
 * @param db - Prisma client / client
 * @param vendorId - Vendor id / รหัส vendor
 * @returns The vendor row / แถว vendor
 */
export async function validateApVendor(db: ApValidationDb, vendorId: string) {
  const vendor = await db.tb_vendor.findFirst({ where: { id: vendorId, deleted_at: null, is_active: true } });
  if (!vendor) return Result.errorFromCatalog(ERROR_CATALOG.AP_VENDOR_NOT_FOUND);
  return Result.ok(vendor);
}

/**
 * Validate line amounts, accounts, cost centers and tax profiles; resolve master rows
 * ตรวจยอด บัญชี cost center และ tax profile ของบรรทัด แล้ว resolve master
 * @param db - Prisma client / client
 * @param lines - Submitted lines / บรรทัดที่ส่งมา
 * @param isDeposit - Deposit lines may not carry VAT / บรรทัดมัดจำห้ามมี VAT
 * @returns Resolved refs or the first violation / master ที่ resolve หรือข้อผิดพลาดแรก
 */
export async function validateApLines(
  db: ApValidationDb,
  lines: IApInvoiceLineInput[],
  isDeposit: boolean,
): Promise<Result<IResolvedApLineRefs>> {
  if (lines.length === 0) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_LINE_INVALID, { sequence_no: 0, reason: 'at least one line is required' });
  for (const l of lines) {
    const amountCheck = checkAmounts(l, isDeposit);
    if (amountCheck.isError()) return Result.error(amountCheck.error);
  }
  const accountIds = [...new Set(lines.flatMap((l) => [l.dr_chart_of_accounts_id, l.cr_chart_of_accounts_id, l.vat_chart_of_accounts_id].filter((x): x is string => Boolean(x))))];
  const accounts = new Map((await db.tb_chart_of_accounts.findMany({ where: { id: { in: accountIds }, deleted_at: null } })).map((a) => [a.id, a]));
  const profileIds = [...new Set(lines.flatMap((l) => [l.vat_tax_profile_id, l.wht_tax_profile_id].filter((x): x is string => Boolean(x))))];
  const taxProfiles = new Map((await db.tb_tax_profile.findMany({ where: { id: { in: profileIds }, deleted_at: null, is_active: true } })).map((t) => [t.id, t]));
  for (const l of lines) {
    const refCheck = checkRefs(l, accounts, taxProfiles);
    if (refCheck.isError()) return Result.error(refCheck.error);
  }
  const ccIds = [...new Set(lines.flatMap((l) => [l.dr_cost_center_id, l.cr_cost_center_id, l.vat_cost_center_id].filter((x): x is string => Boolean(x))))];
  if (ccIds.length > 0) {
    const active = await db.tb_cost_center.count({ where: { id: { in: ccIds }, is_active: true, deleted_at: null } });
    if (active !== ccIds.length) return Result.errorFromCatalog(ERROR_CATALOG.GL_COST_CENTER_NOT_ALLOWED);
  }
  return Result.ok({ accounts, taxProfiles });
}

/**
 * quantity > 0, unit_price ≥ 0, discount ≤ subtotal, no VAT on deposits
 * quantity > 0, unit_price ≥ 0, ส่วนลด ≤ ยอดก่อนลด, มัดจำไม่มี VAT
 * @param l - Line / บรรทัด
 * @param isDeposit - Deposit flag / เป็นมัดจำ
 * @returns ok or violation / ok หรือข้อผิดพลาด
 */
function checkAmounts(l: IApInvoiceLineInput, isDeposit: boolean): Result<true> {
  const qty = new Prisma.Decimal(l.quantity);
  const price = new Prisma.Decimal(l.unit_price);
  const discount = new Prisma.Decimal(l.discount_amount ?? 0);
  const fail = (reason: string) => Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_LINE_INVALID, { sequence_no: l.sequence_no, reason });
  if (qty.lte(0)) return fail('quantity must be > 0');
  if (price.lt(0)) return fail('unit_price must be >= 0');
  if (discount.lt(0) || discount.gt(qty.mul(price))) return fail('discount must be between 0 and subtotal');
  if (isDeposit && l.vat_tax_profile_id) return fail('deposit lines may not carry VAT');
  if (l.vat_is_override && (l.vat_amount === null || l.vat_amount === undefined)) return fail('vat_amount is required when vat_is_override');
  return Result.ok(true);
}

/**
 * Accounts postable + cost-center rule, tax profiles of the right type
 * บัญชีลงได้ + กฎ cost center, tax profile ประเภทถูกต้อง
 * @param l - Line / บรรทัด
 * @param accounts - Resolved accounts / บัญชีที่ resolve
 * @param profiles - Resolved tax profiles / tax profile ที่ resolve
 * @returns ok or violation / ok หรือข้อผิดพลาด
 */
function checkRefs(l: IApInvoiceLineInput, accounts: Map<string, tb_chart_of_accounts>, profiles: Map<string, tb_tax_profile>): Result<true> {
  for (const [accountId, cc] of [[l.dr_chart_of_accounts_id, l.dr_cost_center_id], [l.cr_chart_of_accounts_id, l.cr_cost_center_id], [l.vat_chart_of_accounts_id, l.vat_cost_center_id]] as const) {
    if (!accountId) continue;
    const a = accounts.get(accountId);
    if (!a || !a.is_active || NON_POSTABLE.includes(a.type)) return Result.errorFromCatalog(ERROR_CATALOG.AP_ACCOUNT_NOT_POSTABLE, { account_id: accountId });
    if (a.is_require_cost_center && !cc) return Result.errorFromCatalog(ERROR_CATALOG.AP_COST_CENTER_REQUIRED, { account_code: a.code });
  }
  if (l.vat_tax_profile_id && profiles.get(l.vat_tax_profile_id)?.tax_type !== enum_tax_profile_tax_type.vat) {
    return Result.errorFromCatalog(ERROR_CATALOG.AP_TAX_PROFILE_INVALID, { tax_profile_id: l.vat_tax_profile_id });
  }
  if (l.wht_tax_profile_id && profiles.get(l.wht_tax_profile_id)?.tax_type !== enum_tax_profile_tax_type.wht) {
    return Result.errorFromCatalog(ERROR_CATALOG.AP_TAX_PROFILE_INVALID, { tax_profile_id: l.wht_tax_profile_id });
  }
  return Result.ok(true);
}
```

- [ ] **Step 6: workflow mapper** (`workflow/ap-invoice-workflow.mapper.ts`)

```ts
import { WorkflowDocument, WorkflowHistory } from '@/common/workflow/workflow.interfaces';

/**
 * Map an AP invoice header to the orchestrator's WorkflowDocument (amount routing uses base total)
 * แมป header ของ AP invoice เป็น WorkflowDocument ของ orchestrator (เดินขั้นตามยอด base)
 * @param inv - Header slice / ส่วนของ header
 * @returns WorkflowDocument / เอกสาร workflow
 */
export function apInvoiceToWorkflowDocument(inv: {
  id: string;
  workflow_id: string | null;
  workflow_current_stage: string | null;
  workflow_previous_stage: string | null;
  workflow_history?: unknown;
  created_by_id?: string | null;
  base_total_amount?: unknown;
}): WorkflowDocument {
  if (!inv.workflow_id) {
    throw new Error(`apInvoiceToWorkflowDocument: invoice ${inv.id} has no workflow_id`);
  }
  const history = Array.isArray(inv.workflow_history) ? (inv.workflow_history as WorkflowHistory[]) : [];
  return {
    id: inv.id,
    workflow_id: inv.workflow_id,
    workflow_current_stage: inv.workflow_current_stage,
    workflow_previous_stage: inv.workflow_previous_stage,
    workflow_history: history,
    requestor_id: inv.created_by_id ?? null,
    department: null,
    navigation_request_data: { total_amount: Number(inv.base_total_amount ?? 0), department: null },
  };
}
```

- [ ] **Step 7: check + commit**

```bash
bun run check-types && npx eslint --no-fix apps/micro-business/src/ap/ap-invoice
git add apps/micro-business/src/ap/ap-invoice apps/micro-business/src/common/dto/index.ts
git commit -m "feat(ap): add AP invoice interfaces, calculation logic, numbering, validation and workflow mapper

Spec §6.3–6.8, §8"
```

---

### Task 9: AP invoice service — CRUD, GRN candidates/create-from-GRN, reference candidates

**Files:**
- Create: `apps/micro-business/src/ap/ap-invoice/ap-invoice.service.ts` (ส่วน CRUD; Task 10 เพิ่ม action)
- Create: `apps/micro-business/src/ap/ap-invoice/ap-invoice.writer.ts` (เขียน header+lines ใน tx)

**Interfaces:**
- Consumes: Task 8 ทั้งหมด, `ExchangeRateService.findByDateAndCurrency(date: string, code: string)`, `CommonLogic`
- Produces: `ApInvoiceService.{findOne, findAll, create, update, delete, grnCandidates, createFromGrn, referenceCandidates}`; `ApInvoiceWriter.writeDocument(tx, ctx)`; `ApInvoiceService.loadFull(db, id)`

- [ ] **Step 1: writer** (`ap-invoice.writer.ts`) — เขียน header + lines + dimensions + sources + references ใน tx เดียว ใช้ทั้ง create/update/createFromGrn

```ts
import { Injectable } from '@nestjs/common';
import { enum_ap_invoice_doc_type, enum_ap_invoice_source, enum_ap_invoice_status, Prisma } from '@repo/prisma-shared-schema-tenant';
import { Result } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import { validateLineDimensions } from '@/gl/gl-dimension/gl-dimension.validator';
import { IResolvedDimensionValue } from '@/gl/gl-dimension/interface/gl-dimension.interface';
import { IApInvoiceLineInput, IApInvoiceReferenceInput } from './interface/ap-invoice.interface';
import { computeDueDate, computeLine, IComputedLine, sumLines } from './ap-invoice.logic';
import { IResolvedApLineRefs } from './ap-invoice.validation';

/** Header values already resolved by the service / ค่า header ที่ service resolve แล้ว */
export interface IWriteHeaderContext {
  id: string | null;
  doc_no: string | null;
  doc_type: enum_ap_invoice_doc_type;
  doc_source: enum_ap_invoice_source;
  doc_date: Date;
  description: string | null;
  vendor: { id: string; name: string; credit_term_id: string | null; credit_term_name: string | null; ap_chart_of_accounts_id: string | null };
  vendor_invoice_no: string;
  invoice_date: Date;
  credit_term_days: number;
  currency: { id: string; code: string };
  exchange_rate: Prisma.Decimal;
  base_currency: { id: string; code: string };
  default_cr_account_id: string;
  attachments: unknown[] | null;
  lines: IApInvoiceLineInput[];
  refs: IResolvedApLineRefs;
  references: IApInvoiceReferenceInput[];
  userId: string;
}

/**
 * Single write path for an AP document's header, lines, dimensions, GRN sources and references
 * ทางเดียวที่เขียน header บรรทัด dimension แหล่ง GRN และการอ้างอิงของเอกสาร AP
 */
@Injectable()
export class ApInvoiceWriter {
  /**
   * Create (id null) or replace (id set) the document inside the caller's transaction
   * สร้าง (id เป็น null) หรือแทนที่ (มี id) เอกสารภายใน transaction ของผู้เรียก
   * @param tx - Transaction client / client ของ transaction
   * @param ctx - Resolved header + lines / header และบรรทัดที่ resolve แล้ว
   * @returns New/updated id / id ที่สร้างหรือแก้ไข
   */
  async writeDocument(tx: Prisma.TransactionClient, ctx: IWriteHeaderContext): Promise<Result<{ id: string }>> {
    const dims = await validateLineDimensions(tx, ctx.lines.map((l) => ({ sequence_no: l.sequence_no, chart_of_accounts_id: l.dr_chart_of_accounts_id, dimensions: l.dimensions })), ctx.doc_date);
    if (dims.isError()) return Result.error(dims.error);
    const computed = ctx.lines.map((l) => this.computeForLine(l, ctx));
    const totals = sumLines(computed);
    const headerData = this.headerData(ctx, totals);
    let id = ctx.id;
    if (id) {
      await tx.tb_ap_invoice_detail.updateMany({ where: { ap_invoice_id: id, deleted_at: null }, data: { deleted_at: new Date(), deleted_by_id: ctx.userId } });
      await tx.tb_ap_invoice_reference.deleteMany({ where: { ap_invoice_id: id } });
      await tx.tb_ap_invoice.update({ where: { id }, data: { ...headerData, doc_version: { increment: 1 }, updated_at: new Date(), updated_by_id: ctx.userId } });
    } else {
      const created = await tx.tb_ap_invoice.create({ data: { ...headerData, doc_no: ctx.doc_no as string, doc_type: ctx.doc_type, doc_source: ctx.doc_source, doc_status: enum_ap_invoice_status.draft, created_by_id: ctx.userId, updated_by_id: ctx.userId }, select: { id: true } });
      id = created.id;
    }
    await this.writeLines(tx, id, ctx, computed, dims.value);
    if (ctx.references.length > 0) {
      await tx.tb_ap_invoice_reference.createMany({ data: await this.referenceRows(tx, id, ctx) });
    }
    return Result.ok({ id });
  }

  /** Rates for one line from its tax profiles / อัตราของบรรทัดจาก tax profile @param l Line @param ctx Context @returns Computed */
  private computeForLine(l: IApInvoiceLineInput, ctx: IWriteHeaderContext): IComputedLine {
    const vat = l.vat_tax_profile_id ? ctx.refs.taxProfiles.get(l.vat_tax_profile_id) : null;
    const wht = l.wht_tax_profile_id ? ctx.refs.taxProfiles.get(l.wht_tax_profile_id) : null;
    return computeLine(
      { quantity: new Prisma.Decimal(l.quantity), unit_price: new Prisma.Decimal(l.unit_price), discount_amount: new Prisma.Decimal(l.discount_amount ?? 0) },
      { exchange_rate: ctx.exchange_rate, vat_rate: new Prisma.Decimal(vat?.tax_rate ?? 0), wht_rate: wht ? new Prisma.Decimal(wht.tax_rate ?? 0) : null, vat_override: l.vat_is_override ? new Prisma.Decimal(l.vat_amount ?? 0) : null },
    );
  }

  /** Header columns / คอลัมน์ header @param ctx Context @param totals Totals @returns Prisma data */
  private headerData(ctx: IWriteHeaderContext, totals: IComputedLine) {
    return {
      doc_date: ctx.doc_date, description: ctx.description,
      vendor_id: ctx.vendor.id, vendor_name: ctx.vendor.name, vendor_invoice_no: ctx.vendor_invoice_no.trim(), invoice_date: ctx.invoice_date,
      credit_term_id: ctx.vendor.credit_term_id, credit_term_name: ctx.vendor.credit_term_name, credit_term_days: ctx.credit_term_days,
      due_date: computeDueDate(ctx.invoice_date, ctx.credit_term_days),
      currency_id: ctx.currency.id, currency_code: ctx.currency.code, exchange_rate: ctx.exchange_rate,
      base_currency_id: ctx.base_currency.id, base_currency_code: ctx.base_currency.code,
      sub_total_amount: totals.sub_total_amount, discount_amount: totals.discount_amount, net_amount: totals.net_amount, vat_amount: totals.vat_amount, total_amount: totals.total_amount,
      base_sub_total_amount: totals.base_sub_total_amount, base_discount_amount: totals.base_discount_amount, base_net_amount: totals.base_net_amount, base_vat_amount: totals.base_vat_amount, base_total_amount: totals.base_total_amount,
      wht_estimate_amount: totals.wht_estimate_amount, outstanding_amount: totals.total_amount, reference_applied_amount: new Prisma.Decimal(0),
      attachments: (ctx.attachments ?? []) as Prisma.InputJsonValue,
    };
  }

  /** Lines + dimensions + GRN sources / บรรทัด + dimension + แหล่ง GRN */
  private async writeLines(tx: Prisma.TransactionClient, invoiceId: string, ctx: IWriteHeaderContext, computed: IComputedLine[], dims: Map<string, IResolvedDimensionValue>): Promise<void> {
    for (const [i, l] of ctx.lines.entries()) {
      const c = computed[i];
      const dr = ctx.refs.accounts.get(l.dr_chart_of_accounts_id);
      const crId = l.cr_chart_of_accounts_id ?? ctx.default_cr_account_id;
      const wht = l.wht_tax_profile_id ? ctx.refs.taxProfiles.get(l.wht_tax_profile_id) : null;
      const detail = await tx.tb_ap_invoice_detail.create({
        data: {
          ap_invoice_id: invoiceId, sequence_no: l.sequence_no, description: l.description ?? null,
          product_id: l.product_id ?? null, product_name: l.product_name ?? null, unit_id: l.unit_id ?? null, unit_name: l.unit_name ?? null,
          quantity: new Prisma.Decimal(l.quantity), unit_price: new Prisma.Decimal(l.unit_price),
          sub_total_amount: c.sub_total_amount, discount_amount: c.discount_amount, net_amount: c.net_amount,
          dr_chart_of_accounts_id: l.dr_chart_of_accounts_id, dr_account_code: dr?.code ?? null, dr_cost_center_id: l.dr_cost_center_id ?? null,
          vat_tax_profile_id: l.vat_tax_profile_id ?? null, vat_rate: new Prisma.Decimal(l.vat_tax_profile_id ? (ctx.refs.taxProfiles.get(l.vat_tax_profile_id)?.tax_rate ?? 0) : 0),
          vat_amount: c.vat_amount, vat_is_override: l.vat_is_override ?? false, vat_chart_of_accounts_id: l.vat_chart_of_accounts_id ?? null, vat_cost_center_id: l.vat_cost_center_id ?? null,
          wht_tax_profile_id: l.wht_tax_profile_id ?? null, wht_rate: wht ? new Prisma.Decimal(wht.tax_rate ?? 0) : null, wht_estimate_amount: c.wht_estimate_amount,
          cr_chart_of_accounts_id: crId, cr_account_code: ctx.refs.accounts.get(crId)?.code ?? null, cr_cost_center_id: l.cr_cost_center_id ?? null,
          total_amount: c.total_amount, unpaid_amount: c.total_amount,
          base_sub_total_amount: c.base_sub_total_amount, base_discount_amount: c.base_discount_amount, base_net_amount: c.base_net_amount,
          base_vat_amount: c.base_vat_amount, base_total_amount: c.base_total_amount, base_unpaid_amount: c.base_total_amount,
          created_by_id: ctx.userId, updated_by_id: ctx.userId,
        },
        select: { id: true },
      });
      const dimRows = (l.dimensions ?? []).map((d) => ({ ap_invoice_detail_id: detail.id, gl_dimension_id: d.gl_dimension_id, gl_dimension_value_id: d.gl_dimension_value_id, dimension_code: dims.get(d.gl_dimension_value_id)?.dimension_code ?? '', value_code: dims.get(d.gl_dimension_value_id)?.value_code ?? '', created_by_id: ctx.userId }));
      if (dimRows.length) await tx.tb_ap_invoice_detail_dimension.createMany({ data: dimRows });
      await this.writeSources(tx, detail.id, l, c, ctx.userId);
    }
  }

  /** GRN source rows with variance snapshot (spec §6.5) / แถวแหล่ง GRN พร้อม variance */
  private async writeSources(tx: Prisma.TransactionClient, detailId: string, l: IApInvoiceLineInput, c: IComputedLine, userId: string): Promise<void> {
    if (!l.sources?.length) return;
    const items = await tx.tb_good_received_note_detail_item.findMany({
      where: { id: { in: l.sources.map((s) => s.good_received_note_detail_item_id) } },
      include: { tb_good_received_note_detail: { include: { tb_good_received_note: { select: { id: true, grn_no: true } } } } },
    });
    const rows = l.sources.map((s) => {
      const item = items.find((i) => i.id === s.good_received_note_detail_item_id);
      const grn = item?.tb_good_received_note_detail.tb_good_received_note;
      const qty = new Prisma.Decimal(s.matched_qty);
      const price = new Prisma.Decimal(item?.received_price ?? 0);
      return { ap_invoice_detail_id: detailId, good_received_note_id: grn?.id ?? '', grn_no: grn?.grn_no ?? '', good_received_note_detail_item_id: s.good_received_note_detail_item_id, matched_qty: qty, grn_unit_price: price, matched_amount: c.net_amount, variance_amount: c.net_amount.sub(qty.mul(price)), created_by_id: userId };
    });
    await tx.tb_ap_invoice_detail_source.createMany({ data: rows });
  }

  /** Reference rows with both base amounts (FX booked at post) / แถวอ้างอิงพร้อม base ทั้งสองอัตรา */
  private async referenceRows(tx: Prisma.TransactionClient, invoiceId: string, ctx: IWriteHeaderContext) {
    const refs = await tx.tb_ap_invoice.findMany({ where: { id: { in: ctx.references.map((r) => r.ref_ap_invoice_id) } }, select: { id: true, doc_type: true, exchange_rate: true } });
    return ctx.references.map((r) => {
      const ref = refs.find((x) => x.id === r.ref_ap_invoice_id);
      const applied = new Prisma.Decimal(r.applied_amount);
      const atInvoice = applied.mul(ctx.exchange_rate).toDecimalPlaces(2, Prisma.Decimal.ROUND_HALF_UP);
      const atRef = applied.mul(ref?.exchange_rate ?? 1).toDecimalPlaces(2, Prisma.Decimal.ROUND_HALF_UP);
      return { ap_invoice_id: invoiceId, ref_ap_invoice_id: r.ref_ap_invoice_id, ref_doc_type: ref?.doc_type ?? enum_ap_invoice_doc_type.credit_note, applied_amount: applied, base_applied_at_invoice_rate: atInvoice, base_applied_at_ref_rate: atRef, realized_fx_amount: atRef.sub(atInvoice), created_by_id: ctx.userId };
    });
  }
}
```
(ถ้า `ERROR_CATALOG` ไม่ถูกใช้ในไฟล์ให้ลบ import)

- [ ] **Step 2: service ส่วน CRUD** (`ap-invoice.service.ts`)

```ts
import { Injectable } from '@nestjs/common';
import { TryCatch, Result, ApInvoiceResponseSchema } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import {
  enum_ap_invoice_doc_type, enum_ap_invoice_source, enum_ap_invoice_status, enum_good_received_note_status, enum_workflow_type, Prisma,
} from '@repo/prisma-shared-schema-tenant';
import { TenantScopedService } from '@/common/tenant-scoped.service';
import { BackendLogger } from '@/common/helpers/backend.logger';
import { CommonLogic } from '@/common/common.logic';
import QueryParams from '@/common/libs/paginate.query';
import { withDefaultSort } from '@/common/libs/default-sort';
import getPaginationParams from '@/common/helpers/pagination.params';
import { IPaginate } from '@/common/shared-interface/paginate.interface';
import { isUniqueConstraintViolation } from '@/common/helpers/is-unique-constraint-violation';
import { WorkflowOrchestratorService } from '@/common/workflow/workflow-orchestrator.service';
import { ExchangeRateService } from '@/master/exchange-rate/exchange-rate.service';
import { GlSubledgerPostingService } from '@/gl/gl-subledger-posting/gl-subledger-posting.service';
import { GlPostingService } from '@/gl/gl-posting/gl-posting.service';
import { ApInvoiceWriter, IWriteHeaderContext } from './ap-invoice.writer';
import { generateApDocNo } from './ap-invoice.running-code';
import { validateApLines, validateApVendor } from './ap-invoice.validation';
import { ICreateApInvoice, ICreateApInvoiceFromGrn, IUpdateApInvoice } from './interface/ap-invoice.interface';

/** Full document shape used by actions / รูปเอกสารเต็มที่ action ใช้ */
export const AP_INVOICE_FULL_INCLUDE = {
  tb_ap_invoice_detail: { where: { deleted_at: null }, orderBy: { sequence_no: 'asc' as const }, include: { tb_ap_invoice_detail_dimension: true, tb_ap_invoice_detail_source: true } },
  tb_ap_invoice_reference: true,
  tb_ap_invoice_tax: true,
};
export type ApInvoiceFull = Prisma.tb_ap_invoiceGetPayload<{ include: typeof AP_INVOICE_FULL_INCLUDE }>;

/**
 * AP invoice / debit note / credit note / deposit (spec §6)
 * เอกสาร AP ทั้ง 4 ประเภท (spec §6)
 */
@Injectable()
export class ApInvoiceService extends TenantScopedService {
  private readonly logger: BackendLogger = new BackendLogger(ApInvoiceService.name);

  constructor(
    private readonly commonLogic: CommonLogic,
    private readonly writer: ApInvoiceWriter,
    private readonly exchangeRates: ExchangeRateService,
    private readonly workflowOrchestrator: WorkflowOrchestratorService,
    private readonly subledger: GlSubledgerPostingService,
    private readonly glPosting: GlPostingService,
  ) {
    super();
  }

  /** Load a document with everything actions need / โหลดเอกสารพร้อมทุกอย่างที่ action ต้องใช้ @param db Client @param id Id @returns Row or null */
  async loadFull(db: Pick<Prisma.TransactionClient, 'tb_ap_invoice'>, id: string): Promise<ApInvoiceFull | null> {
    return db.tb_ap_invoice.findFirst({ where: { id, deleted_at: null }, include: AP_INVOICE_FULL_INCLUDE });
  }

  /** Find one / ค้นหาหนึ่งใบ @param id Id @returns Document or NOT_FOUND */
  @TryCatch
  async findOne(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findOne', id }, ApInvoiceService.name);
    const row = await this.loadFull(this.prismaService, id);
    if (!row) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_NOT_FOUND);
    return Result.ok(ApInvoiceResponseSchema.parse(row));
  }

  /** Paginated list; filter by doc_type/doc_status/vendor_id / รายการแบบแบ่งหน้า @param paginate Params @returns Page */
  @TryCatch
  async findAll(paginate: IPaginate): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findAll', paginate }, ApInvoiceService.name);
    const q = new QueryParams(
      paginate.page, paginate.perpage, paginate.search, paginate.searchfields,
      ['doc_no', 'vendor_name', 'vendor_invoice_no', 'description'],
      typeof paginate.filter === 'object' && !Array.isArray(paginate.filter) ? paginate.filter : {},
      withDefaultSort(paginate.sort, ['doc_date:desc', 'doc_no:desc']),
      paginate.advance,
    );
    const pagination = getPaginationParams(q.page, q.perpage);
    const rows = await this.prismaService.tb_ap_invoice.findMany({ where: q.where(), orderBy: q.orderBy(), ...pagination });
    const total = await this.prismaService.tb_ap_invoice.count({ where: q.where() });
    return Result.ok({
      paginate: { total, page: q.perpage < 0 ? 1 : q.page, perpage: q.perpage < 0 ? 1 : q.perpage, pages: total === 0 || q.perpage < 0 ? 1 : Math.ceil(total / q.perpage) },
      data: rows.map((r) => ApInvoiceResponseSchema.parse(r)),
    });
  }

  /**
   * Base currency of the business unit (platform schema) — same source the exchange-rate service uses
   * สกุลเงินฐานของหน่วยธุรกิจ (platform schema) — แหล่งเดียวกับ exchange-rate service
   * @returns `{ id, code }` / รหัสและโค้ด
   */
  private async resolveBaseCurrency(): Promise<Result<{ id: string; code: string }>> {
    const bu = await this.prismaSystem.tb_business_unit.findFirst({ where: { code: this.bu_code }, select: { default_currency_id: true } });
    const currency = bu?.default_currency_id ? await this.prismaService.tb_currency.findFirst({ where: { id: bu.default_currency_id }, select: { id: true, code: true } }) : null;
    if (!currency) return Result.errorFromCatalog(ERROR_CATALOG.GL_SETTING_ACCOUNT_MISSING, { key: 'business_unit.default_currency_id' });
    return Result.ok(currency);
  }

  /**
   * Resolve everything the writer needs from a create/update payload (spec §6.3 defaults)
   * resolve ทุกอย่างที่ writer ต้องใช้จาก payload (default ตาม spec §6.3)
   * @param data - Payload / ข้อมูล
   * @param existing - Current row on update, or null / แถวเดิมตอนแก้ไข หรือ null
   * @returns Writer context / บริบทของ writer
   */
  private async buildContext(data: ICreateApInvoice, existing: ApInvoiceFull | null): Promise<Result<IWriteHeaderContext>> {
    const vendorRes = await validateApVendor(this.prismaService, data.vendor_id);
    if (vendorRes.isError()) return Result.error(vendorRes.error);
    const vendor = vendorRes.value;
    const baseRes = await this.resolveBaseCurrency();
    if (baseRes.isError()) return Result.error(baseRes.error);
    const currencyId = data.currency_id ?? vendor.default_currency_id ?? baseRes.value.id;
    const currency = await this.prismaService.tb_currency.findFirst({ where: { id: currencyId, deleted_at: null }, select: { id: true, code: true } });
    if (!currency) return Result.errorFromCatalog(ERROR_CATALOG.AP_VENDOR_CURRENCY_MISMATCH);
    const rateRes = await this.resolveRate(data.exchange_rate, data.doc_date, currency.code);
    if (rateRes.isError()) return Result.error(rateRes.error);
    const setting = await this.glPosting.getSetting();
    const crAccount = vendor.ap_chart_of_accounts_id ?? setting?.ap_control_account_id;
    if (!crAccount) return Result.errorFromCatalog(ERROR_CATALOG.GL_SETTING_ACCOUNT_MISSING, { key: 'ap_control_account_id' });
    const refs = await validateApLines(this.prismaService, data.details.add, data.doc_type === enum_ap_invoice_doc_type.deposit);
    if (refs.isError()) return Result.error(refs.error);
    const refCheck = await this.validateReferences(data, vendor.id, currency.id);
    if (refCheck.isError()) return Result.error(refCheck.error);
    return Result.ok({
      id: existing?.id ?? null, doc_no: existing?.doc_no ?? null, doc_type: data.doc_type, doc_source: data.doc_source ?? enum_ap_invoice_source.manual,
      doc_date: data.doc_date, description: data.description ?? null,
      vendor: { id: vendor.id, name: vendor.name, credit_term_id: vendor.credit_term_id, credit_term_name: vendor.credit_term_name, ap_chart_of_accounts_id: vendor.ap_chart_of_accounts_id },
      vendor_invoice_no: data.vendor_invoice_no, invoice_date: data.invoice_date, credit_term_days: data.credit_term_days ?? vendor.credit_term_days ?? 0,
      currency, exchange_rate: rateRes.value, base_currency: baseRes.value, default_cr_account_id: crAccount,
      attachments: data.attachments ?? null, lines: data.details.add, refs: refs.value, references: data.references ?? [], userId: this.userId,
    });
  }

  /** Header rate: explicit > 0, else resolved by date / อัตรา header: ที่ส่งมา > 0 ไม่งั้นหาจากวันที่ */
  private async resolveRate(explicit: number | string | null | undefined, docDate: Date, currencyCode: string): Promise<Result<Prisma.Decimal>> {
    if (explicit !== null && explicit !== undefined) {
      const d = new Prisma.Decimal(explicit);
      if (d.lte(0)) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_LINE_INVALID, { sequence_no: 0, reason: 'exchange_rate must be > 0' });
      return Result.ok(d);
    }
    const res = await this.exchangeRates.findByDateAndCurrency(docDate.toISOString().slice(0, 10), currencyCode);
    if (res.isError()) return Result.error(res.error);
    return Result.ok(new Prisma.Decimal((res.value as { exchange_rate: number | string }).exchange_rate));
  }

  /**
   * Doc Reference rules (spec §6.6): invoice only, same vendor+currency, posted, within remaining
   * กฎ Doc Reference (spec §6.6): เฉพาะ invoice, vendor+สกุลเดียวกัน, posted, ไม่เกินยอดคงเหลือ
   */
  private async validateReferences(data: ICreateApInvoice, vendorId: string, currencyId: string): Promise<Result<true>> {
    const refs = data.references ?? [];
    if (refs.length === 0) return Result.ok(true);
    if (data.doc_type !== enum_ap_invoice_doc_type.invoice) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_LINE_INVALID, { sequence_no: 0, reason: 'only invoices may reference other documents' });
    const rows = await this.prismaService.tb_ap_invoice.findMany({ where: { id: { in: refs.map((r) => r.ref_ap_invoice_id) }, deleted_at: null } });
    for (const r of refs) {
      const row = rows.find((x) => x.id === r.ref_ap_invoice_id);
      if (!row || row.doc_type === enum_ap_invoice_doc_type.invoice) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_NOT_FOUND);
      if (row.vendor_id !== vendorId || row.currency_id !== currencyId) return Result.errorFromCatalog(ERROR_CATALOG.AP_VENDOR_CURRENCY_MISMATCH);
      if (row.doc_status !== enum_ap_invoice_status.posted) return Result.errorFromCatalog(ERROR_CATALOG.AP_REFERENCE_NOT_POSTED, { doc_no: row.doc_no });
      const remaining = row.total_amount.sub(row.reference_applied_amount);
      if (new Prisma.Decimal(r.applied_amount).lte(0) || new Prisma.Decimal(r.applied_amount).gt(remaining)) return Result.errorFromCatalog(ERROR_CATALOG.AP_REFERENCE_OVER_APPLIED, { doc_no: row.doc_no });
    }
    return Result.ok(true);
  }

  /** Create a draft / สร้างฉบับร่าง @param data Payload @returns `{ id, doc_no }` */
  @TryCatch
  async create(data: ICreateApInvoice): Promise<Result<unknown>> {
    this.logger.debug({ function: 'create', doc_type: data.doc_type }, ApInvoiceService.name);
    const ctx = await this.buildContext(data, null);
    if (ctx.isError()) return Result.error(ctx.error);
    try {
      return await this.prismaService.$transaction(async (tx) => {
        const docNo = await generateApDocNo({ commonLogic: this.commonLogic, prisma: tx, userId: this.userId, buCode: this.bu_code, docType: data.doc_type, docDate: data.doc_date });
        const written = await this.writer.writeDocument(tx, { ...ctx.value, doc_no: docNo });
        if (written.isError()) return Result.error(written.error);
        return Result.ok({ id: written.value.id, doc_no: docNo });
      });
    } catch (error) {
      if (isUniqueConstraintViolation(error)) return Result.errorFromCatalog(ERROR_CATALOG.AP_VENDOR_INVOICE_NO_DUPLICATE, { vendor_invoice_no: data.vendor_invoice_no });
      throw error;
    }
  }

  /** Update a draft (replace-all lines/references) / แก้ไขฉบับร่าง @param id Id @param data Payload @returns `{ id, doc_version }` */
  @TryCatch
  async update(id: string, data: IUpdateApInvoice): Promise<Result<unknown>> {
    this.logger.debug({ function: 'update', id }, ApInvoiceService.name);
    const existing = await this.loadFull(this.prismaService, id);
    if (!existing) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_NOT_FOUND);
    if (existing.doc_status !== enum_ap_invoice_status.draft || existing.doc_version !== data.doc_version) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_IMMUTABLE);
    const merged: ICreateApInvoice = {
      doc_type: existing.doc_type, doc_source: existing.doc_source, doc_date: data.doc_date ?? existing.doc_date, vendor_id: existing.vendor_id,
      vendor_invoice_no: data.vendor_invoice_no ?? existing.vendor_invoice_no, invoice_date: data.invoice_date ?? existing.invoice_date,
      credit_term_days: data.credit_term_days ?? existing.credit_term_days, currency_id: existing.currency_id,
      exchange_rate: data.exchange_rate ?? existing.exchange_rate.toString(), description: data.description ?? existing.description,
      attachments: data.attachments ?? (existing.attachments as unknown[]),
      details: data.details ?? { add: existing.tb_ap_invoice_detail.map(toLineInput) },
      references: data.references ?? existing.tb_ap_invoice_reference.map((r) => ({ ref_ap_invoice_id: r.ref_ap_invoice_id, applied_amount: r.applied_amount.toString() })),
    };
    const ctx = await this.buildContext(merged, existing);
    if (ctx.isError()) return Result.error(ctx.error);
    try {
      return await this.prismaService.$transaction(async (tx) => {
        const written = await this.writer.writeDocument(tx, ctx.value);
        if (written.isError()) return Result.error(written.error);
        const row = await tx.tb_ap_invoice.findFirst({ where: { id }, select: { doc_version: true } });
        return Result.ok({ id, doc_version: row?.doc_version });
      });
    } catch (error) {
      if (isUniqueConstraintViolation(error)) return Result.errorFromCatalog(ERROR_CATALOG.AP_VENDOR_INVOICE_NO_DUPLICATE, { vendor_invoice_no: merged.vendor_invoice_no });
      throw error;
    }
  }

  /** Soft-delete a draft / ลบฉบับร่าง @param id Id @returns `{ id }` */
  @TryCatch
  async delete(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'delete', id }, ApInvoiceService.name);
    const existing = await this.prismaService.tb_ap_invoice.findFirst({ where: { id, deleted_at: null }, select: { doc_status: true } });
    if (!existing) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_NOT_FOUND);
    if (existing.doc_status !== enum_ap_invoice_status.draft) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_IMMUTABLE);
    const now = new Date();
    await this.prismaService.$transaction([
      this.prismaService.tb_ap_invoice_detail.updateMany({ where: { ap_invoice_id: id, deleted_at: null }, data: { deleted_at: now, deleted_by_id: this.userId } }),
      this.prismaService.tb_ap_invoice.update({ where: { id }, data: { deleted_at: now, deleted_by_id: this.userId } }),
    ]);
    return Result.ok({ id });
  }

  /**
   * Committed GRN items of a vendor/currency that still have quantity to invoice (spec §6.5)
   * รายการ GRN ที่ commit แล้วของ vendor/สกุลนี้ ซึ่งยังมีจำนวนให้ออก invoice (spec §6.5)
   * @param vendorId - Vendor / vendor
   * @param currencyId - Currency / สกุลเงิน
   * @returns Items with remaining_qty / รายการพร้อม remaining_qty
   */
  @TryCatch
  async grnCandidates(vendorId: string, currencyId: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'grnCandidates', vendorId, currencyId }, ApInvoiceService.name);
    const items = await this.prismaService.tb_good_received_note_detail_item.findMany({
      where: { deleted_at: null, tb_good_received_note_detail: { deleted_at: null, tb_good_received_note: { vendor_id: vendorId, currency_id: currencyId, doc_status: enum_good_received_note_status.committed, post_type: 'ap', deleted_at: null } } },
      include: { tb_good_received_note_detail: { include: { tb_good_received_note: { select: { id: true, grn_no: true, grn_date: true, invoice_no: true } } } } },
    });
    const matched = await this.matchedQtyByItem(items.map((i) => i.id));
    const data = items
      .map((i) => ({
        good_received_note_detail_item_id: i.id, good_received_note_id: i.tb_good_received_note_detail.tb_good_received_note.id,
        grn_no: i.tb_good_received_note_detail.tb_good_received_note.grn_no, grn_date: i.tb_good_received_note_detail.tb_good_received_note.grn_date,
        product_id: i.tb_good_received_note_detail.product_id, received_qty: Number(i.received_qty ?? 0), received_price: Number(i.received_price ?? 0),
        tax_profile_id: i.tax_profile_id, tax_rate: Number(i.tax_rate ?? 0),
        remaining_qty: Number(new Prisma.Decimal(i.received_qty ?? 0).sub(matched.get(i.id) ?? 0)),
      }))
      .filter((i) => i.remaining_qty > 0);
    return Result.ok(data);
  }

  /** Σ matched_qty per GRN item over non-void documents / Σ matched_qty ต่อรายการ GRN ในเอกสารที่ไม่ void */
  async matchedQtyByItem(itemIds: string[]): Promise<Map<string, Prisma.Decimal>> {
    const rows = await this.prismaService.tb_ap_invoice_detail_source.findMany({
      where: { good_received_note_detail_item_id: { in: itemIds }, tb_ap_invoice_detail: { deleted_at: null, tb_ap_invoice: { doc_status: { not: enum_ap_invoice_status.void }, deleted_at: null } } },
      select: { good_received_note_detail_item_id: true, matched_qty: true },
    });
    const out = new Map<string, Prisma.Decimal>();
    for (const r of rows) out.set(r.good_received_note_detail_item_id, (out.get(r.good_received_note_detail_item_id) ?? new Prisma.Decimal(0)).add(r.matched_qty));
    return out;
  }

  /**
   * Build a draft invoice from GRN items; Dr account from product account mapping when available
   * สร้าง invoice ร่างจากรายการ GRN บัญชี Dr จาก product account mapping ถ้ามี
   * @param data - Item ids + header fields / รหัสรายการ + ฟิลด์ header
   * @returns `{ id, doc_no }` / รหัสและเลขที่
   */
  @TryCatch
  async createFromGrn(data: ICreateApInvoiceFromGrn): Promise<Result<unknown>> {
    this.logger.debug({ function: 'createFromGrn', count: data.grn_item_ids.length }, ApInvoiceService.name);
    const items = await this.prismaService.tb_good_received_note_detail_item.findMany({
      where: { id: { in: data.grn_item_ids }, deleted_at: null },
      include: { tb_good_received_note_detail: { include: { tb_good_received_note: true, tb_product: { select: { id: true, name: true, product_item_group_id: true } } } } },
    });
    if (items.length === 0) return Result.errorFromCatalog(ERROR_CATALOG.AP_GRN_NOT_COMMITTED, { grn_no: '' });
    const grns = new Set(items.map((i) => i.tb_good_received_note_detail.tb_good_received_note.id));
    const first = items[0].tb_good_received_note_detail.tb_good_received_note;
    for (const i of items) {
      const g = i.tb_good_received_note_detail.tb_good_received_note;
      if (g.doc_status !== enum_good_received_note_status.committed) return Result.errorFromCatalog(ERROR_CATALOG.AP_GRN_NOT_COMMITTED, { grn_no: g.grn_no });
      if (g.vendor_id !== first.vendor_id || g.currency_id !== first.currency_id) return Result.errorFromCatalog(ERROR_CATALOG.AP_VENDOR_CURRENCY_MISMATCH);
    }
    const matched = await this.matchedQtyByItem(items.map((i) => i.id));
    const lines = [];
    for (const [idx, i] of items.entries()) {
      const remaining = new Prisma.Decimal(i.received_qty ?? 0).sub(matched.get(i.id) ?? 0);
      if (remaining.lte(0)) return Result.errorFromCatalog(ERROR_CATALOG.AP_GRN_OVER_MATCHED, { grn_no: i.tb_good_received_note_detail.tb_good_received_note.grn_no });
      const product = i.tb_good_received_note_detail.tb_product;
      const drAccount = await this.resolveProductAccount(product.id, product.product_item_group_id);
      lines.push({
        sequence_no: idx + 1, description: `${i.tb_good_received_note_detail.tb_good_received_note.grn_no} ${product.name}`, product_id: product.id, product_name: product.name,
        unit_id: i.received_unit_id, unit_name: i.received_unit_name, quantity: remaining.toString(), unit_price: (i.received_price ?? 0).toString(),
        dr_chart_of_accounts_id: drAccount ?? '', vat_tax_profile_id: i.tax_profile_id,
        sources: [{ good_received_note_detail_item_id: i.id, matched_qty: remaining.toString() }],
      });
    }
    if (lines.some((l) => !l.dr_chart_of_accounts_id)) {
      return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_LINE_INVALID, { sequence_no: lines.findIndex((l) => !l.dr_chart_of_accounts_id) + 1, reason: 'no product account mapping; set dr_chart_of_accounts_id and create manually' });
    }
    this.logger.debug({ function: 'createFromGrn', grns: [...grns] }, ApInvoiceService.name);
    return this.create({
      doc_type: enum_ap_invoice_doc_type.invoice, doc_source: enum_ap_invoice_source.grn, doc_date: data.doc_date, vendor_id: first.vendor_id as string,
      vendor_invoice_no: data.vendor_invoice_no, invoice_date: data.invoice_date, credit_term_days: data.credit_term_days ?? first.credit_term_days,
      currency_id: first.currency_id, exchange_rate: data.exchange_rate ?? first.exchange_rate?.toString(), description: data.description, details: { add: lines },
    });
  }

  /** COA id from tb_product_account_code_mapping (product → item group), `inventory` type / บัญชีจาก mapping */
  private async resolveProductAccount(productId: string, itemGroupId: string | null): Promise<string | null> {
    const mapping = await this.prismaService.tb_product_account_code_mapping.findFirst({
      where: { account_type: 'inventory', deleted_at: null, OR: [{ product_id: productId }, ...(itemGroupId ? [{ product_item_group_id: itemGroupId }] : [])] },
      orderBy: { product_id: 'desc' },
      select: { account_code: true },
    });
    if (!mapping) return null;
    const account = await this.prismaService.tb_chart_of_accounts.findFirst({ where: { code: mapping.account_code, deleted_at: null }, select: { id: true } });
    return account?.id ?? null;
  }

  /** Posted DN/CN/DP of the same vendor+currency with remaining balance (spec §6.6) / DN/CN/DP ที่อ้างได้ @param id Invoice id @returns Candidates */
  @TryCatch
  async referenceCandidates(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'referenceCandidates', id }, ApInvoiceService.name);
    const inv = await this.prismaService.tb_ap_invoice.findFirst({ where: { id, deleted_at: null }, select: { vendor_id: true, currency_id: true } });
    if (!inv) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_NOT_FOUND);
    const rows = await this.prismaService.tb_ap_invoice.findMany({
      where: { vendor_id: inv.vendor_id, currency_id: inv.currency_id, doc_status: enum_ap_invoice_status.posted, doc_type: { not: enum_ap_invoice_doc_type.invoice }, deleted_at: null },
      select: { id: true, doc_no: true, doc_type: true, doc_date: true, total_amount: true, reference_applied_amount: true, exchange_rate: true },
    });
    return Result.ok(rows.map((r) => ({ ...r, remaining_amount: Number(r.total_amount.sub(r.reference_applied_amount)) })).filter((r) => r.remaining_amount > 0));
  }

  /** Active workflow for AP invoices, or null (no-workflow BU) / workflow ที่ active หรือ null */
  protected async resolveWorkflowId(): Promise<string | null> {
    const workflow = await this.prismaService.tb_workflow.findFirst({ where: { workflow_type: enum_workflow_type.ap_invoice, is_active: true, deleted_at: null }, select: { id: true }, orderBy: { created_at: 'asc' } });
    return workflow?.id ?? null;
  }
}

/** Existing detail row → line input for replace-all update / แถวเดิม → input สำหรับแก้ไข */
function toLineInput(d: ApInvoiceFull['tb_ap_invoice_detail'][number]) {
  return {
    sequence_no: d.sequence_no, description: d.description, product_id: d.product_id, product_name: d.product_name, unit_id: d.unit_id, unit_name: d.unit_name,
    quantity: d.quantity.toString(), unit_price: d.unit_price.toString(), discount_amount: d.discount_amount.toString(),
    dr_chart_of_accounts_id: d.dr_chart_of_accounts_id, dr_cost_center_id: d.dr_cost_center_id,
    vat_tax_profile_id: d.vat_tax_profile_id, vat_amount: d.vat_amount.toString(), vat_is_override: d.vat_is_override, vat_chart_of_accounts_id: d.vat_chart_of_accounts_id, vat_cost_center_id: d.vat_cost_center_id,
    wht_tax_profile_id: d.wht_tax_profile_id, cr_chart_of_accounts_id: d.cr_chart_of_accounts_id, cr_cost_center_id: d.cr_cost_center_id,
    dimensions: d.tb_ap_invoice_detail_dimension.map((x) => ({ gl_dimension_id: x.gl_dimension_id, gl_dimension_value_id: x.gl_dimension_value_id })),
    sources: d.tb_ap_invoice_detail_source.map((s) => ({ good_received_note_detail_item_id: s.good_received_note_detail_item_id, matched_qty: s.matched_qty.toString() })),
  };
}
```
หมายเหตุ: `this.prismaSystem` มีบน `TenantScopedService` อยู่แล้ว (exchange-rate.service ใช้) ถ้าชื่อต่างให้ใช้ชื่อเดียวกับใน `apps/micro-business/src/master/exchange-rate/exchange-rate.service.ts`

- [ ] **Step 3: check + commit** (module/controller มาใน Task 11 — type-check ผ่านได้เพราะยังไม่มีใคร import; ถ้า Nest DI ต้องการ module ให้สร้าง `ap-invoice.module.ts` ชั่วคราวตาม Task 11 Step 2)

```bash
bun run check-types && npx eslint --no-fix apps/micro-business/src/ap/ap-invoice
git add apps/micro-business/src/ap/ap-invoice
git commit -m "feat(ap): AP invoice CRUD, GRN candidates/create-from-GRN and reference candidates

Spec §6.3, §6.5, §6.6"
```

---

### Task 10: AP invoice service — submit/approve/review/reject/void/post + tax record

**Files:**
- Modify: `apps/micro-business/src/ap/ap-invoice/ap-invoice.service.ts` (เพิ่ม method)
- Create: `apps/micro-business/src/ap/ap-invoice/ap-invoice.posting.ts` (ขั้นตอน post/void ใน tx)

**Interfaces:**
- Consumes: `GlSubledgerPostingService`, `buildInvoiceJvLines`, `buildDepositReferenceJvLines`, `allocateAcrossLines`, `WorkflowOrchestratorService`, `apInvoiceToWorkflowDocument`
- Produces: `ApInvoiceService.{submit, approve, review, reject, void}`; `ApInvoicePosting.{postInTx, voidInTx, isPeriodOpen, upsertTaxRecord}`

- [ ] **Step 1: posting helper** (`ap-invoice.posting.ts`)

```ts
import { Injectable } from '@nestjs/common';
import { Result } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import { enum_ap_invoice_doc_type, enum_ap_invoice_status, enum_ap_invoice_tax_status, enum_gl_jv_source, enum_gl_period_status, enum_good_received_note_status, Prisma } from '@repo/prisma-shared-schema-tenant';
import { GlSubledgerPostingService } from '@/gl/gl-subledger-posting/gl-subledger-posting.service';
import { GlPostingService } from '@/gl/gl-posting/gl-posting.service';
import { allocateAcrossLines, buildDepositReferenceJvLines, buildInvoiceJvLines, round2 } from './ap-invoice.logic';
import type { ApInvoiceFull } from './ap-invoice.service';

const LAST_REGULAR_PERIOD = 12;

/**
 * Transaction-scoped steps of posting and voiding an AP document (spec §6.2, §6.6, §6.7)
 * ขั้นตอนใน transaction ของการ post และ void เอกสาร AP (spec §6.2, §6.6, §6.7)
 */
@Injectable()
export class ApInvoicePosting {
  constructor(
    private readonly subledger: GlSubledgerPostingService,
    private readonly glPosting: GlPostingService,
  ) {}

  /**
   * Whether a regular open GL period covers the date
   * มีงวด GL ปกติที่เปิดอยู่ครอบวันที่นี้หรือไม่
   * @param db - Client / client
   * @param date - Document date / วันที่เอกสาร
   * @returns true when open / true เมื่อเปิด
   */
  async isPeriodOpen(db: Pick<Prisma.TransactionClient, 'tb_gl_period'>, date: Date): Promise<boolean> {
    const period = await db.tb_gl_period.findFirst({ where: { status: enum_gl_period_status.open, period_no: { lte: LAST_REGULAR_PERIOD }, start_at: { lte: date }, end_at: { gte: date }, deleted_at: null }, select: { id: true } });
    return Boolean(period);
  }

  /**
   * GRN sources must still be committed and within remaining qty (Review Focus #3)
   * แหล่ง GRN ต้องยัง committed และไม่เกินยอดคงเหลือ (Review Focus #3)
   * @param tx - Transaction / transaction
   * @param doc - Document / เอกสาร
   * @returns ok or violation / ok หรือข้อผิดพลาด
   */
  async checkGrnSources(tx: Prisma.TransactionClient, doc: ApInvoiceFull): Promise<Result<true>> {
    const sources = doc.tb_ap_invoice_detail.flatMap((d) => d.tb_ap_invoice_detail_source);
    if (sources.length === 0) return Result.ok(true);
    const items = await tx.tb_good_received_note_detail_item.findMany({
      where: { id: { in: sources.map((s) => s.good_received_note_detail_item_id) } },
      include: { tb_good_received_note_detail: { include: { tb_good_received_note: { select: { grn_no: true, doc_status: true } } } } },
    });
    const others = await tx.tb_ap_invoice_detail_source.groupBy({
      by: ['good_received_note_detail_item_id'], _sum: { matched_qty: true },
      where: { good_received_note_detail_item_id: { in: sources.map((s) => s.good_received_note_detail_item_id) }, tb_ap_invoice_detail: { deleted_at: null, tb_ap_invoice: { id: { not: doc.id }, doc_status: { not: enum_ap_invoice_status.void }, deleted_at: null } } },
    });
    for (const s of sources) {
      const item = items.find((i) => i.id === s.good_received_note_detail_item_id);
      const grn = item?.tb_good_received_note_detail.tb_good_received_note;
      if (!grn || grn.doc_status !== enum_good_received_note_status.committed) return Result.errorFromCatalog(ERROR_CATALOG.AP_GRN_NOT_COMMITTED, { grn_no: grn?.grn_no ?? s.grn_no });
      const used = others.find((o) => o.good_received_note_detail_item_id === s.good_received_note_detail_item_id)?._sum.matched_qty ?? new Prisma.Decimal(0);
      if (s.matched_qty.add(used).gt(item?.received_qty ?? 0)) return Result.errorFromCatalog(ERROR_CATALOG.AP_GRN_OVER_MATCHED, { grn_no: grn.grn_no });
    }
    return Result.ok(true);
  }

  /**
   * Create or refresh the tax invoice record (spec §6.7)
   * สร้างหรือปรับปรุง tax invoice record (spec §6.7)
   * @param tx - Transaction / transaction
   * @param doc - Document / เอกสาร
   * @param userId - Actor / ผู้ทำ
   */
  async upsertTaxRecord(tx: Prisma.TransactionClient, doc: ApInvoiceFull, userId: string): Promise<void> {
    const hasVat = doc.tb_ap_invoice_detail.some((d) => !d.vat_amount.isZero());
    if (!hasVat) {
      await tx.tb_ap_invoice_tax.deleteMany({ where: { ap_invoice_id: doc.id } });
      return;
    }
    const vendor = await tx.tb_vendor.findFirst({ where: { id: doc.vendor_id }, select: { tax_no: true, branch_no: true, name: true } });
    const sign = doc.doc_type === enum_ap_invoice_doc_type.credit_note ? -1 : 1;
    const expiry = new Date(doc.invoice_date);
    expiry.setUTCMonth(expiry.getUTCMonth() + 6);
    const vatRate = doc.tb_ap_invoice_detail.find((d) => !d.vat_amount.isZero())?.vat_rate ?? new Prisma.Decimal(0);
    const data = {
      tax_invoice_no: doc.vendor_invoice_no, tax_invoice_date: doc.invoice_date, vendor_tax_no: vendor?.tax_no ?? null, vendor_branch_no: vendor?.branch_no ?? null, vendor_name: vendor?.name ?? doc.vendor_name,
      base_amount: doc.base_net_amount.mul(sign), vat_rate: vatRate, vat_amount: doc.base_vat_amount.mul(sign), tax_status: enum_ap_invoice_tax_status.pending, expiry_claim_date: expiry,
    };
    await tx.tb_ap_invoice_tax.upsert({ where: { ap_invoice_id: doc.id }, create: { ap_invoice_id: doc.id, ...data, created_by_id: userId }, update: { ...data, updated_at: new Date(), updated_by_id: userId } });
  }

  /**
   * Post: build JV lines, post through the facade, apply references, mark posted (spec §6.2)
   * post: สร้างบรรทัด JV, post ผ่าน facade, ใช้ reference, ตั้งสถานะ posted (spec §6.2)
   * @param tx - Transaction / transaction
   * @param doc - Document loaded inside tx / เอกสารที่โหลดใน tx
   * @param userId - Actor / ผู้ทำ
   * @returns `{ id, doc_status, gl_jv_no }` / ผลลัพธ์
   */
  async postInTx(tx: Prisma.TransactionClient, doc: ApInvoiceFull, userId: string): Promise<Result<unknown>> {
    if (!(await this.isPeriodOpen(tx, doc.doc_date))) return Result.errorFromCatalog(ERROR_CATALOG.GL_PERIOD_NOT_OPEN);
    const grn = await this.checkGrnSources(tx, doc);
    if (grn.isError()) return Result.error(grn.error);
    const setting = await this.glPosting.getSetting();
    const inputVat = this.subledger.requireSettingAccount(setting, 'input_vat_account_id');
    if (inputVat.isError()) return Result.error(inputVat.error);
    const needsDeposit = doc.doc_type === enum_ap_invoice_doc_type.deposit || doc.tb_ap_invoice_reference.some((r) => r.ref_doc_type === enum_ap_invoice_doc_type.deposit);
    const depositAccount = needsDeposit ? this.subledger.requireSettingAccount(setting, 'advance_deposit_account_id') : Result.ok('');
    if (depositAccount.isError()) return Result.error(depositAccount.error);
    const ctx = { doc_type: doc.doc_type, currency_id: doc.currency_id, exchange_rate: doc.exchange_rate, base_currency_id: doc.base_currency_id, input_vat_account_id: inputVat.value, advance_deposit_account_id: depositAccount.value };
    const lines = buildInvoiceJvLines(ctx, doc.tb_ap_invoice_detail.map((d) => ({ ...d, dimensions: d.tb_ap_invoice_detail_dimension.map((x) => ({ gl_dimension_id: x.gl_dimension_id, gl_dimension_value_id: x.gl_dimension_value_id })) })));
    const depositRefs = doc.tb_ap_invoice_reference.filter((r) => r.ref_doc_type === enum_ap_invoice_doc_type.deposit);
    if (depositRefs.length > 0) {
      const gain = this.subledger.requireSettingAccount(setting, 'realized_fx_gain_account_id');
      const loss = this.subledger.requireSettingAccount(setting, 'realized_fx_loss_account_id');
      if (gain.isError()) return Result.error(gain.error);
      if (loss.isError()) return Result.error(loss.error);
      lines.push(...buildDepositReferenceJvLines(ctx, doc.tb_ap_invoice_detail[0].cr_chart_of_accounts_id, depositRefs, { gain: gain.value, loss: loss.value }, lines.length + 1));
    }
    const posted = await this.subledger.postFromSource({ source: enum_gl_jv_source.ap, source_ref_type: 'ap_invoice', source_ref_id: doc.id, jv_date: doc.doc_date, description: `${doc.doc_no} ${doc.vendor_name}`, currency_id: doc.currency_id, exchange_rate: doc.exchange_rate, lines }, tx);
    if (posted.isError()) return Result.error(posted.error);
    await this.applyReferences(tx, doc, userId);
    const applied = doc.tb_ap_invoice_reference.reduce((s, r) => s.add(r.applied_amount), new Prisma.Decimal(0));
    await this.reduceUnpaid(tx, doc, applied, doc.exchange_rate);
    const outstanding = await this.recomputeOutstanding(tx, doc.id);
    await tx.tb_ap_invoice.update({
      where: { id: doc.id },
      data: { doc_status: enum_ap_invoice_status.posted, gl_jv_id: posted.value.jv_id, gl_jv_no: posted.value.jv_no, posted_at: new Date(), posted_by_id: userId, outstanding_amount: outstanding, doc_version: { increment: 1 }, updated_at: new Date(), updated_by_id: userId },
    });
    return Result.ok({ id: doc.id, doc_status: enum_ap_invoice_status.posted, gl_jv_no: posted.value.jv_no });
  }

  /** Increase reference_applied on referenced docs and reduce their unpaid lines / เพิ่มยอดที่ถูกอ้างและลดยอดค้างของใบที่ถูกอ้าง */
  private async applyReferences(tx: Prisma.TransactionClient, doc: ApInvoiceFull, userId: string): Promise<void> {
    for (const r of doc.tb_ap_invoice_reference) {
      const ref = await tx.tb_ap_invoice.findFirst({ where: { id: r.ref_ap_invoice_id }, include: { tb_ap_invoice_detail: { where: { deleted_at: null }, orderBy: { sequence_no: 'asc' } } } });
      if (!ref) continue;
      await this.reduceUnpaid(tx, ref as ApInvoiceFull, r.applied_amount, ref.exchange_rate);
      const outstanding = await this.recomputeOutstanding(tx, ref.id);
      await tx.tb_ap_invoice.update({ where: { id: ref.id }, data: { reference_applied_amount: ref.reference_applied_amount.add(r.applied_amount), outstanding_amount: outstanding, updated_at: new Date(), updated_by_id: userId } });
    }
  }

  /** Spread an amount over a document's lines FIFO / กระจายยอดลงบรรทัด FIFO */
  private async reduceUnpaid(tx: Prisma.TransactionClient, doc: { tb_ap_invoice_detail: { id: string; unpaid_amount: Prisma.Decimal }[] }, amount: Prisma.Decimal, rate: Prisma.Decimal): Promise<void> {
    if (amount.lte(0)) return;
    const plan = allocateAcrossLines(doc.tb_ap_invoice_detail, amount);
    for (const [lineId, take] of plan) {
      await tx.tb_ap_invoice_detail.update({ where: { id: lineId }, data: { unpaid_amount: { decrement: take }, base_unpaid_amount: { decrement: round2(take.mul(rate)) } } });
    }
  }

  /** outstanding = Σ unpaid of live lines / ยอดค้าง = Σ unpaid ของบรรทัดที่ยังอยู่ */
  async recomputeOutstanding(tx: Prisma.TransactionClient, invoiceId: string): Promise<Prisma.Decimal> {
    const agg = await tx.tb_ap_invoice_detail.aggregate({ where: { ap_invoice_id: invoiceId, deleted_at: null }, _sum: { unpaid_amount: true } });
    return agg._sum.unpaid_amount ?? new Prisma.Decimal(0);
  }

  /**
   * Void a posted document: guards, reversal JV, release references/sources, tax void (spec §6.2)
   * void เอกสารที่ post แล้ว: ตรวจเงื่อนไข, JV กลับรายการ, ปลด reference/GRN, tax void (spec §6.2)
   * @param tx - Transaction / transaction
   * @param doc - Document / เอกสาร
   * @param reason - Reason / เหตุผล
   * @param userId - Actor / ผู้ทำ
   * @returns `{ id, doc_status }` / ผลลัพธ์
   */
  async voidPostedInTx(tx: Prisma.TransactionClient, doc: ApInvoiceFull, reason: string, userId: string): Promise<Result<unknown>> {
    const payments = await tx.tb_ap_payment_detail.count({ where: { ap_invoice_id: doc.id, tb_ap_payment: { doc_status: { not: 'void' }, deleted_at: null } } });
    if (payments > 0) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_HAS_PAYMENT);
    if (!doc.reference_applied_amount.isZero()) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_IS_REFERENCED);
    const now = new Date();
    if (!(await this.isPeriodOpen(tx, now))) return Result.errorFromCatalog(ERROR_CATALOG.GL_PERIOD_NOT_OPEN);
    const reversed = await this.subledger.reverseBySource({ source_ref_type: 'ap_invoice', source_ref_id: doc.id }, now, reason, tx);
    if (reversed.isError()) return Result.error(reversed.error);
    for (const r of doc.tb_ap_invoice_reference) {
      const ref = await tx.tb_ap_invoice.findFirst({ where: { id: r.ref_ap_invoice_id }, include: { tb_ap_invoice_detail: { where: { deleted_at: null }, orderBy: { sequence_no: 'desc' } } } });
      if (!ref) continue;
      await this.restoreUnpaid(tx, ref.tb_ap_invoice_detail, r.applied_amount, ref.exchange_rate);
      const outstanding = await this.recomputeOutstanding(tx, ref.id);
      await tx.tb_ap_invoice.update({ where: { id: ref.id }, data: { reference_applied_amount: ref.reference_applied_amount.sub(r.applied_amount), outstanding_amount: outstanding, updated_at: now, updated_by_id: userId } });
    }
    await tx.tb_ap_invoice_detail_source.deleteMany({ where: { ap_invoice_detail_id: { in: doc.tb_ap_invoice_detail.map((d) => d.id) } } });
    await tx.tb_ap_invoice_tax.updateMany({ where: { ap_invoice_id: doc.id }, data: { tax_status: enum_ap_invoice_tax_status.void, updated_at: now, updated_by_id: userId } });
    await tx.tb_ap_invoice.update({ where: { id: doc.id }, data: { doc_status: enum_ap_invoice_status.void, void_at: now, void_by_id: userId, void_reason: reason, void_gl_jv_id: reversed.value.jv_id, outstanding_amount: new Prisma.Decimal(0), doc_version: { increment: 1 }, updated_at: now, updated_by_id: userId } });
    return Result.ok({ id: doc.id, doc_status: enum_ap_invoice_status.void });
  }

  /** Give unpaid back to lines, last line first (mirror of reduce) / คืนยอดค้างให้บรรทัดจากท้ายก่อน */
  private async restoreUnpaid(tx: Prisma.TransactionClient, lines: { id: string; total_amount: Prisma.Decimal; unpaid_amount: Prisma.Decimal }[], amount: Prisma.Decimal, rate: Prisma.Decimal): Promise<void> {
    const room = lines.map((l) => ({ id: l.id, unpaid_amount: l.total_amount.sub(l.unpaid_amount) }));
    const plan = allocateAcrossLines(room, amount);
    for (const [lineId, give] of plan) {
      await tx.tb_ap_invoice_detail.update({ where: { id: lineId }, data: { unpaid_amount: { increment: give }, base_unpaid_amount: { increment: round2(give.mul(rate)) } } });
    }
  }
}
```

- [ ] **Step 2: workflow actions ใน `ApInvoiceService`** (เพิ่ม constructor param `private readonly posting: ApInvoicePosting` และ method ต่อไปนี้)

```ts
  /** Write orchestrator output onto the header / เขียนผลจาก orchestrator ลง header */
  private workflowColumns(workflow: import('@/common/workflow/workflow.interfaces').WorkflowHeader) {
    return {
      workflow_current_stage: workflow.workflow_current_stage, workflow_previous_stage: workflow.workflow_previous_stage, workflow_next_stage: workflow.workflow_next_stage,
      workflow_history: workflow.workflow_history as unknown as Prisma.InputJsonValue, user_action: workflow.user_action as unknown as Prisma.InputJsonValue,
      last_action: workflow.last_action, last_action_at_date: workflow.last_action_at_date, last_action_by_id: workflow.last_action_by_id, last_action_by_name: workflow.last_action_by_name,
      doc_version: { increment: 1 }, updated_at: new Date(), updated_by_id: this.userId,
    };
  }

  /** Load + status guard shared by actions / โหลด + ตรวจสถานะที่ทุก action ใช้ */
  private async loadForAction(id: string, allowed: enum_ap_invoice_status[], docVersion?: number): Promise<Result<ApInvoiceFull>> {
    const doc = await this.loadFull(this.prismaService, id);
    if (!doc) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_NOT_FOUND);
    if (!allowed.includes(doc.doc_status) || (docVersion !== undefined && doc.doc_version !== docVersion)) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_IMMUTABLE);
    return Result.ok(doc);
  }

  /** Run the post inside one transaction with an optimistic lock on the header (Review Focus #2) / post ใน tx เดียวพร้อม lock header */
  private async postDocument(id: string, expectedVersion: number): Promise<Result<unknown>> {
    return this.prismaService.$transaction(async (tx) => {
      const locked = await tx.tb_ap_invoice.updateMany({ where: { id, doc_version: expectedVersion, doc_status: { in: [enum_ap_invoice_status.draft, enum_ap_invoice_status.in_review] } }, data: { doc_version: { increment: 1 } } });
      if (locked.count !== 1) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_IMMUTABLE);
      const doc = await this.loadFull(tx, id);
      if (!doc) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_NOT_FOUND);
      const refs = await validateApLines(tx, doc.tb_ap_invoice_detail.map(toLineInput), doc.doc_type === enum_ap_invoice_doc_type.deposit);
      if (refs.isError()) return Result.error(refs.error);
      return this.posting.postInTx(tx, doc, this.userId);
    }, { timeout: 20000 });
  }

  /** Submit: validate, tax record, then workflow or immediate post (spec §6.2) / ส่งเข้าตรวจ หรือ post ทันทีถ้าไม่มี workflow */
  @TryCatch
  async submit(id: string, data: { doc_version?: number }): Promise<Result<unknown>> {
    this.logger.debug({ function: 'submit', id }, ApInvoiceService.name);
    const loaded = await this.loadForAction(id, [enum_ap_invoice_status.draft], data?.doc_version);
    if (loaded.isError()) return Result.error(loaded.error);
    const doc = loaded.value;
    if (!(await this.posting.isPeriodOpen(this.prismaService, doc.doc_date))) return Result.errorFromCatalog(ERROR_CATALOG.GL_PERIOD_NOT_OPEN);
    const grn = await this.prismaService.$transaction((tx) => this.posting.checkGrnSources(tx, doc));
    if (grn.isError()) return Result.error(grn.error);
    await this.prismaService.$transaction((tx) => this.posting.upsertTaxRecord(tx, doc, this.userId));
    const workflowId = await this.resolveWorkflowId();
    if (!workflowId) return this.postDocument(id, doc.doc_version);
    const workflow = await this.workflowOrchestrator.buildSubmitWorkflow(apInvoiceToWorkflowDocument({ ...doc, workflow_id: doc.workflow_id ?? workflowId }), this.userId, this.bu_code);
    await this.prismaService.tb_ap_invoice.update({ where: { id, doc_version: doc.doc_version }, data: { workflow_id: workflowId, doc_status: enum_ap_invoice_status.in_review, ...this.workflowColumns(workflow) } });
    return Result.ok({ id, doc_status: enum_ap_invoice_status.in_review });
  }

  /** Approve; final approval posts / อนุมัติ ขั้นสุดท้าย post */
  @TryCatch
  async approve(id: string, data: { doc_version?: number }): Promise<Result<unknown>> {
    this.logger.debug({ function: 'approve', id }, ApInvoiceService.name);
    const loaded = await this.loadForAction(id, [enum_ap_invoice_status.in_review], data?.doc_version);
    if (loaded.isError()) return Result.error(loaded.error);
    const doc = loaded.value;
    const { workflow, isFinalApproval } = await this.workflowOrchestrator.buildApproveWorkflow(apInvoiceToWorkflowDocument(doc), this.userId, this.bu_code);
    const updated = await this.prismaService.tb_ap_invoice.update({ where: { id, doc_version: doc.doc_version }, data: this.workflowColumns(workflow), select: { doc_version: true } });
    if (!isFinalApproval) return Result.ok({ id, doc_status: enum_ap_invoice_status.in_review });
    return this.postDocument(id, updated.doc_version);
  }

  /** Send back to the creator's stage / ตีกลับไปขั้นผู้สร้าง */
  @TryCatch
  async review(id: string, data: { doc_version?: number; stage?: string | null }): Promise<Result<unknown>> {
    this.logger.debug({ function: 'review', id }, ApInvoiceService.name);
    const loaded = await this.loadForAction(id, [enum_ap_invoice_status.in_review], data?.doc_version);
    if (loaded.isError()) return Result.error(loaded.error);
    const doc = loaded.value;
    const workflow = await this.workflowOrchestrator.buildReviewWorkflow(apInvoiceToWorkflowDocument(doc), data?.stage ?? (doc.workflow_previous_stage as string), this.userId, this.bu_code);
    await this.prismaService.tb_ap_invoice.update({ where: { id, doc_version: doc.doc_version }, data: { doc_status: enum_ap_invoice_status.draft, ...this.workflowColumns(workflow) } });
    return Result.ok({ id, doc_status: enum_ap_invoice_status.draft });
  }

  /** Reject back to draft (spec §6.2 deviation from FRD) / ปฏิเสธกลับเป็นร่าง */
  @TryCatch
  async reject(id: string, data: { doc_version?: number; reason?: string | null }): Promise<Result<unknown>> {
    this.logger.debug({ function: 'reject', id }, ApInvoiceService.name);
    const loaded = await this.loadForAction(id, [enum_ap_invoice_status.in_review], data?.doc_version);
    if (loaded.isError()) return Result.error(loaded.error);
    const doc = loaded.value;
    const workflow = await this.workflowOrchestrator.buildRejectWorkflow(apInvoiceToWorkflowDocument(doc), this.userId, this.bu_code);
    await this.prismaService.tb_ap_invoice.update({ where: { id, doc_version: doc.doc_version }, data: { doc_status: enum_ap_invoice_status.draft, description: data?.reason ? `${doc.description ?? ''} [rejected: ${data.reason}]`.trim() : doc.description, ...this.workflowColumns(workflow) } });
    return Result.ok({ id, doc_status: enum_ap_invoice_status.draft });
  }

  /** Void a draft directly or reverse a posted one (spec §6.2) / void ร่างตรง หรือกลับรายการใบที่ post */
  @TryCatch
  async void(id: string, data: { doc_version?: number; reason?: string | null }): Promise<Result<unknown>> {
    this.logger.debug({ function: 'void', id }, ApInvoiceService.name);
    const loaded = await this.loadForAction(id, [enum_ap_invoice_status.draft, enum_ap_invoice_status.posted], data?.doc_version);
    if (loaded.isError()) return Result.error(loaded.error);
    const doc = loaded.value;
    const reason = data?.reason?.trim() || 'void';
    if (doc.doc_status === enum_ap_invoice_status.draft) {
      await this.prismaService.tb_ap_invoice.update({ where: { id, doc_version: doc.doc_version }, data: { doc_status: enum_ap_invoice_status.void, void_at: new Date(), void_by_id: this.userId, void_reason: reason, doc_version: { increment: 1 } } });
      return Result.ok({ id, doc_status: enum_ap_invoice_status.void });
    }
    return this.prismaService.$transaction(async (tx) => {
      const locked = await tx.tb_ap_invoice.updateMany({ where: { id, doc_version: doc.doc_version, doc_status: enum_ap_invoice_status.posted }, data: { doc_version: { increment: 1 } } });
      if (locked.count !== 1) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_IMMUTABLE);
      const fresh = await this.loadFull(tx, id);
      return this.posting.voidPostedInTx(tx, fresh as ApInvoiceFull, reason, this.userId);
    }, { timeout: 20000 });
  }
```
เพิ่ม import ใน service: `import { ApInvoicePosting } from './ap-invoice.posting';` และ `import { apInvoiceToWorkflowDocument } from './workflow/ap-invoice-workflow.mapper';`

- [ ] **Step 3: check + commit**

```bash
bun run check-types && npx eslint --no-fix apps/micro-business/src/ap/ap-invoice
git add apps/micro-business/src/ap/ap-invoice
git commit -m "feat(ap): AP invoice workflow actions, posting through subledger facade, and void with reversal

Spec §6.2, §6.6, §6.7, §8"
```

---

### Task 11: AP invoice — controller, module, contract, activity registry, gateway, ตรวจด้วยมือ

**Files:**
- Create: `apps/micro-business/src/ap/ap-invoice/ap-invoice.controller.ts`
- Create: `apps/micro-business/src/ap/ap-invoice/ap-invoice.module.ts`
- Modify: `apps/micro-business/src/app.module.ts`
- Modify: `apps/micro-business/src/common/activity/activity-registry.ts` (`DOCUMENT_ENTITIES`)
- Generated: `packages/rpc-contract/src/contracts/ap-invoice.ts` (`ApInvoice`)
- Create: `apps/backend-gateway/src/application/ap-invoice/{ap-invoice.controller.ts,ap-invoice.service.ts,ap-invoice.module.ts,swagger/request.ts,swagger/response.ts}`
- Modify: `apps/backend-gateway/src/application/route-application.ts`

**Interfaces:**
- Produces: RPC `ApInvoice.{findAll,findOne,create,update,delete,submit,approve,review,reject,void,grnCandidates,createFromGrn,referenceCandidates}`; REST `api/:bu_code/ap-invoice/*`

- [ ] **Step 1: micro-business controller** (literal pattern ก่อน generate)

```ts
import { Controller, HttpStatus } from '@nestjs/common';
import { MessagePattern, Payload } from '@nestjs/microservices';
import { BaseMicroserviceController, MicroservicePayload, MicroserviceResponse } from '@/common';
import { BackendLogger } from '@/common/helpers/backend.logger';
import { TenantContextRunner } from '@/tenant/tenant-context.runner';
import { ApInvoiceService } from './ap-invoice.service';

const SERVICE = 'micro-business';

/**
 * RPC handlers for AP invoice / DN / CN / deposit
 * ตัวรับ RPC ของเอกสาร AP ทั้ง 4 ประเภท
 */
@Controller()
export class ApInvoiceController extends BaseMicroserviceController {
  private readonly logger: BackendLogger = new BackendLogger(ApInvoiceController.name);

  constructor(private readonly service: ApInvoiceService, private readonly ctx: TenantContextRunner) {
    super();
  }

  /** Find one / ค้นหาหนึ่งใบ @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'ap-invoice.find-one', service: SERVICE })
  async findOne(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'findOne', payload }, ApInvoiceController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.findOne(payload.id)));
  }

  /** List / รายการ @param payload RPC payload @returns Paginated envelope */
  @MessagePattern({ cmd: 'ap-invoice.find-all', service: SERVICE })
  async findAll(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'findAll', payload }, ApInvoiceController.name);
    return this.handlePaginatedResult(await this.ctx.run(payload, () => this.service.findAll(payload.paginate)));
  }

  /** Create draft / สร้างร่าง @param payload RPC payload @returns Envelope 201 */
  @MessagePattern({ cmd: 'ap-invoice.create', service: SERVICE })
  async create(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'create', payload }, ApInvoiceController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.create(payload.data)), HttpStatus.CREATED);
  }

  /** Update draft / แก้ไขร่าง @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'ap-invoice.update', service: SERVICE })
  async update(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'update', payload }, ApInvoiceController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.update(payload.id, payload.data)));
  }

  /** Delete draft / ลบร่าง @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'ap-invoice.delete', service: SERVICE })
  async delete(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'delete', payload }, ApInvoiceController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.delete(payload.id)));
  }

  /** Submit / ส่งเข้าตรวจ @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'ap-invoice.submit', service: SERVICE })
  async submit(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'submit', payload }, ApInvoiceController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.submit(payload.id, payload.data ?? {})));
  }

  /** Approve / อนุมัติ @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'ap-invoice.approve', service: SERVICE })
  async approve(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'approve', payload }, ApInvoiceController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.approve(payload.id, payload.data ?? {})));
  }

  /** Send back / ตีกลับ @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'ap-invoice.review', service: SERVICE })
  async review(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'review', payload }, ApInvoiceController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.review(payload.id, payload.data ?? {})));
  }

  /** Reject / ปฏิเสธ @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'ap-invoice.reject', service: SERVICE })
  async reject(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'reject', payload }, ApInvoiceController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.reject(payload.id, payload.data ?? {})));
  }

  /** Void / ยกเลิก @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'ap-invoice.void', service: SERVICE })
  async void(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'void', payload }, ApInvoiceController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.void(payload.id, payload.data ?? {})));
  }

  /** GRN candidates / รายการ GRN ที่ออก invoice ได้ @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'ap-invoice.grn-candidates', service: SERVICE })
  async grnCandidates(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'grnCandidates', payload }, ApInvoiceController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.grnCandidates(payload.vendor_id, payload.currency_id)));
  }

  /** Create from GRN / สร้างจาก GRN @param payload RPC payload @returns Envelope 201 */
  @MessagePattern({ cmd: 'ap-invoice.create-from-grn', service: SERVICE })
  async createFromGrn(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'createFromGrn', payload }, ApInvoiceController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.createFromGrn(payload.data)), HttpStatus.CREATED);
  }

  /** Reference candidates / เอกสารที่อ้างหักได้ @param payload RPC payload @returns Envelope */
  @MessagePattern({ cmd: 'ap-invoice.reference-candidates', service: SERVICE })
  async referenceCandidates(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'referenceCandidates', payload }, ApInvoiceController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.referenceCandidates(payload.id)));
  }
}
```

- [ ] **Step 2: module + register**

```ts
import { Module } from '@nestjs/common';
import { TenantModule } from '@/tenant/tenant.module';
import { CommonModule } from '@/common/common.module';
import { WorkflowOrchestratorService } from '@/common/workflow/workflow-orchestrator.service';
import { ExchangeRateModule } from '@/master/exchange-rate/exchange-rate.module';
import { GlPostingModule } from '@/gl/gl-posting/gl-posting.module';
import { GlSubledgerPostingModule } from '@/gl/gl-subledger-posting/gl-subledger-posting.module';
import { ApInvoiceController } from './ap-invoice.controller';
import { ApInvoiceService } from './ap-invoice.service';
import { ApInvoiceWriter } from './ap-invoice.writer';
import { ApInvoicePosting } from './ap-invoice.posting';

/** AP invoice module / โมดูลเอกสาร AP */
@Module({
  imports: [TenantModule, CommonModule, ExchangeRateModule, GlPostingModule, GlSubledgerPostingModule],
  controllers: [ApInvoiceController],
  providers: [ApInvoiceService, ApInvoiceWriter, ApInvoicePosting, WorkflowOrchestratorService],
  exports: [ApInvoiceService],
})
export class ApInvoiceModule {}
```
(ถ้า `ExchangeRateModule` ไม่ export `ExchangeRateService` ให้เพิ่ม `exports: [ExchangeRateService]` ในโมดูลนั้น; ตรวจด้วย `grep -n exports apps/micro-business/src/master/exchange-rate/exchange-rate.module.ts`)
`app.module.ts`: import + `ApInvoiceModule,` หลัง `GlSubledgerPostingModule,`

- [ ] **Step 3: activity registry** — เพิ่มใน `DOCUMENT_ENTITIES` (activity-registry.ts ~L139 ต่อท้าย array)

```ts
  {
    entityName: 'tb_ap_invoice',
    mutations: [
      ['ap-invoice.create', 'create', CREATED_ID],
      ['ap-invoice.create-from-grn', 'create', CREATED_ID],
      ['ap-invoice.update', 'update', EDITED_ID],
      ['ap-invoice.submit', 'submit', EDITED_ID],
      ['ap-invoice.approve', 'approve', EDITED_ID],
      ['ap-invoice.review', 'review', EDITED_ID],
      ['ap-invoice.reject', 'reject', EDITED_ID],
      // No `void` value in enum_activity_action — the state change lands in new_data.
      ['ap-invoice.void', 'update', EDITED_ID],
      ['ap-invoice.delete', 'delete', DELETED_ID],
    ],
  },
```
(ถ้า `AuditAction` มีค่า `void`/`cancel` ให้ใช้ค่านั้นแทน `update` — ตรวจใน `packages/log-events-library`)

- [ ] **Step 4: generate contract + แทน literal**

```bash
bun run gen:rpc-contract && bun run build:package
```
แทน 13 literal ใน controller ด้วย `ApInvoice.findOne.pattern`, `ApInvoice.findAll.pattern`, `ApInvoice.create.pattern`, `ApInvoice.update.pattern`, `ApInvoice.delete.pattern`, `ApInvoice.submit.pattern`, `ApInvoice.approve.pattern`, `ApInvoice.review.pattern`, `ApInvoice.reject.pattern`, `ApInvoice.void.pattern`, `ApInvoice.grnCandidates.pattern`, `ApInvoice.createFromGrn.pattern`, `ApInvoice.referenceCandidates.pattern` (ชื่อ camelCase ที่ generator สร้างจาก kebab — เปิดไฟล์ที่ generate เพื่อยืนยัน)

- [ ] **Step 5: gateway swagger/request.ts**

```ts
import { createZodDto } from 'nestjs-zod';
import { z } from 'zod/v4';
import { enum_ap_invoice_doc_type, enum_ap_invoice_source } from '@repo/prisma-shared-schema-tenant';

const DimensionRef = z.object({ gl_dimension_id: z.string().uuid(), gl_dimension_value_id: z.string().uuid() });
const LineSource = z.object({ good_received_note_detail_item_id: z.string().uuid(), matched_qty: z.coerce.number().positive() });

/** One line / บรรทัด */
export const ApInvoiceLineSchema = z.object({
  sequence_no: z.number().int().min(1),
  description: z.string().nullable().optional(),
  product_id: z.string().uuid().nullable().optional(),
  product_name: z.string().nullable().optional(),
  unit_id: z.string().uuid().nullable().optional(),
  unit_name: z.string().nullable().optional(),
  quantity: z.coerce.number().positive(),
  unit_price: z.coerce.number().min(0),
  discount_amount: z.coerce.number().min(0).nullable().optional(),
  dr_chart_of_accounts_id: z.string().uuid(),
  dr_cost_center_id: z.string().uuid().nullable().optional(),
  vat_tax_profile_id: z.string().uuid().nullable().optional(),
  vat_amount: z.coerce.number().min(0).nullable().optional(),
  vat_is_override: z.boolean().optional(),
  vat_chart_of_accounts_id: z.string().uuid().nullable().optional(),
  vat_cost_center_id: z.string().uuid().nullable().optional(),
  wht_tax_profile_id: z.string().uuid().nullable().optional(),
  cr_chart_of_accounts_id: z.string().uuid().nullable().optional(),
  cr_cost_center_id: z.string().uuid().nullable().optional(),
  dimensions: z.array(DimensionRef).nullable().optional(),
  sources: z.array(LineSource).nullable().optional(),
});
const Reference = z.object({ ref_ap_invoice_id: z.string().uuid(), applied_amount: z.coerce.number().positive() });

/** Create body / เนื้อหาคำขอสร้าง */
export const ApInvoiceCreateRequestSchema = z.object({
  doc_type: z.nativeEnum(enum_ap_invoice_doc_type),
  doc_source: z.nativeEnum(enum_ap_invoice_source).optional(),
  doc_date: z.coerce.date(),
  vendor_id: z.string().uuid(),
  vendor_invoice_no: z.string().trim().min(1).max(100),
  invoice_date: z.coerce.date(),
  credit_term_days: z.number().int().min(0).nullable().optional(),
  currency_id: z.string().uuid().nullable().optional(),
  exchange_rate: z.coerce.number().positive().nullable().optional(),
  description: z.string().nullable().optional(),
  attachments: z.array(z.object({ originalName: z.string(), fileToken: z.string(), contentType: z.string() })).nullable().optional(),
  details: z.object({ add: z.array(ApInvoiceLineSchema).min(1) }),
  references: z.array(Reference).nullable().optional(),
});
export class ApInvoiceCreateRequestDto extends createZodDto(ApInvoiceCreateRequestSchema) {}

/** Update body / เนื้อหาคำขอแก้ไข */
export const ApInvoiceUpdateRequestSchema = ApInvoiceCreateRequestSchema.omit({ doc_type: true, doc_source: true, vendor_id: true, currency_id: true, details: true }).partial().extend({
  doc_version: z.number().int(),
  details: z.object({ add: z.array(ApInvoiceLineSchema).min(1) }).optional(),
});
export class ApInvoiceUpdateRequestDto extends createZodDto(ApInvoiceUpdateRequestSchema) {}

/** Action body / เนื้อหาคำขอ action */
export const ApInvoiceActionRequestSchema = z.object({ doc_version: z.number().int().optional(), reason: z.string().nullable().optional(), stage: z.string().nullable().optional() });
export class ApInvoiceActionRequestDto extends createZodDto(ApInvoiceActionRequestSchema) {}

/** Create-from-GRN body / เนื้อหาคำขอสร้างจาก GRN */
export const ApInvoiceFromGrnRequestSchema = z.object({
  grn_item_ids: z.array(z.string().uuid()).min(1),
  doc_date: z.coerce.date(),
  invoice_date: z.coerce.date(),
  vendor_invoice_no: z.string().trim().min(1).max(100),
  credit_term_days: z.number().int().min(0).nullable().optional(),
  exchange_rate: z.coerce.number().positive().nullable().optional(),
  description: z.string().nullable().optional(),
});
export class ApInvoiceFromGrnRequestDto extends createZodDto(ApInvoiceFromGrnRequestSchema) {}
```

`swagger/response.ts`: class `ApInvoiceResponseDto` มี `@ApiProperty` สำหรับ `id, doc_no, doc_type (enum, enumName 'enum_ap_invoice_doc_type'), doc_status (enum), vendor_name, vendor_invoice_no, doc_date, due_date, currency_code, exchange_rate, total_amount, base_total_amount, outstanding_amount, gl_jv_no?, doc_version?` และ `ApInvoiceMutationResponseDto { id; doc_no?; doc_status?; doc_version?; gl_jv_no? }`

- [ ] **Step 6: gateway service + controller + module**

`ap-invoice.service.ts` (gateway): เมธอดละ 1 บรรทัด `return this.rpc.call(ApInvoice.<action>, { ...payload, user_id, bu_code, version })` สำหรับ `findOne(id)`, `findAll(paginate)`, `create(data)`, `update(id, data)`, `delete(id)`, `submit(id, data)`, `approve(id, data)`, `review(id, data)`, `reject(id, data)`, `void(id, data)`, `grnCandidates(vendor_id, currency_id)`, `createFromGrn(data)`, `referenceCandidates(id)` — ทุกเมธอดรับ `(…, user_id, bu_code, version)` ท้ายสุดและ log debug เหมือน `application/gl-jv/gl-jv.service.ts`

`ap-invoice.controller.ts` (gateway) โครงเดียวกับ `application/gl-jv/gl-jv.controller.ts`:
```ts
@Controller('api/:bu_code/ap-invoice')
@ApiTags('Accounting: AP Invoice')
@ApiHeaderRequiredXAppId()
@UseGuards(KeycloakGuard)
@ApiBearerAuth()
export class ApInvoiceController extends BaseHttpController { ... }
```
routes (ทุก route: `@UseGuards(new AppIdGuard('ap-invoice.<action>'))`, `@ApiVersionMinRequest()`, `@ApiParam bu_code`, `@ApiStdResponse`, อ่าน `user_id` จาก `ExtractRequestHeader(req)`, `this.respond(res, result)`):

| method | path | body DTO | service |
|---|---|---|---|
| GET | `/grn-candidates?vendor_id&currency_id` | — | grnCandidates |
| GET | `/` | — (`PaginateQuery(query)`) | findAll |
| GET | `/:ap_invoice_id` | — | findOne |
| GET | `/:ap_invoice_id/reference-candidates` | — | referenceCandidates |
| POST | `/` (201) | ApInvoiceCreateRequestDto | create |
| POST | `/from-grn` (201) | ApInvoiceFromGrnRequestDto | createFromGrn |
| PUT | `/:ap_invoice_id` | ApInvoiceUpdateRequestDto | update |
| DELETE | `/:ap_invoice_id` | — | delete |
| POST | `/:ap_invoice_id/submit` | ApInvoiceActionRequestDto | submit |
| POST | `/:ap_invoice_id/approve` | ApInvoiceActionRequestDto | approve |
| POST | `/:ap_invoice_id/review` | ApInvoiceActionRequestDto | review |
| POST | `/:ap_invoice_id/reject` | ApInvoiceActionRequestDto | reject |
| POST | `/:ap_invoice_id/void` | ApInvoiceActionRequestDto | void — เพิ่ม `@UseGuards(KeycloakGuard, PermissionGuard)` ระดับ route ไม่ได้ ให้ประกาศ `@UseGuards(KeycloakGuard, PermissionGuard)` ที่ class และใส่ `@Permission({ 'accounting.ap': ['void'] })` เฉพาะ route นี้ (route อื่นไม่มี `@Permission` → PermissionGuard ปล่อยผ่านตามธรรมเนียม gl-posting; ยืนยันด้วย `bun run audit:api-system-permission`) |

`ap-invoice.module.ts` (gateway): `@Module({ controllers: [ApInvoiceController], providers: [ApInvoiceService] })` (ถ้า `PermissionGuard` ต้องการ `PermissionService` ใน providers ให้เพิ่มเหมือน `application/gl-posting/gl-posting.module.ts`)
`route-application.ts`: import + `ApInvoiceModule,` หลัง `GlPostingModule,`

- [ ] **Step 7: check + gates + commit**

```bash
bun run build:package && bun run check-types
npx eslint --no-fix apps/micro-business/src/ap/ap-invoice apps/micro-business/src/common/activity/activity-registry.ts apps/backend-gateway/src/application/ap-invoice
bun run gates
git add -A apps/micro-business/src/ap apps/micro-business/src/app.module.ts apps/micro-business/src/common/activity packages/rpc-contract apps/backend-gateway/src/application
git commit -m "feat(ap): expose AP invoice over RPC and REST with activity logging"
```

- [ ] **Step 8: ตรวจด้วยมือ — invoice flow** (spec §13 ข้อ 1, 3–6)

เตรียม (ผ่าน API ที่มีอยู่): ตั้ง `gl_setting` ผ่าน app-config endpoint ให้มี `ap_control_account_id, input_vat_account_id, advance_deposit_account_id, realized_fx_gain_account_id, realized_fx_loss_account_id, ap_jv_prefix_id`; สร้าง tax profile VAT 7% (`tax_type: vat`) และ WHT 3% (`tax_type: wht`, ทำได้หลัง Task 16 หรือ SQL ตรง); dimension value 1 ค่า + rule mandatory บนบัญชีค่าใช้จ่าย; GRN committed 1 ใบของ vendor ทดสอบ

```bash
export API=http://localhost:4000/api/$BU; H=(-H "Authorization: Bearer $TOKEN" -H "x-app-id: $APP_ID" -H "Content-Type: application/json")
# a. GRN candidates
curl -s "$API/ap-invoice/grn-candidates?vendor_id=<VENDOR>&currency_id=<CUR>" "${H[@]}"
# b. สร้างจาก GRN → 201 + doc_no รูป APIV2609xxxx
curl -s -X POST $API/ap-invoice/from-grn "${H[@]}" -d '{"grn_item_ids":["<ITEM>"],"doc_date":"2026-09-23","invoice_date":"2026-09-20","vendor_invoice_no":"INV-001"}'
# c. submit โดยบรรทัดยังไม่มี dimension ที่ mandatory → 422 GL_DIMENSION_REQUIRED
curl -s -X POST $API/ap-invoice/<ID>/submit "${H[@]}" -d '{}'
# d. update ใส่ dimensions ที่บรรทัด → submit → (approve) → doc_status posted, gl_jv_no ไม่ว่าง
# e. ตรวจ DB: tb_gl_jv_header source='ap' source_ref_id=<ID>; tb_gl_jv_detail_dimension มีแถว; tb_gl_balance ขยับ; tb_ap_invoice_tax มี 1 แถว; tb_ap_invoice_detail.unpaid_amount = total
# f. submit/approve ซ้ำ → 409 AP_INVOICE_IMMUTABLE และ JV ยังใบเดียว
# g. สร้าง vendor_invoice_no เดิมอีกใบ → 409 AP_VENDOR_INVOICE_NO_DUPLICATE
# h. deposit: สร้าง doc_type deposit → post; สร้าง invoice ใหม่ references [{ref_ap_invoice_id:<DP>, applied_amount:X}] ด้วย exchange_rate ต่างจาก DP → post → JV มีบรรทัด realized FX และ DP.reference_applied_amount = X
# i. void invoice ใบ h → ต้องผ่าน (ยังไม่มี PV) และ DP.reference_applied_amount กลับเป็น 0, มี JV reversal
```
บันทึกผลจริง (doc_no, jv_no, error code ที่ได้) ลง PR description

---

### Task 12: AP payment — interface, serializer, logic, numbering, validation, mapper

**Files:**
- Create: `apps/micro-business/src/ap/ap-payment/interface/ap-payment.interface.ts`
- Create: `apps/micro-business/src/ap/ap-payment/dto/ap-payment.serializer.ts`
- Create: `apps/micro-business/src/ap/ap-payment/ap-payment.logic.ts`
- Create: `apps/micro-business/src/ap/ap-payment/ap-payment.running-code.ts`
- Create: `apps/micro-business/src/ap/ap-payment/ap-payment.validation.ts`
- Create: `apps/micro-business/src/ap/ap-payment/workflow/ap-payment-workflow.mapper.ts`
- Modify: `apps/micro-business/src/common/dto/index.ts`

**Interfaces:**
- Produces: `IApPaymentAllocationInput`, `IApPaymentWhtInput`, `IApPaymentExpenseInput`, `ICreateApPayment`, `IUpdateApPayment`; `computeWhtRows(lines)`, `computePaymentTotals(...)`, `buildPaymentJvLines(...)`; `generateApPaymentNo(params)`; `validateApPaymentRefs(db, data)`, `loadAllocationLines(db, allocs)`; `apPaymentToWorkflowDocument(pv)`; `ApPaymentResponseSchema`

- [ ] **Step 1: interface**

```ts
import { enum_ap_payment_method } from '@repo/prisma-shared-schema-tenant';

/** Allocation to one invoice line (negative for credit notes) / การจ่ายต่อบรรทัด invoice (ลบสำหรับ CN) */
export interface IApPaymentAllocationInput {
  ap_invoice_detail_id: string;
  applied_amount: number | string;
}

/** WHT row, defaulted by wht-preview and editable / แถว WHT ที่ preview ให้และแก้ได้ */
export interface IApPaymentWhtInput {
  wht_tax_profile_id: string;
  base_amount: number | string;
  wht_amount: number | string;
  is_override?: boolean;
}

/** Bank fee / other expense line / ค่าธรรมเนียมหรือค่าใช้จ่ายอื่น */
export interface IApPaymentExpenseInput {
  chart_of_accounts_id: string;
  cost_center_id?: string | null;
  description?: string | null;
  amount: number | string;
}

/** Create payload / ข้อมูลสร้าง */
export interface ICreateApPayment {
  vendor_id: string;
  payment_date: Date;
  paid_date?: Date | null;
  bank_account_id: string;
  payment_method?: enum_ap_payment_method;
  cheque_no?: string | null;
  cheque_date?: Date | null;
  reference_no?: string | null;
  description?: string | null;
  exchange_rate?: number | string | null;
  attachments?: unknown[] | null;
  allocations: IApPaymentAllocationInput[];
  wht?: IApPaymentWhtInput[] | null;
  expenses?: IApPaymentExpenseInput[] | null;
}

/** Update payload (replace-all children) / ข้อมูลแก้ไข (แทนที่ลูกทั้งชุด) */
export interface IUpdateApPayment extends Partial<Omit<ICreateApPayment, 'vendor_id'>> {
  doc_version: number;
}
```

- [ ] **Step 2: serializer** (`dto/ap-payment.serializer.ts`)

```ts
import { z } from 'zod/v4';
import { enum_ap_invoice_doc_type, enum_ap_payment_method, enum_ap_payment_status, enum_last_action, enum_tax_profile_wht_pnd_form } from '@repo/prisma-shared-schema-tenant';

const money = z.coerce.number();

/** Payment response / การตอบกลับใบจ่ายเงิน */
export const ApPaymentResponseSchema = z.object({
  id: z.string(), doc_version: z.number().nullable().optional(), doc_no: z.string(), doc_status: z.nativeEnum(enum_ap_payment_status),
  payment_date: z.coerce.date(), paid_date: z.coerce.date().nullable().optional(), description: z.string().nullable().optional(), reference_no: z.string().nullable().optional(),
  vendor_id: z.string(), vendor_name: z.string(), payee_name: z.string(), payee_tax_no: z.string().nullable().optional(), payee_branch_no: z.string().nullable().optional(), payee_address: z.string().nullable().optional(),
  bank_account_id: z.string(), bank_account_code: z.string(), bank_account_name: z.string(), payment_method: z.nativeEnum(enum_ap_payment_method),
  cheque_no: z.string().nullable().optional(), cheque_date: z.coerce.date().nullable().optional(),
  currency_id: z.string(), currency_code: z.string(), exchange_rate: money, base_currency_id: z.string(), base_currency_code: z.string(),
  total_applied_amount: money, total_wht_amount: money, total_expense_amount: money, net_paid_amount: money,
  base_total_applied_amount: money, base_total_wht_amount: money, base_total_expense_amount: money, base_net_paid_amount: money, realized_fx_amount: money,
  gl_jv_id: z.string().nullable().optional(), gl_jv_no: z.string().nullable().optional(), posted_at: z.coerce.date().nullable().optional(),
  void_at: z.coerce.date().nullable().optional(), void_reason: z.string().nullable().optional(),
  workflow_id: z.string().nullable().optional(), workflow_current_stage: z.string().nullable().optional(), workflow_next_stage: z.string().nullable().optional(),
  workflow_history: z.unknown().optional(), user_action: z.unknown().optional(), last_action: z.nativeEnum(enum_last_action).nullable().optional(),
  attachments: z.unknown().optional(), created_at: z.coerce.date().nullable().optional(), updated_at: z.coerce.date().nullable().optional(),
  tb_ap_payment_detail: z.array(z.object({ id: z.string(), ap_invoice_id: z.string(), ap_invoice_detail_id: z.string(), doc_type: z.nativeEnum(enum_ap_invoice_doc_type), applied_amount: money, invoice_exchange_rate: money, base_applied_at_invoice_rate: money, base_applied_at_payment_rate: money, realized_fx_amount: money })).optional(),
  tb_ap_payment_wht: z.array(z.object({ id: z.string(), wht_tax_profile_id: z.string(), wht_tax_profile_name: z.string(), pnd_form: z.nativeEnum(enum_tax_profile_wht_pnd_form).nullable().optional(), income_type: z.string().nullable().optional(), base_amount: money, wht_rate: money, wht_amount: money, is_override: z.boolean() })).optional(),
  tb_ap_payment_expense: z.array(z.object({ id: z.string(), chart_of_accounts_id: z.string(), cost_center_id: z.string().nullable().optional(), description: z.string().nullable().optional(), amount: money, base_amount: money })).optional(),
  tax_invoices: z.array(z.object({ ap_invoice_id: z.string(), doc_no: z.string(), tax_invoice_no: z.string(), tax_invoice_date: z.coerce.date(), base_amount: money, vat_amount: money })).optional(),
});
```
เพิ่ม `export * from '@/ap/ap-payment/dto/ap-payment.serializer';` ใน `common/dto/index.ts`

- [ ] **Step 3: logic** (`ap-payment.logic.ts`) — spec §7.1–7.2, §8 แถว APPV

```ts
import { Prisma, tb_tax_profile } from '@repo/prisma-shared-schema-tenant';
import { round2 } from '../ap-invoice/ap-invoice.logic';
import { ISubledgerPostingLine } from '@/gl/gl-subledger-posting/interface/gl-subledger-posting.interface';

const ZERO = new Prisma.Decimal(0);
const HUNDRED = new Prisma.Decimal(100);

/** An invoice line joined to its allocation / บรรทัด invoice ที่จับคู่กับการจ่าย */
export interface IAllocationLine {
  ap_invoice_id: string;
  ap_invoice_detail_id: string;
  doc_no: string;
  doc_type: 'invoice' | 'debit_note' | 'credit_note' | 'deposit';
  sequence_no: number;
  cr_chart_of_accounts_id: string;
  invoice_exchange_rate: Prisma.Decimal;
  net_amount: Prisma.Decimal;
  total_amount: Prisma.Decimal;
  unpaid_amount: Prisma.Decimal;
  wht_tax_profile: tb_tax_profile | null;
  applied_amount: Prisma.Decimal;
}

/** WHT row computed per profile / แถว WHT ต่อ profile */
export interface IWhtRow {
  wht_tax_profile_id: string;
  wht_tax_profile_name: string;
  pnd_form: tb_tax_profile['wht_pnd_form'];
  income_type: string | null;
  wht_rate: Prisma.Decimal;
  base_amount: Prisma.Decimal;
  wht_amount: Prisma.Decimal;
}

/**
 * Default WHT per profile from the lines being paid (spec §7.2)
 * WHT default ต่อ profile จากบรรทัดที่จ่าย (spec §7.2)
 * @param lines - Allocation lines / บรรทัดที่จ่าย
 * @returns WHT rows / แถว WHT
 */
export function computeWhtRows(lines: IAllocationLine[]): IWhtRow[] {
  const byProfile = new Map<string, IWhtRow>();
  for (const l of lines) {
    const p = l.wht_tax_profile;
    if (!p || l.total_amount.isZero()) continue;
    const base = round2(l.applied_amount.mul(l.net_amount).div(l.total_amount));
    const rate = new Prisma.Decimal(p.tax_rate ?? 0);
    const row = byProfile.get(p.id) ?? { wht_tax_profile_id: p.id, wht_tax_profile_name: p.name, pnd_form: p.wht_pnd_form, income_type: p.wht_income_type, wht_rate: rate, base_amount: ZERO, wht_amount: ZERO };
    row.base_amount = row.base_amount.add(base);
    row.wht_amount = row.wht_amount.add(round2(base.mul(rate).div(HUNDRED)));
    byProfile.set(p.id, row);
  }
  return [...byProfile.values()];
}

/** Header totals of a payment / ยอดรวมของใบจ่าย */
export interface IPaymentTotals {
  total_applied_amount: Prisma.Decimal; base_applied_at_invoice: Prisma.Decimal; base_applied_at_payment: Prisma.Decimal;
  total_wht_amount: Prisma.Decimal; base_total_wht_amount: Prisma.Decimal;
  total_expense_amount: Prisma.Decimal; base_total_expense_amount: Prisma.Decimal;
  net_paid_amount: Prisma.Decimal; base_net_paid_amount: Prisma.Decimal; realized_fx_amount: Prisma.Decimal;
}

/**
 * net_paid = applied − wht + expense; realized FX = base at payment rate − base at invoice rate (spec §7.1)
 * net_paid = applied − wht + expense; realized FX = base ที่ payment rate − base ที่ invoice rate (spec §7.1)
 * @param lines - Allocation lines / บรรทัดที่จ่าย
 * @param wht - WHT rows / แถว WHT
 * @param expenses - Expense amounts / ค่าใช้จ่าย
 * @param paymentRate - Payment exchange rate / อัตราของใบจ่าย
 * @returns Totals / ยอดรวม
 */
export function computePaymentTotals(lines: IAllocationLine[], wht: { wht_amount: Prisma.Decimal }[], expenses: { amount: Prisma.Decimal }[], paymentRate: Prisma.Decimal): IPaymentTotals {
  const total_applied_amount = lines.reduce((s, l) => s.add(l.applied_amount), ZERO);
  const base_applied_at_invoice = lines.reduce((s, l) => s.add(round2(l.applied_amount.mul(l.invoice_exchange_rate))), ZERO);
  const base_applied_at_payment = lines.reduce((s, l) => s.add(round2(l.applied_amount.mul(paymentRate))), ZERO);
  const total_wht_amount = wht.reduce((s, w) => s.add(w.wht_amount), ZERO);
  const base_total_wht_amount = round2(total_wht_amount.mul(paymentRate));
  const total_expense_amount = expenses.reduce((s, e) => s.add(e.amount), ZERO);
  const base_total_expense_amount = round2(total_expense_amount.mul(paymentRate));
  return {
    total_applied_amount, base_applied_at_invoice, base_applied_at_payment, total_wht_amount, base_total_wht_amount, total_expense_amount, base_total_expense_amount,
    net_paid_amount: total_applied_amount.sub(total_wht_amount).add(total_expense_amount),
    base_net_paid_amount: base_applied_at_payment.sub(base_total_wht_amount).add(base_total_expense_amount),
    realized_fx_amount: base_applied_at_payment.sub(base_applied_at_invoice),
  };
}

/** Accounts the payment JV needs / บัญชีที่ JV ใบจ่ายต้องใช้ */
export interface IPaymentJvContext {
  currency_id: string; exchange_rate: Prisma.Decimal; base_currency_id: string;
  bank_account_coa_id: string; fx_gain_account_id: string; fx_loss_account_id: string;
}

/**
 * Ledger lines of a payment voucher (spec §8 rows APPV), balanced in base
 * บรรทัด ledger ของใบจ่ายเงิน (spec §8 แถว APPV) สมดุลใน base
 * @param ctx - Accounts and rates / บัญชีและอัตรา
 * @param lines - Allocation lines / บรรทัดที่จ่าย
 * @param wht - WHT rows with their payable account / แถว WHT พร้อมบัญชี
 * @param expenses - Expense lines / ค่าใช้จ่าย
 * @param totals - Header totals / ยอดรวม
 * @returns Facade lines / บรรทัดของ facade
 */
export function buildPaymentJvLines(
  ctx: IPaymentJvContext,
  lines: IAllocationLine[],
  wht: { chart_of_accounts_id: string; wht_amount: Prisma.Decimal; wht_tax_profile_name: string }[],
  expenses: { chart_of_accounts_id: string; cost_center_id: string | null; amount: Prisma.Decimal; description: string | null }[],
  totals: IPaymentTotals,
): ISubledgerPostingLine[] {
  const out: ISubledgerPostingLine[] = [];
  let seq = 1;
  const byAp = new Map<string, { amount: Prisma.Decimal; base: Prisma.Decimal }>();
  for (const l of lines) {
    const cur = byAp.get(l.cr_chart_of_accounts_id) ?? { amount: ZERO, base: ZERO };
    byAp.set(l.cr_chart_of_accounts_id, { amount: cur.amount.add(l.applied_amount), base: cur.base.add(round2(l.applied_amount.mul(l.invoice_exchange_rate))) });
  }
  for (const [account, v] of byAp) {
    if (v.amount.isZero()) continue;
    const isDebit = v.amount.gt(0);
    out.push({ sequence_no: seq++, chart_of_accounts_id: account, debit: isDebit ? v.amount : null, credit: isDebit ? null : v.amount.abs(), base_debit: isDebit ? v.base : ZERO, base_credit: isDebit ? ZERO : v.base.abs(), currency_id: ctx.currency_id, exchange_rate: ctx.exchange_rate, description: 'AP settlement' });
  }
  out.push({ sequence_no: seq++, chart_of_accounts_id: ctx.bank_account_coa_id, credit: totals.net_paid_amount, base_debit: ZERO, base_credit: totals.base_net_paid_amount, currency_id: ctx.currency_id, exchange_rate: ctx.exchange_rate, description: 'Bank' });
  for (const w of wht) {
    if (w.wht_amount.isZero()) continue;
    out.push({ sequence_no: seq++, chart_of_accounts_id: w.chart_of_accounts_id, credit: w.wht_amount, base_debit: ZERO, base_credit: round2(w.wht_amount.mul(ctx.exchange_rate)), currency_id: ctx.currency_id, exchange_rate: ctx.exchange_rate, description: `WHT ${w.wht_tax_profile_name}` });
  }
  for (const e of expenses) {
    out.push({ sequence_no: seq++, chart_of_accounts_id: e.chart_of_accounts_id, cost_center_id: e.cost_center_id, debit: e.amount, base_debit: round2(e.amount.mul(ctx.exchange_rate)), base_credit: ZERO, currency_id: ctx.currency_id, exchange_rate: ctx.exchange_rate, description: e.description ?? 'Payment expense' });
  }
  const fx = totals.realized_fx_amount;
  if (!fx.isZero()) {
    const isLoss = fx.gt(0);
    out.push({ sequence_no: seq++, chart_of_accounts_id: isLoss ? ctx.fx_loss_account_id : ctx.fx_gain_account_id, debit: isLoss ? fx.abs() : null, credit: isLoss ? null : fx.abs(), base_debit: isLoss ? fx.abs() : ZERO, base_credit: isLoss ? ZERO : fx.abs(), currency_id: ctx.base_currency_id, exchange_rate: new Prisma.Decimal(1), description: 'Realized FX on payment' });
  }
  return out;
}
```
หมายเหตุ rounding: `base_total_wht_amount` ปัดจากยอดรวม แต่บรรทัด WHT ปัดต่อแถว — ถ้าต่างกัน 0.01 base จะไม่สมดุล ดังนั้นใน `computePaymentTotals` ให้เปลี่ยน `base_total_wht_amount` เป็น `Σ round2(w.wht_amount × rate)` ต่อแถว (แก้ตอน implement: `wht.reduce((s, w) => s.add(round2(w.wht_amount.mul(paymentRate))), ZERO)`) และ `base_total_expense_amount` เช่นเดียวกัน (Review Focus #1)

- [ ] **Step 4: numbering** (`ap-payment.running-code.ts`) — เหมือน `generateApDocNo` แต่ type `'AP-PV'`, prefix `'APPV'`, lookup บน `tb_ap_payment.doc_no`; export `generateApPaymentNo(params: { commonLogic; prisma; userId; buCode; docDate })`

- [ ] **Step 5: validation** (`ap-payment.validation.ts`) — spec §7.1

```ts
import { enum_ap_invoice_doc_type, enum_ap_invoice_status, enum_chart_of_accounts_type, enum_tax_profile_tax_type, Prisma, type PrismaClient } from '@repo/prisma-shared-schema-tenant';
import { Result } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import { IApPaymentAllocationInput, IApPaymentExpenseInput, IApPaymentWhtInput } from './interface/ap-payment.interface';
import { IAllocationLine } from './ap-payment.logic';

type PaymentDb = Pick<PrismaClient, 'tb_ap_invoice_detail' | 'tb_ap_payment_detail' | 'tb_bank_account' | 'tb_chart_of_accounts' | 'tb_tax_profile' | 'tb_vendor'>;

/**
 * Load allocated invoice lines with their invoice header and WHT profile
 * โหลดบรรทัด invoice ที่จ่ายพร้อม header และ WHT profile
 * @param db - Client / client
 * @param allocs - Allocation inputs / การจ่ายที่ส่งมา
 * @param excludePaymentId - Payment being edited (its own reservations are not counted) / ใบจ่ายที่กำลังแก้
 * @returns Allocation lines / บรรทัดที่จ่าย
 */
export async function loadAllocationLines(db: PaymentDb, allocs: IApPaymentAllocationInput[], excludePaymentId: string | null): Promise<Result<IAllocationLine[]>> {
  if (allocs.length === 0) return Result.errorFromCatalog(ERROR_CATALOG.AP_PAYMENT_EMPTY);
  const ids = allocs.map((a) => a.ap_invoice_detail_id);
  const rows = await db.tb_ap_invoice_detail.findMany({ where: { id: { in: ids }, deleted_at: null }, include: { tb_ap_invoice: { select: { id: true, doc_no: true, doc_type: true, doc_status: true, vendor_id: true, currency_id: true, exchange_rate: true } } } });
  const reserved = await db.tb_ap_payment_detail.groupBy({ by: ['ap_invoice_detail_id'], _sum: { applied_amount: true }, where: { ap_invoice_detail_id: { in: ids }, tb_ap_payment: { doc_status: { in: ['draft', 'in_review'] }, deleted_at: null, ...(excludePaymentId ? { id: { not: excludePaymentId } } : {}) } } });
  const profileIds = [...new Set(rows.map((r) => r.wht_tax_profile_id).filter((x): x is string => Boolean(x)))];
  const profiles = new Map((await db.tb_tax_profile.findMany({ where: { id: { in: profileIds } } })).map((p) => [p.id, p]));
  const out: IAllocationLine[] = [];
  for (const a of allocs) {
    const r = rows.find((x) => x.id === a.ap_invoice_detail_id);
    if (!r) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_NOT_FOUND);
    if (r.tb_ap_invoice.doc_status !== enum_ap_invoice_status.posted) return Result.errorFromCatalog(ERROR_CATALOG.AP_PAYMENT_INVOICE_NOT_POSTED, { doc_no: r.tb_ap_invoice.doc_no });
    const applied = new Prisma.Decimal(a.applied_amount);
    const isCredit = r.tb_ap_invoice.doc_type === enum_ap_invoice_doc_type.credit_note;
    const held = reserved.find((x) => x.ap_invoice_detail_id === r.id)?._sum.applied_amount ?? new Prisma.Decimal(0);
    const available = r.unpaid_amount.sub(held.abs());
    if ((isCredit ? applied.gte(0) : applied.lte(0)) || applied.abs().gt(available)) {
      return Result.errorFromCatalog(ERROR_CATALOG.AP_PAYMENT_OVER_ALLOCATED, { doc_no: r.tb_ap_invoice.doc_no, sequence_no: r.sequence_no });
    }
    out.push({ ap_invoice_id: r.tb_ap_invoice.id, ap_invoice_detail_id: r.id, doc_no: r.tb_ap_invoice.doc_no, doc_type: r.tb_ap_invoice.doc_type, sequence_no: r.sequence_no, cr_chart_of_accounts_id: r.cr_chart_of_accounts_id, invoice_exchange_rate: r.tb_ap_invoice.exchange_rate, net_amount: r.net_amount, total_amount: r.total_amount, unpaid_amount: r.unpaid_amount, wht_tax_profile: r.wht_tax_profile_id ? (profiles.get(r.wht_tax_profile_id) ?? null) : null, applied_amount: applied });
  }
  const vendors = new Set(rows.map((r) => r.tb_ap_invoice.vendor_id));
  const currencies = new Set(rows.map((r) => r.tb_ap_invoice.currency_id));
  if (vendors.size > 1 || currencies.size > 1) return Result.errorFromCatalog(ERROR_CATALOG.AP_VENDOR_CURRENCY_MISMATCH);
  if (out.reduce((s, l) => s.add(l.applied_amount), new Prisma.Decimal(0)).lte(0)) return Result.errorFromCatalog(ERROR_CATALOG.AP_PAYMENT_EMPTY);
  return Result.ok(out);
}

/** Master refs the payment writer needs / master ที่ตัวเขียนใบจ่ายต้องใช้ */
export interface IResolvedPaymentRefs {
  vendor: { id: string; name: string; tax_no: string | null; branch_no: string | null };
  bankAccount: { id: string; code: string; name: string; currency_id: string; currency_code: string; chart_of_accounts_id: string };
  whtProfiles: Map<string, { id: string; name: string; tax_rate: Prisma.Decimal | null; wht_pnd_form: string | null; wht_income_type: string | null; chart_of_accounts_id: string | null }>;
}

/**
 * Vendor active, bank active with matching currency, WHT profiles of type wht, expense accounts postable
 * vendor active, ธนาคาร active สกุลตรง, WHT profile ประเภท wht, บัญชีค่าใช้จ่ายลงได้
 * @param db - Client / client
 * @param data - Vendor/bank/wht/expense ids / รหัสที่เกี่ยวข้อง
 * @param data.vendor_id - Vendor / vendor
 * @param data.bank_account_id - Bank account / บัญชีธนาคาร
 * @param data.currency_id - Payment currency / สกุลใบจ่าย
 * @param data.base_currency_id - Base currency / สกุลฐาน
 * @param data.wht - WHT rows / แถว WHT
 * @param data.expenses - Expense rows / ค่าใช้จ่าย
 * @returns Resolved refs / master ที่ resolve
 */
export async function validateApPaymentRefs(db: PaymentDb, data: { vendor_id: string; bank_account_id: string; currency_id: string; base_currency_id: string; wht: IApPaymentWhtInput[]; expenses: IApPaymentExpenseInput[] }): Promise<Result<IResolvedPaymentRefs>> {
  const vendor = await db.tb_vendor.findFirst({ where: { id: data.vendor_id, deleted_at: null, is_active: true }, select: { id: true, name: true, tax_no: true, branch_no: true } });
  if (!vendor) return Result.errorFromCatalog(ERROR_CATALOG.AP_VENDOR_NOT_FOUND);
  const bankAccount = await db.tb_bank_account.findFirst({ where: { id: data.bank_account_id, deleted_at: null, is_active: true } });
  if (!bankAccount) return Result.errorFromCatalog(ERROR_CATALOG.BANK_ACCOUNT_NOT_FOUND);
  if (bankAccount.currency_id !== data.currency_id && bankAccount.currency_id !== data.base_currency_id) return Result.errorFromCatalog(ERROR_CATALOG.AP_PAYMENT_BANK_CURRENCY_MISMATCH);
  const profiles = await db.tb_tax_profile.findMany({ where: { id: { in: data.wht.map((w) => w.wht_tax_profile_id) }, deleted_at: null, is_active: true, tax_type: enum_tax_profile_tax_type.wht } });
  if (profiles.length !== new Set(data.wht.map((w) => w.wht_tax_profile_id)).size) return Result.errorFromCatalog(ERROR_CATALOG.AP_TAX_PROFILE_INVALID, { tax_profile_id: 'wht' });
  for (const w of data.wht) if (new Prisma.Decimal(w.wht_amount).lt(0)) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_LINE_INVALID, { sequence_no: 0, reason: 'wht_amount must be >= 0' });
  const accountIds = data.expenses.map((e) => e.chart_of_accounts_id);
  const accounts = await db.tb_chart_of_accounts.findMany({ where: { id: { in: accountIds }, deleted_at: null, is_active: true, type: { notIn: [enum_chart_of_accounts_type.header, enum_chart_of_accounts_type.summary] } }, select: { id: true } });
  if (accounts.length !== new Set(accountIds).size) return Result.errorFromCatalog(ERROR_CATALOG.AP_ACCOUNT_NOT_POSTABLE, { account_id: 'expense' });
  for (const e of data.expenses) if (new Prisma.Decimal(e.amount).lte(0)) return Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_LINE_INVALID, { sequence_no: 0, reason: 'expense amount must be > 0' });
  return Result.ok({ vendor, bankAccount, whtProfiles: new Map(profiles.map((p) => [p.id, p])) });
}
```

- [ ] **Step 6: workflow mapper** — คัดลอก `ap-invoice-workflow.mapper.ts` เป็น `workflow/ap-payment-workflow.mapper.ts` เปลี่ยนชื่อฟังก์ชันเป็น `apPaymentToWorkflowDocument`, ข้อความ error เป็น `payment`, และ `total_amount: Number(inv.base_net_paid_amount ?? 0)` (param ชื่อ `base_net_paid_amount`)

- [ ] **Step 7: check + commit**

```bash
bun run check-types && npx eslint --no-fix apps/micro-business/src/ap/ap-payment
git add apps/micro-business/src/ap/ap-payment apps/micro-business/src/common/dto/index.ts
git commit -m "feat(ap): payment voucher interfaces, WHT/FX logic, numbering, validation and workflow mapper

Spec §7.1–7.2, §8"
```

---

### Task 13: AP payment service — CRUD, outstanding documents, WHT preview

**Files:**
- Create: `apps/micro-business/src/ap/ap-payment/ap-payment.writer.ts`
- Create: `apps/micro-business/src/ap/ap-payment/ap-payment.service.ts` (ส่วน CRUD)

**Interfaces:**
- Produces: `ApPaymentService.{findOne, findAll, create, update, delete, outstandingDocuments, whtPreview, loadFull}`, `ApPaymentWriter.writeDocument(tx, ctx)`, `ApPaymentFull`, `AP_PAYMENT_FULL_INCLUDE`

- [ ] **Step 1: writer** (`ap-payment.writer.ts`)

```ts
import { Injectable } from '@nestjs/common';
import { enum_ap_payment_method, enum_ap_payment_status, Prisma } from '@repo/prisma-shared-schema-tenant';
import { IApPaymentExpenseInput, IApPaymentWhtInput } from './interface/ap-payment.interface';
import { buildPaymentJvLines, computePaymentTotals, IAllocationLine } from './ap-payment.logic';
import { IResolvedPaymentRefs } from './ap-payment.validation';
import { round2 } from '../ap-invoice/ap-invoice.logic';

/** Everything the writer needs, resolved by the service / ทุกอย่างที่ writer ต้องใช้ */
export interface IWritePaymentContext {
  id: string | null;
  doc_no: string | null;
  payment_date: Date;
  paid_date: Date | null;
  description: string | null;
  reference_no: string | null;
  payment_method: enum_ap_payment_method;
  cheque_no: string | null;
  cheque_date: Date | null;
  currency: { id: string; code: string };
  exchange_rate: Prisma.Decimal;
  base_currency: { id: string; code: string };
  payee_address: string | null;
  attachments: unknown[] | null;
  lines: IAllocationLine[];
  wht: IApPaymentWhtInput[];
  expenses: IApPaymentExpenseInput[];
  refs: IResolvedPaymentRefs;
  wht_payable_default: string | null;
  userId: string;
}

/**
 * Single write path for a payment voucher and its children
 * ทางเดียวที่เขียนใบจ่ายเงินและลูกของมัน
 */
@Injectable()
export class ApPaymentWriter {
  /**
   * Create (id null) or replace (id set) inside the caller's transaction
   * สร้าง (id null) หรือแทนที่ (มี id) ใน transaction ของผู้เรียก
   * @param tx - Transaction / transaction
   * @param ctx - Context / บริบท
   * @returns `{ id }` / รหัส
   */
  async writeDocument(tx: Prisma.TransactionClient, ctx: IWritePaymentContext): Promise<{ id: string }> {
    const whtRows = ctx.wht.map((w) => {
      const p = ctx.refs.whtProfiles.get(w.wht_tax_profile_id);
      return { wht_tax_profile_id: w.wht_tax_profile_id, wht_tax_profile_name: p?.name ?? '', pnd_form: (p?.wht_pnd_form ?? null) as never, income_type: p?.wht_income_type ?? null, base_amount: round2(new Prisma.Decimal(w.base_amount)), wht_rate: new Prisma.Decimal(p?.tax_rate ?? 0), wht_amount: round2(new Prisma.Decimal(w.wht_amount)), is_override: w.is_override ?? false, chart_of_accounts_id: p?.chart_of_accounts_id ?? (ctx.wht_payable_default as string) };
    });
    const expenseRows = ctx.expenses.map((e) => ({ chart_of_accounts_id: e.chart_of_accounts_id, cost_center_id: e.cost_center_id ?? null, description: e.description ?? null, amount: round2(new Prisma.Decimal(e.amount)), base_amount: round2(new Prisma.Decimal(e.amount).mul(ctx.exchange_rate)) }));
    const totals = computePaymentTotals(ctx.lines, whtRows, expenseRows, ctx.exchange_rate);
    const header = {
      payment_date: ctx.payment_date, paid_date: ctx.paid_date, description: ctx.description, reference_no: ctx.reference_no,
      vendor_id: ctx.refs.vendor.id, vendor_name: ctx.refs.vendor.name, payee_name: ctx.refs.vendor.name, payee_tax_no: ctx.refs.vendor.tax_no, payee_branch_no: ctx.refs.vendor.branch_no, payee_address: ctx.payee_address,
      bank_account_id: ctx.refs.bankAccount.id, bank_account_code: ctx.refs.bankAccount.code, bank_account_name: ctx.refs.bankAccount.name, payment_method: ctx.payment_method, cheque_no: ctx.cheque_no, cheque_date: ctx.cheque_date,
      currency_id: ctx.currency.id, currency_code: ctx.currency.code, exchange_rate: ctx.exchange_rate, base_currency_id: ctx.base_currency.id, base_currency_code: ctx.base_currency.code,
      total_applied_amount: totals.total_applied_amount, total_wht_amount: totals.total_wht_amount, total_expense_amount: totals.total_expense_amount, net_paid_amount: totals.net_paid_amount,
      base_total_applied_amount: totals.base_applied_at_invoice, base_total_wht_amount: totals.base_total_wht_amount, base_total_expense_amount: totals.base_total_expense_amount, base_net_paid_amount: totals.base_net_paid_amount, realized_fx_amount: totals.realized_fx_amount,
      attachments: (ctx.attachments ?? []) as Prisma.InputJsonValue,
    };
    let id = ctx.id;
    if (id) {
      await tx.tb_ap_payment_detail.deleteMany({ where: { ap_payment_id: id } });
      await tx.tb_ap_payment_wht.deleteMany({ where: { ap_payment_id: id } });
      await tx.tb_ap_payment_expense.deleteMany({ where: { ap_payment_id: id } });
      await tx.tb_ap_payment.update({ where: { id }, data: { ...header, doc_version: { increment: 1 }, updated_at: new Date(), updated_by_id: ctx.userId } });
    } else {
      id = (await tx.tb_ap_payment.create({ data: { ...header, doc_no: ctx.doc_no as string, doc_status: enum_ap_payment_status.draft, created_by_id: ctx.userId, updated_by_id: ctx.userId }, select: { id: true } })).id;
    }
    await tx.tb_ap_payment_detail.createMany({ data: ctx.lines.map((l) => ({ ap_payment_id: id as string, ap_invoice_id: l.ap_invoice_id, ap_invoice_detail_id: l.ap_invoice_detail_id, doc_type: l.doc_type, applied_amount: l.applied_amount, invoice_exchange_rate: l.invoice_exchange_rate, base_applied_at_invoice_rate: round2(l.applied_amount.mul(l.invoice_exchange_rate)), base_applied_at_payment_rate: round2(l.applied_amount.mul(ctx.exchange_rate)), realized_fx_amount: round2(l.applied_amount.mul(ctx.exchange_rate)).sub(round2(l.applied_amount.mul(l.invoice_exchange_rate))), created_by_id: ctx.userId })) });
    if (whtRows.length) await tx.tb_ap_payment_wht.createMany({ data: whtRows.map((w) => ({ ...w, ap_payment_id: id as string, created_by_id: ctx.userId })) });
    if (expenseRows.length) await tx.tb_ap_payment_expense.createMany({ data: expenseRows.map((e) => ({ ...e, ap_payment_id: id as string, created_by_id: ctx.userId })) });
    return { id };
  }
}
```
(`buildPaymentJvLines` import ไว้เพื่อ Task 14 ถ้า eslint เตือน unused ให้ย้าย import ไป posting)

- [ ] **Step 2: service ส่วน CRUD** (`ap-payment.service.ts`)

```ts
import { Injectable } from '@nestjs/common';
import { TryCatch, Result, ApPaymentResponseSchema } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import { enum_ap_invoice_status, enum_ap_payment_method, enum_ap_payment_status, enum_workflow_type, Prisma } from '@repo/prisma-shared-schema-tenant';
import { TenantScopedService } from '@/common/tenant-scoped.service';
import { BackendLogger } from '@/common/helpers/backend.logger';
import { CommonLogic } from '@/common/common.logic';
import QueryParams from '@/common/libs/paginate.query';
import { withDefaultSort } from '@/common/libs/default-sort';
import getPaginationParams from '@/common/helpers/pagination.params';
import { IPaginate } from '@/common/shared-interface/paginate.interface';
import { WorkflowOrchestratorService } from '@/common/workflow/workflow-orchestrator.service';
import { ExchangeRateService } from '@/master/exchange-rate/exchange-rate.service';
import { GlPostingService } from '@/gl/gl-posting/gl-posting.service';
import { ApPaymentWriter, IWritePaymentContext } from './ap-payment.writer';
import { generateApPaymentNo } from './ap-payment.running-code';
import { loadAllocationLines, validateApPaymentRefs } from './ap-payment.validation';
import { computeWhtRows } from './ap-payment.logic';
import { ICreateApPayment, IUpdateApPayment } from './interface/ap-payment.interface';

/** Full payment shape / รูปใบจ่ายเต็ม */
export const AP_PAYMENT_FULL_INCLUDE = { tb_ap_payment_detail: true, tb_ap_payment_wht: true, tb_ap_payment_expense: true };
export type ApPaymentFull = Prisma.tb_ap_paymentGetPayload<{ include: typeof AP_PAYMENT_FULL_INCLUDE }>;

/**
 * AP payment voucher (spec §7)
 * ใบจ่ายเงิน AP (spec §7)
 */
@Injectable()
export class ApPaymentService extends TenantScopedService {
  private readonly logger: BackendLogger = new BackendLogger(ApPaymentService.name);

  constructor(
    private readonly commonLogic: CommonLogic,
    private readonly writer: ApPaymentWriter,
    private readonly exchangeRates: ExchangeRateService,
    private readonly workflowOrchestrator: WorkflowOrchestratorService,
    private readonly glPosting: GlPostingService,
  ) {
    super();
  }

  /** Load with children / โหลดพร้อมลูก @param db Client @param id Id @returns Row or null */
  async loadFull(db: Pick<Prisma.TransactionClient, 'tb_ap_payment'>, id: string): Promise<ApPaymentFull | null> {
    return db.tb_ap_payment.findFirst({ where: { id, deleted_at: null }, include: AP_PAYMENT_FULL_INCLUDE });
  }

  /** Find one incl. tax invoices of allocated documents (spec §7.4) / ค้นหาหนึ่งใบพร้อม tax invoice ของเอกสารที่จ่าย */
  @TryCatch
  async findOne(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findOne', id }, ApPaymentService.name);
    const row = await this.loadFull(this.prismaService, id);
    if (!row) return Result.errorFromCatalog(ERROR_CATALOG.AP_PAYMENT_NOT_FOUND);
    const invoiceIds = [...new Set(row.tb_ap_payment_detail.map((d) => d.ap_invoice_id))];
    const taxes = await this.prismaService.tb_ap_invoice_tax.findMany({ where: { ap_invoice_id: { in: invoiceIds } }, include: { tb_ap_invoice: { select: { doc_no: true } } } });
    return Result.ok(ApPaymentResponseSchema.parse({ ...row, tax_invoices: taxes.map((t) => ({ ap_invoice_id: t.ap_invoice_id, doc_no: t.tb_ap_invoice.doc_no, tax_invoice_no: t.tax_invoice_no, tax_invoice_date: t.tax_invoice_date, base_amount: t.base_amount, vat_amount: t.vat_amount })) }));
  }

  /** Paginated list / รายการแบบแบ่งหน้า @param paginate Params @returns Page */
  @TryCatch
  async findAll(paginate: IPaginate): Promise<Result<unknown>> {
    this.logger.debug({ function: 'findAll', paginate }, ApPaymentService.name);
    const q = new QueryParams(paginate.page, paginate.perpage, paginate.search, paginate.searchfields, ['doc_no', 'vendor_name', 'reference_no', 'cheque_no'], typeof paginate.filter === 'object' && !Array.isArray(paginate.filter) ? paginate.filter : {}, withDefaultSort(paginate.sort, ['payment_date:desc', 'doc_no:desc']), paginate.advance);
    const pagination = getPaginationParams(q.page, q.perpage);
    const rows = await this.prismaService.tb_ap_payment.findMany({ where: q.where(), orderBy: q.orderBy(), ...pagination });
    const total = await this.prismaService.tb_ap_payment.count({ where: q.where() });
    return Result.ok({ paginate: { total, page: q.perpage < 0 ? 1 : q.page, perpage: q.perpage < 0 ? 1 : q.perpage, pages: total === 0 || q.perpage < 0 ? 1 : Math.ceil(total / q.perpage) }, data: rows.map((r) => ApPaymentResponseSchema.parse(r)) });
  }

  /** Base currency of the BU / สกุลฐานของ BU */
  private async resolveBaseCurrency(): Promise<Result<{ id: string; code: string }>> {
    const bu = await this.prismaSystem.tb_business_unit.findFirst({ where: { code: this.bu_code }, select: { default_currency_id: true } });
    const currency = bu?.default_currency_id ? await this.prismaService.tb_currency.findFirst({ where: { id: bu.default_currency_id }, select: { id: true, code: true } }) : null;
    if (!currency) return Result.errorFromCatalog(ERROR_CATALOG.GL_SETTING_ACCOUNT_MISSING, { key: 'business_unit.default_currency_id' });
    return Result.ok(currency);
  }

  /**
   * Resolve everything the writer needs; payment currency = currency of the allocated invoices
   * resolve ทุกอย่างที่ writer ต้องใช้ สกุลของใบจ่าย = สกุลของ invoice ที่จ่าย
   */
  private async buildContext(data: ICreateApPayment, existing: ApPaymentFull | null): Promise<Result<IWritePaymentContext>> {
    const linesRes = await loadAllocationLines(this.prismaService, data.allocations, existing?.id ?? null);
    if (linesRes.isError()) return Result.error(linesRes.error);
    const first = await this.prismaService.tb_ap_invoice.findFirst({ where: { id: linesRes.value[0].ap_invoice_id }, select: { vendor_id: true, currency_id: true, currency_code: true } });
    if (!first || first.vendor_id !== data.vendor_id) return Result.errorFromCatalog(ERROR_CATALOG.AP_VENDOR_CURRENCY_MISMATCH);
    const baseRes = await this.resolveBaseCurrency();
    if (baseRes.isError()) return Result.error(baseRes.error);
    const refs = await validateApPaymentRefs(this.prismaService, { vendor_id: data.vendor_id, bank_account_id: data.bank_account_id, currency_id: first.currency_id, base_currency_id: baseRes.value.id, wht: data.wht ?? [], expenses: data.expenses ?? [] });
    if (refs.isError()) return Result.error(refs.error);
    const rate = await this.resolveRate(data.exchange_rate, data.payment_date, first.currency_code);
    if (rate.isError()) return Result.error(rate.error);
    const setting = await this.glPosting.getSetting();
    const address = await this.prismaService.tb_vendor_address.findFirst({ where: { vendor_id: data.vendor_id, address_type: 'register_address', deleted_at: null } });
    const payeeAddress = address ? [address.address_line1, address.address_line2, address.sub_district, address.district, address.province, address.postal_code].filter(Boolean).join(' ') : null;
    return Result.ok({
      id: existing?.id ?? null, doc_no: existing?.doc_no ?? null, payment_date: data.payment_date, paid_date: data.paid_date ?? null, description: data.description ?? null, reference_no: data.reference_no ?? null,
      payment_method: data.payment_method ?? enum_ap_payment_method.bank_transfer, cheque_no: data.cheque_no ?? null, cheque_date: data.cheque_date ?? null,
      currency: { id: first.currency_id, code: first.currency_code }, exchange_rate: rate.value, base_currency: baseRes.value, payee_address: payeeAddress, attachments: data.attachments ?? null,
      lines: linesRes.value, wht: data.wht ?? [], expenses: data.expenses ?? [], refs: refs.value, wht_payable_default: setting?.wht_payable_account_id ?? null, userId: this.userId,
    });
  }

  /** Header rate / อัตราของใบจ่าย */
  private async resolveRate(explicit: number | string | null | undefined, date: Date, code: string): Promise<Result<Prisma.Decimal>> {
    if (explicit !== null && explicit !== undefined) {
      const d = new Prisma.Decimal(explicit);
      return d.gt(0) ? Result.ok(d) : Result.errorFromCatalog(ERROR_CATALOG.AP_INVOICE_LINE_INVALID, { sequence_no: 0, reason: 'exchange_rate must be > 0' });
    }
    const res = await this.exchangeRates.findByDateAndCurrency(date.toISOString().slice(0, 10), code);
    if (res.isError()) return Result.error(res.error);
    return Result.ok(new Prisma.Decimal((res.value as { exchange_rate: number | string }).exchange_rate));
  }

  /** Create draft / สร้างร่าง @param data Payload @returns `{ id, doc_no }` */
  @TryCatch
  async create(data: ICreateApPayment): Promise<Result<unknown>> {
    this.logger.debug({ function: 'create', vendor: data.vendor_id }, ApPaymentService.name);
    const ctx = await this.buildContext(data, null);
    if (ctx.isError()) return Result.error(ctx.error);
    return this.prismaService.$transaction(async (tx) => {
      const docNo = await generateApPaymentNo({ commonLogic: this.commonLogic, prisma: tx, userId: this.userId, buCode: this.bu_code, docDate: data.payment_date });
      const { id } = await this.writer.writeDocument(tx, { ...ctx.value, doc_no: docNo });
      return Result.ok({ id, doc_no: docNo });
    });
  }

  /** Update draft (replace-all) / แก้ไขร่าง @param id Id @param data Payload @returns `{ id, doc_version }` */
  @TryCatch
  async update(id: string, data: IUpdateApPayment): Promise<Result<unknown>> {
    this.logger.debug({ function: 'update', id }, ApPaymentService.name);
    const existing = await this.loadFull(this.prismaService, id);
    if (!existing) return Result.errorFromCatalog(ERROR_CATALOG.AP_PAYMENT_NOT_FOUND);
    if (existing.doc_status !== enum_ap_payment_status.draft || existing.doc_version !== data.doc_version) return Result.errorFromCatalog(ERROR_CATALOG.AP_PAYMENT_IMMUTABLE);
    const merged: ICreateApPayment = {
      vendor_id: existing.vendor_id, payment_date: data.payment_date ?? existing.payment_date, paid_date: data.paid_date ?? existing.paid_date, bank_account_id: data.bank_account_id ?? existing.bank_account_id,
      payment_method: data.payment_method ?? existing.payment_method, cheque_no: data.cheque_no ?? existing.cheque_no, cheque_date: data.cheque_date ?? existing.cheque_date, reference_no: data.reference_no ?? existing.reference_no,
      description: data.description ?? existing.description, exchange_rate: data.exchange_rate ?? existing.exchange_rate.toString(), attachments: data.attachments ?? (existing.attachments as unknown[]),
      allocations: data.allocations ?? existing.tb_ap_payment_detail.map((d) => ({ ap_invoice_detail_id: d.ap_invoice_detail_id, applied_amount: d.applied_amount.toString() })),
      wht: data.wht ?? existing.tb_ap_payment_wht.map((w) => ({ wht_tax_profile_id: w.wht_tax_profile_id, base_amount: w.base_amount.toString(), wht_amount: w.wht_amount.toString(), is_override: w.is_override })),
      expenses: data.expenses ?? existing.tb_ap_payment_expense.map((e) => ({ chart_of_accounts_id: e.chart_of_accounts_id, cost_center_id: e.cost_center_id, description: e.description, amount: e.amount.toString() })),
    };
    const ctx = await this.buildContext(merged, existing);
    if (ctx.isError()) return Result.error(ctx.error);
    return this.prismaService.$transaction(async (tx) => {
      await this.writer.writeDocument(tx, ctx.value);
      const row = await tx.tb_ap_payment.findFirst({ where: { id }, select: { doc_version: true } });
      return Result.ok({ id, doc_version: row?.doc_version });
    });
  }

  /** Soft-delete draft / ลบร่าง @param id Id @returns `{ id }` */
  @TryCatch
  async delete(id: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'delete', id }, ApPaymentService.name);
    const existing = await this.prismaService.tb_ap_payment.findFirst({ where: { id, deleted_at: null }, select: { doc_status: true } });
    if (!existing) return Result.errorFromCatalog(ERROR_CATALOG.AP_PAYMENT_NOT_FOUND);
    if (existing.doc_status !== enum_ap_payment_status.draft) return Result.errorFromCatalog(ERROR_CATALOG.AP_PAYMENT_IMMUTABLE);
    await this.prismaService.$transaction([
      this.prismaService.tb_ap_payment_detail.deleteMany({ where: { ap_payment_id: id } }),
      this.prismaService.tb_ap_payment.update({ where: { id }, data: { deleted_at: new Date(), deleted_by_id: this.userId } }),
    ]);
    return Result.ok({ id });
  }

  /**
   * Posted documents of a vendor with unpaid lines, net of draft/in-review reservations (spec §7.1)
   * เอกสาร posted ของ vendor ที่ยังมีบรรทัดค้าง หักยอดที่ใบจ่ายร่าง/รอตรวจจองไว้ (spec §7.1)
   * @param vendorId - Vendor / vendor
   * @returns Documents with lines / เอกสารพร้อมบรรทัด
   */
  @TryCatch
  async outstandingDocuments(vendorId: string): Promise<Result<unknown>> {
    this.logger.debug({ function: 'outstandingDocuments', vendorId }, ApPaymentService.name);
    const docs = await this.prismaService.tb_ap_invoice.findMany({ where: { vendor_id: vendorId, doc_status: enum_ap_invoice_status.posted, outstanding_amount: { gt: 0 }, deleted_at: null }, include: { tb_ap_invoice_detail: { where: { deleted_at: null, unpaid_amount: { gt: 0 } }, orderBy: { sequence_no: 'asc' } } }, orderBy: { due_date: 'asc' } });
    const lineIds = docs.flatMap((d) => d.tb_ap_invoice_detail.map((l) => l.id));
    const reserved = await this.prismaService.tb_ap_payment_detail.groupBy({ by: ['ap_invoice_detail_id'], _sum: { applied_amount: true }, where: { ap_invoice_detail_id: { in: lineIds }, tb_ap_payment: { doc_status: { in: ['draft', 'in_review'] }, deleted_at: null } } });
    const held = new Map(reserved.map((r) => [r.ap_invoice_detail_id, r._sum.applied_amount ?? new Prisma.Decimal(0)]));
    return Result.ok(docs.map((d) => ({
      ap_invoice_id: d.id, doc_no: d.doc_no, doc_type: d.doc_type, doc_date: d.doc_date, due_date: d.due_date, currency_code: d.currency_code, exchange_rate: Number(d.exchange_rate), outstanding_amount: Number(d.outstanding_amount),
      lines: d.tb_ap_invoice_detail.map((l) => ({ ap_invoice_detail_id: l.id, sequence_no: l.sequence_no, description: l.description, total_amount: Number(l.total_amount), unpaid_amount: Number(l.unpaid_amount), available_amount: Number(l.unpaid_amount.sub((held.get(l.id) ?? new Prisma.Decimal(0)).abs())), wht_tax_profile_id: l.wht_tax_profile_id })),
    })));
  }

  /** Default WHT rows for a set of allocations (spec §7.2) / WHT default @param allocations Allocations @returns Rows */
  @TryCatch
  async whtPreview(allocations: { ap_invoice_detail_id: string; applied_amount: number | string }[]): Promise<Result<unknown>> {
    this.logger.debug({ function: 'whtPreview', count: allocations.length }, ApPaymentService.name);
    const lines = await loadAllocationLines(this.prismaService, allocations, null);
    if (lines.isError()) return Result.error(lines.error);
    return Result.ok(computeWhtRows(lines.value).map((r) => ({ ...r, wht_rate: Number(r.wht_rate), base_amount: Number(r.base_amount), wht_amount: Number(r.wht_amount) })));
  }

  /** Active ap_payment workflow or null / workflow ที่ active หรือ null */
  protected async resolveWorkflowId(): Promise<string | null> {
    const workflow = await this.prismaService.tb_workflow.findFirst({ where: { workflow_type: enum_workflow_type.ap_payment, is_active: true, deleted_at: null }, select: { id: true }, orderBy: { created_at: 'asc' } });
    return workflow?.id ?? null;
  }
}
```

- [ ] **Step 3: check + commit**

```bash
bun run check-types && npx eslint --no-fix apps/micro-business/src/ap/ap-payment
git add apps/micro-business/src/ap/ap-payment
git commit -m "feat(ap): payment voucher CRUD, outstanding documents and WHT preview

Spec §7.1–7.2"
```

---

### Task 14: AP payment — posting, void, workflow actions

**Files:**
- Create: `apps/micro-business/src/ap/ap-payment/ap-payment.posting.ts`
- Modify: `apps/micro-business/src/ap/ap-payment/ap-payment.service.ts`

**Interfaces:**
- Produces: `ApPaymentPosting.{postInTx, voidPostedInTx, lockInvoiceLines}`, `ApPaymentService.{submit, approve, review, reject, void}`

- [ ] **Step 1: posting helper** (`ap-payment.posting.ts`) — Review Focus #4 ล็อกบรรทัดด้วย FOR UPDATE

```ts
import { Injectable } from '@nestjs/common';
import { Result } from '@/common';
import { ERROR_CATALOG } from '@repo/error-catalog';
import { enum_ap_payment_status, enum_gl_jv_source, Prisma, tenantTableRef } from '@repo/prisma-shared-schema-tenant';
import { TenantScopedService } from '@/common/tenant-scoped.service';
import { GlSubledgerPostingService } from '@/gl/gl-subledger-posting/gl-subledger-posting.service';
import { GlPostingService } from '@/gl/gl-posting/gl-posting.service';
import { ApInvoicePosting } from '../ap-invoice/ap-invoice.posting';
import { round2 } from '../ap-invoice/ap-invoice.logic';
import { buildPaymentJvLines, computePaymentTotals } from './ap-payment.logic';
import { loadAllocationLines } from './ap-payment.validation';
import type { ApPaymentFull } from './ap-payment.service';

/**
 * Transaction-scoped post/void of a payment voucher (spec §7.3–7.4, §8)
 * post/void ของใบจ่ายเงินภายใน transaction (spec §7.3–7.4, §8)
 */
@Injectable()
export class ApPaymentPosting extends TenantScopedService {
  constructor(
    private readonly subledger: GlSubledgerPostingService,
    private readonly glPosting: GlPostingService,
    private readonly invoicePosting: ApInvoicePosting,
  ) {
    super();
  }

  /**
   * Row-lock the invoice lines being paid so two vouchers cannot spend the same unpaid amount
   * ล็อกแถวบรรทัด invoice ที่จ่าย เพื่อไม่ให้สองใบจ่ายยอดค้างเดียวกัน
   * @param tx - Transaction / transaction
   * @param lineIds - Invoice line ids / รหัสบรรทัด
   */
  async lockInvoiceLines(tx: Prisma.TransactionClient, lineIds: string[]): Promise<void> {
    if (lineIds.length === 0) return;
    const table = Prisma.raw(tenantTableRef(this.prismaService, 'tb_ap_invoice_detail'));
    await tx.$queryRaw`SELECT id FROM ${table} WHERE id IN (${Prisma.join(lineIds)}) FOR UPDATE`;
  }

  /**
   * Post: re-check availability under lock, post JV, reduce unpaid, mark posted
   * post: ตรวจยอดคงเหลือใหม่ภายใต้ lock, post JV, ลดยอดค้าง, ตั้งสถานะ posted
   * @param tx - Transaction / transaction
   * @param pv - Payment loaded inside tx / ใบจ่ายที่โหลดใน tx
   * @returns `{ id, doc_status, gl_jv_no }` / ผลลัพธ์
   */
  async postInTx(tx: Prisma.TransactionClient, pv: ApPaymentFull): Promise<Result<unknown>> {
    if (!(await this.invoicePosting.isPeriodOpen(tx, pv.payment_date))) return Result.errorFromCatalog(ERROR_CATALOG.GL_PERIOD_NOT_OPEN);
    await this.lockInvoiceLines(tx, pv.tb_ap_payment_detail.map((d) => d.ap_invoice_detail_id));
    const lines = await loadAllocationLines(tx, pv.tb_ap_payment_detail.map((d) => ({ ap_invoice_detail_id: d.ap_invoice_detail_id, applied_amount: d.applied_amount.toString() })), pv.id);
    if (lines.isError()) return Result.error(lines.error);
    const setting = await this.glPosting.getSetting();
    const gain = this.subledger.requireSettingAccount(setting, 'realized_fx_gain_account_id');
    const loss = this.subledger.requireSettingAccount(setting, 'realized_fx_loss_account_id');
    if (gain.isError()) return Result.error(gain.error);
    if (loss.isError()) return Result.error(loss.error);
    const bank = await tx.tb_bank_account.findFirst({ where: { id: pv.bank_account_id }, select: { chart_of_accounts_id: true } });
    if (!bank) return Result.errorFromCatalog(ERROR_CATALOG.BANK_ACCOUNT_NOT_FOUND);
    const totals = computePaymentTotals(lines.value, pv.tb_ap_payment_wht, pv.tb_ap_payment_expense, pv.exchange_rate);
    const jvLines = buildPaymentJvLines({ currency_id: pv.currency_id, exchange_rate: pv.exchange_rate, base_currency_id: pv.base_currency_id, bank_account_coa_id: bank.chart_of_accounts_id, fx_gain_account_id: gain.value, fx_loss_account_id: loss.value }, lines.value, pv.tb_ap_payment_wht, pv.tb_ap_payment_expense, totals);
    const posted = await this.subledger.postFromSource({ source: enum_gl_jv_source.ap, source_ref_type: 'ap_payment', source_ref_id: pv.id, jv_date: pv.payment_date, description: `${pv.doc_no} ${pv.vendor_name}`, currency_id: pv.currency_id, exchange_rate: pv.exchange_rate, lines: jvLines }, tx);
    if (posted.isError()) return Result.error(posted.error);
    for (const l of lines.value) {
      await tx.tb_ap_invoice_detail.update({ where: { id: l.ap_invoice_detail_id }, data: { unpaid_amount: { decrement: l.applied_amount }, base_unpaid_amount: { decrement: round2(l.applied_amount.mul(l.invoice_exchange_rate)) } } });
    }
    for (const invoiceId of new Set(lines.value.map((l) => l.ap_invoice_id))) {
      const outstanding = await this.invoicePosting.recomputeOutstanding(tx, invoiceId);
      await tx.tb_ap_invoice.update({ where: { id: invoiceId }, data: { outstanding_amount: outstanding, updated_at: new Date(), updated_by_id: this.userId } });
    }
    await tx.tb_ap_payment.update({ where: { id: pv.id }, data: { doc_status: enum_ap_payment_status.posted, gl_jv_id: posted.value.jv_id, gl_jv_no: posted.value.jv_no, posted_at: new Date(), posted_by_id: this.userId, doc_version: { increment: 1 }, updated_at: new Date(), updated_by_id: this.userId } });
    return Result.ok({ id: pv.id, doc_status: enum_ap_payment_status.posted, gl_jv_no: posted.value.jv_no });
  }

  /**
   * Void a posted voucher: reversal JV, give unpaid back, mark void (spec §7.3)
   * void ใบที่ post: JV กลับรายการ, คืนยอดค้าง, ตั้งสถานะ void (spec §7.3)
   * @param tx - Transaction / transaction
   * @param pv - Payment / ใบจ่าย
   * @param reason - Reason / เหตุผล
   * @returns `{ id, doc_status }` / ผลลัพธ์
   */
  async voidPostedInTx(tx: Prisma.TransactionClient, pv: ApPaymentFull, reason: string): Promise<Result<unknown>> {
    const now = new Date();
    if (!(await this.invoicePosting.isPeriodOpen(tx, now))) return Result.errorFromCatalog(ERROR_CATALOG.GL_PERIOD_NOT_OPEN);
    await this.lockInvoiceLines(tx, pv.tb_ap_payment_detail.map((d) => d.ap_invoice_detail_id));
    const reversed = await this.subledger.reverseBySource({ source_ref_type: 'ap_payment', source_ref_id: pv.id }, now, reason, tx);
    if (reversed.isError()) return Result.error(reversed.error);
    for (const d of pv.tb_ap_payment_detail) {
      await tx.tb_ap_invoice_detail.update({ where: { id: d.ap_invoice_detail_id }, data: { unpaid_amount: { increment: d.applied_amount }, base_unpaid_amount: { increment: d.base_applied_at_invoice_rate } } });
    }
    for (const invoiceId of new Set(pv.tb_ap_payment_detail.map((d) => d.ap_invoice_id))) {
      const outstanding = await this.invoicePosting.recomputeOutstanding(tx, invoiceId);
      await tx.tb_ap_invoice.update({ where: { id: invoiceId }, data: { outstanding_amount: outstanding, updated_at: now, updated_by_id: this.userId } });
    }
    await tx.tb_ap_payment.update({ where: { id: pv.id }, data: { doc_status: enum_ap_payment_status.void, void_at: now, void_by_id: this.userId, void_reason: reason, void_gl_jv_id: reversed.value.jv_id, doc_version: { increment: 1 }, updated_at: now, updated_by_id: this.userId } });
    return Result.ok({ id: pv.id, doc_status: enum_ap_payment_status.void });
  }
}
```

- [ ] **Step 2: actions ใน `ApPaymentService`** — เพิ่ม constructor param `private readonly posting: ApPaymentPosting`, import `apPaymentToWorkflowDocument`, และ method เหมือน Task 10 Step 2 โดยเปลี่ยน:
  - `loadForAction` ใช้ `enum_ap_payment_status` และ `AP_PAYMENT_NOT_FOUND` / `AP_PAYMENT_IMMUTABLE`
  - `postDocument(id, expectedVersion)`: lock header ด้วย `tb_ap_payment.updateMany({ where: { id, doc_version, doc_status: { in: [draft, in_review] } } })` → `loadFull(tx, id)` → `this.posting.postInTx(tx, pv)` (timeout 20000)
  - `submit`: ตรวจ `isPeriodOpen(payment_date)` + ตรวจ availability ผ่าน `loadAllocationLines(this.prismaService, allocations, id)` (จองยอดตั้งแต่ submit) + ไม่มี tax record; ไม่มี workflow → `postDocument`; มี → `buildSubmitWorkflow(apPaymentToWorkflowDocument(...))`
  - `approve`/`review`/`reject`: เหมือน invoice ใช้ `tb_ap_payment`
  - `void`: draft → void ตรง; posted → tx lock header → `this.posting.voidPostedInTx(tx, fresh, reason)`
  - `workflowColumns` เหมือนเดิม

- [ ] **Step 3: check + commit**

```bash
bun run check-types && npx eslint --no-fix apps/micro-business/src/ap/ap-payment
git add apps/micro-business/src/ap/ap-payment
git commit -m "feat(ap): payment voucher posting with row locks, void with reversal, workflow actions

Spec §7.3–7.4, §8"
```

---

### Task 15: AP payment — controller, module, contract, registry, gateway, ตรวจด้วยมือ

**Files:**
- Create: `apps/micro-business/src/ap/ap-payment/ap-payment.controller.ts`, `ap-payment.module.ts`
- Modify: `apps/micro-business/src/app.module.ts`, `apps/micro-business/src/common/activity/activity-registry.ts`
- Generated: `packages/rpc-contract/src/contracts/ap-payment.ts` (`ApPayment`)
- Create: `apps/backend-gateway/src/application/ap-payment/{controller,service,module,swagger/request.ts,swagger/response.ts}`
- Modify: `apps/backend-gateway/src/application/route-application.ts`

- [ ] **Step 1: controller** — คัดลอก `ap-invoice.controller.ts` (Task 11 Step 1) เปลี่ยน class `ApPaymentController`, service `ApPaymentService`, cmd prefix `ap-payment.`; ตัด handler `grn-candidates`, `create-from-grn`, `reference-candidates` ออก; เพิ่ม:
```ts
  /** Outstanding documents of a vendor / เอกสารค้างจ่ายของ vendor */
  @MessagePattern({ cmd: 'ap-payment.outstanding-documents', service: SERVICE })
  async outstandingDocuments(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'outstandingDocuments', payload }, ApPaymentController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.outstandingDocuments(payload.vendor_id)));
  }

  /** WHT preview / คำนวณ WHT ล่วงหน้า */
  @MessagePattern({ cmd: 'ap-payment.wht-preview', service: SERVICE })
  async whtPreview(@Payload() payload: MicroservicePayload): Promise<MicroserviceResponse> {
    this.logger.debug({ function: 'whtPreview', payload }, ApPaymentController.name);
    return this.handleResult(await this.ctx.run(payload, () => this.service.whtPreview(payload.data?.allocations ?? [])));
  }
```

- [ ] **Step 2: module**

```ts
@Module({
  imports: [TenantModule, CommonModule, ExchangeRateModule, GlPostingModule, GlSubledgerPostingModule, ApInvoiceModule],
  controllers: [ApPaymentController],
  providers: [ApPaymentService, ApPaymentWriter, ApPaymentPosting, ApInvoicePosting, WorkflowOrchestratorService],
})
export class ApPaymentModule {}
```
(`ApInvoicePosting` ต้อง provide ที่นี่ด้วยหรือ export จาก `ApInvoiceModule` — เลือกอย่างหลัง: เพิ่ม `ApInvoicePosting` ใน `exports` ของ `ApInvoiceModule` แล้วไม่ต้องใส่ใน providers นี้)
`app.module.ts`: `ApPaymentModule,` หลัง `ApInvoiceModule,`

- [ ] **Step 3: activity registry** — เพิ่มใน `DOCUMENT_ENTITIES`:
```ts
  {
    entityName: 'tb_ap_payment',
    mutations: [
      ['ap-payment.create', 'create', CREATED_ID],
      ['ap-payment.update', 'update', EDITED_ID],
      ['ap-payment.submit', 'submit', EDITED_ID],
      ['ap-payment.approve', 'approve', EDITED_ID],
      ['ap-payment.review', 'review', EDITED_ID],
      ['ap-payment.reject', 'reject', EDITED_ID],
      ['ap-payment.void', 'update', EDITED_ID],
      ['ap-payment.delete', 'delete', DELETED_ID],
    ],
  },
```

- [ ] **Step 4: generate + gateway**

```bash
bun run gen:rpc-contract && bun run build:package
```
แทน literal ด้วย `ApPayment.<action>.pattern` (12 handler)

gateway `application/ap-payment/`: โครงเดียวกับ `application/ap-invoice/` (Task 11 Step 5–6) โดย:

`swagger/request.ts`
```ts
const Allocation = z.object({ ap_invoice_detail_id: z.string().uuid(), applied_amount: z.coerce.number().refine((n) => n !== 0, 'applied_amount must not be 0') });
const Wht = z.object({ wht_tax_profile_id: z.string().uuid(), base_amount: z.coerce.number().min(0), wht_amount: z.coerce.number().min(0), is_override: z.boolean().optional() });
const Expense = z.object({ chart_of_accounts_id: z.string().uuid(), cost_center_id: z.string().uuid().nullable().optional(), description: z.string().nullable().optional(), amount: z.coerce.number().positive() });
export const ApPaymentCreateRequestSchema = z.object({
  vendor_id: z.string().uuid(), payment_date: z.coerce.date(), paid_date: z.coerce.date().nullable().optional(), bank_account_id: z.string().uuid(),
  payment_method: z.nativeEnum(enum_ap_payment_method).optional(), cheque_no: z.string().max(50).nullable().optional(), cheque_date: z.coerce.date().nullable().optional(),
  reference_no: z.string().max(100).nullable().optional(), description: z.string().nullable().optional(), exchange_rate: z.coerce.number().positive().nullable().optional(),
  attachments: z.array(z.object({ originalName: z.string(), fileToken: z.string(), contentType: z.string() })).nullable().optional(),
  allocations: z.array(Allocation).min(1), wht: z.array(Wht).nullable().optional(), expenses: z.array(Expense).nullable().optional(),
});
export class ApPaymentCreateRequestDto extends createZodDto(ApPaymentCreateRequestSchema) {}
export const ApPaymentUpdateRequestSchema = ApPaymentCreateRequestSchema.omit({ vendor_id: true }).partial().extend({ doc_version: z.number().int() });
export class ApPaymentUpdateRequestDto extends createZodDto(ApPaymentUpdateRequestSchema) {}
export const ApPaymentActionRequestSchema = z.object({ doc_version: z.number().int().optional(), reason: z.string().nullable().optional(), stage: z.string().nullable().optional() });
export class ApPaymentActionRequestDto extends createZodDto(ApPaymentActionRequestSchema) {}
export const ApPaymentWhtPreviewRequestSchema = z.object({ allocations: z.array(Allocation).min(1) });
export class ApPaymentWhtPreviewRequestDto extends createZodDto(ApPaymentWhtPreviewRequestSchema) {}
```
routes: `GET /outstanding-documents?vendor_id`, `POST /wht-preview`, CRUD, `submit/approve/review/reject/void` (`void` มี `@Permission({ 'accounting.ap': ['void'] })`); `@Controller('api/:bu_code/ap-payment')`, `@ApiTags('Accounting: AP Payment')`, AppIdGuard prefix `ap-payment.`
`route-application.ts`: `ApPaymentModule,` หลัง `ApInvoiceModule,`

- [ ] **Step 5: check + gates + commit**

```bash
bun run build:package && bun run check-types
npx eslint --no-fix apps/micro-business/src/ap apps/micro-business/src/common/activity/activity-registry.ts apps/backend-gateway/src/application/ap-payment
bun run gates
git add -A apps/micro-business/src/ap apps/micro-business/src/app.module.ts apps/micro-business/src/common/activity packages/rpc-contract apps/backend-gateway/src/application
git commit -m "feat(ap): expose payment voucher over RPC and REST with activity logging"
```

- [ ] **Step 6: ตรวจด้วยมือ — payment flow** (spec §13 ข้อ 7–9)

```bash
# a. outstanding ของ vendor → ต้องเห็น invoice ที่ post ใน Task 11 พร้อม lines/available_amount
curl -s "$API/ap-payment/outstanding-documents?vendor_id=<VENDOR>" "${H[@]}"
# b. wht-preview จ่ายบรรทัดที่มี WHT profile 3% → base = applied×net/total, wht = 3%
curl -s -X POST $API/ap-payment/wht-preview "${H[@]}" -d '{"allocations":[{"ap_invoice_detail_id":"<LINE>","applied_amount":500}]}'
# c. สร้าง PV จ่ายบางบรรทัด + wht จาก preview + expense ค่าธรรมเนียม 20 → 201 doc_no APPV2609xxxx
# d. สร้าง PV ใบที่ 2 จ่ายบรรทัดเดิมเกิน available → 422 AP_PAYMENT_OVER_ALLOCATED
# e. submit/approve PV ใบแรก → posted; ตรวจ: invoice line unpaid ลด, outstanding ลด, JV มี Dr AP / Cr Bank / Cr WHT / Dr expense (+FX ถ้า rate ต่าง) และ base สมดุล
# f. findOne PV → tax_invoices[] มีรายการของ invoice ที่จ่าย
# g. void PV → void; unpaid/outstanding คืน; มี JV reversal source_ref_type ap_payment
# h. void invoice ที่มี PV posted (ก่อน g) → 409 AP_INVOICE_HAS_PAYMENT
# i. ปิด period ปัจจุบัน (config gl-periods) แล้ว approve PV ที่ค้าง → 422 GL_PERIOD_NOT_OPEN
```

---

### Task 16: เปิดฟิลด์ใหม่ของ tax profile และ vendor ผ่าน API

**Files:**
- Modify: `apps/micro-business/src/master/tax_profile/dto/tax-profile.dto.ts` (Create/Update schema)
- Modify: `apps/micro-business/src/master/tax_profile/dto/tax-profile.serializer.ts` (response)
- Modify: `apps/micro-business/src/master/tax_profile/tax_profile.service.ts` (create/update `data:`)
- Modify: `apps/backend-gateway/src/common/dto/tax-profile/tax-profile.dto.ts`
- Modify: `apps/backend-gateway/src/config/config_tax-profiles/swagger/request.ts` (+ response.ts ถ้ามี field list)
- Modify: `apps/micro-business/src/master/vendors/dto/vendors.dto.ts`, `dto/vendor.serializer.ts`, `vendors.service.ts`
- Modify: `apps/backend-gateway/src/common/dto/vendor/vendor.create.dto.ts`, `vendor.update.dto.ts`, `vendor.serializer.ts`

**Interfaces:**
- Produces: tax profile API รับ/คืน `tax_type, chart_of_accounts_id, wht_pnd_form, wht_income_type`; vendor API รับ/คืน `default_currency_id, default_currency_code, credit_term_id, credit_term_name, credit_term_days, ap_chart_of_accounts_id`

- [ ] **Step 1: tax profile — schema ทั้ง micro-business และ gateway** (ทั้งสองไฟล์ `tax-profile.dto.ts` มีเนื้อหาเหมือนกัน แก้ทั้งคู่)

เพิ่ม import และฟิลด์ใน `TaxProfileCreateSchema` และ `TaxProfileUpdateSchema`:
```ts
import { enum_tax_profile_tax_type, enum_tax_profile_wht_pnd_form } from '@repo/prisma-shared-schema-tenant';
// ใน object ของทั้ง Create และ Update:
  tax_type: z.nativeEnum(enum_tax_profile_tax_type).optional(),
  chart_of_accounts_id: z.string().uuid().nullable().optional(),
  wht_pnd_form: z.nativeEnum(enum_tax_profile_wht_pnd_form).nullable().optional(),
  wht_income_type: z.string().trim().nullable().optional(),
```
gateway `config_tax-profiles/swagger/request.ts`: ถ้าประกาศ class DTO ด้วย `@ApiProperty` แยก ให้เพิ่ม 4 property พร้อม `@ApiPropertyOptional({ enum: enum_tax_profile_tax_type, enumName: 'enum_tax_profile_tax_type' })` ฯลฯ

- [ ] **Step 2: tax profile — service และ serializer**

`tax_profile.service.ts` create (`data: {` ~L206) และ update (`data: {` ใน `async update`) เพิ่ม:
```ts
        tax_type: data.tax_type,
        chart_of_accounts_id: data.chart_of_accounts_id,
        wht_pnd_form: data.wht_pnd_form,
        wht_income_type: data.wht_income_type,
```
`tax-profile.serializer.ts` ทั้ง 2 schema (L12 และ L41 บริเวณ `tax_rate: decimalFieldRequired`) เพิ่ม:
```ts
  tax_type: z.nativeEnum(enum_tax_profile_tax_type).optional(),
  chart_of_accounts_id: z.string().nullable().optional(),
  wht_pnd_form: z.nativeEnum(enum_tax_profile_wht_pnd_form).nullable().optional(),
  wht_income_type: z.string().nullable().optional(),
```
และที่ findOne/findAll ซึ่ง map ฟิลด์ทีละตัว (L53, L85, L132, L176 `tax_rate: Number(...)`) เพิ่ม 4 ฟิลด์ pass-through ในแต่ละจุด

- [ ] **Step 3: vendor — schema, service, serializer**

`vendors.dto.ts` `VendorCreateSchema` (หลัง `branch_no` L54) และ `VendorUpdateSchema` (หา `branch_no` อีกจุด) เพิ่ม:
```ts
  default_currency_id: z.string().uuid().nullable().optional(),
  default_currency_code: z.string().max(3).nullable().optional(),
  credit_term_id: z.string().uuid().nullable().optional(),
  credit_term_name: z.string().nullable().optional(),
  credit_term_days: z.number().int().min(0).nullable().optional(),
  ap_chart_of_accounts_id: z.string().uuid().nullable().optional(),
```
gateway `vendor.create.dto.ts` (L22) และ `vendor.update.dto.ts` (L24): เพิ่มชุดเดียวกัน
`vendors.service.ts`: ทุก `select: { ... branch_no: true` (L144 และจุดอื่นจาก `grep -n "branch_no: true"`) เพิ่ม 6 ฟิลด์ `: true`; ทุก `data: { ... branch_no: data.branch_no` ใน create/update (`grep -n "branch_no: data" apps/micro-business/src/master/vendors/vendors.service.ts`) เพิ่ม 6 ฟิลด์ `x: data.x`
`vendor.serializer.ts` ทั้ง micro-business และ gateway: เพิ่ม 6 ฟิลด์ `z.string().nullable().optional()` / `credit_term_days: z.number().nullable().optional()`

- [ ] **Step 4: check + commit**

```bash
bun run check-types
npx eslint --no-fix apps/micro-business/src/master/tax_profile apps/micro-business/src/master/vendors apps/backend-gateway/src/common/dto/tax-profile apps/backend-gateway/src/common/dto/vendor apps/backend-gateway/src/config/config_tax-profiles
(cd apps/micro-business && bun run test -- tax_profile vendors 2>/dev/null || true)   # spec เดิมของ module เหล่านี้ต้องยังผ่าน ถ้า fail เพราะ snapshot field ใหม่ ให้อัปเดต spec ให้ตรง
git add apps/micro-business/src/master/tax_profile apps/micro-business/src/master/vendors apps/backend-gateway/src/common/dto apps/backend-gateway/src/config/config_tax-profiles
git commit -m "feat(master): expose tax_type/WHT fields on tax profile and AP defaults on vendor

Spec §4.2, §5.4"
```

- [ ] **Step 5: ตรวจด้วยมือ** — สร้าง tax profile `{ name: 'WHT 3% Service', tax_rate: 3, tax_type: 'wht', wht_pnd_form: 'pnd_53', wht_income_type: 'ค่าบริการ' }` ผ่าน `POST /api/config/$BU/tax-profiles` แล้ว GET กลับต้องเห็นฟิลด์; แก้ vendor ทดสอบให้มี `credit_term_days: 30`, `default_currency_id` แล้วสร้าง AP invoice โดยไม่ส่ง `credit_term_days` → `due_date = invoice_date + 30`

---

### Task 17: Gates, Bruno collection, ตรวจรวม, PR

**Files:**
- Create: `../carmen-turborepo-backend-bruno/<collection>/ap-invoice/*.bru`, `ap-payment/*.bru`, `gl-dimension/*.bru`, `bank-account/*.bru` (ตามโครงโฟลเดอร์ที่ collection ใช้อยู่ ดู `ls ../carmen-turborepo-backend-bruno`)
- Modify: `apps/micro-business/CLAUDE.md` (สร้างใหม่ถ้ายังไม่มี) — gotchas ของ AP

- [ ] **Step 1: gates ทั้งชุด**

```bash
bun run build:package && bun run check-types && bun run gates
```
Expected: ทุก audit ผ่าน (`tcp-drift`, `message-pattern-literal` ต้องไม่เหลือ literal ชั่วคราว, `rest-contract`, `guard-providers`, `bu-scope-guard`, `api-system-permission`, `tenant-context`, `raw-sql` — ถ้า `audit:raw-sql` เตือน `FOR UPDATE` ใน `ap-payment.posting.ts` ให้ดูว่า PO service ใช้ข้อยกเว้นแบบใด (`grep -n "raw-sql" scripts/*.sh scripts/**/*.ts`) แล้วใส่ comment/allowlist แบบเดียวกัน)

- [ ] **Step 2: Bruno** — เพิ่ม request ต่อ endpoint ใน §9.2 ของ spec (แต่ละไฟล์ `.bru` มี method/url/headers `Authorization`, `x-app-id`, body ตัวอย่างจาก swagger request ของ task ที่เกี่ยวข้อง) อย่างน้อย: gl-dimensions CRUD, gl-dimension-values create, gl-account-dimension-rules create, bank-accounts create, ap-invoice grn-candidates/from-grn/create/submit/approve/void/reference-candidates, ap-payment outstanding-documents/wht-preview/create/submit/approve/void
```bash
cd ../carmen-turborepo-backend-bruno && git checkout -b feature/accounting-ap && git add . && git commit -m "feat: add AP invoice/payment, GL dimension and bank account requests" && cd -
```

- [ ] **Step 3: `apps/micro-business/CLAUDE.md`** — บันทึก gotchas ที่เจอจริงระหว่างทำ อย่างน้อย:
```markdown
## Accounting (AP / GL subledger)
- AP never writes `tb_gl_jv_*` directly — go through `GlSubledgerPostingService.postFromSource` inside your own `$transaction`; it is idempotent per `(source, source_ref_type, source_ref_id)`.
- `GlPostingService.post()` is a thin wrapper over `postInTx()`; never open a nested `$transaction` from a facade caller.
- AP tables are excluded from the Prisma audit extension; every new `ap-*` @MessagePattern must be registered in `activity-registry.ts` or it leaves no activity trail.
- Base-currency amounts on AP lines are rounded per line and passed explicitly (`base_debit/base_credit`); do not let GL recompute them from the rate.
- New `gl_setting` keys required before posting AP: ap_control_account_id, input_vat_account_id, wht_payable_account_id, advance_deposit_account_id, realized_fx_gain_account_id, realized_fx_loss_account_id, ap_jv_prefix_id.
```

- [ ] **Step 4: ตรวจรวมครั้งสุดท้าย** — รันลำดับ 9 ขั้นใน spec §13 ตั้งแต่ต้นบน BU ทดสอบที่ migrate ใหม่ (`POST /api-system/tenant/migrations/:bu_id/deploy`) แล้วบันทึกผล (doc_no, jv_no, error code) ลง PR description

- [ ] **Step 5: commit + PR**

```bash
git add apps/micro-business/CLAUDE.md
git commit -m "docs(micro-business): add accounting AP gotchas"
git push -u origin feature/accounting-foundation-ap
gh pr create --base main --title "feat(accounting): dimension foundation, subledger posting facade, AP invoice + payment" --body-file - <<'EOF'
## Summary
- Accounting dimension foundation (definitions, values, account rules, JV junction) — spec §5.1
- `GlSubledgerPostingService` facade with idempotent post/reverse; `GlPostingService.postInTx/reverseInTx` refactor — spec §5.2–5.3
- AP invoice / debit note / credit note / deposit with GRN linkage, doc references, tax invoice record — spec §6
- AP payment voucher with line-level allocation, WHT at payment, other expenses, realized FX — spec §7
- Bank account master; tax profile `tax_type`/WHT fields; vendor AP defaults — spec §4.2, §5.4, §5.5

Spec: carmen-accounting-concept `docs/superpowers/specs/2026-09-23-accounting-foundation-ap-design.md`
Plan: carmen-accounting-concept `docs/superpowers/plans/2026-09-23-accounting-foundation-ap.md`

## Manual verification
(paste results of spec §13 steps 1–9 here)

## Migration
Tenant migration `accounting_foundation_ap` — deploy per BU via tenant migration API; set new `gl_setting` keys before posting AP.
EOF
```

---

## Self-review (ทำแล้ว)

- **Spec coverage**: §4 → Task 1–2; §5.1 → Task 4; §5.2–5.3 → Task 6–7; §5.4 → Task 16; §5.5 → Task 5; §6 → Task 8–11; §7 → Task 12–15; §8 → Task 8/12 logic; §9 → Task 4, 5, 11, 15; §10 → Task 3; §11 → Task 7/10/14 (locks, doc_version) + Task 2/11/15 (audit); §12 → Task 2; §13 → Task 7/11/15/17
- **Placeholder scan**: ไม่มี TBD/TODO; จุดที่ให้ "คัดลอกไฟล์แล้วเปลี่ยนชื่อ" (Task 4 Step 5/9, Task 5 Step 3–4, Task 12 Step 4/6, Task 15 Step 1) ระบุทุกสิ่งที่ต่างเป็นตาราง/รายการชัดเจน
- **Type consistency**: `postInTx(tx, header, setting, postAt)` ใช้เหมือนกันใน Task 6 และ 7; `ISubledgerPostingLine.base_debit/base_credit` เป็น required ตรงกับ `toJvLine`; `IAllocationLine` ใช้ร่วม Task 12–14; `ApInvoiceFull`/`ApPaymentFull` export จาก service และ import แบบ `type` ใน posting
- **Review Focus**: #1 → Task 8 `computeLine` (base_total = base_net + base_vat) และ Task 12 หมายเหตุ rounding ต่อแถว; #2 → Task 7 idempotency + Task 10/14 `updateMany` lock ด้วย doc_version; #3 → Task 10 `checkGrnSources` ทั้ง submit และ post; #4 → Task 14 `lockInvoiceLines` + `loadAllocationLines` ใน tx; #5 → Task 10 `voidPostedInTx` ตรวจ payment/reference ก่อน reverse
