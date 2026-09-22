# Carmen Cloud ERP — Budget Control Module

**Module Code:** BC  
**Version:** 1.0  
**Last Updated:** 2026-09-22  
**Parent Document:** [PRD-accounting-system-overview.md](./PRD-accounting-system-overview.md)  
**Standard Compliance:** USALI 12th Edition Budget Framework  

---

## 1. Module Overview

The Budget Control module provides annual budget planning, real-time budget monitoring, and commitment tracking for hospitality businesses. It enforces spending discipline by checking budget availability when AP invoices and other expenses are submitted, with configurable warning and blocking thresholds.

### 1.1 Key Capabilities

- Annual budget setup by account code and cost center
- Monthly budget distribution (equal, seasonal, or manual)
- Budget vs Actual comparison with variance analysis
- Hard Commitment tracking (posted/approved amounts)
- Soft Commitment tracking (draft/pending amounts)
- Budget check on AP invoice submission
- Warning and blocking thresholds per budget line
- Budget revision workflow with audit trail
- USALI departmental budget reporting

### 1.2 GL Posting Mode

**No GL posting** — Budget Control is an advisory/enforcement layer. It does not generate journal entries. It monitors and controls spending through integration with transactional modules (primarily AP).

---

## 2. Budget Structure

### 2.1 Budget Hierarchy

