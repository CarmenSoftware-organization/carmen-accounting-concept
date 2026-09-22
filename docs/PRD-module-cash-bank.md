# Carmen Cloud ERP — Cash & Bank Management Module

**Module Code:** CB  
**Version:** 1.0  
**Last Updated:** 2026-09-22  
**Parent Document:** [PRD-accounting-system-overview.md](./PRD-accounting-system-overview.md)  
**Standard Compliance:** USALI 12th Edition, TFRS, Thai Banking Regulations  

---

## 1. Module Overview

The Cash & Bank Management module handles all cash and banking operations for hospitality businesses, including payment execution (AP), receipt processing (AR), bank account management, bank reconciliation, and petty cash operations.

### 1.1 Key Capabilities

- Bank account setup and management
- Payment Voucher (PV) — execution of AP invoice payments
- Receipt Voucher (RV) — recording of AR invoice receipts
- Bank Reconciliation — matching bank statements to system transactions
- Petty Cash management with replenishment workflow
- Cash Flow tracking and reporting
- Multi-currency payment/receipt with FX gain/loss

### 1.2 GL Posting Mode

**Manual review before GL posting** — bank-related journal entries require accountant review before posting to GL. This ensures bank reconciliation differences are investigated and payment/receipt amounts are validated against actual bank movements.

---

## 2. Sub-Modules

### 2.1 Payment Voucher (PV)

The Payment Voucher is the execution document for paying AP invoices. It bridges the AP module and the bank.

#### Document Flow

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
│ AP Invoice│──▶│  Payment     │──▶│   Payment    │──▶│  Bank    │
│ (Posted)  │    │  Selection   │    │   Voucher    │    │ Account  │
│           │    │  (batch or   │    │   (PV)       │    │ Outflow  │
│           │    │   single)    │    │              │    │          │
└──────────┘    └──────────────┘    └──────────────┘    └──────────┘
```

#### Key Fields

| Field | Type | Description |
|-------|------|-------------|
| `pv_no` | VARCHAR(30) | Auto-generated: PV{YY}{MM}-{NNNN} |
| `payment_date` | DATE | Date the payment is prepared |
| `paid_date` | DATE | Actual date of bank transfer/cheque clearance |
| `bank_account_id` | UUID | Source bank account |
| `payment_method` | ENUM | Transfer, Cheque, Cash, Credit Card |
| `currency_code` | VARCHAR(3) | Payment currency |
| `exchange_rate` | DECIMAL(12,5) | Rate at payment date |
| `total_paid_amount` | DECIMAL(16,2) | Total payment in transaction currency |
| `base_paid_amount` | DECIMAL(16,2) | Total payment in base currency |
| `wht_amount` | DECIMAL(16,2) | Withholding tax deducted |
| `bank_charge_amount` | DECIMAL(16,2) | Bank fees/charges |
| `cheque_no` | VARCHAR(20) | Cheque number (if payment by cheque) |

#### GL Entry Pattern

```
Dr.  Accounts Payable (2110000)     xxx.xx   (Reduce AP balance)
Dr.  Bank Charges (6xxx)            xxx.xx   (If any)
  Cr.  Bank Account (1120xxx)       xxx.xx   (Cash outflow)
  Cr.  WHT Payable (2180000)        xxx.xx   (If WHT deducted)
  Cr.  Realized FX Gain (8020000)   xxx.xx   (If applicable)
Dr.  Realized FX Loss (8010000)     xxx.xx   (If applicable)
```

#### Payment Methods

| Method | Description | Additional Fields |
|--------|-------------|-------------------|
| Bank Transfer | Direct transfer to vendor's bank | Transfer reference number |
| Cheque | Physical cheque issuance | Cheque number, cheque date, payee name |
| Cash | Cash payment | Petty cash fund reference |
| Credit Card | Corporate credit card payment | Card reference, statement period |

#### Batch Payment

- Select multiple AP invoices for the same vendor
- System generates a single PV with consolidated payment
- WHT is calculated and aggregated across all selected invoices
- Supports partial payment per invoice line

### 2.2 Receipt Voucher (RV)

The Receipt Voucher records customer payments received against AR invoices.

#### Key Fields

| Field | Type | Description |
|-------|------|-------------|
| `rv_no` | VARCHAR(30) | Auto-generated: RV{YY}{MM}-{NNNN} |
| `receipt_date` | DATE | Date the receipt is recorded |
| `cleared_date` | DATE | Date the funds cleared in bank |
| `bank_account_id` | UUID | Destination bank account |
| `receipt_method` | ENUM | Transfer, Cheque, Cash, Credit Card |
| `customer_code` | VARCHAR(20) | Customer making payment |
| `currency_code` | VARCHAR(3) | Receipt currency |
| `exchange_rate` | DECIMAL(12,5) | Rate at receipt date |
| `total_received_amount` | DECIMAL(16,2) | Total received |
| `discount_amount` | DECIMAL(16,2) | Early payment discount |

#### GL Entry Pattern

```
Dr.  Bank Account (1120xxx)         xxx.xx   (Cash inflow)
Dr.  Cash Discount Given (6xxx)     xxx.xx   (If early payment discount)
  Cr.  Accounts Receivable (1130000) xxx.xx  (Reduce AR balance)
  Cr.  Realized FX Gain (8020000)   xxx.xx   (If applicable)
