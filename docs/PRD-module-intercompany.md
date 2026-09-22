# Carmen Cloud ERP — Inter-company / Multi-property Module

**Module Code:** IC  
**Version:** 1.0  
**Last Updated:** 2026-09-22  
**Parent Document:** [PRD-accounting-system-overview.md](./PRD-accounting-system-overview.md)  
**Standard Compliance:** USALI 12th Edition (Multi-unit Operations), TFRS 10 (Consolidated Financial Statements)  

---

## 1. Module Overview

The Inter-company / Multi-property module manages financial transactions between business units (hotels/properties) within the same cluster (hotel group), and produces consolidated financial statements by eliminating inter-company balances and transactions.

### 1.1 Key Capabilities

- Inter-company transaction recording (charges between properties)
- IC balance matching and reconciliation
- IC settlement processing
- Consolidated financial reporting across properties
- Elimination entries for consolidation
- Multi-currency IC transactions with configurable base currencies per BU
- Management fee and shared service allocation

### 1.2 GL Posting Mode

**Manual review before GL posting** — inter-company entries require bilateral confirmation (both sending and receiving BU must acknowledge the transaction) before GL posting to prevent one-sided entries.

---

## 2. Inter-company Transaction Types

### 2.1 Common IC Transaction Scenarios

| Scenario | Description | Example |
|----------|-------------|---------|
| **Shared Services** | Central office charges for shared IT, HR, finance | Head office bills each hotel for ERP license fees |
| **Management Fee** | Management company charges property owner | 3% of gross revenue as management fee |
| **Inter-property Transfer** | Goods/services transferred between properties | Hotel A sends linen to Hotel B |
| **Centralized Procurement** | Central purchasing on behalf of properties | Head office buys supplies, allocates cost to hotels |
| **Fund Transfer** | Cash movements between BU bank accounts | Head office funding a new property |
| **Asset Transfer** | Fixed asset moved between properties | Kitchen equipment transferred from Hotel A to Hotel B |

### 2.2 IC Transaction Flow

```
┌──────────────────────────────────────────────────────────────────┐
│              Inter-company Transaction Flow                       │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐       │
│  │ Sending  │──▶│  IC Trans-   │──▶│  Receiving BU    │       │
│  │ BU       │    │  action      │    │  Acknowledgment  │       │
│  │ Creates  │    │  Created     │    │                  │       │
│  │ IC Doc   │    │  (Pending)   │    │  Accept / Reject │       │
│  └──────────┘    └──────────────┘    └────────┬─────────┘       │
│                                                │                 │
│                         ┌──────────────────────┤                 │
│                         │                      │                 │
│                         ▼                      ▼                 │
│                  ┌──────────────┐        ┌──────────┐           │
│                  │  Both BUs    │        │ Rejected │           │
│                  │  Post to GL  │        │ (Dispute)│           │
│                  │  (Matched)   │        │          │           │
│                  └──────────────┘        └──────────┘           │
└──────────────────────────────────────────────────────────────────┘
```

### 2.3 IC Account Structure

Each BU maintains **IC receivable and payable accounts** per counterparty BU:

```
BU: Hotel Bangkok (BKK)
├── IC Receivable from Hotel Phuket (1135-PHK)    Dr. balance
├── IC Receivable from Hotel Chiang Mai (1135-CNX) Dr. balance
├── IC Payable to Head Office (2115-HQ)            Cr. balance
└── IC Payable to Hotel Pattaya (2115-PTY)         Cr. balance
```

---

## 3. IC Balance Reconciliation

### 3.1 Matching Process

IC balances must match between counterparty BUs — what one BU records as receivable, the other must record as payable for the same amount:

$$
\text{BU}_A \text{ IC Receivable from } \text{BU}_B = \text{BU}_B \text{ IC Payable to } \text{BU}_A
$$

### 3.2 Reconciliation Status

| Status | Description |
|--------|-------------|
| **Matched** | Both BUs have recorded the same amount — no difference |
| **Unmatched — Timing** | One BU has recorded, the other hasn't yet (timing difference) |
| **Unmatched — Amount** | Both BUs have recorded but amounts differ |
| **Disputed** | Receiving BU has rejected the transaction |

### 3.3 Reconciliation Report

```
┌──────────────────────────────────────────────────────────────────┐
│       IC Balance Reconciliation — June 2027                      │
│       BU: Hotel Bangkok (BKK) vs Hotel Phuket (PHK)            │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│ IC Trans No  │ BKK Balance  │ PHK Balance  │ Difference         │
├──────────────┼──────────────┼──────────────┼────────────────────┤
│ IC2706-001   │ 50,000 (Dr)  │ 50,000 (Cr)  │ 0 ✓ Matched       │
│ IC2706-002   │ 30,000 (Dr)  │ —            │ 30,000 ⚠ Timing   │
│ IC2706-003   │ 15,000 (Dr)  │ 14,500 (Cr)  │ 500 ✗ Amount      │
├──────────────┼──────────────┼──────────────┼────────────────────┤
│ Total        │ 95,000       │ 64,500       │ 30,500             │
└──────────────┴──────────────┴──────────────┴────────────────────┘
```

