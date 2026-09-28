# Carmen Cloud ERP — Accounts Receivable (AR) Module

**Module Code:** AR  
**Version:** 1.1  
**Last Updated:** 2026-09-28  
**Parent Document:** [PRD-accounting-system-overview.md](./PRD-accounting-system-overview.md)  
**Source FRD:** `Accounting-docs/AR/Invoice/carmen_cloud_erp_ar_module_functional_requirement_document_frd.md` (FRD v1.07, 2026-09-27) + mockup `Carmen - AR - Invoice - Mockup V1.07.html`  
**FRD Review:** [reviews/2026-09-28-ar-frd-v1.07-review.md](./reviews/2026-09-28-ar-frd-v1.07-review.md)  
**Standard Compliance:** USALI 12th Edition, TFRS, Thai Revenue Department VAT & WHT Regulations  

> **Status:** Synced with FRD v1.07. Items marked **⚠ OI-n** are open issues that are
> contradictory or unresolved in the FRD and wait for a BA decision (see §9). They are
> not implementation-ready.

---

## 1. Module Overview

The Accounts Receivable (AR) module manages customer billing, advance deposits, debit/credit adjustments, and collections for hospitality businesses: City Ledger (corporate clients, travel agents, OTAs), credit-card receivables, and PMS guest folios transferred for billing.

### 1.1 Key Capabilities

- Five document types under one UI: AR Invoice (ARIV), Debit Note (ARDN), Credit Note (ARCN), Advance Deposit (ARDP), Receipt / Tax Invoice (ARRC)
- **Separate tax-invoice running number** (`tax_inv_no`) independent of `doc_no`, format `{Prefix}{YY}{MM}{4-digits}` per document type (TXIV / TXDN / TXCN / TXDP / TXRC — ⚠ OI-5)
- PMS Folio pull into ARIV with partial settlement (`0 < Settle Amount ≤ Folio Amount`)
- Advance-deposit application (Doc Reference tab) with realized FX handling (⚠ OI-1, OI-2)
- Level-of-Authority workflow, 3 steps: Preparer → Senior Accountant → Financial Controller (Approve / Send Back / Reject)
- Closed-accounting-period guard on `Input Date` (Save Draft and Submit)
- Multi-currency with dual-currency footer totals (transaction currency + base currency)
- Line-level revenue account, USALI department, 6 analysis dimensions (Market, Sales, Project, Event, Location, Channel), two tax profiles
- Header-level WHT amount recorded for reference only — recognized at receipt (⚠ OI-6)
- AR Aging by due date, with unpaid tracking at both header and line level
- PDPA: guest-name / card-number masking in UI, AES-256 at rest

### 1.2 GL Posting Mode

**⚠ OI-3 — unresolved.** Two positions exist:

| Source | Rule |
|--------|------|
| PRD v1.0 (this document, previous) | Manual review before GL posting — JV generated on submit, held in "Pending Review" until a senior accountant approves |
| FRD v1.07 §7 flow | JV posted to GL **on Submit**, before the LOA workflow runs |
| FRD v1.07 §3.1 / §3.2 statuses | `Draft → Submitted → Approved → Posted → Void`, and "Approve … or post to GL" — implies posting **after** final approval |

The FRD contradicts itself; the posting point must be decided before implementation.
Whatever is chosen, **PMS-Folio invoices never post to GL** (§3).

---

## 2. Sub-Modules & Document Types

### 2.1 Comparison (FRD §2)

