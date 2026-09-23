# Data Documentation

This folder documents the data sources, market definition, preparation workflow, geography harmonisation, population integration, quality-assurance process, and key analytical limitations used in the **PPI Market Commercial Analytics** project.

Raw NHS prescribing files are not stored directly in this repository because of file size.

---

# Data Sources

The project combines three publicly available NHS data sources.

## 1. NHS Business Services Authority — English Prescribing Dataset

Used for:

- monthly prescription items;
- molecule identification;
- presentation-level prescribing;
- Net Ingredient Cost;
- Actual Cost;
- practice-level prescribing;
- historical prescribing trends.

The final analytical period is:

> **January 2025 – July 2026**

The primary headline comparison is:

> **January–July 2026 vs January–July 2025**

---

## 2. Patients Registered at a GP Practice — July 2026

Used for:

- registered GP population;
- practice-level population denominator.

The July 2026 snapshot is used as a static denominator for population-adjusted ICB benchmarking.

---

## 3. July 2026 Practice-to-ICB Mapping

Used for:

- current practice-to-ICB assignment;
- harmonising prescribing geography across 2025 and 2026;
- rebuilding ICB-level comparison after detecting structural geography changes.

---

# Market Definition

The primary PPI market includes five molecules:

- Omeprazole
- Lansoprazole
- Esomeprazole
- Pantoprazole
- Rabeprazole

The analysis is performed at molecule level.

Therefore:

> **Market share refers to prescription-item share within the defined five-molecule PPI market.**

---

# Analytical Period

The final prescribing dataset covers:

> **January 2025 – July 2026**

This provides 19 monthly periods.

The project uses:

- Jan–Jul 2026 vs Jan–Jul 2025 for headline year-on-year analysis;
- Jan 2025–Jul 2026 for historical trends.

This period was selected to balance analytical coverage with Power BI model performance.

---

# Final Prescribing Fact Table

The final prescribing fact table contains:

- `YearMonth`
- `ICBCode`
- `ICBName`
- `PracticeCode`
- `ChemicalCode`
- `Molecule`
- `PresentationCode`
- `PresentationName`
- `Items`
- `TotalQuantity`
- `NIC`
- `ActualCost`
- `SNOMEDCode`

Historical ICB fields remain available for QA but are hidden from report view after the geography model was rebuilt.

---

# Dimension Tables

## `Dim_Calendar`

Contains:

- `YearMonth`
- `Year`
- `MonthNumber`
- `MonthName`
- `Quarter`
- `MonthYear`
- `YearMonthSort`

Expected row count:

> **19**

---

## `Dim_Chemical`

Contains:

- `ChemicalCode`
- `Molecule`
- `TherapyClass`
- `PPIFlag`

Expected molecule count:

> **5**

---

## `Dim_Presentation`

Contains:

- `PresentationCode`
- `PresentationName`
- `ChemicalCode`
- `Molecule`
- `SNOMEDCode`
- `Formulation`
- `Strength`
- `PresentationGroup`

Presentation-level attributes support formulation and strength analysis.

---

## `Dim_Practice`

Contains:

- `PracticeCode`
- `ICBCode`

Each practice is mapped to the harmonised July 2026 ICB structure.

Duplicates are removed based on:

> `PracticeCode`

---

## `Dim_ICB`

Contains:

- `ICBCode`
- `ICBName`

Duplicates are removed based on:

> `ICBCode`

The table represents the July 2026 ICB geography used for comparative analysis.

---

## `Dim_PracticePopulation`

Contains:

- `PracticeCode`
- `ICBCode`
- `RegisteredPopulation`

Duplicates are removed based on:

> `PracticeCode`

This table provides the July 2026 population denominator used for ICB-level prescribing-intensity calculations.

---

# Data Preparation Workflow

The data-preparation workflow was completed in Power Query.

The main steps included:

1. importing monthly NHS prescribing files;
2. keeping the required analytical fields;
3. harmonising source schemas;
4. standardising field names;
5. correcting data types;
6. converting month values into proper dates;
7. filtering to the five defined PPI molecules;
8. appending monthly datasets;
9. building dimension tables;
10. importing July 2026 GP population data;
11. importing July 2026 practice-to-ICB mapping;
12. rebuilding practice geography;
13. removing ambiguous relationship paths;
14. revalidating ICB-level YoY measures;
15. validating population-adjusted measures.

---

# Source Field Mapping

The core prescribing fields were standardised as follows:

