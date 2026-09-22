# Carmen Cloud ERP — Financial Reporting Module

**Module Code:** RPT  
**Version:** 1.0  
**Last Updated:** 2026-09-22  
**Parent Document:** [PRD-accounting-system-overview.md](./PRD-accounting-system-overview.md)  
**Standard Compliance:** USALI 12th Revised Edition, TFRS (Thai Financial Reporting Standards), IAS 1 (Presentation of Financial Statements)  

---

## 1. Module Overview

The Financial Reporting module produces all accounting reports required for management decision-making and regulatory compliance. It supports dual reporting frameworks: **USALI departmental reports** for hotel management and **TFRS statutory reports** for Thai regulatory filing.

### 1.1 Key Capabilities

- USALI departmental P&L (Rooms, F&B, Spa, Admin & General, etc.)
- TFRS statutory financial statements (Balance Sheet, Income Statement, Cash Flow)
- Trial Balance (summary and detailed)
- General Ledger report (account detail listing)
- Sub-ledger reports (AP, AR, FA aging/detail)
- Management dashboard KPIs
- Custom report builder integration (FastReport .frx templates)
- Drill-down from report to source transactions
- Multi-currency and multi-property reporting
- Consolidated reports (via IC module)

### 1.2 Integration Architecture

Reports are generated through the existing Carmen reporting stack:

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
│ User     │──▶│  micro-data  │──▶│ micro-report │──▶│ FastReport│
│ Request  │    │  (dataset    │    │ (template    │    │ (.frx     │
│          │    │   execution) │    │  render)     │    │  compile) │
└──────────┘    └──────────────┘    └──────────────┘    └────┬─────┘
                                                              │
                                                              ▼
                                                       ┌──────────┐
                                                       │ Output   │
                                                       │ PDF/Excel│
                                                       │ /CSV     │
                                                       └──────────┘
