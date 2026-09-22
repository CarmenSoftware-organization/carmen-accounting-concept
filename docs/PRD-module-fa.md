# Carmen Cloud ERP — Fixed Assets (FA) Module

**Module Code:** FA  
**Version:** 1.0  
**Last Updated:** 2026-09-22  
**Parent Document:** [PRD-accounting-system-overview.md](./PRD-accounting-system-overview.md)  
**Standard Compliance:** USALI 12th Edition, TFRS/TAS 16 (Property, Plant & Equipment), Thai Revenue Department Depreciation Rules  

---

## 1. Module Overview

The Fixed Assets module manages the complete lifecycle of capital assets for hospitality businesses — from acquisition through depreciation to disposal. It supports the specific needs of hotels and resorts, which typically have significant investments in property, furniture, fixtures, and equipment (FF&E).

### 1.1 Key Capabilities

- Asset Register with full lifecycle tracking
- Depreciation calculation: **Straight-Line** (default) and **Declining Balance**
- Monthly batch depreciation run
- Asset disposal and write-off
- Asset transfer between departments and locations
- Asset revaluation
- Integration with AP for acquisition and with GL for auto-posting
- 7-dimension analysis for asset-related transactions

### 1.2 GL Posting Mode

**Auto-post** — depreciation entries and disposal entries are automatically posted to GL upon execution. This is appropriate because:
- Depreciation follows predefined mathematical formulas (no judgment required)
- Monthly depreciation runs are batch operations with consistent patterns
- Disposal entries follow standardized gain/loss calculations

---

## 2. Asset Lifecycle

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Fixed Asset Lifecycle                             │
│                                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │Acquisition│──▶│  Active  │──▶│Fully     │──▶│ Disposal │          │
│  │          │    │(Depre-   │    │Depreciated│   │ / Write- │          │
│  │- AP Inv  │    │ ciating) │    │(NBV = 0  │    │   off    │          │
│  │- Direct  │    │          │    │ or        │    │          │          │
│  │  Entry   │    │          │    │ salvage)  │    │          │          │
│  └──────────┘    └────┬─────┘    └──────────┘    └──────────┘          │
│                       │                                                  │
│                       │ During active life:                              │
│                       ├── Monthly Depreciation Run                       │
│                       ├── Asset Transfer (dept/location)                 │
│                       ├── Asset Revaluation                              │
│                       └── Component Addition                             │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Sub-Modules

### 3.1 Asset Register

The central record for all fixed assets owned by the business unit.

#### Key Fields

| Field | Type | Description |
|-------|------|-------------|
| `asset_no` | VARCHAR(20) | Auto-generated: FA-{Category}-{NNNNNN} |
| `asset_name` | VARCHAR(255) | Descriptive name |
| `asset_category_id` | UUID | FK to asset category master |
| `acquisition_date` | DATE | Date of purchase/capitalization |
| `acquisition_cost` | DECIMAL(16,2) | Original cost in base currency |
| `currency_code` | VARCHAR(3) | Currency of acquisition |
| `exchange_rate` | DECIMAL(12,5) | Rate at acquisition date |
| `salvage_value` | DECIMAL(16,2) | Estimated residual value |
| `useful_life_months` | INT | Estimated useful life |
| `depreciation_method` | ENUM | 'STRAIGHT_LINE', 'DECLINING_BALANCE' |
| `depreciation_start_date` | DATE | Date depreciation begins |
| `accumulated_depreciation` | DECIMAL(16,2) | Total depreciation to date |
| `net_book_value` | DECIMAL(16,2) | Acquisition cost - accumulated depreciation |
| `status` | ENUM | 'ACTIVE', 'FULLY_DEPRECIATED', 'DISPOSED', 'WRITTEN_OFF', 'TRANSFERRED' |
| `department_code` | VARCHAR(10) | Current owning department (USALI) |
| `location_code` | VARCHAR(10) | Physical location |
| `serial_no` | VARCHAR(50) | Manufacturer serial number |
| `warranty_end_date` | DATE | Warranty expiration |
| `source_doc_ref` | VARCHAR(50) | AP Invoice reference |

#### Asset Category Hierarchy

Typical hotel asset categories following USALI:

| Category Code | Category Name | Useful Life | Depr. Method | Thai Tax Life |
|--------------|---------------|-------------|--------------|---------------|
| LAND | Land (ที่ดิน) | N/A (no depreciation) | None | N/A |
| BLDG | Buildings (อาคาร) | 240–480 months | Straight-Line | 20 years |
| BLDG-IMP | Building Improvements | 60–120 months | Straight-Line | 5 years |
| FF&E | Furniture, Fixtures & Equipment | 60–120 months | Straight-Line | 5 years |
| VEHICLE | Vehicles (ยานพาหนะ) | 60 months | Declining Balance | 5 years |
| IT-EQUIP | IT Equipment (อุปกรณ์คอมพิวเตอร์) | 36–60 months | Straight-Line | 3 years |
| KITCHEN | Kitchen Equipment | 60–120 months | Straight-Line | 5 years |
| LINEN | Linen & Operating Equipment | 24–36 months | Straight-Line | 5 years |
| SOFTWARE | Software Licenses | 36–60 months | Straight-Line | 3 years |

### 3.2 Depreciation Calculation

#### Straight-Line Method (เส้นตรง)

