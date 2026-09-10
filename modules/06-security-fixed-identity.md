# Module 6 — Fixed identity, SSO & OneLake security (40 min)

**Goal:** compare two Direct Lake security patterns with a peer:

1. **OAuth fixed identity, SSO off:** the peer can query the semantic model
   without access to the underlying lakehouse.
2. **SSO on:** the peer receives lakehouse access through a OneLake security
   role that can read all tables but restricts `dim_worker` to one directorate.

This module matters more here than in most labs: the OneLake Delta files behind
this model contain individual salaries. The first pattern demonstrates "Read on
the model, nothing on the lake"; the second demonstrates governed source access.

## The problem (recap)

A default Direct Lake on OneLake model evaluates queries as the **requesting
user**, so a consumer would need read access on the OneLake Delta tables — i.e.
the raw salary files. That's backwards. **Fixed identity** makes the model query
OneLake as **one fixed principal**; users then need only **Read on the model**.

When SSO is enabled, the model instead uses the requesting user's identity.
That user needs source access, and OneLake security can limit the source rows
and columns available to them across supported Fabric engines.

## 6.1 Pick a peer for testing (~2 min)

Work in pairs. Choose another student and exchange the Microsoft Entra email
addresses used to sign in to Fabric. Each student configures their own lab and
uses their peer as the test consumer.

Your peer must **not** be an Admin, Member, or Contributor in your workspace and
must not already have **ReadAll** on `lh_meridian_hr`. Those broader grants aren't
restricted by OneLake security roles and would invalidate the test. If your peer
already has workspace access above Viewer, use another peer or ask the instructor
for a test account.

## 6.2 Create an OAuth fixed-identity connection (~8 min)

For this lab, use your signed-in student account as the fixed identity. No
service principal, workspace identity, or managed identity is required. The
OAuth credentials stored in the cloud connection identify the student who
creates the connection.

1. Confirm that your student account can read the source lakehouse. Because you
   created the lab artifacts as a workspace Contributor or higher, you already
   have the required **Read** and **ReadAll** permissions on
   `lh_meridian_hr`. Do not remove that access while this connection is in use.
2. Select **Settings** (gear icon) → **Manage connections and gateways**.
3. Select **New** → **Cloud connection**.
4. Configure the connection for the **OneLake / Direct Lake** data source used
   by `sm_meridian_workforce`. Use the server or OneLake path shown in the
   semantic model's data-source settings if the dialog requests one.
5. Set **Authentication method** to **OAuth 2.0**.
6. Select **Sign in** and authenticate with the same student account that owns
   and can read the lab lakehouse.
7. Turn **Single sign-on (SSO) through Microsoft Entra ID** **off**. This is the
   critical setting: with SSO off, report queries use the OAuth identity stored
   in this connection instead of the report consumer's identity.
8. Name the connection `conn_meridian_onelake_oauth`, select **Create**, and
   note the connection name for the next section.

> **Why OAuth is still a fixed identity:** OAuth is the authentication method;
> SSO controls whose identity is used. OAuth with SSO off always uses the
> connection owner's stored identity, making that user the model's fixed
> identity. OAuth with SSO on delegates each query to the current consumer and
> is not the configuration required by this lab.

## 6.3 Bind the model to the fixed identity (~5 min)

1. `sm_meridian_workforce` → **Settings → Gateway and cloud connections** (data
   source settings).
2. Map the model's OneLake data source to
   `conn_meridian_onelake_oauth` from 6.2.
3. Confirm that the mapped connection shows **OAuth 2.0** and SSO **off**.
4. If Fabric prompts you to take over the semantic model, do so only with the
   same student account used to create the OAuth connection. The semantic model
   owner also needs source access for Direct Lake framing.

The model now reads OneLake as the fixed student account. A consumer with only
**Read** on the model gets data back and needs **nothing** on the lakehouse.

## 6.4 Test model-only access with your peer (~5 min)

1. `sm_meridian_workforce` → **Manage permissions / Share.**
2. Grant your peer **Read**. Add **Build** only if they need to author a report.
3. Share an existing report based on this model with the peer, or create a quick
   report that shows `dim_worker[directorate]` and `[Event Count]`.
4. Do **not** grant the peer workspace or lakehouse access yet.
5. Ask the peer to open the report. It should load because the model queries
   OneLake as your fixed OAuth identity.
6. Ask the peer to open `lh_meridian_hr`. Access should be denied because the
   peer has semantic-model/report access only.

This proves the fixed-identity pattern: consumers can use the governed model
without receiving access to the source lakehouse. There is no semantic-model RLS
in this part of the lab.

