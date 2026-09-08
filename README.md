# Meridian People Analytics — Microsoft Fabric End-to-End Lab

A hands-on, ~3.5–4 hour lab that walks a data engineer through a complete
Microsoft Fabric medallion build — ingestion, Silver conformance, a Gold
dimensional model with **two flavours of SCD Type 2**, a **Direct Lake on
OneLake** semantic model, a **Fabric data agent**, and a **fixed-identity**
security pattern that matters especially for compensation data.

## The scenario

> **Meridian** is a (fictional) federal Crown corporation with domestic and
> international offices. Its **Workforce Analytics** team needs to see staffing
> movement, classification/pay-grid compliance, and pay equity — including
> whether a salary was *within the pay grid in force when it was set*, versus
> today's re-benchmarked grid.

That "as-was vs as-is" question is why this lab uses SCD Type 2 for real:
classification **pay grids are re-benchmarked every fiscal year** through
collective-agreement economic increases, and each pay event must be judged
against the grid **in force on the event date**.

This is deliberately **not** a generic headcount dashboard. The grain is a
**workforce event** (hire, promotion, step increment, performance pay, lateral
deployment, leave start/return, departure) per employee / cost centre / date,
with pay amounts in local currency and CAD.

## What you build

```
Bronze  ── pipeline (60 monthly event extracts) + notebook (nested pay-grid JSON, FX, masters)
Silver  ── dedup replays · conform ISO-2/ISO-3 work country · quarantine · convert to CAD
Gold    ── dim_date · dim_cost_center (SCD1)
           dim_pay_band (SCD2 from a history feed)
           dim_worker   (SCD2 via periodic-snapshot MERGE)
           fact_workforce_event (as-of surrogate resolution + compa-ratio)
Model   ── Direct Lake on OneLake · compa-ratio as-was/as-is measures · Prep for AI
Agent   ── Fabric data agent over the model
Security── fixed identity · SSO off · RLS by HR region
```

## Modules

| # | File | Focus | Time |
|---|------|-------|------|
| 0 | `modules/00-setup.md` | workspace, lakehouse, host the data | 15 min |
| 1 | `modules/01-bronze-ingestion.md` | pipeline + notebook ingestion | 45 min |
| 2 | `modules/02-silver.md` | dedup, conform, quarantine, FX | 40 min |
| 3 | `modules/03-gold-scd2.md` | dimensions, **SCD2**, as-of fact | 55 min |
| 3b | `modules/03b-copilot-notebook-lab.md` | generate queries with the Copilot chat pane | 15 min |
| 4 | `modules/04-semantic-model.md` | Direct Lake on OneLake + DAX | 35 min |
| 5 | `modules/05-data-agent.md` | data agent over the model | 25 min |
| 6 | `modules/06-security-fixed-identity.md` | fixed identity + RLS | 25 min |
|   | `modules/99-troubleshooting.md` | common failures | — |
|   | `modules/cleanup.md` | tear down | 5 min |

## Notebooks

| Notebook | Used in | Kernel |
|----------|---------|--------|
| `notebooks/nb_00_generate_source_data.ipynb` | *instructor only* — regenerate data | Python |
| `notebooks/nb_00_setup_lakehouse.ipynb` | Module 1 target-table setup | PySpark |
| `notebooks/nb_00_setup_watermark.ipynb` | Module 1 incremental watermark | PySpark |
| `notebooks/nb_00_list_event_files.ipynb` | Module 1 incremental file discovery | PySpark |
| `notebooks/nb_01_ingest_reference_feeds.ipynb` | Module 1 | PySpark |
| `notebooks/nb_02_silver_conform_dedup.ipynb` | Module 2 | PySpark |
| `notebooks/nb_03a_gold_dimensions_scd2.ipynb` | Module 3 | PySpark |
| `notebooks/nb_03b_gold_fact_asof.ipynb` | Module 3 | PySpark |
| `notebooks/nb_03c_copilot_explore.ipynb` | Module 3b (Copilot chat pane) | PySpark |

## Prerequisites

- A Fabric capacity **F2 or higher** (or Power BI Premium P1+). The data agent
  and Direct Lake features require a paid capacity — the free trial won't cover
  Module 5.
- **Contributor** or higher on a Fabric workspace.
- Tenant settings enabled by your Fabric admin: **data agent** and, if your
  capacity is outside your data region, **cross-geo processing/storing for AI**.
- A place to host the `data/` folder over HTTPS (a public GitHub repo is easiest).

## Data

Everything under `data/` is synthetic and generated deterministically by
`generate.py`. ~118k clean workforce events across 2021–2025, 8 classification
groups × 5 levels with real annual pay-grid history, 5,000 employees (with a
second snapshot), 120 cost centres, and monthly FX. Fork this repo, keep `data/`
public, and point the lab's `BASE` URL at your fork.

> ⚠️ Synthetic data only. Meridian is fictional; no real employees, salaries, or
> pay grids are used. Treat the confidentiality lessons in Module 6 as the point,
> not the data.