| Attribute | ARIV | ARDN | ARCN | ARDP | ARRC |
|-----------|------|------|------|------|------|
| Purpose | Raise receivable (City Ledger) | Increase receivable | Decrease receivable / discount | Receive advance deposit | Record receipt / issue tax invoice on payment |
| Doc prefix | `ARIV` | `ARDN` | `ARCN` | `ARDP` | `ARRC` |
| Tax-invoice prefix | `TXIV` | `TXDN` | `TXCN` | `TXDP` | `TXRC` (⚠ OI-5) |
| Tab 1 Item Details | Full | Positive amounts | Negative amounts | Deposit lines | Settled items |
| Tab 2 Tax Invoice | Yes | Yes (add VAT) | Yes (reduce VAT) | Yes (deposit VAT) | Yes (tax invoice on receipt) |
| Tab 3 Doc Reference | **Enabled** | Disabled | Disabled | Disabled | Disabled |
| Tab 4 Receipt | Receipt history | Receipt history | Disabled | Shows applications | Cash/bank entry |
| Tab 5 Journal (GL) | Dr AR / Cr Revenue, Output VAT | Dr AR / Cr Revenue | Dr Revenue / Cr AR | Dr Cash / Cr Deposit liability (⚠ OI-2) | Dr Bank / Cr AR |
| Aging effect | + outstanding | + outstanding | − outstanding | − outstanding when applied | Clears outstanding |

### 2.2 AR Invoice (ARIV)

- **Document number:** `ARIV{YY}{MM}{4-digits}`, e.g. `ARIV26090001`, read-only
- **Sources:** `Manual`, `PMS Folio`, `Copy` (FRD §3.1); FRD §4 also stamps `AI` from AI document extraction (not in the `doc_source` enum — see review)
- **Header (2 rows):** Doc No, Input Date, Customer, Currency, Exch Rate, Status, Source, Source Doc / Tax-Invoice checkbox + Tax Inv No, Credit days, Due Date, Description, WHT checkbox + amount, LOA panel
- **Due date:** `Input Date + Credit Term Days` (credit term defaults from customer profile)
- **Tax invoice number:** generated on **Submit** when `is_tax_inv = true`; never on Save Draft. Concurrency conflict → rollback and retry next sequence (ERR_AR_006)

#### Line Item (Edit Item Detail modal — 5 sections)

| Section | Fields |
|---------|--------|
| 0 Booking & Reference | `group_no` (summary-invoice grouping), 2-line `reference_info` (folio / booking / guest), `date_from` / `date_to` (stay dates) |
| 1 Discount | Subtotal (= Qty × Price), Discount %, Discount Amount with Override (discount ≤ subtotal, % ≤ 100 — ERR_AR_005) |
| 2 Revenue | Cr revenue account, USALI department, Net (= Subtotal − Discount) |
| 3 Output Tax | Tax 1 profile (`VAT07_ADD`, `VAT07_INC`, `NONE`), account, dept, amount + Override; Tax 2 profile (supplementary tax or WHT — ⚠ OI-6), account, dept, amount + Override |
| 4 Accounts Receivable | Dr AR account, Dr dept, Total (= Net + Tax 1 + Tax 2) |

Line grid shows `Total` and `Unpaid` per line; a line fully received shows unpaid `0.00`.

#### Doc Reference tab — Apply Deposit

- Only on ARIV. Picks ARDP documents with the **same customer and same currency** (search is filter-locked — ERR_AR_004), status `Submitted` or `Posted`
- Rows show ARDP no, date, applied net, applied tax, total offset (negative) and realized FX
- FRD §5.3 also allows `ref_doc_type` ARCN / ARDN, but the UI only offers Apply Deposit (⚠ OI-5)

#### Receipt tab

- **Get Receipt** copies the current invoice's unpaid amount into a new ARRC draft (invoice must be `Submitted` or `Posted`)
- History: ARRC no, date, pay type, paid, WHT, net received, status

### 2.3 AR Debit Note (ARDN)

- Increases the receivable; positive lines; Doc Reference disabled; tax invoice `TXDN`
- **GL:** Dr Accounts Receivable / Cr Revenue (+ Output VAT)

### 2.4 AR Credit Note (ARCN)

- Decreases the receivable; lines entered as negative amounts; Receipt tab disabled; tax invoice `TXCN`
- **GL:** Dr Revenue (+ Output VAT) / Cr Accounts Receivable

### 2.5 AR Advance Deposit (ARDP) — *new in v1.1*

- Deposit received from a customer / agent before stay; tax invoice `TXDP`
- Reduces outstanding only when applied to an ARIV (Doc Reference)
- **GL:** Dr Cash/Bank / Cr Advance Deposit liability. The VAT line on a deposit is **not specified** in the FRD even though a TXDP tax invoice is issued (⚠ OI-2)