```

---

## 2. Report Categories

### 2.1 USALI Departmental Reports

The USALI (Uniform System of Accounts for the Lodging Industry) 12th Edition defines a standardized reporting structure for hotels.

#### Summary Operating Statement

```
┌──────────────────────────────────────────────────────────────────┐
│             SUMMARY OPERATING STATEMENT                          │
│             Hotel Bangkok — June 2027                            │
├──────────────────────────────┬───────────┬───────────┬──────────┤
│                              │  Current  │   Budget  │ Variance │
│                              │   Month   │   Month   │    %     │
├──────────────────────────────┼───────────┼───────────┼──────────┤
│ REVENUE                      │           │           │          │
│   Rooms                      │ 2,500,000 │ 2,400,000 │  +4.2%   │
│   Food & Beverage            │ 1,200,000 │ 1,100,000 │  +9.1%   │
│   Other Operated Departments │   300,000 │   280,000 │  +7.1%   │
│   Miscellaneous Income       │    50,000 │    40,000 │ +25.0%   │
│ Total Revenue                │ 4,050,000 │ 3,820,000 │  +6.0%   │
├──────────────────────────────┼───────────┼───────────┼──────────┤
│ DEPARTMENTAL EXPENSES        │           │           │          │
│   Rooms                      │   600,000 │   580,000 │  +3.4%   │
│   Food & Beverage            │   840,000 │   800,000 │  +5.0%   │
│   Other Operated Departments │   180,000 │   170,000 │  +5.9%   │
│ Total Departmental Expenses  │ 1,620,000 │ 1,550,000 │  +4.5%   │
├──────────────────────────────┼───────────┼───────────┼──────────┤
│ DEPARTMENTAL INCOME          │ 2,430,000 │ 2,270,000 │  +7.0%   │
├──────────────────────────────┼───────────┼───────────┼──────────┤
│ UNDISTRIBUTED EXPENSES       │           │           │          │
│   Administrative & General   │   350,000 │   360,000 │  -2.8%   │
│   Sales & Marketing          │   200,000 │   210,000 │  -4.8%   │
│   Property Ops & Maintenance │   180,000 │   175,000 │  +2.9%   │
│   Utilities                  │   250,000 │   240,000 │  +4.2%   │
│   IT & Telecommunications    │   100,000 │    95,000 │  +5.3%   │
│ Total Undistributed Expenses │ 1,080,000 │ 1,080,000 │   0.0%   │
├──────────────────────────────┼───────────┼───────────┼──────────┤
│ GROSS OPERATING PROFIT (GOP) │ 1,350,000 │ 1,190,000 │ +13.4%   │
├──────────────────────────────┼───────────┼───────────┼──────────┤
│ MANAGEMENT FEES              │   121,500 │   114,600 │  +6.0%   │
│ NON-OPERATING INCOME/EXPENSE │   (50,000)│   (40,000)│ +25.0%   │
├──────────────────────────────┼───────────┼───────────┼──────────┤
│ EBITDA                       │ 1,178,500 │ 1,035,400 │ +13.8%   │
├──────────────────────────────┼───────────┼───────────┼──────────┤
│ FIXED CHARGES                │           │           │          │
│   Depreciation & Amortization│   200,000 │   200,000 │   0.0%   │
│   Insurance                  │    50,000 │    50,000 │   0.0%   │
│   Property Tax               │    30,000 │    30,000 │   0.0%   │
├──────────────────────────────┼───────────┼───────────┼──────────┤
│ NET OPERATING INCOME (NOI)   │   898,500 │   755,400 │ +18.9%   │
└──────────────────────────────┴───────────┴───────────┴──────────┘
```

#### Departmental P&L Reports

Individual P&L for each operated department:

| Department | Cost Center | USALI Section |
|-----------|------------|---------------|
| Rooms | 101 | Schedule 1 |
| Food & Beverage | 102-103 | Schedule 2 |
| Spa / Recreation | 105 | Schedule 3 |
| Other Operated | 106-109 | Schedule 4 |
| Admin & General | 104 | Schedule 5 |
| Sales & Marketing | 110 | Schedule 6 |
| Property Operations | 111 | Schedule 7 |
| Utilities | 112 | Schedule 8 |
| IT & Telecom | 113 | Schedule 9 |

Each departmental P&L shows:
- Revenue lines (for operated departments)
- Cost of Sales
- Payroll & Related Expenses
- Other Expenses
- Departmental Profit/Loss

### 2.2 TFRS Statutory Reports

#### Balance Sheet (Statement of Financial Position)

| Section | Content |
|---------|---------|
| **Current Assets** | Cash, Bank, AR, Inventory, Prepaid, Other Current |
| **Non-Current Assets** | Fixed Assets (net of depreciation), Intangibles, Deferred Tax |
| **Current Liabilities** | AP, Accrued Expenses, Tax Payable, Current Portion of Loans |
| **Non-Current Liabilities** | Long-term Loans, Employee Benefits |
| **Equity** | Share Capital, Retained Earnings, Other Reserves |

#### Income Statement (Statement of Comprehensive Income)

| Section | Content |
|---------|---------|
| **Revenue** | Room Revenue, F&B Revenue, Other Revenue |
| **Cost of Sales** | Cost of Food, Cost of Beverage |
| **Gross Profit** | Revenue - Cost of Sales |
| **Operating Expenses** | By nature: Payroll, Depreciation, Utilities, etc. |
| **Operating Profit** | Gross Profit - Operating Expenses |
| **Finance Costs** | Interest expense |
| **Other Income/Expense** | FX gain/loss, asset disposal gain/loss |
| **Profit Before Tax** | Operating Profit + Other - Finance Costs |
| **Income Tax Expense** | Corporate income tax |
| **Net Profit** | Profit After Tax |

#### Cash Flow Statement

Prepared using the **indirect method** (per TAS 7):

| Section | Starting Point | Adjustments |
|---------|---------------|-------------|
| **Operating** | Net Profit | + Depreciation, ± Working Capital changes |
| **Investing** | — | - Asset purchases, + Asset disposals |
| **Financing** | — | + Borrowings, - Repayments, - Dividends |

### 2.3 General Accounting Reports

#### Trial Balance

| Report | Description |
|--------|-------------|
| **Summary TB** | One line per account code with opening, debit, credit, closing balances |
| **Detailed TB** | Same as summary but with cost center breakdown |
| **Adjusted TB** | Trial balance after adjusting entries (post month-end) |

#### General Ledger Detail

| Report | Description |
|--------|-------------|
| **GL Account Detail** | Transaction listing per account for a date range |
| **GL Summary** | Monthly totals per account |
| **Account Inquiry** | Interactive drill-down from balance → transactions → source documents |

#### Sub-Ledger Reports

| Report | Source Module | Description |
|--------|-------------|-------------|
| AP Aging | AP | Outstanding payables by aging bucket |
| AP Vendor Statement | AP | Transaction history per vendor |
| AR Aging | AR | Outstanding receivables by aging bucket |
| AR Customer Statement | AR | Transaction history per customer |
| FA Register | FA | Asset listing with depreciation schedule |
| FA Depreciation Summary | FA | Monthly depreciation by category/department |
| Bank Balance Summary | CB | Balance per bank account with GL reconciliation |

---

## 3. Report Parameters

### 3.1 Standard Parameters

All reports support the following common parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| Business Unit | Select | Single or multiple BU selection |
| Period | Date Range | From/To date or period selection |
| Currency | Select | Display in transaction currency or base currency |
| Comparison | Select | None, Budget, Prior Year, Prior Period |
| Department | Multi-Select | Filter by USALI department |
| Account Range | Range | From/To account code filter |
| Level of Detail | Select | Summary, Detail, with/without dimensions |
| Output Format | Select | Screen (HTML), PDF, Excel (.xlsx), CSV |

### 3.2 Drill-Down Capability

Reports support interactive drill-down through the data hierarchy:

```
Financial Statement Line (e.g., "Room Revenue: 2,500,000")
  └── Account Code Detail (e.g., 4010001, 4010002)
       └── Journal Entry List (JV2706-0001, ARIV2706-0015)
            └── Source Document (AR Invoice, PMS Posting)
