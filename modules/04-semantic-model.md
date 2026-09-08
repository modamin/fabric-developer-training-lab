# Module 4 — Direct Lake on OneLake semantic model (35 min)

**Goal:** a **Direct Lake on OneLake** model over the `gold` schema, with
relationships, a marked date table, and a compa-ratio measure library that
answers the **as-was vs as-is** question.

> **Direct Lake on OneLake vs on SQL.** We build **on OneLake** (reads the Delta
> tables via `AzureStorage.DataLake`), because Module 6's fixed-identity + OneLake
> security story only applies to the on-OneLake variant.

## 4.1 Create the model (~8 min)

1. Open `lh_meridian_hr` → **New semantic model**. Select the **gold** schema.
2. Name it `sm_meridian_workforce`. Include:
   `fact_workforce_event`, `dim_date`, `dim_cost_center`, `dim_worker`,
   `dim_pay_band`.
3. **Create.** Verify it's Direct Lake on OneLake (the partition should reference
   `AzureStorage.DataLake` in the TMDL, not `Sql.Database`).

## 4.2 Relationships (~7 min)

All single-direction, many-to-one from the fact:

| From (fact) | To (dim) | Cardinality | Cross-filter |
|---|---|---|---|
| `fact[date_key]` | `dim_date[date_key]` | many-to-one | single |
| `fact[cost_center_key]` | `dim_cost_center[cost_center_key]` | many-to-one | single |
| `fact[worker_key]` | `dim_worker[worker_key]` | many-to-one | single |
| `fact[pay_band_key]` | `dim_pay_band[pay_band_key]` | many-to-one | single |

Because `pay_band_key` points at the **versioned** band row, any compa-ratio
sliced by year is automatically **as-was** (the grid in force at the event).

## 4.3 Mark the date table (~2 min)

Select `dim_date` → **Mark as date table** → `date`. Unlocks time intelligence.

## 4.4 Tidy the model (~3 min)

- Hide every `*_key` column.
- Format `amount_cad`, `base_salary_cad`, `bonus_cad`, `band_*_at_event` as
  **Currency**.
- Format `compa_ratio_at_event` as a **decimal** (2–3 places).

## 4.5 Measure library (~12 min)

Also in `measures.dax`. Folders noted.

**Movement** (folder: *Movement*)

```dax
Event Count = COUNTROWS ( fact_workforce_event )
Hires        = CALCULATE ( [Event Count], fact_workforce_event[event_type] = "Hire" )
Departures   = CALCULATE ( [Event Count], fact_workforce_event[event_type] = "Departure" )
Promotions   = CALCULATE ( [Event Count], fact_workforce_event[event_type] = "Promotion" )
Net Movement = [Hires] - [Departures]
```

**Compensation flow** (folder: *Comp*)

```dax
Comp Events = CALCULATE ( [Event Count], NOT ISBLANK ( fact_workforce_event[base_salary_cad] ) )
Base Salary Set CAD   = SUM ( fact_workforce_event[base_salary_cad] )
Performance Pay CAD   = SUM ( fact_workforce_event[bonus_cad] )
```

**Pay-band compliance — as-was** (folder: *Pay Equity*)

```dax
Avg Compa-Ratio (as-was) = AVERAGE ( fact_workforce_event[compa_ratio_at_event] )

Below-Band Count (as-was) =
CALCULATE ( [Comp Events], fact_workforce_event[below_band_at_event] = TRUE () )

% Below Band (as-was) = DIVIDE ( [Below-Band Count (as-was)], [Comp Events] )
```

**Pay-band compliance — as-is** (folder: *Pay Equity*) — reaches past the
versioned relationship to **today's** grid via `is_current`:

```dax
Avg Compa-Ratio (as-is) =
AVERAGEX (
    FILTER ( fact_workforce_event, NOT ISBLANK ( fact_workforce_event[base_salary_cad] ) ),
    VAR g  = fact_workforce_event[classification_group]
    VAR lv = fact_workforce_event[classification_level]
    VAR curMid =
        CALCULATE (
            MAX ( dim_pay_band[band_mid] ),
            ALL ( dim_pay_band ),
            dim_pay_band[classification_group] = g,
            dim_pay_band[classification_level] = lv,
            dim_pay_band[is_current] = TRUE ()
        )
    RETURN DIVIDE ( fact_workforce_event[base_salary_cad], curMid )
)

Below-Band Count (as-is) =
SUMX (
    FILTER ( fact_workforce_event, NOT ISBLANK ( fact_workforce_event[base_salary_cad] ) ),
    VAR g  = fact_workforce_event[classification_group]
    VAR lv = fact_workforce_event[classification_level]
    VAR curMin =
        CALCULATE (
            MAX ( dim_pay_band[band_min] ),
            ALL ( dim_pay_band ),
            dim_pay_band[classification_group] = g,
            dim_pay_band[classification_level] = lv,
            dim_pay_band[is_current] = TRUE ()
        )
    RETURN IF ( fact_workforce_event[base_salary_cad] < curMin, 1 )
)

Compa-Ratio Drift = [Avg Compa-Ratio (as-is)] - [Avg Compa-Ratio (as-was)]
```

`Compa-Ratio Drift` is the headline: pay that was in-band when set has slipped
against the re-benchmarked grid. A negative drift on older cohorts is exactly the
salary-compression signal comp teams look for — and it's only computable because
the band dimension is versioned.

**Trends** (folder: *Trends*)

```dax
Base Salary Set 12M CAD =
CALCULATE (
    [Base Salary Set CAD],
    DATESINPERIOD ( dim_date[date], MAX ( dim_date[date] ), -12, MONTH )
)

Promotions YoY % =
VAR cur = [Promotions]
VAR prior = CALCULATE ( [Promotions], SAMEPERIODLASTYEAR ( dim_date[date] ) )
RETURN DIVIDE ( cur - prior, prior )
```

## 4.6 Prep for AI (~2 min)

Fill in **Descriptions** on tables and key measures, add synonyms an HR analyst
would use ("comp-ratio" → compa-ratio, "classification"/"group" → classification
group, "level"/"step" → classification level), and keep the display folders
above. This directly improves the Module 5 agent.

## Checkpoint

Matrix: rows = `dim_date[year]`, values = `[Avg Compa-Ratio (as-was)]`,
`[Avg Compa-Ratio (as-is)]`, `[Compa-Ratio Drift]`. Older years show a widening
negative drift as the grid moved beneath salaries set back then.

➡️ Continue to **Module 5 — Data agent**.
