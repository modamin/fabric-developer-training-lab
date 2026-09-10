# Module 4 — Direct Lake on OneLake semantic model (35 min)

**Goal:** a **Direct Lake on OneLake** model over the `gold` schema, with
relationships, a marked date table, and a compa-ratio measure library that
answers the **as-was vs as-is** question.

> **Direct Lake on OneLake vs on SQL.** We build **on OneLake** (reads the Delta
> tables via `AzureStorage.DataLake`), because Module 6's fixed-identity + OneLake
> security story only applies to the on-OneLake variant.

## 4.1 Create the model (~8 min)

1. In the Fabric portal, open the workspace used in the previous modules.
2. Open the `lh_meridian_hr` lakehouse. Stay in the **Lakehouse** view; do not
    start from the SQL analytics endpoint unless the creation dialog explicitly
    lets you choose **Direct Lake on OneLake**.
3. On the lakehouse ribbon, select **New semantic model**. Alternatively, use **New Power BI semantic model** and **Get Data from Onelake**. 
4. In the creation dialog:
    - Enter `sm_meridian_workforce` as the semantic model name.
    - Confirm that it will be saved in the current lab workspace.
    - Expand the `gold` schema and select only these five tables:
      `fact_workforce_event`, `dim_date`, `dim_cost_center`, `dim_worker`, and
      `dim_pay_band`.
5. Select **Confirm** (or **OK**). Fabric creates the model and normally opens
    the web model editor in a new browser tab. If no tab opens, allow pop-ups,
    return to the workspace, open `sm_meridian_workforce`, and select
    **Open semantic model** or **Open data model**.
6. The editor initially opens in **Viewing** mode. In the upper-right corner,
    switch to **Editing** mode. Changes in this editor are automatically saved.

### Verify the model type

This lab requires **Direct Lake on OneLake**, not Direct Lake on SQL. Use either
of these checks:

1. In the model editor, inspect a source table's properties and confirm its
    storage mode is **Direct Lake** and that its source is the lakehouse/OneLake.
2. If you inspect the model as TMDL, confirm that a table partition references
    `AzureStorage.DataLake`; a `Sql.Database` partition indicates the SQL-backed
    variant.

If the model is SQL-backed, delete it and repeat the steps from the open
lakehouse or use **Create** → **OneLake catalog** and select `lh_meridian_hr`.

## 4.2 Relationships (~7 min)

1. From the top right corner, make sure to toggle edit mode for the semantic model.
2. In the model editor, select **Home** → **Manage relationships**.
3. Review any relationships Fabric detected automatically. Keep a relationship
    only when its columns and settings match the table below.
4. For each missing relationship, select **New relationship** and set:
    - the first table and column to the fact-side entry below;
    - the second table and column to the dimension-side entry below;
    - **Cardinality** to **Many to one (*:1)**;
    - **Cross-filter direction** to **Single**;
    - **Make this relationship active** to enabled.
5. Select **OK** after each relationship, then close **Manage relationships**.

All relationships are single-direction, many-to-one from the fact:

| From (fact) | To (dim) | Cardinality | Cross-filter |
|---|---|---|---|
| `fact[date_key]` | `dim_date[date_key]` | many-to-one | single |
| `fact[cost_center_key]` | `dim_cost_center[cost_center_key]` | many-to-one | single |
| `fact[worker_key]` | `dim_worker[worker_key]` | many-to-one | single |
| `fact[pay_band_key]` | `dim_pay_band[pay_band_key]` | many-to-one | single |

In the table, `fact` means `fact_workforce_event`. In the diagram, verify that
each relationship shows `*` on `fact_workforce_event`, `1` on the dimension,
and one filter arrow pointing from the dimension toward the fact. There should
be no relationships between dimensions.

Because `pay_band_key` points at the **versioned** band row, any compa-ratio
sliced by year is automatically **as-was** (the grid in force at the event).

## 4.3 Mark the date table (~2 min)

1. In the **Data** pane, right-click `dim_date`.
2. Select **Mark as date table** → **Mark as date table**.
3. Choose `date` in the date-column dropdown and confirm.

The `date` column must contain unique, nonblank, contiguous dates. The Gold
notebook created it that way. Marking the table unlocks reliable time
intelligence for the 12-month and year-over-year measures later in this module.

## 4.4 Tidy the model (~3 min)

