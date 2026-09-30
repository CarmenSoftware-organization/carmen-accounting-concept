# Design — AR Schema + Migration (schema-only)

**วันที่:** 2026-09-30
**repo ที่แก้:** `carmen-turborepo-backend-v2` (branch `feature/accounting-ar-schema` จาก `main`)
**ต่อจาก:** [2026-09-23-accounting-foundation-ap-design.md](./2026-09-23-accounting-foundation-ap-design.md) (foundation + AP, merged PR #671)
**แหล่งอ้างอิง:** `docs/PRD-module-ar.md` v1.1 (FRD v1.07), draft schema `prisma/schema.prisma` §4 AR ใน repo นี้ (PR #1), review `docs/reviews/2026-09-28-ar-frd-v1.07-review.md`

---

## 1. ที่มาและการตัดสินใจหลัก

- ผู้ใช้ต้องการ "real prisma database" สำหรับ AR ในขอบเขต **schema + migration อย่างเดียว** ไม่มี service/RPC/gateway (ตัดสิน 2026-09-30)
- ทำเป็น **migration เดียว** `accounting_ar_tables` (customer master + AR documents + receipt) ตาม pattern `accounting_foundation_ap` (แนวทาง A)
- โครงตารางยึด draft ที่ merge แล้วใน concept repo PR #1 ทุก field รวมจุดที่ยังรอ BA (§1.1) — ตารางยังไม่มีข้อมูล การแก้ภายหลังเป็น migration ราคาถูก
- ไม่ข้ามกลไก deploy ของ repo: migration ถูกยิงราย BU หลัง merge ด้วย tenant-migrations endpoint ไม่ใช่ใน PR

### 1.1 จุดที่ตัดสินไปก่อน BA (เดิมติดป้าย OI ใน draft)

| OI | เรื่อง | ตัดสินใน schema นี้ |
|---|---|---|
| OI-3 | จุด post GL / ชุดสถานะ | `enum_ar_invoice_status = draft, in_review, posted, void` แบบ AP; workflow stage อยู่ใน `workflow_*` |
| OI-4 | Receipt (ARRC) | ตารางแยก `tb_ar_receipt` + `_detail` + `_wht` mirror `tb_ap_payment` |
| OI-5 | source / tax prefix | `enum_ar_invoice_source = manual, pms_folio, copy, ai`; `tb_ar_tax_invoice.tax_prefix` เป็น VarChar รองรับ TXRC |
| OI-6 | WHT ที่ invoice | เก็บ `tax2_*` บนบรรทัด + `is_wht_recorded`/`wht_amount` บน header เป็น reference; ไม่กำหนดพฤติกรรม GL ใน schema |
| OI-7 | customer master | สร้าง `tb_customer` + `tb_customer_address` mirror `tb_vendor` + `tb_vendor_address` |
| OI-1/OI-2 | FX ของ deposit | `tb_ar_invoice_reference.realized_fx_amount` มีเครื่องหมาย (> 0 = loss) รอ BA กำหนดสูตรตอนทำ service |

ถ้า BA ตัดสินต่างจากนี้ ให้ออก migration แก้ (ALTER) ใน spec ของ AR service ไม่ย้อนแก้ migration นี้

---

## 2. ขอบเขต

### ในขอบเขต

- 7 enum + 10 model ใน tenant `schema.prisma` (ภาคผนวก A คือข้อความที่ใช้จริง)
- back-relation 4 บรรทัดบน model เดิม: `tb_tax_profile.tb_customer`, `tb_gl_dimension.tb_ar_invoice_detail_dimension`, `tb_gl_dimension_value.tb_ar_invoice_detail_dimension`, `tb_bank_account.tb_ar_receipt`
- migration folder `prisma/migrations/<timestamp>_accounting_ar_tables/migration.sql` จาก `prisma migrate diff`
- `excludeModels` ใน `packages/prisma-shared-schema-tenant/src/client.ts` เพิ่ม 10 ชื่อ
- `apps/micro-business/CLAUDE.md` §Tenant migrations เพิ่ม 1 bullet
- concept repo หลัง merge: README แถว AR → "Schema only", `prisma/schema.prisma` ย้าย AR จาก DRAFT ไป IMPLEMENTED (commit แยก)

### นอกขอบเขต

service / RPC contract / gateway / permission / error catalog / activity registry · seed running code `AR-IV`, `AR-DN`, `AR-CN`, `AR-DP`, `AR-RC` · key ใหม่ใน `gl_setting` (`ar_control_account_id`, `output_vat_account_id`, `advance_deposit_received_account_id`, `wht_receivable_account_id`, `ar_jv_prefix_id`) · CHECK constraint · frontend · PMS interface · การยิง migration ลง BU จริง

---

## 3. ไฟล์ที่แตะ (backend-v2)

| ไฟล์ | การเปลี่ยนแปลง |
|---|---|
| `packages/prisma-shared-schema-tenant/prisma/schema.prisma` | enum 7 ตัวต่อท้าย `enum_ap_payment_method` (บรรทัด ~344); model 10 ตัวต่อท้าย `tb_ap_payment_expense` (บรรทัด ~7424); back-relation 4 บรรทัด |
| `packages/prisma-shared-schema-tenant/prisma/migrations/<ts>_accounting_ar_tables/migration.sql` | ใหม่ |
| `packages/prisma-shared-schema-tenant/src/client.ts` | `excludeModels` +10 |
| `apps/micro-business/CLAUDE.md` | +1 bullet |

ไม่แตะ `apps/*/src` อื่น

---

## 4. Schema

### 4.1 กติกา (ตาม pattern AP)

- ชนิด/ชื่อคอลัมน์ตาม AP: `Decimal(20,5)` ยอดเงิน, `Decimal(15,5)` rate, `doc_version`, audit 6 คอลัมน์, `deleted_at` soft delete บน header/detail/master, ไม่มีบน allocation/wht/dimension
- `@relation` เฉพาะ: ภายในชุด AR/customer ทั้งหมด · `tb_customer → tb_tax_profile` · `tb_ar_invoice_detail_dimension → tb_gl_dimension`, `tb_gl_dimension_value` · `tb_ar_receipt → tb_bank_account`
- id อย่างเดียว (ไม่มี relation) เหมือน AP: currency, base currency, chart of accounts, cost center, credit term, tax profile บนบรรทัด, unit, gl_jv
- `onDelete`: child ของเอกสาร = `Cascade` เฉพาะ allocation/wht/dimension/reference/tax; detail และ customer_address = `NoAction` (เหมือน `tb_ap_invoice_detail`, `tb_vendor_address`)
- unique/index ตั้งชื่อ `customer_*`, `customeraddress_*`, `arinvoice_*`, `arinvoicedetail_*`, `arinvoicedetaildimension_*`, `arinvoicereference_*`, `artaxinvoice_*`, `arreceipt_*`, `arreceiptdetail_*`, `arreceiptwht_*`
- ไม่มี `@@map`; ไม่มี comment ป้าย OI ในไฟล์จริง

### 4.2 รายการ

| # | model | ต้นแบบ | หมายเหตุ |
|---|---|---|---|
| 1 | `tb_customer` | `tb_vendor` | + `registered_name`, `credit_limit`, `ar_chart_of_accounts_id`, `pms_account_code`, `customer_group Json` |
| 2 | `tb_customer_address` | `tb_vendor_address` | enum `enum_customer_address_type` มี `billing_address` เพิ่ม |
| 3 | `tb_ar_invoice` | `tb_ap_invoice` | − `vendor_invoice_no`, `invoice_date`; + `source_doc_ref`, `is_pms_folio`, `is_tax_invoice`, `tax2_amount`, `is_wht_recorded`, `wht_amount` |
| 4 | `tb_ar_invoice_detail` | `tb_ap_invoice_detail` | + `group_no`, `reference_info`, `date_from/to`, folio 3 ช่อง, `discount_pct`, `discount_is_override`, `tax2_*` 6 ช่อง; `cr_cost_center_id` บังคับ |
| 5 | `tb_ar_invoice_detail_dimension` | `tb_ap_invoice_detail_dimension` | เหมือนกัน |
| 6 | `tb_ar_invoice_reference` | `tb_ap_invoice_reference` | + `applied_net_amount`, `applied_vat_amount`, `remarks` |
| 7 | `tb_ar_tax_invoice` | `tb_ap_invoice_tax` | FK `ar_invoice_id?` / `ar_receipt_id?` unique แยก (XOR บังคับที่ service); + ชื่อจดทะเบียน, ที่อยู่ 4 ช่อง, `tax_prefix`, `filing_month/year`, `deleted_at` |
| 8 | `tb_ar_receipt` | `tb_ap_payment` | `payee_name → payer_name`; `total_expense → total_charge`; `net_paid → net_received`; + `received_date`, `is_tax_invoice` |
| 9 | `tb_ar_receipt_detail` | `tb_ap_payment_detail` | `base_applied_at_payment_rate → _receipt_rate` |
| 10 | `tb_ar_receipt_wht` | `tb_ap_payment_wht` | + `wht_certificate_no`, `wht_certificate_date`; profile optional |

enum: `enum_ar_invoice_doc_type`, `enum_ar_invoice_status`, `enum_ar_invoice_source`, `enum_ar_tax_invoice_status`, `enum_ar_receipt_status`, `enum_ar_receipt_method`, `enum_customer_address_type`

ข้อความเต็มอยู่ในภาคผนวก A — implementer คัดลอกตรงตัว ไม่ต้องออกแบบใหม่

---

## 5. วิธีสร้างและตรวจ migration

ทำตาม `apps/micro-business/CLAUDE.md` §Tenant migrations และ skill `tenant-migrations` (dev DB มี drift ห้ามใช้ `migrate dev`)

1. สร้าง scratch database ใน Postgres local ด้วย connection string ที่พิมพ์เองทั้งเส้น (**ห้าม** เอา `.env` มาแก้ชื่อ database) และตรวจ `SELECT current_database(), current_schema()` ก่อนยิง DDL ใดๆ
2. `DATABASE_URL=<scratch> bunx prisma migrate deploy` — migration เดิมทั้งหมดต้องลงครบ (`prisma.config.ts` อ่าน `DATABASE_URL`; ค่าที่ใส่บน command line ชนะ `.env` ตรวจแล้ว 2026-09-30)
3. แก้ `schema.prisma` ตาม §4
4. `DATABASE_URL=<scratch> bunx prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma --script > prisma/migrations/<ts>_accounting_ar_tables/migration.sql` (Prisma 7 ไม่มี `--from-url`/`--to-schema-datamodel`) โดย `<ts>` = เวลาปัจจุบัน UTC รูปแบบ `YYYYMMDDHHmmss` ต้องมากกว่า `20260928130000`
5. review SQL: มีเฉพาะ `CREATE TYPE` 7, `CREATE TABLE` 10, index/unique, `ADD CONSTRAINT ... FOREIGN KEY` ตาม §4.1 — ห้ามมี `ALTER`/`DROP` ของตารางอื่น (ถ้ามีแปลว่า scratch หรือ schema เพี้ยน หยุดแล้วหาสาเหตุ)
6. `DATABASE_URL=<scratch> bunx prisma migrate deploy` แล้ว `DATABASE_URL=<scratch> bunx prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma --exit-code` ต้องได้ exit 0
7. `bun run db:generate` ใน prisma package, `bun run check-types` ที่ root (turbo ครอบ micro-business + gateway), `bun run test` ใน prisma package (`client.test.ts`)
8. ลบ scratch database

---

## 6. Audit / activity

- `excludeModels` เพิ่ม `tb_customer`, `tb_customer_address`, `tb_ar_invoice`, `tb_ar_invoice_detail`, `tb_ar_invoice_detail_dimension`, `tb_ar_invoice_reference`, `tb_ar_tax_invoice`, `tb_ar_receipt`, `tb_ar_receipt_detail`, `tb_ar_receipt_wht` ต่อจากกลุ่ม AP ใต้ comment `// accounting — receivable`
- ยังไม่มี handler จึงยังไม่ต้องลงทะเบียน activity registry; spec ของ AR service ต้องทำตามกฎเดียวกับ AP (CLAUDE.md micro-business บรรทัด 12)

---

## 7. Deploy (นอก PR, บันทึกใน PR body)

migration เป็น ADD-only จึงปลอดภัยต่อโค้ดเดิม; หลัง merge และ `Deploy Dev` ให้ยิง `POST /api-system/tenant/migrations/:bu_id/deploy` ครบทุก BU (14 ตัว ณ 2026-09-07) แล้วตรวจด้วย `prisma migrate status` ต่อ schema

---

## 8. การตรวจสอบก่อน merge

- §5 ข้อ 5–7 ผ่านทั้งหมด
- `git diff --stat` มีเฉพาะ 4 ไฟล์ใน §3
- migration.sql: `grep -c 'CREATE TABLE'` = 10, `grep -c 'CREATE TYPE'` = 7, ไม่มี `DROP`
- PR body ระบุขั้น deploy §7 และลิงก์ spec นี้

---

## 9. งานต่อเนื่อง

1. concept repo: README + `prisma/schema.prisma` ย้าย AR เข้า IMPLEMENTED โดยคัดลอกจาก tenant schema @ commit ที่ merge
2. spec AR service (หลัง BA ปิด OI-1..7): posting ผ่าน `GlSubledgerPostingService`, running code, gl_setting keys, activity registry, error catalog `AR_*`
3. customer master service/RPC/gateway ทำได้ก่อน AR service ถ้าต้องการ

---

## ภาคผนวก A — ข้อความ Prisma ที่ใช้จริง

วางต่อท้าย `enum_ap_payment_method` (enum) และต่อท้าย `tb_ap_payment_expense` (model) แล้วเพิ่ม back-relation 4 บรรทัดตาม §2

```prisma
// AR document types — mirrors enum_ap_invoice_doc_type (ARIV / ARDN / ARCN / ARDP)
enum enum_ar_invoice_doc_type {
  invoice
  debit_note
  credit_note
  deposit
}

// (workflow stages live in workflow_* columns, GL posting happens once at `posted`)
enum enum_ar_invoice_status {
  draft
  in_review
  posted
  void
}

enum enum_ar_invoice_source {
  manual
  pms_folio
  copy
  ai
}

enum enum_ar_tax_invoice_status {
  pending
  confirmed
  submitted
}

enum enum_ar_receipt_status {
  draft
  in_review
  posted
  void
}

enum enum_ar_receipt_method {
  bank_transfer
  cheque
  cash
  credit_card
  promptpay
}

enum enum_customer_address_type {
  contact_address
  mailing_address
  register_address
  billing_address
}

model tb_customer {
  doc_version Int     @default(0) @db.Integer
  id          String  @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  code        String  @db.VarChar // AR code (FRD `ar_code`)
  name        String  @db.VarChar
  description String? @db.VarChar
  note        String? @db.VarChar

  customer_group Json? @default("[]") @db.JsonB // [{id, name}] e.g. CORP / FIT / OTA / GOVT — same shape as tb_vendor.business_type

  registered_name String? @db.VarChar // ชื่อผู้ประกอบการจดทะเบียน ที่พิมพ์บนใบกำกับภาษี
  tax_no          String? @db.VarChar // เลขประจำตัวผู้เสียภาษี 13 หลัก (ต่างประเทศอาจไม่มี)
  branch_no       String? @db.VarChar // "00000" = สำนักงานใหญ่
  tax_profile_id  String? @db.Uuid // default output-tax profile

  default_currency_id     String? @db.Uuid
  default_currency_code   String? @db.VarChar(3)
  credit_term_id          String? @db.Uuid
  credit_term_name        String? @db.VarChar
  credit_term_days        Int?    @db.Integer
  credit_limit            Decimal @default(0) @db.Decimal(20, 5)
  ar_chart_of_accounts_id String? @db.Uuid // default Dr AR account; falls back to gl_setting

  pms_account_code String? @db.VarChar // City Ledger account code in the PMS (folio lookup key)

  is_active Boolean? @default(true)
  info      Json?    @default("{}") @db.JsonB
  dimension Json?    @default("[]") @db.JsonB

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid
  updated_at    DateTime? @default(now()) @db.Timestamptz(6)
  updated_by_id String?   @db.Uuid
  deleted_at    DateTime? @db.Timestamptz(6)
  deleted_by_id String?   @db.Uuid

  tb_tax_profile      tb_tax_profile?       @relation(fields: [tax_profile_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_customer_address tb_customer_address[]
  tb_ar_invoice       tb_ar_invoice[]
  tb_ar_receipt       tb_ar_receipt[]

  @@unique([code, deleted_at], map: "customer_code_u")
  @@index([name], map: "customer_name_idx")
  @@index([tax_no], map: "customer_tax_no_idx")
}

model tb_customer_address {
  doc_version   Int                         @default(0) @db.Integer
  id            String                      @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  customer_id   String                      @db.Uuid
  address_type  enum_customer_address_type
  address_line1 String?                     @db.VarChar // บ้านเลขที่ / ถนน / ซอย
  address_line2 String?                     @db.VarChar // อาคาร / ชั้น
  sub_district  String?                     @db.VarChar // ตำบล / แขวง
  district      String?                     @db.VarChar // อำเภอ / เขต
  city          String?                     @db.VarChar
  province      String?                     @db.VarChar
  postal_code   String?                     @db.VarChar
  country       String?                     @db.VarChar
  is_active     Boolean?                    @default(true)
  info          Json?                       @default("{}") @db.JsonB

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid
  updated_at    DateTime? @default(now()) @db.Timestamptz(6)
  updated_by_id String?   @db.Uuid
  deleted_at    DateTime? @db.Timestamptz(6)
  deleted_by_id String?   @db.Uuid

  tb_customer tb_customer @relation(fields: [customer_id], references: [id], onDelete: NoAction, onUpdate: NoAction)

  @@unique([customer_id, address_type, deleted_at], map: "customeraddress_type_u")
  @@index([customer_id], map: "customeraddress_customer_idx")
}

// ARIV / ARDN / ARCN / ARDP in one table (FRD ar_invoice_header) — mirrors tb_ap_invoice.
model tb_ar_invoice {
  doc_version Int    @default(0) @db.Integer
  id          String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid

  doc_no         String                   @db.VarChar(30) // ARIV{YY}{MM}{0000}, running-code type AR-IV/DN/CN/DP
  doc_type       enum_ar_invoice_doc_type
  doc_status     enum_ar_invoice_status   @default(draft)
  doc_source     enum_ar_invoice_source   @default(manual)
  doc_date       DateTime                 @db.Date // FRD input_date = GL date
  description    String?                  @db.VarChar
  source_doc_ref String?                  @db.VarChar(100) // external ref (booking / folio batch / copied doc no)
  is_pms_folio   Boolean                  @default(false) // true = billed from PMS folios, no GL post (PRD §3.1)

  customer_id      String   @db.Uuid
  customer_code    String   @db.VarChar
  customer_name    String   @db.VarChar
  credit_term_id   String?  @db.Uuid
  credit_term_name String?  @db.VarChar
  credit_term_days Int      @default(0) @db.Integer
  due_date         DateTime @db.Date // doc_date + credit_term_days; aging key

  is_tax_invoice Boolean @default(false) // Tab 2: issue tax invoice on submit → tb_ar_tax_invoice

  currency_id        String  @db.Uuid
  currency_code      String  @db.VarChar(3)
  exchange_rate      Decimal @default(1) @db.Decimal(15, 5)
  base_currency_id   String  @db.Uuid
  base_currency_code String  @db.VarChar(3)

  sub_total_amount         Decimal @default(0) @db.Decimal(20, 5)
  discount_amount          Decimal @default(0) @db.Decimal(20, 5)
  net_amount               Decimal @default(0) @db.Decimal(20, 5)
  vat_amount               Decimal @default(0) @db.Decimal(20, 5) // FRD tax1
  tax2_amount              Decimal @default(0) @db.Decimal(20, 5) // FRD tax2 (supplementary tax)
  total_amount             Decimal @default(0) @db.Decimal(20, 5)
  base_sub_total_amount    Decimal @default(0) @db.Decimal(20, 5)
  base_discount_amount     Decimal @default(0) @db.Decimal(20, 5)
  base_net_amount          Decimal @default(0) @db.Decimal(20, 5)
  base_vat_amount          Decimal @default(0) @db.Decimal(20, 5)
  base_tax2_amount         Decimal @default(0) @db.Decimal(20, 5)
  base_total_amount        Decimal @default(0) @db.Decimal(20, 5)
  is_wht_recorded          Boolean @default(false)
  wht_amount               Decimal @default(0) @db.Decimal(20, 5) // reference only — no GL at invoice
  outstanding_amount       Decimal @default(0) @db.Decimal(20, 5) // Σ detail.unpaid_amount (FRD unpaid_amount)
  reference_applied_amount Decimal @default(0) @db.Decimal(20, 5) // Σ deposits applied via tb_ar_invoice_reference

  gl_jv_id     String?   @db.Uuid // null when is_pms_folio (no GL post)
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

  tb_customer                 tb_customer               @relation(fields: [customer_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_ar_invoice_detail        tb_ar_invoice_detail[]
  tb_ar_invoice_reference     tb_ar_invoice_reference[] @relation("ArInvoiceReferences")
  tb_ar_invoice_referenced_by tb_ar_invoice_reference[] @relation("ArInvoiceReferencedBy")
  tb_ar_tax_invoice           tb_ar_tax_invoice?
  tb_ar_receipt_detail        tb_ar_receipt_detail[]

  @@unique([doc_no, deleted_at], map: "arinvoice_doc_no_u")
  @@index([customer_id, doc_status], map: "arinvoice_customer_status_idx")
  @@index([doc_date], map: "arinvoice_doc_date_idx")
  @@index([due_date], map: "arinvoice_due_date_idx")
  @@index([gl_jv_id], map: "arinvoice_gl_jv_idx")
}

// Line items (FRD ar_invoice_detail) — mirrors tb_ap_invoice_detail; cr = revenue, dr = AR.
model tb_ar_invoice_detail {
  doc_version   Int     @default(0) @db.Integer
  id            String  @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ar_invoice_id String  @db.Uuid
  sequence_no   Int     @db.Integer
  group_no      Int     @default(1) @db.Integer // summary-invoice print grouping
  description   String? @db.VarChar

  // Section 0 — booking & reference
  reference_info String?   @db.VarChar // 2-line folio / booking / guest text
  date_from      DateTime? @db.Date
  date_to        DateTime? @db.Date
  is_pms_folio   Boolean   @default(false)
  folio_no       String?   @db.VarChar(50)
  folio_date     DateTime? @db.Date
  folio_amount   Decimal?  @db.Decimal(20, 5) // folio balance at import; settle = net_amount (0 < settle ≤ folio_amount, ERR_AR_002)

  unit_id      String? @db.Uuid
  unit_name    String? @db.VarChar
  quantity     Decimal @default(1) @db.Decimal(20, 5)
  unit_price   Decimal @default(0) @db.Decimal(20, 5)

  // Section 1 — discount
  sub_total_amount     Decimal @default(0) @db.Decimal(20, 5)
  discount_pct         Decimal @default(0) @db.Decimal(15, 5)
  discount_amount      Decimal @default(0) @db.Decimal(20, 5)
  discount_is_override Boolean @default(false)
  net_amount           Decimal @default(0) @db.Decimal(20, 5)

  // Section 2 — revenue (Cr)
  cr_chart_of_accounts_id String  @db.Uuid
  cr_account_code         String? @db.VarChar
  cr_cost_center_id       String  @db.Uuid // USALI department, required for revenue
  cr_cost_center_code     String? @db.VarChar

  // Section 3 — output tax 1 (VAT)
  vat_tax_profile_id       String? @db.Uuid
  vat_rate                 Decimal @default(0) @db.Decimal(15, 5)
  vat_amount               Decimal @default(0) @db.Decimal(20, 5)
  vat_is_override          Boolean @default(false)
  vat_chart_of_accounts_id String? @db.Uuid // Output VAT Pending
  vat_cost_center_id       String? @db.Uuid

  tax2_tax_profile_id       String? @db.Uuid
  tax2_rate                 Decimal @default(0) @db.Decimal(15, 5)
  tax2_amount               Decimal @default(0) @db.Decimal(20, 5)
  tax2_is_override          Boolean @default(false)
  tax2_chart_of_accounts_id String? @db.Uuid
  tax2_cost_center_id       String? @db.Uuid

  // Section 4 — accounts receivable (Dr)
  dr_chart_of_accounts_id String  @db.Uuid
  dr_account_code         String? @db.VarChar
  dr_cost_center_id       String? @db.Uuid

  total_amount          Decimal @default(0) @db.Decimal(20, 5) // net + vat + tax2
  unpaid_amount         Decimal @default(0) @db.Decimal(20, 5)
  base_sub_total_amount Decimal @default(0) @db.Decimal(20, 5)
  base_discount_amount  Decimal @default(0) @db.Decimal(20, 5)
  base_net_amount       Decimal @default(0) @db.Decimal(20, 5)
  base_vat_amount       Decimal @default(0) @db.Decimal(20, 5)
  base_tax2_amount      Decimal @default(0) @db.Decimal(20, 5)
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

  tb_ar_invoice                  tb_ar_invoice                    @relation(fields: [ar_invoice_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_ar_invoice_detail_dimension tb_ar_invoice_detail_dimension[]
  tb_ar_receipt_detail           tb_ar_receipt_detail[]

  @@unique([ar_invoice_id, sequence_no, deleted_at], map: "arinvoicedetail_seq_u")
  @@index([ar_invoice_id], map: "arinvoicedetail_invoice_idx")
  @@index([folio_no], map: "arinvoicedetail_folio_idx")
}

// 6 analysis dimensions per line (FRD dim_market … dim_channel) — data-driven like AP
model tb_ar_invoice_detail_dimension {
  id                    String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ar_invoice_detail_id  String @db.Uuid
  gl_dimension_id       String @db.Uuid
  gl_dimension_value_id String @db.Uuid
  dimension_code        String @db.VarChar(20)
  value_code            String @db.VarChar(30)

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid

  tb_ar_invoice_detail  tb_ar_invoice_detail  @relation(fields: [ar_invoice_detail_id], references: [id], onDelete: Cascade, onUpdate: NoAction)
  tb_gl_dimension       tb_gl_dimension       @relation(fields: [gl_dimension_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_gl_dimension_value tb_gl_dimension_value @relation(fields: [gl_dimension_value_id], references: [id], onDelete: NoAction, onUpdate: NoAction)

  @@unique([ar_invoice_detail_id, gl_dimension_id], map: "arinvoicedetaildimension_u")
}

model tb_ar_invoice_reference {
  id                           String                   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ar_invoice_id                String                   @db.Uuid
  ref_ar_invoice_id            String                   @db.Uuid
  ref_doc_type                 enum_ar_invoice_doc_type
  applied_net_amount           Decimal                  @default(0) @db.Decimal(20, 5)
  applied_vat_amount           Decimal                  @default(0) @db.Decimal(20, 5)
  applied_amount               Decimal                  @default(0) @db.Decimal(20, 5) // net + vat = total offset
  base_applied_at_invoice_rate Decimal                  @default(0) @db.Decimal(20, 5)
  base_applied_at_ref_rate     Decimal                  @default(0) @db.Decimal(20, 5)
  realized_fx_amount           Decimal                  @default(0) @db.Decimal(20, 5) // > 0 = loss (Dr), < 0 = gain (Cr)
  remarks                      String?                  @db.VarChar

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid

  tb_ar_invoice     tb_ar_invoice @relation("ArInvoiceReferences", fields: [ar_invoice_id], references: [id], onDelete: Cascade, onUpdate: NoAction)
  tb_ref_ar_invoice tb_ar_invoice @relation("ArInvoiceReferencedBy", fields: [ref_ar_invoice_id], references: [id], onDelete: NoAction, onUpdate: NoAction)

  @@unique([ar_invoice_id, ref_ar_invoice_id], map: "arinvoicereference_u")
  @@index([ref_ar_invoice_id], map: "arinvoicereference_ref_idx")
}

// Output tax register (FRD ar_tax_invoice) — mirrors tb_ap_invoice_tax. One row per issuing
model tb_ar_tax_invoice {
  doc_version      Int                         @default(0) @db.Integer
  id               String                      @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ar_invoice_id    String?                     @unique(map: "artaxinvoice_invoice_u") @db.Uuid
  ar_receipt_id    String?                     @unique(map: "artaxinvoice_receipt_u") @db.Uuid
  tax_prefix       String                      @db.VarChar(10) // TXIV / TXDN / TXCN / TXDP / TXRC
  tax_invoice_no   String                      @db.VarChar(30) // generated on submit, never on save draft (ERR_AR_006 on collision)
  tax_invoice_date DateTime                    @db.Date
  filing_month     Int?                        @db.Integer // FRD tax_period MM/YYYY
  filing_year      Int?                        @db.Integer
  tax_status       enum_ar_tax_invoice_status  @default(pending)

  customer_registered_name String  @db.VarChar
  customer_tax_no          String? @db.VarChar(13)
  customer_branch_no       String? @db.VarChar(5)
  address_line1            String? @db.VarChar
  address_line2            String? @db.VarChar
  province                 String? @db.VarChar
  postal_code              String? @db.VarChar(10)

  base_amount  Decimal @default(0) @db.Decimal(20, 5) // THB
  vat_rate     Decimal @default(7) @db.Decimal(15, 5)
  vat_amount   Decimal @default(0) @db.Decimal(20, 5) // THB
  total_amount Decimal @default(0) @db.Decimal(20, 5) // THB

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid
  updated_at    DateTime? @default(now()) @db.Timestamptz(6)
  updated_by_id String?   @db.Uuid
  deleted_at    DateTime? @db.Timestamptz(6)
  deleted_by_id String?   @db.Uuid

  tb_ar_invoice tb_ar_invoice? @relation(fields: [ar_invoice_id], references: [id], onDelete: Cascade, onUpdate: NoAction)
  tb_ar_receipt tb_ar_receipt? @relation(fields: [ar_receipt_id], references: [id], onDelete: Cascade, onUpdate: NoAction)

  @@unique([tax_invoice_no, deleted_at], map: "artaxinvoice_no_u")
  @@index([tax_invoice_date], map: "artaxinvoice_date_idx")
  @@index([filing_year, filing_month], map: "artaxinvoice_period_idx")
}

// "Get Receipt" or standalone; allocates to posted AR document lines.
model tb_ar_receipt {
  doc_version Int    @default(0) @db.Integer
  id          String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid

  doc_no        String                 @db.VarChar(30) // ARRC{YY}{MM}{0000}
  doc_status    enum_ar_receipt_status @default(draft)
  receipt_date  DateTime               @db.Date // GL date
  received_date DateTime?              @db.Date // value date on bank statement
  description   String?                @db.VarChar
  reference_no  String?                @db.VarChar(100)

  customer_id   String  @db.Uuid
  customer_code String  @db.VarChar
  customer_name String  @db.VarChar
  payer_name    String? @db.VarChar // name on cheque / transfer when not the customer

  bank_account_id   String                  @db.Uuid
  bank_account_code String                  @db.VarChar(20)
  bank_account_name String                  @db.VarChar
  receipt_method    enum_ar_receipt_method  @default(bank_transfer)
  cheque_no         String?                 @db.VarChar(50)
  cheque_date       DateTime?               @db.Date

  is_tax_invoice Boolean @default(false) // TXRC issued at receipt (services) → tb_ar_tax_invoice

  currency_id        String  @db.Uuid
  currency_code      String  @db.VarChar(3)
  exchange_rate      Decimal @default(1) @db.Decimal(15, 5)
  base_currency_id   String  @db.Uuid
  base_currency_code String  @db.VarChar(3)

  total_applied_amount      Decimal @default(0) @db.Decimal(20, 5) // Σ detail.applied_amount
  total_wht_amount          Decimal @default(0) @db.Decimal(20, 5) // WHT deducted by customer (50 ทวิ received)
  total_charge_amount       Decimal @default(0) @db.Decimal(20, 5) // bank charges / short payment written off
  net_received_amount       Decimal @default(0) @db.Decimal(20, 5) // applied − wht − charge = bank Dr
  base_total_applied_amount Decimal @default(0) @db.Decimal(20, 5)
  base_total_wht_amount     Decimal @default(0) @db.Decimal(20, 5)
  base_total_charge_amount  Decimal @default(0) @db.Decimal(20, 5)
  base_net_received_amount  Decimal @default(0) @db.Decimal(20, 5)
  realized_fx_amount        Decimal @default(0) @db.Decimal(20, 5) // base@receipt − base@invoice; > 0 = gain (Cr)

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

  tb_customer                tb_customer                  @relation(fields: [customer_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_bank_account            tb_bank_account              @relation(fields: [bank_account_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_ar_receipt_detail       tb_ar_receipt_detail[]
  tb_ar_receipt_wht          tb_ar_receipt_wht[]
  tb_ar_tax_invoice          tb_ar_tax_invoice?

  @@unique([doc_no, deleted_at], map: "arreceipt_doc_no_u")
  @@index([customer_id, doc_status], map: "arreceipt_customer_status_idx")
  @@index([receipt_date], map: "arreceipt_date_idx")
}

// Line-level allocation — mirrors tb_ap_payment_detail (credit_note lines carry a negative amount)
model tb_ar_receipt_detail {
  id                           String                   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ar_receipt_id                String                   @db.Uuid
  ar_invoice_id                String                   @db.Uuid
  ar_invoice_detail_id         String                   @db.Uuid
  doc_type                     enum_ar_invoice_doc_type
  applied_amount               Decimal                  @default(0) @db.Decimal(20, 5)
  invoice_exchange_rate        Decimal                  @default(1) @db.Decimal(15, 5)
  base_applied_at_invoice_rate Decimal                  @default(0) @db.Decimal(20, 5)
  base_applied_at_receipt_rate Decimal                  @default(0) @db.Decimal(20, 5)
  realized_fx_amount           Decimal                  @default(0) @db.Decimal(20, 5)

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid

  tb_ar_receipt        tb_ar_receipt        @relation(fields: [ar_receipt_id], references: [id], onDelete: Cascade, onUpdate: NoAction)
  tb_ar_invoice        tb_ar_invoice        @relation(fields: [ar_invoice_id], references: [id], onDelete: NoAction, onUpdate: NoAction)
  tb_ar_invoice_detail tb_ar_invoice_detail @relation(fields: [ar_invoice_detail_id], references: [id], onDelete: NoAction, onUpdate: NoAction)

  @@unique([ar_receipt_id, ar_invoice_detail_id], map: "arreceiptdetail_u")
  @@index([ar_invoice_detail_id], map: "arreceiptdetail_invoice_detail_idx")
}

// WHT certificate (50 ทวิ) received from the customer — mirrors tb_ap_payment_wht; posts Dr WHT receivable
model tb_ar_receipt_wht {
  id                   String                         @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ar_receipt_id        String                         @db.Uuid
  wht_certificate_no   String?                        @db.VarChar(50) // customer's certificate number
  wht_certificate_date DateTime?                      @db.Date
  wht_tax_profile_id   String?                        @db.Uuid
  wht_tax_profile_name String?                        @db.VarChar
  pnd_form             enum_tax_profile_wht_pnd_form?
  income_type          String?                        @db.VarChar
  base_amount          Decimal                        @default(0) @db.Decimal(20, 5)
  wht_rate             Decimal                        @default(0) @db.Decimal(15, 5)
  wht_amount           Decimal                        @default(0) @db.Decimal(20, 5)
  is_override          Boolean                        @default(false)
  chart_of_accounts_id String                         @db.Uuid // WHT receivable (gl_setting.wht_receivable_account_id)

  created_at    DateTime? @default(now()) @db.Timestamptz(6)
  created_by_id String?   @db.Uuid

  tb_ar_receipt tb_ar_receipt @relation(fields: [ar_receipt_id], references: [id], onDelete: Cascade, onUpdate: NoAction)

  @@index([ar_receipt_id], map: "arreceiptwht_receipt_idx")
}
```
