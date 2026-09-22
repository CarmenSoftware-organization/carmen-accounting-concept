# Carmen Cloud ERP — Accounts Receivable (AR) Module

**Module Code:** AR  
**Version:** 1.0  
**Last Updated:** 2026-09-22  
**Parent Document:** [PRD-accounting-system-overview.md](./PRD-accounting-system-overview.md)  
**Standard Compliance:** USALI 12th Edition, TFRS, Thai Revenue Department Output Tax Regulations  

---

## 1. Module Overview

The Accounts Receivable (AR) module manages revenue recognition, customer billing, and collections for hospitality businesses. It handles both **direct billing** (corporate clients, travel agents) and **PMS-interfaced guest folios** (city ledger transfers from Property Management Systems).

### 1.1 Key Capabilities

- AR Invoice creation (direct billing and PMS interface posting)
- Debit Note and Credit Note management
- Receipt processing and allocation
- City Ledger management for guest account transfers from PMS
- AR Aging analysis (current, 30, 60, 90, 120+ days)
- Output Tax (VAT Sales / ภาษีขาย) generation
- Multi-currency invoicing with configurable base currency
- 7-dimension cost analysis at line level

### 1.2 GL Posting Mode

**Manual review before GL posting** — AR journal entries are generated on submit but held in a "Pending Review" state. A senior accountant must approve the GL posting, as revenue recognition in hospitality often requires judgment (e.g., validating PMS interface data, handling disputed charges, timing of revenue recognition for advance deposits).

---

## 2. Sub-Modules & Document Types

### 2.1 AR Invoice (ARIV)

- **Purpose:** Record revenue from direct billing — corporate clients, travel agencies, and other non-PMS sources
- **Document Code:** ARIV + auto-running number (e.g., ARIV26060001)
- **Source Types:** Manual, PMS Interface, Copy, AI Upload, Excel Template
- **GL Effect:** Dr. Accounts Receivable (ลูกหนี้การค้า) / Cr. Revenue Account (รายได้)

#### Key Fields

| Field | Type | Description |
|-------|------|-------------|
| `doc_no` | VARCHAR(30) | Auto-generated document number |
| `invoice_date` | DATE | Invoice date for tax and aging purposes |
| `customer_code` | VARCHAR(20) | Customer/guest account code |
| `currency_code` | VARCHAR(3) | Transaction currency |
| `exchange_rate` | DECIMAL(12,5) | Exchange rate to base currency |
| `credit_term_days` | INT | Payment terms in days |
| `due_date` | DATE | Calculated: invoice_date + credit_term_days |
| `revenue_account_code` | VARCHAR(20) | USALI revenue account (e.g., 4010 Room Revenue) |
| `cost_center_code` | VARCHAR(10) | USALI department (e.g., 101 Rooms) |

### 2.2 AR Debit Note (ARDN)

- **Purpose:** Increase outstanding receivable amount (additional charges, adjustments)
- **Document Code:** ARDN + auto-running number
- **GL Effect:** Dr. Accounts Receivable / Cr. Revenue/Adjustment Account

### 2.3 AR Credit Note (ARCN)

- **Purpose:** Reduce outstanding receivable amount (discounts, returns, adjustments)
- **Document Code:** ARCN + auto-running number
- **GL Effect:** Dr. Revenue/Adjustment Account / Cr. Accounts Receivable
- **Special Behavior:** Auto-generates negative output tax invoice for VAT credit note submissions

### 2.4 AR Receipt (ARRC)

- **Purpose:** Record customer payments against outstanding AR invoices
- **Document Code:** ARRC + auto-running number
- **GL Effect:** Dr. Bank/Cash Account / Cr. Accounts Receivable
- **FX Handling:** Realized FX gain/loss calculated when receipt currency rate differs from invoice rate

---

## 3. PMS Interface Integration

### 3.1 City Ledger Architecture

The City Ledger is the bridge between the Property Management System and the Accounting System:

