# Module 1 — Bronze ingestion (45 min)

**Goal:** land raw source data in the `bronze` schema exactly as it arrives — no
cleaning. Two ingestion styles, chosen for what each is good at:

- **Data pipeline** → the 60 flat monthly workforce-event extracts. Repeatable
  batch copy with a month-offset loop.
- **Notebook** → the nested JSON pay-grid feed, FX, and reference masters. A
  pipeline Copy activity flattens nested arrays poorly; a notebook does it in a
  few lines.

---

## Part A — Pipeline: copy the monthly event extracts (25 min)

The event files are named `workforce_events_YYYY-MM.csv`. A single notebook does
all of the incremental logic: it reads the watermark, lists the source files on
GitHub, and returns the array of new files to copy. The pipeline just loops over
that array with a plain Copy activity, then advances the watermark after every
Copy succeeds. No pipeline-side date arithmetic.

### A.1 Create the pipeline

1. Import these notebooks into the workspace and attach `lh_meridian_hr` as the
   default lakehouse for each:
   - `notebooks/nb_00_setup_lakehouse.ipynb` — pre-creates the target table.
   - `notebooks/nb_00_list_event_files.ipynb` — reads the watermark, lists the
     GitHub files, and returns the new ones.
   - `notebooks/nb_00_setup_watermark.ipynb` — advances the watermark.
2. In `nb_00_list_event_files` and `nb_00_setup_watermark`, mark Cell 2 as the
   **parameter cell** so the pipeline can override the values.
3. In `Meridian-HR-Lab`: **+ New item → Data pipeline**, name it
   `pl_ingest_events`.
4. Add a **Notebook** activity named `SetupTargetTable` and select
   `nb_00_setup_lakehouse`.

### A.2 List the new files in a notebook

1. Add a **Notebook** activity named `ListNewFiles`, select
   `nb_00_list_event_files`, and connect it after `SetupTargetTable` with an
   **On success** dependency. No base parameters are required for the defaults.
2. The notebook creates `bronze.ingestion_watermark` if missing, treats a first
   run as December 2020, lists the files through the GitHub Contents API, keeps
   only months later than the watermark (capped at 60), and exits this JSON:
   ```json
   {
     "files": ["events/workforce_events_2021-01.csv", "..."],
     "watermark": "2025-12-01 00:00:00",
     "count": 60
   }
   ```
   - `files` — the relative Copy paths the ForEach iterates over.
   - `watermark` — the month to store after the batch succeeds.
   - `count` — how many files were selected (0 when already current).

### A.3 Loop over the returned files

1. Add a **ForEach** activity named `ForEachFile` after `ListNewFiles`.
2. Select **Settings** and set **Items** to the `files` array from the notebook
   exit value:
   ```text
   @json(activity('ListNewFiles').output.result.exitValue).files
   ```
3. Leave **Sequential** unchecked. `SetupTargetTable` already created the target
   table, so parallel Copy activities append instead of racing to create it.
   When `files` is empty the loop simply does nothing.

### A.4 Copy activity inside the loop

1. Inside `ForEachFile`, add **Copy data** → `CopyFile`.
2. **Source**: create an HTTP connection with authentication set to
   **Anonymous**. Enter the following literal value in the connection **URL**;
   make sure it ends with `/`:
   ```text
   https://raw.githubusercontent.com/modamin/fabric-developer-training-lab/main/data/
   ```
   Set the relative URL in the Copy activity to the current item — the notebook
   already returns the correct path, so no expression building is needed:
   ```text
   @item()
   ```
   Format DelimitedText, header on.
3. **Destination**: select Lakehouse `lh_meridian_hr`, choose **Tables**, and
   select the `bronze` schema. Set the table name to `workforce_events_raw` and
   the table action to **Append**. The pipeline writes directly to the Delta
   table `bronze.workforce_events_raw`.

### A.5 Update the watermark after the batch

1. Add a **Notebook** activity named `UpdateWatermark` after `ForEachFile` with
   an **On success** dependency. Select `nb_00_setup_watermark`.
2. Under **Base parameters**, set:
   - `mode` = `update`.
   - `pipeline_name` = `workforce_events`.
   - `watermark_timestamp` = the `watermark` value the discovery notebook
     returned:
     ```text
     @json(activity('ListNewFiles').output.result.exitValue).watermark
     ```

When no new files were found, the notebook returns the existing watermark, so
this update is harmless. If any Copy fails, `UpdateWatermark` does not run and
the failed batch is retried from the same watermark.

### A.6 Run and verify

You run this pipeline three times to see incremental loading end to end. At the
start only the months through **2025-09** are published; the instructor releases
the final three files (`2025-10`, `2025-11`, `2025-12`) between runs.

1. **Initial run.** The watermark is December 2020, so `ListNewFiles` returns
   every published file and the loop copies them through **2025-09**. Query
   `bronze.ingestion_watermark` and confirm `watermark_timestamp` is
   `2025-09-01 00:00:00`. 
2. **Incremental run.** After the instructor publishes the last three files,
   run the pipeline again. `ListNewFiles` now returns only `2025-10`, `2025-11`,
   and `2025-12`, so the loop runs exactly **three** Copy activities and the
   watermark advances to `2025-12-01 00:00:00`.
3. **No-op run.** Run the pipeline a third time. With the watermark at the
   latest month, `ListNewFiles` returns zero files, the loop does nothing, and
   the watermark stays at `2025-12-01 00:00:00`.
4. Now open **Tables → bronze → workforce_events_raw** and confirm the Delta
   table contains approximately **124,485** rows.

> **Why ~124k and not ~118k?** Bronze keeps the replayed duplicates and the
> intentionally dirty rows. Silver fixes those — Bronze stays faithful.

---

## Part B — Notebook: pay-grid feed, FX, reference masters (20 min)

1. Import **`nb_01_ingest_reference_feeds.ipynb`**.
2. **Attach** `lh_meridian_hr` as the default lakehouse (📎). Required for the
   `Files/…` paths and `bronze.*` names to resolve.
3. Set `BASE` in the first cell to your raw base URL.
4. **Run all**. The notebook:
   - downloads the feeds into `Files/landing/…`
   - **explodes** the nested `band_history` array → `bronze.pay_bands` (one row
     per group/level/effective date — the SCD2 source)
   - lands `bronze.fx_rates`, `bronze.workers`, `bronze.workers_delta`,
     `bronze.cost_centers`
   - confirms `bronze.workforce_events_raw` exists (falls back to load it if the
     pipeline hasn't run)

### Checkpoint

```python
for t in spark.sql("SHOW TABLES IN bronze").collect(): print(t.tableName)
spark.table("bronze.pay_bands").filter("classification_group='PA' AND classification_level=3") \
     .orderBy("band_effective_date").show()
```

You should see the PA-3 band midpoint step up each year — that history is the
whole point of Module 3.

➡️ Continue to **Module 2 — Silver**.