### 2.6 AR Receipt / Tax Invoice (ARRC)

- Records customer payment against outstanding documents; issues tax invoice `TXRC` where the tax point is the receipt (services)
- WHT deducted by the customer is recognized here (not at invoice)
- **GL:** Dr Bank/Cash (+ Dr WHT receivable) / Cr Accounts Receivable; realized FX gain/loss when receipt rate ≠ invoice rate
- The FRD covers ARRC only from the invoice side (Get Receipt); a dedicated Receipt FRD is still needed

---

## 3. PMS Folio Integration

### 3.1 Model (FRD v1.07)

Revenue and City Ledger / credit-card receivables are already journalized in GL by the **PMS Night Audit Interface JV**. The AR module pulls folios **only to bill, print invoices, and track aging**.

```
PMS (Opera, Cloudbeds, …)
   │  Night Audit Interface ──────────▶ GL  (Interface JV: Dr City Ledger / Cr Revenue, VAT)
   │
   └─ Unbilled folios ──▶ [Add Folio modal] ──▶ ARIV (source = PMS Folio, is_pms_folio = true)
                                                  │
                                                  └─▶ AR Aging only — NO GL post
```

### 3.2 Add Folio Modal Rules

1. Customer is locked to the invoice's AR code; `Folio Date To` cannot exceed `Input Date`
2. Grid shows folio date, folio no, masked guest name (e.g. `Mr. J*** D**`), folio amount, settle amount
3. **Settle guard:** `0 < Settle Amount ≤ Folio Amount`; over-entry resets to the folio amount (ERR_AR_002). Partial billing is allowed
4. Imported lines are stamped `is_pms_folio = true`
5. Journal tab of a PMS-Folio invoice shows "PMS Folio: No Direct GL Post"; no JV is created
6. SLA: ≤ 500 unbilled folios load in < 3 s, paginated / lazy-loaded

### 3.3 Dependencies (not yet built)

- A PMS interface that produces the Night Audit JV and exposes unbilled folios with remaining balances
- Folio balance tracking so a folio partially settled on one invoice shows only the remainder next time

> The v1.0 model (interface batch → validation queue → auto-created ARIV with source
> "Interface Posting") is **superseded** by §3.1. PMS revenue-code → COA mapping now
> belongs to the Night Audit interface, not to AR.

---

## 4. GL Posting Logic

### 4.1 Manual vs PMS-Folio Invoice

| Invoice type | GL |
|--------------|----|
| Manual (source `Manual` / `Copy`) | JV (prefix AR): Dr AR / Cr Revenue by USALI dept / Cr Output VAT Pending |
| PMS Folio | No GL post (already booked by Night Audit Interface JV) |

A single invoice mixing manual lines and folio lines is not addressed by the FRD (see review).

### 4.2 Non-Compound Entry for Deposit Application

When ARDP deposits are applied, one JV holds two independent blocks for audit trail:

1. **Full invoice recognition** — Dr AR (full) / Cr Revenue (full) / Cr Output VAT Pending (full)
2. **Deposit offset** — Dr Advance Deposit liability (at deposit rate) / Cr AR (at invoice rate) / FX difference

### 4.3 Worked Example (corrected — ⚠ OI-1)

Invoice `ARIV26090001`: 100.00 USD − 10.00 discount = 90.00 net + 6.30 VAT = **96.30 USD** @ 35.00000.
Applied deposit `ARDP26090005`: **21.40 USD** received @ 34.00000.

| # | Account | Dr THB | Cr THB | FC (USD) |
|---|---------|-------:|-------:|---------:|
| 1 | 1130000 AR – City Ledger | 3,370.50 | | 96.30 |
| 2 | 4110000 Room Sales Revenue (dept 101) | | 3,150.00 | 90.00 |
| 3 | 2151000 Output VAT Pending | | 220.50 | 6.30 |
| 4 | 2130000 Advance Guest Deposit | 727.60 | | 21.40 |
| 5 | 1130000 AR – City Ledger | | 749.00 | 21.40 |
| 6 | Realized FX **Loss** | 21.40 | | 0.00 |
| | **Total** | **4,119.50** | **4,119.50** | |

