# Cleanup (5 min)

1. **Delete the workspace** (removes everything at once):
   `Meridian-HR-Lab` → **Workspace settings → Remove this workspace**. Deletes the
   lakehouse, notebooks, pipeline, semantic model, and data agent together.

2. To keep the workspace but free storage, drop the data:
   ```python
   for sch in ["bronze","silver","gold"]:
       for t in spark.sql(f"SHOW TABLES IN {sch}").collect():
           spark.sql(f"DROP TABLE {sch}.{t.tableName}")
   ```
   and delete `Files/landing/` in the lakehouse.

3. **Fixed-identity connection & SPN:** if you created a dedicated service
   principal and cloud connection for Module 6, remove the connection under
   **Manage connections and gateways**, and delete/disable the SPN in Entra ID if
   it was lab-only.

4. **Capacity:** if you started a trial/paid F-capacity just for this lab, pause
   or delete it in the Azure portal so it stops billing.

Nothing in this lab touches anything outside the workspace except the optional
service principal in step 3.
