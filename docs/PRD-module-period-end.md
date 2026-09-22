# Carmen Cloud ERP — Period End / Closing Module

**Module Code:** PE  
**Version:** 1.0  
**Last Updated:** 2026-09-22  
**Parent Document:** [PRD-accounting-system-overview.md](./PRD-accounting-system-overview.md)  
**Standard Compliance:** USALI 12th Edition, TFRS, Thai Accounting Standards  

---

## 1. Module Overview

The Period End / Closing module manages the accounting period lifecycle — from opening periods through month-end closing procedures to year-end closing with retained earnings rollover. It orchestrates the sequence of closing activities across all accounting modules and enforces period locking to maintain data integrity.

### 1.1 Key Capabilities

- Accounting period management (Open, Soft Close, Hard Close)
- Month-end closing checklist with workflow
- Foreign currency revaluation (unrealized FX gain/loss)
- Accrual and prepaid expense processing
- Year-end closing with Retained Earnings rollover
- Period lock enforcement across all modules
- Closing journal auto-generation
- Reopening controls with audit trail

### 1.2 GL Posting Mode

**Auto-post** for system-generated entries — closing entries such as FX revaluation, accruals, and retained earnings rollover are automatically posted to GL as they follow predefined templates and calculations.

---

## 2. Period Lifecycle

### 2.1 Period States

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
│   Open   │──▶│  Soft Close  │──▶│  Hard Close  │──▶│  Locked  │
│          │    │              │    │              │    │          │
│ All      │    │ No new docs  │    │ No changes   │    │ Permanent│
│ modules  │    │ but edits    │    │ at all.      │    │ archive  │
│ active   │    │ allowed for  │    │ Period is    │    │ state    │
│          │    │ adjustments  │    │ finalized.   │    │          │
└──────────┘    └──────────────┘    └──────────────┘    └──────────┘
                       │                    │
                       │ Reopen             │ Reopen (FC Manager + Audit)
                       ▼                    ▼
                ┌──────────┐         ┌──────────────┐
                │   Open   │         │  Soft Close  │
                └──────────┘         └──────────────┘
```

### 2.2 Period State Rules

| State | New Documents | Edit Existing | Post to GL | Reopen By |
|-------|--------------|---------------|------------|-----------|
| **Open** | ✓ | ✓ | ✓ | N/A |
| **Soft Close** | ✗ | ✓ (adjustments only) | ✓ (adjustments only) | Senior Accountant |
| **Hard Close** | ✗ | ✗ | ✗ | FC Manager (with audit trail) |
| **Locked** | ✗ | ✗ | ✗ | System Admin (emergency only) |

### 2.3 Module-Level Period Control

Each module can have independent period status, allowing staggered closing:

| Module | Typical Close Order | Notes |
|--------|-------------------|-------|
| Inventory (existing) | 1st | Must close before AP (cost finalization) |
| Accounts Payable | 2nd | All invoices for the period must be entered |
| Accounts Receivable | 3rd | Revenue recognition complete |
| Cash & Bank | 4th | Bank reconciliation complete |
| Fixed Assets | 5th | Depreciation run executed |
| Tax Management | 6th | Tax registers reconciled |
| General Ledger | Last | Final adjustments, closing entries |

---

## 3. Month-End Closing

### 3.1 Closing Checklist

The closing checklist is a **configurable workflow** that tracks required tasks before a period can be closed:

```
┌──────────────────────────────────────────────────────────────────┐
│            Month-End Closing Checklist — June 2027               │
│                                                                  │
│  ☑ 1. Verify all AP invoices entered for the period             │
│  ☑ 2. Verify all AR invoices and PMS postings received          │
│  ☑ 3. Complete bank reconciliation for all accounts             │
│  ☑ 4. Run fixed asset depreciation                              │
│  ☑ 5. Process foreign currency revaluation                      │
│  ☑ 6. Process accruals and prepaid amortization                 │
│  ☑ 7. Review and post pending GL journal entries                │
│  ☑ 8. Reconcile input tax and output tax registers              │
│  ☐ 9. Review trial balance for anomalies                        │
│  ☐ 10. Generate financial statements                            │
│  ☐ 11. FC Manager sign-off                                      │
│                                                                  │
│  Status: 8/11 Complete                    [Close Period]         │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 Checklist Configuration

Each checklist item has:

| Field | Type | Description |
|-------|------|-------------|
| `seq_no` | INT | Execution order |
| `task_name` | VARCHAR(255) | Task description |
| `task_type` | ENUM | 'MANUAL', 'AUTOMATED', 'APPROVAL' |
| `module_code` | VARCHAR(10) | Related module (AP, AR, CB, FA, GL, TAX) |
| `is_blocking` | BOOLEAN | If true, must complete before proceeding |
| `assigned_role` | VARCHAR(50) | Role responsible for the task |
| `auto_action` | VARCHAR(50) | System action to execute (e.g., 'RUN_DEPRECIATION') |

---

## 4. Foreign Currency Revaluation

### 4.1 Purpose

At period end, all open foreign currency balances (AP, AR, bank accounts) must be revalued at the closing exchange rate to recognize **unrealized FX gains/losses**.

### 4.2 Revaluation Process

```
┌──────────────────────────────────────────────────────────────────┐
│              FX Revaluation Process                               │
│                                                                  │
│  Step 1: Determine closing exchange rate for each currency       │
│  Step 2: Identify open foreign currency balances:                │
│          - AP unpaid invoices                                    │
│          - AR outstanding invoices                               │
│          - Foreign currency bank balances                        │
│  Step 3: Calculate unrealized gain/loss per balance              │
│  Step 4: Generate revaluation journal entry                      │
│  Step 5: Auto-reverse in next period opening                     │
└──────────────────────────────────────────────────────────────────┘
```

