# Carmen Cloud ERP — Accounting System PRD

**Product:** Carmen Cloud ERP — Accounting Module  
**Version:** 1.1  
**Last Updated:** 2026-09-23 (architecture aligned with the implemented `micro-business` design)  
**Scope:** Full-spectrum Accounting System for Hospitality Industry  
**Standard Compliance:** USALI 12th Revised Edition, TFRS (Thai Financial Reporting Standards), Thai Revenue Department Regulations  

---

## 1. Executive Summary

The Carmen Cloud ERP Accounting System is a comprehensive, multi-module accounting platform purpose-built for the hospitality industry. It extends the existing Carmen ERP ecosystem — which currently covers procurement and inventory management — with full financial accounting capabilities.

This system is designed for **hotels, resorts, and restaurant chains** operating in Thailand and internationally, providing:

- **USALI 12th Edition** departmental accounting and reporting
- **TFRS / Thai GAAP** statutory compliance for regulatory filing
- **Thai Revenue Department** tax compliance (VAT, WHT, ภ.พ.30, ภ.ง.ด.3/53)
- **Multi-property, multi-currency** operations with configurable base currency per business unit
- **7-dimension cost analysis** for granular management reporting

### 1.1 Module Inventory

The Accounting System comprises **10 functional modules**:

| # | Module | Status | Document Reference |
|---|--------|--------|-------------------|
| 1 | General Ledger (GL) | **Implemented** (`micro-business/src/gl`) | [GL JV Fast Entry FRD V2.14](../Accounting-docs/GL/Journal%20Voucher/carmen_cloud_erp_functional_requirement_document_frd.md) |
| 2 | Accounts Payable (AP) | **Implemented** (`micro-business/src/ap`, backend-v2 PR #671) — design: [spec](./superpowers/specs/2026-09-23-accounting-foundation-ap-design.md) | [AP Module FRD v4.5.06](../Accounting-docs/AP/Invoice/carmen_cloud_erp_ap_module_functional_requirement_document_frd.md) |
| 3 | Accounts Receivable (AR) | **New** | [PRD-module-ar.md](./PRD-module-ar.md) |
| 4 | Cash & Bank Management | **New** | [PRD-module-cash-bank.md](./PRD-module-cash-bank.md) |
| 5 | Fixed Assets (FA) | **New** | [PRD-module-fa.md](./PRD-module-fa.md) |
| 6 | Budget Control | **New** | [PRD-module-budget.md](./PRD-module-budget.md) |
| 7 | Tax Management | **New** | [PRD-module-tax.md](./PRD-module-tax.md) |
| 8 | Period End / Closing | **New** | [PRD-module-period-end.md](./PRD-module-period-end.md) |
| 9 | Inter-company / Multi-property | **New** | [PRD-module-intercompany.md](./PRD-module-intercompany.md) |
| 10 | Financial Reporting | **New** | [PRD-module-reporting.md](./PRD-module-reporting.md) |

---

## 2. System Architecture

### 2.1 Integration into Carmen ERP Platform

The Accounting System is exposed through **`backend-gateway`** and implemented **inside the existing `micro-business` service** of `carmen-turborepo-backend-v2`, next to procurement, inventory and master data. There is no separate accounting microservice (decision 2026-09-23): the GL, the workflow engine, running-code numbering and tenant context already live in `micro-business`, AP must join vendor/PO/GRN data in the same tenant database, and the platform has no event bus to decouple a second service.

```
                  ┌──────────────────────────────────────────┐
                  │               Client Layer               │
                  │  React SPA  │  Platform Admin  │ Mobile  │
                  │  + Accounting UI Module                  │
                  └────────────────────┬─────────────────────┘
                                       │ HTTPS / REST (Keycloak JWT)
                                       ▼
┌────────────────────────────────────────────────────────────────────┐
│        backend-gateway (NestJS 11) — carmen-turborepo-backend-v2   │
│   api/:bu_code/gl-jv · ap-invoice · ap-payment · config/…          │
└──────┬──────────────────────────────┬──────────────────────────────┘
       │ HTTP-as-RPC (@MessagePattern, @repo/rpc-contract)           │
       ▼                              ▼
┌──────────────────────────────────┐  ┌───────────────────────────────┐
│ micro-business                   │  │ micro-cluster · micro-file ·   │
│  procurement · inventory · master│  │ micro-keycloak · micro-notif   │
│  ├── src/gl/  GL, dimensions,    │  └───────────────────────────────┘
│  │            subledger posting  │
│  ├── src/ap/  invoice, payment   │   sidecars: micro-report (FastReport),
│  └── src/<ar|cb|fa|tax|…>/ (next)│             micro-cronjobs, micro-data
└──────────────┬───────────────────┘
               ▼
     ┌──────────────────────────────┐
     │ PostgreSQL                   │
     │  platform schema (cluster/BU)│
     │  tenant schema per BU (tb_*) │
     └──────────────────────────────┘
```

**Module layout inside `micro-business`:** one flat NestJS module per feature (e.g. `gl-dimension`, `gl-subledger-posting`, `ap-invoice`, `ap-payment`, `bank-account`), each registered in `app.module.ts`, with a matching gateway module under `apps/backend-gateway/src/application/` or `src/config/`.

**Posting contract:** subledgers never write `tb_gl_*` tables. They call `GlSubledgerPostingService.postFromSource / reverseBySource` inside a single transaction (`runAtomic`), idempotent per `(source, source_ref_type, source_ref_id)`. GL void/reverse of a subledger-generated voucher is refused — the source document must be voided instead.

### 2.2 Technology Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| API entry | `backend-gateway` | NestJS 11 gateway: HTTP routing, Keycloak auth, `AppIdGuard`, `@Permission` |
| Domain service | `micro-business` | NestJS 11 on Bun; hosts GL, AP and the remaining accounting modules |
| ORM | Prisma | `prisma-shared-schema-platform` + `prisma-shared-schema-tenant` |
| Database | PostgreSQL | Platform schema (cross-tenant) + one tenant schema per BU |
| Communication | HTTP-as-RPC | `@repo/nest-http-transport`, `@MessagePattern`, generated `@repo/rpc-contract`; synchronous, no message queue |
| Authentication | Keycloak (OIDC/JWT) | Shared with Carmen platform |
| File Storage | MinIO via `micro-file` | Attachments as `fileToken` arrays on documents |
| Scheduled jobs | `micro-cronjobs` | e.g. GL `run-due` for scheduled posts and auto-reversal |
| Reporting | FastReport .NET (.frx) | Via existing `micro-report` / `report-render` services |

### 2.3 Multi-Tenancy Model

Accounting follows the same model as the rest of Carmen ERP:

- **Platform Schema** (shared): Cluster configuration, business unit setup (incl. base currency), user management, application roles, module licenses
- **Tenant Schema** (per-BU): All accounting transactions, master data, tax records, period controls — `TenantContextRunner` resolves the BU's schema from `bu_code` on every RPC call
- Accounting settings (control accounts, prefixes, FX accounts) live in the tenant's `tb_application_config` under the `gl_setting` key

---

## 3. Shared Design Decisions

### 3.1 Multi-Currency Architecture

| Aspect | Design |
|--------|--------|
| Base Currency | **Configurable per Business Unit** — each hotel/property can define its own base currency (e.g., THB for Bangkok hotels, USD for international properties) |
| Transaction Currency | Any active currency from the master currency list |
| Exchange Rate | Daily rates maintained in master data; auto-populated on transaction entry |
| Realized FX | Calculated on payment/settlement — difference between transaction rate and settlement rate |
| Unrealized FX | Calculated at period-end revaluation — open balances revalued at closing rate |
| Rounding | Half-Up rounding to 2 decimal places for base amounts; exchange rates stored to 5 decimal places |

### 3.2 Multi-Dimension Cost Analysis

All accounting modules support **7 configurable analysis dimensions** at the transaction line level:

| # | Dimension | Example Values | Purpose |
|---|-----------|---------------|---------|
| 1 | Market | Domestic, International, OTA | Revenue source segmentation |
| 2 | Sales | Direct, Agency, Corporate | Sales channel analysis |
| 3 | Project | Renovation 2026, IT Upgrade | Capital project tracking |
| 4 | Event | Wedding, Conference, Gala | Event-based cost tracking |
| 5 | Location | Main Building, Pool, Spa Wing | Physical location analysis |
| 6 | Channel | Website, Booking.com, Walk-in | Distribution channel |
| 7 | Guest Type | FIT, Group, Long-stay, VIP | Guest segmentation |

- Dimensions are **data-driven** (`tb_gl_dimension`, `tb_gl_dimension_value`) — the 7 above are seeded, more can be added
- Each COA account can mark a dimension **mandatory, optional or prohibited** (`tb_gl_account_dimension_rule`); rules are enforced at JV create, submit and post (system closing/reversal and template-run vouchers are exempt)
- Dimension values are maintained as master data with active/inactive status
- All modules generate journal entries with dimension data for drill-down reporting

### 3.3 GL Posting Pattern

Sub-modules interact with the General Ledger using a **hybrid posting approach**:

| Module | GL Posting Mode | Rationale |
|--------|----------------|-----------|
| Accounts Payable (AP) | **Auto-post** on final approval (or on submit when the BU has no AP workflow) | Standardized AP journal entries; high volume, consistent patterns. Implemented through the subledger posting facade |
| Fixed Assets (FA) | **Auto-post** on depreciation run and disposal | Automated monthly calculations; no manual judgment needed |
| Accounts Receivable (AR) | **Manual review** before GL posting | Revenue recognition may require judgment; PMS interface data needs validation |
| Cash & Bank (CB) | **Manual review** before GL posting | Bank reconciliation differences may need investigation before posting |
| Tax Management | **Auto-post** (indirect — via originating module) | Tax entries are embedded in AP/AR journal entries |
| Budget Control | **No GL posting** | Budget is a control/advisory layer, not a transactional module |
| Period End | **Auto-post** for system-generated entries (FX reval, accruals) | Closing entries follow predefined templates |
| Inter-company | **Manual review** before GL posting | IC entries require bilateral confirmation |

### 3.4 Approval Workflow (LOA)

All transactional modules reuse the **Level of Authority (LOA) workflow** pattern through the shared `WorkflowOrchestratorService` in `micro-business` (workflow types `gl_jv`, `ap_invoice`, `ap_payment`, …). Implemented documents use the statuses `draft → in_review → posted → void`; *reject* returns the document to draft:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Standard LOA Workflow Pattern                        │
│                                                                        │
│  ┌──────┐     ┌───────────┐     ┌───────────┐     ┌──────────┐       │
│  │Draft │────▶│ Submitted │────▶│In Progress│────▶│  Posted  │       │
│  │      │     │ (Step 1)  │     │ (Step 2-N)│     │          │       │
│  └──────┘     └─────┬─────┘     └─────┬─────┘     └──────────┘       │
│                     │                 │                                │
│                     │ Send Back       │ Send Back                     │
│                     ▼                 ▼                                │
│               ┌───────────┐     ┌───────────┐                         │
│               │  Returned │     │  Returned │                         │
│               └───────────┘     └───────────┘                         │
│                                                                        │
│  Alternative: No Workflow Mode                                         │
│  ┌──────┐     ┌──────────┐     ┌──────────┐                          │
│  │Draft │────▶│  Posted  │────▶│  Voided  │                          │
│  └──────┘     └──────────┘     └──────────┘                          │
└─────────────────────────────────────────────────────────────────────────┘
```

- Step count and authority limits are **configurable per module per business unit**
- Each step can define: role requirement, amount threshold, and approver assignment
- All approval actions are recorded in the audit trail

### 3.5 Audit Trail

Every accounting module must implement a consistent audit trail:

| Field | Description |
|-------|-------------|
| `action` | CREATE, UPDATE, SUBMIT, APPROVE, SEND_BACK, VOID, POST, REVERSE |
| `description` | Human-readable summary of the change |
| `old_value` / `new_value` | For field-level change tracking |
| `performed_by` | User ID of the actor |
| `performed_at` | Timestamp (UTC) |
| `ip_address` | Client IP address |

Audit logs are **immutable** — they cannot be modified or deleted. Implementation: row-level audit through the Prisma audit extension (`@repo/log-events-library`) for GL/master tables, and action-level activity entries (`activity-registry.ts`) for document modules such as AP.

---

## 4. Entity Relationship Overview

### 4.1 Cross-Module Entity Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MASTER DATA (Shared)                                │
│                                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ Chart of │  │  Cost    │  │ Currency │  │   Tax    │  │ Dimension│    │
│  │ Accounts │  │  Center  │  │  Master  │  │ Profile  │  │  Master  │    │
│  │ (COA)    │  │  (Dept)  │  │ + Rates  │  │ VAT/WHT  │  │ (7 dims) │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
│       │              │             │              │              │          │
│       └──────────────┴─────────────┴──────────────┴──────────────┘          │
│                                    │                                        │
│  ┌──────────┐  ┌──────────┐       │       ┌──────────┐  ┌──────────┐      │
│  │  Vendor  │  │ Customer │       │       │   Bank   │  │  Asset   │      │
│  │  Master  │  │  Master  │       │       │ Account  │  │ Category │      │
│  └────┬─────┘  └────┬─────┘       │       └────┬─────┘  └────┬─────┘      │
└───────┼──────────────┼─────────────┼────────────┼─────────────┼────────────┘
        │              │             │            │             │
        ▼              ▼             ▼            ▼             ▼
┌──────────────┐ ┌──────────────┐ ┌────────────────┐ ┌──────────────────┐
│  Accounts    │ │  Accounts    │ │  General       │ │  Fixed Assets    │
│  Payable     │ │  Receivable  │ │  Ledger        │ │                  │
│              │ │              │ │                │ │                  │
│ AP Invoice   │ │ AR Invoice   │ │ Journal Voucher│ │ Asset Register   │
│ AP Payment   │ │ AR Receipt   │ │ Trial Balance  │ │ Depreciation     │
│ AP Aging     │ │ AR Aging     │ │ Financial Stmt │ │ Disposal         │
└──────┬───────┘ └──────┬───────┘ └───────┬────────┘ └────────┬─────────┘
       │                │                 │                    │
       │                │                 │                    │
       ▼                ▼                 ▼                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     Cash & Bank Management                               │
│                                                                          │
│  Payment Voucher  │  Receipt Voucher  │  Bank Reconciliation  │ Petty Cash│
└──────────────────────────────────┬───────────────────────────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
             ┌──────────┐  ┌──────────┐  ┌──────────────┐
             │  Budget  │  │   Tax    │  │  Period End  │
             │ Control  │  │ Manage-  │  │  / Closing   │
             │          │  │  ment    │  │              │
             └──────────┘  └──────────┘  └──────────────┘
                                                │
                                                ▼
                                    ┌──────────────────────┐
                                    │ Inter-company /      │
                                    │ Multi-property       │
                                    │ Consolidation        │
                                    └──────────────────────┘
                                                │
                                                ▼
                                    ┌──────────────────────┐
                                    │ Financial Reporting  │
                                    │ USALI + TFRS         │
                                    └──────────────────────┘
```

### 4.2 Module Integration Points

| From Module | To Module | Integration Description |
|-------------|-----------|------------------------|
| AP | GL | Auto-post journal entries on invoice submit/approval |
| AP | Cash & Bank | Payment voucher creation from AP invoice |
| AP | Tax | Input tax (VAT) and WHT certificate generation |
| AP | Budget | Budget commitment check on AP submit |
| AP | Procurement (existing) | Invoice creation from GRN/PO |
| AR | GL | Manual review + GL posting for revenue recognition |
| AR | Cash & Bank | Receipt voucher creation from AR receipt |
| AR | Tax | Output tax (VAT) generation |
| AR | PMS (external) | Guest folio interface for city ledger postings |
| FA | GL | Auto-post depreciation and disposal entries |
| FA | AP | Asset acquisition from AP invoice |
| Cash & Bank | GL | Manual review + GL posting |
| Cash & Bank | Bank (external) | Bank statement import for reconciliation |
| Period End | GL | Auto-post closing entries, FX revaluation |
| Period End | All modules | Period lock enforcement |
| IC | GL | Inter-company elimination entries |
| IC | Period End | Consolidated closing |
| Reporting | GL | Data source for all financial reports |
| Reporting | All modules | Drill-down into source transactions |

---

## 5. Inter-Module Data Flow

### 5.1 Procure-to-Pay-to-GL Flow

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Purchase │──▶│ Purchase │──▶│   GRN    │──▶│    AP    │──▶│ Payment  │
│ Request  │    │  Order   │    │ Receive  │    │ Invoice  │    │ Voucher  │
│(existing)│    │(existing)│    │(existing)│    │(existing)│    │  (NEW)   │
└──────────┘    └──────────┘    └──────────┘    └────┬─────┘    └────┬─────┘
                                                     │               │
                                          ┌──────────┘               │
                                          ▼                          ▼
                                    ┌──────────┐              ┌──────────┐
                                    │  GL JV   │              │   Bank   │
                                    │(auto-post│              │  Recon   │
                                    │  AP→GL)  │              │  (NEW)   │
                                    └──────────┘              └──────────┘
```

### 5.2 Revenue-to-Receipt-to-GL Flow

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│   PMS    │──▶│  City    │──▶│    AR    │──▶│ Receipt  │
│  System  │    │ Ledger   │    │ Invoice  │    │ Voucher  │
│(external)│    │ Posting  │    │  (NEW)   │    │  (NEW)   │
└──────────┘    └──────────┘    └────┬─────┘    └────┬─────┘
                                     │               │
                          ┌──────────┘               │
                          ▼                          ▼
                    ┌──────────┐              ┌──────────┐
                    │  GL JV   │              │   Bank   │
                    │(manual   │              │  Recon   │
                    │ review)  │              │  (NEW)   │
                    └──────────┘              └──────────┘
```

### 5.3 Period End Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                    Month-End Closing Sequence                     │
│                                                                  │
│  Step 1          Step 2          Step 3          Step 4          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │Run FA    │──▶│Run FX    │──▶│Process   │──▶│Review &  │    │
│  │Deprec.  │   │Revaluation│  │Accruals  │   │Close     │    │
│  │(auto-GL) │   │(auto-GL) │   │(auto-GL) │   │Period    │    │
│  └──────────┘   └──────────┘   └──────────┘   └────┬─────┘    │
│                                                      │          │
│  Step 5          Step 6          Step 7              │          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐         │          │
│  │Generate  │──▶│IC Elimi- │──▶│Generate  │◀────────┘          │
│  │Tax       │   │nation    │   │Financial │                    │
│  │Reports   │   │Entries   │   │Statements│                    │
│  └──────────┘   └──────────┘   └──────────┘                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. Master Data Dependencies

The following master data entities are **prerequisites** for the Accounting System and must be set up before transactional processing:

### 6.1 Chart of Accounts (COA)

- Hierarchical account structure following USALI 12th Edition
- Account classification: Asset (1xxx), Liability (2xxx), Equity (3xxx), Revenue (4xxx), Cost of Sales (5xxx), Expense (6xxx-7xxx), Other Income/Expense (8xxx)
- Each account defines: allowed cost centers, mandatory dimensions, posting rules
- Existing FRD: Master Data / COA documentation

### 6.2 Cost Center (Department)

- USALI departmental structure: Rooms (101), Food & Beverage (102-103), Spa (105), Admin & General (104), Sales & Marketing (106), etc.
- Each cost center maps to USALI report sections
- Existing FRD: Master Data / Cost Center documentation

### 6.3 Currency & Exchange Rate

- Currency master with ISO 4217 codes
- Daily exchange rate table (auto-populated or manual entry)
- Base currency defined per business unit (configurable)

### 6.4 Tax Profile

- VAT profiles: VAT07_ADD (7% exclusive), VAT07_INC (7% inclusive), VAT_UNCLAIM, NONE
- WHT profiles: WHT01 (1% transport), WHT02 (2% advertising), WHT03 (3% services), WHT05 (5% rental), etc.
- Each profile defines: tax rate, GL account code, tax form type — implemented by extending the existing `tb_tax_profile` with `tax_type` (vat/wht), `chart_of_accounts_id`, `wht_pnd_form`, `wht_income_type`
- Existing FRD: Master Data / WHT documentation

### 6.5 Vendor & Customer Master

- Vendor master: used by AP module (existing in Carmen procurement)
- Customer master: **new** for AR module — guest accounts, corporate clients, travel agents
- Each master record defines: default currency, credit terms, tax profile, GL account mapping — `tb_vendor` now carries `default_currency_*`, `credit_term_*` and `ap_chart_of_accounts_id` as AP defaults

### 6.6 Bank Account Master

- Implemented as `tb_bank_account` (code, bank, branch, account number, currency, GL account) — used by AP payment vouchers today; Cash & Bank will extend it
- Account type (current/savings) and reconciliation data come with the Cash & Bank module
- Used for payment vouchers, receipt vouchers, and bank reconciliation

### 6.7 Asset Category

- **New** master data for Fixed Assets module
- Category hierarchy: Land, Buildings, Furniture & Equipment, Vehicles, IT Equipment, etc.
- Each category defines: depreciation method, useful life, GL account mapping (asset, accumulated depreciation, depreciation expense)
- Existing mockup: Master Data / Asset Category documentation

---

## 7. Non-Functional Requirements

### 7.1 Performance

| Metric | Target |
|--------|--------|
| Screen load time | < 1.5 seconds |
| API response (Save/Submit) | < 2.5 seconds |
| Client-side calculation (real-time) | < 100ms for 200+ line items |
| Report generation | < 10 seconds for standard reports |
| Bank statement import | < 15 seconds for 1,000 transactions |
| Depreciation batch run | < 30 seconds for 5,000 assets |

### 7.2 Security

- **RBAC**: Role-based access control via Keycloak, with module-level and function-level permissions
- **Audit Trail**: Immutable activity logs for all transactional operations
- **Data Encryption**: AES-256 at rest, TLS 1.3 in transit
- **Segregation of Duties**: Enforced separation between document creator, reviewer, and approver roles
- **Data Isolation**: Tenant schema isolation prevents cross-BU data access

### 7.3 Localization

- Bilingual UI: Thai (TH) and English (EN) with instant switching
- Number formatting: Thai (comma for thousands, period for decimals) and international formats
- Date formatting: Thai Buddhist Era (พ.ศ.) and Gregorian (ค.ศ.) support
- OKLCH color palette supporting Light Mode and Dark Mode

### 7.4 Availability & Reliability

- 99.5% uptime SLA
- Database transactions with ACID compliance for all financial operations
- Automatic rollback on partial failures (e.g., auto-reverse JV creation must be atomic with parent JV)

---

## 8. Glossary

| Term (EN) | Term (TH) | Description |
|-----------|-----------|-------------|
| USALI | - | Uniform System of Accounts for the Lodging Industry |
| TFRS | มาตรฐานการรายงานทางการเงินของไทย | Thai Financial Reporting Standards |
| COA | ผังบัญชี | Chart of Accounts |
| GL | บัญชีแยกประเภททั่วไป | General Ledger |
| JV | สมุดรายวันทั่วไป | Journal Voucher |
| AP | บัญชีเจ้าหนี้ | Accounts Payable |
| AR | บัญชีลูกหนี้ | Accounts Receivable |
| GRN | ใบรับสินค้า | Goods Received Note |
| VAT | ภาษีมูลค่าเพิ่ม | Value Added Tax |
| WHT | ภาษีหัก ณ ที่จ่าย | Withholding Tax |
| ภ.พ.30 | แบบแสดงรายการภาษีมูลค่าเพิ่ม | VAT Return Form |
| ภ.ง.ด.3 | แบบยื่นภาษีหัก ณ ที่จ่ายบุคคลธรรมดา | WHT Return — Individual |
| ภ.ง.ด.53 | แบบยื่นภาษีหัก ณ ที่จ่ายนิติบุคคล | WHT Return — Corporate |
| LOA | ลำดับขั้นอำนาจดำเนินการ | Level of Authority |
| FX | อัตราแลกเปลี่ยนเงินตราต่างประเทศ | Foreign Exchange |
| PMS | ระบบบริหารจัดการโรงแรม | Property Management System |
| City Ledger | บัญชีลูกหนี้ค้างรับ | AR sub-ledger for direct billing |
| FIFO | เข้าก่อนออกก่อน | First In First Out (inventory costing) |
| BU | หน่วยธุรกิจ | Business Unit |

---

## Appendix A: Documentation Index

### New Concept Documentation (`Accounting-docs/`)

| Document | Location | Version / Note |
|----------|----------|----------------|
| System Relations PRD | `../PRD-SYSTEM-RELATIONS.md` | 2026-08-06 |
| AP Invoice FRD | `../Accounting-docs/AP/Invoice/` | v4.5.06 (New Concept FRD) |
| AP Payment Approval Mockup | `../Accounting-docs/AP/Payment Approval/` | v2.16 (New Concept UI) |
| AP Dashboard Mockup | `../Accounting-docs/AP/Dashboard/` | v4.4.4 (New Concept UI) |
| GL JV Fast Entry FRD | `../Accounting-docs/GL/Journal Voucher/` | V2.14 (New Concept FRD) |
| GL Module Sitemap Mockup | `../Accounting-docs/GL/` | New Concept UI |
| COA Master FRD | `../Accounting-docs/Master Data/COA/` | v1.2 (New Concept FRD) |
| Cost Center FRD | `../Accounting-docs/Master Data/Cost Center (Department)/` | v1.0 (New Concept FRD) |
| Dimension FRD | `../Accounting-docs/Master Data/Dimension/` | v1.0 (New Concept FRD) |
| WHT Form FRD | `../Accounting-docs/Master Data/WHT Form/` | v1.0 (New Concept FRD) |
| WHT Service Type FRD | `../Accounting-docs/Master Data/WHT Service Type/` | v1.0 (New Concept FRD) |
| Payment Type FRD | `../Accounting-docs/Master Data/Payment Type/` | v1.0 (New Concept FRD) |
| JV Prefix FRD | `../Accounting-docs/Master Data/JV Prefix/` | v1.1 (New Concept FRD) |
| Account Code Grouping FRD | `../Accounting-docs/Master Data/Account Code Grouping/` | v1.2 (New Concept FRD) |
| Asset Category FRD | `../Accounting-docs/Master Data/Asset Category/` | v1.0 (New Concept FRD) |
| Title FRD | `../Accounting-docs/Master Data/Title/` | v1.0 (New Concept FRD) |

### Old Concept & System Reference

| Reference | Location | Note |
|-----------|----------|------|
| `carmen-4-doc` | `../carmen-4-doc/` | Old concept documentation (Carmen 4 architecture, db schemas, workflows) |
| Legacy Carmen ERP | `../developer-carmensoftware/` | Old system codebase reference (Carmen4, carmen.web, Carmen.Report) |