Dr.  Realized FX Loss (8010000)     xxx.xx   (If applicable)
```

### 2.3 Bank Reconciliation

Bank reconciliation matches bank statement entries against system transactions to identify discrepancies.

#### Reconciliation Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                   Bank Reconciliation Process                     │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐       │
│  │  Import  │──▶│  Auto-Match  │──▶│  Manual Match    │       │
│  │  Bank    │    │  (amount +   │    │  (unmatched      │       │
│  │  State-  │    │   date +     │    │   items)         │       │
│  │  ment    │    │   reference) │    │                  │       │
│  └──────────┘    └──────────────┘    └────────┬─────────┘       │
│                                                │                 │
│                                                ▼                 │
│                                    ┌──────────────────┐         │
│                                    │  Review & Post   │         │
│                                    │  Adjustments     │         │
│                                    │                  │         │
│                                    │  - Bank charges  │         │
│                                    │  - Interest      │         │
│                                    │  - Unidentified  │         │
│                                    └──────────────────┘         │
└──────────────────────────────────────────────────────────────────┘
```

#### Bank Statement Import

| Format | Description |
|--------|-------------|
| CSV | Standard comma-separated format |
| MT940 | SWIFT standard bank statement format |
| OFX | Open Financial Exchange format |
| Manual Entry | Direct entry for small-volume banks |

#### Reconciliation Status

| Status | Description |
|--------|-------------|
| Unreconciled | No match found between bank and system |
| Auto-Matched | System automatically matched by amount + date + reference |
| Manually Matched | Accountant manually linked bank entry to system transaction |
| Adjustment | Bank charge, interest, or other item requiring new JV |

#### Reconciliation Statement

$$
\text{Bank Balance (per statement)} \pm \text{Outstanding Items} = \text{Book Balance (per GL)}
$$

Outstanding items include:
- **Outstanding Cheques:** Cheques issued but not yet cleared at bank
- **Deposits in Transit:** Receipts recorded in system but not yet reflected in bank statement
- **Bank Errors:** Items in bank statement not belonging to the account

### 2.4 Petty Cash

Petty cash management for small, routine cash expenses.

#### Petty Cash Flow

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
│ Setup    │──▶│  Expense     │──▶│  Replenish-  │──▶│  GL      │
│ Petty    │    │  Voucher     │    │  ment        │    │ Posting  │
│ Cash Fund│    │  (Small      │    │  Request     │    │          │
│          │    │   expenses)  │    │  (PV from    │    │          │
│          │    │              │    │   main bank) │    │          │
└──────────┘    └──────────────┘    └──────────────┘    └──────────┘
```

#### Key Fields

| Field | Type | Description |
|-------|------|-------------|
| `fund_code` | VARCHAR(10) | Petty cash fund identifier |
| `fund_limit` | DECIMAL(16,2) | Maximum fund amount |
| `custodian_user_id` | VARCHAR(50) | Responsible person |
| `current_balance` | DECIMAL(16,2) | Current cash on hand |
| `location_code` | VARCHAR(10) | Physical location of the fund |

#### Imprest System

The petty cash follows an **imprest system**:
1. Fund is established at a fixed amount (e.g., 10,000 THB)
2. Cash disbursements are recorded as expense vouchers
3. When the fund runs low, a replenishment PV is created
4. Replenishment amount = total of expense vouchers since last replenishment
5. Fund balance is restored to the original imprest amount

---

## 3. Cash Flow Tracking

### 3.1 Cash Flow Categories

Cash flows are categorized following **Thai Accounting Standard (TAS) No. 7** (equivalent to IAS 7):

| Category | Examples |
|----------|---------|
| **Operating** | AP payments, AR receipts, petty cash expenses, tax payments |
| **Investing** | Fixed asset purchases, asset disposals |
| **Financing** | Loan drawdowns, loan repayments, dividend payments |

### 3.2 Cash Flow Assignment

Each payment/receipt transaction is tagged with a cash flow category:
- AP payments → Operating (default) or Investing (for asset purchases)
- AR receipts → Operating
- FA-related payments → Investing
- The category is derived from the GL account code mapping

---

## 4. Key Entity Summary

| Entity | Table Name | Description |
|--------|-----------|-------------|
| Bank Account | `cb_bank_account` | Bank account master data |
| Payment Voucher Header | `cb_payment_voucher_header` | Payment execution records |
| Payment Voucher Detail | `cb_payment_voucher_detail` | Payment line items (linked to AP invoices) |
| Receipt Voucher Header | `cb_receipt_voucher_header` | Receipt records |
| Receipt Voucher Detail | `cb_receipt_voucher_detail` | Receipt line items (linked to AR invoices) |
| Bank Statement | `cb_bank_statement` | Imported bank statement data |
| Bank Reconciliation | `cb_bank_reconciliation` | Reconciliation session records |
| Reconciliation Match | `cb_reconciliation_match` | Matched bank-to-system items |
| Petty Cash Fund | `cb_petty_cash_fund` | Petty cash fund setup |
| Petty Cash Voucher | `cb_petty_cash_voucher` | Petty cash expense records |

---

## 5. Integration Points

| Direction | System/Module | Description |
|-----------|--------------|-------------|
| **Inbound** | AP | Invoices pending payment → PV creation |
| **Inbound** | AR | Receipts to record → RV creation |
| **Inbound** | Bank (external) | Bank statement import for reconciliation |
| **Outbound** | GL | Journal entries for payments, receipts, adjustments (manual review) |
| **Outbound** | Tax Management | WHT certificate data from payment vouchers |
| **Reference** | Bank Account Master | Account details, GL mapping |
| **Reference** | Vendor / Customer Master | Payee/payer information |