## 6.5 Turn on SSO (~3 min)

Now change the model to evaluate source permissions as the requesting user.

1. Open **Settings** → **Manage connections and gateways**.
2. Find `conn_meridian_onelake_oauth` and select **Edit**.
3. Turn **Single sign-on (SSO) through Microsoft Entra ID** **on**, then save.
4. Return to `sm_meridian_workforce` → **Settings** → **Gateway and cloud
   connections** and confirm that the model is still mapped to this connection.

Ask the peer to refresh the report now. It should fail because SSO sends the
peer's identity to OneLake and the peer doesn't yet have lakehouse data access.

## 6.6 Create a OneLake reader role for all tables (~7 min)

> Creating OneLake security roles requires suitable permissions on the
> lakehouse, typically workspace Admin or Member. If **Manage OneLake security**
> isn't available, ask the instructor or workspace administrator to perform
> these steps using your peer's email address.

1. Open `lh_meridian_hr` and select **Manage OneLake security**.
2. Select **New** and configure the role:
   - **Role name:** `FieldOperationsReader`
   - **Role type:** **Grant**
   - **Permission:** **Read**
3. For **Add data to your role**, choose **Selected data**, select the
   **Tables** node and all schemas/tables beneath it, then select **Add data**.
   Do not select the **Files** node. This grants the role access to every table
   currently in the lakehouse without exposing landing files.
4. Add your peer's Fabric sign-in email as a role member.
5. Select **Create**.
6. Give the peer control-plane access to the lakehouse: open **Manage
   permissions** for `lh_meridian_hr`, add the peer, and grant **Read** only.
   Do not grant **ReadAll**, **Write**, or a Contributor-or-higher workspace
   role.

Before adding RLS, ask the peer to refresh the report. It should now load under
SSO because the peer has lakehouse **Read** plus table access from the OneLake
security role.

## 6.7 Restrict `dim_worker` to one directorate (~5 min)

Add row security to the same role. OneLake security uses a SQL `SELECT` rule,
not a DAX role on the semantic model.

1. Return to **Manage OneLake security** and open
   `FieldOperationsReader`.
2. Find `gold.dim_worker`, select **...** → **Row security**.
3. Paste this rule and select **Save**:

   ```sql
   SELECT * FROM gold.dim_worker
   WHERE directorate = 'Field Operations'
   ```

4. Confirm that your peer is not also a member of `DefaultReader` or another
   role that grants unrestricted access to `gold.dim_worker`. OneLake roles use
   additive grants; a broader role can defeat this restricted-role test.

The restriction applies at the OneLake data layer. Direct Lake on OneLake uses
the peer's identity under SSO and receives only the permitted `dim_worker` rows.

## 6.8 Build and test the report table (~5 min)

1. As the model/report author, create or edit a report based on
   `sm_meridian_workforce`.
2. Add a **Table** visual.
3. Add these fields:
   - `dim_worker[directorate]`
   - `[Event Count]`
   - `[Hires]`
   - `[Performance Pay CAD]`
4. Save the report and share it with your peer with **Read** access.
5. Ask the peer to reopen or refresh the report in their own browser session.

The peer should see exactly one row: **Field Operations**. The measures on that
row should reflect events related to worker versions in that directorate. As the
author, you can still see all directorates because Admin, Member, and Contributor
workspace roles aren't restricted by OneLake security roles.

If the peer sees every directorate, check for workspace Contributor-or-higher
membership, `ReadAll`, `DefaultReader` membership, or another unrestricted
OneLake role. If the report fails, confirm SSO is on and the role includes all
five Gold model tables.

## 6.9 Compare the two patterns

| Configuration | Effective identity | Peer needs lakehouse access? | Result |
|---|---|---|---|
| OAuth connection, SSO off | Connection owner's student account | No | Report works; lakehouse is denied |
| OAuth connection, SSO on | Peer's account | Yes | OneLake role permits tables and filters `dim_worker` |

Fixed identity is appropriate when consumers should use the semantic model but
must not access the source. SSO plus OneLake security is appropriate when the
same data-layer rules should follow users across supported Fabric engines.

## Wrap-up

Raw extracts → conformed Silver (null-safe) → a Gold star schema with two SCD
Type 2 dimensions and as-of resolution → a Direct Lake on OneLake model with an
as-was/as-is compa-ratio library → a data agent → two governed access patterns:
model-only access through fixed identity, and user-specific source access through
SSO plus OneLake security.

➡️ Optional: **modules/99-troubleshooting.md** and **modules/cleanup.md**.
