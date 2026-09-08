# Module 6 — Fixed identity & row-level security (25 min)

**Goal:** let HRBPs and agent users query `sm_meridian_workforce` **without any
access to the underlying lakehouse**, and enforce **row-level security by HR
region** — proving that even a user who tries to go around the model can't read
another region's compensation data.

This module matters more here than in most labs: the OneLake Delta files behind
this model contain individual salaries. "Read on the model, nothing on the lake"
is the whole governance requirement.

## The problem (recap)

A default Direct Lake on OneLake model evaluates queries as the **requesting
user**, so a consumer would need read access on the OneLake Delta tables — i.e.
the raw salary files. That's backwards. **Fixed identity** makes the model query
OneLake as **one fixed principal**; users then need only **Read on the model**.

## 6.1 Create a fixed-identity cloud connection (~8 min)

You need a principal that *does* have read access to the lakehouse — a **service
principal** or a **managed identity**.

1. Give that principal data access: add it to `Meridian-HR-Lab` as a **Viewer**,
   or grant **ReadData** on `lh_meridian_hr` (lakehouse → **Manage OneLake data
   access** / item permissions). This is the identity that will actually read the
   Delta files.
2. **Settings → Manage connections and gateways → New → Cloud connection.**
   - Connection type: the **Direct Lake / OneLake** source for your model.
   - **Authentication:** **Service principal** (tenant/client/secret) or **Managed
     identity**.
   - **Single sign-on: OFF.** This switch is what makes the connection use the
     *fixed* identity instead of the caller's.
   - Save; note the connection name.

## 6.2 Bind the model to the fixed identity (~5 min)

1. `sm_meridian_workforce` → **Settings → Gateway and cloud connections** (data
   source settings).
2. Map the model's data source to the **cloud connection** from 6.1.
3. Confirm the model now uses that connection with SSO off.

The model now reads OneLake as the fixed principal. A user with only **Read** on
the model gets data back and needs **nothing** on the lakehouse.

## 6.3 Grant report builders the model only (~3 min)

1. `sm_meridian_workforce` → **Manage permissions / Share.**
2. Grant your HRBP group **Read** (add **Build** only if they author reports). Do
   **not** grant workspace or lakehouse access.
3. That same Read is what the Module 5 agent consumers need.

## 6.4 Row-level security by HR region (~7 min)

Region lives on `dim_cost_center[hr_region]`, and the fact filters through it, so
a role there restricts every measure.

1. Model → **Modeling → Manage roles → New.**
2. Role **`RLS_Pacific`**, table `dim_cost_center`:
   ```dax
   [hr_region] = "Pacific"
   ```
3. Repeat per region, or go dynamic with a mapping table:
   ```dax
   [hr_region] =
       LOOKUPVALUE (
           region_map[hr_region], region_map[user_email], USERPRINCIPALNAME ()
       )
   ```
4. **View as → RLS_Pacific** and confirm the numbers drop to the Pacific slice.

## 6.5 Prove it can't be bypassed (~2 min)

Sign in as a **second test account** in the HRBP group, assigned `RLS_Pacific`:

- Opening a report or the data agent → **works**, Pacific rows only.
- Trying to open `lh_meridian_hr` or query its **SQL endpoint** directly →
  **denied**. They have no lakehouse permission; the only door is the model, and
  the model is filtered by their role.

Contrast with the default (non-fixed) setup: you'd have had to grant lakehouse
ReadData, and a curious HRBP could query the SQL endpoint and read **every
region's salaries**, sailing past RLS entirely. Fixed identity closes that door —
which, for compensation data, is the difference between a governed model and a
privacy incident.

## Wrap-up

Raw extracts → conformed Silver (null-safe) → a Gold star schema with two SCD
Type 2 dimensions and as-of resolution → a Direct Lake on OneLake model with an
as-was/as-is compa-ratio library → a data agent → a governed sharing model where
HRBPs see only their region and never touch the lake.

➡️ Optional: **modules/99-troubleshooting.md** and **modules/cleanup.md**.
