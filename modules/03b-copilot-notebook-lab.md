# Module 3b — Explore the Gold model with Copilot (15 min)

**Goal:** use the **Copilot chat pane** in a Fabric notebook to *generate* simple
analytics queries against the Gold model you just built — you describe the
question in plain English and Copilot writes the code.

This is a light, hands-on interlude between the Gold build (Module 3) and the
semantic model (Module 4). It uses `notebooks/nb_03c_copilot_explore.ipynb`.

> **Preview feature.** The notebook Copilot chat pane is in preview and requires
> Copilot to be enabled for your tenant on a supported capacity. See
> [Use the Copilot chat pane](https://learn.microsoft.com/en-us/fabric/data-engineering/copilot-notebooks-chat-pane).

## Setup

1. Import **`nb_03c_copilot_explore.ipynb`** and **attach** `lh_meridian_hr` as
   the default lakehouse (📎). Copilot reads the attached lakehouse to discover
   your schemas and tables.
2. On the notebook ribbon, select **Copilot** to open the chat pane on the right.
3. Copilot asks permission before it adds or runs a cell. Review the diff, then
   **Keep** or **Undo**.

## What the students do

The notebook has five tasks, each a plain-English question plus a ready-to-paste
prompt. Students paste the prompt into the chat pane (or use **in-cell Copilot**
on the empty cell under each task), review the generated code, run it, and
sanity-check the result.

| Task | Question | Shape |
|------|----------|-------|
| 1 | Event volume by `event_type` | one table, group-by |
| 2 | Hires vs Departures per year | join to `dim_date` |
| 3 | Average as-was compa-ratio per classification group | filter + group-by |
| 4 | Below-band pay events per `hr_region` | join to `dim_cost_center` |
| 5 | Top directorates by hiring | join to `dim_worker`, top-N |

## Teaching points

- **Attach the lakehouse first** — without it, Copilot has no schema context.
- **Name tables and columns in the prompt** to get precise SQL on the first try.
- **Always review before Keep** — read the diff and confirm the result is sane.
- **Slash commands:** `/explain`, `/optimize`, `/fix` (or **Fix with Copilot**
  under a failed cell).
- **Follow-ups keep context** — refine a result by asking for another cut.

➡️ Continue to **Module 4 — Direct Lake on OneLake semantic model**.