Use the **Data** pane to select a column and the **Properties** pane to change
its settings. Hold **Ctrl** to select several columns with the same setting.

1. Hide every `*_key` column from report view. Keep the columns in the model;
    they are still required by the relationships.
2. In the fact table, set `amount_cad`, `base_salary_cad`, `bonus_cad`, and each
    `band_*_at_event` amount to **Currency**, CAD, with 0 or 2 decimal places.
3. Set `compa_ratio_at_event` to **Decimal number** with 2–3 decimal places.
4. In the date table, Set the `date` field  to a **Date** format and whole-number fields such as `year` to
    **Whole number** with the thousands separator disabled.
5. Give user-facing columns readable display names if desired, but don't rename
    any fields until after all DAX in this module has been created.

## 4.5 Create an empty measure table (~3 min)

Keep measures in a dedicated table instead of scattering them among source
tables. The table below contains only a hidden blank placeholder, so it appears
as a measure-only table to report authors and has no relationship to the model.

1. In **Editing** mode, select **Home** → **New table**.
2. Replace the formula-bar contents with this DAX, then select the check mark:

```dax
_Measures = { BLANK () }
```

3. Expand `_Measures` in the **Data** pane. Fabric creates one placeholder
    column, normally named `Value`.
4. Right-click the placeholder column and select **Hide in report view**. Do not
    create a relationship from `_Measures` to any other table.
5. Select `_Measures` before choosing **Home** → **New measure**. This makes
    `_Measures` the new measure's home table.

If a measure is accidentally created under another table, select the measure
and change its **Home table** property to `_Measures`.

## 4.6 Add the measure library (~12 min)

### Create the Movement measures manually

Create the first group through the graphical editor so you learn the standard
measure workflow:

1. Select `_Measures` in the **Data** pane.
2. Select **Home** → **New measure**.
3. Paste only the first definition below into the formula bar and select the
    check mark.
4. Select the new measure and set its **Display folder** to `Movement` and its
    format to **Whole number**, 0 decimal places.
5. Repeat these steps for the other four definitions. Create one measure at a
    time; don't paste the entire block into the formula bar.

```dax
Event Count = COUNTROWS ( fact_workforce_event )
Hires        = CALCULATE ( [Event Count], fact_workforce_event[event_type] = "Hire" )
Departures   = CALCULATE ( [Event Count], fact_workforce_event[event_type] = "Departure" )
Promotions   = CALCULATE ( [Event Count], fact_workforce_event[event_type] = "Promotion" )
Net Movement = [Hires] - [Departures]
```

Expand `_Measures` and confirm that all five Movement measures appear beneath
it before continuing.

### Add the remaining measures with TMDL view

TMDL view lets you edit multiple semantic-model objects as code and apply them
in one operation. It is currently a preview feature in the web model editor.

1. Select **TMDL View (Preview)** from the model editor.
2. If needed, switch TMDL view from **View** mode to **Edit** mode so that the
    **Apply** command is available.
3. In the **Data** pane, drag the entire `_Measures` table onto the empty TMDL
    editor. Fabric serializes the table into a `createOrReplace` script. The
    generated script includes the placeholder column, calculated partition, and
    five Movement measures; keep all of that generated content.
4. In the generated `table '_Measures'` block, find the last Movement measure.
    Paste the TMDL below immediately after it and before the next `column` or
    `partition` declaration. Keep the new `measure` declarations aligned with
    the existing Movement measure declarations.
5. Select **Preview** and confirm that the diff adds 11 measures without
    deleting the table, placeholder column, partition, or Movement measures.
6. Select **Apply**. A success notification confirms that the semantic-model
    metadata was updated. If it fails, select **Show details** and check that the
    pasted measures are inside `table '_Measures'` with their indentation
    intact.

> Do not replace the entire generated script with the fragment below. This is a
> set of child objects to insert inside the serialized `_Measures` table.

