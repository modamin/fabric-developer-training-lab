## 0.1 Create the lakehouse

We use **one schema-enabled lakehouse** with `bronze` / `silver` / `gold`
schemas. Direct Lake on OneLake works cleanly against the `gold` schema.

1. **+ New item → Lakehouse**. Name it `lh_meridian_hr`.
2. Tick **Lakehouse schemas (public preview)** so the lakehouse is
   schema-enabled. (If you miss this, run the lab with flat table names — drop the
   `bronze.`/`silver.`/`gold.` prefixes.)

## 0.2 Download notebooks and import in your workspace

Downloads all notebooks in the notebooks folder. 
Go to your workspace > Import > from computer > Select the notebooks. 


➡️ Continue to **Module 1 — Bronze ingestion**.
