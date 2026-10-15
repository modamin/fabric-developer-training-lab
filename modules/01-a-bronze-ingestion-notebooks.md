# Module 01-a - Bronze ingestion with notebooks only (30 min)

**Goal:** land the same raw Bronze tables as Module 1 without creating or running
a data pipeline. Use the existing
[nb_01_ingest_reference_feeds.ipynb](../notebooks/nb_01_ingest_reference_feeds.ipynb)
to ingest both the monthly workforce events and the reference feeds.

> **Choose one ingestion path:** complete either
> [Module 1 - pipeline + notebook](01-bronze-ingestion.md) or this lab, not both
> in the same lakehouse. Both paths feed the same Silver and Gold modules.
> This path is a full snapshot reload, not the pipeline's incremental watermark
> exercise.

## A.1 Prepare the notebook (5 min)

1. Complete [Module 0 - Setup](00-setup.md), including creating the
   schema-enabled lakehouse `lh_meridian_hr`.
2. Import
   [nb_01_ingest_reference_feeds.ipynb](../notebooks/nb_01_ingest_reference_feeds.ipynb)
   into your Fabric workspace and attach `lh_meridian_hr` as its **default
   lakehouse**.
3. In the **Configuration and Spark setup** code cell:
   - Set `BASE` to the raw HTTPS URL of your repository's `data` folder, without
     a trailing slash. The supplied URL works for the public lab repository.
   - Set `INGEST_EVENTS = True`. This explicitly loads workforce events even if
     the target table already exists or is empty.
4. Ensure all **60** monthly event files, January 2021 through December 2025,
   are published under `data/events`. If the instructor is staging the final
   three months for the pipeline exercise, wait for their release before running
   this full-snapshot path.

No HTTP connection, pipeline, setup notebook, or watermark table is required.

## A.2 Ingest all sources (20 min)

Select **Run all**, or run the cells in order. The notebook:

1. Creates the `bronze` schema if needed.
2. Downloads the pay-grid, FX, and reference feeds to `Files/landing`.
3. Explodes the pay-grid feed's nested `band_history` arrays into
   `bronze.pay_bands`, preserving each effective date for the SCD2 exercise.
4. Loads `bronze.fx_rates`, `bronze.workers`, `bronze.workers_delta`, and
   `bronze.cost_centers`.
5. Downloads the 60 monthly workforce-event CSVs to `Files/landing/events`
   and writes `bronze.workforce_events_raw`, with `_source_file` and
   `_ingested_at` lineage columns.

The notebook preserves source duplicates and intentionally dirty event rows:
cleaning belongs in Silver. It stops on a failed download; the events table is
only replaced after all 60 downloads succeed.

**Reruns:** with `INGEST_EVENTS = True`, each run overwrites the event snapshot
and the reference tables rather than appending another copy. It does not read
or advance `bronze.ingestion_watermark`. Do not run the incremental pipeline
against this snapshot afterward; use a separate lakehouse to compare the paths.

## A.3 Verify the Bronze checkpoint (5 min)

Run this in a new PySpark cell in the same notebook:

```python
required_tables = [
    "workforce_events_raw", "pay_bands", "fx_rates",
    "workers", "workers_delta", "cost_centers",
]
for name in required_tables:
    count = spark.table(f"bronze.{name}").count()
    assert count > 0, f"bronze.{name} is empty"
    print(f"bronze.{name}: {count:,} rows")

events = spark.table("bronze.workforce_events_raw")
assert events.select("_source_file").distinct().count() == 60
print("Raw events:", events.count())  # approximately 124,485

spark.table("bronze.pay_bands") \
    .filter("classification_group='PA' AND classification_level=3") \
    .orderBy("band_effective_date").show()
```

All six tables must be nonempty. Expect approximately **124,485 raw event
rows**, including replayed duplicates, and an annually increasing PA-3 band
midpoint. Run the notebook again and confirm the event row count is unchanged.

Continue to [Module 2 - Silver](02-silver.md). No changes to the remaining
modules are needed.