| Source Field | Final Name |
|---|---|
| `YEAR_MONTH` | `YearMonth` |
| `ICB_CODE` | `ICBCode` |
| `ICB_NAME` | `ICBName` |
| `PRACTICE_CODE` | `PracticeCode` |
| `BNF_CHEMICAL_SUBSTANCE` | `ChemicalCode` |
| `CHEMICAL_SUBSTANCE_BNF_DESCR` | `Molecule` |
| `BNF_CODE` | `PresentationCode` |
| `BNF_DESCRIPTION` | `PresentationName` |
| `ITEMS` | `Items` |
| `TOTAL_QUANTITY` | `TotalQuantity` |
| `NIC` | `NIC` |
| `ACTUAL_COST` | `ActualCost` |
| `SNOMED_CODE` | `SNOMEDCode` |

---

# Data Types

The final prescribing table uses:

| Column | Data Type |
|---|---|
| YearMonth | Date |
| ICBCode | Text |
| ICBName | Text |
| PracticeCode | Text |
| ChemicalCode | Text |
| Molecule | Text |
| PresentationCode | Text |
| PresentationName | Text |
| Items | Whole Number |
| TotalQuantity | Decimal Number |
| NIC | Decimal Number |
| ActualCost | Decimal Number |
| SNOMEDCode | Text |

---

# YearMonth Transformation

Monthly values stored in `YYYYMM` format were converted into proper dates using Power Query.

Example:

```powerquery
#date(
    Number.FromText(Text.Start(Text.From([YearMonth]), 4)),
    Number.FromText(Text.End(Text.From([YearMonth]), 2)),
    1
)
```

Example:

```text
202501
```

becomes:

```text
01/01/2025
```

---

# Calendar Fields

Additional calendar fields were created.

## MonthYear

```powerquery
Date.ToText([YearMonth], "MMM yyyy")
```

Example:

```text
Jan 2025
```

## YearMonthSort

```powerquery
Date.Year([YearMonth]) * 100 + Date.Month([YearMonth])
```

Example:

```text
202501
```

`MonthYear` is sorted by `YearMonthSort`.

---

# Financial Data-Type Handling

NIC and Actual Cost required careful parsing.

The fields were explicitly converted to decimal numeric values before further modelling.

Example:

```powerquery
{"NIC", type number},
{"ACTUAL_COST", type number},
{"TOTAL_QUANTITY", type number},
{"ITEMS", Int64.Type}
```

Locale-aware conversion was used where required.

Fixed scaling factors were not applied.

This prevented artificial distortion of cost values.

---

# Presentation Attributes

## Formulation

Presentation names were classified into simplified formulation categories.

Example Power Query logic:

```powerquery
let
    p = Text.Lower([PresentationName])
in
    if Text.Contains(p, "capsule") then "Capsule"
    else if Text.Contains(p, "tablet") then "Tablet"
    else if Text.Contains(p, "oral suspension") then "Oral Suspension"
    else if Text.Contains(p, "oral solution") then "Oral Solution"
    else if Text.Contains(p, "granule") then "Granules"
    else if Text.Contains(p, "sachet") then "Sachet"
    else if Text.Contains(p, "dispersible") then "Dispersible Tablet"
    else "Other"
```

---

## Strength

Strength was extracted from presentation names where possible.

Example logic:

```powerquery
let
    t = Text.Lower([PresentationName]),
    result =
        if Text.Contains(t, "microgram") then
            let
                beforeUnit = Text.BeforeDelimiter(t, "microgram"),
                tokens = Text.Split(beforeUnit, " "),
                lastToken = List.Last(List.Select(tokens, each _ <> ""))
            in
                lastToken & " micrograms"

        else if Text.Contains(t, "mg") then
            let
                beforeUnit = Text.BeforeDelimiter(t, "mg"),
                tokens = Text.Split(beforeUnit, " "),
                lastToken = List.Last(List.Select(tokens, each _ <> ""))
            in
                lastToken & " mg"

        else
            null
in
    result
```

---

# Population Data Preparation

The July 2026 GP population file was reduced to:

- `PracticeCode`
- `RegisteredPopulation`

The practice-level population was then merged with the July 2026 practice-to-ICB mapping to add:

- `ICBCode`

Final structure:

```text
PracticeCode
ICBCode
RegisteredPopulation
```

Rows with missing ICB mappings were excluded from the geography denominator table.

This does not remove prescribing records from the prescribing fact table.

---

# Practice-to-ICB Mapping

The July 2026 mapping file was used to harmonise geography.

Core fields used:

