# Healthcare Financial & Clinical Operations Dashboard

> An end-to-end analytics project built on **Synthea** synthetic patient data — covering Python data cleaning, Power Query M transformations, a star schema semantic model, 50+ DAX measures, and a 5-page interactive Power BI dashboard.

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=flat&logo=powerbi&logoColor=black)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-150458?style=flat&logo=pandas&logoColor=white)
![Synthea](https://img.shields.io/badge/Data-Synthea-1D9E75?style=flat)

---

## Table of Contents

- [Project Overview](#project-overview)
- [Dataset](#dataset)
- [Architecture](#architecture)
- [Python Data Cleaning](#python-data-cleaning)
- [Power Query M Transformations](#power-query-m-transformations)
- [Semantic Model](#semantic-model)
- [DAX Measures](#dax-measures)
- [Dashboard Pages](#dashboard-pages)
- [Key Insights](#key-insights)
- [How to Run](#how-to-run)
- [Tools & Technologies](#tools--technologies)

---

## Project Overview

This project simulates a real-world healthcare analytics pipeline — from raw synthetic EHR data to a fully interactive executive dashboard. It is designed to demonstrate proficiency across the full modern BI stack: data engineering, dimensional modeling, semantic layer design, and dashboard storytelling.

The dashboard covers four analytical domains:

| Domain | Focus |
|---|---|
| Financial | Claim costs, payer coverage, patient OOP, medication & procedure spend |
| Clinical | Condition prevalence, chronic vs acute, immunizations, rapid returns |
| Provider & Org | Utilization distribution, revenue performance, specialty breakdown |
| Payer | Coverage rates, QOLS, revenue per member, transition analysis |

---

## Dataset

**Source:** [Synthea](https://synthetichealth.github.io/synthea/) — open-source synthetic patient generator by MITRE Corporation. Generates realistic, fully de-identified patient data based on real epidemiological models. No real patient data is used anywhere in this project.

**Geography:** Massachusetts, United States

**Scale:**

| File | Rows | Description |
|---|---|---|
| patients.csv | 1,171 | Patient demographics, expenses, coverage |
| encounters.csv | 53,346 | All clinical visits |
| conditions.csv | 8,376 | Patient diagnoses |
| medications.csv | 42,989 | Prescription dispenses |
| procedures.csv | 34,981 | Clinical procedures |
| payers.csv | 10 | Insurance payer details |
| providers.csv | 5,855 | Healthcare providers |
| organizations.csv | 1,119 | Healthcare organizations |
| immunizations.csv | 15,478 | Vaccination records |
| allergies.csv | 597 | Patient allergies |
| careplans.csv | 3,483 | Care plan records |
| imaging_studies.csv | 855 | Radiology/imaging |
| payer_transitions.csv | 3,801 | Insurance coverage history |
| devices.csv | 78 | Medical devices |
| supplies.csv | 0 | Supplies (empty in this export) |

**Date range:** 1912-07-21 to 2020-04-28

**Financial scale:**
- Total claim cost (encounters): $6.87M
- Total medication cost: $100.1M
- Total procedure cost: $172.5M
- Total patient healthcare expenses: $895.7M

---

## Architecture

```
Synthea CSVs (16 files)
        │
        ▼
Python Data Cleaning (Untitled-1.ipynb)
  - Null handling
  - ZIP code enrichment
  - Name parsing
  - Date parsing
  - Derived columns (AGE, Age Group, patient_oop, DURATION)
  - Procedure reason categorization
  - Export to /cleaned/*.csv
        │
        ▼
Power Query M (Power BI Desktop)
  - Load cleaned CSVs
  - Type enforcement
  - Star schema shaping
  - Date dimension generation
  - Derived columns (care_setting, coverage_ratio, is_chronic)
  - Encounter type lookup table
        │
        ▼
Star Schema Semantic Model (12 tables)
        │
        ▼
DAX Measures (50+ measures)
        │
        ▼
5-Page Interactive Dashboard
```

---

## Python Data Cleaning

**File:** `Untitled-1.ipynb`

All raw Synthea CSVs are loaded, inspected, cleaned, and exported to a `/cleaned/` folder before being loaded into Power BI. The notebook uses `pandas`, `numpy`, and `datetime`.

### patients.csv

**Issues found:**
- Missing ZIP codes for many Massachusetts cities
- Synthea generates numeric suffixes in patient names (e.g. `Garth972`)
- DEATHDATE is null for living patients — needs handling before age calculation

**Transformations applied:**

```python
# 1. ZIP code enrichment — manual USPS lookup dictionary
usps_zip_lookup = {
    "Pembroke": "02359",
    "Boston":   "02108",
    "Worcester":"01608",
    # ... 200+ Massachusetts cities
}
patients['ZIP'] = patients.apply(
    lambda row: usps_zip_lookup.get(row['CITY'], row['ZIP']),
    axis=1
)

# Second pass for cities missed in first lookup
patients['ZIP'] = patients.apply(
    lambda row: usps_zip_lookup_missing.get(row['CITY'], row['ZIP']),
    axis=1
)

# 2. Name cleaning — strip numeric suffixes from Synthea-generated names
patients['FIRST_CLEAN'] = patients['FIRST'].str.replace(r'\d+', '', regex=True)
patients['LAST_CLEAN']  = patients['LAST'].str.replace(r'\d+', '', regex=True)
patients['FULL_NAME']   = patients['FIRST_CLEAN'] + ' ' + patients['LAST_CLEAN']

# 3. Date parsing
patients['DEATHDATE'] = pd.to_datetime(patients['DEATHDATE'], errors='coerce')
patients['BIRTHDATE'] = pd.to_datetime(patients['BIRTHDATE'], errors='coerce')

# 4. Age calculation — uses DEATHDATE for deceased patients, today for living
patients['REFERENCE_DATE'] = patients['DEATHDATE'].fillna(pd.Timestamp.today())
patients['AGE'] = (patients['REFERENCE_DATE'] - patients['BIRTHDATE']).dt.days // 365

# 5. Age group bucketing
patients['Age Group'] = pd.cut(
    patients['AGE'],
    bins=[0, 18, 35, 50, 65, 80, 100],
    labels=['0-18', '19-35', '36-50', '51-65', '66-80', '81+'],
    right=False
)
```

**Why REFERENCE_DATE matters:** Using `pd.Timestamp.today()` for all patients would give deceased patients an inflated age. Using `DEATHDATE.fillna(today)` ensures age is calculated at time of death for deceased patients and at today for living patients.

---

### providers.csv

**Issues found:**
- Synthea generates names with numeric suffixes (e.g. `Gaynell126 Streich926`)
- Names include special characters (accented letters: á, é, í, ñ)
- Simple regex fails on names with accented characters

**Transformations applied:**

```python
# Attempt 1 — basic regex (fails on accented names)
providers[['FIRST_NAME', 'LAST_NAME']] = providers['NAME'].str.extract(
    r'([A-Za-z]+)\d*\s+([A-Za-z]+)\d*'
)

# Attempt 2 — extended regex covering accented characters
providers['CLEAN_FIRST'] = providers['NAME'].str.extract(
    r'([A-Za-zÁÉÍÓÚáéíóúñÑ]+)'
)
providers['CLEAN_LAST'] = providers['NAME'].str.extract(
    r'\s([A-Za-zÁÉÍÓÚáéíóúñÑ]+)'
)

# Final approach — strip all digits then split
providers['CLEAN_NAME'] = providers['NAME'].str.replace(r'\d+', '', regex=True)
providers[['FIRST_CLEAN', 'LAST_CLEAN']] = providers['CLEAN_NAME'].str.split(
    ' ', n=1, expand=True
)
providers['FULL NAME'] = providers['FIRST_CLEAN'] + ' ' + providers['LAST_CLEAN']
```

**Why three attempts:** The first regex missed accented characters. The second covered accents but split incorrectly on some names. The final approach of stripping digits first then splitting on whitespace handled all edge cases cleanly.

---

### procedures.csv

**Issues found:**
- `REASONDESCRIPTION` contains raw clinical text — not grouped for analysis

**Transformations applied:**

```python
# Reason category mapping — clinical text → grouped category
reason_category_map = {
    'Normal pregnancy':                    'Pregnancy',
    'Tubal pregnancy':                     'Pregnancy',
    'Atrial Fibrillation':                 'Cardiology',
    'Myocardial Infarction':               'Cardiology',
    'Malignant neoplasm of breast':        'Oncology',
    'Acute bronchitis (disorder)':         'Respiratory',
    'Fracture of forearm':                 'Musculoskeletal',
    'Laceration of forearm':               'Trauma',
    'Appendicitis':                        'Gastrointestinal',
    'Major depression  single episode':    'Mental Health',
    # ... 40+ mappings across 8 categories
}

procedures['REASON_CATEGORY'] = procedures['REASONDESCRIPTION'].map(reason_category_map)
```

**Categories created:** Pregnancy, Cardiology, Oncology, Respiratory, Musculoskeletal, Trauma, Gastrointestinal, Mental Health

---

### allergies.csv

```python
allergies['START'] = pd.to_datetime(allergies['START'])
allergies['STOP']  = pd.to_datetime(allergies['STOP'])
# Records with null STOP = active allergies (retained as-is)
```

---

### encounters.csv

```python
encounter['START'] = pd.to_datetime(encounter['START'], errors='coerce')
encounter['STOP']  = pd.to_datetime(encounter['STOP'],  errors='coerce')

# Duration of each encounter
encounter['DURATION'] = encounter['STOP'] - encounter['START']

# Patient out-of-pocket — null-safe using numpy.where
encounter['patient_oop'] = np.where(
    encounter['TOTAL_CLAIM_COST'].isna() | encounter['PAYER_COVERAGE'].isna(),
    np.nan,
    np.maximum(encounter['TOTAL_CLAIM_COST'] - encounter['PAYER_COVERAGE'], 0)
)
```

---

### conditions.csv

```python
conditions['START'] = pd.to_datetime(conditions['START'], errors='coerce')
conditions['STOP']  = pd.to_datetime(conditions['STOP'],  errors='coerce')
# Null STOP = chronic/active condition — no imputation applied
```

---

### payers.csv

```python
# Null check on CITY for payer records
payers.loc[payers['CITY'].isna(), 'NAME']
# Exported as-is after inspection — no critical nulls found
```

**Output:** All cleaned files exported to `/cleaned/` folder as `*_clean.csv` — these are the source files loaded into Power BI.

---

## Power Query M Transformations

**File:** `PowerQuery_M_AllTables.txt`

All cleaned CSVs are loaded into Power BI via Power Query. Each query follows this pattern:

1. `Csv.Document()` — load with explicit column count and UTF-8 encoding
2. `Table.PromoteHeaders()` — lift first row as headers
3. `Table.SelectColumns()` — keep only required columns
4. `Table.RenameColumns()` — apply snake_case naming
5. `Table.TransformColumnTypes()` — enforce correct data types
6. `Table.AddColumn()` — derived column additions

### Key M transformations

**dim_date — generated entirely in M:**
```
let
    StartDate = #date(2010, 1, 1),
    EndDate   = #date(2030, 12, 31),
    DayCount  = Duration.Days(EndDate - StartDate) + 1,
    DateList  = List.Dates(StartDate, DayCount, #duration(1,0,0,0)),
    DateTable = Table.FromList(DateList, Splitter.SplitByNothing(), {"full_date"}),
    AddDateId = Table.AddColumn(DateTable, "date_id",
                  each Date.Year([full_date]) * 10000
                     + Date.Month([full_date]) * 100
                     + Date.Day([full_date]), Int64.Type),
    AddYear      = Table.AddColumn(AddDateId,    "year",         each Date.Year([full_date]),          Int64.Type),
    AddQuarter   = Table.AddColumn(AddYear,      "quarter",      each Date.QuarterOfYear([full_date]), Int64.Type),
    AddMonth     = Table.AddColumn(AddQuarter,   "month",        each Date.Month([full_date]),         Int64.Type),
    AddMonthName = Table.AddColumn(AddMonth,     "month_name",   each Date.MonthName([full_date]),     type text),
    AddWeek      = Table.AddColumn(AddMonthName, "week_of_year", each Date.WeekOfYear([full_date]),    Int64.Type),
    AddDayName   = Table.AddColumn(AddWeek,      "day_name",     each Date.DayOfWeekName([full_date]), type text),
    AddIsWeekend = Table.AddColumn(AddDayName,   "is_weekend",
                    each if Date.DayOfWeek([full_date]) >= 5 then 1 else 0, Int64.Type),
    AddQtrLabel  = Table.AddColumn(AddIsWeekend, "quarter_label",
                    each "Q" & Text.From(Date.QuarterOfYear([full_date]))
                       & " " & Text.From(Date.Year([full_date])), type text)
in
    AddQtrLabel
```

**dim_encounter_type — deduplication + care setting lookup:**
```
Deduplicate    = Table.Distinct(RenameColumns),
AddIndex       = Table.AddIndexColumn(Deduplicate, "encounter_type_id", 1, 1, Int64.Type),
AddCareSetting = Table.AddColumn(AddIndex, "care_setting",
    each if [encounter_class] = "wellness"   then "Preventive"
         else if [encounter_class] = "ambulatory" then "Outpatient"
         else if [encounter_class] = "outpatient" then "Outpatient"
         else if [encounter_class] = "urgentcare" then "Urgent Care"
         else if [encounter_class] = "emergency"  then "Emergency"
         else if [encounter_class] = "inpatient"  then "Inpatient"
         else "Other", type text)
```

**fact_encounters — patient OOP + encounter_type_id join:**
```
AddOOP = Table.AddColumn(AddDuration, "patient_oop",
    each if [TOTAL_CLAIM_COST] = null or [PAYER_COVERAGE] = null then null
         else Number.Max([TOTAL_CLAIM_COST] - [PAYER_COVERAGE], 0),
    type number),

MergeType  = Table.NestedJoin(
    AddOOP, {"ENCOUNTERCLASS","CODE"},
    dim_encounter_type, {"encounter_class","code"},
    "EncType", JoinKind.Left
),
ExpandType = Table.ExpandTableColumn(MergeType, "EncType",
    {"encounter_type_id"}, {"encounter_type_id"})
```

**dim_payer — coverage ratio:**
```
AddCoverageRatio = Table.AddColumn(ChangeTypes, "coverage_ratio",
    each if ([amount_covered] + [amount_uncovered]) = 0 then 0
         else Number.Round(
             [amount_covered] / ([amount_covered] + [amount_uncovered]), 4
         ), type number)
```

---

## Semantic Model

**Schema type:** Star schema

### Tables

| Table | Type | Rows | Grain |
|---|---|---|---|
| fact_encounters | Fact | 53,346 | One row per clinical encounter |
| fact_conditions | Bridge | 8,376 | One row per patient diagnosis |
| fact_medications | Bridge | 42,989 | One row per prescription dispense |
| fact_procedures | Bridge | 34,981 | One row per procedure performed |
| fact_immunizations | Bridge | 15,478 | One row per immunization |
| fact_allergies | Bridge | 597 | One row per allergy record |
| dim_patient | Dimension | 1,171 | One row per patient |
| dim_payer | Dimension | 10 | One row per insurance payer |
| dim_provider | Dimension | 5,855 | One row per provider |
| dim_organization | Dimension | 1,119 | One row per organization |
| dim_date | Dimension | 7,426 | One row per calendar date |
| dim_encounter_type | Dimension | 58 | One row per encounter class/code combination |
| payer_transitions | Reference | 3,801 | One row per patient payer coverage period |

### Relationships

| From (Many) | Key | To (One) | Key |
|---|---|---|---|
| fact_encounters | patient_id | dim_patient | patient_id |
| fact_encounters | payer_id | dim_payer | payer_id |
| fact_encounters | provider_id | dim_provider | provider_id |
| fact_encounters | organization_id | dim_organization | organization_id |
| fact_encounters | date_id | dim_date | date_id |
| fact_encounters | encounter_type_id | dim_encounter_type | encounter_type_id |
| fact_conditions | patient_id | dim_patient | patient_id |
| fact_medications | patient_id | dim_patient | patient_id |
| fact_medications | payer_id | dim_payer | payer_id |
| fact_procedures | patient_id | dim_patient | patient_id |
| fact_immunizations | patient_id | dim_patient | patient_id |
| fact_allergies | patient_id | dim_patient | patient_id |
| payer_transitions | patient_id | dim_patient | patient_id |
| payer_transitions | payer_id | dim_payer | payer_id |
| dim_provider | organization_id | dim_organization | organization_id |

All relationships: Many-to-One (*:1), single-direction filter (dimension → fact).

### Design decisions

**Star schema over galaxy schema** — early prototype used 9 directly connected fact tables. Simplified to star schema centered on `fact_encounters` with bridge tables for clinical entities, reducing cross-filtering ambiguity.

**Integer date key** — `date_id` stored as YYYYMMDD integer for join performance. `dim_date[full_date]` holds the actual date column for time intelligence.

**Measure table** — all DAX measures stored in a dedicated blank `_Measures` table to keep the field list clean.

**Calculated column for days since last visit** — computed in DAX not M because it requires row context across the same table (self-referencing filter per patient):

```dax
Days Since Last Visit =
VAR _patient  = FACT_ENCOUNTER[patient_id]
VAR _currdate = FACT_ENCOUNTER[start_date]
VAR _prevdate =
    MAXX(
        FILTER(
            FACT_ENCOUNTER,
            FACT_ENCOUNTER[patient_id] = _patient
            && FACT_ENCOUNTER[start_date] < _currdate
        ),
        FACT_ENCOUNTER[start_date]
    )
RETURN IF(ISBLANK(_prevdate), BLANK(), DATEDIFF(_prevdate, _currdate, DAY))
```

---

## DAX Measures

All measures organized into 11 categories in `DAX_Measures_Healthcare.txt`.

### Financial — Encounters

```dax
Total Claim Cost =
SUM(FACT_ENCOUNTER[total_claim_cost])

Total Payer Coverage =
SUM(FACT_ENCOUNTER[payer_coverage])

Total Patient OOP =
SUM(FACT_ENCOUNTER[patient_oop])

Coverage Rate % =
DIVIDE(
    SUM(FACT_ENCOUNTER[payer_coverage]),
    SUM(FACT_ENCOUNTER[total_claim_cost]),
    0
)

Avg Claim Cost per Encounter =
AVERAGE(FACT_ENCOUNTER[total_claim_cost])
```

### Financial — Medications

```dax
Total Medication Cost =
SUM(FACT_MEDICATIONS[total_cost])

Medication Coverage Rate % =
DIVIDE(
    SUM(FACT_MEDICATIONS[payer_coverage]),
    SUM(FACT_MEDICATIONS[total_cost]),
    0
)

Total Medication Patient OOP =
SUM(FACT_MEDICATIONS[patient_oop])
```

### Utilization

```dax
Total Encounters =
COUNTROWS(FACT_ENCOUNTER)

Total Patients =
DISTINCTCOUNT(FACT_ENCOUNTER[patient_id])

Encounters per Patient =
DIVIDE([Total Encounters], [Total Patients], 0)

Emergency Encounters =
CALCULATE(
    [Total Encounters],
    DIM_ENCOUNTER_TYPE[encounter_class] = "emergency"
)

Emergency Rate % =
DIVIDE([Emergency Encounters], [Total Encounters], 0)
```

### Clinical

```dax
Total Conditions =
COUNTROWS(FACT_CONDITIONS)

Chronic Conditions =
CALCULATE(
    COUNTROWS(FACT_CONDITIONS),
    FACT_CONDITIONS[is_chronic] = 1
)

Chronic Rate % =
DIVIDE([Chronic Conditions], [Total Conditions], 0)

Total Immunizations =
COUNTROWS(FACT_IMMUNIZATIONS)

Rapid Return Encounters =
CALCULATE(
    COUNTROWS(FACT_ENCOUNTER),
    FACT_ENCOUNTER[Days Since Last Visit] <= 30
)
```

### Time Intelligence

```dax
Total Encounters PY =
CALCULATE(
    [Total Encounters],
    SAMEPERIODLASTYEAR(DIM_DATE[full_date])
)

Encounters YoY % =
DIVIDE(
    [Total Encounters] - [Total Encounters PY],
    [Total Encounters PY],
    BLANK()
)

Total Claim Cost YTD =
CALCULATE(
    [Total Claim Cost],
    DATESYTD(DIM_DATE[full_date])
)

Encounters MoM % =
VAR CurrentMonth =
    CALCULATE([Total Encounters], DATESMTD(DIM_DATE[full_date]))
VAR PrevMonth =
    CALCULATE(
        [Total Encounters],
        DATEADD(DATESMTD(DIM_DATE[full_date]), -1, MONTH)
    )
RETURN DIVIDE(CurrentMonth - PrevMonth, PrevMonth, BLANK())
```

### HTML Content KPI Cards

All KPI cards are DAX measures rendered via the HTML Content visual (by Daniel Marsh-Patrick). Pattern:

```dax
<HTML> KPI Total Encounters =

VAR _curr      = [Total Encounters]
VAR _py        = [Total Encounters PY]
VAR _pct       = [Encounters YoY %] * 100
VAR _arrow     = IF(_pct < 0, "▼ ", "▲ ")
VAR _bgcol     = IF(_pct < 0, "#FCEBEB", "#E1F5EE")
VAR _txtcol    = IF(_pct < 0, "#A32D2D", "#085041")
VAR _pctlabel  = FORMAT(ABS(_pct), "0.00") & "%"
VAR _pylabel   = FORMAT(_py, "#,0")
VAR _currlabel = FORMAT(_curr, "#,0")

RETURN
    "<style>
        .kenc { font-family:Segoe UI,sans-serif; background:#FFFFFF;
                border-left:3px solid #1A1F5E; border-radius:0 8px 8px 0;
                padding:12px 16px; box-sizing:border-box; width:100%; }
        ...
    </style>
    <div class='kenc'>
        <div class='kenc-l'>Total encounters</div>
        <div class='kenc-v'>" & _currlabel & "</div>
        <div class='kenc-d'></div>
        <div class='kenc-f'>
            <span class='kenc-py'>vs last year · " & _pylabel & "</span>
            <span class='kenc-b' style='background:" & _bgcol & ";color:" & _txtcol & ";'>
                " & _arrow & _pctlabel & "
            </span>
        </div>
    </div>"
```

Key rules across all cards:
- Unique CSS class names per card to prevent style bleed across visuals on the same page
- Badge color logic inverted for metrics where decrease is good (Emergency Rate %)
- `white-space:nowrap` on footer elements keeps comparison row on one line
- `border-radius:0 8px 8px 0` matches the left-accent border design

---

## Dashboard Pages

### Page 1 — Executive Summary
**Audience:** Leadership, CIO

KPIs: Total Encounters · Total Claim Cost · Coverage Rate % · Patient OOP

Visuals: Dual-axis encounter/cost trend · Encounter class donut · Payer cost ranking bar · Monthly encounter volume

### Page 2 — Financial Deep Dive
**Audience:** CFO, billing analysts

KPIs: Total Claim Cost · Medication Cost · Procedure Cost · Urgent Care OOP Rate

Visuals: Stacked spend by category (year) · Payer coverage rate bars with logos · Patient OOP by encounter class · Top 10 drugs by cost · Top 10 procedures by cost · Medication coverage donut

Key finding: 93.7% of $100.1M medication spend is patient OOP

### Page 3 — Clinical Operations
**Audience:** Clinical directors, care managers

KPIs: Total Conditions · Chronic Conditions · Total Immunizations · Rapid Returns ≤30 days

Visuals: Top 15 conditions bar · Chronic vs resolved donut · Conditions by gender · Rapid returns by encounter class · Top immunizations · Top allergies · Imaging modality donut

Key finding: 21,804 encounters within 30 days of a prior visit across 1,059 patients

### Page 4 — Provider & Organization Performance
**Audience:** Operations, network management

KPIs: Total Providers · Zero Utilization % · Top Provider Encounters · Top Org Revenue

Visuals: Top 10 providers bar · Utilization distribution histogram · Specialty bar · Provider gender donut · Organization revenue bar · Utilization scatter

Key finding: 80.9% of providers have zero encounters

### Page 5 — Organization & Payer Deep Dive
**Audience:** Finance, payer relations

KPIs: Total Payers · Uninsured Encounters · Highest QOLS · Avg Coverage Years

Visuals: Coverage rate vs encounters combo · QOLS vs coverage scatter · Revenue per member bar with logos · Payer category coverage stacked bar · Ownership type donut · Payer scorecard table · Top 15 orgs by encounters · Top 15 orgs by revenue

Key finding: NO_INSURANCE is the largest payer by encounter volume at 10,175 encounters

### Drill-through pages

**Payer Detail** — right-click any payer → dedicated detail page filtered to that payer.
Setup: Add `DIM_PAYER[payer_name]` to the page's Drill-through field well.

**Patient 360** — right-click any patient → full encounter history, conditions, medications, OOP.
Setup: Add `DIM_PATIENT[patient_id]` to the page's Drill-through field well.

### Power BI Theme

**File:** `Synthea_Healthcare_Theme.json`

| Role | Hex | Usage |
|---|---|---|
| Primary | #1A1F5E | Bars, headers, card values |
| Accent | #F5C518 | Cost KPIs, secondary series |
| Positive | #1D9E75 | Coverage, healthy trends |
| Negative | #E24B4A | Alerts, emergency, OOP |
| Purple | #7F77DD | Procedures, 5th series |
| Page bg | #F0F2F8 | Report canvas |

Apply: View → Themes → Browse → `Synthea_Healthcare_Theme.json`

---

## Key Insights

| # | Insight | Metric |
|---|---|---|
| 1 | Medication spend dwarfs encounter cost | $100.1M medications vs $6.87M encounters |
| 2 | 93.7% of medication cost is uninsured | Only $6.3M of $100.1M payer-covered |
| 3 | Urgent care = 100% patient OOP | $306K with zero payer coverage |
| 4 | Cardioversion = 18% of all procedure spend | $31.1M of $172.5M total |
| 5 | 80.9% of providers have zero utilization | 4,735 of 5,855 providers |
| 6 | Top provider handles 7.5% of all encounters | 3,983 of 53,346 |
| 7 | 90% of patients are rapid returners | 1,059 of 1,171 return within 30 days |
| 8 | Uninsured = largest payer by encounter volume | 10,175 encounters, $1.31M OOP |
| 9 | Dual Eligible patients have lowest QOLS | 0.363 vs Anthem/UnitedHealth at 0.932 |
| 10 | VA Boston tops both revenue and utilization | $595K revenue, 4,828 utilization, 1 provider |

---

## How to Run

### Prerequisites

- Python 3.8+ with pandas, numpy
- Power BI Desktop (free)

### Steps

```bash
# 1. Clone the repo
git clone https://github.com/sarth-1411/Healthcare-Financial-Clinical-Operations-Dashboard
cd Healthcare-Financial-Clinical-Operations-Dashboard

# 2. Install dependencies
pip install pandas numpy

# 3. Generate Synthea data
# Download: https://synthetichealth.github.io/synthea/
# Place CSVs in: /csv/

# 4. Run the cleaning notebook
jupyter notebook Untitled-1.ipynb
# Cleaned files export to: /csv/cleaned/

# 5. Open Power BI Desktop
# Get Data → Text/CSV → select each file from /csv/cleaned/
# Or use the M queries from PowerQuery_M_AllTables.txt
#   → Home → New Source → Blank Query → Advanced Editor → paste M code
#   → Update FolderPath variable to your /csv/cleaned/ path

# 6. Apply theme
# View → Themes → Browse → Synthea_Healthcare_Theme.json

# 7. Import DAX measures
# Create blank table named _Measures
# Paste measures from DAX_Measures_Healthcare.txt
```

---

## Project Structure

```
Healthcare-Financial-Clinical-Operations-Dashboard/
│
├── csv/                               # Raw Synthea CSVs (not committed)
│   ├── patients.csv
│   ├── encounters.csv
│   └── ... (16 files)
│
├── csv/cleaned/                       # Python-cleaned output CSVs
│   ├── patients_clean.csv
│   ├── encounters_clean.csv
│   ├── procedures_clean.csv
│   ├── providers_clean.csv
│   ├── allergies_clean.csv
│   ├── conditions_clean.csv
│   └── payers_clean.csv
│
├── Untitled-1.ipynb                   # Python data cleaning notebook
├── PowerQuery_M_AllTables.txt         # M code for all tables in Power BI
├── DAX_Measures_Healthcare.txt        # All 50+ DAX measures by category
├── Synthea_Healthcare_Theme.json      # Power BI color theme
│
├── Healthcare_Dashboard.pbix          # Power BI report (publish separately)
└── README.md
```

---

## Author

**Sarth Patel**
MS Information Management (Data Analytics) — University of Illinois Urbana-Champaign
Microsoft Power BI Certified

- GitHub: [sarth-1411](https://github.com/sarth-1411)
- LinkedIn: [sarth-patel-s1411](https://linkedin.com/in/sarth-patel-s1411)

---

*Built with Synthea synthetic data. No real patient information is used or represented in this project.*
