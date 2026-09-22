# Carmen Cloud ERP — Accounting System

> Comprehensive accounting platform for the hospitality industry, built as part of the [Carmen Cloud ERP](../PRD-SYSTEM-RELATIONS.md) ecosystem.

[![USALI 12th Edition](https://img.shields.io/badge/USALI-12th%20Edition-blue)]()
[![TFRS Compliant](https://img.shields.io/badge/TFRS-Compliant-green)]()
[![Thai Tax](https://img.shields.io/badge/Thai%20Revenue%20Dept-VAT%20%2B%20WHT-orange)]()

---

## Overview

The Carmen Accounting System is a full-spectrum, multi-module accounting platform purpose-built for **hotels, resorts, and restaurant chains**. It integrates into the existing Carmen ERP architecture as a new accounting microservice (`micro-accounting`), providing:

- **USALI 12th Edition** departmental accounting and reporting
- **TFRS / Thai GAAP** statutory compliance
- **Thai Revenue Department** tax compliance (VAT, WHT, ภ.พ.30, ภ.ง.ด.3/53)
- **Multi-property, multi-currency** operations with configurable base currency per business unit
- **7-dimension cost analysis** for granular management reporting

## Architecture

```text
Carmen ERP Platform
├── React SPA (Vite, React 19)          ← Frontend
├── carmen-turborepo-backend-v2         ← Main API Application (NestJS 11 Gateway, Auth, HTTP→TCP Bridge)
├── micro-business (existing)           ← Procurement & Inventory
├── micro-accounting (NEW)              ← This system
│   ├── General Ledger (GL)
│   ├── Accounts Payable (AP)
│   ├── Accounts Receivable (AR)
│   ├── Cash & Bank Management (CB)
│   ├── Fixed Assets (FA)
│   ├── Budget Control (BC)
│   ├── Tax Management (TAX)
│   ├── Period End / Closing (PE)
│   ├── Inter-company (IC)
│   └── Financial Reporting (RPT)
└── PostgreSQL (dual-schema multi-tenancy)
```

## Module Inventory

| # | Module | Code | Status | PRD |
|---|--------|------|--------|-----|
| 1 | General Ledger | GL | New Concept FRD | [GL JV Fast Entry FRD](../Accounting-docs/GL/Journal%20Voucher/carmen_cloud_erp_functional_requirement_document_frd.md) |
| 2 | Accounts Payable | AP | New Concept FRD | [AP Module FRD v4.5.06](../Accounting-docs/AP/Invoice/carmen_cloud_erp_ap_module_functional_requirement_document_frd.md) |
| 3 | Accounts Receivable | AR | **New** | [PRD-module-ar.md](docs/PRD-module-ar.md) |
| 4 | Cash & Bank Management | CB | **New** | [PRD-module-cash-bank.md](docs/PRD-module-cash-bank.md) |
| 5 | Fixed Assets | FA | **New** | [PRD-module-fa.md](docs/PRD-module-fa.md) |
| 6 | Budget Control | BC | **New** | [PRD-module-budget.md](docs/PRD-module-budget.md) |
| 7 | Tax Management | TAX | **New** | [PRD-module-tax.md](docs/PRD-module-tax.md) |
| 8 | Period End / Closing | PE | **New** | [PRD-module-period-end.md](docs/PRD-module-period-end.md) |
| 9 | Inter-company | IC | **New** | [PRD-module-intercompany.md](docs/PRD-module-intercompany.md) |
| 10 | Financial Reporting | RPT | **New** | [PRD-module-reporting.md](docs/PRD-module-reporting.md) |

> 📄 **Start here →** [PRD-accounting-system-overview.md](docs/PRD-accounting-system-overview.md) — master overview with architecture, shared design decisions, entity relationships, and cross-module data flows.

## Key Design Decisions

| Decision | Details |
|----------|---------|
| **Multi-currency** | Configurable base currency per BU; realized FX on payment; unrealized FX on period-end revaluation |
| **GL Posting** | Hybrid — AP & FA auto-post; AR & Bank require manual review |
| **Dimensions** | 7 configurable: Market, Sales, Project, Event, Location, Channel, Guest Type |
| **Reporting** | Dual framework: USALI departmental + TFRS statutory |
| **Approval** | LOA (Level of Authority) workflow reused across all modules |
| **Multi-tenancy** | Dual-schema: platform (cross-tenant) + tenant (per-BU) |
| **Audit Trail** | Immutable activity logs across all modules |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | NestJS 11, Bun |
| ORM | Prisma |
| Database | PostgreSQL (dual-schema) |
| Communication | TCP MessagePattern (NestJS RPC) |
| Auth | Keycloak (OIDC/JWT) |
| Files | MinIO |
| Cache/Queue | Redis (asynq) |
| Reports | FastReport .NET (.frx) |
| Frontend | React 19, Vite, Tailwind 4 |

## Database Schema

The core database design is modeled as a canonical, production-ready Prisma schema:

- 📄 [prisma/schema.prisma](prisma/schema.prisma) — Complete PostgreSQL schema covering 35+ relational entities across all accounting modules.
  - **Core Foundation**: `tb_chart_of_accounts`, `tb_dimension_type`, `tb_dimension_value`, `tb_account_dimension_rule`, `tb_fiscal_year`, `tb_fiscal_period`, `tb_period_close_history`
  - **General Ledger (Two-Tier)**: `tb_journal_voucher`, `tb_journal_voucher_line`, `tb_journal_voucher_line_dimension`, `tb_gl_entry`, `tb_gl_entry_dimension`, `tb_fx_revaluation_run`
  - **Accounts Payable**: `tb_ap_invoice`, `tb_ap_invoice_line`, `tb_ap_invoice_line_dimension`, `tb_ap_invoice_matching` (3-way match to PO/GRN), `tb_ap_payment`, `tb_ap_payment_allocation`
  - **Accounts Receivable**: `tb_customer`, `tb_customer_group`, `tb_ar_invoice`, `tb_ar_invoice_line`, `tb_ar_invoice_line_dimension`, `tb_ar_receipt`, `tb_ar_receipt_allocation`
  - **Cash & Bank**: `tb_bank`, `tb_bank_account`, `tb_petty_cash_fund`, `tb_petty_cash_expense`, `tb_bank_statement`, `tb_bank_statement_line`, `tb_bank_reconciliation`, `tb_bank_reconciliation_item`
  - **Fixed Assets**: `tb_asset_category`, `tb_fixed_asset`, `tb_asset_depreciation_run`, `tb_asset_depreciation_entry`, `tb_asset_transfer`, `tb_asset_disposal`
  - **Tax Management (Thai Revenue)**: `tb_tax_invoice` (PP30 Input/Output/Undue VAT), `tb_wht_certificate` & `tb_wht_certificate_line` (50 ทวิ, PND 3/53/54)
  - **Budget Control**: `tb_budget_plan`, `tb_budget_line`
  - **Inter-Company**: `tb_intercompany_partner` (Due-To/Due-From mappings), `tb_intercompany_transaction`

## Documentation Structure

```text
carmen-accounting-concept/
├── README.md                              ← You are here
├── prisma/
│   └── schema.prisma                      ← Canonical Accounting Prisma Schema (validated)
├── docs/
│   ├── PRD-accounting-system-overview.md  ← Master PRD
│   ├── PRD-module-ar.md                   ← Accounts Receivable
│   ├── PRD-module-cash-bank.md            ← Cash & Bank
│   ├── PRD-module-fa.md                   ← Fixed Assets
│   ├── PRD-module-budget.md               ← Budget Control
│   ├── PRD-module-tax.md                  ← Tax Management
│   ├── PRD-module-period-end.md           ← Period End / Closing
│   ├── PRD-module-intercompany.md         ← Inter-company
│   └── PRD-module-reporting.md            ← Financial Reporting
├── Accounting-docs/                       ← New Concept FRDs & Mockups (AP, GL, Master Data)
│   ├── AP/                                ← AP Invoice FRD + Mockups
│   ├── GL/                                ← GL JV FRD + Mockups
│   └── Master Data/                       ← COA, Cost Center, WHT, etc.
├── carmen-4-doc/                          ← Old Concept Documentation (Carmen 4 architecture, db, workflows)
└── developer-carmensoftware/              ← Old / Legacy System codebase (Carmen4, carmen.web, Carmen.Report)
```

## Related Repositories

| Repository | Description |
|-----------|-------------|
| `carmen-turborepo-backend-v2` | **Main API Application** (NestJS 11 Gateway & Microservices Monorepo) |
| `Accounting-docs` | New Concept FRDs & UI Mockups (AP, GL, Master Data) |
| `carmen-inventory-frontend-react` | Main ERP Frontend (React 19) |
| `carmen-platform` | Platform Admin (Cluster/BU/User management) |
| `carmen` | micro-business (Procurement & Inventory domain) |
| `carmen-4-doc` | Old Concept Documentation (Carmen 4 architecture, database, workflows) |
| `developer-carmensoftware` | Old System / Legacy ERP Codebase (Carmen4, carmen.web, Carmen.Report) |

## License

Proprietary — Carmen Software