- `PracticeCode`
- `ICBCode`
- `ICBName`

This mapping was merged into:

- `Dim_Practice`
- `Dim_PracticePopulation`

The final practice dimension contains:

```text
PracticeCode
ICBCode
```

---

# ICB Geography Harmonisation

## Initial Issue

During QA, several ICB-level YoY growth rates showed implausible declines of approximately:

> **-56% to -58%**

These values appeared across multiple ICBs rather than as isolated outliers.

This pattern indicated that the issue was likely structural rather than a genuine prescribing collapse.

---

## Root Cause

The project period spans changes to the English ICB structure during 2026.

Historical prescribing records contained geography associated with the structure that existed when the prescribing data was recorded.

The July 2026 population snapshot reflected the later ICB structure.

Using the historical ICB codes directly therefore created inconsistent geography between:

- Jan–Jul 2025;
- Jan–Jul 2026; and
- the July 2026 population denominator.

This produced artificial ICB-level year-on-year changes.

---

# Geography Harmonisation Solution

The model was rebuilt so that both 2025 and 2026 prescribing were evaluated using the same July 2026 practice geography.

The logic is:

```text
Historical Prescribing
       |
       | PracticeCode
       v
July 2026 Practice Mapping
       |
       | ICBCode
       v
July 2026 ICB Structure
```

This means prescribing from different periods is assigned to the same harmonised ICB structure.

---

# Data Quality Workflow

The geography issue was resolved through the following process:

1. identify abnormal ICB YoY growth outliers;
2. confirm that the pattern affected multiple ICBs;
3. investigate practice and ICB mappings;
4. identify geography inconsistency across periods;
5. obtain July 2026 NHS practice-to-ICB mapping;
6. merge the mapping into `Dim_Practice`;
7. merge the mapping into `Dim_PracticePopulation`;
8. rebuild `Dim_ICB` using the July 2026 structure;
9. remove the direct historical ICB filtering path from prescribing;
10. route prescribing geography through `PracticeCode`;
11. eliminate ambiguous relationship paths;
12. rerun ICB YoY growth QA;
13. confirm that artificial -56% to -58% declines were removed;
14. rerun Items per 1,000 and MDI validation.

This quality-assurance process is a central part of the project rather than a cosmetic modelling adjustment.

---

# Final Geography Relationships

The geography model uses:

```text
Dim_ICB
   |
   | ICBCode
   v
Dim_Practice
   |
   | PracticeCode
   v
Fact_Prescribing
```

Population uses:

```text
Dim_ICB
   |
   | ICBCode
   v
Dim_PracticePopulation
```

This structure ensures that:

- prescribing;
- population; and
- ICB-level analysis

use the same harmonised geography.

---

# Avoiding Ambiguous Relationships

No direct active relationship is used between:

```text
Dim_ICB
and
Fact_Prescribing
```

once the harmonised practice mapping is applied.

Likewise, no direct relationship is required between:

```text
Dim_Practice
and
Dim_PracticePopulation
```

The two analytical branches instead meet through:

> `Dim_ICB`

This avoids multiple active filtering paths.

---

# Population Measures

## Registered Population

```DAX
Registered Population =
SUM(
    Dim_PracticePopulation[RegisteredPopulation]
)
```

---

## PPI Items per 1,000

```DAX
PPI Items per 1,000 =
DIVIDE(
    [Total Items],
    [Registered Population]
) * 1000
```

This measure adjusts prescribing volume for registered GP population.

---

# England Benchmark

```DAX
England PPI Items per 1,000 =
VAR EnglandItems =
    CALCULATE(
        [Total Items],
        REMOVEFILTERS(Dim_ICB)
    )
VAR EnglandPopulation =
    CALCULATE(
        [Registered Population],
        REMOVEFILTERS(Dim_ICB)
    )
RETURN
DIVIDE(
    EnglandItems,
    EnglandPopulation
) * 1000
```

For Jan–Jul 2026, the England benchmark was approximately:

> **717.6 PPI items per 1,000 registered population**

---

# Market Development Index

Population MDI is calculated as:

```DAX
Population MDI =
DIVIDE(
    [PPI Items per 1,000],
    [England PPI Items per 1,000]
) * 100
```

Interpretation:

```text
MDI = 100
Equal to England prescribing intensity

MDI > 100
Above England prescribing intensity

MDI < 100
Below England prescribing intensity
```

MDI is formatted as a decimal index rather than a percentage.

---

# Geographic Growth

ICB YoY Growth is calculated using the same prescribing-growth logic as the national analysis.