The FRD example posts line 6 as a *Cr FX Gain* and states totals of 4,098.10 = 4,098.10; the
credit side actually sums to 4,140.90, so the FRD JV is unbalanced by 42.80 (= 2 × 21.40).
With this JV shape (deposit cleared at its original rate, AR cleared at the invoice rate), a rise
in the rate produces a **loss**:

$$
\text{FX difference} = \text{Applied FC} \times (\text{Deposit rate} - \text{Invoice rate}) \quad
\begin{cases} < 0 & \Rightarrow \text{Dr Realized FX Loss} \\ > 0 & \Rightarrow \text{Cr Realized FX Gain} \end{cases}
$$

Remaining AR after application: 3,370.50 − 749.00 = 2,621.50 THB = 74.90 USD × 35.

> **⚠ OI-2 (TFRS):** Under TFRIC 22, an advance deposit received is a non-monetary liability; the
> portion of revenue it covers is recognized at the **deposit-date rate**, and no FX gain/loss
> arises on application. The FRD model (and the corrected example above) recognizes the full
> invoice at the invoice rate and books an FX difference instead. Also, if ARDP already issued a
> TXDP tax invoice (VAT at deposit), crediting the full 6.30 USD VAT again at invoice would
> double-count output VAT on the deposited portion. Needs accounting sign-off.

### 4.4 WHT

Header `wht_amount` is recorded for reference and has **no GL effect at invoice**; WHT is recognized at ARRC. Line-level Tax 2 may be configured as WHT with a posting account (e.g. `2180000`), which would post at invoice (⚠ OI-6).

---

## 5. AR Aging Analysis

### 5.1 Aging Buckets

| Bucket | Range | Description |
|--------|-------|-------------|
| Current | 0 days | Not yet due |
| 1-30 | 1–30 days past due | First follow-up |
| 31-60 | 31–60 days past due | Second follow-up |
| 61-90 | 61–90 days past due | Escalation |
| 91-120 | 91–120 days past due | Final notice |
| 120+ | > 120 days past due | Write-off candidate |

### 5.2 Aging Calculation

$$
\text{Days Outstanding} = \text{Report Date} - \text{Due Date}
$$

- Aging is based on **Due Date** (`Input Date + Credit Term Days`)
- Includes both manual and PMS-Folio invoices (aging is the main purpose of folio invoices)
- Partially paid invoices show only the remaining `unpaid_amount`; unpaid is also tracked per line
- ARCN reduces outstanding; ARDP reduces outstanding only once applied

### 5.3 Aging Reports

- **Summary by Customer:** outstanding per customer by bucket
- **Detail by Invoice:** invoice listing with bucket
- **Department Summary:** outstanding by USALI department

---

## 6. Key Entity Summary

| Entity | Table (FRD) | Description |
|--------|-------------|-------------|
| AR document header | `ar_invoice_header` | ARIV / ARDN / ARCN / ARDP header: status, source, tax-invoice flag + number, credit term, due date, WHT (reference), totals, unpaid |
| AR document detail | `ar_invoice_detail` | Lines: group no, reference, stay dates, qty/price, discount, revenue account/dept, Tax 1/2 (profile, account, dept, amount, override), AR account/dept, 6 dimensions, `is_pms_folio`, unpaid |
| Doc reference | `ar_document_reference` | Applied deposits on an ARIV: ref doc, original rate, applied net/tax/total, realized FX |
| Output tax register | `ar_tax_invoice` | Tax invoice no/date, tax period, status (`Pending`/`Confirm`/`Submitted`), customer registered name, 13-digit tax ID, 5-digit branch, split address, THB base/tax/total |
| Customer profile | `ar_profile` | Customer master: AR code, default currency, credit term, tax entity — **referenced but not specified by the FRD** |
| Receipt | *(to be specified)* | FRD puts `ARRC` in the `ar_invoice_header.doc_prefix` enum. Recommendation: separate `ar_receipt` + allocation tables, mirroring `ap_invoice` / `ap_payment` (review item 5) |
| Audit log | `carmen_audit_logs` | Immutable log of create/edit/override/approve/reject/void with IP and user |

