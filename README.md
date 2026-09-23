# Carmen Cloud ERP — Accounting System

> Comprehensive accounting platform for the hospitality industry, built as part of the [Carmen Cloud ERP](../PRD-SYSTEM-RELATIONS.md) ecosystem.

[![USALI 12th Edition](https://img.shields.io/badge/USALI-12th%20Edition-blue)]()
[![TFRS Compliant](https://img.shields.io/badge/TFRS-Compliant-green)]()
[![Thai Tax](https://img.shields.io/badge/Thai%20Revenue%20Dept-VAT%20%2B%20WHT-orange)]()

---

## Overview

The Carmen Accounting System is a full-spectrum, multi-module accounting platform purpose-built for **hotels, resorts, and restaurant chains**. It is implemented inside the existing `micro-business` service of [`carmen-turborepo-backend-v2`](../carmen-turborepo-backend-v2) (`apps/micro-business/src/gl`, `src/ap`, …) and exposed through `backend-gateway`, providing:

- **USALI 12th Edition** departmental accounting and reporting
- **TFRS / Thai GAAP** statutory compliance
- **Thai Revenue Department** tax compliance (VAT, WHT, ภ.พ.30, ภ.ง.ด.3/53)
- **Multi-property, multi-currency** operations with configurable base currency per business unit
- **7-dimension cost analysis** for granular management reporting

## Architecture

```text
Carmen ERP Platform
├── React SPA (Vite, React 19)          ← Frontend
└── carmen-turborepo-backend-v2         ← NestJS 11 monorepo (Bun, Turborepo)
    ├── backend-gateway                 ← Single HTTP entry point, Keycloak auth, HTTP-as-RPC to services
    ├── micro-business                  ← Domain service: procurement, inventory, master data + ACCOUNTING
    │   ├── src/gl/                     ← General Ledger (implemented) + dimensions + subledger posting facade
    │   ├── src/ap/                     ← Accounts Payable: invoice/DN/CN/deposit + payment voucher (implemented)
    │   └── src/<ar|cb|fa|tax|…>/       ← Remaining modules (planned, same service)
    ├── micro-cluster / micro-file / micro-keycloak / micro-notification
    └── PostgreSQL                      ← platform schema + one tenant schema per BU (tb_* tables)
```

## Module Inventory

| # | Module | Code | Status | PRD |
|---|--------|------|--------|-----|
| 1 | General Ledger | GL | **Implemented** — `micro-business/src/gl` | [GL JV Fast Entry FRD](../Accounting-docs/GL/Journal%20Voucher/carmen_cloud_erp_functional_requirement_document_frd.md) |
| 2 | Accounts Payable | AP | **Implemented** — `micro-business/src/ap` (backend-v2 PR #671) | [AP Module FRD v4.5.06](../Accounting-docs/AP/Invoice/carmen_cloud_erp_ap_module_functional_requirement_document_frd.md) |
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
| **Service boundary** | All accounting modules live in `micro-business` next to `src/gl` (decided 2026-09-23) — no separate accounting service |
| **GL Posting** | Subledgers never write `tb_gl_*` directly; they post through `GlSubledgerPostingService` (idempotent per source document). Hybrid: AP & FA auto-post; AR & Bank require manual review |
| **Dimensions** | 7 configurable: Market, Sales, Project, Event, Location, Channel, Guest Type |
| **Reporting** | Dual framework: USALI departmental + TFRS statutory |
| **Approval** | LOA (Level of Authority) workflow reused across all modules |
| **Multi-tenancy** | Platform schema (cross-tenant) + one tenant schema per BU, resolved per request by `TenantContextRunner` |
| **Audit Trail** | Immutable activity logs across all modules |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | NestJS 11, Bun |
| ORM | Prisma |
| Database | PostgreSQL — `prisma-shared-schema-platform` + `prisma-shared-schema-tenant` |
| Communication | HTTP-as-RPC (`@repo/nest-http-transport`, `@MessagePattern`, generated `@repo/rpc-contract`) — synchronous, no event bus |
| Auth | Keycloak (OIDC/JWT) |
| Files | MinIO |
| Scheduled jobs | `micro-cronjobs` (e.g. GL `run-due` for scheduled posts and auto-reversal) |
| Reports | FastReport .NET (.frx) via `micro-report` |
| Frontend | React 19, Vite, Tailwind 4 |

## Database Schema

The **authoritative schema** is the tenant Prisma schema in `carmen-turborepo-backend-v2/packages/prisma-shared-schema-tenant/prisma/schema.prisma`. Implemented accounting tables:

- **General Ledger (existing)**: `tb_chart_of_accounts`, `tb_gl_account_group`, `tb_gl_period`, `tb_gl_jv_prefix`, `tb_gl_jv_header`, `tb_gl_jv_detail`, `tb_gl_balance`, `tb_gl_budget*`, `tb_gl_jv_template*`, `tb_cost_center*`
- **Dimensions**: `tb_gl_dimension`, `tb_gl_dimension_value`, `tb_gl_account_dimension_rule`, `tb_gl_jv_detail_dimension`
- **Accounts Payable**: `tb_ap_invoice`, `tb_ap_invoice_detail`, `tb_ap_invoice_detail_dimension`, `tb_ap_invoice_detail_source` (GRN link), `tb_ap_invoice_reference` (CN/deposit netting), `tb_ap_invoice_tax` (ภ.พ.30 record), `tb_ap_payment`, `tb_ap_payment_detail`, `tb_ap_payment_wht`, `tb_ap_payment_expense`
- **Master data**: `tb_bank_account`; `tb_tax_profile` gains `tax_type`/WHT fields; `tb_vendor` gains AP defaults
- Migration: `20260923060040_accounting_foundation_ap`

[prisma/schema.prisma](prisma/schema.prisma) in this repo is the **original concept draft** (35+ entities for every module, written for a separate service). It is kept as a reference for the modules not yet built; do not treat its table or field names as the implemented design.

## Documentation Structure

```text
carmen-accounting-concept/
├── README.md                              ← You are here
├── prisma/
│   └── schema.prisma                      ← Concept draft (superseded by the backend-v2 tenant schema)
├── docs/
│   ├── PRD-accounting-system-overview.md  ← Master PRD
│   ├── PRD-module-ar.md                   ← Accounts Receivable
│   ├── PRD-module-cash-bank.md            ← Cash & Bank
│   ├── PRD-module-fa.md                   ← Fixed Assets
│   ├── PRD-module-budget.md               ← Budget Control
│   ├── PRD-module-tax.md                  ← Tax Management
│   ├── PRD-module-period-end.md           ← Period End / Closing
│   ├── PRD-module-intercompany.md         ← Inter-company
│   ├── PRD-module-reporting.md            ← Financial Reporting
│   └── superpowers/
│       ├── specs/2026-09-23-accounting-foundation-ap-design.md  ← Implemented design: foundation + AP
│       └── plans/2026-09-23-accounting-foundation-ap.md         ← Implementation plan (17 tasks)
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