```

---

## 4. Management Dashboard KPIs

### 4.1 Financial KPIs

| KPI | Formula | Target |
|-----|---------|--------|
| **RevPAR** | Room Revenue ÷ Available Rooms | > Budget |
| **GOP %** | GOP ÷ Total Revenue × 100 | > 35% |
| **EBITDA Margin** | EBITDA ÷ Total Revenue × 100 | > 28% |
| **Food Cost %** | Cost of Food ÷ Food Revenue × 100 | < 30% |
| **Beverage Cost %** | Cost of Beverage ÷ Bev Revenue × 100 | < 22% |
| **Payroll %** | Total Payroll ÷ Total Revenue × 100 | < 35% |
| **Utility Cost per Occupied Room** | Total Utilities ÷ Occupied Rooms | < Budget |

### 4.2 AP/AR KPIs

| KPI | Formula | Target |
|-----|---------|--------|
| **Days Payable Outstanding (DPO)** | AP Balance ÷ (COGS ÷ 365) | Within credit terms |
| **Days Sales Outstanding (DSO)** | AR Balance ÷ (Revenue ÷ 365) | < 45 days |
| **AP Aging > 90 Days** | Sum of AP > 90 days ÷ Total AP | < 5% |
| **AR Aging > 90 Days** | Sum of AR > 90 days ÷ Total AR | < 10% |
| **Payment On-Time Rate** | Payments within terms ÷ Total payments | > 95% |

### 4.3 Dashboard Widgets

| Widget | Visualization | Data Source |
|--------|--------------|-------------|
| Revenue Trend | Line chart (12 months rolling) | GL Revenue accounts |
| GOP Trend | Bar chart (12 months) | GL calculation |
| Budget vs Actual | Gauge chart | Budget module |
| Cash Position | Card with sparkline | Bank balances |
| AP Aging Summary | Stacked bar chart | AP sub-ledger |
| AR Aging Summary | Stacked bar chart | AR sub-ledger |
| Top 10 Vendors by Spend | Horizontal bar | AP transactions |
| Department P&L Overview | Heat map | GL by cost center |

---

## 5. Custom Report Builder

### 5.1 Report Template System

Integration with the existing **FastReport** infrastructure:

| Component | Description |
|-----------|-------------|
| `report_template` | .frx template definition (layout, bands, formulas) |
| `report_dataset` | SQL dataset definition executed by micro-data service |
| `report_parameter` | User-facing parameter definitions |
| `report_schedule` | Scheduled report generation (via micro-cronjobs) |

### 5.2 Template Management

- Templates are stored per BU with cluster-level sharing
- Support for custom report creation by power users
- Version control for template changes
- Preview mode before publishing

### 5.3 Scheduled Reports

Reports can be scheduled for automatic generation and distribution:

| Field | Description |
|-------|-------------|
| Frequency | Daily, Weekly, Monthly, On-demand |
| Recipients | Email addresses for distribution |
| Format | PDF, Excel |
| Parameters | Pre-configured parameter values |
| Delivery | Email attachment or MinIO file link |

---

## 6. Multi-Property Reporting

### 6.1 Report Scope Options

| Scope | Description |
|-------|-------------|
| **Single Property** | Report for one business unit |
| **Property Comparison** | Side-by-side comparison of selected properties |
| **Cluster Summary** | Aggregated totals across all properties in a cluster |
| **Consolidated** | Consolidated with IC elimination (via IC module) |

### 6.2 Currency in Multi-Property Reports

| Report Type | Currency Handling |
|-------------|------------------|
| Single Property | Property's base currency |
| Comparison | Each in own base currency, or translated to group currency |
| Cluster Summary | Translated to group reporting currency |
| Consolidated | Group reporting currency with translation adjustments |

---

## 7. Key Entity Summary

| Entity | Table Name | Description |
|--------|-----------|-------------|
| Report Definition | `rpt_report_definition` | Report metadata (name, category, type) |
| Report Template | `rpt_report_template` | FastReport .frx template reference |
| Report Dataset | `rpt_report_dataset` | SQL/stored procedure for data extraction |
| Report Parameter | `rpt_report_parameter` | User-facing input parameters |
| Report Schedule | `rpt_report_schedule` | Scheduled generation configuration |
| Report Output | `rpt_report_output` | Generated report file references |
| Dashboard Widget | `rpt_dashboard_widget` | Dashboard KPI/widget configuration |

---

## 8. Integration Points

| Direction | System/Module | Description |
|-----------|--------------|-------------|
| **Inbound** | GL | Trial balance and account detail data |
| **Inbound** | AP | AP aging, vendor spend data |
| **Inbound** | AR | AR aging, customer revenue data |
| **Inbound** | FA | Asset register and depreciation data |
| **Inbound** | Budget | Budget amounts for variance reports |
| **Inbound** | IC | Consolidated data with elimination entries |
| **Inbound** | Tax | Tax summary data |
| **Outbound** | micro-data (existing) | Dataset execution requests |
| **Outbound** | micro-report (existing) | Template rendering requests |
| **Outbound** | micro-cronjobs (existing) | Scheduled report generation |
| **Outbound** | MinIO (existing) | Report file storage |
| **Outbound** | SMTP (existing) | Email distribution |