```tmdl
    measure 'Comp Events' = CALCULATE ( [Event Count], NOT ISBLANK ( fact_workforce_event[base_salary_cad] ) )
        formatString: #,0
        displayFolder: Comp

    measure 'Base Salary Set CAD' = SUM ( fact_workforce_event[base_salary_cad] )
        formatString: $#,0.00;($#,0.00);$#,0.00
        displayFolder: Comp

    measure 'Performance Pay CAD' = SUM ( fact_workforce_event[bonus_cad] )
        formatString: $#,0.00;($#,0.00);$#,0.00
        displayFolder: Comp

    measure 'Avg Compa-Ratio (as-was)' = AVERAGE ( fact_workforce_event[compa_ratio_at_event] )
        formatString: 0.000
        displayFolder: Pay Equity

    measure 'Below-Band Count (as-was)' =
            CALCULATE (
                COUNTROWS ( fact_workforce_event ),
                NOT ISBLANK ( fact_workforce_event[base_salary_cad] ),
                fact_workforce_event[below_band_at_event] = TRUE ()
            )
        formatString: #,0
        displayFolder: Pay Equity

    measure '% Below Band (as-was)' =
            DIVIDE (
                CALCULATE (
                    COUNTROWS ( fact_workforce_event ),
                    NOT ISBLANK ( fact_workforce_event[base_salary_cad] ),
                    fact_workforce_event[below_band_at_event] = TRUE ()
                ),
                CALCULATE (
                    COUNTROWS ( fact_workforce_event ),
                    NOT ISBLANK ( fact_workforce_event[base_salary_cad] )
                )
            )
        formatString: 0.0%
        displayFolder: Pay Equity

    measure 'Avg Compa-Ratio (as-is)' =
            AVERAGEX (
                FILTER ( fact_workforce_event, NOT ISBLANK ( fact_workforce_event[base_salary_cad] ) ),
                VAR g = fact_workforce_event[classification_group]
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
        formatString: 0.000
        displayFolder: Pay Equity

    measure 'Below-Band Count (as-is)' =
            SUMX (
                FILTER ( fact_workforce_event, NOT ISBLANK ( fact_workforce_event[base_salary_cad] ) ),
                VAR g = fact_workforce_event[classification_group]
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
        formatString: #,0
        displayFolder: Pay Equity

    measure 'Compa-Ratio Drift' =
            AVERAGEX (
                FILTER ( fact_workforce_event, NOT ISBLANK ( fact_workforce_event[base_salary_cad] ) ),
                VAR g = fact_workforce_event[classification_group]
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
                - AVERAGE ( fact_workforce_event[compa_ratio_at_event] )
        formatString: 0.000
        displayFolder: Pay Equity

    measure 'Base Salary Set 12M CAD' =
            CALCULATE (
                SUM ( fact_workforce_event[base_salary_cad] ),
                DATESINPERIOD ( dim_date[date], MAX ( dim_date[date] ), -12, MONTH )
            )
        formatString: $#,0.00;($#,0.00);$#,0.00
        displayFolder: Trends

    measure 'Promotions YoY %' =
            VAR cur = [Promotions]
            VAR prior = CALCULATE ( [Promotions], SAMEPERIODLASTYEAR ( dim_date[date] ) )
            RETURN DIVIDE ( cur - prior, prior )
        formatString: 0.0%
        displayFolder: Trends
```

The script actually adds **11** measures: three Compensation measures, six Pay
Equity measures, and two Trend measures. Their display folders and formats are
included in the TMDL, so no separate formatting pass is needed. Measures added
in the same pending TMDL batch don't reliably resolve references to one another
in the web editor. For that reason, the dependent measures repeat their base
expressions instead of referring to another new measure. References to the five
Movement measures work because those measures were applied manually first.

`Compa-Ratio Drift` is the headline: pay that was in-band when set has slipped
against the re-benchmarked grid. A negative drift on older cohorts is exactly the
salary-compression signal comp teams look for, and it's only computable because
the band dimension is versioned.

## 4.7 Prep data for AI (~10 min)

The semantic model's names, descriptions, AI data schema, and AI instructions
are used by both Power BI Copilot and the Fabric data agent. In particular, the
data agent's DAX generation is grounded in this model metadata, so configure it
before creating the agent in Module 5.

### Add table descriptions

In the model editor, select each table, find **Description** in the
**Properties** pane, and paste the matching description below. Changes are
saved automatically. Each description is fewer than 200 characters.

**`fact_workforce_event`**

```text
One row per workforce event. Use for movement and compensation-event analysis, not current headcount. Includes CAD amounts, compa-ratio, and below-band status when pay was set.
```

**`dim_date`**

```text
Daily calendar from 2021 through 2025 with year, quarter, month, and April-to-March fiscal year labeled by its starting year. Use for date filters and time intelligence.
```

**`dim_cost_center`**

