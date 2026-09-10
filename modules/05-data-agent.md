# Module 5 — Fabric data agent over the model (25 min)

**Goal:** a natural-language **data agent** that answers Workforce Analytics
questions against `sm_meridian_workforce`, and an understanding of what a
*consumer* of that agent needs — which sets up Module 6.

> **Capacity & tenant prerequisites.** Data agents need **F2+ (or P1+)**, plus
> the **data agent** tenant setting (and **cross-geo processing/storing for AI**
> if your capacity and AI regions differ). If **Data agent** is missing from
> **+ New item**, that's the tenant setting — talk to your Fabric admin.

## 5.1 Create the agent (~5 min)

1. In `Meridian-HR-Lab`: **+ New item → Data agent**, name it
   `da_meridian_workforce`.
2. **Add data source →** the semantic model `sm_meridian_workforce`.
3. Include the fact and all four dimensions.

## 5.2 Give the agent instructions (~8 min)

Encode the HR domain so answers are correct **and safe**:

```
You answer questions for Meridian's Workforce Analytics team about staffing
movement and pay-grid compliance.

Definitions:
- "Compa-ratio" = salary divided by the pay-grid midpoint for that classification.
- "as-was" = judged against the grid in force when the pay was set (default).
- "as-is" / "today's grid" = judged against the current re-benchmarked grid.
- "Below band" = base salary below the grid minimum for the classification.
- "Comp events" = Hire, Promotion, and Step Increment (they set a base salary).
- "Movement" = Hires minus Departures.
- Classification is a group (e.g. PA, IT, EC) plus a level (1-5).
- Employment types: Indeterminate, Term, Casual, Student.

Rules:
- Report money in CAD.
- If a question could mean as-was or as-is, answer as-was and note the as-is value.
- Aggregate only. NEVER return an individual employee's name or salary, and never
  list individuals. If asked for a person's pay, decline and offer the aggregate
  for their classification or team instead.
`
## 5.4 Test it (~5 min)

1. *"What is our average compa-ratio as-was vs as-is?"*
2. *"How many hires and departures in Ontario in 2024?"*
3. *"Which groups are most often paid below band?"*
4. *"What's the promotions year-over-year change?"*

Reconcile against the Module 4 matrix. Try *"What is EMP01234's salary?"* and
confirm the agent **declines** and offers an aggregate.

> **Guardrails to note.** Results cap at ~25 rows — analysis, not extract. If the
> agent picks the wrong measure, fix the model's **descriptions/synonyms** (4.6),
> don't over-stuff the instructions.

## 5.5 What a consumer needs — the bridge to Module 6

An HRBP who only *uses* the agent needs **Read** on the semantic model — nothing
on the workspace, nothing on the lakehouse. But with a default Direct Lake model,
"Read on the model" still evaluates queries as **that user** against the OneLake
Delta files — which they can't read, and which you don't want to grant, because
those files *are* the raw salaries. **Fixed identity** (Module 6) is how you give
them the model without the lake.

➡️ Continue to **Module 6 — Fixed identity, SSO & OneLake security**.