```
┌──────────────────────────────────────────────────────────────────┐
│                     PMS Interface Flow                            │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐       │
│  │   PMS    │──▶│  Interface   │──▶│  AR City Ledger   │       │
│  │  System  │    │  Posting     │    │  Transfer Queue   │       │
│  │ (Opera,  │    │  Batch       │    │                   │       │
│  │ Hotelogix│    │              │    │  - Guest charges  │       │
│  │  etc.)   │    │  - Night     │    │  - Room revenue   │       │
│  │          │    │    audit     │    │  - F&B revenue    │       │
│  │          │    │  - Check-out │    │  - Other charges  │       │
│  └──────────┘    └──────────────┘    └────────┬─────────┘       │
│                                                │                 │
│                                    ┌───────────┘                 │
│                                    ▼                             │
│                           ┌──────────────┐    ┌──────────────┐  │
│                           │  Review &    │──▶│  AR Invoice  │  │
│                           │  Validate    │    │  (ARIV)      │  │
│                           │  Interface   │    │  Source:      │  │
│                           │  Data        │    │  "Interface   │  │
│                           └──────────────┘    │   Posting"   │  │
│                                               └──────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 Interface Data Mapping

| PMS Field | AR Field | Notes |
|-----------|----------|-------|
| Folio Number | `source_doc_ref` | Reference to original PMS folio |
| Guest Name | `customer_name` | Mapped to customer master or created on-the-fly |
| Revenue Code (PMS) | `revenue_account_code` | Mapped via interface configuration table |
| Department Code (PMS) | `cost_center_code` | Mapped to USALI department |
| Charge Amount | `line_amount` | Converted at daily exchange rate |
| Tax Amount | `tax_amount` | Recalculated per Thai VAT rules |
| Transfer Date | `invoice_date` | Night audit or check-out date |

### 3.3 Interface Processing Rules

1. **Batch Import:** PMS postings are received as batches (typically nightly from night audit)
2. **Validation Queue:** All interface postings enter a review queue before AR invoice creation
3. **Revenue Code Mapping:** PMS revenue codes are mapped to USALI COA accounts via a configurable mapping table
4. **Exception Handling:** Unmapped revenue codes, zero amounts, and duplicate folios are flagged for manual review
5. **Reconciliation:** Daily PMS revenue summary must reconcile with AR interface postings

---

## 4. AR Aging Analysis

### 4.1 Aging Buckets

| Bucket | Range | Description |
|--------|-------|-------------|
| Current | 0 days | Not yet due |
| 1-30 | 1–30 days past due | First follow-up |
| 31-60 | 31–60 days past due | Second follow-up |
| 61-90 | 61–90 days past due | Escalation |
| 91-120 | 91–120 days past due | Final notice |
| 120+ | > 120 days past due | Write-off candidate |

### 4.2 Aging Calculation

$$
\text{Days Outstanding} = \text{Report Date} - \text{Due Date}
$$

- Aging is calculated based on **Due Date** (not Invoice Date)
- Partially paid invoices show only the **remaining unpaid amount** in the aging bucket
- Credit notes reduce the outstanding balance of the related invoice

### 4.3 Aging Reports

- **Summary by Customer:** Total outstanding per customer grouped by aging bucket
- **Detail by Invoice:** Individual invoice listing with aging classification
- **Department Summary:** Outstanding receivables grouped by USALI department

---

## 5. Key Entity Summary

| Entity | Table Name | Description |
|--------|-----------|-------------|
| AR Invoice Header | `ar_invoice_header` | Invoice/DN/CN/Receipt header data |
| AR Invoice Detail | `ar_invoice_detail` | Line items with account, amount, dimensions |
| AR Receipt | `ar_receipt_header` | Receipt records with payment allocation |
| AR Receipt Allocation | `ar_receipt_allocation` | Links receipts to specific invoices |
| AR City Ledger | `ar_city_ledger_posting` | PMS interface transfer queue |
| AR Tax Invoice | `ar_tax_invoice` | Output tax (VAT sales) register |
| Customer Master | `ar_customer_master` | Customer/guest account master data |
| PMS Interface Map | `ar_pms_revenue_mapping` | Revenue code mapping PMS → COA |

---

## 6. Integration Points

| Direction | System/Module | Description |
|-----------|--------------|-------------|
| **Inbound** | PMS (external) | Guest folio transfers via interface batch |
| **Outbound** | GL | Journal entries for revenue and receipts (manual review) |
| **Outbound** | Cash & Bank | Receipt voucher creation on AR receipt |
| **Outbound** | Tax Management | Output tax register entries |
| **Reference** | Customer Master | Default currency, credit terms, tax profile |
| **Reference** | COA / Cost Center | Account validation and dimension rules |
