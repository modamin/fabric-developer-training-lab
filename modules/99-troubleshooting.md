# Troubleshooting

Common failures, in the order you'll hit them.

## Module 1 — ingestion

**Pipeline Copy fails with 404.** Wrong base/relative URL. Open a raw file in a
browser first. Base must be the **raw** GitHub host; relative URL starts with
`/events/`.

**Pipeline succeeds but no Bronze table.** The pipeline lands *files*; you still
need the load cell (`…saveAsTable("bronze.workforce_events_raw")`) or `nb_01`'s
fallback. Files ≠ tables.

**`Path does not exist: Files/landing/...`.** The lakehouse isn't the **default**
lakehouse in the notebook. Pin `lh_meridian_hr` (📎) and re-run.

**Pay-grid JSON reads as one garbled row.** You forgot
`.option("multiline","true")` — the feed is pretty-printed JSON, not JSON-lines.

## Module 2 — Silver

**Half your events landed in quarantine.** You used `amount <= 0` instead of
`amount < 0`. Most workforce events (leaves, deployments) legitimately have a
**null** amount — null is valid, only **negative** pay is bad. This is the #1
mistake in this lab.

**`AMBIGUOUS_REFERENCE: rate_month`.** Both frames carry `rate_month`. Alias the
FX side (`_rm`, `_ccy`).

**`amount_cad` null for rows that have a local amount.** A currency/month is
missing from FX, or an unknown currency (`XXX`) slipped through instead of being
quarantined under `unresolved_reference`. Check the quarantine table.

**Quarantine counts ~0.** You loaded a cleaned file or pointed `BASE` wrong.
Bronze must hold the raw ~124k rows including the injected dirty ones.

## Module 3 — Gold / SCD2

**Fact row count > Silver row count.** A range join fanned out — a dimension has
**overlapping** validity ranges. Re-check `effective_to = date_sub(next, 1)`; two
versions must not share a day. The notebook's overlap assert catches this.

**Null `worker_key` on some events.** An event predates the worker's first
version. New employees appear in the **2023-07-01** snapshot with
`effective_from = 2023-07-01`; if you hand-edit the data so an event predates
that, it won't resolve. Keep new-hire events on/after their snapshot date (the
generator already does).

**Null `pay_band_key`.** The event's (group, level) isn't in `dim_pay_band`, or
its date is before 2021-01-01 (those rows should have been quarantined as
`date_out_of_range`). All 40 (group, level) combos have history from 2021-01-01.

**MERGE inserts duplicates instead of versioning.** You appended the new snapshot
without the `whenMatchedUpdate` close step, or the `row_hash` includes a column
that changes every run. Hash only the four tracked business attributes.

## Module 4 — semantic model

**Model created as Direct Lake on SQL, not OneLake.** You started from the SQL
endpoint. Create from **lakehouse → New semantic model** and verify the partition
uses `AzureStorage.DataLake`.

**Time-intelligence measures blank.** `dim_date` isn't marked as a date table.

**as-is equals as-was.** Your as-is measure is still filtering through the
versioned relationship. It must use `ALL(dim_pay_band)` + `is_current = TRUE()` to
reach today's grid.

**as-is measure is slow.** `AVERAGEX`/`SUMX` over the whole fact re-evaluates the
current-band lookup per row. Fine for the lab; in production, materialize a
`current_band_mid` column per (group, level) on the fact or a helper table.

## Module 5 — data agent

**No "Data agent" item type.** Tenant setting off, or capacity below F2.

**Agent returns an individual's salary.** Strengthen the instruction (5.2) *and*
rely on Module 6 RLS — the agent should be tested by a region-scoped account.

**Agent picks the wrong measure.** Improve model **descriptions/synonyms** (4.6)
and add an example Q&A pair.

## Module 6 — fixed identity

**Consumer gets a permission error opening a report.** Either the fixed-identity
connection isn't bound, or SSO is still **on** (still querying as the user).
Re-check 6.1–6.2.

**Fixed identity itself can't read the data.** The SPN/managed identity needs
**ReadData** on `lh_meridian_hr` (or Viewer on the workspace). The *fixed*
principal must have lakehouse access — the end users must not.

**RLS doesn't restrict.** You filtered a table the fact doesn't filter through, or
tested as an admin (admins bypass RLS). Use **View as role** or a real second
account, and filter `dim_cost_center[hr_region]`.