$$
\text{Monthly Depreciation} = \frac{\text{Acquisition Cost} - \text{Salvage Value}}{\text{Useful Life (months)}}
$$

- Depreciation is constant each month
- Begins on `depreciation_start_date` (typically the month after acquisition)
- Ends when NBV reaches salvage value or asset is disposed

#### Declining Balance Method (ยอดลดลง)

$$
\text{Annual Rate} = \frac{2}{\text{Useful Life (years)}} \times 100\%
$$

$$
\text{Monthly Depreciation} = \frac{\text{Net Book Value} \times \text{Annual Rate}}{12}
$$

- Depreciation decreases over time as NBV reduces
- Switches to straight-line when straight-line produces a higher charge
- Thai Revenue Department maximum rate: 200% of straight-line rate (double declining balance)

#### Monthly Depreciation Run

The depreciation run is a **batch process** executed monthly:

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
│ Initiate │──▶│  Calculate   │──▶│  Generate    │──▶│  Auto-   │
│ Depr.    │    │  Depreciation│    │  JV Entries  │    │  Post    │
│ Run      │    │  per Asset   │    │  (grouped by │    │  to GL   │
│ (Month)  │    │              │    │   dept/cat)  │    │          │
└──────────┘    └──────────────┘    └──────────────┘    └──────────┘
```

**GL Entry Pattern (per department/category group):**

```
Dr.  Depreciation Expense (6xxx)          xxx.xx   (USALI dept)
  Cr.  Accumulated Depreciation (1xxx)    xxx.xx   (Asset category)
```

### 3.3 Asset Disposal

Recording the removal of an asset from the register.

#### Disposal Types

| Type | Description |
|------|-------------|
| Sale | Asset sold for a price — gain/loss recognized |
| Scrapped | Asset discarded — write off remaining NBV |
| Trade-in | Asset exchanged for a new asset |
| Donation | Asset donated — write off remaining NBV |
| Lost/Stolen | Insurance claim process |

#### Disposal Calculation

$$
\text{Gain/Loss on Disposal} = \text{Disposal Proceeds} - \text{Net Book Value at Disposal Date}
$$

**GL Entry Pattern:**

```
Dr.  Cash/Bank/AR (1xxx)                  xxx.xx   (Proceeds received)
Dr.  Accumulated Depreciation (1xxx)      xxx.xx   (Reverse accum. depr.)
Dr.  Loss on Disposal (8xxx)              xxx.xx   (If loss)
  Cr.  Fixed Asset Cost (1xxx)            xxx.xx   (Remove original cost)
  Cr.  Gain on Disposal (8xxx)            xxx.xx   (If gain)
```

### 3.4 Asset Transfer

Transfer an asset between departments, locations, or business units.

#### Transfer Types

| Type | GL Impact | Approval Required |
|------|-----------|-------------------|
| Department Transfer | Reclassify expense department | Department heads |
| Location Transfer | No GL impact (memo only) | Location manager |
| BU Transfer (Inter-company) | Full disposal + re-acquisition via IC module | FC Manager |

**Department Transfer GL Entry:**

```
Dr.  Asset Cost — New Department (1xxx)      xxx.xx
  Cr.  Asset Cost — Old Department (1xxx)    xxx.xx
Dr.  Accum. Depr. — Old Department (1xxx)    xxx.xx
  Cr.  Accum. Depr. — New Department (1xxx)  xxx.xx
```

---

## 4. Thai Tax Compliance

### 4.1 Tax Depreciation Rules

Thai Revenue Department prescribes maximum depreciation rates:

| Asset Type | Max Rate (Straight-Line) | Max Useful Life |
|-----------|-------------------------|-----------------|
| Buildings | 5% per year | 20 years |
| Building Improvements | 20% per year | 5 years |
| Vehicles | 20% per year | 5 years |
| Furniture & Equipment | 20% per year | 5 years |
| Computer Equipment | 33.33% per year | 3 years |
| Software | 33.33% per year | 3 years |

### 4.2 Book vs Tax Depreciation

The system maintains **dual depreciation schedules** when book useful life differs from tax useful life:
- **Book depreciation:** Used for TFRS financial statements
- **Tax depreciation:** Used for corporate income tax calculation
- Temporary differences are tracked for deferred tax purposes

---

## 5. Key Entity Summary

| Entity | Table Name | Description |
|--------|-----------|-------------|
| Asset Register | `fa_asset_register` | Master record for each fixed asset |
| Asset Category | `fa_asset_category` | Category setup with depreciation rules and GL mapping |
| Depreciation Schedule | `fa_depreciation_schedule` | Monthly depreciation records per asset |
| Depreciation Run | `fa_depreciation_run` | Batch run header (period, status, totals) |
| Asset Transaction | `fa_asset_transaction` | Acquisition, disposal, transfer, revaluation events |
| Asset Component | `fa_asset_component` | Sub-components of a parent asset |

---

## 6. Integration Points

| Direction | System/Module | Description |
|-----------|--------------|-------------|
| **Inbound** | AP | Asset acquisition from AP invoice (capitalization) |
| **Outbound** | GL | Auto-post depreciation, disposal, and transfer entries |
| **Outbound** | IC | Inter-company asset transfer entries |
| **Outbound** | Period End | Depreciation run triggered as part of month-end checklist |
| **Reference** | Asset Category Master | Depreciation method, useful life, GL account mapping |
| **Reference** | COA / Cost Center | Department and account validation |
