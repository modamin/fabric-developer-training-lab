# Module 3 — Gold: dimensional model + SCD Type 2 (55 min)

Two notebooks: `nb_03a_gold_dimensions_scd2.ipynb` then
`nb_03b_gold_fact_asof.ipynb`. Attach `lh_meridian_hr` to both.

## Why two SCD2s?

| Dimension | Source shape | Technique | Real-world reason |
|-----------|-------------|-----------|-------------------|
| `dim_pay_band` | full **history feed** | derive validity from `lead()` | grids re-benchmarked yearly; you have the whole history |
| `dim_worker` | periodic **snapshots** | Delta **MERGE** (close + insert) | you get a fresh worker snapshot and must *detect* changes |

---

## 3a.1 `dim_date` (~3 min)

A generated date spine, keyed `yyyyMMdd`, with a `fiscal_year` column (April
start). Mark it as a date table in Module 4.

## 3a.2 `dim_cost_center` — SCD Type 1 (~5 min)

Overwrite current state, add a surrogate key. Carries **`hr_region`** — the
row-level-security driver used in Module 6.

## 3a.3 `dim_pay_band` — SCD2 from a history feed (~15 min)

Bronze holds one row per (group, level, effective_date). Turn it into valid SCD2
per (group, level) ordered by date:

```python
w = W.partitionBy("classification_group","classification_level").orderBy("band_effective_date")
scd2 = (hist
    .withColumn("effective_from", F.col("band_effective_date"))
    .withColumn("_next", F.lead("band_effective_date").over(w))
    .withColumn("effective_to",
        F.when(F.col("_next").isNull(), F.to_date(F.lit("9999-12-31")))
         .otherwise(F.date_sub(F.col("_next"), 1)))
    .withColumn("is_current", F.col("_next").isNull()))
```

**Invariants the notebook asserts:** exactly **one current row per (group,
level)**; **no overlapping** validity ranges. **Expected:** 200 versioned rows =
8 groups × 5 levels × 5 fiscal years; 40 current.

## 3a.4 `dim_worker` — SCD2 via periodic-snapshot MERGE (~20 min)

The **close-then-insert** MERGE, tracking `classification_group`,
`classification_level`, `directorate`, `employment_type` via a `row_hash`.

**Step 0 — initial load:** every worker → version 1 (`is_current=true`).

**Step 1 — close changed rows** (the 2023-07-01 snapshot brings promotions,
deployments, conversions):

```python
(tgt.alias("t").merge(snap2.alias("s"),
        "t.employee_id = s.employee_id AND t.is_current = true")
   .whenMatchedUpdate(
        condition="t.row_hash <> s.row_hash",
        set={"is_current": F.lit(False),
             "effective_to": F.expr("date_sub(s.effective_from, 1)")})
   .execute())
```

**Step 2 — insert new versions** for changed workers *and* brand-new employees:

```python
current = spark.table("gold.dim_worker").filter("is_current = true")
changed_or_new = (snap2.alias("s")
    .join(current.alias("c"), "employee_id", "left")
    .where("c.employee_id IS NULL OR c.row_hash <> s.row_hash")
    .select("s.employee_id","s.full_name","s.home_cost_center_id",
            "s.classification_group","s.classification_level","s.directorate",
            "s.employment_type","s.row_hash","s.effective_from",
            F.to_date(F.lit("9999-12-31")).alias("effective_to"),
            F.lit(True).alias("is_current")))
changed_or_new.write.format("delta").mode("append").saveAsTable("gold.dim_worker")
```

**Expected:** 5,000 → ~5,476 rows (5,200 current: 5,000 + 200 new hires; ~276
workers now carry a closed prior version). One current row per employee.

> **New employees appear at 2023-07-01**, so their `effective_from` is that date
> and every event for them is dated on/after it — the as-of join in 3b resolves
> cleanly with no nulls. This is why the generator only emits events for a worker
> on/after they exist.

Finally add the `worker_key` surrogate.

---

## 3b — `fact_workforce_event` with **as-of** resolution (~15 min)

SCD1 keys are direct equijoins. The two SCD2 keys use **as-of range joins**:

```python
# as-of worker
.join(dw, (ev.employee_id==dw.employee_id) &
          (ev.event_date>=dw.w_from) & (ev.event_date<=dw.w_to), "left")
# as-of pay band (group + level + date)
.join(dpb, (ev.classification_group==dpb.pb_group) &
           (ev.classification_level==dpb.pb_level) &
           (ev.event_date>=dpb.effective_from) &
           (ev.event_date<=dpb.effective_to), "left")
```

We precompute, for pay-setting events (`Hire`, `Promotion`, `Step Increment`):
`base_salary_cad`, the band bounds **at the event**, `compa_ratio_at_event`
(= base / band_mid in force then), and `below_band_at_event`. Performance-pay
amounts go to `bonus_cad`.

**Critical check** (the notebook runs it): fact row count **equals** Silver row
count. If it grew, a dimension has overlapping validity ranges and the range join
fanned out. Expected: 118,085 = 118,085, zero null keys.

### The payoff

```sql
SELECT d.year AS pay_set_year,
       round(avg(f.compa_ratio_at_event),3) AS avg_compa_ratio_as_was,
       sum(case when f.below_band_at_event then 1 else 0 end) AS below_band_as_was
FROM gold.fact_workforce_event f
JOIN gold.dim_date d ON f.date_key = d.date_key
WHERE f.base_salary_cad IS NOT NULL
GROUP BY d.year ORDER BY d.year;
```

Every year's pay looks healthy (~0.99) **against the grid in force then**. Module
4 adds the *as-is* measure — the same salaries against **today's** grid — and the
gap is real pay drift you can only compute because `pay_band_key` is versioned.

➡️ Continue to **Module 4 — Semantic model**.