Implementation note: the FRD models `BIGINT` keys and hard-coded defaults (`currency_code` = `USD`, `dr_account_code` = `1130000`, `dr_dept_code` = `GEN`). In the backend these follow micro-business conventions — UUID keys, BU base currency, and account defaults from `gl_setting` as done for AP.

---

## 7. Actions & Error Handling

### 7.1 Actions (FRD §4)

| Action | Rule |
|--------|------|
| AI | Extract a PDF/image folio into the form (< 5 s); stamps source `AI` |
| Add | New draft, next doc no |
| Save Draft | Requires customer + input date; no GL, no tax-invoice number |
| Submit | Period open → lines complete → generate tax-invoice no if flagged → `Submitted`, enter LOA step 1 |
| Void | Blocked if any receipt/settlement exists (ERR_AR_003); otherwise `Void` and releases applied deposits / folio balances |
| Copy | New draft copying lines, accounts, departments |
| Print | AR invoice report |
| Add Folio | Requires customer; opens folio modal |
| Get Receipt | Creates ARRC draft from unpaid (`Submitted`/`Posted` only) |
| LOA Reject | Sets document to `Void` |

### 7.2 Error Codes

| Code | Condition | Behavior |
|------|-----------|----------|
| ERR_AR_001 | Input date in a closed accounting period | Block Save Draft and Submit |
| ERR_AR_002 | Settle amount > folio amount | Reset to folio amount + warning |
| ERR_AR_003 | Void on a document with receipts / settlements | Void disabled; API rejects |
| ERR_AR_004 | Deposit of another customer or currency | Search locked to same AR code + currency |
| ERR_AR_005 | Discount > 100 % or > subtotal | Clamp + highlight |
| ERR_AR_006 | Tax-invoice number collision | Rollback, take next sequence |

### 7.3 Roles

| Role | Rights |
|------|--------|
| AR Officer | Create, edit, save draft, pull folios |
| Senior Accountant | LOA step 2 (Approve / Send Back), edit tax-invoice entity |
| Financial Controller | Final approval, Void |

---

## 8. Integration Points

| Direction | System/Module | Description |
|-----------|--------------|-------------|
| **Inbound** | PMS (external) | Unbilled folios for billing; revenue already posted by Night Audit Interface JV |
| **Outbound** | GL | Manual-invoice JVs and deposit offsets (posting point ⚠ OI-3); none for PMS-Folio invoices |
| **Outbound** | Cash & Bank | ARDP and ARRC cash/bank entries |
| **Outbound** | Tax Management | Output tax register (`ar_tax_invoice`), WHT at receipt |
| **Reference** | Customer profile | Default currency, credit term, tax entity |
| **Reference** | COA / Cost Center / Dimensions / Tax profile | Account validation, USALI departments, 6 dimensions |
| **Reference** | GL Period | Closed-period guard |

---

## 9. Open Issues (pending BA decision)

Details, evidence and proposals: [reviews/2026-09-28-ar-frd-v1.07-review.md](./reviews/2026-09-28-ar-frd-v1.07-review.md).

| ID | Issue | FRD ref |
|----|-------|---------|
| OI-1 | Worked JV unbalanced (Cr 4,140.90 vs Dr 4,098.10); FX sign inverted | §6.3, §6.4, TC-AR-09 |
| OI-2 | Deposit FX vs TFRIC 22; VAT on deposit (TXDP) possibly double-counted at invoice | §6.2–6.4, §2 |
| OI-3 | GL posting point: on Submit vs after final approval | §3.2, §4, §7 |
| OI-4 | Receipt (ARRC) stored in invoice header table; no receipt spec | §5.1 |
| OI-5 | Inconsistent lists: TXRC prefix, `ref_doc_type` ARCN/ARDN vs Apply-Deposit only, `AI` source | §1.2, §2, §3.6, §4, §5 |
| OI-6 | WHT: header "no GL" vs line Tax 2 with WHT account | §3.1, §3.4 |
| OI-7 | Backend prerequisites missing: customer master, PMS interface | §3.1, §3.10, §6.1 |
