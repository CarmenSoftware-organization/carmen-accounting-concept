# Review: AR Folio Management FRD v2.0 (New Concept)

**เอกสารที่ review:** [Drive › AR › Folio](https://drive.google.com/drive/folders/1vpk85hjZFTD0MbYmXK-t9PhsZmyIq8k4?usp=drive_link) — `Functional Requirement Document: AR Folio Management.md` (FRD v2.0, sign-off 8 พ.ย. 2026) + mockup `ar_folio_management.html`
**วันที่ review:** 2026-11-09
**Backend ที่ตรวจ:** `carmen-turborepo-backend` main @ `1abb6a9c0` (tenant schema, `apps/micro-business/src/ar`, `gl`)
**เทียบกับ:** [../PRD-module-ar.md](../PRD-module-ar.md) §3, `specs/2026-09-30-accounting-ar-schema-design.md`, `specs/2026-10-01-accounting-ar-invoice-service-design.md`, `prisma/schema.prisma`
**อ้างอิงบรรทัด:** `FRD:n` = บรรทัดใน FRD v2.0 ฉบับ mirror

> FRD นี้เป็น **concept** ที่ทีมนำมาปรับ ไม่ใช่ spec ของเรา และไม่ใช่สิ่งที่เราจะไปขอให้ผู้เขียนแก้
> เอกสารนี้จึงจัดเป็น Adopt / Adapt / Reject-Defer และบันทึกความไม่สอดคล้องภายใน FRD เป็น **ความเสี่ยงของเรา** เท่านั้น

---

## 1. สรุป verdict

**Adopt แนวคิด, Adapt การออกแบบ — ยังไม่ implement จนกว่าจะมี PMS interface และตัดสินเรื่อง GL control (ข้อ 5)**

- ตัว FRD ชี้ถูกจุดว่า folio ต้องมี **staging/holding area ของตัวเอง** (list + สถานะ + ยอดคงเหลือ + audit) ซึ่งของเรา **ยังไม่มี**: PRD §3.3 ระบุ "folio balance tracking" เป็น dependency ที่ยังไม่สร้าง และ schema ปัจจุบันมีแค่คอลัมน์ `folio_no/folio_date/folio_amount` บน `tb_ar_invoice_detail` (`schema.prisma:1607-1609`) ไม่มีตาราง folio
- สามอย่างที่ควรรับเกือบตรง ๆ: ยอดแยก 4 ช่อง (โดยมี invariant), กฎ Void/Switch เฉพาะ folio ที่ยังไม่ถูกวางบิล, append-only audit
- สองอย่างที่ต้อง Adapt หนัก: **Non-Re-Posting (§6)** ไม่ครอบคลุม reconciliation GL control vs sub-ledger และ **Switch A/R** ต้องการข้อมูลจาก customer master ที่ยังไม่มี (`used_credit`)
- ไม่พบ PMS/Cloudbeds code ใน backend เลย ขอบเขต integration ที่ FRD เขียนไว้ (Opera/Cloudbeds/POS, API + file drop) กว้างกว่าที่เราควรเริ่ม
- **ความเสี่ยง posting:** ปัจจุบันไม่มี double-post เพราะ `checkArHeader` บล็อก `is_pms_folio` ทั้งหมด (`ar-invoice.validation.ts:62`) แต่ `ar-invoice.posting.ts` **ไม่มีเงื่อนไข `is_pms_folio` เลย** ถ้าปลดบล็อกโดยไม่เพิ่ม guard ARIV จาก folio จะลง JV ซ้ำ (ดูข้อ 5)

---

## 2. Adopt

| # | องค์ประกอบ FRD | อ้างอิง | เหตุผล |
|---|----------------|---------|--------|
| A1 | folio เป็น staging hub ที่ **ไม่ถูกแก้ตัวเลขต้นทาง** (Amount/VAT/Total มาจาก PMS เท่านั้น) | FRD:51 | ตรงกับหลัก PRD §3.1 และกัน drift จาก Night Audit JV |
| A2 | Void ได้เฉพาะ folio ที่ยังไม่ถูกวางบิล (`un_bill = total`) พร้อม reason code + remark ≥ 5 ตัวอักษร | FRD:109, 344 | เข้ากับ pattern void ของ ARIV (`void_reason`, `schema.prisma:1546` ช่วง void_*) |
| A3 | Switch A/R บล็อกเมื่อมีบิลแล้ว (รวม partial) | FRD:111, 340 | กันไม่ให้ invoice ที่ออกไปแล้วอ้าง customer คนละราย |
| A4 | Switch A/R: ปลายทางต้อง active | FRD:342 | ตรงกับ `validateArCustomer` → `CUSTOMER_INACTIVE` (`ar-invoice.validation.ts:49`) |
| A5 | ERR_ARF_007: ห้าม Void/แก้ เมื่องวด GL ของ `folio_date` ปิดแล้ว | FRD:346, 249 | มี `tb_gl_period`/`gl-period` รองรับอยู่แล้ว |
| A6 | View Linked Invoices (folio ↔ ARIV แบบ many-to-many พร้อม `billed_amount`) | FRD:110, 163-180 | ต้องใช้จริงเพื่อ partial billing ตาม PRD §3.2 ข้อ 3 |
| A7 | KPI 4 ใบ + default filter `Pending` | FRD:46, 371 | เป็นแค่ query/aggregate ไม่กระทบ schema |

## 3. Adapt

| # | FRD ว่า | เราปรับเป็น | เหตุผล |
|---|---------|-------------|--------|
| P1 | สถานะ `Pending / Billed / Void` (`ENUM` ใน `carmen_ar_folios`, FRD:146) | `enum_ar_folio_status = pending, billed, void` (lowercase ตามแบบ `enum_ar_invoice_status = draft, in_review, posted, void` `schema.prisma:361`) และ **derive `billed` จากยอด** ไม่ใช่ set ด้วยมือ | FRD เก็บทั้ง `status` และ `un_bill`/`billed_amount` ซ้ำกัน ต้องกัน 2 แหล่งความจริงขัดกัน (FRD เองมี `CHECK total = billed + un_bill` แต่ไม่มี check ผูกกับ `status`) |
| P2 | 4 ช่อง `amount / tax_amount / total / un_bill` + `billed_amount` (`DECIMAL(15,2)`) | ใช้ `Decimal(20,5)` ตามมาตรฐานเรา (`schema.prisma:1609`); ชื่อ `net_amount / vat_amount / total_amount / billed_amount / unbilled_amount`; เพิ่ม `base_*` | ดูตารางข้อ 4. FRD fix 7% (`FRD:96`) และ `THB` default แต่เรามี `tb_tax_profile` และ multi-currency |
| P3 | `un_bill` = ยอดคงเหลือ แต่ **ใน ARIV แถวเรียก `unpaid_amount`** | แยกคำให้ชัด: folio `unbilled_amount` (ยังไม่วางบิล) ≠ invoice `outstanding_amount` (ยังไม่รับชำระ) | FRD เปลี่ยนชื่อ `Unpaid → Un-Bill` เพราะสับสนเรื่องเดียวกันนี้ เราควรเลี่ยงตั้งแต่ชื่อคอลัมน์ |
| P4 | Switch A/R: เปลี่ยน `ar_no`/`ar_name` ในแถว folio (denormalized `VARCHAR`) | เก็บ `customer_id` (FK `tb_customer`) + snapshot `customer_code/name` ตามแบบ `tb_ar_invoice` | เรา FK ทุกที่ ไม่เก็บ code เป็น key; ต้อง snapshot ณ วัน switch เพื่อ audit |
| P5 | Credit Limit Warning คำนวณ `limit − used_credit` ใน **UI** (FRD:316) | คำนวณฝั่ง service; เป็น warning ไม่ block (ตาม ERR_ARF_004 FRD:343) แล้วเก็บ flag ใน activity log | ไม่ควรเชื่อ client; ต้องการ `used_credit` ที่ master ไม่มี (ข้อ 6) |
| P6 | Activity Log ตารางเฉพาะ `carmen_ar_folio_activity_logs`, `ENUM` 5 ค่า, `ON DELETE CASCADE` (FRD:182-200) | ใช้ `tb_activity` + activity registry (`entityName: 'tb_ar_folio'`) และเพิ่ม action ที่ยังไม่มี (`switch_ar`, `invoice_linked`) หรือเก็บใน `meta_data` | เรามี audit pattern กลางอยู่แล้ว (invoice service spec §12.5: ตาราง AR อยู่ใน `excludeModels` ต้องลงทะเบียนเอง). ดู "audit" ด้านล่าง |
| P7 | `carmen_ar_folio_invoices` junction (FRD:167) | `tb_ar_folio_invoice` (folio_id, ar_invoice_id, ar_invoice_detail_id, billed_amount) ผูกกับ `tb_ar_invoice_detail.folio_*` เดิม | ของเรามี `folio_amount` ที่ detail แล้ว ต้อง reconcile กับ junction ไม่ใช่ซ้ำ; **FRD เก็บ `invoice_no`/`invoice_status` ซ้ำ** ควรอ้าง FK แล้ว join |
| P8 | `INVOICE_UNLINKED` ใน log แต่ **ไม่มี flow** ที่ทำให้เกิดเหตุการณ์นี้ | ต้อง spec เอง: void ARIV ที่มี `is_pms_folio` → คืน `billed_amount` เข้า folio | PRD §6 "Void … releases applied deposits / folio balances" ระบุไว้แล้ว; FRD ข้อ ERR_ARF_001/002 พึ่ง flow นี้ (ต้อง void invoice ก่อน) |

**เรื่อง audit:** FRD ต้องการ append-only (ห้าม UPDATE/DELETE, FRD:358) แต่ `tb_activity` มี `deleted_at`/`updated_at` (soft-delete แบบมาตรฐาน) ไม่ได้ immutable ระดับ DB. ถ้าจะรับ "immutable" จริง ต้องตัดสินว่าพอแค่ convention + ห้ามมี API update/delete หรือต้อง trigger/REVOKE ที่ DB (Open Q4). ส่วน `carmen_audit_logs` (FRD:251) เป็นตารางที่เราไม่มี ใช้ `tb_activity` แทน

## 4. Reject / Defer

| # | องค์ประกอบ | อ้างอิง | สถานะ | เหตุผล |
|---|-----------|---------|-------|--------|
| R1 | Cloudbeds/Opera/POS interface (API JSON + file drop) ในขอบเขตเดียว | FRD:38, 19 | **Defer** | ไม่มี PMS code ใน backend (grep ไม่พบ folio/cloudbeds นอก AR schema); PRD OI-7 ยังรอ spec interface. ต้องเลือก PMS แรกก่อน ไม่ใช่ทำทุกตัว |
| R2 | ตาราง `carmen_*` แบบ MySQL (`AUTO_INCREMENT`, `ENGINE=InnoDB`, `ENUM`) | FRD:125-199 | **Reject** | stack เราคือ PostgreSQL + Prisma, UUID, Timestamptz, soft-delete (`deleted_at`), `doc_version`. ใช้เป็น field list เท่านั้น |
| R3 | `ar_customer_profile`, `gl_period_control`, `ar_invoice_header` ตามชื่อ | FRD:248-250 | **Reject (ชื่อ)** | ของเราคือ `tb_customer`, `tb_gl_period`, `tb_ar_invoice` |
| R4 | `AI & Exception Listener` / Notification Center | FRD:66 | **Defer** | ไม่มี spec/ข้อมูล; เกินขอบเขตรอบแรก |
| R5 | Data masking AES-256 ของข้อมูลบัตรเครดิต/PII จาก PMS | FRD:356 | **Defer / ตรวจแยก** | ไม่ควรรับเลขบัตรเข้า folio เลย (เก็บ token/masked จาก PMS) — ตัดสินร่วม security ก่อนออกแบบ field |
| R6 | "Credit Card Folios" เป็น folio ประเภทเดียวกับ City Ledger | FRD:27 | **Defer** | ลูกหนี้บัตรเครดิตไม่ใช่ customer ที่ออก ARIV ตามปกติ (ได้ settle จาก acquirer); PRD §1 รวมไว้แต่ยังไม่มี flow — ถามเรื่อง ownership |
| R7 | `View By`, Theme OKLCH, TH/EN toggle, layout multi-line | FRD:73, 359 | **Out of concept** | เรื่อง frontend ตาม design system ของเรา |

---

## 5. Posting / ความเสี่ยง double-count (จุดสำคัญ)

### 5.1 FRD §6 พูดอะไร

Night Audit ลง GL เองทั้งหมด (`Dr AR Control / Cr Revenue / Cr Output VAT Pending`, FRD:210-212); ARIV จาก folio เป็น "billing doc only — NO GL RE-POSTING" (FRD:214, 240). ตรงกับ PRD §3.1/§4.1 ที่เรารับไว้แล้ว ดังนั้น **ในหลักการไม่ขัดกัน**

### 5.2 สิ่งที่ code เราทำจริง

| จุด | ผล |
|-----|----|
| `checkArHeader`: `!isManual \|\| is_pms_folio` → `AR_SOURCE_NOT_SUPPORTED` (`ar-invoice.validation.ts:62`); บรรทัดก็บล็อก (`:201`) | วันนี้ folio invoice **สร้างไม่ได้เลย** จึงไม่มี double-post |
| `ar-invoice.posting.ts` ไม่มีการอ้าง `is_pms_folio` (grep ว่าง); `postInTx` สร้าง JV ผ่าน `subledger.postFromSource` (`:184`) แล้ว set `gl_jv_id` (`:648`) | **ไม่มี guard ที่ชั้น posting** — ปลดบล็อกที่ validation เมื่อไร ARIV ทุกใบจะลง `Dr AR / Cr Revenue / Cr VAT` ซ้ำกับ Night Audit JV ทันที |
| `schema.prisma:1546` comment `gl_jv_id` null เมื่อ `is_pms_folio` | เป็นเพียงคำอธิบาย ไม่มี CHECK constraint |

**ความเสี่ยง R-1 (สูง):** การเปิดใช้ folio ต้องเพิ่ม guard ใน posting (`is_pms_folio → skip JV, mark posted`) **และ** CHECK `is_pms_folio = true ⇒ gl_jv_id IS NULL` ที่ DB ไม่ใช่แค่ comment

### 5.3 ช่องว่างที่ FRD ไม่ครอบคลุม

| # | ประเด็น | ผลต่อเรา |
|---|---------|----------|
| G1 | **GL control ≠ sub-ledger ระหว่าง Pending** — Night Audit ลง `Dr AR Control` ทันที แต่ folio ที่ยังไม่วางบิลไม่อยู่ใน `tb_ar_invoice` → aging/outstanding ของ sub-ledger **ต่ำกว่า** GL control (FRD:216 "บันทึกลง AR Aging" เมื่อออก ARIV เท่านั้น) | รายงาน reconciliation control vs sub-ledger ต้องนับ `Σ folio.unbilled_amount` เป็นรายการกระทบยอด; aging รวม folio pending หรือไม่ ต้องตัดสิน |
| G2 | **ARIV ที่ void แล้ว** — FRD ไม่พูดว่า GL ต้อง reverse ไหม. ตาม non-re-post ต้อง **ไม่** reverse (GL เดิมยังอยู่) แต่ folio คืนกลับ pending. แต่ `voidPostedInTx` ของเรามี `void_gl_jv_id` (`posting.ts:260`) ต้องกัน path นี้ไม่ให้ reverse JV ของ Night Audit | ต้องมีเงื่อนไข void สำหรับ folio invoice โดยเฉพาะ |
| G3 | **Switch A/R "sub-ledger only"** (FRD:111): ถ้า Night Audit JV ลงที่ **single AR control account** ไม่ระบุลูกค้า → ไม่กระทบ GL จริง. แต่ถ้าบรรทัด JV มี dimension/ref ของลูกค้า (`pms_account_code` ที่ `schema.prisma:1446`) การเปลี่ยน customer ใน folio จะทำให้ GL กับ sub-ledger คนละลูกค้า | ต้องยืนยันรูปแบบ Night Audit JV ก่อน — ถ้า JV ระบุลูกค้า Switch A/R ทำ "ไม่มี JV" ไม่ได้ |
| G4 | **VAT** — FRD:217 ว่า ARIV "แปลงภาษีขายรอตั้งหนี้เป็นภาษีขายจริง" (ต่างจากบรรทัด 214 ที่บอก no GL post) ถ้าเกิดจริงคือ JV `Dr Output VAT Pending / Cr Output VAT` ซึ่งเป็น GL post. ฝั่งเรา `is_tax_invoice` ออก `tb_ar_tax_invoice` ตอน post | **ความเสี่ยง R-2:** ตีความ 2 ทาง (post/ไม่ post) — ใบกำกับภาษีเกิดตอนไหน และใครรับรู้ภาษีขาย Night Audit หรือ ARIV ต้องให้ที่ปรึกษาบัญชียืนยัน; ถ้า ARIV ออก tax invoice แต่ไม่มี JV ใบกำกับจะไม่ตรงบัญชีภาษี |
| G5 | **Folio ที่ Void แล้ว** — Night Audit JV ยังอยู่ใน GL แต่ folio ถูก void (`un_bill = 0`, FRD:43). ถ้า "void" หมายถึงส่งซ้ำจาก PMS (ตัวอย่าง `PMS_POSTING_ERROR`, FRD:374) GL มียอดซ้ำค้างอยู่ | **ความเสี่ยง R-3:** Void ใน sub-ledger อย่างเดียวไม่แก้ GL; ต้องมีกระบวนการกลับรายการ Night Audit JV (ไม่อยู่ใน FRD) |
| G6 | ปิดงวด: FRD บล็อก Void เมื่องวดปิด (FRD:346) แต่ **ไม่บล็อกการสร้าง ARIV** | ARIV ไม่ลง GL ดังนั้นไม่ขัดกับ period แต่ aging ณ วันที่ในอดีตเปลี่ยนได้ — ตัดสินว่ายอมไหม |
| G7 | ARIV ผสมบรรทัด manual + folio | PRD §4.1 ยังไม่ตัดสิน; ถ้าผสมกัน JV จะต้องรวมเฉพาะบรรทัด manual — แนะนำ **ห้ามผสม** ในรอบแรก (ลดความซับซ้อน guard) |

---

## 6. Schema delta เทียบของที่มีอยู่

ปัจจุบัน (schema-only): folio อยู่ใน `tb_ar_invoice_detail` (`reference_info`, `is_pms_folio`, `folio_no`, `folio_date`, `folio_amount`, index `arinvoicedetail_folio_idx`) และ `tb_customer.pms_account_code / credit_limit / is_active / default_currency_*` (`schema.prisma:1446-1470`). **ไม่มี** ตาราง folio, ไม่มี enum สถานะ folio

| FRD field | มีแล้ว? | Delta |
|-----------|---------|-------|
| `folio_no`, `folio_date`, `guest_no` | detail มี `folio_no`, `folio_date` | ตารางใหม่ `tb_ar_folio`: เพิ่ม `guest_no`/masked guest ตาม PRD §3.2 ข้อ 2; unique `(source, folio_no)` (FRD มีแค่ index ไม่ unique → เสี่ยง import ซ้ำ ดู Q6) |
| `ar_no`, `ar_name` | — | `customer_id` FK + snapshot `customer_code/name` |
| `amount / tax_amount / total` | detail ใช้ `net_amount`, `vat_amount`, `total_amount` | ตั้งชื่อตามเดิม + `base_*` + `currency_id/code/exchange_rate`; CHECK `total = net + vat` |
| `un_bill`, `billed_amount` | detail มี `folio_amount` (balance ณ import) | `billed_amount`, `unbilled_amount` + CHECK `total = billed + unbilled` และ `status` สอดคล้อง |
| `status` Pending/Billed/Void | — | `enum_ar_folio_status` ใหม่ |
| `source` (CarmenAI/Opera/Cloudbeds) | `enum_ar_invoice_source` มี `pms_folio` (ระดับ invoice) | folio ระดับตัวเองเก็บ `pms_system` เป็น VarChar หรือ enum แยก — อย่านำไปปน `doc_source` |
| `input_date`, `sync_remark`, `remark`, `contact_info` | — | `contact_info` ไม่ควรเก็บซ้ำ (ดึงจาก `tb_customer_address`); `sync_remark` เป็น field ของ interface batch |
| junction folio↔invoice | — | `tb_ar_folio_invoice` (P7) |
| void: reason code + remark, `void_at/by` | invoice มี `void_reason` | เพิ่ม `void_reason_code` (ต้องมี master) + `void_at/by`; โครงตาม `tb_credit_note_reason` ถ้าต้องการ master |
| activity log | `tb_activity` | ลงทะเบียน entity; ไม่สร้างตารางใหม่ (P6) |
| `used_credit` (FRD:310) | **ไม่มี** ที่ `tb_customer` | ไม่เพิ่มคอลัมน์ — คำนวณจาก `Σ tb_ar_invoice.outstanding_amount` (+ ตัดสินว่ารวม folio pending หรือไม่, Q3) |
| ตรวจ Active | `tb_customer.is_active` | ใช้ได้ทันที |
| ตรวจ period ปิด | `tb_gl_period` | ใช้ได้ทันที |

ข้อสังเกตเพิ่ม: FRD `chk_ar_folio_total` ไม่คำนึง rounding หลายสกุลเงิน; `exchange_rate DECIMAL(10,6)` ต่างจากของเรา `Decimal(15,5)`

## 7. Switch A/R ต้องการอะไรจาก customer master

| ต้องการ | มีใน `tb_customer` | หมายเหตุ |
|---------|-------------------|----------|
| รายชื่อ active | `is_active` ✓ | ตรงกับ `validateArCustomer` |
| `credit_limit` | ✓ `Decimal(20,5) default 0` | `0` ต้องหมายถึง "ไม่จำกัด" หรือ "ไม่มีเครดิต"? FRD ไม่ระบุ (Q3) |
| `used_credit` | ✗ | ต้อง derive; ต้นทางที่แท้จริงต้อง include ARIV posted คงค้าง + folio pending ของลูกค้านั้น |
| สกุลเงิน | `default_currency_code` ✓ | FRD ไม่ตรวจว่าสกุล folio ตรงกับลูกค้าปลายทาง — เรา warn/บล็อกได้ |
| `pms_account_code` | ✓ | อาจเป็น key ที่ import ใช้ map folio → customer; Switch A/R เป็นการแก้ mapping ผิด ควรพิจารณา "จำ" mapping ไม่ให้ผิดซ้ำ (Q5) |
| สิทธิ์ | — | FRD ให้ Senior Accountant/FC (FRD:64) ต้อง map เข้า permission ของเรา |

---

## 8. คำถามเปิด (Open Questions)

1. **รูปแบบ Night Audit JV** — ระบุ customer ในบรรทัด AR หรือลง control account เดียว? (กำหนดว่า Switch A/R "sub-ledger only" ทำได้จริงหรือไม่ — G3)
2. **ภาษี** — ใครรับรู้ Output VAT และออกใบกำกับภาษี: Night Audit หรือ ARIV จาก folio? JV `Output VAT Pending → Output VAT` มีหรือไม่? (G4 / R-2) ต้องให้ที่ปรึกษาบัญชียืนยัน
3. **Credit exposure** — `used_credit` นับ folio pending ด้วยหรือไม่? `credit_limit = 0` แปลว่าอะไร?
4. **Immutability ของ audit** — พอแค่ `tb_activity` + ไม่มี API แก้/ลบ หรือต้อง DB-level (trigger/REVOKE)?
5. **Switch A/R ซ้ำ** — จะอัปเดต `pms_account_code` ของลูกค้า/สร้าง mapping เพื่อไม่ให้ night audit ถัดไปส่งผิดซ้ำหรือไม่? (FRD ไม่พูด)
6. **Idempotency** ของ import — key ที่ใช้กัน folio ซ้ำจาก PMS คืออะไร (`source + folio_no + folio_date`?) และ re-send แก้ตัวเลขทำอย่างไรเมื่อ folio ถูกวางบิลบางส่วนแล้ว (FRD:51 ห้ามแก้ แต่ PMS แก้ได้จริง)
7. **Void folio vs GL** — กรณีส่งซ้ำ ใครกลับรายการ Night Audit JV? (G5 / R-3)
8. **Partial billing กับ Switch/Void** — FRD ล็อกทั้งสองเมื่อ `un_bill < total` ผู้ใช้ต้องไป void ARIV ก่อน ยอมรับ UX นี้หรือไม่ (ARIV approve flow ยาว)
9. **Credit Card Folio** (R6) อยู่ใน scope AR Folio หรือไม่ ใครเป็น "customer"?
10. **PMS แรก** ที่จะทำ interface คือ Cloudbeds, Opera หรืออย่างอื่น? ขอบเขต: pull (API) หรือ push (file drop)?
11. **ARIV ผสม manual+folio** — ห้ามหรืออนุญาต? (G7)
12. **Reason code** สำหรับ Void (`PMS_POSTING_ERROR` ฯลฯ) — ใคร own master list?

## 9. ขั้นตอนถัดไปที่แนะนำ

1. ส่ง Q1, Q2, Q7 ให้ BA/ที่ปรึกษาบัญชี (กำหนดว่า posting model ใช้ได้จริงหรือไม่) — **ต้องได้คำตอบก่อนออกแบบ schema**
2. เพิ่มแถว/หัวข้อใน PRD §3 และ OI ใหม่ (OI-8 control vs sub-ledger, OI-9 VAT บน folio invoice, OI-10 Void folio vs Night Audit JV)
3. เขียน spec ใหม่ `accounting-ar-folio-design` (schema `tb_ar_folio` + `tb_ar_folio_invoice`, สถานะ, guard ที่ posting, activity registry) ต่อจาก invoice service spec ข้อ "PMS folio (หลังมี interface)"
4. ระหว่างนั้น **ไม่ปลดบล็อก** `AR_SOURCE_NOT_SUPPORTED` ของ folio; ถ้าจะปลดต้องเพิ่ม guard posting + CHECK ก่อน (R-1)
5. ขอ sample payload ของ PMS จริง (Cloudbeds) 1 ชุดเพื่อกำหนด field ของ interface batch