### 4.3 Revaluation Calculation

For each open foreign currency balance:

$$
\text{Original Base Amount} = \text{Foreign Amount} \times \text{Transaction Rate}
$$

$$
\text{Revalued Base Amount} = \text{Foreign Amount} \times \text{Closing Rate}
$$

$$
\text{Unrealized FX Gain/Loss} = \text{Revalued Base Amount} - \text{Original Base Amount}
$$

**GL Entry Pattern:**

```
If Unrealized Loss:
  Dr.  Unrealized FX Loss (8xxx)         xxx.xx
    Cr.  FX Revaluation Adjustment (2xxx/1xxx) xxx.xx

If Unrealized Gain:
  Dr.  FX Revaluation Adjustment (2xxx/1xxx) xxx.xx
    Cr.  Unrealized FX Gain (8xxx)         xxx.xx

(Auto-reversed on first day of next period)
```

---

## 5. Accrual & Prepaid Processing

### 5.1 Accrual Entries

Recognize expenses incurred but not yet invoiced:

| Type | Example | GL Entry |
|------|---------|----------|
| Expense Accrual | Utilities consumed but bill not received | Dr. Expense / Cr. Accrued Liabilities |
| Revenue Accrual | Services rendered but not yet billed | Dr. Accrued Revenue / Cr. Revenue |

### 5.2 Prepaid Amortization

Allocate prepaid expenses over benefit periods:

$$
\text{Monthly Amortization} = \frac{\text{Prepaid Amount}}{\text{Number of Benefit Months}}
$$

| Type | Example | GL Entry |
|------|---------|----------|
| Prepaid Insurance | Annual insurance premium paid upfront | Dr. Insurance Expense / Cr. Prepaid Insurance |
| Prepaid Rent | Advance rent payment | Dr. Rent Expense / Cr. Prepaid Rent |

### 5.3 Recurring Entry Templates

Accruals and amortizations can be set up as **recurring templates** that auto-generate each period:

| Field | Type | Description |
|-------|------|-------------|
| `template_name` | VARCHAR(100) | Descriptive name |
| `frequency` | ENUM | 'MONTHLY', 'QUARTERLY', 'ANNUALLY' |
| `start_period` | VARCHAR(7) | First period (MM/YYYY) |
| `end_period` | VARCHAR(7) | Last period (MM/YYYY) |
| `auto_reverse` | BOOLEAN | Auto-reverse next period (for accruals) |

---

## 6. Year-End Closing

### 6.1 Year-End Process

```
┌──────────────────────────────────────────────────────────────────┐
│                    Year-End Closing Sequence                      │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐       │
│  │ Complete │──▶│  Close all   │──▶│  Close Revenue   │       │
│  │ all      │    │  12 monthly  │    │  & Expense to    │       │
│  │ month-end│    │  periods     │    │  Retained        │       │
│  │ closings │    │              │    │  Earnings        │       │
│  └──────────┘    └──────────────┘    └────────┬─────────┘       │
│                                                │                 │
│  ┌──────────┐    ┌──────────────┐              │                │
│  │ Open new │◀──│  Roll forward│◀─────────────┘                │
│  │ fiscal   │    │  Balance     │                                │
│  │ year     │    │  Sheet       │                                │
│  │ periods  │    │  balances    │                                │
│  └──────────┘    └──────────────┘                                │
└──────────────────────────────────────────────────────────────────┘
```

### 6.2 Retained Earnings Rollover

$$
\text{Retained Earnings (closing)} = \text{Retained Earnings (opening)} + \text{Net Income for Year}
$$

$$
\text{Net Income} = \sum \text{Revenue (4xxx)} - \sum \text{Expenses (5xxx-8xxx)}
$$

**Year-End Closing JV:**

```
Dr.  All Revenue Accounts (4xxx)          xxx,xxx.xx
  Cr.  All Expense Accounts (5xxx-8xxx)   xxx,xxx.xx
  Cr.  Retained Earnings (3xxx)           xxx,xxx.xx  (Net Income)
  (or Dr. Retained Earnings if Net Loss)
```

### 6.3 Balance Sheet Rollforward

After year-end closing:
- All revenue and expense accounts reset to zero
- Balance sheet accounts (Assets, Liabilities, Equity) carry forward opening balances
- Retained Earnings reflects cumulative net income/loss

---

## 7. Key Entity Summary

| Entity | Table Name | Description |
|--------|-----------|-------------|
| Accounting Period | `pe_accounting_period` | Period definitions (year, month, module, status) |
| Closing Checklist | `pe_closing_checklist` | Configurable checklist template |
| Closing Checklist Run | `pe_closing_run` | Actual closing execution per period |
| Closing Task Status | `pe_closing_task_status` | Per-task completion tracking |
| FX Revaluation Run | `pe_fx_revaluation` | FX revaluation session records |
| FX Revaluation Detail | `pe_fx_revaluation_detail` | Per-balance revaluation calculations |
| Recurring Entry Template | `pe_recurring_template` | Accrual/amortization templates |
| Year-End Closing | `pe_year_end_closing` | Year-end closing records |

---

## 8. Integration Points

| Direction | System/Module | Description |
|-----------|--------------|-------------|
| **Outbound** | All modules | Period lock enforcement (block transactions in closed periods) |
| **Outbound** | GL | Auto-post FX revaluation, accruals, year-end closing entries |
| **Outbound** | FA | Trigger depreciation run as closing checklist task |
| **Inbound** | AP / AR / CB | Open balance data for FX revaluation |
| **Inbound** | GL | Trial balance for year-end closing calculation |
| **Reference** | Currency Master | Closing exchange rates for revaluation |
