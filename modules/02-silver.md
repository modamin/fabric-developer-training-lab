# Module 2 — Silver: conform, dedup, validate, convert (40 min)

**Goal:** one trustworthy row per workforce event. Open
**`nb_02_silver_conform_dedup.ipynb`**, attach `lh_meridian_hr`, run cell by cell.

## 2.1 Deduplicate replays (~5 min)

The payroll system **replays** events — the same `event_id` arrives again with a
fresher `ingest_ts`. Rank by `ingest_ts` descending within each `event_id`, keep
rank 1.

```python
w = W.partitionBy("event_id").orderBy(F.col("ingest_ts").desc())
dedup = raw.withColumn("_rn", F.row_number().over(w)).filter("_rn=1").drop("_rn")
```

**Expected:** ~124,485 → ~119,697 (≈4,788 replays removed).

## 2.2 Conform the work-country code (~8 min)

Two source systems disagree: `CORE_HR` emits **ISO-3** (`CAN`), `PAYROLL` emits
**ISO-2** (`CA`). The workforce spans a fixed set of countries, so a small static
map is enough.

```python
iso = spark.createDataFrame([("CA","CAN"),("US","USA"),("GB","GBR"),...], ["_i2","_i3"])
conf = (dedup
    .withColumn("wcc3", F.when(F.length("work_country_code")==3, F.col("work_country_code")))
    .join(iso, dedup.work_country_code==iso._i2, "left")
    .withColumn("work_country_iso3", F.coalesce("wcc3","_i3")))
```

## 2.3 Validate & quarantine (~12 min)

| Rule | Reason code |
|------|-------------|
| `amount_local < 0` | `negative_pay` |
| `employee_id` doesn't resolve | `orphan_employee` |
| `event_date` before 2021-01-01 or in the future | `date_out_of_range` |
| unknown currency **or** unresolved country | `unresolved_reference` |

> **The key difference from a sales fact:** most workforce events (leaves,
> deployments) legitimately have **no amount**. The rule flags *negative* pay,
> **not null** pay — a naïve `amount <= 0` check would quarantine half your
> events. This is the single most common mistake in this module.

**Expected quarantine:** ~1,600 total — ~280 negative pay, ~450 orphan, ~440 bad
date, ~450 unresolved. **Clean:** ~118,085, of which ~48,000 have a null amount
(and that's correct).

## 2.4 Convert pay to CAD (~10 min)

The grid is in CAD; international offices pay in local currency. Convert
`amount_local` → `amount_cad` on `(rate_month, currency)`; keep null amounts null.

```python
fx_lkp = fx.select(F.col("rate_month").alias("_rm"),
                   F.col("currency").alias("_ccy"), "cad_per_unit")
silver = (clean
    .withColumn("rate_month", F.date_format("event_date","yyyy-MM"))
    .join(fx_lkp, (F.col("rate_month")==F.col("_rm")) &
                  (F.col("local_currency")==F.col("_ccy")), "left")
    .withColumn("amount_cad",
        F.when(F.col("amount_local").isNull(), None)
         .otherwise(F.round(F.col("amount_local")*F.coalesce("cad_per_unit",F.lit(1.0)),2))))
```

> ⚠️ Watch the **ambiguous column** trap: both sides can carry `rate_month`.
> Alias the FX side (`_rm`, `_ccy`).

## Checkpoint

```python
print(spark.table("silver.workforce_event").count())          # ~118,085
# rows that HAVE a local amount must have a CAD amount:
spark.table("silver.workforce_event") \
     .filter("amount_local IS NOT NULL AND amount_cad IS NULL").count()  # 0
spark.table("silver.workforce_event_quarantine").groupBy("dq_reason").count().show()
```

➡️ Continue to **Module 3 — Gold + SCD2**.