```
┌──────────────────────────────────────────────────────────────┐
│                    Budget Hierarchy                            │
│                                                              │
│  ┌──────────────────────────┐                                │
│  │    Budget Year           │  e.g., Fiscal Year 2027        │
│  │    (Annual Budget)       │                                │
│  └────────────┬─────────────┘                                │
│               │ 1:N                                          │
│               ▼                                              │
│  ┌──────────────────────────┐                                │
│  │    Budget Department     │  e.g., 101 Rooms,              │
│  │    (Cost Center)         │  102 F&B, 104 A&G              │
│  └────────────┬─────────────┘                                │
│               │ 1:N                                          │
│               ▼                                              │
│  ┌──────────────────────────┐                                │
│  │    Budget Line           │  e.g., 6030005 Vehicle Fuel    │
│  │    (Account Code)        │                                │
│  └────────────┬─────────────┘                                │
│               │ 1:12                                         │
│               ▼                                              │
│  ┌──────────────────────────┐                                │
│  │    Monthly Allocation    │  Jan, Feb, ... Dec             │
│  │                          │                                │
│  └──────────────────────────┘                                │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 Budget Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Original Budget** | Initial approved annual budget | Beginning of fiscal year |
| **Revised Budget** | Adjusted budget after mid-year revision | Budget reallocation |
| **Supplementary Budget** | Additional budget allocation | Unexpected needs |

### 2.3 Monthly Distribution Methods

| Method | Description |
|--------|-------------|
| **Equal** | Annual amount ÷ 12 for each month |
| **Seasonal** | Weighted distribution based on hotel seasonality pattern (high/low season) |
| **Manual** | Manually specify amount for each month |

---

## 3. Commitment Tracking

### 3.1 Commitment Types

| Type | Definition | Source |
|------|-----------|--------|
| **Hard Commitment** | Amounts that are submitted, approved, or posted — legally binding obligations | AP Invoices (Submitted/Approved/Posted status) |
| **Soft Commitment** | Amounts in draft or pending state — tentative obligations | AP Invoices (Draft status), Purchase Orders |
| **Actual** | Amounts that have been posted to GL | GL journal entries |

### 3.2 Budget Availability Calculation

$$
\text{Available Budget} = \text{Budget Amount} - \text{Hard Commitment} - \text{Actual Spent}
$$

$$
\text{Projected Available} = \text{Available Budget} - \text{Soft Commitment}
$$

### 3.3 Budget Check on AP Submit

When an AP invoice is submitted, the system performs a budget check:

```
┌──────────────────────────────────────────────────────────────────┐
│                    Budget Check Flow                              │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐       │
│  │ AP Submit│──▶│  Budget      │──▶│  Check Against   │       │
│  │ Trigger  │    │  Lookup      │    │  Threshold       │       │
│  │          │    │  (Acc Code + │    │                  │       │
│  │          │    │   Cost Ctr + │    │                  │       │
│  │          │    │   Period)    │    │                  │       │
│  └──────────┘    └──────────────┘    └────────┬─────────┘       │
│                                                │                 │
│                              ┌─────────────────┼────────────┐    │
│                              │                 │            │    │
│                              ▼                 ▼            ▼    │
│                       ┌──────────┐      ┌──────────┐  ┌────────┐│
│                       │  Within  │      │  Warning │  │ Over   ││
│                       │  Budget  │      │  Zone    │  │ Budget ││
│                       │  ✓ Allow │      │  ⚠ Warn  │  │ ✗ Block││
│                       └──────────┘      └──────────┘  └────────┘│
└──────────────────────────────────────────────────────────────────┘
```

### 3.4 Threshold Configuration

Each budget line can define:

| Threshold | Default | Behavior |
|-----------|---------|----------|
| **Warning Level** | 80% | System shows warning toast but allows submission |
| **Blocking Level** | 100% | System blocks submission; requires budget revision or override |
| **Override Authority** | FC Manager | Role authorized to override budget blocks |

---

## 4. Budget vs Actual Reporting

### 4.1 Report Structure (USALI Departmental)

```
┌──────────────────────────────────────────────────────────────────┐
│           Budget vs Actual Report — Department 102 (F&B)         │
│           Period: June 2027                                      │
├──────────────┬──────────┬──────────┬──────────┬─────────────────┤
│ Account      │ Budget   │ Actual   │ Variance │ % Utilized      │
│              │ (Month)  │ (Month)  │          │                 │
├──────────────┼──────────┼──────────┼──────────┼─────────────────┤
│ 4020 F&B Rev │ 500,000  │ 520,000  │ +20,000  │ 104.0% ✓       │
│ 5020 Cost of │ 150,000  │ 165,000  │ -15,000  │ 110.0% ⚠       │
│   Sales      │          │          │          │                 │
│ 6020 Payroll │ 200,000  │ 190,000  │ +10,000  │ 95.0% ✓        │
│ 6025 Supplies│ 30,000   │ 35,000   │ -5,000   │ 116.7% ✗       │
│ ...          │          │          │          │                 │
├──────────────┼──────────┼──────────┼──────────┼─────────────────┤
│ Total Dept   │ 880,000  │ 910,000  │ -30,000  │ 103.4%         │
└──────────────┴──────────┴──────────┴──────────┴─────────────────┘
```

### 4.2 Variance Analysis Views

| View | Description |
|------|-------------|
| **Monthly** | Current month budget vs actual |
| **YTD (Year-to-Date)** | Cumulative from start of fiscal year |
| **Full Year Forecast** | Actual YTD + remaining months budget |
| **Prior Year Comparison** | Current year vs prior year actual |

---

## 5. Key Entity Summary

| Entity | Table Name | Description |
|--------|-----------|-------------|
| Budget Header | `bc_budget_header` | Annual budget by BU (year, status, version) |
| Budget Line | `bc_budget_detail` | Budget per account + cost center (annual total) |
| Monthly Allocation | `bc_budget_monthly` | Monthly distribution per budget line |
| Commitment | `bc_commitment` | Hard/Soft commitment records (linked to AP/PO) |
| Budget Revision | `bc_budget_revision` | Revision history with approval trail |
| Budget Override Log | `bc_override_log` | Records of budget block overrides |

---

## 6. Integration Points

| Direction | System/Module | Description |
|-----------|--------------|-------------|
| **Inbound** | AP | Budget check triggered on AP invoice Submit |
| **Inbound** | GL | Actual amounts from posted journal entries |
| **Inbound** | Procurement (existing) | Soft commitment from Purchase Orders |
| **Outbound** | AP | Allow/Warn/Block response on budget check |
| **Outbound** | Reporting | Budget vs Actual data for financial reports |
| **Reference** | COA / Cost Center | Budget structure follows COA + department hierarchy |
| **Reference** | Period End | Budget amounts locked when period is closed |