```text
Current Type 1 cost-centre reference with name, branch, business line, and HR region. Use HR region for regional analysis and row-level security; attributes are not historical.
```

**`dim_worker`**

```text
Type 2 worker history with classification, directorate, and employment type valid for each period. Events link to the event-date version. Use only for aggregate analysis.
```

**`dim_pay_band`**

```text
Type 2 pay-band history by classification group and level, with minimum, midpoint, maximum, validity dates, and current flag. Supports event-date as-was and current-grid as-is analysis.
```

**`_Measures`**

```text
Curated measures for movement, compensation, pay-band compliance, and trends. Prefer these over raw fact columns. As-was uses the event-date grid; as-is uses the current grid.
```

Also add descriptions to the three headline measures:

| Measure | Description |
|---|---|
| `Avg Compa-Ratio (as-was)` | Average salary-to-band-midpoint ratio using the pay grid in force when each salary was set. |
| `Avg Compa-Ratio (as-is)` | Average salary-to-band-midpoint ratio after reevaluating each salary against the current pay grid. |
| `Compa-Ratio Drift` | Difference between as-is and as-was average compa-ratio; a negative value indicates salaries have fallen behind the re-benchmarked grid. |



### Add AI instructions

1. From the top right corner in the **Home** menu, select **Prep data for AI** select **Add AI instructions**.
2. Paste the complete instruction block below into the text box.
3. Select **Apply**. It can take a few minutes before Copilot and data-agent
   responses reflect the update.

```text
You analyze Meridian workforce events for the Workforce Analytics team.

Model and calculation rules:
- fact_workforce_event contains one row per workforce event. It is an event ledger, not a current-headcount snapshot. Do not interpret its row count as employee headcount.
- Prefer the curated measures in _Measures instead of creating implicit aggregations from raw fact columns.
- Event Count counts workforce events. Hires, Departures, and Promotions count their named event types. Net Movement equals Hires minus Departures.
- Comp events are Hire, Promotion, and Step Increment events because those events set a base salary.
- Compa-ratio means base salary divided by the pay-band midpoint for the employee's classification group and level.
- As-was means evaluating pay against the pay grid in force when the salary was set. Use as-was measures by default for pay compliance.
- As-is or today's grid means reevaluating the historical salary against the current re-benchmarked pay grid. When a question asks about drift or today's grid, use the as-is measures and Compa-Ratio Drift.
- Below band means base salary is below the applicable pay-band minimum.
- Report monetary values in Canadian dollars (CAD).
- Use dim_date for calendar and fiscal time filters. Meridian's fiscal year runs from April through March and is labeled by its starting year.
- Use dim_cost_center for branch, business line, and HR region analysis. Use dim_worker for aggregate classification, directorate, and employment-type analysis.
- Classification consists of a classification group, such as PA, IT, or EC, and a classification level from 1 through 5.
- Never use employee_id or full_name in generated queries or responses. Return aggregate results only. If asked about an individual, explain that the model is intended for aggregate workforce analysis and offer a classification, directorate, cost-centre, or regional summary instead.
- If a question could mean either as-was or as-is, answer with as-was and clearly label it. Include the as-is result when it helps explain pay-grid drift.
```

These are model-level AI instructions and are distinct from the data-agent
behavioral instructions added in Module 5. Keep both: this configuration guides
semantic-model field and measure selection, while Module 5 configures how the
agent presents and safeguards its final answers.

If synonym editing is available, add "comp-ratio" for compa-ratio,
"classification" or "group" for classification group, and "level" or "step"
for classification level. Confirm the display folders remain `Movement`,
`Comp`, `Pay Equity`, and `Trends`.

## Checkpoint

1. Select **New report** from the model editor.
2. Add a **Matrix** visual.
3. Add `dim_date[year]` to **Rows**.
4. Add `[Avg Compa-Ratio (as-was)]`, `[Avg Compa-Ratio (as-is)]`, and
    `[Compa-Ratio Drift]` to **Values**.
5. Sort by year ascending.

Older years should show a widening negative drift as the grid moved beneath
salaries set back then. If every year is blank, confirm the date relationship is
active and the measures are valid. If as-was and as-is are identical for every
year, confirm `dim_pay_band[is_current]` contains `TRUE` values and that the
as-is measures use `ALL ( dim_pay_band )`.

➡️ Continue to **Module 5 — Data agent**.