```DAX
ICB YoY Growth =
[Items YoY Growth]
```

England growth is calculated by removing the ICB filter:

```DAX
England ICB YoY Growth =
CALCULATE(
    [Items YoY Growth],
    REMOVEFILTERS(Dim_ICB)
)
```

For Jan–Jul 2026, the England benchmark was approximately:

> **+0.23%**

---

# Key QA Results

## National Market

For Jan–Jul 2026:

- Total PPI Items: approximately **45.5M**
- PPI YoY Growth: approximately **+0.23%**
- NIC: approximately **£91.6M**
- NIC YoY Growth: approximately **-2.55%**

---

## Molecule Share

Volume Share totals were validated to:

> **100%**

NIC Share totals were also validated to:

> **100%**

Value-Volume Gap totals were validated to:

> approximately **0 percentage points**

---

## Competitive Movement

Examples identified during QA:

- Lansoprazole absolute growth: approximately **+571K items**
- Omeprazole absolute growth: approximately **-517K items**
- Net category growth: approximately **+106K items**

This explains why contribution-to-growth percentages can exceed 100%.

---

# Interpretation Notes

## Prescription Items Are Not Patients

Prescription items reflect prescribing activity.

They should not be interpreted as unique patient counts.

---

## Market Share Is Item-Based

Volume share is calculated using prescription items.

It is not equivalent to:

- patient share;
- treatment-equivalent share;
- defined daily dose share.

---

## NIC Is Not Revenue

NIC represents recorded ingredient cost.

It should not be interpreted as:

- manufacturer revenue;
- sales;
- margin;
- profitability;
- realised selling price.

---

## NIC per Item Is Not ASP

NIC per Item is used as an analytical cost-intensity measure.

It is not an average selling price.

---

## MDI Is Not Treatment Penetration

Population-adjusted prescribing intensity may reflect differences in:

- age structure;
- underlying disease burden;
- local prescribing behaviour;
- formularies;
- prescribing guidelines;
- treatment duration;
- deprescribing activity.

Therefore:

> **Population MDI is a relative prescribing-intensity benchmark, not an estimate of treatment penetration or unmet need.**

---

# Static Population Denominator

The project uses July 2026 registered GP population as a static denominator.

This supports cross-sectional ICB comparison but does not model population change over time.

---

# Harmonised Geography

Historical prescribing is analysed using the July 2026 practice-to-ICB structure.

This improves comparability across periods but means the geography is an analytical harmonisation rather than an exact recreation of the historical organisational structure.

---

# Missing Geography

Some prescribing records may not match the July 2026 current-practice mapping.

These records remain in national prescribing totals where possible.

They are excluded from geography-specific ICB analysis if no valid current ICB mapping is available.

---

# Reproducibility Workflow

To reproduce the dataset:

1. Download monthly NHS prescribing files for Jan 2025–Jul 2026.
2. Filter to the five PPI molecules.
3. Standardise source schemas.
4. Parse NIC and Actual Cost as decimal numeric fields.
5. Convert month values to proper dates.
6. Append monthly prescribing files.
7. Build `Dim_Calendar`.
8. Build `Dim_Chemical`.
9. Build `Dim_Presentation`.
10. Download July 2026 GP population data.
11. Download July 2026 practice-to-ICB mapping.
12. Build `Dim_Practice`.
13. Build `Dim_ICB`.
14. Build `Dim_PracticePopulation`.
15. Harmonise prescribing geography through `PracticeCode`.
16. Validate model relationships.
17. Create growth, share, NIC, population, and MDI measures.
18. Validate national totals.
19. Validate molecule share totals.
20. Investigate geographic outliers before dashboard publication.

---

# Why Raw Data Is Not Included

The raw NHS prescribing files are large and are not stored in this repository.

Instead, this folder documents:

- source data;
- field selection;
- transformation logic;
- data types;
- geography mapping;
- population integration;
- QA workflow;
- methodological limitations.

Users wishing to reproduce the project should obtain the source data directly from the relevant NHS public data services.

---

# Disclaimer

This project was developed independently for educational and portfolio purposes using publicly available NHS data.

It is not affiliated with, endorsed by, or produced on behalf of the NHS, any pharmaceutical manufacturer, or any other commercial organisation.

The documentation is intended to demonstrate data preparation, Power Query, data modelling, DAX, data-quality assurance, geography harmonisation, and pharmaceutical commercial analytics.

It should not be interpreted as clinical, financial, investment, or commercial advice.