---

## 4. IC Settlement

### 4.1 Netting

Periodic netting of IC balances to reduce actual cash transfers:

$$
\text{Net Settlement} = \sum \text{IC Receivables} - \sum \text{IC Payables}
$$

- If positive: Net amount owed **to** the BU
- If negative: Net amount owed **by** the BU

### 4.2 Settlement Process

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
│ Generate │──▶│  Review &    │──▶│  Approve     │──▶│  Execute │
│ Netting  │    │  Adjust      │    │  Settlement  │    │  Fund    │
│ Statement│    │  Netting     │    │  (Both BU    │    │  Transfer│
│          │    │              │    │   sign-off)  │    │          │
└──────────┘    └──────────────┘    └──────────────┘    └──────────┘
```

---

## 5. Consolidated Financial Reporting

### 5.1 Consolidation Process

```
┌──────────────────────────────────────────────────────────────────┐
│              Consolidation Process                                │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐       │
│  │ Collect  │──▶│  Currency    │──▶│  Eliminate IC    │       │
│  │ BU Trial │    │  Translation │    │  Balances &      │       │
│  │ Balances │    │  (to Group   │    │  Transactions    │       │
│  │          │    │   base curr) │    │                  │       │
│  └──────────┘    └──────────────┘    └────────┬─────────┘       │
│                                                │                 │
│                                                ▼                 │
│                                    ┌──────────────────┐         │
│                                    │  Consolidated    │         │
│                                    │  Financial       │         │
│                                    │  Statements      │         │
│                                    └──────────────────┘         │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2 Currency Translation

When BUs have different base currencies, translation to group reporting currency is required:

| Balance Sheet Item | Translation Rate |
|-------------------|-----------------|
| Assets & Liabilities | Closing rate (period-end rate) |
| Equity | Historical rate |
| Revenue & Expenses | Average rate for the period |
| Translation difference | Accumulated in OCI (Other Comprehensive Income) |

### 5.3 Elimination Entries

IC transactions and balances must be eliminated for consolidated reporting:

| Type | Elimination Entry |
|------|------------------|
| IC Revenue/Expense | Dr. IC Revenue / Cr. IC Expense |
| IC Receivable/Payable | Dr. IC Payable / Cr. IC Receivable |
| IC Profit on Transfers | Dr. Revenue / Cr. Inventory/Asset (unrealized profit) |

---

## 6. Management Fee & Shared Service Allocation

### 6.1 Allocation Methods

| Method | Description | Use Case |
|--------|-------------|----------|
| **% of Revenue** | Fixed percentage of gross or net revenue | Management fees |
| **Headcount** | Allocated based on employee count per property | HR shared services |
| **Room Count** | Allocated based on number of rooms | IT infrastructure |
| **Equal Split** | Divided equally among participating BUs | Group marketing |
| **Custom** | User-defined allocation keys | Special projects |

### 6.2 Allocation Process

1. Define allocation template (method, base, participating BUs)
2. System calculates allocation amounts per BU
3. Generate IC transactions automatically
4. Both sending and receiving BUs acknowledge
5. Post to GL in each BU

---

## 7. Key Entity Summary

| Entity | Table Name | Description |
|--------|-----------|-------------|
| IC Transaction Header | `ic_transaction_header` | IC transaction records |
| IC Transaction Detail | `ic_transaction_detail` | Line items with accounts and amounts |
| IC Balance | `ic_balance` | Running IC balance per BU pair |
| IC Reconciliation | `ic_reconciliation` | Reconciliation session records |
| IC Settlement | `ic_settlement` | Netting and settlement records |
| IC Elimination Entry | `ic_elimination_entry` | Consolidation elimination JVs |
| Allocation Template | `ic_allocation_template` | Shared service allocation rules |

---

## 8. Integration Points

| Direction | System/Module | Description |
|-----------|--------------|-------------|
| **Outbound** | GL | IC journal entries in each BU (manual review) |
| **Outbound** | Cash & Bank | Fund transfer PVs for IC settlement |
| **Outbound** | FA | Asset transfer entries between BUs |
| **Outbound** | Reporting | Consolidated financial statements |
| **Inbound** | All modules (per BU) | Trial balance data for consolidation |
| **Inbound** | Period End | IC reconciliation as closing checklist item |
| **Reference** | Platform (existing) | Cluster and BU configuration |
